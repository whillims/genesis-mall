# 蓝图：任务管理Worker（含交易监督者与交易员）

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0045  
> **蓝图名称**: Task Management Worker  
> **版本**: v1.1.0  
> **提交时间**: 2026-05-12  
> **提交者**: 创世Worker  
> **目标语言**: Python  
> **关联蓝图**: BP-0028-task-parameterization, BP-0014-config, BP-0004-exception  
> **功能域**: mall

---

## 一、蓝图概述

### 1.1 设计目标

本蓝图定义商场域内的**任务管理Worker**，作为消费者任务与商场调度之间的管理中枢。任务管理Worker内含两个核心角色：

- **交易监督者（Trade Supervisor）**：监控任务执行过程中的交易合规性，审计交易流向，执行熔断与异常处置
- **交易员（Trader）**：负责任务的提交、契约撮合、资源竞价、订单生命周期管理

### 1.2 系统定位

```
┌─────────────────────────────────────────────────────────────────┐
│                        商场层 (Market)                           │
│                                                                 │
│  ┌──────────────┐     ┌───────────────────────────────────┐    │
│  │  消费者意图   │────▶│         任务管理Worker              │    │
│  │  (Intent)    │     │  ┌─────────────┐ ┌─────────────┐  │    │
│  └──────────────┘     │  │  交易员      │ │ 交易监督者   │  │    │
│                       │  │ (Trader)    │ │ (Supervisor) │  │    │
│  ┌──────────────┐     │  └──────┬──────┘ └──────┬──────┘  │    │
│  │  生产者产出   │────▶│         │               │         │    │
│  │  (Product)   │     │         ▼               ▼         │    │
│  └──────────────┘     │  ┌─────────────┐ ┌─────────────┐  │    │
│                       │  │ 契约撮合    │ │ 交易审计    │  │    │
│  ┌──────────────┐     │  │ 订单管理    │ │ 合规检查    │  │    │
│  │  调度器       │◀────│  │ 资源竞价    │ │ 熔断处置    │  │    │
│  │  (Scheduler) │     │  └─────────────┘ └─────────────┘  │    │
│  └──────────────┘     └───────────────────────────────────┘    │
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  SHM交换区   │     │  审计链      │     │  信用总账    │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 核心原则

1. **职责分离**: 交易员负责撮合与推进，交易监督者负责审计与拦截，两者互不隶属
2. **单向性铁律**: 严格遵守交易掩码法则，所有交易单向流动（生产者→消费者）
3. **商场中立**: 任务管理Worker作为商场组件，对消费者和生产者保持绝对中立
4. **契约驱动**: 一切任务操作通过契约机制，双向批准后方可执行
5. **审计不可篡改**: 交易监督者的每条审计记录写入审计链，不可回滚

---

## 二、角色定义

### 2.1 交易员（Trader）

**定位**: 任务交易的**推进者**，负责任务从意图到履约的全链路管理。

**职责清单**:

| 编号 | 职责 | 说明 |
|------|------|------|
| T-01 | 意图接收与解析 | 接收消费者意图，验证格式与权限 |
| T-02 | 契约撮合 | 将消费者需求与生产者供给进行匹配，生成契约草案 |
| T-03 | 资源竞价 | 多消费者竞争同一资源时，按优先级与信用分进行竞价排序 |
| T-04 | 订单创建与管理 | 将撮合成功的契约转化为订单，管理订单生命周期 |
| T-05 | 履约推进 | 监控订单履约进度，协调调度器分配算力 |
| T-06 | 结算处理 | 订单完成后触发结算，更新信用总账 |
| T-07 | 参数化任务关联 | 与BP-0028参数化任务系统对接，管理重复/定时/条件执行实例 |

**权限边界**:
- ✅ 可以：创建/推进/结算订单，撮合契约，管理竞价队列
- ❌ 禁止：绕过交易监督者的合规检查，修改审计记录，直接访问生产者SHM

### 2.2 交易监督者（Trade Supervisor）

**定位**: 任务交易的**守门人**，负责全链路合规监控与异常处置。

**职责清单**:

| 编号 | 职责 | 说明 |
|------|------|------|
| S-01 | 交易合规审计 | 验证每笔交易是否符合单向性法则、掩码规则、路径分级 |
| S-02 | 契约合规审查 | 审查契约草案的合法性，拦截违规契约 |
| S-03 | 实时监控 | 监控SHM交换区的交易流量，检测异常模式 |
| S-04 | 熔断处置 | 检测到违规或异常时，执行熔断操作 |
| S-05 | 冲突仲裁 | 多方冲突时，按三方宪章的仲裁原则进行裁决 |
| S-06 | 审计链维护 | 将所有审计事件写入不可篡改的审计链 |
| S-07 | 信用评估输入 | 向信用总账提供交易行为数据，影响各方信用评分 |

**权限边界**:
- ✅ 可以：审计交易、拦截违规、触发熔断、写入审计链、提交仲裁建议
- ❌ 禁止：主动推进交易、修改订单内容、直接调度资源

### 2.3 两角色协作关系

```
                    ┌─────────────────────────────────┐
                    │         任务管理Worker            │
                    │                                 │
  消费者意图 ──────▶│  ┌──────────┐   审计结果   ┌──────────┐  │
                    │  │  交易员   │─────────────▶│ 交易监督者│  │
                    │  │ (Trader)  │◀─────────────│(Supervisor)│ │
                    │  └────┬─────┘   合规放行/   └────┬─────┘  │
                    │       │         拦截指令         │        │
                    │       ▼                         │        │
                    │  ┌──────────┐                   │        │
                    │  │ 订单队列  │                   │        │
                    │  └──────────┘                   │        │
                    └─────────────────────────────────┘
