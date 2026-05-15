# 蓝图文档（定稿版）：BP-0041-PATH-ROUTER -- 路径分级路由器

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0041-PATH-ROUTER |
| 蓝图名称 | 路径分级路由器 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-12 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-0（创世纪验证级） |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

路径分级路由器实现P0-P3四等级交易路径的路由与资源管理，确保不同时效等级的交易获得对应的延迟保障和SHM资源分配。核心职责包括：

1. **路径分配**：根据交易的时效等级（Latency Class）自动映射到对应的路径等级（P0-P3），分配SHM段资源
2. **路径升级**：支持消费者请求升级路径等级（只能升级不能降级），重新分配SHM资源
3. **路径冲突解决**：当多个交易竞争同一路径资源时，按优先级规则裁决（路径等级 > 信用分 > 提交时间）
4. **资源管理**：管理各路径等级的SHM段资源池，跟踪利用率，触发资源告警

本路由器严格遵循《交易掩码与单向性法则》（law-trade-mask.md）第四章定义的四等级路径体系，以及《商场概念定位宪章》（law-mall-identity.md）第二章定义的路径分级规则。

### 2.2 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| 交易请求 | SHM bus域 | 交易的时效等级、掩码、优先级 |
| SHM段状态 | SHM vec域 | 各路径等级SHM段的利用率 |
| 生产者信用分 | SHM auth域 | 生产者历史信用评分 |
| 调度器决策 | SHM bus域 | 调度器的路径分配建议 |
| 交易指标 | SHM metrics域 | 路径延迟、带宽利用率 |

### 2.3 数据去向

| 数据项 | 去向 | 说明 |
|--------|------|------|
| 路径分配结果 | SHM bus域 | 交易的路径等级与SHM段引用 |
| 资源利用率 | SHM metrics域 | 各路径等级的利用率指标 |
| 冲突裁决记录 | SHM bus域 | 路径冲突解决审计日志 |

### 2.4 来源锁定

本路由器的数据来源锁定为SHM矢量空间，不接收外部输入。

---

## 3. 函数设计

### 3.1 函数签名

```python
def path_router_init(
    ctx: dict,
    config: dict
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: WRITE_SHM_CONFIG
    初始化路径路由器，配置P0-P3四等级路径参数。
    """

def path_router_assign(
    ctx: dict,
    transaction: dict
) -> dict:
    """
    [复杂度]: O(P) 其中P为路径等级数（固定为4）
    [安全等级]: READ_SHM + WRITE_SHM_BUS
    为交易分配路径等级和SHM段资源。
    """

def path_router_upgrade(
    ctx: dict,
    tx_id: str,
    new_level: str
) -> dict:
    """
    [复杂度]: O(P)
    [安全等级]: READ_SHM + WRITE_SHM_BUS + WRITE_SHM_VEC
    升级交易的路径等级，重新分配SHM资源。
    """

def path_router_conflict_resolve(
    ctx: dict,
    competing_txs: list
) -> dict:
    """
    [复杂度]: O(N * log N) 其中N为竞争交易数
    [安全等级]: READ_SHM + WRITE_SHM_BUS
    解决多个交易竞争同一路径资源的冲突。
    """
```

### 3.2 参数规格表

#### path_router_init

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空，含shm_root, path_pool_ref | 上下文字典 |
| config | dict | 是 | 含path_definitions, conflict_rules | 路径配置 |

**config结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| path_definitions | dict | 含P0/P1/P2/P3配置 | 四等级路径定义 |
| conflict_rules | dict | 含priority_weights | 冲突解决权重 |
| max_p0_concurrent | int | [1, 10] | P0通道最大并发交易数 |
| upgrade_allowed | bool | - | 是否允许路径升级 |
| shm_segment_sizes | dict | 含P0/P1/P2/P3大小 | 各路径SHM段大小（字节） |

