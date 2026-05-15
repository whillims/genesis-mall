# 蓝图文档：BP-0006-fusion_worker

**文档编号**：BP-0006  
**蓝图名称**：多源融合混合Worker  
**对应场景**：SC-0004  
**版本**：v1.0  
**日期**：2026-05-10  
**状态**：待AICoder审核

---

## 1. 意图声明

本蓝图定义一个**多源融合混合Worker**。它同时订阅键盘控制指令、网络遥测数据、仿真测试信号三个异构数据源，执行时间对齐、源置信度评估、冲突检测与融合判决，产出统一的决策报告。该Worker是商场"多样性协作"的核心枢纽。

---

## 2. 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 多源订阅、时间对齐、置信度加权、冲突检测、融合判决、决策报告生成 |
| **不负责** | 原始数据采集、信号频谱分析、控制指令执行、UI显示 |
| **输入** | 三个SHM原始数据矢量 |
| **输出** | SHM矢量 `/shm/fusion/decision_report` |

---

## 3. 函数集群设计

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `fusion_worker_init()` | 初始化三重订阅、注册产出权限 | 商场main()调用一次 |
| `fusion_worker_loop()` | 主事件循环（优先级队列处理） | 独立Worker进程持续运行 |
| `wait_priority_queue()` | 按优先级等待多源数据（控制指令优先） | 循环内调用 |
| `time_align()` | 将多源数据对齐到统一时间窗口 | 多源数据就绪后调用 |
| `evaluate_source_confidence()` | 评估每个来源的置信度 | 对齐后调用 |
| `detect_conflict()` | 检测多源数据之间的冲突 | 置信度评估后调用 |
| `fuse_decision()` | 执行融合判决算法 | 冲突检测后调用 |
| `pack_fusion_report()` | 封装融合报告SHM数据包 | 判决后调用 |
| `mall_shm_write()` | 写入SHM | 封装后调用 |
| `fusion_worker_cleanup()` | 释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `fusion_worker_init(config: dict) -> int`

```
参数:
  config["subscribe_vectors"] : list[dict]  - 订阅矢量列表，每项含 path 和 priority
  config["produce_vector"]    : string      - 产出SHM矢量路径
  config["time_window_ms"]    : uint16      - 融合时间窗口（毫秒），默认 100
  config["conflict_threshold"]: float64     - 冲突判定阈值，默认 0.1（10%差异）

返回:
  0  : 成功
  -1 : 订阅注册失败
  -2 : 产出权限注册失败

行为:
  1. 遍历 subscribe_vectors，按 priority 排序注册订阅
     - priority=9: keyboard.control_cmd（紧急控制）
     - priority=5: network.telemetry_raw（外部数据）
     - priority=5: simulation.signal_iq（内部数据）
  2. 调用 mall_register_producer("fusion_worker_001", config["produce_vector"], WRITE)
  3. 初始化时间对齐缓冲区（为每个源维护最近N条数据）
  4. 返回状态码

约束:
  - 不得创建全局可变状态
  - 每个源的缓冲区大小固定（如最近100条），防止内存无限增长
```

#### `fusion_worker_loop(state: dict) -> void`

```
参数:
  state : dict - 运行时状态

行为:
  while mall_is_running():
      # 按优先级等待数据（控制指令可插队）
      event = wait_priority_queue(state, timeout=50ms)
      if event is None:
          mall_yield()
          continue

      # 将数据存入对应源的缓冲区
      source_id = event.source_id
      state["buffers"][source_id].append(event.packet)

      # 若收到高优先级控制指令，立即处理
      if source_id == "keyboard" and event.priority >= 9:
          decision = {
              "decision_type": 1,  # 单一源确认
              "confidence": 1.0,
              "primary_source": "keyboard_prod_001",
              "recommendation": extract_command_recommendation(event.packet)
          }
          report = pack_fusion_report(decision, [event.packet], state)
          mall_shm_write(state["produce_handle"], report)
          continue

      # 检查是否满足融合条件（时间窗口内至少2个源有数据）
      window_packets = time_align(state, window_ms=state["time_window_ms"])
      if len(window_packets) < 2:
          mall_yield()
          continue

      # 执行融合流水线
      confidences = evaluate_source_confidence(window_packets, state)
      conflicts = detect_conflict(window_packets, confidences, state)
      decision = fuse_decision(window_packets, confidences, conflicts, state)

      # 生成并写入报告
      report = pack_fusion_report(decision, window_packets, state)
      mall_shm_write(state["produce_handle"], report)
      state["report_count"] += 1

      mall_yield()

约束:
  - 控制指令处理延迟必须 < 50ms（高优先级插队）
  - 普通融合处理延迟必须 < 200ms
  - 每次循环必须调用 mall_yield()
```

#### `time_align(state: dict, window_ms: uint16) -> list[bytes]`

