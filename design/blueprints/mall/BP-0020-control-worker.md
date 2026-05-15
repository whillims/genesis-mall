# 蓝图文档：BP-0007-control_worker

**文档编号**：BP-0007  
**蓝图名称**：闭环控制混合Worker  
**对应场景**：SC-0005  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**闭环控制混合Worker**。它订阅外部遥测数据，执行控制算法（PID、状态机等），生成控制指令并写入SHM。该Worker的产出直接触发物理世界动作，因此必须经过**最严格的安全审核**。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 遥测数据订阅、控制算法执行、控制指令生成、安全校验辅助 |
| **不负责** | 网络数据接收、控制指令网络发送、UI显示、信号分析 |
| **输入** | SHM遥测数据矢量 |
| **输出** | SHM控制指令矢量 `/shm/control/tx_command` |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `control_worker_init()` | 初始化订阅、注册产出权限、加载控制参数 | 商场main()调用一次 |
| `control_worker_loop()` | 主事件循环（等待遥测、执行控制、生成指令） | 独立Worker进程持续运行 |
| `read_telemetry()` | 从SHM读取遥测数据 | 被唤醒后调用 |
| `pid_controller()` | 执行PID控制算法 | 读取遥测后调用 |
| `state_machine()` | 执行状态机逻辑（模式切换、异常处理） | PID输出后调用 |
| `safety_check()` | 对控制指令进行安全校验（参数范围、变化率） | 状态机输出后调用 |
| `pack_control_command()` | 封装控制指令SHM数据包 | 安全校验通过后调用 |
| `mall_shm_write()` | 写入SHM | 封装后调用 |
| `control_worker_cleanup()` | 释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `control_worker_init(config: dict) -> int`

```
参数:
  config["subscribe_vector"]   : string  - 订阅的遥测矢量，默认 "/shm/network/rx_frame"
  config["produce_vector"]     : string  - 产出矢量，默认 "/shm/control/tx_command"
  config["pid_kp"]             : float64 - PID比例系数，默认 1.0
  config["pid_ki"]             : float64 - PID积分系数，默认 0.1
  config["pid_kd"]             : float64 - PID微分系数，默认 0.01
  config["setpoint"]           : float64 - 控制目标值（如目标频率2.4e9）
  config["output_min"]         : float64 - 输出下限（安全边界）
  config["output_max"]         : float64 - 输出上限（安全边界）
  config["max_change_rate"]    : float64 - 最大变化率（每秒最大调整量）
  config["control_period_ms"]  : uint16  - 控制周期（毫秒），默认 100

返回:
  0  : 成功
  -1 : 订阅注册失败
  -2 : 产出权限注册失败（可能因安全审核未通过）
  -3 : 控制参数非法（如 output_min > output_max）

行为:
  1. 调用 mall_subscribe("control_worker_001", config["subscribe_vector"])
  2. 调用 mall_register_producer("control_worker_001", config["produce_vector"], WRITE)
     （此调用需通过商场安全审核层，审核Worker是否有权生成控制指令）
  3. 初始化PID状态（积分项、上一次误差、上一次输出）
     （注意：PID状态存储在 state 字典中，由商场管理，非全局变量）
  4. 加载控制参数至 state
  5. 返回状态码

约束:
  - PID状态必须存储在 state 中，不得使用全局变量
  - 产出权限注册必须通过商场安全审核层
  - 控制参数必须在安全范围内（由AICoder审核）
```

#### `control_worker_loop(state: dict) -> void`

```
参数:
  state : dict - 运行时状态

行为:
  while mall_is_running():
      # 等待遥测数据（带超时，确保控制周期）
      packet = mall_shm_read(state["subscribe_handle"], timeout=state["control_period_ms"])

      if packet is not None:
          # 解析遥测数据
          telemetry = parse_telemetry_packet(packet)
          state["last_telemetry"] = telemetry
      elif state["last_telemetry"] is None:
          # 无历史数据，无法执行控制
          mall_yield()
          continue

      # 使用最新（或缓存的）遥测数据执行控制
      telemetry = state["last_telemetry"]

      # PID计算
      pid_output = pid_controller(telemetry, state)

      # 状态机处理（模式切换、异常检测）
      sm_output = state_machine(pid_output, telemetry, state)

      # 安全校验
      if not safety_check(sm_output, state):
          state["safety_violation_count"] += 1
          mall_yield()
          continue

      # 生成控制指令
      command = pack_control_command(sm_output, telemetry, state)

      # 写入SHM
      mall_shm_write(state["produce_handle"], command)
      state["command_count"] += 1

      # 控制周期同步
      elapsed_ms = mall_get_elapsed_ms(state["last_control_time"])
      if elapsed_ms < state["control_period_ms"]:
          mall_sleep_ms(state["control_period_ms"] - elapsed_ms)
      state["last_control_time"] = mall_get_timestamp_ms()

      mall_yield()

约束:
  - 控制周期必须稳定（默认100ms，允许±10%抖动）
  - 无遥测数据时，使用最后一次有效数据（保持控制连续性）
  - 安全校验失败时，丢弃本次指令，不写入SHM，记录违规计数
  - 每次循环必须调用 mall_yield()
```

