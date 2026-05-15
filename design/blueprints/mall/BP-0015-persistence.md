# 蓝图文档（定稿版）：BP-0015-persistence -- 文件持久化消费者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0015-persistence |
| 蓝图名称 | 文件持久化消费者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**海马体与长期记忆系统**。SHM是易失的，商场崩溃后若无持久化层，所有日志、配置、状态矢量灰飞烟灭。持久化消费者负责将SHM中的关键记忆异步刷写到磁盘，并在重启时执行记忆恢复。

### 2.2 数据来源

`shm_vector` - 需持久化的SHM矢量

### 2.3 数据去向

`file_system` - 磁盘快照文件

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def persist_subscribe(
    ctx: dict,
    source_topic: str,
    flush_policy: dict
) -> dict:
    """
    注册一个持久化作业。持久化消费者作为消费者身份订阅topic。
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM + FILE_IO
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `source_topic` | str | 是 | 长度<=256 | 订阅的topic名称 |
| `flush_policy` | dict | 是 | 非空 | 刷写策略 |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "persistence_consumer"
    "metadata": {
        "job_id": int,              # 作业ID
        "queue_depth": int          # 当前队列深度
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def persist_unsubscribe(ctx: dict, job_id: int) -> dict:
    """
    停止对某topic的持久化。
    [复杂度]: O(1)
    """

def persist_flush(ctx: dict, job_id: int, force: bool) -> dict:
    """
    执行一次刷写。
    [复杂度]: O(n)
    """

def persist_recover(ctx: dict, snapshot_path: str) -> dict:
    """
    商场重启时由main()调用。读取磁盘快照恢复状态。
    [复杂度]: O(n)
    """

def persist_schedule_async(ctx: dict, job_id: int, interval_ms: int) -> dict:
    """
    为定时作业设定异步调度。
    [复杂度]: O(1)
    """

def persist_validate_checksum(ctx: dict, file_path: str) -> dict:
    """
    验证单个快照文件的校验和。
    [复杂度]: O(n)
    """

def persist_mark_vector(ctx: dict, vector_addr: int, persist_flags: int) -> dict:
    """
    供其他生产者显式标记某个SHM矢量需要持久化。
    [复杂度]: O(1)
    """

def persist_get_status(ctx: dict, job_id: int) -> dict:
    """
    返回作业状态。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | source_topic长度<=256 | `ERR_TOPIC_TOO_LONG` | `error` |
| PRE-002 | flush_policy非空 | `ERR_POLICY_EMPTY` | `error` |
| PRE-003 | 作业表未满 | `ERR_JOB_TABLE_FULL` | `error` |
| PRE-004 | 快照路径有效 | `ERR_SNAPSHOT_PATH_INVALID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | 刷写操作异步进行 | 不阻塞消息路由 |
| POST-003 | 写入文件包含校验和 | 防止磁盘静默损坏 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零网络调用**: 不进行网络IO
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_TOPIC_TOO_LONG` | source_topic长度>256 | `error` | "Topic name too long" |
| `ERR_POLICY_EMPTY` | flush_policy为空 | `error` | "Flush policy is empty" |
| `ERR_JOB_TABLE_FULL` | 作业表容量耗尽 | `error` | "Job table full" |
| `ERR_SNAPSHOT_PATH_INVALID` | 快照路径无效 | `error` | "Invalid snapshot path" |
| `ERR_FLUSH_FAILED` | 磁盘满或权限错误 | `error` | "Flush failed" |
| `ERR_RECOVER_CORRUPTED` | 快照文件校验和不通过 | `error` | "Snapshot corrupted" |
| `ERR_RECOVER_INCOMPLETE` | 部分子系统快照缺失 | `error` | "Recovery incomplete" |
| `ERR_QUEUE_OVERFLOW` | 刷写队列深度超过10000 | `error` | "Queue overflow" |
| `ERR_ASYNC_RACE` | 异步flush与同步recover并发 | `error` | "Async race, busy" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | persist_subscribe("system.log.rotate", policy) | 默认 | status="ok", job_id返回 |
| TC-002 | 正常 | persist_flush(job_id, force=True) | 有数据 | status="ok", 写入字节数返回 |
| TC-003 | 正常 | persist_recover("./mall_snapshots/latest/") | 有快照 | status="ok", 恢复摘要返回 |
| TC-004 | 边界 | 队列深度=10000 | 默认 | status="ok" |
| TC-005 | 边界 | 快照文件大小=最大值 | 默认 | status="ok" |
| TC-006 | 异常 | 磁盘满时flush | 默认 | status="error", error="Flush failed" |
| TC-007 | 异常 | 篡改快照文件后recover | 默认 | status="error", error="Snapshot corrupted" |
| TC-008 | 性能 | 日志rotate后100ms内快照文件出现 | 默认 | 快照文件存在 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | 校验和计算 | Python 3.8+ |
| `json` | 序列化 | Python 3.8+ |
| `os` | 文件操作 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n) | 刷写操作线性时间 |
| 内存 | <= 256KB | 作业表+队列 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 100ms | 单次刷写 |
| 磁盘 | <= 10MB | 快照文件上限 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ⚠️ | 仅写入快照文件，不读取其他文件 |
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

### 作业表头矢量（`persist_job_header`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x50525354` ("PRST") |
| 0x04 | `job_count` | uint32 | 当前作业数 |
| 0x08 | `max_jobs` | uint32 | 容量 |
| 0x0C | `total_pending` | uint32 | 全局待刷写计数 |
| 0x10 | `last_flush_timestamp` | uint64 | |

### 作业条目矢量（`persist_job_vector`，固定128B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `job_id` | uint32 | |
| 0x04 | `source_topic_addr` | uint64 | 订阅的topic字符串地址 |
| 0x0C | `policy_flags` | uint32 | 定时/定量/压缩等 |
| 0x10 | `interval_ms` | uint32 | 定时间隔 |
| 0x14 | `threshold_count` | uint32 | 定量阈值 |
| 0x18 | `pending_queue_head` | uint64 | 待刷写矢量链表头 |
| 0x20 | `pending_count` | uint32 | 当前队列深度 |
| 0x24 | `flush_count` | uint64 | 累计刷写次数 |
| 0x2C | `error_count` | uint32 | 连续失败次数 |

---

## 附录B：生命周期状态机

```
INIT ──► ALIVE ──► FLUSHING ──► ALIVE ──► RECOVERING ──► ALIVE
```

- **FLUSHING**: 执行`persist_flush()`期间，新到达的矢量继续入队
- **RECOVERING**: `persist_recover()`执行期间，商场其他组件冻结

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 消息路由 | ← 持久化 | 订阅`system.log.rotate`、`system.config.changed`等 |
| 日志生产者 | ← 持久化 | 收到rotate事件后刷写旧日志缓冲区 |
| 配置消费者 | ← 持久化 | 收到config_changed后保存配置快照 |
| 进程注册表 | ← 持久化 | 定时触发`registry.snapshot` topic |
| 商场`main()` | → 持久化 | 启动时调用`persist_recover()`；关闭时调用强制flush |
| SHM矢量管理 | → 持久化 | `persist_mark_vector()`标记的矢量 |
| 心跳生产者 | → 持久化 | 提供时间戳用于快照命名 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
