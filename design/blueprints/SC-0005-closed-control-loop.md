# 场景文档：SC-0005-闭环控制回路

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

**文档编号**：SC-0005  
**场景名称**：闭环控制回路 — 网络生产者 → 数据分析Worker → 网络回传生产者  
**对应蓝图**：BP-0016-network-rx.md, BP-0020-control-worker.md, BP-0021-network-tx.md  
**版本**：v1.0  
**日期**：2026-05-10

---

## 1. 场景概述

本场景是商场的**高级形态**：完成从"感知→分析→决策→执行"的完整闭环。网络生产者接收外部控制指令或遥测，数据分析Worker执行算法并生成控制决策，最终通过另一个网络通道将控制指令回传至外部设备。

核心验证点：
- 商场的**输出能力**（不仅是内部消费，还能影响外部世界）
- Worker的**决策权威性**（其产出直接触发物理动作）
- **安全隔离**（控制指令必须经过严格审核与权限校验）
- **反馈验证**（控制执行后，系统通过新遥测验证效果）

---

## 2. 参与者清单

| 角色 | 实例 | 职能 | SHM权限 |
|------|------|------|---------|
| 生产者A（输入） | network_rx_producer | 接收外部遥测/指令（TCP/UDP监听） | 写入：`/shm/network/rx_frame` |
| 混合Worker | control_worker | 订阅输入数据，执行控制算法，生成控制指令 | 读取：`/shm/network/rx_frame`；写入：`/shm/control/tx_command` |
| 生产者B（输出） | network_tx_producer | 读取控制指令，发送至外部执行器 | 读取：`/shm/control/tx_command`；外部写入：TCP/UDP发送 |
| 消费者 | console_consumer | 监控闭环全链路状态 | 读取：全部三个矢量 |
| 商场 | mall_core | 管理闭环数据流、控制权限审核、安全隔离 | 管理：全部SHM矢量 |

---

## 3. 数据流架构

```
外部设备（传感器/上位机）
    │
    │ TCP/UDP 上行
    ▼
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐     ┌──────────────────────┐
│ network_rx      │     │ 商场SHM（感知段）     │     │ control_worker  │     │ 商场SHM（决策段）     │
│ _producer       │     │                      │     │                 │     │                      │
│                 │     │ /shm/network/        │     │                 │     │ /shm/control/        │
│ recv_external() │────→│ /rx_frame            │────→│ control_algo()  │────→│ /tx_command          │
│   [无状态函数]   │     │                      │     │   [无状态函数]   │     │                      │
└─────────────────┘     └──────────────────────┘     └─────────────────┘     └──────────────────────┘
                                                                                  │
                                                                                  ▼
                                                                           ┌─────────────────┐
                                                                           │ network_tx        │
                                                                           │ _producer         │
                                                                           │                 │
                                                                           │ send_external() │
                                                                           │   [无状态函数]   │
                                                                           └─────────────────┘
                                                                                  │
                                                                                  │ TCP/UDP 下行
                                                                                  ▼
                                                                           外部执行器（电机/开关/射频模块）
                                                                                  │
                                                                                  │ 执行结果遥测
                                                                                  ▼
                                                                           ┌─────────────────┐
                                                                           │ 新遥测帧         │
                                                                           │（进入下一轮闭环） │
                                                                           └─────────────────┘
```

---

## 4. SHM矢量规范

### 4.1 感知段矢量：`/shm/network/rx_frame`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"network_rx_001"` |
| `product_type` | string(16) | `"network.rx_frame"` |
| `timestamp` | uint64 | 接收时间戳 |
| `version` | uint16 | `1` |
| `src_ip` | uint32 | 源IP |
| `src_port` | uint16 | 源端口 |
| `protocol` | uint8 | `1`=TCP, `2`=UDP |
| `frame_type` | uint8 | `1`=遥测数据, `2`=状态查询, `3`=心跳包 |
| `payload_len` | uint32 | 数据长度 |
| `payload` | byte[] | 原始数据 |
| `checksum` | uint32 | CRC32 |

### 4.2 决策段矢量：`/shm/control/tx_command`

