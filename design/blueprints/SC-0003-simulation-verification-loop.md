# 场景文档：SC-0003-仿真验证回路

**文档编号**：SC-0003  
**场景名称**：仿真验证回路 — 仿真生产者 → 数据分析Worker → Console消费者  
**对应蓝图**：BP-0018-simulation.md, BP-0017-analysis-worker.md, BP-0002-console.md  
**版本**：v1.0  
**日期**：2026-05-10

---

## 1. 场景概述

本场景面向**微波TR测试系统**的核心需求：在没有真实硬件信号的情况下，通过仿真生成标准测试信号（脉冲、CW、调频），由数据分析Worker执行算法验证，Console呈现结果。

与SC-0002的区别：
- 数据源不是外部网络，而是内部仿真引擎
- 数据具有严格的时序和参数可控性（便于验证算法正确性）
- 强调**可重复性**：相同仿真参数必须产生相同分析结果

---

## 2. 参与者清单

| 角色 | 实例 | 职能 | SHM权限 |
|------|------|------|---------|
| 生产者 | simulation_producer | 生成标准测试信号（脉冲/CW/LFM） | 写入：`/shm/simulation/signal_iq` |
| 混合Worker | analysis_worker | 执行脉冲参数提取、频谱分析、APFFT | 读取：`/shm/simulation/signal_iq`；写入：`/shm/analysis/pulse_report` |
| 消费者 | console_consumer | 显示时域波形、频谱图、脉冲参数表 | 读取：`/shm/analysis/pulse_report`, `/shm/simulation/signal_iq` |
| 商场 | mall_core | 协调仿真时钟、管理双矢量路由 | 管理：全部SHM矢量 |

---

## 3. 数据流架构

```
仿真参数输入（键盘/文件/网络）
    │
    ▼
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐     ┌──────────────────────┐
│ 仿真生产者       │     │ 商场SHM（第一段）     │     │ 数据分析Worker   │     │ 商场SHM（第二段）     │
│                 │     │                      │     │                 │     │                      │
│ generate_pulse()│────→│ /shm/simulation/     │────→│ extract_params()│────→│ /shm/analysis/       │
│ generate_cw()   │     │ /signal_iq           │     │ apfft_analysis()│     │ /pulse_report        │
│ generate_lfm()  │     │                      │     │   [无状态函数]   │     │                      │
│   [无状态函数]   │     └──────────────────────┘     └─────────────────┘     └──────────────────────┘
└─────────────────┘                                                              │
                                                                                 ▼
                                                                          ┌─────────────────┐
                                                                          │ Console消费者     │
                                                                          │                 │
                                                                          │ display_waveform()│
                                                                          │ display_spectrum()│
                                                                          │ display_table()   │
                                                                          │   [无状态函数]   │
                                                                          └─────────────────┘
                                                                                 │
                                                                                 ▼
                                                                              终端stdout
```

---

## 4. SHM矢量规范

### 4.1 第一段矢量：`/shm/simulation/signal_iq`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"simulation_prod_001"` |
| `product_type` | string(16) | `"simulation.iq_signal"` |
| `timestamp` | uint64 | 仿真时间戳（纳秒，从仿真启动计时） |
| `version` | uint16 | `1` |
| **信号参数** |||
| `signal_type` | uint8 | `1`=脉冲, `2`=CW, `3`=LFM, `4`=噪声 |
| `carrier_freq` | float64 | 载波频率（Hz） |
| `sample_rate` | float64 | 采样率（Hz） |
| `pulse_width` | float64 | 脉冲宽度（秒），CW为0 |
| `pri` | float64 | 脉冲重复间隔（秒），CW为0 |
| **数据负载** |||
| `sample_count` | uint32 | I/Q采样点数量 |
| `iq_data` | complex64[] | 复数数组（I为实部，Q为虚部） |
| `noise_floor` | float64 | 添加的噪声底（dBm） |

### 4.2 第二段矢量：`/shm/analysis/pulse_report`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"analysis_worker_001"` |
| `product_type` | string(16) | `"analysis.pulse_report"` |
| `timestamp` | uint64 | 分析完成时间戳 |
| `version` | uint16 | `1` |
| **血缘信息** |||
| `parent_vector_id` | string(36) | 来源仿真信号矢量的UUID |
| `parent_producer_id` | string(32) | `"simulation_prod_001"` |
| **脉冲参数提取结果** |||
| `detected_type` | uint8 | 检测到的信号类型 |
| `measured_freq` | float64 | 测量载频（Hz） |
| `measured_pw` | float64 | 测量脉宽（秒） |
| `measured_pri` | float64 | 测量PRI（秒） |
| `rise_time` | float64 | 上升沿时间（秒） |
| `fall_time` | float64 | 下降沿时间（秒） |
| `peak_power` | float64 | 峰值功率（dBm） |
| `pulse_top_flatness` | float64 | 脉顶平坦度（dB） |
| **频谱分析结果** |||
| `spectrum_peak_freq` | float64 | 频谱峰值频率 |
| `spectrum_bandwidth` | float64 | 3dB带宽 |
| `spectrum_sidelobe` | float64 | 第一旁瓣电平（dB） |

---

## 5. 运行时序流程

