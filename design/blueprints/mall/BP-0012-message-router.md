# 蓝图文档（定稿版）：BP-0012-message-router -- 消息路由生产者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0012-message-router |
| 蓝图名称 | 消息路由生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**神经系统与淋巴系统**。在SHM上实现基于矢量的异步发布/订阅总线，是「SHM为唯一通讯通道」这一宪法在消息层的具体实现。所有生产者与消费者之间的数据交换必须经过路由，禁止任何硬编码的管道或队列旁路。

### 2.2 数据来源

`shm_vector` - 消息信封矢量

### 2.3 数据去向

`shm_vector` - 订阅者接收队列

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def route_publish(
    ctx: dict,
    topic: str,
    payload_vector: bytes,
    qos_level: int,
    sender_entity_id: str
) -> dict:
    """
    将消息信封矢量异步投递到所有匹配订阅者的接收队列。
    [复杂度]: O(n) n=订阅者数
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `topic` | str | 是 | 长度<=256 | 主题名称 |
| `payload_vector` | bytes | 是 | 长度<=64KB | 数据负载 |
| `qos_level` | int | 是 | 0或1 | 服务质量等级 |
| `sender_entity_id` | str | 是 | 长度=16 | 发送者ID |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "message_router_producer"
    "metadata": {
        "ticket_id": int,           # 投递票据ID
        "estimated_delivery": int   # 预估投递数
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def route_create_topic(ctx: dict, topic: str, policy: dict) -> dict:
    """
    在topic目录中注册新主题。
    [复杂度]: O(1)
    """

def route_destroy_topic(ctx: dict, topic_id: int, force: bool) -> dict:
    """
    清理topic及其所有订阅关系与滞留消息。
    [复杂度]: O(n)
    """

def route_subscribe(ctx: dict, consumer_entity_id: str, topic_filter: str) -> dict:
    """
    注册订阅关系。
    [复杂度]: O(1)
    """

def route_unsubscribe(ctx: dict, subscription_id: int) -> dict:
    """
    移除订阅。
    [复杂度]: O(1)
    """

def route_deliver(ctx: dict, max_messages: int) -> dict:
    """
    扫描所有topic的发布队列，将消息信封搬运到各订阅者的接收队列。
    [复杂度]: O(n*m)
    """

def route_query_subscribers(ctx: dict, topic: str) -> dict:
    """
    返回某topic的订阅者列表与队列深度。
    [复杂度]: O(n)
    """

def route_ack(ctx: dict, consumer_entity_id: str, ticket_id: int) -> dict:
    """
    qos=1时，消费者处理完消息后调用。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | topic长度<=256 | `ERR_TOPIC_TOO_LONG` | `error` |
