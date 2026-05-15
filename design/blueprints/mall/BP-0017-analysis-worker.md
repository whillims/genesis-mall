# 蓝图文档：BP-0004-analysis_worker

**文档编号**：BP-0004  
**蓝图名称**：数据分析混合Worker  
**对应场景**：SC-0002, SC-0003, SC-0004  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**通用数据分析混合Worker**。它订阅一个或多个原始数据矢量（网络遥测、仿真信号等），执行频谱分析、脉冲参数提取、特征识别等算法，将分析结果写入新的SHM矢量。该Worker是**消费者与生产者的双重身份**，是商场"专业化分工"的核心体现。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 数据订阅、频谱分析（FFT/APFFT）、脉冲参数提取、特征识别、结果封装 |
| **不负责** | 原始数据采集、控制指令生成、UI显示、网络收发 |
| **输入** | SHM原始数据矢量（网络遥测帧或仿真IQ信号） |
| **输出** | SHM分析结果矢量（频谱报告或脉冲报告） |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `analysis_worker_init()` | 初始化订阅、注册产出权限、加载算法参数 | 商场main()调用一次 |
| `analysis_worker_loop()` | 主事件循环（等待SHM通知、调度分析任务） | 独立Worker进程持续运行 |
| `subscribe_and_wait()` | 阻塞等待订阅矢量有新数据 | `analysis_worker_loop()`内调用 |
| `read_raw_data()` | 从SHM读取原始数据包 | 被唤醒后调用 |
| `fft_spectrum()` | 执行FFT频谱分析 | 数据读取后调用 |
| `apfft_analysis()` | 执行全相位FFT（APFFT）高精度分析 | 需要高精度时调用 |
| `extract_pulse_params()` | 提取脉冲参数（PW/PRI/上升沿/下降沿） | 时域分析时调用 |
| `pack_analysis_report()` | 将分析结果封装为SHM标准数据包 | 分析完成后调用 |
| `mall_shm_write()` | 调用商场API写入SHM | 封装后调用 |
| `analysis_worker_cleanup()` | 释放资源、注销订阅 | 商场终止时调用 |

### 3.2 函数详细设计

#### `analysis_worker_init(config: dict) -> int`

```
参数:
  config["subscribe_vectors"]   : list[string]  - 订阅的SHM矢量路径列表
  config["produce_vector"]      : string        - 产出SHM矢量路径
  config["algorithm_mode"]      : string        - "spectrum" | "pulse" | "auto"
  config["fft_size"]            : uint32        - FFT点数，默认 4096
  config["window_type"]         : string        - "hanning" | "hamming" | "blackman" | "rect"
  config["sample_rate"]         : float64       - 采样率（Hz），auto模式从数据包读取
  config["analysis_timeout_ms"] : uint16        - 单次分析超时，默认 500

返回:
  0  : 成功
  -1 : 订阅注册失败
  -2 : 产出权限注册失败
  -3 : 算法参数非法

行为:
  1. 遍历 config["subscribe_vectors"]，调用 mall_subscribe(worker_id, vector_path)
  2. 调用 mall_register_producer(worker_id, config["produce_vector"], WRITE)
  3. 预分配FFT工作缓冲区（避免运行时malloc）
  4. 根据 algorithm_mode 加载对应的算法参数表
  5. 返回状态码

约束:
  - 不得创建全局可变状态
  - FFT工作缓冲区大小 = fft_size * sizeof(complex64)，由商场分配并管理
  - 不得预读取或缓存订阅矢量的历史数据
```

#### `analysis_worker_loop(state: dict) -> void`

```
参数:
  state : dict - 运行时状态，包含订阅句柄列表、产出句柄、算法参数、统计计数器

行为:
  while mall_is_running():
      # 等待任意订阅矢量有新数据（多路复用）
      ready_vector = mall_wait_any(state["subscribe_handles"], timeout=100ms)
      if ready_vector is None:
          mall_yield()
          continue

      # 读取原始数据
      raw_packet = mall_shm_read(ready_vector)
      if raw_packet is None:
          continue

      # 根据数据类型自动选择算法
      data_type = detect_data_type(raw_packet)
      if data_type == "network.telemetry_raw":
          result = analyze_network_frame(raw_packet, state)
      elif data_type == "simulation.iq_signal":
          result = analyze_simulation_signal(raw_packet, state)
      else:
          state["unknown_type_count"] += 1
          continue

      # 写入分析结果
      if result is not None:
          report = pack_analysis_report(result, raw_packet, state)
          mall_shm_write(state["produce_handle"], report)
          state["report_count"] += 1

      mall_yield()

约束:
  - 单次分析必须在 analysis_timeout_ms 内完成，超时则丢弃本次数据
  - 不得阻塞等待超过 100ms（确保Worker可被商场终止）
  - 分析失败不得影响Worker继续处理下一帧
```