**path_definitions结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| P0.shm_type | str | "exclusive" | SHM段类型：独占 |
| P0.latency guarantee_ms | float | (0, 1.0] | 延迟保障（毫秒） |
| P0.priority | int | 4 | 闸口优先级（最高） |
| P1.shm_type | str | "dedicated" | SHM段类型：专用 |
| P1.latency_guarantee_ms | float | (1.0, 10.0] | 延迟保障 |
| P1.priority | int | 3 | 闸口优先级 |
| P2.shm_type | str | "shared" | SHM段类型：共享 |
| P2.latency_guarantee_ms | float | (10.0, 100.0] | 延迟保障 |
| P2.priority | int | 2 | 闸口优先级 |
| P3.shm_type | str | "shared" | SHM段类型：共享 |
| P3.latency_guarantee_ms | float | None（无保障） | 无延迟保障 |
| P3.priority | int | 1 | 闸口优先级（最低） |

#### path_router_assign

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| transaction | dict | 是 | 含tx_id, latency_class, producer_id, demand_mask | 交易请求 |

**transaction结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| tx_id | str | 长度 <= 64 | 交易唯一标识 |
| latency_class | str | L0/L1/L2/L3 | 时效等级 |
| producer_id | str | 已注册 | 生产者ID |
| demand_mask | int | 32位无符号 | 需求掩码 |
| submit_timestamp | float | > 0 | 提交时间戳 |
| credit_score | float | [0.0, 1.0] | 生产者信用分 |

#### path_router_upgrade

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| tx_id | str | 是 | 长度 <= 64，已分配路径 | 交易标识 |
| new_level | str | 是 | P0/P1/P2/P3 | 目标路径等级 |

#### path_router_conflict_resolve

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| competing_txs | list | 是 | 长度 >= 2，每项含tx_id, current_level, credit_score, submit_timestamp | 竞争交易列表 |

### 3.3 数据来源定义

| SHM域 | 数据类型 | 预期规模 | 访问模式 |
|-------|----------|----------|----------|
| bus (总线域) | json | 交易记录 | 读写 |
| vec (矢量域) | bytes | SHM段资源池 | 读写 |
| auth (授权域) | json | 生产者信用分 | 只读 |
| metrics (指标域) | float64 | 路径利用率指标 | 读写 |

### 3.4 返回值规格

#### path_router_init 返回值

```python
{
    "status": str,              # "ok" / "error"
    "router_id": str,           # 路由器实例ID
    "config_hash": str,         # 配置摘要
    "paths_initialized": list,  # 初始化的路径等级列表 ["P0", "P1", "P2", "P3"]
    "timestamp": float,
    "error": str
}
```

#### path_router_assign 返回值

```python
{
    "status": str,              # "ok" / "error"
    "tx_id": str,               # 交易ID
    "path_level": str,          # 分配的路径等级 P0/P1/P2/P3
    "shm_segment_ref": str,     # 分配的SHM段引用
    "latency_guarantee_ms": float,  # 延迟保障（P3为None）
    "shm_type": str,            # SHM段类型 exclusive/dedicated/shared
    "priority": int,            # 闸口优先级
    "timestamp": float,
    "error": str
}
```

#### path_router_upgrade 返回值

```python
{
    "status": str,              # "ok" / "error"
    "tx_id": str,
    "previous_level": str,      # 升级前路径等级
    "new_level": str,           # 升级后路径等级
    "previous_shm_ref": str,    # 原SHM段引用
    "new_shm_ref": str,         # 新SHM段引用
    "latency_guarantee_ms": float,
    "timestamp": float,
    "error": str
}
```

#### path_router_conflict_resolve 返回值

```python
{
    "status": str,              # "ok" / "error"
    "resolution_count": int,    # 裁决的交易数
    "winners": list,            # 获胜交易列表 [{"tx_id", "path_level", "shm_segment_ref", "reason"}]
    "losers": list,             # 失败交易列表 [{"tx_id", "reason", "suggested_action"}]
    "resolution_log": list,     # 裁决详情记录
    "timestamp": float,
    "error": str
}
```