```

**协作规则**:
1. 交易员的每一步操作（创建契约/推进订单/结算）必须经过交易监督者审计
2. 交易监督者的审计结果是**建议性**的，但在安全违规场景下具有**一票否决权**
3. 两者通过SHM交换区内的专用审计通道通信，不共享内存状态

---

## 三、需求定义

### 3.1 功能需求

| 编号 | 需求描述 | 优先级 | 负责角色 |
|------|----------|--------|----------|
| R001 | 接收消费者意图并解析为结构化需求 | 必须 | 交易员 |
| R002 | 执行契约撮合，匹配需求与供给 | 必须 | 交易员 |
| R003 | 多消费者资源竞争时按优先级竞价 | 必须 | 交易员 |
| R004 | 管理订单全生命周期（创建→分配→生产→履约→结算） | 必须 | 交易员 |
| R005 | 验证交易单向性、掩码匹配、路径分级合规 | 必须 | 交易监督者 |
| R006 | 实时监控交易流量，检测异常模式 | 必须 | 交易监督者 |
| R007 | 违规交易拦截与熔断处置 | 必须 | 交易监督者 |
| R008 | 审计事件写入不可篡改审计链 | 必须 | 交易监督者 |
| R009 | 冲突仲裁，按三方宪章原则裁决 | 应该 | 交易监督者 |
| R010 | 与BP-0028参数化任务系统对接 | 应该 | 交易员 |
| R011 | 交易行为数据输入信用评估 | 应该 | 交易监督者 |
| R012 | 订单结算后更新信用总账 | 应该 | 交易员 |

### 3.2 非功能需求

| 类别 | 要求 | 指标 |
|------|------|------|
| 吞吐 | 订单处理吞吐量 | ≥1000订单/秒 |
| 延迟 | 交易审计延迟 | ≤1ms |
| 可靠性 | 审计链完整性 | 100%，不可篡改 |
| 隔离性 | 交易员与监督者故障隔离 | 单方故障不影响另一方 |
| 资源 | 内存占用 | <20MB |

---

## 四、架构设计

### 4.1 模块划分

```
任务管理Worker
├── 交易员模块 (Trader Module)
│   ├── 意图接收器 (Intent Receiver)
│   ├── 契约撮合引擎 (Contract Matching Engine)
│   ├── 竞价排序器 (Bid Sorter)
│   ├── 订单管理器 (Order Manager)
│   ├── 履约推进器 (Fulfillment Driver)
│   ├── 结算处理器 (Settlement Handler)
│   └── 参数化任务适配器 (Param Task Adapter)
├── 交易监督者模块 (Supervisor Module)
│   ├── 交易合规审计器 (Trade Compliance Auditor)
│   ├── 契约合规审查器 (Contract Compliance Reviewer)
│   ├── 实时流量监控器 (Real-time Flow Monitor)
│   ├── 熔断执行器 (Circuit Breaker)
│   ├── 冲突仲裁器 (Conflict Arbitrator)
│   └── 审计链写入器 (Audit Chain Writer)
└── 共享接口层 (Shared Interface)
    ├── SHM审计通道 (SHM Audit Channel)
    ├── 状态查询接口 (Status Query API)
    └── 事件通知总线 (Event Notification Bus)