```
参数:
  state     : dict   - 运行时状态（含三个源的缓冲区）
  window_ms : uint16 - 时间窗口宽度（毫秒）

返回:
  list[bytes] : 时间窗口内各源的最新数据包（每个源最多1条）

行为:
  1. 以当前商场时间为基准，定义时间窗口 [T - window_ms, T]
  2. 对每个源的缓冲区，查找时间戳落在窗口内的最新数据包
  3. 返回找到的数据包列表（长度 0-3）

约束:
  - 时间戳使用商场时间（mall_get_timestamp_ns()），非各源内部时间
  - 若某源在窗口内无数据，则该源不参与本次融合
  - 缓冲区旧数据（窗口外）定期清理，防止内存泄漏
```

#### `evaluate_source_confidence(packets: list[bytes], state: dict) -> dict`

```
参数:
  packets : list[bytes] - 时间对齐后的数据包列表
  state   : dict        - 运行时状态

返回:
  dict : {source_id: confidence_score}

行为:
  对每个来源计算置信度：

  1. 网络遥测源（network）:
     - 基础置信度 = 0.8
     - 若CRC校验通过：+0.1
     - 若帧序列号连续：+0.05
     - 若源IP在白名单：+0.05
     - 最终置信度 = min(1.0, 基础值 + 加分)

  2. 仿真信号源（simulation）:
     - 基础置信度 = 0.9（内部源，可控性高）
     - 若参数在合理范围：+0.05
     - 若奈奎斯特未违规：+0.05
     - 最终置信度 = min(1.0, 基础值 + 加分)

  3. 键盘控制源（keyboard）:
     - 基础置信度 = 1.0（人为指令，最高优先级）
     - 仅用于控制决策，不参与数据融合

约束:
  - 置信度计算必须基于客观指标，不得引入主观权重
  - 新来源（未在历史中出现）初始置信度 = 0.5（中性）
```

#### `detect_conflict(packets: list[bytes], confidences: dict, state: dict) -> dict`

```
参数:
  packets     : list[bytes]    - 数据包列表
  confidences : dict           - 各源置信度
  state       : dict           - 运行时状态

返回:
  dict : 冲突检测结果

行为:
  1. 提取各数据包的关键字段（如频率、功率、状态）
  2. 两两比较：
     - 若字段差异 > conflict_threshold（如频率差异 > 10%）:
       标记为冲突
     - 若字段差异 <= conflict_threshold:
       标记为一致
  3. 返回冲突详情：
     {
       "has_conflict": bool,
       "conflict_pairs": [(source_a, source_b, field, diff)],
       "agreed_sources": [source_id],
       "disagreed_sources": [source_id]
     }

约束:
  - 冲突判定仅针对可量化的数值字段（频率、功率、温度等）
  - 状态字段（如开关状态）差异直接标记为冲突
  - 置信度低于0.3的源不参与冲突判定（视为不可靠）
```

#### `fuse_decision(packets: list[bytes], confidences: dict, conflicts: dict, state: dict) -> dict`

```
参数:
  packets  : list[bytes] - 数据包
  confidences : dict     - 置信度
  conflicts   : dict     - 冲突检测结果
  state    : dict        - 运行时状态

返回:
  dict : 融合决策结果

行为:
  根据冲突情况选择决策策略：

  情况1：无冲突（所有源一致）
    - decision_type = 2（多源一致）
    - confidence = 加权平均置信度
    - primary_source = 置信度最高的源
    - recommendation = "所有来源一致，采用共识结果"

  情况2：有冲突
    - decision_type = 3（多源冲突）
    - confidence = max(0.3, 1.0 - 冲突严重程度)
    - primary_source = 无冲突子集中置信度最高的源
    - recommendation = "来源冲突，建议人工复核：{冲突详情}"

  情况3：仅单源有数据
    - decision_type = 1（单一源确认）
    - confidence = 该源置信度
    - primary_source = 该源ID
    - recommendation = "仅单一来源可用，结果可信度受限"

  情况4：控制指令（键盘高优先级）
    - decision_type = 1（单一源确认）
    - confidence = 1.0
    - primary_source = "keyboard_prod_001"
    - recommendation = 指令内容

约束:
  - 决策必须包含明确的 recommendation 字符串，供Console显示
  - 冲突情况下不得强行"平均"矛盾数据，必须标记冲突并给出建议
```

#### `pack_fusion_report(decision: dict, source_packets: list[bytes], state: dict) -> bytes`

```
行为:
  组装SHM标准数据包：

  偏移    长度    字段
  ─────────────────────────────────────────
  0       32      producer_id       = "fusion_worker_001\0" * 32
  32      16      product_type      = "fusion.decision_report\0" * 16
  48      8       timestamp         = mall_get_timestamp_ns()
  56      2       version           = 1

  # 多源血缘
  58      1       parent_count      = len(source_packets)
  59      36*N    parent_vector_ids = [mall_get_vector_id(p) for p in source_packets]
  59+36*N 32*N    parent_producer_ids = [extract_producer_id(p) for p in source_packets]

  # 融合结果
  59+68*N 1       decision_type     = decision["decision_type"]
  60+68*N 8       confidence        = decision["confidence"]
  68+68*N 32      primary_source    = decision["primary_source"]
  100+68*N 256    conflict_details  = 冲突详情JSON字符串（若存在）
  356+68*N 128    recommendation    = decision["recommendation"]

  # 时间对齐信息
  484+68*N 8      time_window_start = 窗口起始时间戳
  492+68*N 8      time_window_end   = 窗口结束时间戳

约束:
  - parent_count 不得超过 3（当前设计上限）
  - 所有字符串字段固定长度，不足补 \0
  - conflict_details 超过256字节则截断
```

