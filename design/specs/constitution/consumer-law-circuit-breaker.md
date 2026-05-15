# CONSUMER-LAW-004 消费者异常与熔断法则

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档类型**: 纲领性文件 / 商场异常处理总则附件  
> **版本**: v1.0  
> **状态**: 创世阶段·韧性体系确立  
> **别名**: 《熔断法》  
> **依赖**: CONSUMER-PROFILE-001 / 商场异常处理总则

---

## 一、异常哲学

**异常不是错误，异常是商场的呼吸。**

在商场生态中，生产者会异常，消费者会异常，商场自身也会异常。异常处理系统不是消灭异常，而是：
- 将异常控制在局部，防止扩散
- 将异常转化为可观测的信号，驱动进化
- 在异常中保障核心功能的最低可用性

---

## 二、消费者异常分类

### 2.1 按来源分类

| 类别 | 标识 | 描述 | 典型场景 |
|------|------|------|----------|
| **意图异常** | E-I | 消费者提交的意图格式错误、权限不足或违反铁律 | 意图矢量字段越界、敏感度过高 |
| **消费异常** | E-C | 消费者在消费过程中行为异常 | 读取越界、消费频率过高、拒绝确认 |
| **资源异常** | E-R | 消费者占用的SHM资源出现异常 | 内存泄漏、矢量区块溢出、句柄未释放 |
| **连接异常** | E-N | 消费者与商场的SHM连接中断 | 消费者进程崩溃、网络分区(分布式场景) |
| **安全异常** | E-S | 消费者触发安全铁律 | 越权访问、数据篡改、外发违规 |
| **代价异常** | E-T | 消费者的时间代价行为异常 | 代价债务累积、申诉滥发、契约撕毁 |

### 2.2 按 severity 分级

| 级别 | 标识 | 影响范围 | 响应时效 | 处理策略 |
|------|------|----------|----------|----------|
| **提示** | S0 | 仅影响该消费者自身 | 下次调度时处理 | 日志记录，无需干预 |
| **警告** | S1 | 可能影响该消费者的后续订单 | 100ms内响应 | 降级服务，通知消费者 |
| **严重** | S2 | 影响商场局部资源分配 | 10ms内响应 | 局部熔断，隔离消费者 |
| **紧急** | S3 | 可能波及其他消费者或生产者 | 1ms内响应 | 全局熔断，强制注销 |
| **灾难** | S4 | 威胁商场整体安全或可用性 | 立即响应 | 商场紧急停机，创世委员会介入 |

---

## 三、熔断机制

### 3.1 熔断器模型

商场为每个消费者维护一个**熔断器 (Circuit Breaker)**:

```
熔断器状态机:

                    ┌─────────────┐
         ┌─────────→│   CLOSED    │←─────────┐
         │ 成功恢复  │  (闭合/正常) │  异常计数 │
         │          └──────┬──────┘  超限     │
         │                 │                   │
         │                 ↓                   │
         │          ┌─────────────┐           │
         │          │   OPEN      │           │
         │          │  (断开/熔断) │           │
         │          └──────┬──────┘           │
         │                 │ 超时              │
         │                 ↓                   │
         │          ┌─────────────┐           │
         └──────────│  HALF-OPEN  │───────────┘
                    │ (半开/探测)  │
                    └─────────────┘
```

**状态定义**:
- **CLOSED**: 正常状态，消费者可正常提交意图与接收产出
- **OPEN**: 熔断状态，商场拒绝该消费者的一切新意图，已存在的订单按异常策略处理
- **HALF-OPEN**: 探测状态，商场允许该消费者提交一个探测意图，若成功则恢复CLOSED，若失败则回到OPEN

### 3.2 熔断触发条件

| 触发条件 | 阈值 | 熔断级别 | 进入状态 |
|----------|------|----------|----------|
| 连续消费异常次数 | ≥ 5次/秒 | S2 | OPEN |
| 连续安全异常次数 | ≥ 1次 | S3 | OPEN |
| 连续代价异常次数 | ≥ 3次/分钟 | S2 | OPEN |
| SHM资源泄漏率 | ≥ 20%分配量未释放 | S2 | OPEN |
| 连接心跳丢失 | ≥ 3个心跳周期 | S1 → S2 | OPEN |
| 消费者进程崩溃 | 1次 | S2 | OPEN |
| 恶意攻击检测 | 1次 | S4 | OPEN + 商场告警 |

### 3.3 熔断恢复策略

```
OPEN → HALF-OPEN 的恢复条件:
  - 熔断持续时间达到冷却期 (根据severity: S1=5s, S2=30s, S3=5min, S4=人工确认)
  - 消费者主动发起恢复请求，并通过身份重验证
  - 商场负载降至安全线以下

HALF-OPEN → CLOSED 的恢复条件:
  - 探测意图成功履约，ATC ≤ ETC × 1.5
  - 消费者行为在探测期间无异常
  - 审计沙箱确认消费者状态正常

HALF-OPEN → OPEN 的再次熔断:
  - 探测意图失败
  - 探测期间再次触发任何异常
  - 冷却期重置为原值的 2 倍 (指数退避)
```

---

## 四、消费者降级模式

当消费者未被完全熔断，但商场资源紧张或生产者异常时，启动**降级模式**:

### 4.1 降级等级

| 等级 | 名称 | 措施 | 消费者感知 |
|------|------|------|------------|
| **D0** | 正常 | 无降级 | 无 |
| **D1** | 节流 | 降低产出投递频率至 50% | 数据刷新变慢 |
| **D2** | 降质 | 降低产出数据精度/分辨率 | 数据变粗糙 |
| **D3** | 缓存 | 停止实时投递，改为缓存快照 | 看到历史数据而非实时 |
| **D4** | 静默 | 停止一切产出投递，保留连接 | 无数据更新，但连接存活 |
| **D5** | 断开 | 强制断开SHM连接，保留消费者注册 | 完全离线，需手动重连 |

