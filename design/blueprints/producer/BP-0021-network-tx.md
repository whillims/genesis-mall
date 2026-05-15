# 蓝图文档：BP-0008-network_tx

**文档编号**：BP-0008  
**蓝图名称**：网络控制指令生产者（发送端）  
**对应场景**：SC-0005  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**网络控制指令发送生产者**。它订阅商场SHM中的控制指令矢量，将指令通过TCP/UDP发送至外部执行器，并接收ACK确认。该生产者是商场**输出到物理世界**的最终闸门，必须具备**高可靠性**与**故障恢复能力**。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 订阅控制指令、TCP/UDP发送、ACK接收、超时重传、故障恢复 |
| **不负责** | 控制算法、遥测数据接收、信号分析、UI显示 |
| **输入** | SHM矢量 `/shm/control/tx_command` |
| **输出** | 外部执行器（TCP/UDP网络） |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `network_tx_init()` | 初始化订阅、建立外部连接池 | 商场main()调用一次 |
| `network_tx_loop()` | 主事件循环（等待指令、发送、等待ACK） | 独立Worker进程持续运行 |
| `read_tx_command()` | 从SHM读取控制指令 | 被唤醒后调用 |
| `validate_command()` | 验证指令格式与商场安全标记 | 读取后调用 |
| `select_connection()` | 从连接池选择目标连接 | 验证通过后调用 |
| `send_command_tcp()` | 通过TCP发送指令 | 选择连接后调用 |
| `send_command_udp()` | 通过UDP发送指令 | 选择连接后调用 |
| `wait_for_ack()` | 等待ACK确认（带超时） | 发送后调用 |
| `handle_timeout()` | 处理ACK超时（重传或标记失败） | 超时后调用 |
| `network_tx_cleanup()` | 关闭连接、释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `network_tx_init(config: dict) -> int`

```
参数:
  config["subscribe_vector"]  : string  - 订阅矢量，默认 "/shm/control/tx_command"
  config["target_hosts"]      : list[dict] - 目标执行器列表，每项含 ip, port, protocol
  config["max_retries"]       : uint8   - 最大重传次数，默认 3
  config["ack_timeout_ms"]    : uint16  - ACK超时时间（毫秒），默认 1000
  config["connection_pool_size"]: uint8  - 连接池大小，默认 10

返回:
  0  : 成功
  -1 : 订阅注册失败
  -2 : 连接建立失败

行为:
  1. 调用 mall_subscribe("network_tx_001", config["subscribe_vector"])
  2. 遍历 target_hosts，建立TCP长连接或预绑定UDP Socket
  3. 将连接存入连接池（state["connections"]）
  4. 初始化重传队列（state["pending_acks"]）
  5. 返回状态码

约束:
  - 目标IP必须在商场白名单中（由商场安全层配置）
  - TCP连接必须设置 keepalive（防止静默断线）
  - UDP Socket必须设置非阻塞模式
```

#### `network_tx_loop(state: dict) -> void`

```
参数:
  state : dict - 运行时状态

行为:
  while mall_is_running():
      # 检查重传队列（优先处理未确认指令）
      for cmd_id, pending in state["pending_acks"].items():
          if mall_get_elapsed_ms(pending["send_time"]) > state["ack_timeout_ms"]:
              if pending["retry_count"] < state["max_retries"]:
                  handle_timeout(state, pending)
              else:
                  # 标记失败，通知商场
                  mall_notify_command_failed(cmd_id, "MAX_RETRIES_EXCEEDED")
                  del state["pending_acks"][cmd_id]

      # 等待新指令（非阻塞，快速轮询重传队列）
      packet = mall_shm_read(state["subscribe_handle"], timeout=10ms)

      if packet is not None:
          # 读取并验证指令
          command = read_tx_command(packet)
          if not validate_command(command, state):
              state["invalid_command_count"] += 1
              mall_yield()
              continue

          # 选择连接并发送
          conn = select_connection(command, state)
          if conn is None:
              mall_notify_command_failed(command["command_id"], "NO_CONNECTION")
              mall_yield()
              continue

          # 发送指令
          if command["protocol"] == 1:
              send_command_tcp(command, conn, state)
          else:
              send_command_udp(command, conn, state)

          # 记录待确认
          state["pending_acks"][command["command_id"]] = {
              "command": command,
              "send_time": mall_get_timestamp_ms(),
              "retry_count": 0,
              "conn": conn
          }

      # 检查ACK（非阻塞读取）
      check_incoming_acks(state)

      mall_yield()

约束:
  - 重传队列检查优先级高于新指令处理
  - 每次循环必须调用 mall_yield()
  - 连接断开时必须自动重连（最多3次）
```

#### `validate_command(command: dict, state: dict) -> bool`

```
参数:
  command : dict - 解析后的控制指令
  state   : dict - 运行时状态

返回:
  bool : True=通过，False=拒绝

行为:
  1. 验证商场安全标记：command["safety_check"] 必须为 1
  2. 验证目标IP白名单：command["target_ip"] 必须在 state["target_hosts"] 中
  3. 验证指令格式：所有必填字段存在且类型正确
  4. 验证 command_id 单调性：必须 > state["last_command_id"]
  5. 全部通过 → True，任一失败 → False

约束:
  - 安全标记验证是第一道防线，未通过立即拒绝
  - command_id 非单调可能表示乱序或重放攻击，必须拒绝
```