### 3.5 关键设计：四等级路径体系

与law-trade-mask.md第12条及law-mall-identity.md第6条完全对齐：

```
P0 紧急通道：
  - SHM策略：独占SHM段（exclusive）
  - 延迟保障：<= 1ms
  - 闸口优先级：最高（4）
  - 并发限制：max_p0_concurrent（可配置，默认3）
  - 适用场景：闭环控制指令、安全告警
  - 资源回收：交易完成后立即回收独占段

P1 快速通道：
  - SHM策略：专用SHM段（dedicated）
  - 延迟保障：<= 10ms
  - 闸口优先级：高（3）
  - 适用场景：频谱显示、状态监控
  - 资源回收：交易完成后释放专用段

P2 标准通道：
  - SHM策略：共享SHM段（shared）
  - 延迟保障：<= 100ms
  - 闸口优先级：标准（2）
  - 适用场景：数据分析、融合处理
  - 资源回收：共享段由资源池统一管理

P3 经济通道：
  - SHM策略：共享SHM段（shared）
  - 延迟保障：无保障
  - 闸口优先级：最低（1）
  - 适用场景：日志归档、审计备份
  - 资源回收：共享段由资源池统一管理
```

### 3.6 关键设计：路径分配规则

与law-trade-mask.md第13条及law-mall-identity.md第7条完全对齐：

```
路径分配流程：

1. 默认映射：latency_class -> path_level
   L0 -> P0
   L1 -> P1
   L2 -> P2
   L3 -> P3

2. 掩码匹配优先：先验证 DemandMask & OutputMask != 0

3. 资源可用性检查：
   P0: 当前并发数 < max_p0_concurrent
   P1: 专用SHM段有可用槽位
   P2/P3: 共享SHM段利用率 < 95%

4. 分配SHM段引用，返回路径分配结果

5. 若目标路径资源不足：
   - P0不足：返回ERR_P0_CAPACITY_EXCEEDED
   - P1不足：降级至P2（记录降级事件）
   - P2不足：降级至P3（记录降级事件）
   - P3不足：排队等待（返回WAITING状态）
```

### 3.7 关键设计：路径冲突解决

与law-trade-mask.md第14条完全对齐：

```
冲突解决三阶优先级：

第一阶：路径等级优先
  P0 > P1 > P2 > P3
  高等级交易优先获得资源

第二阶：信用分优先（同等级时）
  credit_score 更高者优先
  信用分范围 [0.0, 1.0]

第三阶：提交时间优先（同等级同信用时）
  submit_timestamp 更早者优先（FIFO）

裁决算法：
  sort_key = (path_level_priority, credit_score, -submit_timestamp)
  按sort_key降序排列，资源不足时截断

失败交易处理：
  suggested_action:
    - "retry_later": 稍后重试
    - "downgrade_path": 降级到更低路径
    - "queue_wait": 排队等待资源释放
```

### 3.8 关键设计：路径升级

