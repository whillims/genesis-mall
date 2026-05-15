# 蓝图文档：BP-0005-simulation

**文档编号**：BP-0005  
**蓝图名称**：仿真信号生产者  
**对应场景**：SC-0003, SC-0004  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**标准信号仿真生产者**。它根据配置参数生成精确的测试信号（脉冲、CW、LFM等），以IQ复数数据形式写入商场SHM。仿真信号是**已知输入**，用于验证下游Worker算法的正确性，是商场"自洽验证单元"的核心组件。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 根据参数生成标准信号、叠加可控噪声、写入SHM |
| **不负责** | 信号分析、控制决策、网络收发、UI显示 |
| **输入** | 仿真参数（频率、脉宽、PRI、采样率、信号类型、噪声底） |
| **输出** | SHM矢量 `/shm/simulation/signal_iq` |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `simulation_init()` | 初始化SHM写入权限、加载默认参数 | 商场main()调用一次 |
| `simulation_loop()` | 主循环（读取参数、生成信号、写入SHM） | 独立Worker进程持续运行 |
| `read_simulation_params()` | 从SHM配置矢量读取最新参数 | 每次生成前调用 |
| `generate_pulse()` | 生成脉冲信号IQ数据 | 参数signal_type=1时调用 |
| `generate_cw()` | 生成CW连续波IQ数据 | 参数signal_type=2时调用 |
| `generate_lfm()` | 生成线性调频信号IQ数据 | 参数signal_type=3时调用 |
| `add_noise()` | 叠加高斯白噪声 | 信号生成后调用 |
| `pack_simulation_packet()` | 封装SHM标准数据包 | 噪声叠加后调用 |
| `mall_shm_write()` | 写入SHM | 封装后调用 |
| `simulation_cleanup()` | 释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `simulation_init(config: dict) -> int`

```
参数:
  config["shm_vector"]       : string  - 目标SHM矢量，默认 "/shm/simulation/signal_iq"
  config["config_vector"]    : string  - 参数配置矢量，默认 "/shm/config/simulation_params"
  config["default_params"]   : dict    - 默认仿真参数
  config["seed"]             : uint32  - 随机数种子，默认 42（固定种子确保可重复性）

返回:
  0  : 成功
  -1 : SHM注册失败
  -2 : 配置矢量订阅失败

行为:
  1. 调用 mall_register_producer("simulation_prod_001", config["shm_vector"], WRITE)
  2. 调用 mall_subscribe("simulation_prod_001", config["config_vector"])
  3. 初始化随机数生成器（config["seed"]）
  4. 预分配IQ数据缓冲区（最大支持 10M 采样点）
  5. 返回状态码

约束:
  - 随机数生成器必须使用确定性的算法（如MT19937），确保相同种子产生相同序列
  - 不得创建全局可变状态
```

#### `simulation_loop(state: dict) -> void`

```
参数:
  state : dict - 运行时状态

行为:
  while mall_is_running():
      # 读取最新参数（非阻塞，使用缓存值若未更新）
      new_params = read_simulation_params(state)
      if new_params is not None:
          state["current_params"] = new_params

      params = state["current_params"]

      # 根据信号类型生成数据
      if params["signal_type"] == 1:
          iq_data = generate_pulse(params, state)
      elif params["signal_type"] == 2:
          iq_data = generate_cw(params, state)
      elif params["signal_type"] == 3:
          iq_data = generate_lfm(params, state)
      else:
          state["error_count"] += 1
          mall_yield()
          continue

      # 叠加噪声
      if params["noise_floor"] > -999:
          iq_data = add_noise(iq_data, params["noise_floor"], params["sample_rate"], state)

      # 封装并写入
      packet = pack_simulation_packet(iq_data, params, state)
      mall_shm_write(state["shm_handle"], packet)
      state["frame_count"] += 1

      # 仿真时钟推进
      state["sim_time_ns"] += int(len(iq_data) / params["sample_rate"] * 1e9)

      # 根据PRI决定下次生成时间（脉冲信号）
      if params["signal_type"] == 1 and params["pri"] > 0:
          wait_ns = int(params["pri"] * 1e9) - int(len(iq_data) / params["sample_rate"] * 1e9)
          if wait_ns > 0:
              mall_sleep_ns(wait_ns)  # 商场提供的非阻塞睡眠

      mall_yield()

约束:
  - 每次循环必须调用 mall_yield()
  - 仿真时钟与商场真实时钟分离，由生产者自行维护
  - 参数变更必须立即生效（下一帧使用新参数）
```

#### `generate_pulse(params: dict, state: dict) -> list[complex64]`