```

### 4.2 数据流

```
消费者提交意图
      │
      ▼
┌─────────────┐
│  交易员      │
│  意图接收器  │
└──────┬──────┘
       │ 解析完成
       ▼
┌─────────────┐     审计请求     ┌─────────────┐
│  交易员      │────────────────▶│ 交易监督者   │
│  契约撮合    │◀────────────────│ 合规审查     │
└──────┬──────┘     审计结果     └─────────────┘
       │ 放行
       ▼
┌─────────────┐
│  交易员      │
│  竞价排序    │
└──────┬──────┘
       │ 竞价完成
       ▼
┌─────────────┐     审计请求     ┌─────────────┐
│  交易员      │────────────────▶│ 交易监督者   │
│  订单创建    │◀────────────────│ 订单审计     │
└──────┬──────┘     审计结果     └─────────────┘
       │ 放行
       ▼
┌─────────────┐
│  调度器      │
│  算力分配    │
└──────┬──────┘
       │ 生产者履约
       ▼
┌─────────────┐     实时监控     ┌─────────────┐
│  SHM交换区   │────────────────▶│ 交易监督者   │
│  交易流动    │                 │ 流量监控     │
└──────┬──────┘                 └──────┬──────┘
       │ 履约完成                       │ 异常检测
       ▼                                ▼
┌─────────────┐                 ┌─────────────┐
│  交易员      │                 │ 交易监督者   │
│  结算处理    │                 │ 审计链写入   │
└─────────────┘                 └─────────────┘
```

---

## 五、函数级设计

### 5.1 交易员函数

#### 函数1：意图接收与解析

```python
def trader_intent_receive(
    ctx: dict,
    intent_vector: dict,         # 消费者意图矢量
    consumer_id: str             # 消费者标识
) -> dict:
    """
    交易员接收消费者意图
    
    职责: 验证意图格式、提取需求特征、生成结构化需求
    
    返回值 dict:
    {
        "status": "accepted" | "rejected",
        "intent_id": str,
        "parsed_demand": dict,
        "error": str | None
    }
    """
```

#### 函数2：契约撮合

```python
def trader_contract_match(
    ctx: dict,
    parsed_demand: dict,         # 解析后的需求
    supply_snapshot: dict         # 供给矩阵快照
) -> dict:
    """
    契约撮合引擎
    
    职责: 将需求与供给进行匹配，生成契约草案
    
    返回值 dict:
    {
        "status": "matched" | "no_match" | "partial_match",
        "contract_draft": dict | None,
        "match_score": float,
        "candidates": list,          # 候选生产者列表
        "error": str | None
    }
    """
```

#### 函数3：资源竞价排序

```python
def trader_bid_sort(
    ctx: dict,
    competing_orders: list,       # 竞争同一资源的订单列表
    resource_id: str              # 资源标识
) -> dict:
    """
    竞价排序器
    
    职责: 当多个消费者竞争同一生产者资源时，按优先级与信用分排序
    
    排序因子:
    1. 时效等级 (L0 > L1 > L2 > L3)
    2. 信用评分 (高 > 低)
    3. 提交时间 (先 > 后)
    
    返回值 dict:
    {
        "status": "sorted",
        "ranking": [               # 排序后的订单列表
            {"order_id": str, "score": float, "priority": int},
            ...
        ],
        "rejected_orders": list,   # 被淘汰的订单
        "error": str | None
    }
    """
```

#### 函数4：订单创建

```python
def trader_order_create(
    ctx: dict,
    contract_draft: dict,         # 已批准的契约草案
    consumer_id: str,
    producer_id: str
) -> dict:
    """
    订单创建
    
    职责: 将批准的契约转化为可执行的订单
    
    返回值 dict:
    {
        "status": "created" | "creation_failed",
        "order_id": str,
        "order_state": str,           # CREATED
        "shm_channel": dict,          # 分配的SHM通道信息
        "error": str | None
    }
    """