```
升级规则：
  - 只能升级不能降级：P3->P2->P1->P0
  - 升级需目标路径有可用资源
  - 升级需消费者确认（authority = consumer_confirm）
  - 升级操作释放原SHM段，分配新SHM段
  - 升级事件记入审计日志
  - 若upgrade_allowed=false，所有升级请求拒绝
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | ctx 非空且包含必需键 | `ERR_CTX_EMPTY` | `error` |
| PRE-002 | config 非空且包含path_definitions | `ERR_CONFIG_INVALID` | `error` |
| PRE-003 | path_definitions 包含P0/P1/P2/P3全部四个等级 | `ERR_PATH_DEFINITIONS_INCOMPLETE` | `error` |
| PRE-004 | max_p0_concurrent 在 [1, 10] 范围内 | `ERR_P0_LIMIT_INVALID` | `error` |
| PRE-005 | transaction.tx_id 长度 <= 64 | `ERR_TX_ID_INVALID` | `error` |
| PRE-006 | transaction.latency_class 为 L0/L1/L2/L3 之一 | `ERR_LATENCY_CLASS_INVALID` | `error` |
| PRE-007 | transaction.producer_id 已注册 | `ERR_PRODUCER_NOT_REGISTERED` | `error` |
| PRE-008 | transaction.demand_mask 为32位无符号整数 | `ERR_MASK_INVALID` | `error` |
| PRE-009 | tx_id 已分配路径（升级时） | `ERR_TX_NOT_ASSIGNED` | `error` |
| PRE-010 | new_level 为 P0/P1/P2/P3 之一 | `ERR_PATH_LEVEL_INVALID` | `error` |
| PRE-011 | new_level 高于当前路径等级（升级方向） | `ERR_UPGRADE_DIRECTION_INVALID` | `error` |
| PRE-012 | competing_txs 长度 >= 2 | `ERR_COMPETING_TXS_INSUFFICIENT` | `error` |
| PRE-013 | upgrade_allowed 为 True（升级时） | `ERR_UPGRADE_DISABLED` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | path_level 必须为 P0/P1/P2/P3 之一 | 路径等级合法 |
| POST-003 | P0延迟保障 <= 1ms | P0时效性保障 |
| POST-004 | P1延迟保障 <= 10ms | P1时效性保障 |
| POST-005 | P2延迟保障 <= 100ms | P2时效性保障 |
| POST-006 | shm_segment_ref 非空（分配成功时） | SHM段引用有效 |
| POST-007 | winners + losers 长度 = competing_txs 长度 | 冲突裁决完整性 |
| POST-008 | new_level > previous_level（升级成功时） | 升级方向正确 |

### 4.3 不变量

- **零全局状态**：不读写任何全局变量，所有状态通过ctx参数传入
- **零堆分配**：不使用动态内存分配
- **零系统调用**：不进行文件IO、网络IO、进程管理
- **零副作用**：不修改输入参数，所有变更通过返回值和SHM写入传递
- **确定性**：相同输入必得相同输出
- **升级单向性**：路径只能升级不能降级
- **P0资源保护**：P0通道并发数严格不超过max_p0_concurrent
- **裁决公平性**：冲突裁决严格遵循三阶优先级规则

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_CTX_EMPTY` | ctx为空或缺少必需键 | `error` | "ctx is empty or missing required keys" |
| `ERR_CONFIG_INVALID` | config为空或缺少path_definitions | `error` | "config is empty or missing path_definitions" |
| `ERR_PATH_DEFINITIONS_INCOMPLETE` | 缺少P0-P3中任一等级定义 | `error` | "path_definitions must include P0, P1, P2, P3" |
| `ERR_P0_LIMIT_INVALID` | max_p0_concurrent超出范围 | `error` | "max_p0_concurrent must be in [1, 10]" |
| `ERR_TX_ID_INVALID` | tx_id长度 > 64 | `error` | "tx_id length must be <= 64" |
| `ERR_LATENCY_CLASS_INVALID` | latency_class不是L0-L3之一 | `error` | "latency_class must be L0/L1/L2/L3" |
| `ERR_PRODUCER_NOT_REGISTERED` | producer_id未注册 | `error` | "producer not found in registry" |
| `ERR_MASK_INVALID` | demand_mask超出32位范围 | `error` | "demand_mask must be a 32-bit unsigned integer" |
| `ERR_TX_NOT_ASSIGNED` | tx_id未分配路径 | `error` | "transaction has no assigned path" |
| `ERR_PATH_LEVEL_INVALID` | new_level不是P0-P3之一 | `error` | "path_level must be P0/P1/P2/P3" |
| `ERR_UPGRADE_DIRECTION_INVALID` | new_level不高于当前等级 | `error` | "can only upgrade to a higher path level" |
| `ERR_COMPETING_TXS_INSUFFICIENT` | 竞争交易数 < 2 | `error` | "competing_txs must contain at least 2 transactions" |
| `ERR_UPGRADE_DISABLED` | 路径升级被禁用 | `error` | "path upgrade is disabled" |
| `ERR_P0_CAPACITY_EXCEEDED` | P0通道并发数超限 | `error` | "P0 channel capacity exceeded" |
| `ERR_NO_SHM_AVAILABLE` | 无可用SHM段 | `error` | "no SHM segment available for the requested path" |

