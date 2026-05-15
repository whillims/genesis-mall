# 关键工具深度设计 v1.0

> **覆盖工具**：参数动态调节器、异常告警器、用户自动决策引擎
> **设计深度**：函数级架构、状态机、数据流、安全边界

---

## 一、参数动态调节器深度设计

### 1.1 核心状态机

```
[IDLE] ──调节请求──→ [VALIDATING]
                          │
                          ▼
                   [边界检查通过?]
                    /                            是            否
                  /                               ▼                ▼
           [SIGNING]         [REJECTED]
              │                  │
              ▼                  ▼
         [ENCRYPTING]      返回错误码+原因
              │
              ▼
         [TRANSMITTING]
              │
              ▼
         [WAIT_ACK]
            /             收到ACK   超时
          /                 ▼          ▼
    [APPLIED]    [ROLLBACK]
         │          │
         ▼          ▼
    [LOGGING]   [LOGGING]
         │          │
         └────┬─────┘
              ▼
           [IDLE]
```

### 1.2 函数级接口设计

```c
/* 消费者域：UI调用 */
int submit_adjustment(
    const char* consumer_id,
    const char* producer_id,
    adjustment_param_t* param,
    uint32_t timeout_ms
);

/* 商场域：Worker处理 */
int validate_adjustment(
    adjustment_param_t* param,
    contract_bounds_t* bounds
);

int sign_adjustment(
    adjustment_param_t* param,
    private_key_t* consumer_key
);

int apply_adjustment(
    const char* producer_id,
    adjustment_param_t* param
);

/* 生产者域：接收端 */
int receive_adjustment(
    adjustment_param_t* param,
    uint32_t* ack_timeout_ms
);
```

### 1.3 安全边界校验规则

| 校验项 | 校验逻辑 | 失败处理 |
|--------|---------|---------|
| 数值范围 | param.value ∈ [bounds.min, bounds.max] | 拒绝，提示安全边界 |
| 调节速率 | |Δparam| / Δt ≤ bounds.max_rate | 拒绝，提示调节过快 |
| 契约状态 | contract.status == "EXECUTING" | 拒绝，提示契约未激活 |
| 消费者权限 | consumer_id ∈ contract.authorized_consumers | 拒绝，提示无权调节 |
| 生产者在线 | producer.heartbeat_age < 3s | 拒绝，提示生产者离线 |

### 1.4 IO不可切割性保障

```
数据帧结构：
┌─────────┬─────────┬─────────┬─────────┐
│ 帧头    │ 数据载荷 │ 校验和   │ 帧尾    │
│ (4B)    │ (N B)   │ (4B)    │ (4B)    │
└─────────┴─────────┴─────────┴─────────┘

调节指令插入规则：
- 仅允许在帧尾之后、下一帧头之前插入
- 调节指令本身携带"生效帧序号"，指示从哪一帧开始应用新参数
- 生产者收到调节指令后，完成当前帧传输，在下一帧应用新参数
```

---

## 二、异常告警器深度设计

### 2.1 分级状态机

```
[NORMAL]
   │
   │ 指标偏离阈值10%
   ▼
[NOTICE] ──自动恢复──→ [NORMAL]
   │
   │ 指标偏离阈值25% 或 异常码出现
   ▼
[WARNING] ──消费者确认──→ [NORMAL]
   │
   │ 生产者离线 或 严重失锁
   ▼
[CRITICAL] ──自动制动──→ [SAFE_SHUTDOWN]
   │
   │ SHM污染 或 签名失效
   ▼
[DISASTER] ──强制隔离──→ [ISOLATED]
```

### 2.2 告警升级矩阵

| 当前级别 | 升级条件 | 升级目标 | 消费者确认 |
|---------|---------|---------|-----------|
| NORMAL | 任意指标偏离10% | NOTICE | 无需 |
| NOTICE | 偏离25%或异常码 | WARNING | 建议确认 |
| WARNING | 生产者离线 | CRITICAL | 自动升级 |
| CRITICAL | SHM污染 | DISASTER | 自动升级 |

### 2.3 告警抑制机制

