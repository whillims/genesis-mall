# 微型商场 · 审计Worker设计蓝图

> **文档编号**: MW-AUD-001  
> **版本**: v1.0  
> **日期**: 2026-05-09  
> **对应规范**: 微型商场设计规范 v1.2  
> **设计层级**: Python函数级  
> **关联文档**: 算力法条、调度Worker蓝图、安全Worker蓝图

---

## 1. Worker职责定义

审计Worker是微型商场的"经济系统记账员"与"历史见证者"，承担以下核心职责：

- **行为日志记录**: 记录所有Worker的生命周期事件、SHM访问、算力消耗
- **算力费用核算**: 根据算力法条，核算每次调度的算力费用与生产者收益
- **追溯链条构建**: 为每个任务构建不可篡改的追溯链（从消费者需求到生产者产出）
- **动态法条执行**: 执行算力评价体系法条，防止算力暴政
- **报表生成**: 为商场治理提供数据支撑，包括Worker绩效、资源利用率、异常统计

---

## 2. 入口参数设计

```python
def audit_worker_entry(
    shm_config: dict,           # SHM共享内存配置
    log_buffer_size: int,       # 环形日志缓冲区大小（条目数）
    ledger_storage_path: str,   # 账本持久化路径（如适用）
    billing_policy: dict,       # 计费策略（来自算力法条）
    trace_depth: int,          # 追溯链最大深度
    report_interval_ms: int,   # 报表生成间隔
    scheduler_callback: callable,  # 调度Worker回调
    security_callback: callable     # 安全Worker回调
) -> int:
    """
    审计Worker主入口函数

    章节对应:
        - 第1节: 职责定义
        - 第3节: 状态机实现
        - 第4节: 核心函数调用链

    返回值:
        0: 正常退出
        1: 日志缓冲区初始化失败
        2: 计费策略解析错误
        3: 账本损坏
    """
```

---

## 3. 状态机设计

```
[INIT] --加载计费策略--> [POLICY_LOAD] --初始化日志环--> [LOGGING]
   |                                                              |
   |<------------------ 策略更新 ---------------------------------|
   v                                                              v
[BILLING] <--调度事件-- [LOGGING] <--安全事件-- [TRACING]
   |                           ^    |                              |
   |--费用异常--> [ARBITRAGE]   |    |--追溯请求--> [CHAIN_BUILD]  |
   |                           |    |                              |
   |--周期到达--> [REPORTING]   |    |--深度超限--> [ARCHIVING]    |
   |                           |    |                              |
   v                           |    v                              |
[ARCHIVING] --归档完成--> [LOGGING] --存储满--> [ROTATION]        |
   |                                                              |
   v                                                              v
[SHUTDOWN] <----------------- 收到终止信号 -----------------------|
```

状态说明：
- **INIT**: 加载配置，建立与其他Worker的回调通道
- **POLICY_LOAD**: 解析算力法条计费策略
- **LOGGING**: 正常运行，接收并记录事件
- **BILLING**: 处理算力费用核算
- **ARBITRAGE**: 费用争议仲裁（如算力报价与评估不符）
- **TRACING**: 处理追溯请求
- **CHAIN_BUILD**: 构建完整追溯链
- **ARCHIVING**: 深度超限或周期到达，归档历史数据
- **ROTATION**: 环形缓冲区满，轮转旧日志
- **REPORTING**: 生成周期报表
- **SHUTDOWN**: 优雅关闭，确保所有日志落盘

---

## 4. 核心函数设计（函数级）

### 4.1 行为日志记录

```python
def log_record(
    event_type: str,             # 事件类型: "worker_register" | "task_dispatch" | "lease_renew" | "worker_unregister" | "security_alert"
    actor_id: str,              # 行为主体Worker ID
    target_id: str,             # 行为对象Worker ID（如适用）
    event_data: dict,           # 事件结构化数据
    timestamp_ns: int,          # 时间戳（纳秒级）
    shm_access_vector: list,    # 涉及的SHM访问向量
    signature: bytes             # 行为主体签名（防抵赖）
) -> dict:
    """
    行为日志记录函数

    职责: 将商场内所有关键事件写入环形日志缓冲区
          每条日志包含完整上下文，支持后续追溯与核算

    返回值 dict:
        {
            "log_id": str,               # 日志唯一ID
            "sequence_number": int,      # 全局序列号（单调递增）
            "buffer_slot": int,          # 环形缓冲区槽位
            "hash_chain": bytes,         # 与前一条日志的哈希链
            "status": "recorded" | "dropped" | "corrupted"
        }

    测试方法:
        1. 以10000事件/秒速率写入，验证无丢包、序列号连续
        2. 模拟缓冲区满，验证旧日志被轮转且不破坏哈希链
        3. 篡改单条日志后验证哈希链断裂可被检测
        4. 验证日志包含完整的SHM访问向量，支持空间追溯
    """
```

