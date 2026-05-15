# 场景文档：SC-0004-多源融合回路

**文档编号**：SC-0004  
**场景名称**：多源融合回路 — 键盘+网络+仿真 → 数据分析Worker → Console消费者  
**对应蓝图**：BP-0001-keyboard.md, BP-0016-network-rx.md, BP-0018-simulation.md, BP-0019-fusion-worker.md, BP-0002-console.md  
**版本**：v1.0  
**日期**：2026-05-10

---

## 1. 场景概述

本场景是商场的**压力测试与能力展示**。三个异构生产者（键盘事件、网络遥测、仿真信号）同时运行，数据分析Worker作为**多源融合节点**，执行跨源关联分析，Console消费者统一呈现。

核心验证点：
- 商场的**多对多路由**能力
- 不同频率/格式数据的**共存与隔离**
- Worker的**多源订阅与融合计算**
- 商场的**公平调度**（无生产者独占资源）

---

## 2. 参与者清单

| 角色 | 实例 | 职能 | SHM权限 |
|------|------|------|---------|
| 生产者A | keyboard_producer | 用户交互输入（控制指令） | 写入：`/shm/keyboard/control_cmd` |
| 生产者B | network_producer | 外部设备遥测数据 | 写入：`/shm/network/telemetry_raw` |
| 生产者C | simulation_producer | 内部标准测试信号 | 写入：`/shm/simulation/signal_iq` |
| 混合Worker | fusion_analysis_worker | 订阅三源数据，执行关联分析、融合判决 | 读取：上述三个矢量；写入：`/shm/fusion/decision_report` |
| 消费者 | console_consumer | 订阅所有中间及最终数据，分层显示 | 读取：全部五个矢量 |
| 商场 | mall_core | 五矢量管理、三生产者公平调度、全链路血缘追踪 | 管理：全部SHM矢量 |

---

## 3. 数据流架构

```
                    ┌─────────────────┐
       物理键盘 ───→│ 键盘生产者       │────→ /shm/keyboard/control_cmd
                    └─────────────────┘
                    ┌─────────────────┐
       外部网络 ───→│ 网络生产者       │────→ /shm/network/telemetry_raw
                    └─────────────────┘
                    ┌─────────────────┐
       仿真引擎 ───→│ 仿真生产者       │────→ /shm/simulation/signal_iq
                    └─────────────────┘
                           │
                           ▼
                    ┌─────────────────┐
                    │ 商场SHM路由层    │
                    │  （五矢量并行）   │
                    └─────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ 键盘数据     │ │ 网络数据     │ │ 仿真数据     │
    │ 消费者可直阅 │ │ 消费者可直阅 │ │ 消费者可直阅 │
    └─────────────┘ └─────────────┘ └─────────────┘
           │               │               │
           └───────────────┼───────────────┘
                           ▼
                    ┌─────────────────┐
                    │ fusion_analysis  │
                    │ _worker          │
                    │                 │
                    │ 关联分析算法      │
                    │ - 时间对齐        │
                    │ - 源置信度加权    │
                    │ - 冲突检测        │
                    │ - 融合判决        │
                    └─────────────────┘
                           │
                           ▼
                    /shm/fusion/decision_report
                           │
                           ▼
                    ┌─────────────────┐
                    │ Console消费者     │
                    │                 │
                    │ 分层显示：       │
                    │ - 实时原始数据流  │
                    │ - 融合分析结果    │
                    │ - 系统状态看板    │
                    └─────────────────┘
                           │
                           ▼
                        终端stdout
```

---

## 4. SHM矢量规范（新增）

### 4.1 控制指令矢量：`/shm/keyboard/control_cmd`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"keyboard_prod_001"` |
| `product_type` | string(16) | `"keyboard.control_cmd"` |
| `timestamp` | uint64 | 按键时间戳 |
| `version` | uint16 | `1` |
| `cmd_type` | uint8 | `1`=启动仿真, `2`=停止仿真, `3`=切换模式, `4`=紧急停止 |
| `cmd_param` | string(64) | 命令参数（如模式名称） |
| `priority` | uint8 | `0`=普通, `1`=高优先级, `9`=紧急 |

### 4.2 融合报告矢量：`/shm/fusion/decision_report`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"fusion_worker_001"` |
| `product_type` | string(16) | `"fusion.decision_report"` |
| `timestamp` | uint64 | 融合判决时间戳 |
| `version` | uint16 | `1` |
| **多源血缘** |||
| `parent_count` | uint8 | 参与融合的来源数量（1-3） |
| `parent_vector_ids` | string(36)[] | 来源矢量UUID数组 |
| `parent_producer_ids` | string(32)[] | 来源生产者ID数组 |
| **融合结果** |||
| `decision_type` | uint8 | `1`=单一源确认, `2`=多源一致, `3`=多源冲突, `4`=源缺失 |
| `confidence` | float64 | 融合置信度（0.0-1.0） |
| `primary_source` | string(32) | 主导来源ID |
| `conflict_details` | string(256) | 冲突描述（若存在） |
| `recommendation` | string(128) | 系统建议动作 |
| **时间对齐信息** |||
| `time_window_start` | uint64 | 融合时间窗口起始 |
| `time_window_end` | uint64 | 融合时间窗口结束 |

---

## 5. 运行时序流程

### 阶段A：多源并行启动

```
T0: 商场启动，创建五矢量
T0+Δt: 三生产者并行初始化（商场分配独立Worker进程）
T0+2Δt: fusion_worker初始化
    ├─→ 注册三重订阅：
    │   subscribe("/shm/keyboard/control_cmd", priority=9)
    │   subscribe("/shm/network/telemetry_raw", priority=5)
    │   subscribe("/shm/simulation/signal_iq", priority=5)
    └─→ 注册产出权限：`/shm/fusion/decision_report`
T0+3Δt: Console消费者初始化
    └─→ 注册五重订阅（全部矢量）
```

