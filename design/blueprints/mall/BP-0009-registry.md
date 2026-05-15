# 蓝图文档（定稿版）：BP-0009-registry -- 进程注册表生产者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0009-registry |
| 蓝图名称 | 进程注册表生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**户籍系统与身份宪法执行者**。在SHM上维护所有活着的实体（生产者/消费者）的唯一身份映射。没有注册表，SHM矢量无法寻址，授权文件无处注入，商场无法区分「内部成员」与「外部噪音」。

### 2.2 数据来源

`shm_vector` - SHM共享内存矢量

### 2.3 数据去向

`shm_vector` - 注册表矢量区

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def registry_register(
    ctx: dict,
    entity_id: str,
    entity_type: int,
    shm_base_addr: int,
    auth_hash: bytes
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数，含SHM配置 |
| `entity_id` | str | 是 | 长度=16 | 实体唯一标识 |
| `entity_type` | int | 是 | 0<=值<=2 | 类型枚举：TYPE_KERNEL=0, TYPE_PRODUCER=1, TYPE_CONSUMER=2 |
| `shm_base_addr` | int | 是 | >=0 | SHM矢量基址 |
| `auth_hash` | bytes | 是 | 长度=32 | 授权文件哈希 |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "registry_producer"
    "metadata": {
        "assigned_entity_id": str,  # 分配后的实体ID（可能修正）
        "registry_slot": int,       # 注册表槽位号
        "state": str                # 当前状态
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def registry_deregister(ctx: dict, entity_id: str, reason: str) -> dict:
    """
    软注销：状态改为DEAD，保留记录3个心跳周期后转入RECLAIMED。
    [复杂度]: O(1)
    """

def registry_query(ctx: dict, entity_id: str) -> dict:
    """
    返回实体完整描述符。若entity_id不存在，返回错误。
    [复杂度]: O(1)
    """

def registry_list_by_type(ctx: dict, type_filter: int, state_filter: int) -> dict:
    """
    扫描注册表，返回符合条件的实体ID列表。
    [复杂度]: O(n)
    """

def registry_update_state(ctx: dict, entity_id: str, new_state: int, requester_auth_hash: bytes) -> dict:
    """
    状态迁移必须经过合法校验。
    [复杂度]: O(1)
    """

def registry_lock(ctx: dict, entity_id: str) -> dict:
    """
    对指定实体的描述符加读锁。
    [复杂度]: O(1)
    """

def registry_unlock(ctx: dict, lock_token: bytes) -> dict:
    """
    释放锁。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | entity_id长度=16 | `ERR_ENTITY_ID_INVALID` | `error` |
| PRE-002 | entity_type在0-2范围内 | `ERR_ENTITY_TYPE_INVALID` | `error` |
| PRE-003 | shm_base_addr>=0 | `ERR_SHM_ADDR_INVALID` | `error` |
| PRE-004 | auth_hash长度=32 | `ERR_AUTH_HASH_INVALID` | `error` |
| PRE-005 | 注册表未满 | `ERR_REGISTRY_FULL` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | 注册成功后entry_count递增 | 原子操作 |
| POST-003 | 实体状态初始化为INIT | 生命周期起点 |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行文件IO、网络IO、进程管理
- **零副作用**: 不修改输入参数，不产生外部可观察效果（除返回值和SHM写入）
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_ENTITY_ID_INVALID` | entity_id长度不等于16 | `error` | "Entity ID must be 16 bytes" |
| `ERR_ENTITY_TYPE_INVALID` | entity_type不在0-2范围 | `error` | "Entity type must be 0, 1, or 2" |
| `ERR_SHM_ADDR_INVALID` | shm_base_addr为负数 | `error` | "SHM address must be non-negative" |
| `ERR_AUTH_HASH_INVALID` | auth_hash长度不等于32 | `error` | "Auth hash must be 32 bytes" |
| `ERR_REGISTRY_FULL` | 注册表容量耗尽 | `error` | "Registry capacity exhausted" |
| `ERR_ENTITY_NOT_FOUND` | 查询的entity_id不存在 | `error` | "Entity not found" |
| `ERR_DUPLICATE_ID` | entity_id已存在 | `error` | "Entity ID already exists" |
| `ERR_STATE_VIOLATION` | 非法状态迁移请求 | `error` | "Invalid state transition" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | entity_id="test_entity_001", entity_type=1, shm_base_addr=0x1000, auth_hash=32字节 | 默认 | status="ok", assigned_entity_id="test_entity_001" |
| TC-002 | 正常 | 连续注册10个实体 | 默认 | 全部成功，entry_count=10 |
| TC-003 | 边界 | entity_id长度=15 | 默认 | status="error", error="Entity ID must be 16 bytes" |
| TC-004 | 边界 | entity_type=3 | 默认 | status="error", error="Entity type must be 0, 1, or 2" |
| TC-005 | 异常 | 注册表满时注册新实体 | capacity=1 | status="error", error="Registry capacity exhausted" |
| TC-006 | 异常 | 重复注册相同entity_id | 默认 | status="error", error="Entity ID already exists" |
| TC-007 | 性能 | 100个实体并发注册 | 默认 | 全部成功，无竞态条件，耗时<100ms |
| TC-008 | 性能 | 查询操作1000次 | entry_count=100 | 全部成功，平均耗时<1ms |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | SHA-256校验 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(1) | 注册/查询操作常数时间 |
| 内存 | <= 64KB | 注册表头+描述符数组 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 10ms | 单次操作 |

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

将蓝图意图转化为纯函数级Python实现，确保：
- 所有函数返回dict结构
- 全局状态外置为ctx参数
- 遵循三大零原则

### 8.3 测试覆盖

基于第5节测试场景生成验收测试用例，确保：
- 正常路径至少3个用例
- 边界路径至少2个用例
- 异常路径至少2个用例
- 性能路径至少1个用例

### 8.4 安全扫描

验证三大零原则、SHM权限、输入消毒。

### 8.5 授权文件输出

按5字段JSON结构生成授权文件。

---

## 附录A：SHM矢量结构设计

### 注册表头矢量（`registry_header_vector`，固定64B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x52454753` ("REGS") |
| 0x04 | `version` | uint32 | 注册表格式版本，当前为1 |
| 0x08 | `entry_count` | uint32 | 当前有效实体数 |
| 0x0C | `capacity` | uint32 | 最大容量 |
| 0x10 | `free_list_head` | uint32 | 空闲描述符槽位链表头索引 |
| 0x14 | `self_entity_id` | vector[16] | 注册表自身的entity_id（自指） |
| 0x24 | `last_scan_tick` | uint64 | 上一次心跳扫描的全局tick |

### 实体描述符矢量（`entity_descriptor_vector`，固定128B × capacity）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `entity_id` | vector[16] | 唯一标识 |
| 0x10 | `entity_type` | uint32 | 类型枚举 |
| 0x14 | `state` | uint32 | 当前状态 |
| 0x18 | `shm_base_addr` | uint64 | SHM基址 |
| 0x20 | `auth_hash` | vector[32] | 授权文件SHA256前32字节 |
| 0x40 | `heartbeat_timestamp` | uint64 | 最后一次心跳ack时间戳 |
| 0x48 | `create_timestamp` | uint64 | 注册时间 |
| 0x50 | `next_free` | uint32 | 空闲链表下一索引 |
| 0x54 | `reserved` | vector[44] | 对齐与扩展保留 |

---

## 附录B：生命周期状态机

```
INIT ──► ALIVE ──► SUSPEND ──► DEAD ──► RECLAIMED
  │        │          │          │
  │        └─ 异常处理系统可强制迁移
  └─ 注册完成但尚未收到首次心跳 ack
```

- **INIT**: 注册后5个tick内未收到`heartbeat_register()`调用，自动迁移至DEAD。
- **ALIVE**: 正常服务状态，可接收消息、可发布topic。
- **SUSPEND**: 由异常处理系统或配置消费者触发，实体暂停服务但保留SHM矢量。
- **DEAD**: 心跳超时或主动注销，消息路由停止向其投递，注册表保留记录3个tick。
- **RECLAIMED**: SHM管理生产者回收矢量后，描述符槽位归还free_list。

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 商场`main()` | → 注册表 | `registry_register("mall_kernel", TYPE_KERNEL, kernel_shm, null_hash)` 建立自指 |
| 授权文件加载生产者 | → 注册表 | 每注入一个新实体，先调用`registry_register()` |
| SHM矢量管理生产者 | ← 注册表 | 注册表查询`shm_base_addr`合法性时，反向校验 |
| 心跳生产者 | → 注册表 | `registry_list_by_type(0, STATE_ALIVE)` 获取待扫描列表 |
| 消息路由生产者 | → 注册表 | 订阅/发布前验证consumer_id/producer_id存在性 |
| 异常处理系统 | ↔ 注册表 | 异常处理有权强制迁移状态，注册表记录异常日志 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