```

#### 函数5：订单状态推进

```python
def trader_order_advance(
    ctx: dict,
    order_id: str,
    target_state: str,            # 目标状态
    advance_context: dict         # 推进上下文
) -> dict:
    """
    订单状态推进
    
    职责: 推进订单生命周期状态机
    
    合法跃迁:
    CREATED → PENDING → ASSIGNED → PRODUCING → FULFILLED → SETTLED → COMPLETED
    任意态 → FAILED / CANCELLED (需监督者审批)
    
    返回值 dict:
    {
        "status": "advanced" | "rejected" | "invalid_transition",
        "order_id": str,
        "previous_state": str,
        "current_state": str,
        "error": str | None
    }
    """
```

#### 函数6：结算处理

```python
def trader_settle(
    ctx: dict,
    order_id: str,
    fulfillment_proof: dict       # 履约证明
) -> dict:
    """
    结算处理
    
    职责: 订单履约完成后，执行结算，更新信用总账
    
    返回值 dict:
    {
        "status": "settled" | "settlement_failed",
        "order_id": str,
        "time_cost_actual_ms": int,
        "credit_delta": dict,         # 信用变化 {"consumer": int, "producer": int}
        "error": str | None
    }
    """
```

#### 函数7：参数化任务适配

```python
def trader_param_task_adapt(
    ctx: dict,
    param_task_id: str,           # BP-0028参数化任务标识
    instance_guid: str,           # 执行实例GUID
    instance_seq: int             # 实例序号
) -> dict:
    """
    参数化任务适配器
    
    职责: 将BP-0028的执行实例转化为标准订单，接入订单管理流程
    
    返回值 dict:
    {
        "status": "adapted" | "adaptation_failed",
        "order_id": str,
        "param_task_id": str,
        "instance_guid": str,
        "error": str | None
    }
    """
```

### 5.2 交易监督者函数

#### 函数8：交易合规审计

```python
def supervisor_trade_audit(
    ctx: dict,
    trade_record: dict            # 交易记录
) -> dict:
    """
    交易合规审计
    
    职责: 验证交易是否符合单向性法则、掩码规则、路径分级
    
    审计项:
    - 单向性: 数据流向必须为 生产者→消费者
    - 掩码匹配: DemandMask & OutputMask ≠ 0
    - 路径分级: 时效等级与路径等级匹配
    - 完整性: IntegrityHash校验通过
    - 权限: 消费者/生产者权限范围检查
    
    返回值 dict:
    {
        "status": "compliant" | "violated",
        "violation_type": str | None,    # 违规类型编码
        "violation_detail": str | None,
        "audit_event_id": str,           # 审计事件ID
        "action": "pass" | "block" | "alert" | "escalate",
        "error": str | None
    }
    """
```

#### 函数9：契约合规审查

```python
def supervisor_contract_review(
    ctx: dict,
    contract_draft: dict          # 契约草案
) -> dict:
    """
    契约合规审查
    
    职责: 审查契约草案的合法性
    
    审查项:
    - 需求矢量合法性（边界检查）
    - 供给承诺可达性（资源验证）
    - 代价评估合理性（防欺诈）
    - 安全等级匹配（敏感度检查）
    - 历史信用检查（违约风险）
    
    返回值 dict:
    {
        "status": "approved" | "rejected" | "conditional",
        "review_findings": list,     # 审查发现列表
        "conditions": list | None,   # 有条件批准的附加条件
        "risk_score": float,         # 风险评分 0.0-1.0
        "audit_event_id": str,
        "error": str | None
    }
    """