### 阶段A：仿真参数配置

```
T0: 用户通过键盘输入仿真参数（或从文件加载）
    └─→ 参数示例：
        signal_type = 1（脉冲）
        carrier_freq = 2.4e9
        sample_rate = 1e9
        pulse_width = 1e-6
        pri = 1e-3
        noise_floor = -80

T0+Δt: 商场接收参数，写入 `/shm/config/simulation_params`
    └─→ 仿真生产者读取参数，准备生成信号
```

### 阶段B：信号生成与分析（单次脉冲）

```
T1: simulation_producer.generate_pulse() 执行
    ├─→ 根据参数生成理想脉冲波形
    ├─→ 叠加高斯噪声（noise_floor = -80dBm）
    ├─→ 组装 signal_iq 数据包
    ├─→ mall_shm_write(simulation_vector, packet)
    └─→ 函数返回

T1+ε: 商场通知 analysis_worker
    └─→ Worker执行分析流水线
        ├─→ extract_params(): 
        │   时域门控检测 → 脉宽/PRI测量 → 上升/下降沿提取
        ├─→ apfft_analysis():
        │   加窗 → APFFT → 峰值搜索 → 旁瓣测量
        ├─→ 组装 pulse_report
        ├─→ mall_shm_write(analysis_vector, report)
        └─→ 函数返回

T1+2ε: 商场通知 console_consumer
    └─→ Console显示三层信息：
        [SIMULATION] 2026-05-10T13:32:00.123456
        Signal: Pulse @ 2.400GHz, PW=1.000us, PRI=1.000ms
        ───────────────────────────────────────
        [ANALYSIS]  Worker: analysis_worker_001
        Detected:  Pulse @ 2.400GHz (err: +0.00MHz)
        PW: 1.002us (err: +0.2%) | PRI: 1.001ms (err: +0.1%)
        Power: -45.3dBm | Flatness: 0.05dB
        Spectrum: BW=20.1MHz | Sidelobe=-13.2dB
        ───────────────────────────────────────
        [VERDICT]  PASS (所有误差 < 1%)
```

---

## 6. 可重复性机制

### 6.1 仿真时钟与商场时钟分离

```
仿真时钟（Simulation Time）：
  - 由 simulation_producer 维护
  - 从0开始，按 `1/sample_rate` 步进
  - 写入数据包的 `timestamp` 字段

商场时钟（Mall Time）：
  - 由 mall_core 维护
  - 真实UNIX时间戳（纳秒）
  - 用于路由延迟测量、日志记录
```

### 6.2 确定性验证

| 测试模式 | 机制 | 目的 |
|----------|------|------|
| **固定种子模式** | 仿真生产者使用固定随机种子生成噪声 | 确保多次运行产生完全相同的IQ数据 |
| **参数冻结模式** | 商场锁定 `/shm/config/simulation_params`，禁止运行时修改 | 防止参数漂移导致结果不可比 |
| **Worker版本锁定** | 分析Worker必须使用特定版本的授权函数 | 防止算法变更引入分析误差 |

---

## 7. 异常处理点

| 异常场景 | 触发条件 | 商场响应 |
|----------|----------|----------|
| **参数违规** | pulse_width > pri（物理不可能） | 商场在配置阶段拒绝参数，仿真生产者不启动 |
| **采样率不足** | sample_rate < 2*carrier_freq（奈奎斯特违规） | 仿真生产者生成警告标记，数据仍进入SHM，Worker分析时标注"欠采样" |
| **Worker检测失败** | 信噪比过低，无法检测脉冲 | Worker返回空报告（字段为NaN），Console显示"检测失败"，不崩溃 |
| **仿真-分析时序失步** | 仿真生产者速度 > Worker处理速度 | 商场启用背压，或丢弃旧仿真帧（依配置策略） |
| **Console双订阅冲突** | Console同时显示原始IQ和报告 | 允许。Console可自由组合多个订阅源，商场保证数据一致性 |

---

## 8. 测试验证标准

| 测试项 | 方法 | 通过标准 |
|--------|------|----------|
| **参数测量精度** | 已知参数仿真信号，测量误差 | 频率误差 < 0.1%，脉宽误差 < 1%，PRI误差 < 0.5% |
| **可重复性** | 固定种子运行10次 | 10次IQ数据逐点一致，10次分析报告字段完全一致 |
| **算法鲁棒性** | SNR从-10dB到+60dB扫描 | 在SNR>0dB时检测成功率>99%，不崩溃 |
| **背压保护** | 仿真以10倍速率运行 | Worker不溢出，商场背压机制生效 |
| **血缘完整性** | 抽查100份报告 | 100%可追溯至原始仿真帧，参数匹配 |

---

## 9. 场景哲学注释

> 本场景是商场的"实验室"。仿真信号是**已知输入**，分析结果是**可验证输出**，Console是**可视化裁判**。这个闭环构成了商场的**自洽验证单元**：任何新算法、新Worker、新生产者，都必须先通过仿真场景的"闭卷考试"，才能接入真实数据。仿真不是"假数据"，而是**信任的基石**。

---

**审核状态**：待审核  
**前置依赖**：SC-0002（网络遥测回路）  
**下一演进**：SC-0004（多源融合回路）