### 阶段B：正常运行（多源交织）

```
T1: 网络生产者收到遥测帧 → 写入 /shm/network/telemetry_raw
    └─→ 商场通知：fusion_worker, console_consumer

T1+10ms: 仿真生产者生成脉冲信号 → 写入 /shm/simulation/signal_iq
    └─→ 商场通知：fusion_worker, console_consumer

T1+50ms: 用户按 'M' 键（切换模式命令）→ 写入 /shm/keyboard/control_cmd
    └─→ 商场通知：fusion_worker（高优先级插队）, console_consumer

T1+52ms: fusion_worker 被高优先级唤醒（控制指令）
    ├─→ 读取 control_cmd：cmd_type=3（切换模式）
    ├─→ 立即执行模式切换逻辑
    ├─→ 生成 decision_report：
    │   decision_type=1（单一源确认）
    │   confidence=1.0
    │   recommendation="Switch to mode: advanced_analysis"
    └─→ mall_shm_write(fusion_vector, report)

T1+55ms: fusion_worker 继续处理网络/仿真数据（低优先级队列）
    ├─→ 时间对齐：匹配时间窗口内（T1 ~ T1+10ms）的网络帧与仿真帧
    ├─→ 关联分析：网络遥测频率 vs 仿真载频
    ├─→ 判决：若差异 < 1%，标记为多源一致；否则标记冲突
    └─→ 生成融合报告，写入SHM

T1+60ms: Console消费者刷新显示
    ├─→ 原始数据面板：网络帧摘要 | 仿真波形缩略图 | 最新按键
    ├─→ 融合分析面板：置信度仪表盘 | 来源占比饼图 | 冲突警报
    └─→ 系统状态面板：当前模式 | 三生产者在线状态 | 商场负载
```

---

## 6. 公平调度机制

### 6.1 生产者防饥饿策略

```
商场调度器（Mall Scheduler）：

每个生产者分配 "信用额度"（Credit）：
  - 初始值：1000
  - 每次成功写入：-1（消耗信用）
  - 每毫秒：+10（恢复信用）

调度规则：
  1. 高信用生产者优先获得SHM写入槽位
  2. 若生产者信用 < 0，商场延迟其写入请求，但不拒绝
  3. 紧急控制指令（priority=9）绕过信用机制，直接插队

目的：防止高频生产者（如仿真1GHz采样）垄断SHM带宽，
      确保低频但关键的生产者（如键盘控制）不被饿死。
```

### 6.2 Worker多源优先级

```
fusion_worker 内部优先级队列：

Queue 0（紧急）：keyboard.control_cmd（系统控制指令）
Queue 1（高优）：network.telemetry_raw（外部实时数据）
Queue 2（普通）：simulation.signal_iq（内部测试数据）

Worker每次从Queue 0开始处理，确保控制指令不被数据淹没。
```

---

## 7. 异常处理点

| 异常场景 | 触发条件 | 商场响应 |
|----------|----------|----------|
| **生产者崩溃** | network_producer 进程退出 | 商场检测到订阅者失效，标记该矢量"源离线"，fusion_worker收到源缺失标记，Console显示红色警报 |
| **Worker融合冲突** | 网络帧与仿真帧频率差异 > 10% | fusion_worker标记 `decision_type=3`（冲突），商场记录冲突日志，Console显示冲突详情 |
| **紧急指令风暴** | 用户连续按键（如按住Ctrl+C） | 商场启用指令去抖：500ms内相同指令合并，防止Worker过载 |
| **SHM带宽耗尽** | 三生产者总写入速率 > SHM物理带宽 | 商场触发全局背压：按信用比例降速，优先保障控制指令通道 |
| **Console显示滞后** | Console处理速度 < 数据产生速度 | 商场启用订阅者采样模式：Console可选择"仅显示最新数据"或"显示关键帧" |

---

## 8. 测试验证标准

| 测试项 | 方法 | 通过标准 |
|--------|------|----------|
| **三源共存** | 同时运行三生产者30分钟 | 商场不崩溃，SHM无泄漏，数据不混淆 |
| **公平调度** | 仿真以1000倍速率运行，键盘每10秒一次 | 键盘指令延迟 < 100ms，不被饿死 |
| **融合正确性** | 注入已知关联数据（网络帧与仿真帧匹配） | fusion_worker 100%判定"多源一致" |
| **冲突检测** | 注入矛盾数据（频率差异20%） | fusion_worker 100%判定"多源冲突"，置信度 < 0.5 |
| **故障恢复** | 中途杀死 network_producer，10秒后重启 | 商场自动恢复订阅，Worker标记源缺失→源恢复，Console显示正确状态迁移 |
| **紧急指令** | 仿真满负荷时发送紧急停止指令 | 指令延迟 < 50ms，商场立即响应 |

---

## 9. 场景哲学注释

> 本场景是商场的"联合国"。三个生产者说着不同的"语言"（键盘事件、网络帧、IQ信号），商场是"翻译官"和"调度员"，Worker是"分析师"，Console是"新闻发言人"。关键洞察：**多样性不是混乱的根源，而是鲁棒性的来源**。当网络生产者崩溃时，仿真生产者仍在运行；当仿真信号失真时，网络遥测提供外部验证。商场的使命不是消除多样性，而是**让多样性有序协作**。

---

**审核状态**：待审核  
**前置依赖**：SC-0003（仿真验证回路）  
**下一演进**：SC-0005（闭环控制回路）