```

#### 函数10：实时流量监控

```python
def supervisor_flow_monitor(
    ctx: dict,
    monitor_config: dict          # 监控配置
) -> dict:
    """
    实时流量监控
    
    职责: 监控SHM交换区的交易流量，检测异常模式
    
    异常模式:
    - 流量突增: 短时间内交易量超过阈值
    - 频率异常: 单一消费者/生产者交易频率异常
    - 掩码越界: 交易类型超出声明的掩码范围
    - 路径滥用: 低优先级交易占用高优先级通道
    - 回环检测: 检测违反单向性的隐秘回环
    
    返回值 dict:
    {
        "status": "normal" | "anomaly_detected",
        "anomaly_type": str | None,
        "anomaly_details": dict | None,
        "metrics": {                   # 当前流量指标
            "total_trades": int,
            "trades_per_second": float,
            "shm_utilization": float,
            "path_distribution": dict
        },
        "error": str | None
    }
    """
```

#### 函数11：熔断执行

```python
def supervisor_circuit_break(
    ctx: dict,
    break_request: dict           # 熔断请求
) -> dict:
    """
    熔断执行器
    
    职责: 检测到违规或异常时，执行熔断操作
    
    熔断级别:
    - TRADE_BLOCK: 阻止单笔违规交易
    - ORDER_PAUSE: 暂停可疑订单
    - CONSUMER_QUARANTINE: 隔离可疑消费者
    - PRODUCER_QUARANTINE: 隔离可疑生产者
    - CHANNEL_SHUTDOWN: 关闭SHM通道
    - GLOBAL_EMERGENCY: 全局紧急熔断
    
    返回值 dict:
    {
        "status": "executed" | "not_justified",
        "break_level": str,
        "affected_entities": list,    # 受影响的实体列表
        "recovery_plan": dict,        # 恢复计划
        "audit_event_id": str,
        "error": str | None
    }
    """
```

#### 函数12：冲突仲裁

```python
def supervisor_arbitrate(
    ctx: dict,
    conflict_report: dict         # 冲突报告
) -> dict:
    """
    冲突仲裁器
    
    职责: 按三方宪章的仲裁原则裁决冲突
    
    仲裁原则（优先级从高到低）:
    1. 安全压倒原则: 安全策略自动获胜
    2. 时间优先原则: L0>L1>L2>L3，同等级比信用
    3. 代价补偿原则: 对受损方补偿信用积分
    4. 降级生存原则: 无法调和时启动降级模式
    
    返回值 dict:
    {
        "status": "resolved" | "escalated",
        "arbitration_decision": dict,
        "compensation_plan": dict | None,
        "degradation_plan": dict | None,
        "audit_event_id": str,
        "error": str | None
    }
    """
```

#### 函数13：审计链写入

```python
def supervisor_audit_write(
    ctx: dict,
    audit_event: dict             # 审计事件
) -> dict:
    """
    审计链写入器
    
    职责: 将审计事件写入不可篡改的审计链
    
    审计事件结构:
    {
        "event_id": str,
        "timestamp_ns": int,
        "actor": "trader" | "supervisor" | "system",
        "action": str,
        "target": str,
        "pre_hash": str,              # 上一事件哈希
        "payload_hash": str,
        "severity": "info" | "warning" | "critical" | "emergency"
    }
    
    返回值 dict:
    {
        "status": "written" | "write_failed",
        "event_id": str,
        "chain_position": int,        # 在审计链中的位置
        "event_hash": str,            # 事件哈希（用于链式验证）
        "error": str | None
    }
    """
```

### 5.3 共享接口函数

#### 函数14：任务管理Worker初始化

```python
def task_mgmt_worker_init(
    ctx: dict,
    worker_config: dict           # Worker配置
) -> dict:
    """
    任务管理Worker初始化
    
    职责: 初始化交易员模块和交易监督者模块，建立SHM审计通道
    
    返回值 dict:
    {
        "status": "initialized" | "init_failed",
        "worker_id": str,
        "trader_state": dict,
        "supervisor_state": dict,
        "audit_channel_id": str,
        "error": str | None
    }
    """
```

#### 函数15：任务管理Worker主循环

```python
def task_mgmt_worker_loop(
    ctx: dict,
    worker_state: dict            # Worker状态
) -> dict:
    """
    任务管理Worker主循环
    
    职责: 协调交易员与交易监督者的协作循环
    
    每个循环周期:
    1. 交易员处理待处理意图和订单
    2. 交易监督者执行审计和监控
    3. 处理两角色之间的审计请求/响应
    4. 检查熔断状态
    5. 写入周期性审计事件
    
    返回值 dict:
    {
        "status": "tick_complete" | "error",
        "orders_processed": int,
        "audits_performed": int,
        "anomalies_detected": int,
        "next_tick_ms": int,
        "error": str | None
    }
    """