### 4.2 算力费用核算

```python
def cost_calculate(
    task_id: str,                # 任务唯一标识
    function_ref: str,           # 执行的函数引用
    compute_actual: int,         # 实际消耗算力单元
    compute_estimated: int,      # 预估算力单元
    lease_count: int,           # 续约次数
    worker_id: str,             # 执行Worker ID
    consumer_id: str,           # 消费者Worker ID
    billing_policy_id: str      # 适用的计费策略ID
) -> dict:
    """
    算力费用核算函数

    职责: 根据算力法条核算单次任务的费用：
          - 基础费用 = 实际算力 × 单位算力价格
          - 预估偏差罚金 = |实际-预估| × 偏差费率（防止虚假报价）
          - 续约附加费 = 续约次数 × 续约费率
          - 动态法条调节 = 根据商场整体负载调整系数

    返回值 dict:
        {
            "status": "calculated" | "disputed" | "pending",
            "base_cost": float,
            "deviation_penalty": float,
            "renewal_surcharge": float,
            "dynamic_adjustment": float,
            "total_cost": float,         # 消费者应付
            "producer_revenue": float,   # 生产者应收（扣除商场管理费）
            "mall_fee": float,         # 商场平台费
            "audit_log_id": str
        }

    测试方法:
        1. 实际算力=预估算力，验证偏差罚金为0
        2. 实际算力远超预估，验证偏差罚金按法条比例收取
        3. 商场高负载时段，验证动态调整系数>1.0
        4. 验证费用低于评估过程产生的算力费用时，按法条拒绝交易
    """
```

### 4.3 追溯链条构建

```python
def trace_chain_build(
    task_id: str,                # 目标任务ID
    depth: int,                  # 追溯深度
    include_data_lineage: bool,  # 是否包含数据血缘
    include_compute_lineage: bool # 是否包含算力血缘
) -> dict:
    """
    追溯链条构建函数

    职责: 从消费者需求出发，逆向追溯完整的生产链路：
          - 消费者需求 -> 调度决策 -> 生产者Worker -> 函数库引用 -> AICoder编译记录
          - 正向验证: 产出数据是否确实由该链路生成
          - 算力血缘: 哪些算力单元参与了该任务

    返回值 dict:
        {
            "chain_id": str,
            "chain_hash": bytes,         # 整个链条的默克尔根哈希
            "nodes": list,               # 链条节点 [{"type", "id", "timestamp", "hash"}, ...]
            "data_lineage": list,        # 数据血缘路径
            "compute_lineage": list,   # 算力血缘路径
            "integrity_verified": bool,  # 完整性校验结果
            "audit_log_id": str
        }

    测试方法:
        1. 构建深度为10的追溯链，验证所有节点哈希连续
        2. 篡改链条中间某节点数据，验证integrity_verified=False
        3. 验证数据血缘可追溯到原始输入数据的SHM地址
        4. 验证算力血缘可追溯到具体Worker和算力单元
    """
```

### 4.4 动态法条执行