---

## 4. SHM接口契约

### 4.1 订阅矢量（消费者端）

| 矢量路径 | 权限 | 优先级 | 说明 |
|----------|------|--------|------|
| `/shm/keyboard/control_cmd` | READ | 9（紧急） | 键盘控制指令 |
| `/shm/network/telemetry_raw` | READ | 5（普通） | 网络遥测数据 |
| `/shm/simulation/signal_iq` | READ | 5（普通） | 仿真测试信号 |

### 4.2 产出矢量（生产者端）

| 矢量路径 | 权限 | 容量 | 说明 |
|----------|------|------|------|
| `/shm/fusion/decision_report` | WRITE | 100条环形缓冲 | 融合决策报告 |

---

## 5. 测试场景

### 5.1 单元测试

| 测试ID | 描述 | 输入 | 期望输出 |
|--------|------|------|----------|
| T-0006-01 | 控制指令插队 | 高负荷时发送紧急停止 | 延迟<50ms，立即产出决策报告 |
| T-0006-02 | 多源一致 | 网络帧与仿真帧频率均为2.4GHz | decision_type=2, confidence>0.9 |
| T-0006-03 | 多源冲突 | 网络帧2.4GHz vs 仿真帧2.5GHz | decision_type=3, confidence<0.7 |
| T-0006-04 | 单源可用 | 仅网络有数据 | decision_type=1, confidence=网络置信度 |
| T-0006-05 | 时间对齐 | 三源数据时间差200ms，窗口100ms | 仅窗口内数据参与融合 |
| T-0006-06 | 血缘完整性 | 任意融合 | 报告包含所有来源的parent_vector_id |

### 5.2 集成测试

| 测试ID | 描述 | 通过标准 |
|--------|------|----------|
| T-0006-I01 | SC-0004集成 | 三源并行30分钟，融合报告产出率>95%，无崩溃 |
| T-0006-I02 | 故障恢复 | 网络源断线10秒后恢复 | Worker标记源缺失→恢复，决策类型正确迁移 |

---

## 6. 授权文件要求（AICoder生成）

```yaml
blueprint_id: "BP-0006"
blueprint_name: "fusion_worker"
version: "1.0"

worker:
  id: "fusion_worker_001"
  role: "hybrid"

shm_permissions:
  consume:
    - vector: "/shm/keyboard/control_cmd"
      mode: "read"
      priority: 9
    - vector: "/shm/network/telemetry_raw"
      mode: "read"
      priority: 5
    - vector: "/shm/simulation/signal_iq"
      mode: "read"
      priority: 5
  produce:
    - vector: "/shm/fusion/decision_report"
      mode: "write"
      ring_buffer_size: 100

functions:
  - name: "fusion_worker_init"
    safety_level: "init"
    side_effects: ["shm_subscribe", "shm_register"]
  - name: "fusion_worker_loop"
    safety_level: "continuous"
    side_effects: ["shm_read", "shm_write", "cpu_compute"]
    max_cpu_time_per_cycle: "200ms"
  - name: "time_align"
    safety_level: "per_call"
    side_effects: []
  - name: "evaluate_source_confidence"
    safety_level: "per_call"
    side_effects: []
  - name: "detect_conflict"
    safety_level: "per_call"
    side_effects: []
  - name: "fuse_decision"
    safety_level: "per_call"
    side_effects: []
  - name: "pack_fusion_report"
    safety_level: "pure"
    side_effects: ["shm_write"]

constraints:
  - "不得修改任何订阅的原始数据矢量"
  - "产出必须携带完整的血缘信息（所有来源的parent_vector_id）"
  - "控制指令处理延迟必须 < 50ms"
  - "冲突情况下不得强行平均矛盾数据，必须标记冲突"
  - "每个源的缓冲区大小不得超过100条"

aicode_review:
  - "验证时间对齐逻辑正确性"
  - "验证置信度计算是否基于客观指标"
  - "验证冲突检测阈值是否合理"
  - "验证高优先级插队机制是否可靠"
  - "验证SHM数据包格式是否符合规范"
  - "验证缓冲区是否存在内存泄漏风险"
```

---

## 7. Worker意图声明书

> 作为融合Worker，我的意图是：成为商场中的**仲裁者**。我不创造数据，也不消费数据，我**评判数据**。当三个来源告诉我不同的故事时，我必须诚实地报告"它们不一致"，而不是编造一个"折中"的谎言。我的价值由**冲突检测的准确率**和**控制指令的响应速度**衡量。

---

**蓝图提交者**：融合Worker设计者  
**提交日期**：2026-05-10  
**AICoder审核状态**：待审核