#### `send_command_tcp(command: dict, conn: dict, state: dict) -> int`

```
参数:
  command : dict - 控制指令
  conn    : dict - 连接对象
  state   : dict - 运行时状态

返回:
  0  : 发送成功
  -1 : 发送失败（连接断开）

行为:
  1. 序列化指令为二进制帧：
     | command_id (4B) | command_type (1B) | command_param (8B) | checksum (4B) |
  2. 添加帧头长度字段：| frame_len (4B) | 帧体 | CRC32 (4B) |
  3. 通过TCP发送完整帧
  4. 更新统计：state["tcp_sent_count"] += 1
  5. 返回状态码

约束:
  - 发送必须原子完成（帧不可分割）
  - 若发送失败（连接断开），标记连接为"失效"，触发重连
```

#### `check_incoming_acks(state: dict) -> void`

```
行为:
  1. 遍历所有TCP连接，非阻塞读取ACK帧
  2. ACK帧格式：| command_id (4B) | status (1B) | timestamp (8B) |
  3. 若收到ACK：
     - 从 pending_acks 中移除对应 command_id
     - 通知商场：mall_notify_command_acked(command_id)
     - 更新统计：state["ack_count"] += 1
  4. 若收到NACK（status != 0）：
     - 标记指令失败
     - 通知商场：mall_notify_command_failed(command_id, "NACK")

约束:
  - ACK检查必须非阻塞，不得影响主循环响应速度
  - 乱序ACK必须正确处理（先收到ACK-2再收到ACK-1）
```

---

## 4. SHM接口契约

### 4.1 订阅矢量（消费者端）

| 矢量路径 | 权限 | 说明 |
|----------|------|------|
| `/shm/control/tx_command` | READ | 控制指令（带优先级队列） |

### 4.2 商场API依赖

```
mall_subscribe(consumer_id: str, vector_path: str) -> handle
mall_shm_read(handle, timeout_ms: int) -> bytes | None
mall_get_timestamp_ms() -> uint64
mall_get_elapsed_ms(start_time: uint64) -> uint64
mall_notify_command_acked(command_id: uint32) -> void
mall_notify_command_failed(command_id: uint32, reason: str) -> void
mall_is_running() -> bool
mall_yield() -> void
```

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0008-01 | TCP指令发送 | 合法控制指令 | 外部设备收到正确帧，返回ACK |
| T-0008-02 | UDP指令发送 | 合法控制指令 | 外部设备收到正确报文 |
| T-0008-03 | ACK超时重传 | 外部设备不回复ACK | 3次重传后标记失败 |
| T-0008-04 | 安全标记拒绝 | safety_check=0的指令 | 拒绝发送，记录违规 |
| T-0008-05 | 白名单拒绝 | target_ip不在白名单 | 拒绝发送，记录违规 |
| T-0008-06 | 连接断开恢复 | 发送中途TCP断开 | 标记失效，自动重连，重传 |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0008-I01 | SC-0005集成 | 遥测→控制→发送→ACK→新遥测，闭环延迟<150ms |
| T-0008-I02 | 故障恢复 | 外部执行器断线10秒 | 自动重连，未确认指令重传或标记失败 |
| T-0008-I03 | 高并发 | 100条指令连续涌入 | 连接池调度正确，无数据丢失 |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0008"
blueprint_name: "network_tx"
version: "1.0"

producer:
  id: "network_tx_001"
  role: "producer"  # 注意：虽然是发送端，但角色仍是producer（向外部世界"生产"动作）

shm_permissions:
  consume:
    - vector: "/shm/control/tx_command"
      mode: "read"

functions:
  - name: "network_tx_init"
    safety_level: "init"
    side_effects: ["shm_subscribe", "tcp_connect", "udp_bind"]
  - name: "network_tx_loop"
    safety_level: "continuous"
    side_effects: ["shm_read", "network_send", "network_recv"]
    max_cpu_time_per_cycle: "50ms"
  - name: "validate_command"
    safety_level: "per_call"
    side_effects: []
  - name: "send_command_tcp"
    safety_level: "per_call"
    side_effects: ["network_send"]
  - name: "send_command_udp"
    safety_level: "per_call"
    side_effects: ["network_send"]
  - name: "check_incoming_acks"
    safety_level: "per_call"
    side_effects: ["network_recv"]
  - name: "network_tx_cleanup"
    safety_level: "cleanup"
    side_effects: ["tcp_close", "resource_free"]

constraints:
  - "不得修改订阅的控制指令矢量"
  - "不得向白名单外的IP发送数据"
  - "安全标记为0的指令必须拒绝发送"
  - "TCP连接必须设置keepalive"
  - "重传次数不得超过3次"
  - "ACK超时不得超过2000ms"

aicode_review:
  - "验证TCP帧序列化格式正确性"
  - "验证ACK超时机制可靠性"
  - "验证连接断开检测灵敏度"
  - "验证重传队列内存管理（无泄漏）"
  - "验证白名单校验逻辑完备性"
```

---

## 7. 生产者意图声明书

> 作为网络回传生产者，我的意图是：成为商场指令的**可靠执行者**。我不质疑指令的正确性（那是control_worker和商场安全层的责任），我只确保指令**完整、有序、可确认**地送达外部世界。我的ACK是商场闭环的最后一个环节，没有ACK，闭环就不完整。

---

**蓝图提交者**：网络回传生产者设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