```
参数:
  params["carrier_freq"] : float64 - 载波频率（Hz）
  params["sample_rate"]  : float64 - 采样率（Hz）
  params["pulse_width"]  : float64 - 脉冲宽度（秒）
  params["pri"]          : float64 - 脉冲重复间隔（秒），0表示单脉冲
  params["peak_power"]   : float64 - 峰值功率（dBm）

返回:
  list[complex64] : IQ采样数据

行为:
  1. 计算采样点数：N = int(params["pulse_width"] * params["sample_rate"])
  2. 生成时间轴：t[n] = n / params["sample_rate"], n = 0, 1, ..., N-1
  3. 生成载波：carrier[n] = exp(j * 2π * params["carrier_freq"] * t[n])
  4. 生成包络（矩形脉冲）：envelope[n] = 1.0（0 <= n < N）
  5. 计算幅度：A = sqrt(10^((params["peak_power"] - 30) / 10) * 2 * 50) / sqrt(2)
     （假设50Ω系统，将dBm转换为电压幅度）
  6. IQ数据：iq[n] = A * envelope[n] * carrier[n]
  7. 返回 iq 数组

约束:
  - 载波频率必须满足奈奎斯特准则：carrier_freq < sample_rate / 2
  - 若违反，生成警告标记但不阻止数据生成
  - 相位从0开始，确保可重复性
```

#### `generate_cw(params: dict, state: dict) -> list[complex64]`

```
行为:
  1. 计算采样点数：N = int(params["duration"] * params["sample_rate"])，若duration未指定则使用默认值0.001秒
  2. 生成时间轴和载波（同generate_pulse）
  3. 包络为常数1.0（无脉冲调制）
  4. 幅度计算同generate_pulse
  5. 返回IQ数据

约束:
  - CW信号无脉宽/PRI概念，params["pulse_width"] 和 params["pri"] 被忽略
```

#### `generate_lfm(params: dict, state: dict) -> list[complex64]`

```
参数:
  params["carrier_freq"] : float64 - 起始频率（Hz）
  params["bandwidth"]    : float64 - 调频带宽（Hz）
  params["pulse_width"]  : float64 - 脉冲宽度（秒）

行为:
  1. 计算采样点数：N = int(params["pulse_width"] * params["sample_rate"])
  2. 生成时间轴：t[n] = n / params["sample_rate"]
  3. 瞬时频率：f[n] = params["carrier_freq"] + (params["bandwidth"] / params["pulse_width"]) * t[n]
  4. 相位积分：φ[n] = 2π * Σ(f[k] * Δt), k=0..n
  5. IQ数据：iq[n] = A * exp(j * φ[n])
  6. 返回IQ数据

约束:
  - 终止频率 = carrier_freq + bandwidth，必须 < sample_rate / 2
  - 相位积分使用梯形法或解析积分，确保数值精度
```

#### `add_noise(iq_data: list[complex64], noise_floor_dbm: float64, sample_rate: float64, state: dict) -> list[complex64]`

```
行为:
  1. 将噪声底从dBm转换为功率：P_noise = 10^((noise_floor_dbm - 30) / 10)
  2. 计算噪声方差：σ² = P_noise * 50 / 2  （假设50Ω，IQ两路均分功率）
  3. 生成高斯白噪声：
     I_noise[n] ~ N(0, σ)
     Q_noise[n] ~ N(0, σ)
  4. 叠加：iq_noisy[n] = iq_data[n] + (I_noise[n] + j * Q_noise[n])
  5. 返回 iq_noisy

约束:
  - 随机数生成器必须使用 state["rng"]（由初始化时的种子确定）
  - 确保相同种子产生完全相同的噪声序列（可重复性）
  - noise_floor_dbm = -999 表示不添加噪声（纯净信号）
```

#### `pack_simulation_packet(iq_data: list[complex64], params: dict, state: dict) -> bytes`

```
行为:
  组装SHM标准数据包（二进制序列化，小端序）：

  偏移    长度    字段
  ─────────────────────────────────────────
  0       32      producer_id   = "simulation_prod_001\0" * 32
  32      16      product_type  = "simulation.iq_signal\0" * 16
  48      8       timestamp     = state["sim_time_ns"]  (uint64，仿真时钟)
  56      2       version       = 1
  58      1       signal_type   = params["signal_type"]
  59      8       carrier_freq  = params["carrier_freq"]
  67      8       sample_rate   = params["sample_rate"]
  75      8       pulse_width   = params["pulse_width"]
  83      8       pri           = params["pri"]
  91      4       sample_count  = len(iq_data)
  95      8*N     iq_data       = complex64数组（交错I/Q或复数结构）
  95+8*N  8       noise_floor   = params["noise_floor"]

  总长度 = 103 + 8*N 字节

约束:
  - timestamp 使用仿真时钟，非商场真实时钟
  - iq_data 的 I/Q 分量使用 float32（complex64 = float32 I + float32 Q）
  - sample_count 不得超过 10M（预分配缓冲区上限）
```

