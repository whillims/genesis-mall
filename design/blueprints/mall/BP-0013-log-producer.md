# 蓝图文档（定稿版）：BP-0013-log-producer -- 日志生产者

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0013-log-producer |
| 蓝图名称 | 日志生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**黑匣子与审计链**。所有内部事件、错误、状态变化必须经由日志生产者转化为SHM上的可追溯记录。日志生产者自身**禁止直接操作stdout/stderr**（stdio封印铁律），其输出经消息路由投递给控制台消费者渲染。

### 2.2 数据来源

`shm_vector` - 各实体的日志事件

### 2.3 数据去向

`shm_vector` - 环形日志缓冲区、消息路由

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def log_emit(
    ctx: dict,
    level: int,
    source_entity_id: str,
    message: str,
    timestamp: float,
    message_type: int
) -> dict:
    """
    核心入口。将日志写入SHM环形缓冲区，并视级别立即通过路由发布。
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `level` | int | 是 | 0<=值<=4 | DEBUG=0, INFO=1, WARN=2, ERROR=3, FATAL=4 |
| `source_entity_id` | str | 是 | 长度=16 | 来源实体ID |
| `message` | str | 是 | 长度<=1024 | 日志内容 |
| `timestamp` | float | 是 | >=0 | 时间戳 |
| `message_type` | int | 是 | 0或1 | 0=纯文本, 1=结构化 |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "log_producer"
    "metadata": {
        "log_entry_addr": int,      # 日志条目SHM地址
        "buffer_slot": int          # 环形缓冲区槽位
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def log_query(ctx: dict, start_time: float, end_time: float, level_filter: int, source_filter: str) -> dict:
    """
    扫描环形缓冲区，返回匹配的日志条目列表。
    [复杂度]: O(n)
    """

def log_rotate_trigger(ctx: dict) -> dict:
    """
    手动触发rotate。
    [复杂度]: O(1)
    """

def log_set_buffer_size(ctx: dict, size_bytes: int, entry_size: int) -> dict:
    """
    动态调整缓冲区。
    [复杂度]: O(1)
    """

def log_get_stats(ctx: dict) -> dict:
    """
    返回当前缓冲区统计信息。
    [复杂度]: O(1)
    """

def log_subscribe_pattern(ctx: dict, level_filter: int, source_filter: str, subscriber_entity_id: str) -> dict:
    """
    供外部消费者注册日志订阅。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | level在0-4范围内 | `ERR_LEVEL_INVALID` | `error` |
| PRE-002 | source_entity_id长度=16 | `ERR_SOURCE_ID_INVALID` | `error` |
| PRE-003 | message长度<=1024 | `ERR_MESSAGE_TOO_LONG` | `error` |
| PRE-004 | message_type在0-1范围内 | `ERR_MESSAGE_TYPE_INVALID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | FATAL级日志触发异常上报 | 自动调用异常处理系统 |
| POST-003 | 缓冲区满时触发rotate | 通知持久化消费者 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行文件IO、网络IO
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出
- **stdio封印**: 禁止直接操作stdout/stderr

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_LEVEL_INVALID` | level不在0-4范围 | `error` | "Log level must be 0-4" |
| `ERR_SOURCE_ID_INVALID` | source_entity_id长度不等于16 | `error` | "Source entity ID must be 16 bytes" |
| `ERR_MESSAGE_TOO_LONG` | message长度>1024 | `error` | "Message too long" |
| `ERR_MESSAGE_TYPE_INVALID` | message_type不在0-1范围 | `error` | "Message type must be 0 or 1" |
| `ERR_BUFFER_EXHAUSTED` | 环形缓冲区满且rotate失败 | `error` | "Buffer exhausted" |
| `ERR_EMIT_FLOOD` | 单tick内同一实体调用超过100次 | `error` | "Log flood detected" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | log_emit(level=INFO, message="test") | 默认 | status="ok", log_entry_addr返回 |
| TC-002 | 正常 | log_emit(level=FATAL, message="crash") | 默认 | status="ok", 异常处理系统被触发 |
| TC-003 | 正常 | log_query(start_time, end_time) | 有日志 | status="ok", 日志列表返回 |
| TC-004 | 边界 | message长度=1024 | 默认 | status="ok" |
| TC-005 | 边界 | level=0 (DEBUG) | 默认 | status="ok" |
| TC-006 | 异常 | level=5 | 默认 | status="error", error="Log level must be 0-4" |
| TC-007 | 异常 | message长度=1025 | 默认 | status="error", error="Message too long" |
| TC-008 | 性能 | 1ms内调用10000次log_emit | 默认 | 丢弃超额，自身不崩溃 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | 条目校验 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(1) | 日志写入常数时间 |
| 内存 | <= 256KB | 环形缓冲区 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 1ms | 单次写入 |

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

**禁止stdout输出**：日志生产者自身禁止直接操作stdout/stderr，所有输出必须经消息路由投递给控制台消费者渲染。唯一例外：日志生产者自身无法写入（SHM损坏）时，通过stdio紧急通道输出一行崩溃信息。

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 契约审查

依据本蓝图第4节行为契约进行逐条验证。

### 8.2 代码生成

将蓝图意图转化为纯函数级Python实现。

### 8.3 测试覆盖

基于第5节测试场景生成验收测试用例。

### 8.4 安全扫描

验证三大零原则、SHM权限、输入消毒、stdio封印。

### 8.5 授权文件输出

按5字段JSON结构生成授权文件。

---

## 附录A：SHM矢量结构设计

### 日志缓冲区头矢量（`log_buffer_header`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x4C4F4742` ("LOGB") |
| 0x04 | `version` | uint32 | 1 |
| 0x08 | `head_idx` | uint32 | 写入游标 |
| 0x0C | `tail_idx` | uint32 | 最旧有效条目游标 |
| 0x10 | `capacity` | uint32 | 条目数上限 |
| 0x14 | `entry_size` | uint32 | 单条大小 |
| 0x18 | `rotate_count` | uint32 | 已rotate次数 |
| 0x1C | `overflow_count` | uint32 | 丢弃条目数 |

### 日志条目矢量（`log_entry_vector`，固定256B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `timestamp` | uint64 | |
| 0x08 | `level` | uint32 | |
| 0x0C | `source_entity_id` | vector[16] | |
| 0x1C | `message_len` | uint32 | |
| 0x20 | `message_addr` | uint64 | 指向消息内容矢量 |
| 0x28 | `checksum` | uint32 | 条目校验 |
| 0x30 | `inline_message` | vector[192] | 内联消息区 |

---

## 附录B：生命周期状态机

```
INIT ──► ALIVE ──► ROTATING ──► ALIVE ──► DEGRADED ──► DEAD
```

- **ROTATING**: 旧缓冲区锁定，新缓冲区激活
- **DEGRADED**: overflow_count在100个tick内增长超过1000

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 所有实体 | → 日志 | `log_emit()` |
| 消息路由 | ← 日志 | `route_publish("system.log", log_entry, qos=1)` |
| 控制台消费者 | ← 路由 ← 日志 | 订阅`system.log`，渲染到stdout |
| 持久化消费者 | ← 路由 ← 日志 | 订阅`system.log.rotate`，执行落盘 |
| 心跳生产者 | → 日志 | 提供统一时间戳 |
| 异常处理系统 | ↔ 日志 | FATAL日志触发异常上报 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
