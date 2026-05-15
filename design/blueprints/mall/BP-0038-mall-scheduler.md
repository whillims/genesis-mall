# 蓝图文档（定稿版）：BP-0038-MALL-SCHEDULER -- 商场主调度器

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0038-MALL-SCHEDULER |
| 蓝图名称 | 商场主调度器 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-12 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-0（创世纪验证级） |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场主调度器是商场的核心控制中枢，负责以下关键职责：

1. **任务路由**：接收待调度任务，根据掩码匹配结果和路径等级将任务分配至对应Worker
2. **算力分配**：基于调度因子向量评估Worker适配度，动态分配算力资源
3. **产出速率调节**：根据SHM带宽利用率和系统负载，连续调节Worker产出速率（output_rate in [0.0, 1.0]）
4. **P0-P3路径分级调度**：实现四等级交易路径的优先级调度，确保关键交易的时效性保障
5. **主动建议**：基于指标分析发布建议事件，供消费者或管理者参考

调度器不直接执行业务计算，而是作为纯决策函数，将调度决策以dict结构返回，由商场主循环执行。

### 2.2 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| Worker状态表 | SHM hb域 | Worker存活、负载、注册信息 |
| 函数注册表 | SHM auth域 | 已注册函数的元数据与能力声明 |
| 心跳指标 | SHM hb域 | Worker心跳时间戳、健康状态 |
| 监控指标 | SHM metrics域 | CPU、内存、带宽、延迟等运行时指标 |
| 任务队列 | SHM bus域 | 待调度任务列表 |

### 2.3 数据去向

| 数据项 | 去向 | 说明 |
|--------|------|------|
| 调度决策 | 返回值dict | 包含任务分配、速率调节、建议事件 |
| 速率调节信号 | SHM bus域 | 写入Worker控制段 |
| 建议事件 | SHM bus域 | 写入建议事件队列 |

### 2.4 来源锁定

本调度器的数据来源锁定为SHM矢量空间，不接收外部网络输入或文件输入。

---

## 3. 函数设计

### 3.1 函数签名

```python
def mall_scheduler_init(
    ctx: dict,
    config: dict
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: WRITE_SHM_CONFIG
    初始化调度器状态，写入SHM配置段。
    """

def mall_scheduler_tick(
    ctx: dict,
    state: dict
) -> dict:
    """
    [复杂度]: O(W * T) 其中W为Worker数，T为待调度任务数
    [安全等级]: READ_SHM + WRITE_SHM_BUS
    执行一个调度周期：采集指标、评估因子、分配任务、调节速率、发布建议。
    """

def mall_scheduler_dispatch(
    ctx: dict,
    task: dict
) -> dict:
    """
    [复杂度]: O(W) 其中W为候选Worker数
    [安全等级]: READ_SHM + WRITE_SHM_BUS
    对单个任务执行路由决策：掩码匹配、路径分配、Worker选择。
    """

def mall_scheduler_rebalance(
    ctx: dict
) -> dict:
    """
    [复杂度]: O(W * P) 其中W为Worker数，P为路径等级数
    [安全等级]: READ_SHM + WRITE_SHM_BUS
    触发全局再平衡：重新评估所有Worker的调度因子，调整任务分配和速率。
    """
```

### 3.2 参数规格表

#### mall_scheduler_init

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空，含shm_root, registry_ref | 上下文字典 |
| config | dict | 是 | 含tick_interval_ms, weights, path_config | 调度器配置 |

**config结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| tick_interval_ms | int | [10, 10000] | 调度周期间隔（毫秒） |
| weights | dict | 四项权重之和为1.0 | 调度因子权重 {w1, w2, w3, w4} |
| path_config | dict | 含P0-P3配置 | 路径等级配置 |
| max_p0_concurrent | int | [1, 10] | P0通道最大并发数 |
| output_rate_range | list | [min, max] in [0.0, 1.0] | 速率调节范围 |

#### mall_scheduler_tick

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| state | dict | 是 | 含last_tick_time, tick_seq, worker_scores | 当前调度器状态快照 |

#### mall_scheduler_dispatch

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| task | dict | 是 | 含task_id, producer_id, demand_mask, latency_class | 待调度任务 |

**task结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| task_id | str | 长度 <= 64 | 任务唯一标识 |
| producer_id | str | 已注册 | 生产者ID |
| demand_mask | int | 32位无符号 | 需求掩码 |
| latency_class | str | L0/L1/L2/L3 | 时效等级 |
| priority | int | [0, 100] | 任务优先级 |

#### mall_scheduler_rebalance

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |

### 3.3 数据来源定义

