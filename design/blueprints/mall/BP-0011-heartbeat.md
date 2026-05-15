# 蓝图文档（定稿版）：BP-0011-heartbeat -- 心跳/时钟生产者

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0011-heartbeat |
| 蓝图名称 | 心跳/时钟生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**脉搏与生命体征监测仪**。产生全局时间基准，检测Worker僵死，驱动调度器节拍，是商场从「静态加载」进入「动态运行」的第一推动力。

### 2.2 数据来源

`system_clock` - 系统时钟

### 2.3 数据去向

`shm_vector` - 全局tick矢量、心跳登记表

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def tick_emit(
    ctx: dict,
    interval_ms: int
) -> dict:
    """
    向tick_vector写入递增计数器与系统时间戳。
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `interval_ms` | int | 是 | 100<=值<=10000 | tick间隔（毫秒） |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "heartbeat_producer"
    "metadata": {
        "tick_counter": int,        # 当前tick计数
        "drift_us": int             # 漂移微秒数
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def heartbeat_register(ctx: dict, entity_id: str, timeout_ms: int) -> dict:
    """
    实体在初始化完成后必须调用，向心跳登记表申报。
    [复杂度]: O(1)
    """

def heartbeat_ack(ctx: dict, entity_id: str, status_vector: bytes) -> dict:
    """
    实体在每个tick周期或完成一个工作单元后调用。
    [复杂度]: O(1)
    """

def heartbeat_check_all(ctx: dict, global_tick: int) -> dict:
    """
    每个tick周期执行一次扫描，返回僵死实体列表。
    [复杂度]: O(n)
    """

def heartbeat_wait(ctx: dict, tick_count: int) -> dict:
    """
    阻塞调用者直到经过tick_count个全局tick。
    [复杂度]: O(1)
    """

def heartbeat_get_timestamp(ctx: dict) -> dict:
    """
    返回最近一次tick_emit的时间戳。
    [复杂度]: O(1)
    """

def heartbeat_set_timeout(ctx: dict, entity_id: str, new_timeout_ms: int, requester_id: str) -> dict:
    """
    动态调整实体心跳阈值。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | interval_ms在100-10000范围内 | `ERR_INTERVAL_INVALID` | `error` |
| PRE-002 | entity_id已注册 | `ERR_ENTITY_NOT_REGISTERED` | `error` |
| PRE-003 | timeout_ms>0 | `ERR_TIMEOUT_INVALID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | tick_counter单调递增 | 不回退 |
| POST-003 | 僵死实体触发状态迁移 | 通知注册表 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行文件IO、网络IO
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_INTERVAL_INVALID` | interval_ms不在有效范围 | `error` | "Interval must be 100-10000ms" |
| `ERR_ENTITY_NOT_REGISTERED` | entity_id未注册 | `error` | "Entity not registered" |
| `ERR_TIMEOUT_INVALID` | timeout_ms<=0 | `error` | "Timeout must be positive" |
| `ERR_TICK_DRIFT` | drift_us超过10%interval | `error` | "Tick drift exceeded threshold" |
| `ERR_HEARTSTORM` | 单tick内ack调用超过容量2倍 | `error` | "Heartstorm attack detected" |
| `ERR_ACK_FOR_UNKNOWN` | ack的entity_id不存在 | `error` | "ACK for unknown entity" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | tick_emit(interval_ms=1000) | 默认 | status="ok", tick_counter递增 |
| TC-002 | 正常 | heartbeat_register(entity_id, timeout_ms=5000) | 默认 | status="ok" |
| TC-003 | 正常 | heartbeat_ack(entity_id, status_vector) | 默认 | status="ok", last_ack_tick更新 |
| TC-004 | 边界 | interval_ms=100 | 默认 | status="ok" |
| TC-005 | 边界 | interval_ms=10000 | 默认 | status="ok" |
| TC-006 | 异常 | interval_ms=50 | 默认 | status="error", error="Interval must be 100-10000ms" |
| TC-007 | 异常 | 未注册实体调用heartbeat_ack | 默认 | status="error", error="Entity not registered" |
| TC-008 | 性能 | 1000个实体同时ack | 默认 | 扫描耗时<10%interval_ms |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `time` | 时间戳 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(1) | tick发射常数时间 |
| 内存 | <= 128KB | 心跳登记表 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 1ms | 单次tick |

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

### 全局Tick矢量（`global_tick_vector`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `tick_counter` | uint64 | 单调递增，重启归零 |
| 0x08 | `last_emit_timestamp` | uint64 | 纳秒级时间戳 |
| 0x10 | `interval_ms` | uint32 | 当前发射间隔 |
| 0x14 | `drift_us` | int32 | 漂移微秒数 |
| 0x18 | `reserved` | vector[8] | 对齐 |

### 心跳登记条目（`heartbeat_entry_vector`，固定64B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `entity_id` | vector[16] | |
| 0x10 | `timeout_ms` | uint32 | |
| 0x14 | `last_ack_tick` | uint64 | 最后响应tick |
| 0x1C | `last_ack_timestamp` | uint64 | |
| 0x24 | `miss_count` | uint32 | 连续未响应次数 |
| 0x28 | `status_summary` | vector[16] | 实体自报状态 |
| 0x38 | `reserved` | vector[8] | |

---

## 附录B：生命周期触发机制

```
每次 tick_emit 触发：
    1. 更新 global_tick_vector
    2. 调用 heartbeat_check_all()
    3. 对 miss_count >= stale_threshold 的实体：
       a. 发布 stale_event_vector 到消息路由
       b. 通知注册表 registry_update_state(entity_id, SUSPEND/DEAD)
       c. 通知日志生产者记录 FATAL 级事件
    4. 对 miss_count >= threshold + 3 的实体：
       a. 通知授权加载生产者 auth_revoke(immediate=False)
```

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 商场`main()` | → 心跳 | 启动后第二个加载（注册表之后） |
| 所有实体 | → 心跳 | 初始化时`heartbeat_register()`，运行时`heartbeat_ack()` |
| 进程注册表 | ← 心跳 | `heartbeat_check_all()`内部调用`registry_update_state()` |
| 消息路由 | ← 心跳 | 发布`heartbeat.stale`与`heartbeat.tick` topic |
| 日志生产者 | ← 心跳 | 记录每次状态迁移与漂移告警 |
| 配置消费者 | → 心跳 | 调整`interval_ms`或`stale_threshold` |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