---

## 5. 测试场景

### 5.1 正常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-001 | 标准ctx + 标准config（含P0-P3） | 默认配置 | init返回status="ok"，paths_initialized=["P0","P1","P2","P3"] |
| TC-002 | L0交易 | 默认配置 | assign返回path_level="P0"，latency_guarantee_ms <= 1.0 |
| TC-003 | L1交易 | 默认配置 | assign返回path_level="P1"，latency_guarantee_ms <= 10.0 |
| TC-004 | L2交易 | 默认配置 | assign返回path_level="P2"，latency_guarantee_ms <= 100.0 |
| TC-005 | L3交易 | 默认配置 | assign返回path_level="P3"，latency_guarantee_ms为None |
| TC-006 | P2交易升级至P1 | upgrade_allowed=True | upgrade返回new_level="P1"，previous_level="P2" |
| TC-007 | 3个竞争P1交易（不同信用分） | 默认配置 | conflict_resolve返回winners长度=1，最高信用分者获胜 |

### 5.2 边界路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-008 | max_p0_concurrent=1 | 最小P0并发 | init返回status="ok" |
| TC-009 | max_p0_concurrent=10 | 最大P0并发 | init返回status="ok" |
| TC-010 | P0交易达到max_p0_concurrent上限 | max_p0_concurrent=1 | 第2个P0交易返回ERR_P0_CAPACITY_EXCEEDED |
| TC-011 | P3升级至P0 | upgrade_allowed=True | upgrade返回new_level="P0" |
| TC-012 | 2个竞争交易（同等级同信用分） | 默认配置 | 先提交者（submit_timestamp更小）获胜 |
| TC-013 | 2个竞争交易（同等级不同信用分） | 默认配置 | 信用分更高者获胜 |
| TC-014 | 2个竞争交易（不同等级） | 默认配置 | 高等级者获胜 |
| TC-015 | P1资源不足时L1交易 | P1专用段满 | assign返回降级至P2，记录降级事件 |

### 5.3 异常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-016 | ctx={} | 任意 | init返回status="error"，error含ERR_CTX_EMPTY |
| TC-017 | config={}（无path_definitions） | 任意 | init返回status="error"，error含ERR_CONFIG_INVALID |
| TC-018 | path_definitions仅含P0/P1 | 缺少P2/P3 | init返回status="error"，error含ERR_PATH_DEFINITIONS_INCOMPLETE |
| TC-019 | transaction.latency_class="L5" | 任意 | assign返回status="error"，error含ERR_LATENCY_CLASS_INVALID |
| TC-020 | transaction.producer_id="UNREGISTERED" | 任意 | assign返回status="error"，error含ERR_PRODUCER_NOT_REGISTERED |
| TC-021 | transaction.demand_mask=2**33 | 任意 | assign返回status="error"，error含ERR_MASK_INVALID |
| TC-022 | 升级P1至P2（降级方向） | 任意 | upgrade返回status="error"，error含ERR_UPGRADE_DIRECTION_INVALID |
| TC-023 | 升级但upgrade_allowed=False | upgrade_allowed=False | upgrade返回status="error"，error含ERR_UPGRADE_DISABLED |
| TC-024 | competing_txs长度=1 | 任意 | conflict_resolve返回status="error"，error含ERR_COMPETING_TXS_INSUFFICIENT |
| TC-025 | new_level="P5" | 任意 | upgrade返回status="error"，error含ERR_PATH_LEVEL_INVALID |

