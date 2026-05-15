# 蓝图文档（定稿版）：BP-0040-TRADE-METRICS -- 交易指标系统

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0040-TRADE-METRICS |
| 蓝图名称 | 交易指标系统 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-12 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-0（创世纪验证级） |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

交易指标系统是商场的"公共信息产品"生产者，负责计算并发布三层交易指标，为管理者、消费者和调度器提供决策依据。核心职责包括：

1. **宏观指标采集**：全局时间膨胀系数(delta_mall)、全局吞吐量(throughput_mall)、SHM通道利用率(congestion_index)，每秒更新
2. **中观指标采集**：单个生产者膨胀系数(delta_producer)、待处理队列深度(queue_depth)、掩码匹配命中率(mask_hit_rate)，每5秒更新
3. **微观指标采集**：单交易端到端延迟(tx_latency)、单SHM段带宽利用率(shm_bandwidth)，实时更新
4. **建议事件生成**：基于指标阈值触发建议事件，发布至SHM建议事件队列

本系统严格遵循《商场概念定位宪章》（law-mall-identity.md）第三章定义的三层指标体系。

### 2.2 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| 交易记录 | SHM bus域 | 交易时间戳、ProducerID、ConsumerID、状态 |
| Worker状态 | SHM hb域 | Worker心跳、CPU、内存 |
| SHM段状态 | SHM vec域 | 段利用率、带宽 |
| 调度器状态 | SHM bus域 | 任务队列深度、匹配结果 |
| 监控指标 | SHM metrics域 | 基础运行时指标 |

### 2.3 数据去向

| 数据项 | 去向 | 说明 |
|--------|------|------|
| 宏观指标 | SHM metrics/global | 所有角色只读 |
| 中观指标 | SHM metrics/producer/{id} | 管理者和相关消费者只读 |
| 微观指标 | SHM metrics/tx/{tx_id} | 仅当事方只读 |
| 建议事件 | SHM bus域 | 建议事件队列 |

### 2.4 来源锁定

本系统的数据来源锁定为SHM矢量空间，不接收外部输入。

---

## 3. 函数设计

### 3.1 函数签名

```python
def metrics_init(
    ctx: dict,
    config: dict
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: WRITE_SHM_CONFIG
    初始化指标系统，创建指标存储段和阈值规则。
    """

def metrics_collect_global(
    ctx: dict
) -> dict:
    """
    [复杂度]: O(T) 其中T为时间窗口内交易数
    [安全等级]: READ_SHM + WRITE_SHM_METRICS
    采集宏观指标：delta_mall, throughput_mall, congestion_index。
    """

def metrics_collect_producer(
    ctx: dict,
    producer_id: str
) -> dict:
    """
    [复杂度]: O(T_p) 其中T_p为该生产者时间窗口内交易数
    [安全等级]: READ_SHM + WRITE_SHM_METRICS
    采集中观指标：delta_producer, queue_depth, mask_hit_rate。
    """

def metrics_collect_tx(
    ctx: dict,
    tx_id: str
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_SHM + WRITE_SHM_METRICS
    采集微观指标：tx_latency, shm_bandwidth。
    """

def metrics_publish_advisory(
    ctx: dict,
    suggestion: dict
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: WRITE_SHM_BUS
    发布建议事件至SHM建议事件队列。
    """
```

### 3.2 参数规格表

#### metrics_init

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空，含shm_root, metrics_ref | 上下文字典 |
| config | dict | 是 | 含collection_intervals, thresholds | 指标配置 |

**config结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| global_interval_ms | int | [100, 10000] | 宏观指标采集间隔（毫秒） |
| producer_interval_ms | int | [1000, 60000] | 中观指标采集间隔（毫秒） |
| tx_realtime | bool | - | 是否启用微观指标实时采集 |
| thresholds | dict | 含各指标阈值 | 建议触发阈值 |
| advisory_ttl_ms | int | [1000, 300000] | 建议事件有效期（毫秒） |

**thresholds结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| delta_mall_warning | float | > 1.0 | 全局膨胀系数警告阈值 |
| delta_mall_critical | float | > delta_mall_warning | 全局膨胀系数严重阈值 |
| congestion_warning | float | [0.0, 1.0] | SHM通道利用率警告阈值 |
| congestion_critical | float | > congestion_warning | SHM通道利用率严重阈值 |
| throughput_low | float | >= 0.0 | 吞吐量低阈值（交易/秒） |
| producer_delta_warning | float | > 1.0 | 生产者膨胀系数警告阈值 |
| queue_depth_warning | int | >= 0 | 队列深度警告阈值 |
| tx_latency_warning_ms | float | > 0.0 | 交易延迟警告阈值（毫秒） |