| 字段 | 类型 | 说明 |
|------|------|------|
| `producer_id` | string(32) | `"control_worker_001"` |
| `product_type` | string(16) | `"control.tx_command"` |
| `timestamp` | uint64 | 指令生成时间戳 |
| `version` | uint16 | `1` |
| **血缘信息** |||
| `parent_vector_id` | string(36) | 来源遥测帧UUID |
| `parent_producer_id` | string(32) | `"network_rx_001"` |
| **控制指令** |||
| `command_id` | uint32 | 指令唯一编号（用于ACK匹配） |
| `target_ip` | uint32 | 目标执行器IP |
| `target_port` | uint16 | 目标端口 |
| `command_type` | uint8 | `1`=设置频率, `2`=设置功率, `3`=开关控制, `4`=模式切换 |
| `command_param` | float64 | 参数值（如频率Hz、功率dBm） |
| `priority` | uint8 | `0`=普通, `9`=紧急 |
| `timeout_ms` | uint16 | 指令超时时间（毫秒） |
| **安全字段** |||
| `auth_hash` | string(64) | 控制Worker授权签名（商场验证） |
| `safety_check` | uint8 | `1`=已通过安全校验, `0`=待校验 |

---

## 5. 运行时序流程

### 阶段A：闭环初始化

```
T0: 商场启动
    ├─→ 创建 `/shm/network/rx_frame`（环形缓冲：1000帧）
    ├─→ 创建 `/shm/control/tx_command`（环形缓冲：100指令，带优先级队列）
    └─→ 加载授权文件（network_rx, control_worker, network_tx）

T0+Δt: network_rx_producer 启动
    └─→ 绑定监听端口，进入异步接收

T0+2Δt: control_worker 启动
    ├─→ 订阅 `/shm/network/rx_frame`
    ├─→ 注册产出权限 `/shm/control/tx_command`
    └─→ 加载控制算法参数（PID系数、阈值表等）

T0+3Δt: network_tx_producer 启动
    ├─→ 订阅 `/shm/control/tx_command`
    ├─→ 建立外部设备连接池（TCP长连接）
    └─→ 进入等待循环
```

### 阶段B：单次闭环周期

```
T1: 外部传感器发送遥测帧（频率偏差告警）
    └─→ network_rx_producer.recv_external() 触发
        ├─→ 解析帧：frame_type=1（遥测），payload含当前频率 2.399GHz（目标2.400GHz）
        ├─→ 写入 /shm/network/rx_frame
        └─→ 函数返回

T1+ε: 商场通知 control_worker
    └─→ control_algo() 执行
        ├─→ 读取遥测帧，提取当前频率误差：-1MHz
        ├─→ PID控制算法计算修正量：+1.005MHz（含超调补偿）
        ├─→ 组装 tx_command：
        │   command_type = 1（设置频率）
        │   command_param = 2.400005e9
        │   priority = 5（普通修正）
        │   timeout_ms = 1000
        ├─→ 生成 auth_hash（Worker私钥签名）
        ├─→ mall_shm_write(tx_command_vector, command)
        └─→ 函数返回

T1+2ε: 商场安全校验层介入（关键！）
    ├─→ 验证 auth_hash 是否匹配 control_worker 授权
    ├─→ 验证 command_param 是否在安全范围（频率：2.0-3.0GHz）
    ├─→ 验证 target_ip 是否在白名单
    ├─→ 全部通过 → 标记 safety_check=1，允许路由
    └─→ 任一失败 → 拒收指令，记录安全事件，Worker信用分 -20

T1+3ε: 商场通知 network_tx_producer（仅当 safety_check=1）
    └─→ send_external() 执行
        ├─→ 读取 tx_command
        ├─→ 通过TCP连接发送至外部执行器
        ├─→ 启动超时计时器（1000ms）
        └─→ 函数返回

T1+4ε: 外部执行器执行频率调整
    └─→ 执行器发送ACK（确认帧）
    └─→ 执行器发送新遥测（频率已修正为2.400GHz）
    └─→ 新一轮闭环启动（T2 ≈ T1 + 100ms）
```

---

## 6. 安全隔离机制

### 6.1 控制指令三重门