| PRE-002 | payload长度<=64KB | `ERR_PAYLOAD_TOO_LARGE` | `error` |
| PRE-003 | qos_level在0-1范围内 | `ERR_QOS_INVALID` | `error` |
| PRE-004 | sender_entity_id已注册 | `ERR_SENDER_NOT_REGISTERED` | `error` |
| PRE-005 | topic目录未满 | `ERR_TOPIC_OVERFLOW` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | 同一topic内消息按时间戳FIFO | 顺序保证 |
| POST-003 | 订阅者接收队列深度不超过max_depth | 溢出策略 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行文件IO、网络IO
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_TOPIC_TOO_LONG` | topic长度>256 | `error` | "Topic name too long" |
| `ERR_PAYLOAD_TOO_LARGE` | payload长度>64KB | `error` | "Payload too large" |
| `ERR_QOS_INVALID` | qos_level不在0-1范围 | `error` | "QoS level must be 0 or 1" |
| `ERR_SENDER_NOT_REGISTERED` | sender_entity_id未注册 | `error` | "Sender not registered" |
| `ERR_TOPIC_OVERFLOW` | 队列深度超过max_depth | `error` | "Topic queue overflow" |
| `ERR_DEAD_SUBSCRIBER` | 消费者状态为DEAD | `error` | "Dead subscriber" |
| `ERR_ROUTER_LOOP` | 消息循环路由 | `error` | "Router loop detected" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | route_publish("input.keyboard", payload, qos=0) | 默认 | status="ok", ticket_id返回 |
| TC-002 | 正常 | route_subscribe(consumer_id, "input.*") | 默认 | status="ok", subscription_id返回 |
| TC-003 | 正常 | 端到端：键盘发布→路由投递→控制台接收 | 默认 | 延迟<5ms |
| TC-004 | 边界 | topic长度=256 | 默认 | status="ok" |
| TC-005 | 边界 | payload长度=64KB | 默认 | status="ok" |
| TC-006 | 异常 | topic长度=257 | 默认 | status="error", error="Topic name too long" |
| TC-007 | 异常 | qos_level=2 | 默认 | status="error", error="QoS level must be 0 or 1" |
| TC-008 | 性能 | 1000条qos=1消息投递 | 默认 | 最终一致性，无丢失 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | 消息哈希 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n*m) | 投递操作 |
| 内存 | <= 512KB | topic目录+订阅表 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 10ms | 单次投递 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 不进行任何文件读写 |
| 网络零IO | ✅ | 不进行任何网络通讯 |
| 零全局副作用 | ✅ | 不修改任何全局状态 |
| 输入消毒 | ✅ | 对所有输入进行边界校验 |
| 确定性输出 | ✅ | 相同输入产生相同输出 |
| SHM权限合规 | ✅ | 仅访问授权的SHM矢量 |
| 栈帧安全 | ✅ | 不存在栈溢出风险 |
| 零堆分配 | ✅ | 不使用动态内存分配 |

### 7.2 stdout特许声明

无stdout输出需求。

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 契约审查

依据本蓝图第4节行为契约进行逐条验证。

### 8.2 代码生成

将蓝图意图转化为纯函数级Python实现。

### 8.3 测试覆盖

基于第5节测试场景生成验收测试用例。

### 8.4 安全扫描

验证三大零原则、SHM权限、输入消毒。

### 8.5 授权文件输出

按5字段JSON结构生成授权文件。

---

## 附录A：SHM矢量结构设计

### Topic目录头矢量（`topic_directory_header`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x524F5554` ("ROUT") |
| 0x04 | `topic_count` | uint32 | 当前topic数 |
| 0x08 | `max_topics` | uint32 | 容量上限 |
| 0x0C | `total_subscriptions` | uint32 | 总订阅数 |

### 消息信封矢量（`message_envelope_vector`，固定64B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `ticket_id` | uint64 | |
| 0x08 | `sender_entity_id` | vector[16] | |
| 0x18 | `topic_id` | uint32 | |
| 0x1C | `qos_level` | uint32 | |
| 0x20 | `payload_addr` | uint64 | 指向实际负载矢量 |
| 0x28 | `payload_size` | uint32 | |
| 0x2C | `timestamp` | uint64 | 发布时间 |
| 0x34 | `next_envelope_ptr` | uint64 | 链表下一个信封 |

---

## 附录B：生命周期状态机

```
INIT ──► ALIVE ──► DEGRADED ──► DEAD
```

- **DEGRADED**: 消息堆积超过80%总容量，开始丢弃qos=0消息

Topic生命周期：
```
topic_created ──► active ──► idle（无订阅者且60 tick无消息）──► destroyed
```

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 键盘生产者 | → 路由 | `route_publish("input.keyboard", key_event, qos=0)` |
| 控制台消费者 | ← 路由 | `route_subscribe(console_id, "display.#")` |
| 日志生产者 | → 路由 | `route_publish("system.log", log_entry, qos=1)` |
| 心跳生产者 | ← 路由 | 心跳tick驱动`route_deliver()` |
| 进程注册表 | → 路由 | 实体注销时，路由自动清理其所有subscription |
| 配置消费者 | → 路由 | 调整`max_depth`与溢出策略 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