#### metrics_collect_global

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |

#### metrics_collect_producer

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| producer_id | str | 是 | 长度 <= 64，已注册 | 生产者标识 |

#### metrics_collect_tx

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| tx_id | str | 是 | 长度 <= 64，已存在 | 交易标识 |

#### metrics_publish_advisory

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| suggestion | dict | 是 | 含suggest_type, target, reason, evidence, action, authority | 建议事件内容 |

**suggestion结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| suggest_type | str | 枚举值 | throttle_producer / reallocate / upgrade_path / switch_producer / optimize_blueprint |
| target | str | 长度 <= 64 | 建议对象ID |
| reason | str | 非空 | 触发原因编码 |
| evidence | dict | 非空 | 证据数据（指标名->值） |
| action | dict | 非空 | 建议参数 |
| authority | str | 枚举值 | auto_execute / consumer_confirm / human_review |

### 3.3 数据来源定义

| SHM域 | 数据类型 | 预期规模 | 访问模式 |
|-------|----------|----------|----------|
| bus (总线域) | json | 交易记录、任务队列 | 只读 |
| hb (心跳域) | json | Worker状态 | 只读 |
| vec (矢量域) | float64/int64 | SHM段利用率 | 只读 |
| metrics (指标域) | float64/int64 | 三层指标存储 | 读写 |

### 3.4 返回值规格

#### metrics_init 返回值

```python
{
    "status": str,              # "ok" / "error"
    "metrics_id": str,          # 指标系统实例ID
    "config_hash": str,         # 配置摘要
    "thresholds_loaded": int,   # 加载的阈值规则数
    "timestamp": float,
    "error": str
}
```

#### metrics_collect_global 返回值

```python
{
    "status": str,              # "ok" / "error"
    "metrics_type": "global",   # 指标层级
    "delta_mall": float,        # 全局时间膨胀系数 t_actual / t_0
    "throughput_mall": float,   # 全局吞吐量（交易/秒）
    "congestion_index": float,  # SHM通道利用率 [0.0, 1.0]
    "collection_time_ms": float, # 采集耗时（毫秒）
    "advisories_triggered": list, # 触发的建议事件列表
    "timestamp": float,
    "error": str
}
```

#### metrics_collect_producer 返回值

```python
{
    "status": str,              # "ok" / "error"
    "metrics_type": "producer", # 指标层级
    "producer_id": str,         # 生产者ID
    "delta_producer": float,    # 生产者膨胀系数
    "queue_depth": int,         # 待处理队列深度
    "mask_hit_rate": float,     # 掩码匹配命中率 [0.0, 1.0]
    "collection_time_ms": float,
    "advisories_triggered": list,
    "timestamp": float,
    "error": str
}
```

#### metrics_collect_tx 返回值

```python
{
    "status": str,              # "ok" / "error"
    "metrics_type": "tx",       # 指标层级
    "tx_id": str,               # 交易ID
    "tx_latency_ms": float,     # 端到端延迟（毫秒）
    "shm_bandwidth": float,     # SHM段带宽利用率 [0.0, 1.0]
    "timestamp": float,
    "error": str
}
```

#### metrics_publish_advisory 返回值

```python
{
    "status": str,              # "ok" / "error"
    "suggest_id": str,          # 建议事件ID "SUG-{timestamp}-{seq}"
    "suggest_type": str,        # 建议类型
    "target": str,              # 建议对象
    "authority": str,           # 执行权限级别
    "ttl_ms": int,              # 有效期
    "timestamp": float,
    "error": str
}
```

### 3.5 关键设计：三层指标体系

与law-mall-identity.md第10条完全对齐：

| 层级 | 指标名称 | 定义 | 计算方式 | 发布频率 |
|------|----------|------|----------|----------|
| **宏观** | delta_mall | 全局时间膨胀系数 | t_actual_global / t_0_global | 每秒 |
| **宏观** | throughput_mall | 全局吞吐量 | completed_tx_count / time_window | 每秒 |
| **宏观** | congestion_index | SHM通道利用率 | used_shm_bytes / total_shm_bytes | 每秒 |
| **中观** | delta_producer | 单生产者膨胀系数 | t_actual_producer / t_0_producer | 每5秒 |
| **中观** | queue_depth | 待处理队列长度 | pending_task_count | 每5秒 |
| **中观** | mask_hit_rate | 掩码匹配成功率 | matched_count / total_intent_count | 每10秒 |
| **微观** | tx_latency | 单交易端到端延迟 | t_delivery - t_submit | 实时 |
| **微观** | shm_bandwidth | 单SHM段带宽利用率 | segment_used / segment_capacity | 每5秒 |