| SHM域 | 数据类型 | 预期规模 | 访问模式 |
|-------|----------|----------|----------|
| hb (心跳域) | json | <= 256条Worker记录 | 只读 |
| auth (授权域) | json | <= 1024条函数注册 | 只读 |
| bus (总线域) | json | 任务队列 + 控制信号 | 读写 |
| metrics (指标域) | float64/int64 | <= 64个指标 | 只读 |

### 3.4 返回值规格

#### mall_scheduler_init 返回值

```python
{
    "status": str,              # "ok" / "error"
    "scheduler_id": str,        # 调度器实例ID
    "config_hash": str,         # 配置摘要
    "timestamp": float,         # 初始化时间戳
    "error": str                # 错误信息（仅status="error"时）
}
```

#### mall_scheduler_tick 返回值

```python
{
    "status": str,              # "ok" / "error"
    "tick_seq": int,            # 调度周期序号
    "tasks_dispatched": int,    # 本周期分配任务数
    "rate_adjustments": list,   # 速率调节记录 [{worker_id, old_rate, new_rate, reason}]
    "suggestions": list,        # 建议事件列表
    "worker_scores": dict,      # Worker调度因子评分 {worker_id: score}
    "path_utilization": dict,   # 路径利用率 {P0: float, P1: float, P2: float, P3: float}
    "timestamp": float,
    "error": str
}
```

#### mall_scheduler_dispatch 返回值

```python
{
    "status": str,              # "ok" / "error"
    "task_id": str,             # 任务ID
    "assigned_worker": str,     # 分配的Worker ID
    "path_level": str,          # 分配的路径等级 P0/P1/P2/P3
    "score": float,             # 调度因子评分
    "estimated_latency_ms": float,  # 预估延迟
    "timestamp": float,
    "error": str
}
```

#### mall_scheduler_rebalance 返回值

```python
{
    "status": str,              # "ok" / "error"
    "rebalanced_workers": int,  # 再平衡的Worker数
    "migrations": list,         # 任务迁移记录 [{task_id, from_worker, to_worker}]
    "rate_changes": list,       # 速率变更记录
    "timestamp": float,
    "error": str
}
```

### 3.5 关键算法：调度因子向量

调度因子向量用于评估Worker对任务的适配度：

```
score = w1 * (1 / cpu_load)
      + w2 * (memory_avail / shm_demand)
      + w3 * (1 / network_latency)
      + w4 * historical_success_rate
```

其中：
- `cpu_load`: Worker当前CPU负载率 [0.0, 1.0]
- `memory_avail`: Worker可用内存（字节）
- `shm_demand`: 任务SHM需求（字节）
- `network_latency`: Worker通信延迟（毫秒）
- `historical_success_rate`: Worker历史任务成功率 [0.0, 1.0]
- `w1, w2, w3, w4`: 权重因子，由config传入，默认均为0.25

### 3.6 关键算法：P0-P3路径分级调度

```
路径分配策略：
  L0 -> P0（紧急通道）：独占SHM段，最高闸口优先级，<=1ms延迟保障
  L1 -> P1（快速通道）：专用SHM段，高闸口优先级，<=10ms延迟保障
  L2 -> P2（标准通道）：共享SHM段，标准闸口，<=100ms延迟保障
  L3 -> P3（经济通道）：共享SHM段，最低优先级，无延迟保障

冲突解决规则：
  1. 路径等级更高者优先（P0 > P1 > P2 > P3）
  2. 同等级时，时间信用分更高者优先
  3. 同信用时，先提交者优先（FIFO）
```

### 3.7 关键算法：产出速率调节

```
调节规则（每个tick周期评估）：
  IF shm_bandwidth > 0.80 THEN output_rate -= 0.1
  IF shm_bandwidth > 0.95 THEN output_rate -= 0.3
  IF consumer_ack_delay > t1 * 0.8 THEN output_rate -= 0.2
  IF target_consumer_offline THEN output_rate = 0.0
  IF shm_bandwidth < 0.40 THEN output_rate += 0.1（不超过1.0）
  IF emergency_task_inserted(P0) THEN non_emergency_workers -= 0.2

output_rate 范围：[0.0, 1.0]
默认值：1.0（全速）
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | ctx 非空且包含必需键 (shm_root, registry_ref) | `ERR_CTX_EMPTY` | `error` |
| PRE-002 | config 非空且包含必需字段 | `ERR_CONFIG_INVALID` | `error` |
| PRE-003 | tick_interval_ms 在 [10, 10000] 范围内 | `ERR_TICK_INTERVAL_INVALID` | `error` |
| PRE-004 | weights 四项权重之和为1.0（容差 +/- 0.01） | `ERR_WEIGHTS_INVALID` | `error` |
| PRE-005 | task.demand_mask 为32位无符号整数 | `ERR_MASK_INVALID` | `error` |
| PRE-006 | task.latency_class 为 L0/L1/L2/L3 之一 | `ERR_LATENCY_CLASS_INVALID` | `error` |
| PRE-007 | task.producer_id 已在注册表中注册 | `ERR_PRODUCER_NOT_REGISTERED` | `error` |
| PRE-008 | max_p0_concurrent 在 [1, 10] 范围内 | `ERR_P0_LIMIT_INVALID` | `error` |
| PRE-009 | state.last_tick_time 为有效时间戳 | `ERR_STATE_INVALID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | tasks_dispatched >= 0 | 非负整数 |
| POST-003 | rate_adjustments 中每条记录的 new_rate in [0.0, 1.0] | 速率范围合法 |
| POST-004 | path_level 必须为 P0/P1/P2/P3 之一 | 路径等级合法 |
| POST-005 | score >= 0.0 | 调度因子非负 |
| POST-006 | tick_seq 严格递增 | 周期序号单调递增 |
| POST-007 | 所有输出列表长度 <= 1024 | 防止输出爆炸 |