```

---

## 六、数据结构设计

### 6.1 订单状态机

```
[CREATED] ──交易员创建──▶ [PENDING] ──调度器分配──▶ [ASSIGNED]
                                              │
                                              ▼
                                         [PRODUCING] ──产出完成──▶ [FULFILLED]
                                              │                      │
                                              │ 异常                  │ 消费者确认
                                              ▼                      ▼
                                         [FAILED]              [SETTLED]
                                                                  │
                                                                  ▼
                                                             [COMPLETED]

取消路径: [CREATED/PENDING/ASSIGNED] ──消费者撤销──▶ [CANCELLED]
熔断路径: [ANY_STATE] ──交易监督者熔断──▶ [QUARANTINED]
```

### 6.2 核心数据结构

#### 订单记录

```python
ORDER_RECORD = {
    "order_id": str,               # 订单唯一标识
    "contract_id": str,            # 关联契约标识
    "consumer_id": str,
    "producer_id": str,
    "intent_id": str,              # 原始意图标识
    
    "state": str,                  # 当前状态
    "state_history": [             # 状态变更历史
        {"from": str, "to": str, "timestamp": int, "reason": str}
    ],
    
    "trade_spec": {                # 交易规格
        "product_signature": str,
        "demand_mask": int,
        "output_mask": int,
        "path_level": str,         # P0/P1/P2/P3
        "latency_class": int,      # L0/L1/L2/L3
        "sensitivity_class": int   # S0/S1/S2/S3
    },
    
    "cost": {                      # 代价记录
        "time_cost_budget_ms": int,
        "time_cost_actual_ms": int | None,
        "shm_cost_mb_s": float | None
    },
    
    "timestamps": {
        "created_at": int,
        "assigned_at": int | None,
        "producing_at": int | None,
        "fulfilled_at": int | None,
        "settled_at": int | None
    },
    
    "param_task_ref": str | None,  # 关联的参数化任务标识（BP-0028）
    "audit_event_ids": list        # 关联的审计事件ID列表
}
```

#### 审计事件

```python
AUDIT_EVENT = {
    "event_id": str,
    "timestamp_ns": int,
    "actor": str,                  # trader / supervisor / system
    "action": str,                 # 操作名称
    "target": str,                 # 目标实体ID
    "target_type": str,            # order / trade / contract / consumer / producer
    "severity": str,               # info / warning / critical / emergency
    "pre_hash": str,               # 上一事件哈希（链式）
    "payload_hash": str,           # 事件载荷哈希
    "findings": list,              # 审计发现
    "action_taken": str            # pass / block / alert / escalate
}
```

#### 熔断状态

```python
CIRCUIT_BREAK_STATE = {
    "break_id": str,
    "level": str,                  # TRADE_BLOCK / ORDER_PAUSE / ... / GLOBAL_EMERGENCY
    "trigger_reason": str,
    "trigger_event_id": str,
    "affected_entities": [
        {"entity_id": str, "entity_type": str, "action": str}
    ],
    "recovery_plan": {
        "strategy": str,           # auto / manual / conditional
        "conditions": list,
        "estimated_recovery_ms": int
    },
    "is_active": bool,
    "activated_at": int,
    "resolved_at": int | None
}
```

---

## 七、两角色协作协议

### 7.1 审计请求-响应协议

交易员与交易监督者通过SHM审计通道通信：

```
交易员 ──审计请求──▶ [SHM审计通道] ──▶ 交易监督者
交易员 ◀──审计结果── [SHM审计通道] ◀── 交易监督者

审计请求结构:
{
    "request_id": str,
    "request_type": "trade_audit" | "contract_review" | "order_cancel",
    "payload": dict,
    "urgency": "normal" | "high" | "critical",
    "timeout_ms": int
}