### 3.6 关键设计：建议事件生成

基于指标阈值自动触发建议事件，与law-mall-identity.md第12-14条完全对齐：

```
建议触发规则：

1. delta_mall > delta_mall_critical 持续30秒
   -> suggest_type: "reallocate"
   -> reason: "global_time_dilation_critical"
   -> authority: "auto_execute"

2. congestion_index > congestion_critical 持续10秒
   -> suggest_type: "throttle_producer"
   -> reason: "shm_congestion_critical"
   -> authority: "auto_execute"

3. delta_producer > producer_delta_warning 持续30秒
   -> suggest_type: "reallocate"
   -> reason: "producer_dilation_warning"
   -> authority: "consumer_confirm"

4. queue_depth > queue_depth_warning 持续15秒
   -> suggest_type: "upgrade_path"
   -> reason: "queue_depth_exceeded"
   -> authority: "consumer_confirm"

5. tx_latency > tx_latency_warning_ms 持续5秒
   -> suggest_type: "upgrade_path"
   -> reason: "tx_latency_exceeded"
   -> authority: "consumer_confirm"

6. throughput_mall < throughput_low 持续60秒
   -> suggest_type: "optimize_blueprint"
   -> reason: "throughput_degraded"
   -> authority: "human_review"
```

### 3.7 关键设计：指标存储与访问

```
指标存储位置（与law-mall-identity.md第11条对齐）：

宏观指标: shm://mall/metrics/global
  - 所有角色只读
  - 数据结构: {"delta_mall": float, "throughput_mall": float, "congestion_index": float, "timestamp": float}

中观指标: shm://mall/metrics/producer/{producer_id}
  - 管理者和相关消费者只读
  - 数据结构: {"delta_producer": float, "queue_depth": int, "mask_hit_rate": float, "timestamp": float}

微观指标: shm://mall/metrics/tx/{tx_id}
  - 仅当事方只读
  - 数据结构: {"tx_latency_ms": float, "shm_bandwidth": float, "timestamp": float}

指标数据为SEALED态只读产品，不可篡改。
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | ctx 非空且包含必需键 | `ERR_CTX_EMPTY` | `error` |
| PRE-002 | config 非空且包含必需字段 | `ERR_CONFIG_INVALID` | `error` |
| PRE-003 | global_interval_ms 在 [100, 10000] 范围内 | `ERR_GLOBAL_INTERVAL_INVALID` | `error` |
| PRE-004 | producer_interval_ms 在 [1000, 60000] 范围内 | `ERR_PRODUCER_INTERVAL_INVALID` | `error` |
| PRE-005 | thresholds 中所有阈值在合法范围内 | `ERR_THRESHOLDS_INVALID` | `error` |
| PRE-006 | producer_id 已在注册表中注册 | `ERR_PRODUCER_NOT_FOUND` | `error` |
| PRE-007 | tx_id 已存在于交易记录中 | `ERR_TX_NOT_FOUND` | `error` |
| PRE-008 | suggestion.suggest_type 为合法枚举值 | `ERR_SUGGEST_TYPE_INVALID` | `error` |
| PRE-009 | suggestion.authority 为合法枚举值 | `ERR_AUTHORITY_INVALID` | `error` |
| PRE-010 | suggestion.evidence 非空 | `ERR_EVIDENCE_EMPTY` | `error` |
| PRE-011 | advisory_ttl_ms 在 [1000, 300000] 范围内 | `ERR_TTL_INVALID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | delta_mall >= 0.0 | 膨胀系数非负 |
| POST-003 | throughput_mall >= 0.0 | 吞吐量非负 |
| POST-004 | congestion_index in [0.0, 1.0] | 利用率范围合法 |
| POST-005 | mask_hit_rate in [0.0, 1.0] | 命中率范围合法 |
| POST-006 | tx_latency_ms >= 0.0 | 延迟非负 |
| POST-007 | suggest_id 格式为 "SUG-{timestamp}-{seq}" | 建议ID格式正确 |
| POST-008 | 指标写入SHM后为只读（SEALED态） | 不可篡改性 |