```
第一重：Worker权限门
  - control_worker 必须在授权文件中声明 "can_produce_control_commands: true"
  - 普通Worker（如analysis_worker）尝试写入 `/shm/control/tx_command` → 商场拒绝

第二重：参数安全门
  - 商场维护 "安全参数范围表"（如频率范围、功率上限）
  - Worker产出超出范围 → 商场拒收，不路由至 network_tx_producer

第三重：目标白名单门
  - network_tx_producer 只能向预配置的白名单IP发送指令
  - 若 tx_command.target_ip 不在白名单 → 商场拒收
```

### 6.2 紧急指令特权通道

```
priority=9 的指令绕过部分校验：
  - 跳过参数范围检查（信任Worker在紧急情况的判断）
  - 但永不跳过 auth_hash 验证（防止伪造紧急指令）
  - 商场记录紧急指令日志，供事后审计
```

---

## 7. 异常处理点

| 异常场景 | 触发条件 | 商场响应 |
|----------|----------|----------|
| **Worker决策错误** | PID算法发散，输出荒谬值（如频率=999GHz） | 参数安全门拦截，拒收指令，Worker信用分 -10，Console显示"控制指令被安全系统拦截" |
| **网络回传失败** | TCP连接断开，send_external() 失败 | network_tx_producer 报告失败，商场标记指令为"待重传"，3次失败后标记"失败"，通知Worker |
| **执行器超时** | 1000ms内未收到ACK | 商场标记指令"超时"，Worker可选择重发或升级告警级别 |
| **权限伪造** | Worker使用他人 auth_hash | 商场检测到签名不匹配，立即冻结Worker所有控制权限，触发安全审计 |
| **闭环振荡** | 控制指令导致系统不稳定（遥测数据剧烈波动） | 商场检测到振荡模式（连续5次反向修正），自动启用"保守模式"：降低Worker控制增益 |

---

## 8. 测试验证标准

| 测试项 | 方法 | 通过标准 |
|--------|------|----------|
| **闭环延迟** | 测量遥测输入→控制输出→ACK返回时间 | P99 < 150ms（本地网络） |
| **安全拦截** | Worker发送频率=999GHz的指令 | 商场100%拦截，不发送至外部设备 |
| **权限伪造** | 尝试用analysis_worker签名发送控制指令 | 商场100%拒绝，冻结涉事Worker |
| **紧急指令** | 满负荷时发送紧急停止 | 延迟 < 20ms，立即执行 |
| **闭环稳定性** | 持续运行1000个周期 | 系统不发散，频率误差收敛至 < 0.01% |
| **故障恢复** | 外部执行器断线10秒后恢复 | 商场自动重连，闭环恢复，无数据丢失 |

---

## 9. 场景哲学注释

> 本场景是商场的"成人礼"。前四个场景都是"只读"或"内部循环"，本场景第一次让商场的决策**触及物理世界**。这是巨大的权力，也是巨大的责任。因此，安全三重门不是束缚，而是**信任的放大器**——正因为有严格的校验，外部设备才敢于接受商场的指令。商场的终极形态不是数据中心，而是**物理世界的数字孪生控制器**。

---

## 10. 五场景演进关系图

```
SC-0001: 基础IO回路
    │ 键盘 → Console
    │ 验证：商场最小可行运行
    ▼
SC-0002: 网络遥测回路
    │ 网络 → Worker → Console
    │ 验证：链式处理与Worker混合角色
    ▼
SC-0003: 仿真验证回路
    │ 仿真 → Worker → Console
    │ 验证：可控输入与可验证输出
    ▼
SC-0004: 多源融合回路
    │ 键盘+网络+仿真 → Worker → Console
    │ 验证：多对多路由与公平调度
    ▼
SC-0005: 闭环控制回路
    │ 网络 → Worker → 网络回传
    │ 验证：输出能力、安全隔离、物理世界交互
```

> 五个场景构成商场的**能力阶梯**：从最简单的数据搬运，到最复杂的物理控制，每一步都在前一步的基础上增加一个新维度，但始终遵循同一套铁律。

---

**审核状态**：待审核  
**前置依赖**：SC-0004（多源融合回路）  
**状态**：商场完整能力验证终点
