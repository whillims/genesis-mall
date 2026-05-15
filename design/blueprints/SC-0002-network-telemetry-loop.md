# 场景文档：SC-0002-网络遥测回路

**文档编号**：SC-0002  
**场景名称**：网络遥测回路 — 网络生产者 → 数据分析Worker → Console消费者  
**对应蓝图**：BP-0016-network-rx.md, BP-0017-analysis-worker.md, BP-0002-console.md  
**版本**：v1.0  
**日期**：2026-05-10

---

## 1. 场景概述

本场景引入**混合Worker**概念，展示商场的**链式处理能力**。网络生产者接收外部遥测数据，数据分析Worker作为"中间消费者+二次生产者"执行实时频谱分析，Console消费者最终呈现分析结果。

核心验证点：
- Worker的混合角色（既是消费者又是生产者）
- 多阶段SHM路由（网络原始数据 → 分析结果数据）
- 数据血缘追踪（分析结果必须标记其来源）

---

## 2. 参与者清单

| 角色 | 实例 | 职能 | SHM权限 |
|------|------|------|---------|
| 生产者 | network_producer | 监听TCP/UDP端口，接收遥测数据帧 | 写入：`/shm/network/telemetry_raw` |
| 混合Worker | analysis_worker | 订阅原始数据，执行FFT/特征提取，产出分析报告 | 读取：`/shm/network/telemetry_raw`；写入：`/shm/analysis/spectrum_report` |
| 消费者 | console_consumer | 订阅分析报告，可视化显示 | 读取：`/shm/analysis/spectrum_report` |
| 商场 | mall_core | 两段SHM的管理与路由协调 | 管理：全部SHM矢量 |

---

## 3. 数据流架构

```
外部网络
    │
    ▼
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ 网络生产者       │     │   商场SHM（第一段）   │     │ 数据分析Worker   │     │ 商场SHM（第二段）  │
│                 │     │                      │     │                 │     │                  │
│ recv_packet()   │────→│ /shm/network/        │────→│ analyze_fft()   │────→│ /shm/analysis/   │
│   [无状态函数]   │     │ /telemetry_raw         │     │   [无状态函数]   │     │ /spectrum_report │
└─────────────────┘     └──────────────────────┘     └─────────────────┘     └─────────────────┘
                                                                              │
                                                                              ▼
                                                                       ┌─────────────────┐
                                                                       │ Console消费者     │
                                                                       │                 │
                                                                       │ display_report()│
                                                                       │   [无状态函数]   │
                                                                       └─────────────────┘
                                                                              │
                                                                              ▼
                                                                           终端stdout
```

---

## 4. SHM矢量规范

### 4.1 第一段矢量：`/shm/network/telemetry_raw`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"network_prod_001"` |
| `product_type` | string(16) | `"network.telemetry_raw"` |
| `timestamp` | uint64 | 数据包捕获时间戳（纳秒） |
| `version` | uint16 | `1` |
| `src_ip` | uint32 | 源IP地址（IPv4） |
| `src_port` | uint16 | 源端口 |
| `protocol` | uint8 | `1`=TCP, `2`=UDP |
| `payload_len` | uint32 | 数据负载长度 |
| `payload` | byte[] | 原始二进制数据（最大65535字节） |
| `checksum` | uint32 | CRC32校验值 |

### 4.2 第二段矢量：`/shm/analysis/spectrum_report`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"analysis_worker_001"`（Worker作为二次生产者） |
| `product_type` | string(16) | `"analysis.spectrum_report"` |
| `timestamp` | uint64 | 分析完成时间戳 |
| `version` | uint16 | `1` |
| **血缘信息** |||
| `parent_vector_id` | string(36) | 来源数据矢量的UUID（商场自动继承） |
| `parent_producer_id` | string(32) | `"network_prod_001"` |
| **分析结果** |||
| `sample_rate` | float64 | 采样率（Hz） |
| `fft_size` | uint32 | FFT点数 |
| `peak_freq` | float64 | 峰值频率（Hz） |
| `peak_power` | float64 | 峰值功率（dBm） |
| `bandwidth` | float64 | 3dB带宽（Hz） |
| `snr` | float64 | 信噪比（dB） |
| `spectrum_data_len` | uint32 | 频谱数据点数量 |
| `spectrum_data` | float64[] | 功率谱密度数组 |

---

## 5. 运行时序流程

### 阶段A：初始化

```
T0: 商场启动
    ├─→ 创建 `/shm/network/telemetry_raw`（环形缓冲：1000帧）
    ├─→ 创建 `/shm/analysis/spectrum_report`（环形缓冲：100报告）
    └─→ 加载三段授权文件

T0+Δt: 网络生产者初始化
    └─→ 绑定监听端口（TCP:9001/UDP:9002）
    └─→ 进入异步接收循环（非阻塞，epoll/kqueue）

T0+2Δt: 数据分析Worker初始化
    └─→ 商场验证Worker授权（混合角色需双重权限声明）
    ├─→ 注册为 `/shm/network/telemetry_raw` 的消费者
    └─→ 注册为 `/shm/analysis/spectrum_report` 的生产者

T0+3Δt: Console消费者初始化
    └─→ 订阅 `/shm/analysis/spectrum_report`
```