### 4.3 不变量

- **零全局状态**：不读写任何全局变量，所有状态通过ctx参数传入
- **零堆分配**：不使用动态内存分配
- **零系统调用**：不进行文件IO、网络IO、进程管理
- **零副作用**：不修改输入参数，所有变更通过返回值和SHM写入传递
- **确定性**：相同输入必得相同输出
- **指标不可篡改**：写入SHM的指标数据为SEALED态只读产品
- **建议事件有时效**：每条建议事件都有TTL，过期自动失效

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_CTX_EMPTY` | ctx为空或缺少必需键 | `error` | "ctx is empty or missing required keys" |
| `ERR_CONFIG_INVALID` | config为空或缺少必需字段 | `error` | "config is empty or missing required fields" |
| `ERR_GLOBAL_INTERVAL_INVALID` | global_interval_ms超出范围 | `error` | "global_interval_ms must be in [100, 10000]" |
| `ERR_PRODUCER_INTERVAL_INVALID` | producer_interval_ms超出范围 | `error` | "producer_interval_ms must be in [1000, 60000]" |
| `ERR_THRESHOLDS_INVALID` | 阈值超出合法范围 | `error` | "one or more thresholds are out of valid range" |
| `ERR_PRODUCER_NOT_FOUND` | producer_id未注册 | `error` | "producer not found in registry" |
| `ERR_TX_NOT_FOUND` | tx_id不存在 | `error` | "transaction not found" |
| `ERR_SUGGEST_TYPE_INVALID` | suggest_type不是合法枚举值 | `error` | "suggest_type must be one of: throttle_producer, reallocate, upgrade_path, switch_producer, optimize_blueprint" |
| `ERR_AUTHORITY_INVALID` | authority不是合法枚举值 | `error` | "authority must be one of: auto_execute, consumer_confirm, human_review" |
| `ERR_EVIDENCE_EMPTY` | evidence为空 | `error` | "evidence must not be empty" |
| `ERR_TTL_INVALID` | advisory_ttl_ms超出范围 | `error` | "advisory_ttl_ms must be in [1000, 300000]" |

---

## 5. 测试场景

### 5.1 正常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-001 | 标准ctx + 标准config | 默认阈值 | init返回status="ok"，metrics_id非空 |
| TC-002 | 含交易记录的ctx | 任意 | collect_global返回status="ok"，delta_mall >= 0.0 |
| TC-003 | 含生产者记录的ctx | 任意 | collect_producer返回status="ok"，queue_depth >= 0 |
| TC-004 | 含交易记录的ctx | 任意 | collect_tx返回status="ok"，tx_latency_ms >= 0.0 |
| TC-005 | 合法suggestion | 任意 | publish_advisory返回status="ok"，suggest_id格式正确 |
| TC-006 | delta_mall超过critical阈值 | delta_mall_critical=2.0 | collect_global返回advisories_triggered非空 |

### 5.2 边界路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-007 | global_interval_ms=100 | 最小间隔 | init返回status="ok" |
| TC-008 | global_interval_ms=10000 | 最大间隔 | init返回status="ok" |
| TC-009 | 零交易记录 | 任意 | collect_global返回throughput_mall=0.0 |
| TC-010 | congestion_index=0.0 | 任意 | collect_global返回congestion_index=0.0 |
| TC-011 | congestion_index=1.0 | 任意 | collect_global返回congestion_index=1.0 |
| TC-012 | mask_hit_rate=0.0 | 任意 | collect_producer返回mask_hit_rate=0.0 |
| TC-013 | mask_hit_rate=1.0 | 任意 | collect_producer返回mask_hit_rate=1.0 |
| TC-014 | advisory_ttl_ms=1000 | 最小TTL | publish_advisory返回status="ok" |
| TC-015 | advisory_ttl_ms=300000 | 最大TTL | publish_advisory返回status="ok" |

### 5.3 异常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-016 | ctx={} | 任意 | init返回status="error"，error含ERR_CTX_EMPTY |
| TC-017 | config={} | 任意 | init返回status="error"，error含ERR_CONFIG_INVALID |
| TC-018 | global_interval_ms=50 | 低于最小值 | init返回status="error"，error含ERR_GLOBAL_INTERVAL_INVALID |
| TC-019 | 未注册的producer_id | 任意 | collect_producer返回status="error"，error含ERR_PRODUCER_NOT_FOUND |
| TC-020 | 不存在的tx_id | 任意 | collect_tx返回status="error"，error含ERR_TX_NOT_FOUND |
| TC-021 | suggest_type="INVALID" | 任意 | publish_advisory返回status="error"，error含ERR_SUGGEST_TYPE_INVALID |
| TC-022 | authority="INVALID" | 任意 | publish_advisory返回status="error"，error含ERR_AUTHORITY_INVALID |
| TC-023 | evidence={} | 任意 | publish_advisory返回status="error"，error含ERR_EVIDENCE_EMPTY |
| TC-024 | delta_mall_critical=0.5 < delta_mall_warning=1.0 | 非法阈值 | init返回status="error"，error含ERR_THRESHOLDS_INVALID |

### 5.4 性能路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-025 | 10000条交易记录 | 任意 | collect_global耗时 < 200ms |
| TC-026 | 100个生产者同时采集 | 任意 | 100次collect_producer总耗时 < 2s |
| TC-027 | 连续1000次publish_advisory | 任意 | 无内存泄漏，suggest_id严格递增 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `time` | 时间戳获取与间隔计算 | Python 3.8+ |
| `hashlib` | 指标数据完整性校验 | Python 3.8+ |

### 6.2 外部依赖

**零外部依赖**。不依赖任何第三方库。

### 6.3 蓝图间依赖

| 依赖蓝图 | 依赖类型 | 依赖内容 |
|----------|----------|----------|
| BP-0003（SHM矢量管理器） | 数据读写 | SHM指标段的读写接口 |
| BP-0036（监控） | 数据读取 | 基础运行时指标 |
| BP-0038（调度器） | 功能协作 | 任务队列状态、调度决策反馈 |

### 6.4 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(T) per collection | T为时间窗口内交易数 |
| 内存 | <= 256KB | 指标系统状态与中间数据 |
| 栈帧 | <= 8KB | 单次函数调用最大栈深 |
| 执行时限 | collect_global <= 200ms | 宏观指标采集上限 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | YES | 不进行任何文件读写 |
| 网络零IO | YES | 不进行任何网络通讯 |
| 零全局副作用 | YES | 不修改任何全局状态 |
| 输入消毒 | YES | 对所有输入参数进行类型、范围、存在性校验 |
| 确定性输出 | YES | 相同输入产生相同输出 |
| SHM权限合规 | YES | 宏观指标所有角色只读，中观指标管理者+相关消费者只读，微观指标仅当事方只读 |
| 栈帧安全 | YES | 不存在递归，最大栈深 <= 8KB |
| 零堆分配 | YES | 不使用动态内存分配 |
| 零OOP | YES | 不使用类定义、继承、self引用 |
| 零系统调用 | YES | 不调用操作系统API |

### 7.2 stdout特许

本系统不直接向stdout输出。所有诊断信息通过返回值dict传递。

---

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证，确保11条前置条件全部有对应错误码 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现，禁止class/struct/self，所有状态通过ctx参数传递 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例，覆盖TC-001至TC-027全部27个用例 |
| 安全扫描 | 验证三大零原则、SHM权限合规、指标只读性、建议事件TTL机制 |
| 授权文件输出 | 按5字段JSON结构生成授权文件（auth_header, design_meta, source_code, compiled_binary, verification_report） |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据（BP-0040-TRADE-METRICS, v1.0.0, Genesis-Worker-001） |
| `design_meta` | 第3节函数设计（5个函数签名、参数规格、返回值规格、三层指标定义、建议触发规则） |
| `source_code` | AICoder根据蓝图意图生成的纯函数Python实现 |
| `compiled_binary` | PYC编译产物（base64编码） |
| `verification_report` | 基于第5节27个测试场景的验收测试报告 |

### 8.3 特别审查要求

1. **三层指标对齐**：确保指标定义与law-mall-identity.md第10条完全一致（名称、定义、频率、消费者）
2. **建议事件格式**：确保建议事件结构与law-mall-identity.md第14条完全一致（suggest_id, suggest_type, target, reason, evidence, action, authority, ttl_ms）
3. **指标只读性**：确保写入SHM的指标数据为SEALED态，不可被任何实体篡改
4. **阈值合理性**：确保critical阈值 > warning阈值，防止阈值配置逻辑反转
5. **存储路径对齐**：确保指标存储路径与law-mall-identity.md第11条完全一致

---

*蓝图审核状态：待审核*
*设计者：Genesis-Worker-001*
*AICoder评估：待外置AICoder审查*