```python
def dynamic_policy_enforce(
    policy_type: str,            # 法条类型: "compute_anti_tyranny" | "fair_billing" | "resource_protection"
    market_state: dict,           # 当前商场状态（负载、Worker数、任务队列）
    proposal: dict               # 待执行的政策提案
) -> dict:
    """
    动态法条执行函数

    职责: 执行算力评价体系动态法条，防止算力暴政：
          - 算力反暴政法条: 当某Worker算力占比超过阈值时，自动限制其新租约
          - 公平计费法条: 确保交易不得低于评估过程产生的算力费用
          - 资源保护法条: 对低阶商场（微型商场）进行生态保护和污染防范

    返回值 dict:
        {
            "policy_applied": bool,
            "enforcement_actions": list,  # 执行动作列表
            "affected_tasks": list,       # 受影响的任务ID
            "compensation_plan": dict,    # 补偿方案（如适用）
            "audit_log_id": str
        }

    测试方法:
        1. 模拟某Worker占据80%算力，验证反暴政法条限制其新租约
        2. 验证任何交易报价低于评估算力费用时，交易被拒绝
        3. 模拟微型商场资源不足，验证保护法条触发资源倾斜
        4. 验证法条执行记录完整写入审计日志
    """
```

### 4.5 周期报表生成

```python
def report_generate(
    report_type: str,            # 报表类型: "daily" | "weekly" | "worker_performance" | "mall_health"
    time_range: tuple,           # 时间范围 (start_ns, end_ns)
    aggregation_level: str,     # 聚合粒度: "task" | "worker" | "function"
    format_spec: str             # 输出格式: "json" | "csv" | "shm_struct"
) -> dict:
    """
    周期报表生成函数

    职责: 为商场治理和人类架构师提供多维度数据报表：
          - Worker绩效: 成功率、平均算力消耗、消费者评分
          - 资源利用率: 算力池负载曲线、SHM使用效率
          - 异常统计: 安全事件、任务失败、算力争议
          - 经济流动: 算力费用总额、生产者收益分布

    返回值 dict:
        {
            "report_id": str,
            "record_count": int,
            "aggregations": dict,        # 聚合结果
            "anomalies_highlighted": list, # 异常高亮项
            "shm_report_addr": int,      # 报表写入的SHM地址
            "audit_log_id": str
        }

    测试方法:
        1. 生成包含100万条日志的日报，验证聚合结果准确
        2. 验证报表中的异常高亮项与日志中的安全事件一一对应
        3. 报表写入SHM后，验证消费者Worker可正常读取
        4. 验证报表生成过程不影响实时日志记录性能
    """
```

---

## 5. 与其他Worker的交互协议

### 5.1 与调度Worker交互
- **task_dispatch后**: 接收调度Worker的调度事件，记录算力分配
- **lease_renew时**: 接收续约事件，更新算力消耗流水
- **worker_unregister时**: 接收注销事件，结算该Worker的最终账单

### 5.2 与安全Worker交互
- **security_alert时**: 接收安全Worker的威胁确认通知，冻结相关交易结算
- **token_revoke时**: 记录令牌吊销事件，追溯该令牌的所有历史操作

### 5.3 与探针Worker交互
- **metric_collect时**: 接收探针Worker的性能指标，作为报表数据源
- **anomaly_report时**: 将探针异常与审计日志关联，构建完整事件视图

### 5.4 与创世Worker交互
- **INIT阶段**: 从创世Worker获取初始计费策略和账本状态
- **ARCHIVING时**: 重大归档事件通知创世Worker进行状态同步

---

## 6. 测试方法（设计检验标准）

| 测试场景 | 检验标准 | 通过条件 |
|---------|---------|---------|
| 日志风暴测试 | 20000事件/秒持续写入 | 无丢包，序列号连续，哈希链完整 |
| 费用核算精度 | 1000次随机任务核算 | 与手工计算误差<0.001% |
| 追溯链完整性 | 篡改中间某条日志 | 追溯链检测断裂，定位到具体槽位 |
| 反暴政法条 | 单Worker申请超过50%算力 | 新租约被拒绝，现有租约不受影响 |
| 报表一致性 | 并发生成5种报表 | 各报表间数据一致，无竞态错误 |
| 优雅关闭 | 写入过程中强制关闭 | 已确认日志全部落盘，无损坏 |

---

## 7. 代码注释与文档章节映射

```python
# [对应第2节: 入口参数设计]
def audit_worker_entry(...):
    # [对应第3节: INIT状态]
    state = State.INIT
    # [对应第5.4节: 与创世Worker交互]
    billing_policy = shm_read(shm_config["genesis_policy_channel"])
    ...
```

---

*本蓝图由创世Worker通过AICoder生成，待人类架构师审核后进入函数库。*