#### `analyze_simulation_signal(packet: bytes, state: dict) -> dict`

```
参数:
  packet : bytes - SHM数据包（simulation.iq_signal格式）
  state  : dict  - 运行时状态

返回:
  dict  : 分析结果，字段见下方
  None  : 分析失败

行为:
  1. 解析 packet 中的 iq_data（complex64数组）
  2. 解析 sample_rate, signal_type, carrier_freq 等参数
  3. 根据 signal_type 选择分析路径：

     若 signal_type == 1（脉冲）:
       a. 时域门控检测：计算滑动窗口能量，寻找脉冲起始/结束点
       b. 脉宽测量：PW = end_sample - start_sample / sample_rate
       c. PRI测量：检测连续脉冲间隔（若存在多个脉冲）
       d. 上升沿/下降沿提取：10%-90%幅度点线性插值
       e. 峰值功率：max(|iq_data|^2) 转换为 dBm
       f. 脉顶平坦度：脉冲顶部幅度标准差

     若 signal_type == 2（CW）:
       a. 频谱分析：FFT → 峰值搜索 → 精确频率测量
       b. 功率测量：平均功率

     若 signal_type == 3（LFM）:
       a. 脉冲压缩（匹配滤波）
       b. 调频斜率估计
       c. 压缩后脉宽测量

     通用步骤（所有信号类型）：
       g. 加窗（hanning/hamming/blackman）
       h. FFT计算（可选APFFT提升精度）
       i. 峰值频率、3dB带宽、旁瓣电平测量
       j. SNR估计（信号功率 / 噪声功率）

  4. 组装结果字典：
     {
       "detected_type": signal_type,
       "measured_freq": float,
       "measured_pw": float,
       "measured_pri": float,
       "rise_time": float,
       "fall_time": float,
       "peak_power": float,
       "pulse_top_flatness": float,
       "spectrum_peak_freq": float,
       "spectrum_bandwidth": float,
       "spectrum_sidelobe": float,
       "snr": float,
       "spectrum_data": list[float]  # 功率谱密度数组
     }

  5. 返回结果字典

约束:
  - 所有浮点计算使用 double precision（float64）
  - FFT实现必须支持非2的幂次长度（通过补零或混合基算法）
  - 时域门控阈值自适应：基于噪声底动态调整
  - 若 iq_data 长度 < 16，返回 None（数据不足）
```

#### `pack_analysis_report(result: dict, parent_packet: bytes, state: dict) -> bytes`

```
行为:
  组装SHM标准数据包（二进制序列化，小端序）：

  偏移    长度    字段
  ─────────────────────────────────────────
  0       32      producer_id  = "analysis_worker_001\0" * 32
  32      16      product_type = "analysis.pulse_report\0" * 16
  48      8       timestamp    = mall_get_timestamp_ns()
  56      2       version      = 1

  # 血缘信息（商场自动继承 parent_packet 的 vector_id）
  58      36      parent_vector_id    = mall_get_vector_id(parent_packet)
  94      32      parent_producer_id  = extract_producer_id(parent_packet)

  # 分析结果
  126     1       detected_type       = result["detected_type"]
  127     8       measured_freq       = result["measured_freq"]
  135     8       measured_pw         = result["measured_pw"]
  143     8       measured_pri        = result["measured_pri"]
  151     8       rise_time           = result["rise_time"]
  159     8       fall_time           = result["fall_time"]
  167     8       peak_power          = result["peak_power"]
  175     8       pulse_top_flatness  = result["pulse_top_flatness"]
  183     8       spectrum_peak_freq  = result["spectrum_peak_freq"]
  191     8       spectrum_bandwidth  = result["spectrum_bandwidth"]
  199     8       spectrum_sidelobe   = result["spectrum_sidelobe"]
  207     8       snr                 = result["snr"]
  215     4       spectrum_data_len   = len(result["spectrum_data"])
  219     8*N     spectrum_data       = float64数组

  总长度 = 219 + 8*N 字节

约束:
  - parent_vector_id 必须通过 mall_get_vector_id() 从 parent_packet 提取，不得伪造
  - 所有测量值为 NaN 时，必须写入 IEEE 754 NaN 编码，不得写入 0 或随机值
  - spectrum_data_len 不得超过 65535
```

---

## 4. SHM接口契约

