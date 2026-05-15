# 微型商场 · 消费者Worker设计蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档编号**: MW-CON-001  
> **版本**: v1.0  
> **日期**: 2026-05-09  
> **对应规范**: 微型商场设计规范 v1.2  
> **设计层级**: Python函数级  
> **关联文档**: 调度Worker蓝图、审计Worker蓝图、商场模式个性化订制

---

## 1. Worker职责定义

消费者Worker是微型商场的"需求发起端"与"价值验证者"，承担以下核心职责：

- **需求提交**: 将人类用户的软件需求转化为商场可理解的函数调用请求
- **服务消费**: 通过商场调度系统消费生产者Worker提供的服务
- **结果接收**: 从SHM结果通道获取任务产出，验证完整性
- **评价反馈**: 对生产者服务质量进行评分，影响其调度优先级
- **个性化订制**: 支持消费者自行设计算力组织方式，提高创新性

---

## 2. 入口参数设计

```python
def consumer_worker_entry(
    shm_config: dict,           # SHM共享内存配置
    consumer_id: str,            # 消费者唯一标识
    auth_token: bytes,           # 安全Worker签发的认证令牌
    budget_quota: float,         # 算力预算配额（费用上限）
    preference_profile: dict,    # 偏好配置: 延迟敏感/成本敏感/质量敏感
    result_callback: callable,   # 结果到达回调
    scheduler_proxy: callable,  # 调度Worker代理接口
    audit_proxy: callable        # 审计Worker代理接口
) -> int:
    """
    消费者Worker主入口函数

    章节对应:
        - 第1节: 职责定义
        - 第3节: 状态机实现
        - 第4节: 核心函数调用链

    返回值:
        0: 正常退出
        1: 认证失败
        2: 预算配额不足
        3: 调度Worker不可达
    """
```

---

## 3. 状态机设计

```
[INIT] --认证与预算加载--> [AUTHED] --加载偏好配置--> [READY]
   |                                                              |
   |<------------------ 预算充值 --------------------------------|
   v                                                              v
[REQUESTING] --提交需求-- [READY] --结果到达-- [RECEIVING]
   |                           ^    |                              |
   |--等待调度--> [WAITING]     |    |--验证通过--> [CONSUMING]     |
   |                           |    |                              |
   |--超时--> [TIMEOUT]        |    |--验证失败--> [DISPUTING]    |
   |                           |    |                              |
   v                           |    v                              |
[RETRYING] --重试策略--> [REQUESTING] --完成--> [RATING]           |
   |                           |    |                              |
   v                           |    v                              |
[ABANDONED] <-------------- 重试耗尽 -----------------------------|
   |                                                              |
   v                                                              v
[SHUTDOWN] <----------------- 收到终止信号 -----------------------|
```

状态说明：
- **INIT**: 初始化，加载消费者身份
- **AUTHED**: 通过安全Worker认证，获取SHM访问权限
- **READY**: 准备就绪，等待提交需求
- **REQUESTING**: 正在向调度Worker提交服务请求
- **WAITING**: 请求已受理，等待分配算力和生产者
- **RECEIVING**: 结果数据到达SHM结果通道
- **CONSUMING**: 验证并消费结果数据
- **DISPUTING**: 结果验证失败，发起争议
- **RATING**: 对服务进行评分和反馈
- **TIMEOUT**: 任务超时未返回
- **RETRYING**: 根据重试策略重新提交
- **ABANDONED**: 重试耗尽，放弃该任务
- **SHUTDOWN**: 优雅关闭，结算未完成任务

---

## 4. 核心函数设计（函数级）

### 4.1 需求提交

```python
def requirement_submit(
    requirement_id: str,         # 需求唯一标识
    function_category: str,       # 功能类别: "signal_processing" | "verilog_compile" | "ui_render" | "data_analysis"
    specification: dict,          # 需求规格说明
    input_data_handle: int,      # 输入数据SHM句柄
    deadline_ms: int,            # 期望完成时限
    max_cost: float,             # 最高可接受费用
    quality_requirement: str    # 质量要求: "draft" | "standard" | "premium"
) -> dict:
    """
    需求提交函数

    职责: 将消费者需求转化为商场任务请求，包含完整的输入数据和期望规格
          支持个性化订制：消费者可指定算力组织偏好、生产者筛选条件

    返回值 dict:
        {
            "status": "submitted" | "rejected" | "budget_insufficient",
            "task_id": str,
            "estimated_cost": float,
            "estimated_duration_ms": int,
            "queue_depth": int,          # 当前队列深度
            "prepaid_amount": float,    # 预扣费用
            "audit_log_id": str
        }

    测试方法:
        1. 提交合法需求，验证task_id返回且预扣费用正确
        2. 提交超过budget_quota的需求，验证被拒绝
        3. 验证输入数据SHM句柄在提交后被调度Worker正确引用
        4. 测试个性化订制参数（如指定特定生产者）被调度Worker识别
    """
```

### 4.2 服务消费与结果接收

```python
def service_consume(
    task_id: str,                # 任务唯一标识
    result_shm_addr: int,        # 结果数据SHM地址
    result_checksum: bytes,     # 结果校验和
    timeout_ms: int             # 消费超时
) -> dict:
    """
    服务消费与结果接收函数

    职责: 从SHM结果通道读取任务产出，验证数据完整性（校验和比对）
          支持分块消费大结果数据，避免SHM占用过久

    返回值 dict:
        {
            "status": "consumed" | "checksum_mismatch" | "timeout" | "partial",
            "data_size": int,
            "blocks_received": int,
            "verification_result": bool,
            "consume_duration_ms": int,
            "audit_log_id": str
        }

    测试方法:
        1. 正常消费结果，验证校验和匹配
        2. 模拟结果数据损坏，验证checksum_mismatch
        3. 模拟SHM数据延迟到达，验证timeout处理
        4. 大结果数据分块消费，验证每块独立校验
    """
```

