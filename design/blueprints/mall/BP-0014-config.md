# 蓝图文档（定稿版）：BP-0014-config -- 配置消费者/生产者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0014-config |
| 蓝图名称 | 配置消费者/生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**基因编辑酶与动态进化控制器**。允许在商场运行期间修改行为参数、调整阈值、切换策略，无需重启。配置变更通过消息路由广播，订阅者实时响应，是商场从「静态部署」走向「自我适应」的关键器官。

### 2.2 数据来源

`shm_vector` - 配置变更请求

### 2.3 数据去向

`shm_vector` - 配置表矢量区

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def config_apply(
    ctx: dict,
    key: str,
    value: bytes,
    requester_id: str,
    validation_mode: int
) -> dict:
    """
    原子操作：验证 → 写入配置表 → 版本号+1 → 广播变更。
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `key` | str | 是 | 长度<=256 | 配置项键名 |
| `value` | bytes | 是 | 长度<=4096 | 配置值 |
| `requester_id` | str | 是 | 长度=16 | 请求者ID |
| `validation_mode` | int | 是 | 0或1 | 0=严格, 1=宽松 |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "config_producer_consumer"
    "metadata": {
        "old_value": bytes,         # 旧值副本
        "new_version": int,         # 新版本号
        "affected_subscribers": int # 影响的订阅者数
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def config_rollback(ctx: dict, steps: int) -> dict:
    """
    从历史快照恢复。
    [复杂度]: O(n)
    """

def config_subscribe(ctx: dict, key_pattern: str, subscriber_entity_id: str, callback_topic: str) -> dict:
    """
    注册配置订阅。
    [复杂度]: O(1)
    """

def config_get(ctx: dict, key: str) -> dict:
    """
    读取当前生效值。
    [复杂度]: O(1)
    """

def config_list_all(ctx: dict) -> dict:
    """
    返回全部配置项的键值对列表。
    [复杂度]: O(n)
    """

def config_validate(ctx: dict, key: str, value: bytes) -> dict:
    """
    预验证，不实际写入。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | key长度<=256 | `ERR_KEY_TOO_LONG` | `error` |
| PRE-002 | value长度<=4096 | `ERR_VALUE_TOO_LARGE` | `error` |
| PRE-003 | requester_id已注册 | `ERR_REQUESTER_NOT_REGISTERED` | `error` |
| PRE-004 | requester_id有权限 | `ERR_PERMISSION_DENIED` | `error` |
| PRE-005 | 配置表未满 | `ERR_CONFIG_TABLE_FULL` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | version_counter单调递增 | 不回退 |
| POST-003 | 变更广播到订阅者 | 通过消息路由 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行文件IO、网络IO
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_KEY_TOO_LONG` | key长度>256 | `error` | "Config key too long" |
| `ERR_VALUE_TOO_LARGE` | value长度>4096 | `error` | "Config value too large" |
| `ERR_REQUESTER_NOT_REGISTERED` | requester_id未注册 | `error` | "Requester not registered" |
| `ERR_PERMISSION_DENIED` | requester_id无权限 | `error` | "Permission denied" |
| `ERR_CONFIG_TABLE_FULL` | 配置表容量耗尽 | `error` | "Config table full" |
| `ERR_KEY_NOT_FOUND` | key不存在 | `error` | "Config key not found" |
| `ERR_VALIDATION_FAILED` | 值校验失败 | `error` | "Validation failed" |
| `ERR_APPLY_RACE` | 并发apply同一key | `error` | "Concurrent apply, retry" |
| `ERR_ROLLBACK_IMPOSSIBLE` | 回滚步数超过快照数 | `error` | "Rollback impossible" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | config_apply("heartbeat.interval_ms", 500) | 默认 | status="ok", version递增 |
| TC-002 | 正常 | config_get("heartbeat.interval_ms") | 有配置 | status="ok", value返回 |
| TC-003 | 正常 | config_rollback(1) | 有历史 | status="ok", 恢复到上一版本 |
| TC-004 | 边界 | key长度=256 | 默认 | status="ok" |
| TC-005 | 边界 | value长度=4096 | 默认 | status="ok" |
| TC-006 | 异常 | key长度=257 | 默认 | status="error", error="Config key too long" |
| TC-007 | 异常 | 未授权requester_id | 默认 | status="error", error="Permission denied" |
| TC-008 | 性能 | 热调整heartbeat.interval_ms | 默认 | 3个tick内生效 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | 值哈希 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(1) | 配置读写常数时间 |
| 内存 | <= 128KB | 配置表+快照 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 5ms | 单次操作 |

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

### 配置表头矢量（`config_table_header`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x43464E47` ("CFNG") |
| 0x04 | `version_counter` | uint64 | 单调递增 |
| 0x0C | `entry_count` | uint32 | 当前条目数 |
| 0x10 | `max_entries` | uint32 | 容量 |
| 0x14 | `snapshot_count` | uint32 | 保留快照数 |

### 配置条目矢量（`config_entry_vector`，固定256B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `key_len` | uint32 | |
| 0x04 | `key_addr` | uint64 | 指向key字符串 |
| 0x0C | `value_len` | uint32 | |
| 0x10 | `value_addr` | uint64 | 指向value矢量 |
| 0x18 | `value_type` | uint32 | 0=int, 1=float, 2=string, 3=bool, 4=json |
| 0x1C | `version` | uint64 | 该条目当前版本 |
| 0x24 | `timestamp` | uint64 | 最后修改时间 |
| 0x2C | `requester_id` | vector[16] | 最后修改者 |

---

## 附录B：生命周期状态机

```
INIT ──► ALIVE ──► APPLYING ──► ALIVE ──► ROLLBACKING ──► ALIVE
```

- **APPLYING**: 写入配置表与广播变更期间，短暂锁定
- **ROLLBACKING**: 恢复快照期间同样锁定

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 控制型消费者 | → 配置 | `config_apply()` 下发命令 |
| 心跳生产者 | ← 配置 | 订阅`heartbeat.*`，响应interval调整 |
| 日志生产者 | ← 配置 | 订阅`log.*`，响应buffer_size调整 |
| 消息路由 | ← 配置 | `config_notify_broadcast()`内部调用`route_publish()` |
| 持久化消费者 | ← 配置 | 订阅`system.config.changed`，保存配置快照 |
| 进程注册表 | → 配置 | `config_apply()`验证requester_id存在性与权限 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