### 4.1 订阅矢量（消费者端）

| 矢量路径 | 权限 | 说明 |
|----------|------|------|
| `/shm/network/telemetry_raw` | READ | 网络遥测原始帧（SC-0002, SC-0004） |
| `/shm/simulation/signal_iq` | READ | 仿真IQ信号（SC-0003, SC-0004） |

### 4.2 产出矢量（生产者端）

| 矢量路径 | 权限 | 容量 | 说明 |
|----------|------|------|------|
| `/shm/analysis/pulse_report` | WRITE | 100条环形缓冲 | 脉冲参数与频谱分析报告 |

### 4.3 商场API依赖

```
mall_subscribe(worker_id: str, vector_path: str) -> handle
mall_wait_any(handles: list, timeout_ms: int) -> handle | None
mall_shm_read(handle) -> bytes | None
mall_shm_write(handle, packet: bytes) -> int
mall_get_timestamp_ns() -> uint64
mall_get_vector_id(packet: bytes) -> string
mall_is_running() -> bool
mall_yield() -> void
```

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0004-01 | 脉冲信号分析 | 标准脉冲IQ（2.4GHz, 1us PW, 1ms PRI） | 频率误差<0.1%, PW误差<1%, PRI误差<0.5% |
| T-0004-02 | CW信号分析 | 标准CW IQ（2.4GHz） | 频率误差<0.01%, 功率测量正确 |
| T-0004-03 | LFM信号分析 | 标准LFM IQ（带宽20MHz） | 调频斜率误差<1%, 压缩后脉宽正确 |
| T-0004-04 | 低SNR信号 | SNR=0dB脉冲 | 检测成功率>99%, 参数误差<5% |
| T-0004-05 | 超短数据 | iq_data长度=8 | 返回None，不崩溃 |
| T-0004-06 | 血缘完整性 | 任意输入 | 产出100%包含有效的parent_vector_id |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0004-I01 | SC-0002集成 | 网络帧→分析报告→Console，端到端<200ms |
| T-0004-I02 | SC-0003集成 | 仿真信号→分析报告，10次固定种子运行结果完全一致 |
| T-0004-I03 | SC-0004集成 | 三源数据同时涌入，Worker不崩溃，产出有序 |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0004"
blueprint_name: "analysis_worker"
version: "1.0"

worker:
  id: "analysis_worker_001"
  role: "hybrid"  # 消费者+生产者双重身份

shm_permissions:
  consume:
    - vector: "/shm/network/telemetry_raw"
      mode: "read"
    - vector: "/shm/simulation/signal_iq"
      mode: "read"
  produce:
    - vector: "/shm/analysis/pulse_report"
      mode: "write"
      max_payload: 219 + 8*65535
      ring_buffer_size: 100

functions:
  - name: "analysis_worker_init"
    safety_level: "init"
    side_effects: ["shm_subscribe", "shm_register"]
  - name: "analysis_worker_loop"
    safety_level: "continuous"
    side_effects: ["shm_read", "shm_write", "cpu_compute"]
    max_cpu_time_per_cycle: "500ms"
  - name: "analyze_simulation_signal"
    safety_level: "per_call"
    side_effects: ["cpu_compute"]
    max_cpu_time: "400ms"
  - name: "pack_analysis_report"
    safety_level: "pure"
    side_effects: ["shm_write"]
  - name: "analysis_worker_cleanup"
    safety_level: "cleanup"
    side_effects: ["shm_unsubscribe", "resource_free"]

constraints:
  - "不得修改订阅的原始数据矢量"
  - "产出必须携带完整的血缘信息（parent_vector_id）"
  - "分析超时必须丢弃数据，不得阻塞"
  - "不得创建持久化的跨调用状态"
  - "FFT实现必须支持任意长度输入"

aicode_review:
  - "验证FFT实现数值精度（与MATLAB/NumPy对比）"
  - "验证APFFT算法正确性（相位不变性测试）"
  - "验证时域门控阈值自适应逻辑"
  - "验证SHM数据包格式是否符合规范"
  - "验证是否存在内存泄漏（预分配缓冲区策略）"
  - "验证分析超时机制是否可靠"
```

---

## 7. Worker意图声明书

> 作为数据分析Worker，我的意图是：成为商场中的**专业分析师**。我不关心数据从何而来（网络、仿真、或其他），只关心数据**承载了什么信息**。我将原始信号转化为人类可理解的参数（频率、功率、脉宽），并将这些参数以标准化格式交付给下游消费者。我的价值由**测量精度**和**分析延迟**衡量。

---

**蓝图提交者**：数据分析Worker设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