### 4.3 评价反馈

```python
def feedback_rate(
    task_id: str,                # 任务唯一标识
    producer_id: str,            # 生产者Worker ID
    overall_score: float,       # 综合评分 0.0-5.0
    dimension_scores: dict,      # 维度评分: {"quality", "speed", "cost_efficiency", "communication"}
    comment_hash: bytes,        # 评论内容哈希（内容存SHM）
    would_recommend: bool       # 是否推荐
) -> dict:
    """
    评价反馈函数

    职责: 消费者对生产者服务进行多维度评分，影响该生产者的调度优先级
          评价数据写入审计日志，作为商场治理的依据

    返回值 dict:
        {
            "status": "recorded" | "rejected" | "already_rated",
            "new_producer_priority": int,  # 更新后的生产者优先级
            "rating_id": str,
            "audit_log_id": str
        }

    测试方法:
        1. 正常评分后，验证生产者priority被更新
        2. 对同一任务重复评分，验证already_rated拒绝
        3. 模拟恶意低分攻击，验证安全Worker检测异常评分模式
        4. 验证评分与审计日志中的任务记录正确关联
    """
```

### 4.4 预算管理

```python
def budget_manage(
    action: str,                 # 操作: "query" | "recharge" | "set_alert" | "lock"
    amount: float,              # 金额（如适用）
    alert_threshold: float      # 告警阈值（如适用）
) -> dict:
    """
    预算管理函数

    职责: 管理消费者的算力预算配额，支持查询余额、充值、设置告警、锁定预算
          预算不足时自动阻止新需求提交

    返回值 dict:
        {
            "status": "success" | "insufficient_funds" | "locked",
            "current_balance": float,
            "pending_charges": float,    # 待结算费用
            "available_balance": float,  # 可用余额
            "alert_triggered": bool,
            "audit_log_id": str
        }

    测试方法:
        1. 充值后验证余额增加，待结算费用不变
        2. 设置告警阈值，消费至阈值时触发alert_triggered
        3. 锁定预算后验证新需求提交被拒绝
        4. 验证预算操作全部记录审计日志
    """
```

### 4.5 争议处理

```python
def dispute_raise(
    task_id: str,                # 争议任务ID
    dispute_type: str,           # 争议类型: "result_mismatch" | "overcharge" | "timeout_no_compensation" | "quality_not_met"
    evidence_shm_addr: int,      # 证据数据SHM地址
    expected_resolution: str     # 期望解决方式: "refund" | "redo" | "partial_refund"
) -> dict:
    """
    争议处理函数

    职责: 当消费者对服务不满时，发起争议请求
          争议进入审计Worker的仲裁流程，由动态法条裁定

    返回值 dict:
        {
            "dispute_id": str,
            "status": "pending" | "accepted" | "rejected",
            "arbitration_deadline": int,
            "escrow_locked": float,      # 冻结金额
            "audit_log_id": str
        }

    测试方法:
        1. 提交有效争议，验证dispute_id生成且金额被冻结
        2. 模拟虚假争议（证据与日志不符），验证被审计Worker拒绝
        3. 争议仲裁超时，验证自动按消费者有利原则处理
        4. 验证争议不影响该消费者其他正常需求提交
    """
```

---

## 5. 与其他Worker的交互协议

### 5.1 与调度Worker交互
- **REQUESTING状态**: 调用 `scheduler_proxy()` 提交task_dispatch请求
- **WAITING状态**: 接收调度Worker的任务状态更新（排队位置、预计开始时间）
- **RECEIVING状态**: 从调度Worker指定的SHM结果地址读取数据

### 5.2 与审计Worker交互
- **每次消费后**: 调用 `audit_proxy()` 记录消费行为和费用
- **争议时**: 向审计Worker提交争议申请和证据
- **预算变动**: 所有预算操作同步审计日志

### 5.3 与安全Worker交互
- **INIT阶段**: 使用auth_token通过安全Worker认证
- **异常时**: 检测到结果数据异常时，向安全Worker申请威胁评估

### 5.4 与探针Worker交互
- **RECEIVING状态**: 利用探针Worker监控结果通道的健康状态
- **TIMEOUT时**: 请求探针Worker确认是生产者故障还是通道故障

---

## 6. 测试方法（设计检验标准）

| 测试场景 | 检验标准 | 通过条件 |
|---------|---------|---------|
| 需求提交并发 | 100个消费者同时提交 | 所有需求被受理，无重复task_id |
| 预算耗尽保护 | 余额为0时提交需求 | 立即拒绝，不进入调度队列 |
| 结果完整性 | 结果数据被篡改 | checksum_mismatch，消费者不消费 |
| 评分影响调度 | 高评分生产者 vs 低评分生产者 | 后续调度中优先级差异>20% |
| 争议仲裁 | 提交overcharge争议 | 审计Worker在5秒内受理，金额冻结 |
| 个性化订制 | 指定特定算力组织方式 | 调度Worker按消费者偏好分配资源 |

---

## 7. 代码注释与文档章节映射

```python
# [对应第2节: 入口参数设计]
def consumer_worker_entry(...):
    # [对应第3节: INIT状态]
    state = State.INIT
    # [对应第5.3节: 与安全Worker交互]
    auth_result = security_verify(auth_token)
    ...
```

---

*本蓝图由创世Worker通过AICoder生成，待人类架构师审核后进入函数库。*