审计结果结构:
{
    "request_id": str,
    "verdict": "pass" | "block" | "conditional" | "escalate",
    "findings": list,
    "conditions": list | None,
    "audit_event_id": str
}
```

### 7.2 超时与降级规则

| 场景 | 超时时间 | 降级策略 |
|------|----------|----------|
| 交易审计（正常） | 1ms | 自动放行，事后审计 |
| 交易审计（高风险） | 5ms | 阻塞等待，超时则阻止 |
| 契约审查 | 10ms | 阻塞等待，超时则拒绝 |
| 冲突仲裁 | 100ms | 阻塞等待，超时则升级至商场主进程 |

### 7.3 熔断协作

```
交易监督者检测到异常
        │
        ▼
┌───────────────────┐
│ 评估异常严重程度   │
└────────┬──────────┘
         │
    ┌────┴────┐
    ▼         ▼
 低风险     高风险
    │         │
    ▼         ▼
 告警+记录  熔断+通知交易员
    │         │
    │         ▼
    │    ┌───────────────────┐
    │    │ 交易员执行熔断指令 │
    │    │ 暂停相关订单       │
    │    │ 释放相关资源       │
    │    └───────────────────┘
    │
    ▼
┌───────────────────┐
│ 写入审计链         │
│ 通知信用总账       │
└───────────────────┘
```

---

## 八、与现有体系的关联

### 8.1 与任务流程总纲领的关系

| 总纲领概念 | 任务管理Worker对应 |
|------------|-------------------|
| 任务GUID | 订单ID（order_id） |
| 状态机 | 订单状态机（扩展版，增加QUARANTINED态） |
| DAG依赖 | 交易员通过订单依赖管理实现 |
| 熔断/重试/降级 | 交易监督者执行，交易员配合 |
| 审计追踪链 | 交易监督者的审计链写入器 |

### 8.2 与三方宪章的关系

| 三方宪章原则 | 任务管理Worker实现 |
|-------------|-------------------|
| 商场中立 | 交易员竞价排序器按规则排序，不偏袒 |
| 单向性 | 交易监督者的交易合规审计器强制执行 |
| 信用互联 | 交易员结算时更新信用，监督者提供行为数据 |
| 冲突仲裁 | 交易监督者的冲突仲裁器按五原则执行 |

### 8.3 与交易掩码法则的关系

| 交易掩码法则 | 任务管理Worker实现 |
|-------------|-------------------|
| 掩码匹配 | 交易员契约撮合时检查 DemandMask & OutputMask |
| 路径分级 | 交易监督者审计时验证路径等级匹配 |
| 确认即终结 | 交易员结算后订单状态不可回滚 |
| 控制交易 | 交易员通过专用通道处理控制指令 |

### 8.4 与BP-0028参数化任务的关系

BP-0028的参数化任务通过**交易员的参数化任务适配器**（函数7）接入订单管理流程：

```
BP-0028 触发器引擎生成执行实例
        │
        ▼
trader_param_task_adapt()  ──▶ 生成标准订单
        │
        ▼