```python
# 伪代码：告警抑制逻辑
def should_alarm(alarm_type, current_level):
    # 检查消费者白名单
    if alarm_type in consumer_whitelist:
        return False, "WHITELISTED"

    # 检查告警冷却期
    if time_since_last_alarm(alarm_type) < COOLDOWN_PERIOD:
        return False, "COOLDOWN"

    # 检查重复告警抑制
    if is_duplicate_alarm(alarm_type, current_level):
        return False, "DUPLICATE"

    return True, "ALARM"
```

### 2.4 小场景：功率测量故障告警

```
场景：功率计在第15频点返回异常码-999

告警器行为：
1. 检测到异常码 → 触发WARNING级别
2. UI弹窗显示："功率计异常：频点15，异常码-999"
3. 决策引擎建议："降低激励-10dB或跳过该点"
4. 消费者选择："降低激励-10dB"
5. 调节器执行 → 功率计恢复正常
6. 告警器降级 → NOTICE → NORMAL
7. 日志记录完整告警-响应链
```

---

## 三、用户自动决策引擎深度设计

### 3.1 架构定位

**结论：决策引擎在Worker，决策界面在UI。**

**分工边界**：
- **Worker端（决策引擎）**：运行规则引擎，根据消费者预设的"如果-那么"规则提出决策建议
- **UI端（决策界面）**：将建议转化为人类可理解的选项，等待消费者点击"同意/修改/拒绝"

### 3.2 规则引擎函数设计

```c
/* 规则结构 */
typedef struct {
    char* condition;      /* 条件表达式，如 "power < threshold" */
    char* suggestion;     /* 建议文本 */
    char* action_type;    /* 建议动作类型：ADJUST / SKIP / ABORT */
    void* action_param;   /* 动作参数 */
    uint8_t priority;     /* 优先级 1-10 */
} decision_rule_t;

/* 引擎接口 */
int evaluate_rules(
    sensor_data_t* current_data,
    decision_rule_t* rules,
    uint32_t rule_count,
    suggestion_t** out_suggestions,
    uint32_t* out_count
);
```

### 3.3 决策数据流

```
传感器数据 → 决策引擎Worker → 建议包(加密+签名)
                                    ↓
                              UI渲染建议面板
                                    ↓
消费者点击"执行建议" → 确认包(加密+签名) → 执行Worker → 生产者
```

### 3.4 为什么不能在UI实现决策？

| 风险 | 说明 |
|------|------|
| JS篡改 | 浏览器插件、中间人攻击可篡改JavaScript逻辑 |
| 绕过人类 | 若决策在UI执行，可能绕过消费者确认环节 |
| 签名断裂 | UI端无法安全签名，导致审计链断裂 |
| 来源伪造 | 攻击者可伪造决策来源，冒充消费者意志 |

### 3.5 消费者规则预设模板

```json
{
  "rules": [
    {
      "name": "功率饱和保护",
      "condition": "power_meter.status == -999",
      "suggestion": "功率计可能饱和，建议降低激励功率",
      "action": {"type": "ADJUST", "param": "excitation_power -= 10"},
      "priority": 8,
      "auto_execute": false
    },
    {
      "name": "超时保护",
      "condition": "elapsed_time > contract.max_duration * 0.8",
      "suggestion": "任务即将超时，建议拆分或终止",
      "action": {"type": "SUGGEST", "param": "split_or_abort"},
      "priority": 9,
      "auto_execute": false
    }
  ]
}
```

**关键**：`auto_execute` 永远为 `false`，确保人类确认不可绕过。

---

## 四、三个工具的协同关系

```
┌─────────────────────────────────────────────┐
│              消费者UI层                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ 调节旋钮 │  │ 告警弹窗 │  │ 建议面板 │      │
│  └────┬────┘  └────┬────┘  └────┬────┘      │
└───────┼────────────┼────────────┼───────────┘
        │            │            │
        ▼            ▼            ▼
┌─────────────────────────────────────────────┐
│              商场Worker层                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │调节指令  │  │告警引擎  │  │决策引擎  │      │
│  │验证/签名 │  │分级/抑制 │  │规则评估  │      │
│  └────┬────┘  └────┬────┘  └────┬────┘      │
└───────┼────────────┼────────────┼───────────┘
        │            │            │
        └────────────┴────────────┘
                      │
                      ▼
              ┌─────────────┐
              │   生产者域    │
              │ 控制接口接收  │
              └─────────────┘
```

---

**审核状态**：待审核
**版本**：v1.0
**日期**：2026-05-12