### 阶段B：链式处理（单次遥测帧）

```
T1: 外部设备发送遥测帧（UDP:9002）
    └─→ network_producer.recv_packet() 触发
        ├─→ 读取UDP载荷
        ├─→ 计算CRC32，校验完整性
        ├─→ 组装 telemetry_raw 数据包
        ├─→ mall_shm_write(network_vector, packet)
        └─→ 函数返回

T1+ε: 商场通知 analysis_worker
    └─→ Worker从SHM读取原始帧
    └─→ analyze_fft() 执行
        ├─→ 解析二进制载荷为I/Q采样数据
        ├─→ 执行APFFT（全相位FFT）算法
        ├─→ 提取峰值频率、功率、带宽、SNR
        ├─→ 组装 spectrum_report 数据包
        │   └─→ 血缘字段自动填充（商场API提供 parent_vector_id 查询）
        ├─→ mall_shm_write(analysis_vector, report)
        └─→ 函数返回

T1+2ε: 商场通知 console_consumer
    └─→ display_report() 执行
        ├─→ 读取 spectrum_report
        ├─→ 格式化输出：
        │   [analysis] 2026-05-10T13:31:00.456789
        │   Source: network_prod_001 → analysis_worker_001
        │   Peak: 2.450GHz @ -45.2dBm | BW: 20MHz | SNR: 32dB
        └─→ 函数返回
```

---

## 6. 混合Worker的特殊规则

### 6.1 双重身份声明

Worker的授权文件必须显式声明：

```yaml
worker_role: hybrid
permissions:
  consume:
    - vector: "/shm/network/telemetry_raw"
      mode: "read"
  produce:
    - vector: "/shm/analysis/spectrum_report"
      mode: "write"
constraints:
  # 关键约束：Worker不能修改其消费的数据源
  no_modify_source: true
  # Worker产出必须携带血缘信息
  lineage_required: true
```

### 6.2 商场对Worker的审计

- **输入输出比**：商场监控Worker的读取次数与写入次数，异常比例（如读取100次写入0次）触发审计
- **血缘完整性**：若Worker产出缺少 `parent_vector_id`，商场拒收
- **延迟上限**：若Worker处理延迟超过阈值（如500ms），商场标记Worker为"慢节点"，可能影响调度优先级

---

## 7. 异常处理点

| 异常场景 | 触发条件 | 商场响应 |
|----------|----------|----------|
| **网络丢包** | UDP帧CRC校验失败 | 拒收，通知网络生产者记录丢包计数，不惩罚信用分（网络层问题） |
| **Worker算法崩溃** | FFT计算溢出（异常输入） | 捕获异常，Worker函数返回错误码，商场丢弃本次产出，Worker信用分 -1 |
| **血缘断裂** | Worker产出缺少 `parent_vector_id` | 商场拒收，Worker信用分 -10，连续3次则暂停Worker混合权限 |
| **分析结果溢出** | 原始数据涌入过快，Worker处理不过来 | 商场启用背压机制：通知网络生产者降速（丢包或缓存），保护Worker |
| **Console订阅冲突** | Console同时订阅原始数据和分析报告 | 允许。Console作为纯消费者，可订阅多个矢量，商场独立路由 |

---

## 8. 测试验证标准

| 测试项 | 方法 | 通过标准 |
|--------|------|----------|
| **端到端延迟** | 发送标准测试帧，测量网络→Console时间 | P99 < 200ms（含FFT计算） |
| **血缘追踪** | 随机抽查10份分析报告 | 100%包含有效的 `parent_vector_id`，且能在商场日志中追溯到原始帧 |
| **Worker无状态** | 连续发送1000帧相同数据 | 分析报告结果完全一致，无累积误差 |
| **背压生效** | 以10倍正常速率发送数据 | Worker不崩溃，商场触发背压，网络生产者收到降速信号 |
| **混合权限隔离** | Worker尝试写入 `/shm/network/telemetry_raw` | 商场拒绝，记录安全事件 |

---

## 9. 场景哲学注释

> 本场景是商场的"第一次分工"。它证明：商场不仅连接生产者和消费者，还支持**专业化分工**——网络生产者专注IO，Worker专注计算，Console专注呈现。每个角色只处理自己擅长的环节，通过SHM交换标准化数据。这是**比较优势原理**在软件架构中的实现。

---

**审核状态**：待审核  
**前置依赖**：SC-0001（基础IO回路）  
**下一演进**：SC-0003（仿真验证回路）