### 4.3 不变量

- **零全局状态**：不读写任何全局变量，所有状态通过ctx和state参数传入
- **零堆分配**：不使用动态内存分配，所有数据结构在ctx中预分配
- **零系统调用**：不进行文件IO、网络IO、进程管理
- **零副作用**：不修改输入参数ctx和state，所有变更通过返回值传递
- **确定性**：相同ctx + 相同state + 相同task 必得相同输出
- **调度公平性**：同等级任务的调度顺序遵循FIFO，不因内部状态产生饥饿

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_CTX_EMPTY` | ctx为空或缺少必需键 | `error` | "ctx is empty or missing required keys" |
| `ERR_CONFIG_INVALID` | config为空或缺少必需字段 | `error` | "config is empty or missing required fields" |
| `ERR_TICK_INTERVAL_INVALID` | tick_interval_ms不在合法范围 | `error` | "tick_interval_ms must be in [10, 10000]" |
| `ERR_WEIGHTS_INVALID` | 权重之和不等于1.0 | `error` | "weights must sum to 1.0" |
| `ERR_MASK_INVALID` | demand_mask超出32位范围 | `error` | "demand_mask must be a 32-bit unsigned integer" |
| `ERR_LATENCY_CLASS_INVALID` | latency_class不是L0-L3之一 | `error` | "latency_class must be L0/L1/L2/L3" |
| `ERR_PRODUCER_NOT_REGISTERED` | producer_id未注册 | `error` | "producer not found in registry" |
| `ERR_P0_LIMIT_INVALID` | max_p0_concurrent超出范围 | `error` | "max_p0_concurrent must be in [1, 10]" |
| `ERR_STATE_INVALID` | state时间戳无效 | `error` | "state contains invalid timestamp" |
| `ERR_NO_ELIGIBLE_WORKER` | 无可用Worker匹配任务 | `error` | "no eligible worker found for task" |
| `ERR_P0_CAPACITY_EXCEEDED` | P0通道并发数超限 | `error` | "P0 channel capacity exceeded" |
| `ERR_REBALANCE_IN_PROGRESS` | 再平衡正在进行中 | `error` | "rebalance already in progress" |

---

## 5. 测试场景

### 5.1 正常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-001 | 标准ctx + 标准config | 默认权重均0.25 | init返回status="ok"，scheduler_id非空 |
| TC-002 | 含3个Worker的ctx + 含1个L2任务的task | 默认配置 | dispatch返回status="ok"，path_level="P2"，assigned_worker非空 |
| TC-003 | 含5个Worker的ctx + 含2个L0任务的task列表 | 默认配置 | tick返回tasks_dispatched=2，path_level均为"P0" |
| TC-004 | 含Worker的ctx（shm_bandwidth=0.9） | 默认配置 | tick返回rate_adjustments非空，new_rate < 1.0 |

### 5.2 边界路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-005 | tick_interval_ms=10 | 最小周期 | init返回status="ok" |
| TC-006 | tick_interval_ms=10000 | 最大周期 | init返回status="ok" |
| TC-007 | 单Worker + 单任务 | 默认配置 | dispatch返回score > 0，assigned_worker为该Worker |
| TC-008 | 100个Worker + 1000个任务 | 默认配置 | tick在合理时间内完成，tasks_dispatched > 0 |
| TC-009 | shm_bandwidth=0.0 | 默认配置 | rate_adjustments中new_rate趋向1.0 |
| TC-010 | shm_bandwidth=1.0 | 默认配置 | rate_adjustments中new_rate大幅降低 |
| TC-011 | max_p0_concurrent=1 + 2个P0任务 | 默认配置 | 第1个P0成功，第2个返回ERR_P0_CAPACITY_EXCEEDED |

### 5.3 异常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-012 | ctx={} | 任意 | init返回status="error"，error含ERR_CTX_EMPTY |
| TC-013 | config={} | 任意 | init返回status="error"，error含ERR_CONFIG_INVALID |
| TC-014 | weights={w1:0.5, w2:0.5, w3:0.0, w4:0.0}（和为1.0） | 合法 | init返回status="ok" |
| TC-015 | weights={w1:1.0, w2:0.0, w3:0.0, w4:0.0}（和为1.0但极端） | 合法 | init返回status="ok"，但调度行为偏向CPU |
| TC-016 | task.latency_class="L5" | 任意 | dispatch返回status="error"，error含ERR_LATENCY_CLASS_INVALID |
| TC-017 | task.producer_id="UNREGISTERED" | 任意 | dispatch返回status="error"，error含ERR_PRODUCER_NOT_REGISTERED |
| TC-018 | task.demand_mask=2**33 | 任意 | dispatch返回status="error"，error含ERR_MASK_INVALID |
| TC-019 | 无可用Worker | 任意 | dispatch返回status="error"，error含ERR_NO_ELIGIBLE_WORKER |

### 5.4 性能路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-020 | 256个Worker + 10000个任务 | 默认配置 | 单次tick耗时 < 500ms |
| TC-021 | 连续1000次tick调用 | 默认配置 | 无内存泄漏，tick_seq严格递增 |
| TC-022 | 100次rebalance调用 | 默认配置 | 每次rebalance耗时 < 200ms |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | 配置摘要计算 | Python 3.8+ |
| `time` | 时间戳获取 | Python 3.8+ |
| `math` | 数学运算 | Python 3.8+ |

### 6.2 外部依赖

**零外部依赖**。不依赖任何第三方库。

### 6.3 蓝图间依赖

| 依赖蓝图 | 依赖类型 | 依赖内容 |
|----------|----------|----------|
| BP-0003（SHM矢量管理器） | 数据读取 | SHM矢量空间的读写接口 |
| BP-0009（注册表） | 数据读取 | 函数注册表查询接口 |
| BP-0011（心跳） | 数据读取 | Worker心跳与健康状态 |
| BP-0036（监控） | 数据读取 | 运行时指标（CPU、内存、带宽） |
| BP-0040（交易指标） | 数据读取 | 三层交易指标 |
| BP-0041（路径路由器） | 功能协作 | P0-P3路径分配决策 |

### 6.4 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(W * T) per tick | W=Worker数，T=任务数 |
| 内存 | <= 512KB | 调度器状态与中间数据 |
| 栈帧 | <= 16KB | 单次函数调用最大栈深 |
| 执行时限 | tick <= 500ms | 单个调度周期上限 |

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
| SHM权限合规 | YES | 仅读取授权的SHM域（hb/auth/metrics），写入授权的bus域 |
| 栈帧安全 | YES | 不存在递归和深度调用链，最大栈深 <= 16KB |
| 零堆分配 | YES | 不使用动态内存分配 |
| 零OOP | YES | 不使用类定义、继承、self引用 |
| 零系统调用 | YES | 不调用操作系统API |

### 7.2 stdout特许

本调度器不直接向stdout输出。所有诊断信息通过返回值dict传递。

---

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证，确保所有前置条件有对应错误码 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现，禁止class/struct/self，所有状态通过ctx参数传递 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例，覆盖TC-001至TC-022全部22个用例 |
| 安全扫描 | 验证三大零原则（零全局、零堆、零系统调用）、SHM权限合规、输入消毒完整性 |
| 授权文件输出 | 按5字段JSON结构生成授权文件（auth_header, design_meta, source_code, compiled_binary, verification_report） |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据（BP-0038-MALL-SCHEDULER, v1.0.0, Genesis-Worker-001） |
| `design_meta` | 第3节函数设计（4个函数签名、参数规格、返回值规格、调度因子算法、路径分级算法、速率调节算法） |
| `source_code` | AICoder根据蓝图意图生成的纯函数Python实现 |
| `compiled_binary` | PYC编译产物（base64编码） |
| `verification_report` | 基于第5节22个测试场景的验收测试报告 |

### 8.3 特别审查要求

1. **调度因子向量**：确保权重归一化校验逻辑正确（容差 +/- 0.01）
2. **路径分级**：确保P0-P3映射与law-mall-identity.md第6条完全一致
3. **速率调节**：确保output_rate始终在[0.0, 1.0]范围内，调节步长与law-mall-identity.md第18条一致
4. **建议事件**：确保建议事件结构与law-mall-identity.md第14条完全一致
5. **公平性**：确保同级任务FIFO调度，不产生饥饿

---

*蓝图审核状态：待审核*
*设计者：Genesis-Worker-001*
*AICoder评估：待外置AICoder审查*