#### `pid_controller(telemetry: dict, state: dict) -> float64`

```
参数:
  telemetry : dict   - 解析后的遥测数据
  state     : dict   - 运行时状态（含PID历史状态）

返回:
  float64 : PID控制输出

行为:
  1. 计算误差：error = state["setpoint"] - telemetry["current_value"]
  2. 计算比例项：P = state["pid_kp"] * error
  3. 计算积分项：I = state["pid_ki"] * Σ(error * dt)
     （dt = control_period_ms / 1000.0）
     （积分项限制在 [-I_max, I_max]，防止积分饱和）
  4. 计算微分项：D = state["pid_kd"] * (error - state["last_error"]) / dt
  5. 计算输出：output = P + I + D
  6. 更新状态：
     state["last_error"] = error
     state["integral"] = I
  7. 返回 output

约束:
  - 积分项必须限制范围，防止积分饱和导致系统不稳定
  - 微分项必须处理首次调用（last_error不存在时，D=0）
  - 所有计算使用 double precision
```

#### `state_machine(pid_output: float64, telemetry: dict, state: dict) -> dict`

```
参数:
  pid_output : float64 - PID输出
  telemetry  : dict    - 遥测数据
  state      : dict    - 运行时状态

返回:
  dict : 状态机输出，含 command_type 和 command_param

行为:
  当前状态决定行为：

  状态 "NORMAL":
    - 若 |error| < threshold：保持当前设置
    - 若 |error| >= threshold：转换到 "ADJUSTING"，输出修正指令

  状态 "ADJUSTING":
    - 若 |error| < threshold 且持续3个周期：转换到 "NORMAL"
    - 若 |error| 持续增大：转换到 "EMERGENCY"
    - 输出修正指令

  状态 "EMERGENCY":
    - 输出保守修正（增益降低50%）
    - 若 |error| 减小：转换到 "ADJUSTING"
    - 若 |error| 持续增大超过安全边界：输出紧急停止指令

  状态 "STOPPED":
    - 不输出任何指令
    - 等待外部复位信号

约束:
  - 状态转换必须有明确的条件和日志记录
  - 紧急停止指令优先级最高（priority=9）
  - 状态机不得引入死锁或循环依赖
```

#### `safety_check(command: dict, state: dict) -> bool`

```
参数:
  command : dict - 状态机输出的指令
  state   : dict - 运行时状态

返回:
  bool : True=通过，False=拒绝

行为:
  1. 参数范围检查：
     command["command_param"] 必须在 [state["output_min"], state["output_max"]] 内
  2. 变化率检查：
     |command["command_param"] - state["last_output"]| / dt <= state["max_change_rate"]
  3. 振荡检测：
     若最近5次输出方向交替变化（+ - + - +），标记为振荡，拒绝本次指令
  4. 全部通过 → True，任一失败 → False

约束:
  - 安全校验是Worker的自检，商场的安全三重门是独立校验
  - Worker自检失败不惩罚信用分（属于正常保护机制）
  - 商场安全门拦截则惩罚信用分（属于违规）
```

#### `pack_control_command(sm_output: dict, telemetry: dict, state: dict) -> bytes`

```
行为:
  组装SHM标准数据包：

  偏移    长度    字段
  ─────────────────────────────────────────
  0       32      producer_id     = "control_worker_001\0" * 32
  32      16      product_type    = "control.tx_command\0" * 16
  48      8       timestamp       = mall_get_timestamp_ns()
  56      2       version         = 1

  # 血缘信息
  58      36      parent_vector_id   = mall_get_vector_id(telemetry["raw_packet"])
  94      32      parent_producer_id = extract_producer_id(telemetry["raw_packet"])

  # 控制指令
  126     4       command_id      = state["command_count"] + 1
  130     4       target_ip       = state["target_ip"]
  134     2       target_port     = state["target_port"]
  136     1       command_type    = sm_output["command_type"]
  137     8       command_param   = sm_output["command_param"]
  145     1       priority        = sm_output["priority"]
  146     2       timeout_ms      = state["command_timeout_ms"]

  # 安全字段
  148     64      auth_hash       = sign_with_worker_key(packet_without_hash)
  212     1       safety_check    = 0  # 商场安全层填写

  总长度 = 213 字节

约束:
  - auth_hash 必须使用Worker私钥签名，商场公钥验证
  - command_id 必须单调递增，用于ACK匹配和重传检测
  - target_ip 必须在商场白名单中（由商场安全门校验）
```