### 5.4 性能路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-026 | 10000个L3交易连续分配 | 默认配置 | 全部assign在5秒内完成 |
| TC-027 | 100个竞争交易冲突解决 | 默认配置 | conflict_resolve耗时 < 50ms |
| TC-028 | 连续1000次路径升级/降级操作 | 默认配置 | 无SHM段泄漏，资源正确回收 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `time` | 时间戳获取与延迟计算 | Python 3.8+ |
| `hashlib` | 配置摘要计算 | Python 3.8+ |

### 6.2 外部依赖

**零外部依赖**。不依赖任何第三方库。

### 6.3 蓝图间依赖

| 依赖蓝图 | 依赖类型 | 依赖内容 |
|----------|----------|----------|
| BP-0003（SHM矢量管理器） | 功能依赖 | SHM段的分配、释放、利用率查询接口 |
| BP-0038（调度器） | 功能协作 | 调度器的路径分配建议、任务路由决策 |
| BP-0040（交易指标） | 数据读取 | 路径延迟指标、带宽利用率指标 |

### 6.4 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(N * log N) per conflict resolution | N为竞争交易数 |
| 内存 | <= 256KB | 路由器状态与路径资源池 |
| 栈帧 | <= 8KB | 单次函数调用最大栈深 |
| 执行时限 | assign <= 1ms（P0保障） | 路径分配必须满足延迟保障 |

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
| SHM权限合规 | YES | 仅访问授权的SHM段，P0独占段严格隔离 |
| 栈帧安全 | YES | 不存在递归，最大栈深 <= 8KB |
| 零堆分配 | YES | 不使用动态内存分配 |
| 零OOP | YES | 不使用类定义、继承、self引用 |
| 零系统调用 | YES | 不调用操作系统API |

### 7.2 stdout特许

本路由器不直接向stdout输出。所有诊断信息通过返回值dict传递。

---

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证，确保15条前置条件全部有对应错误码 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现，禁止class/struct/self，所有状态通过ctx参数传递 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例，覆盖TC-001至TC-028全部28个用例 |
| 安全扫描 | 验证三大零原则、SHM权限合规、P0独占段隔离性、升级单向性 |
| 授权文件输出 | 按5字段JSON结构生成授权文件（auth_header, design_meta, source_code, compiled_binary, verification_report） |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据（BP-0041-PATH-ROUTER, v1.0.0, Genesis-Worker-001） |
| `design_meta` | 第3节函数设计（4个函数签名、参数规格、返回值规格、四等级路径定义、冲突解决算法、升级规则） |
| `source_code` | AICoder根据蓝图意图生成的纯函数Python实现 |
| `compiled_binary` | PYC编译产物（base64编码） |
| `verification_report` | 基于第5节28个测试场景的验收测试报告 |

### 8.3 特别审查要求

1. **路径体系对齐**：确保P0-P3四等级路径定义与law-trade-mask.md第12条及law-mall-identity.md第6条完全一致（名称、SHM策略、延迟保障、优先级）
2. **冲突解决对齐**：确保三阶优先级规则与law-trade-mask.md第14条完全一致（路径等级 > 信用分 > 提交时间）
3. **P0隔离性**：确保P0独占SHM段严格隔离，不被其他等级交易访问
4. **升级单向性**：确保路径升级只能从低到高（P3->P2->P1->P0），禁止降级
5. **延迟保障**：确保P0分配操作本身耗时不超过1ms，不成为延迟瓶颈
6. **资源回收**：确保路径变更（升级/降级/完成）时SHM段正确回收，无泄漏

---

*蓝图审核状态：待审核*
*设计者：Genesis-Worker-001*
*AICoder评估：待外置AICoder审查*
