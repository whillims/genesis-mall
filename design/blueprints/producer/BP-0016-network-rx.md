# 蓝图文档：BP-0003-network_rx

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

**文档编号**：BP-0003  
**蓝图名称**：网络遥测生产者（接收端）  
**对应场景**：SC-0002, SC-0004, SC-0005  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**网络遥测数据接收生产者**。它监听指定的TCP/UDP端口，接收外部设备发送的遥测数据帧，将原始帧封装为标准SHM数据包后写入商场SHM。该生产者不解析遥测内容，仅执行**协议层解封装**与**完整性校验**。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 网络Socket监听、数据接收、CRC校验、SHM写入 |
| **不负责** | 遥测数据内容解析、频谱分析、控制决策 |
| **输入** | 外部网络数据包（TCP流或UDP报文） |
| **输出** | SHM矢量 `/shm/network/telemetry_raw` |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `network_rx_init()` | 初始化Socket、绑定端口、注册SHM写入权限 | 商场main()调用一次 |
| `network_rx_loop()` | 主事件循环（非阻塞，epoll/kqueue/select） | 独立Worker进程持续运行 |
| `recv_frame_tcp()` | 从TCP流中读取完整帧（处理粘包） | `network_rx_loop()`内调用 |
| `recv_frame_udp()` | 从UDP报文中读取单帧 | `network_rx_loop()`内调用 |
| `validate_crc32()` | 计算并校验帧CRC32 | 接收后调用 |
| `pack_shm_packet()` | 将原始帧封装为SHM标准数据包 | 校验通过后调用 |
| `mall_shm_write()` | 调用商场API写入SHM（由商场提供） | 封装后调用 |
| `network_rx_cleanup()` | 关闭Socket、释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `network_rx_init(config: dict) -> int`

```
参数:
  config["listen_ip"]      : string  - 监听IP地址，默认 "0.0.0.0"
  config["tcp_port"]       : uint16  - TCP监听端口，默认 9001
  config["udp_port"]       : uint16  - UDP监听端口，默认 9002
  config["max_frame_size"] : uint32  - 最大帧长度，默认 65535
  config["shm_vector"]     : string  - 目标SHM矢量路径，默认 "/shm/network/telemetry_raw"
  config["ring_buffer_size"]: uint32  - 环形缓冲容量，默认 1000

返回:
  0  : 成功
  -1 : Socket创建失败
  -2 : 端口绑定失败
  -3 : SHM注册失败

行为:
  1. 创建TCP Socket（SO_REUSEADDR），绑定tcp_port，监听（backlog=128）
  2. 创建UDP Socket，绑定udp_port
  3. 设置非阻塞模式（fcntl O_NONBLOCK 或 ioctlsocket FIONBIO）
  4. 创建epoll（Linux）或kqueue（BSD/macOS）或select（通用回退）实例
  5. 注册TCP和UDP Socket到事件监听集
  6. 调用 mall_register_producer("network_rx_001", config["shm_vector"], WRITE)
  7. 返回状态码

约束:
  - 不得创建任何全局可变状态
  - 所有运行时状态封装在传入的 config 字典中（由商场管理其生命周期）
  - 不得直接操作除指定SHM矢量外的任何内存区域
```

#### `network_rx_loop(state: dict) -> void`

```
参数:
  state : dict - 由 network_rx_init() 返回并持续传递的状态字典
                 包含：epoll_fd, tcp_fd, udp_fd, shm_handle, 统计计数器等

行为:
  while mall_is_running():
      events = epoll_wait(state["epoll_fd"], timeout=100ms)
      for event in events:
          if event.fd == state["tcp_fd"]:
              # 新TCP连接
              client_fd = accept(state["tcp_fd"])
              set_nonblocking(client_fd)
              epoll_ctl_add(state["epoll_fd"], client_fd, EPOLLIN)
              state["tcp_clients"][client_fd] = {"buffer": b"", "expected_len": 0}
          elif event.fd in state["tcp_clients"]:
              # TCP客户端数据到达
              client = state["tcp_clients"][event.fd]
              data = recv(event.fd, 4096)
              if len(data) == 0:
                  # 连接关闭
                  close(event.fd)
                  epoll_ctl_del(state["epoll_fd"], event.fd)
                  del state["tcp_clients"][event.fd]
                  continue
              client["buffer"] += data
              # 帧解析循环（处理粘包）
              while True:
                  frame, consumed = extract_frame_from_stream(client["buffer"])
                  if frame is None:
                      break
                  client["buffer"] = client["buffer"][consumed:]
                  process_received_frame(state, frame, protocol="TCP")
          elif event.fd == state["udp_fd"]:
              # UDP数据报到达
              data, addr = recvfrom(state["udp_fd"], state["max_frame_size"])
              process_received_frame(state, data, protocol="UDP", src_addr=addr)
      mall_yield()  # 不阻塞主进程

约束:
  - 每次循环必须调用 mall_yield()，确保不独占CPU
  - TCP粘包处理必须基于帧头长度字段，不得假设定长帧
  - UDP报文超过 max_frame_size 的部分必须截断并记录警告
```

#### `process_received_frame(state: dict, raw_frame: bytes, protocol: str, src_addr: tuple = None) -> int`