进入标准订单生命周期（审计→撮合→履约→结算）
```

每次参数化任务的执行实例对应一个独立订单，拥有独立审计链记录。

---

## 九、测试场景

### 9.1 交易员测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 意图接收 | 合法意图矢量 | accepted | 解析结果正确 |
| 意图拒绝 | 格式错误意图 | rejected | 错误信息准确 |
| 契约撮合 | 需求+供给 | matched | 匹配分数合理 |
| 竞价排序 | 5个竞争订单 | 排序结果 | 按优先级+信用排序正确 |
| 订单创建 | 已批准契约 | created | SHM通道分配成功 |
| 订单推进 | 合法状态跃迁 | advanced | 状态正确变更 |
| 非法跃迁 | CREATED→PRODUCING | rejected | 拒绝非法跃迁 |
| 结算处理 | 已履约订单 | settled | 信用更新正确 |

### 9.2 交易监督者测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 合规交易审计 | 合法交易记录 | compliant | action=pass |
| 逆向交易拦截 | 消费者→生产者交易 | violated | action=block |
| 掩码越界检测 | 超出OutputMask的交易 | violated | action=block |
| 路径滥用检测 | L3交易占用P0通道 | violated | action=block |
| 契约审查通过 | 合法契约草案 | approved | risk_score<0.3 |
| 契约审查拒绝 | 高风险契约 | rejected | findings非空 |
| 流量异常检测 | 流量突增3倍 | anomaly_detected | 异常类型正确 |
| 熔断执行 | TRADE_BLOCK请求 | executed | 受影响实体正确 |
| 冲突仲裁 | 需求冲突报告 | resolved | 按安全优先原则裁决 |
| 审计链写入 | 审计事件 | written | 链式哈希正确 |

### 9.3 两角色协作测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 正常流程 | 意图→审计→撮合→审计→订单 | 全部通过 | 订单创建成功 |
| 审计拒绝 | 意图→审计(拒绝) | 订单未创建 | 交易员收到block |
| 熔断协作 | 异常检测→熔断→交易员执行 | 订单暂停 | 状态=QUARANTINED |
| 参数化任务适配 | BP-0028实例→适配→订单 | adapted | 订单与实例关联正确 |
| 超时降级 | 审计超时(低风险) | 自动放行 | 事后审计记录存在 |

---

## 十、安全约束

### 10.1 职责隔离

- 交易员和交易监督者运行在独立的逻辑上下文中
- 两者不共享可变状态，仅通过SHM审计通道通信
- 单方故障不导致另一方崩溃

### 10.2 权限矩阵

| 操作 | 交易员 | 交易监督者 |
|------|--------|------------|
| 创建/推进订单 | ✅ | ❌ |
| 拦截交易 | ❌ | ✅ |
| 执行熔断 | ❌（配合执行） | ✅（发起） |
| 写入审计链 | ❌ | ✅ |
| 查询订单状态 | ✅ | ✅ |
| 查询审计记录 | ✅（只读） | ✅ |
| 修改信用评分 | ✅（结算时） | ❌（仅提供数据） |

### 10.3 资源限制

```python
WORKER_LIMITS = {
    "max_pending_orders": 10000,
    "max_concurrent_audits": 100,
    "audit_channel_size_mb": 10,
    "max_circuit_breaks_per_hour": 50,
    "flow_monitor_window_sec": 60,
    "anomaly_threshold_multiplier": 3.0
}
```

---

## 十一、授权文档申请

### 11.1 授权申请单

```
蓝图编号: BP-0045
蓝图名称: Task Management Worker
申请事项: 请求商场授权部署任务管理Worker（含交易监督者与交易员）

依赖蓝图:
  - BP-0028-task-parameterization (参数化任务)
  - BP-0014-config (配置管理)
  - BP-0004-exception (异常处理)

授权范围:
  - 交易员模块: 意图接收、契约撮合、竞价排序、订单管理、结算处理
  - 交易监督者模块: 交易审计、契约审查、流量监控、熔断执行、冲突仲裁
  - SHM审计通道: 交易员与监督者之间的通信通道
  - 最大并发订单: 10000
  - 审计链写入权限: 授予

安全约束:
  - 交易员与监督者职责隔离
  - 所有交易必须经过合规审计
  - 审计链不可篡改
  - 熔断机制必须生效
  - 遵循单向性法则与掩码规则

签名: [创世Worker]
```

### 11.2 商场授权回应（预期）

```
授权编号: AUTH-BP-0045-20260512
状态: 待授权

授权内容:
  - 允许部署任务管理Worker
  - 交易员与交易监督者双角色授权
  - SHM审计通道分配

约束条件:
  - 交易监督者对安全违规拥有一票否决权
  - 审计链写入不可关闭
  - 熔断操作需记录完整审计事件

有效期: 永久
签章: [商场调度中心]
```

---

## 十二、附录

### 12.1 术语定义

| 术语 | 定义 |
|------|------|
| 任务管理Worker | 商场域内管理任务交易的核心Worker，内含交易员与交易监督者 |
| 交易员 | 负责任务撮合、订单管理、结算推进的角色 |
| 交易监督者 | 负责交易合规审计、流量监控、熔断处置的角色 |
| 订单 | 意图经撮合后生成的可执行任务单元 |
| 契约草案 | 消费者与商场就任务达成的双向约束文件 |
| SHM审计通道 | 交易员与交易监督者之间的专用通信通道 |

### 12.2 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0.0 | 2026-05-12 | 初始版本，定义交易员与交易监督者双角色架构 |

---

**审核状态**: 待审核  
**下一步动作**: 提交AICoder进行七维审查与代码生成