### 4.2 降级决策逻辑

```python
def degrade_decision(consumer_profile, mall_load, producer_status):
    # 输入: 消费者画像、商场负载、生产者状态
    # 输出: 降级等级 D0-D5

    if producer_status == "CRASHED":
        return D5 if consumer_profile.sensitivity >= S2 else D3

    if mall_load > 0.95:
        if consumer_profile.budget == B0:
            return D5
        elif consumer_profile.budget == B1:
            return D3
        elif consumer_profile.budget == B2:
            return D1

    if mall_load > 0.85:
        if consumer_profile.latency >= L2:
            return D2

    if producer_status == "DEGRADED":
        return D1

    return D0
```

---

## 五、异常处理函数级设计

### 5.1 消费者异常捕获

```python
def consumer_exception_handler(consumer_id, exception_vector, mall_state):
    # 阶段 1: 异常分类与定级
    exc_class = classify_exception(exception_vector)
    severity = assess_severity(exc_class, exception_vector)

    # 阶段 2: 审计记录
    audit_log_write(consumer_id, exc_class, severity, mall_state.snapshot())

    # 阶段 3: 熔断检查
    breaker = circuit_breaker_get(consumer_id)
    if breaker.should_trip(exc_class, severity):
        breaker.trip()
        notify_consumer(consumer_id, "CIRCUIT_OPEN", breaker.cooldown_period)
        return ExceptionResult.TRIPPED

    # 阶段 4: 降级检查
    if severity >= S1:
        degrade_level = degrade_decision(consumer_profile_get(consumer_id), 
                                         mall_state.load, 
                                         producer_status_get(consumer_id))
        if degrade_level > D0:
            apply_degradation(consumer_id, degrade_level)
            notify_consumer(consumer_id, "DEGRADED", degrade_level)

    # 阶段 5: 恢复尝试 (仅对非安全异常)
    if severity < S3 and exc_class != E_S:
        recovery_result = attempt_recovery(consumer_id, exception_vector)
        if recovery_result == RecoveryResult.SUCCESS:
            breaker.record_success()
            return ExceptionResult.RECOVERED

    # 阶段 6: 最终处置
    if severity >= S3:
        force_unregister(consumer_id)
        return ExceptionResult.TERMINATED

    return ExceptionResult.HANDLED
```

### 5.2 商场级异常聚合

```python
def mall_exception_aggregator(time_window_ms=1000):
    # 聚合指定时间窗口内的全部消费者异常
    # 检测异常模式，判断是否存在系统性风险

    exc_batch = audit_log_query(time_window_ms, category="CONSUMER_EXCEPTION")

    # 模式检测
    if detect_cascade_pattern(exc_batch):
        # 级联异常: 一个消费者异常引发多个消费者连锁异常
        trigger_mall_degradation("CASCADE_PROTECT")

    if detect_resource_exhaustion(exc_batch):
        # 资源耗尽: 大量消费者因SHM不足而异常
        trigger_mall_degradation("RESOURCE_EMERGENCY")

    if detect_attack_pattern(exc_batch):
        # 攻击模式: 异常分布符合攻击特征
        trigger_mall_degradation("SECURITY_LOCKDOWN")
        alert_aicoder("ATTACK_DETECTED", exc_batch)

    return MallHealthStatus.from_batch(exc_batch)
```

---

## 六、消费者自愈机制

商场鼓励消费者实现**函数级自愈**，而非依赖商场的全局保护：

```
自愈能力 1: 意图重试
    当意图提交失败时，消费者应根据 retry_policy 自动重试，
    重试间隔采用指数退避: 10ms → 20ms → 40ms → 80ms → 放弃

自愈能力 2: 产出缓冲
    消费者应维护本地环形缓冲区，当商场投递变慢时，
    缓冲区提供平滑的数据流，避免UI卡顿或控制震荡

自愈能力 3: 生产者切换
    混合型消费者应实现主辅生产者自动切换，
    当主生产者异常时，平滑过渡至备用订阅源

自愈能力 4: 降级适配
    消费者应识别商场的降级通知，自动调整自身的处理策略:
    - D1节流: 降低本地处理频率，匹配输入频率
    - D2降质: 切换至低精度处理算法
    - D3缓存: 启用本地历史回放模式
    - D4静默: 进入待机状态，定期发送心跳探测

自愈能力 5: 状态快照
    持久态消费者应定期将关键状态快照写入商场持久化SHM区，
    崩溃恢复后可从最近快照重建，最小化数据丢失
```

---

## 七、异常与进化的关系

**异常是进化的饲料。**

```
异常处理流程的进化闭环:

异常发生 → 异常捕获 → 审计记录 → AICoder分析 → 蓝图/调度/消费者画像优化
    ↑                                                              ↓
    └──────────────── 新版本部署 ← 创世委员会审核 ← 进化建议报告 ←─┘
```

AICoder对异常数据的分析重点:
- 异常模式聚类: 识别哪些消费者画像特征与异常高发相关
- 生产者稳定性关联: 分析特定生产者的异常传导路径
- 商场调度缺陷: 检测调度决策是否加剧了异常传播
- 消费者自愈效果: 评估哪些自愈策略真正降低了异常影响

---

## 八、修订记录

| 版本 | 日期 | 修订内容 | 审核状态 |
|------|------|----------|----------|
| v1.0 | 2026-05-10 | 确立异常分类、熔断器模型、降级模式、函数级处理设计、自愈机制、进化闭环 | 待审核 |

---

> **法则箴言**: "熔断非弃，降级非辱，异常非敌，进化之母。"