```
参数:
  state      : dict   - 运行时状态
  raw_frame  : bytes  - 接收到的原始帧数据
  protocol   : string - "TCP" 或 "UDP"
  src_addr   : tuple  - (ip, port)，UDP时提供

返回:
  0  : 成功写入SHM
  -1 : CRC校验失败
  -2 : 帧格式非法
  -3 : SHM写入失败

行为:
  1. 校验帧最小长度（>= 8字节：4字节长度头 + 4字节CRC）
  2. 提取 payload = raw_frame[4:-4]（假设帧格式：| 长度(4B) | 数据(NB) | CRC(4B) |）
  3. 计算CRC32(payload)，与帧尾CRC比对
  4. 若校验失败：记录丢包计数，返回 -1（不惩罚信用分，网络层问题）
  5. 若校验通过：
     packet = pack_shm_packet(payload, protocol, src_addr, state)
     mall_shm_write(state["shm_handle"], packet)
     state["frame_count"] += 1
     state["byte_count"] += len(raw_frame)
  6. 返回状态码
```

#### `pack_shm_packet(payload: bytes, protocol: str, src_addr: tuple, state: dict) -> bytes`

```
行为:
  组装SHM标准数据包（二进制序列化，小端序）：

  偏移    长度    字段
  ─────────────────────────────────────────
  0       32      producer_id  = "network_rx_001\0" * 32
  32      16      product_type = "network.telemetry_raw\0" * 16
  48      8       timestamp    = mall_get_timestamp_ns()  (uint64)
  56      2       version      = 1  (uint16)
  58      4       src_ip       = ip_to_uint32(src_addr[0]) 或 0 (TCP无显式src)
  62      2       src_port     = src_addr[1] 或 0
  64      1       protocol     = 1(TCP) 或 2(UDP)
  65      4       payload_len  = len(payload)
  69      N       payload      = 原始二进制数据
  69+N    4       checksum     = CRC32(payload)

  总长度 = 73 + N 字节

约束:
  - 所有字符串字段必须填充至固定长度，不足补 \0
  - timestamp 必须使用商场提供的 mall_get_timestamp_ns()，不得使用系统time()
  - payload_len 不得超过 65535，否则截断并记录警告
```

---

## 4. SHM接口契约

### 4.1 写入矢量

| 矢量路径 | 权限 | 容量 | 说明 |
|----------|------|------|------|
| `/shm/network/telemetry_raw` | WRITE | 1000条环形缓冲 | 每条最大 73+65535=65608 字节 |

### 4.2 商场API依赖

```
mall_register_producer(producer_id: str, vector_path: str, mode: str) -> handle
mall_shm_write(handle, packet: bytes) -> int
mall_get_timestamp_ns() -> uint64
mall_is_running() -> bool
mall_yield() -> void
```

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0003-01 | TCP单帧接收 | 发送1个合法TCP帧 | SHM写入1条记录，frame_count=1 |
| T-0003-02 | TCP粘包处理 | 发送3个合法帧粘在一起 | SHM写入3条记录，顺序正确 |
| T-0003-03 | UDP单报接收 | 发送1个合法UDP报文 | SHM写入1条记录，src_addr正确 |
| T-0003-04 | CRC校验失败 | 发送CRC错误的帧 | 拒收，丢包计数+1，SHM无写入 |
| T-0003-05 | 超大帧截断 | 发送payload=70000字节的帧 | payload截断至65535，记录警告 |
| T-0003-06 | 并发连接 | 10个TCP客户端同时发送 | 所有帧正确接收，无数据混淆 |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0003-I01 | 与SC-0002集成 | 网络生产者 → 商场 → analysis_worker 端到端延迟 < 200ms |
| T-0003-I02 | 与SC-0004集成 | 三源并行时，网络生产者不被饿死，帧丢失率 < 0.1% |
| T-0003-I03 | 与SC-0005集成 | 网络生产者接收控制ACK帧，正确路由至 control_worker |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0003"
blueprint_name: "network_rx"
version: "1.0"

producer:
  id: "network_rx_001"
  role: "producer"

shm_permissions:
  - vector: "/shm/network/telemetry_raw"
    mode: "write"
    max_payload: 65535
    ring_buffer_size: 1000

functions:
  - name: "network_rx_init"
    safety_level: "init"
    side_effects: ["socket_create", "port_bind", "shm_register"]
  - name: "network_rx_loop"
    safety_level: "continuous"
    side_effects: ["network_recv", "shm_write"]
    max_cpu_time_per_cycle: "100ms"
  - name: "process_received_frame"
    safety_level: "per_call"
    side_effects: ["shm_write"]
  - name: "pack_shm_packet"
    safety_level: "pure"
    side_effects: []
  - name: "network_rx_cleanup"
    safety_level: "cleanup"
    side_effects: ["socket_close", "resource_free"]

constraints:
  - "不得写入除 /shm/network/telemetry_raw 外的任何SHM矢量"
  - "不得解析或修改 payload 内容"
  - "不得建立出站网络连接"
  - "TCP backlog 不得超过 128"
  - "UDP报文超过 max_frame_size 必须截断"

aicode_review:
  - "验证所有socket操作是否设置非阻塞"
  - "验证epoll/select是否正确处理EAGAIN/EWOULDBLOCK"
  - "验证CRC32实现是否与标准IEEE 802.3一致"
  - "验证SHM数据包格式是否符合4.1规范"
  - "验证是否存在缓冲区溢出风险"
```

---

## 7. 生产者意图声明书

> 作为网络遥测生产者，我的意图是：成为外部物理世界与商场之间的**可靠数据闸门**。我不解释数据含义，只确保数据**完整、有序、可追溯**地进入商场。我接受商场的完全调度，我的存在价值由每秒成功交付的帧数衡量。

---

**蓝图提交者**：网络生产者设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