---

## 4. SHM接口契约

### 4.1 订阅矢量（消费者端）

| 矢量路径 | 权限 | 说明 |
|----------|------|------|
| `/shm/network/rx_frame` | READ | 外部遥测数据帧 |

### 4.2 产出矢量（生产者端）

| 矢量路径 | 权限 | 容量 | 说明 |
|----------|------|------|------|
| `/shm/control/tx_command` | WRITE | 100条环形缓冲 | 控制指令（带优先级队列） |

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0007-01 | PID稳态跟踪 | setpoint=2.4e9, 初始值2.399e9 | 3个周期内收敛，稳态误差<0.01% |
| T-0007-02 | PID抗干扰 | 稳态时注入+10MHz扰动 | 2个周期内恢复，无超调>5% |
| T-0007-03 | 安全范围拦截 | 指令参数=999GHz | safety_check返回False，不写入SHM |
| T-0007-04 | 振荡检测 | 连续5次反向修正 | 标记振荡，输出保守指令 |
| T-0007-05 | 状态机转换 | 误差从大到小 | NORMAL→ADJUSTING→NORMAL，转换正确 |
| T-0007-06 | 血缘完整性 | 任意遥测帧 | 指令100%包含parent_vector_id |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0007-I01 | SC-0005集成 | 遥测→控制→回传→ACK，闭环延迟<150ms |
| T-0007-I02 | 故障恢复 | 外部执行器断线10秒 | Worker检测到ACK超时，进入保守模式 |
| T-0007-I03 | 权限伪造 | 尝试用analysis_worker签名 | 商场100%拒绝，冻结涉事Worker |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0007"
blueprint_name: "control_worker"
version: "1.0"

worker:
  id: "control_worker_001"
  role: "hybrid"
  control_authority: true  # 特殊标记：有权生成控制指令

shm_permissions:
  consume:
    - vector: "/shm/network/rx_frame"
      mode: "read"
  produce:
    - vector: "/shm/control/tx_command"
      mode: "write"
      ring_buffer_size: 100
      priority_queue: true

functions:
  - name: "control_worker_init"
    safety_level: "init"
    side_effects: ["shm_subscribe", "shm_register", "control_auth_register"]
  - name: "control_worker_loop"
    safety_level: "continuous"
    side_effects: ["shm_read", "shm_write", "cpu_compute"]
    max_cpu_time_per_cycle: "100ms"
  - name: "pid_controller"
    safety_level: "per_call"
    side_effects: ["state_update"]
  - name: "state_machine"
    safety_level: "per_call"
    side_effects: ["state_update"]
  - name: "safety_check"
    safety_level: "per_call"
    side_effects: []
  - name: "pack_control_command"
    safety_level: "pure"
    side_effects: ["shm_write"]

constraints:
  - "不得修改订阅的遥测数据矢量"
  - "产出必须通过商场安全三重门校验"
  - "控制指令必须携带auth_hash签名"
  - "PID积分项必须限制范围，防止饱和"
  - "振荡检测必须触发保守模式"
  - "紧急停止指令优先级必须为9"

aicode_review:
  - "验证PID算法数值稳定性（极限条件测试）"
  - "验证状态机是否存在死锁"
  - "验证安全校验逻辑完备性"
  - "验证auth_hash签名算法安全性"
  - "验证SHM数据包格式是否符合规范"
  - "验证控制周期稳定性（抖动<10%）"
```

---

## 7. Worker意图声明书

> 作为控制Worker，我的意图是：成为商场与物理世界之间的**谨慎桥梁**。我的每一个指令都可能触发真实的动作——电机转动、射频发射、阀门开关。因此，我必须在**效率**与**安全**之间永远偏向安全。我的PID算法追求快速收敛，但我的安全校验确保永不越界。我的价值由**闭环稳定性**和**零安全事故**衡量。

---

**蓝图提交者**：控制Worker设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