---

## 4. SHM接口契约

### 4.1 写入矢量

| 矢量路径 | 权限 | 容量 | 说明 |
|----------|------|------|------|
| `/shm/simulation/signal_iq` | WRITE | 100条环形缓冲 | 每条最大 103+8*10M 字节 |

### 4.2 订阅矢量（参数输入）

| 矢量路径 | 权限 | 说明 |
|----------|------|------|
| `/shm/config/simulation_params` | READ | 仿真参数配置（由键盘生产者或文件加载器写入） |

### 4.3 商场API依赖

```
mall_register_producer(producer_id: str, vector_path: str, mode: str) -> handle
mall_subscribe(producer_id: str, vector_path: str) -> handle
mall_shm_read(handle) -> bytes | None
mall_shm_write(handle, packet: bytes) -> int
mall_sleep_ns(ns: uint64) -> void
mall_is_running() -> bool
mall_yield() -> void
```

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0005-01 | 脉冲生成 | carrier=2.4GHz, PW=1us, SR=1GHz | 1000个采样点，峰值功率正确 |
| T-0005-02 | CW生成 | carrier=2.4GHz, duration=1ms | 1M个采样点，恒定包络 |
| T-0005-03 | LFM生成 | carrier=2.4GHz, BW=20MHz, PW=10us | 频率线性扫描，终止频率=2.42GHz |
| T-0005-04 | 噪声可重复 | 固定种子，相同参数运行2次 | 两次IQ数据逐点一致 |
| T-0005-05 | 参数变更 | 运行中修改carrier_freq | 下一帧立即使用新频率 |
| T-0005-06 | 奈奎斯特违规 | carrier=600MHz, SR=1GHz | 生成警告标记，数据仍生成 |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0005-I01 | SC-0003集成 | 仿真→analysis_worker→Console，参数测量误差<1% |
| T-0005-I02 | SC-0004集成 | 三源并行，仿真生产者不被饿死，帧间隔稳定 |
| T-0005-I03 | 可重复性验证 | 固定种子运行10次，10次结果逐点一致 |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0005"
blueprint_name: "simulation"
version: "1.0"

producer:
  id: "simulation_prod_001"
  role: "producer"

shm_permissions:
  write:
    - vector: "/shm/simulation/signal_iq"
      mode: "write"
      max_payload: 103 + 8*10000000
      ring_buffer_size: 100
  read:
    - vector: "/shm/config/simulation_params"
      mode: "read"

functions:
  - name: "simulation_init"
    safety_level: "init"
    side_effects: ["shm_register", "shm_subscribe", "rng_init"]
  - name: "simulation_loop"
    safety_level: "continuous"
    side_effects: ["shm_read", "shm_write", "cpu_compute"]
    max_cpu_time_per_cycle: "1000ms"
  - name: "generate_pulse"
    safety_level: "pure"
    side_effects: ["cpu_compute"]
  - name: "generate_cw"
    safety_level: "pure"
    side_effects: ["cpu_compute"]
  - name: "generate_lfm"
    safety_level: "pure"
    side_effects: ["cpu_compute"]
  - name: "add_noise"
    safety_level: "pure"
    side_effects: ["rng_advance"]
  - name: "pack_simulation_packet"
    safety_level: "pure"
    side_effects: ["shm_write"]
  - name: "simulation_cleanup"
    safety_level: "cleanup"
    side_effects: ["resource_free"]

constraints:
  - "不得写入除 /shm/simulation/signal_iq 外的任何SHM矢量"
  - "随机数种子必须固定且可配置，确保可重复性"
  - "仿真时钟与商场真实时钟分离，由生产者自行维护"
  - "奈奎斯特违规必须生成警告但不阻止数据生成"
  - "参数变更必须立即生效（下一帧）"

aicode_review:
  - "验证信号生成数学公式正确性（与理论公式对比）"
  - "验证噪声功率计算是否正确（dBm→电压转换）"
  - "验证随机数生成器确定性（相同种子相同输出）"
  - "验证SHM数据包格式是否符合规范"
  - "验证大采样点数（10M）下内存使用是否可控"
```

---

## 7. 生产者意图声明书

> 作为仿真生产者，我的意图是：成为商场的**标准信号源**。我生成的不是"假数据"，而是**可验证的真理**。每一个IQ采样点都严格遵循物理定律，每一个噪声样本都可追溯至确定的种子。下游Worker用我的信号证明自己的能力，我用我的精确性证明商场的可信度。

---

**蓝图提交者**：仿真生产者设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
