# 蓝图文档（定稿版）：BP-0010-auth-loader -- 授权文件加载生产者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0010-auth-loader |
| 蓝图名称 | 授权文件加载生产者 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-10 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

商场的**海关与入境签证处**。外置AICoder生成的授权文件是生产者/消费者进入商场的唯一合法通道。该生产者不生成代码，只**验证、拆解、映射、申报**。

### 2.2 数据来源

`file_system` - 授权文件目录 `./mall_inbound/` 或 `shm://inbound_queue`

### 2.3 数据去向

`shm_vector` - 代码段SHM区、授权索引区

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```python
def auth_validate(
    ctx: dict,
    auth_blob: bytes
) -> dict:
    """
    三层验证：结构校验 → 哈希校验 → 签名校验
    [复杂度]: O(n)
    [安全等级]: READ_ONLY_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | dict | 是 | 非空 | 上下文参数 |
| `auth_blob` | bytes | 是 | 长度>0 | 完整的授权文件原始字节 |

### 3.3 返回值规格

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 控制台输出内容
    "timestamp": float,         # 操作时间戳
    "source": str,              # "auth_loader_producer"
    "metadata": {
        "validation_result": str,   # "passed" / "failed"
        "error_layer": str,         # 失败层级：structure/hash/signature
        "error_detail": str         # 详细错误信息
    },
    "error": str                # 错误信息（仅status="error"时）
}
```

### 3.4 其他函数签名

```python
def auth_watch_path(ctx: dict, path: str, poll_interval_ms: int) -> dict:
    """
    启动监控，周期性扫描目录。
    [复杂度]: O(1)
    """

def auth_inject_functions(ctx: dict, auth_blob: bytes, target_shm_segment: int) -> dict:
    """
    将授权文件拆解并注入SHM。
    [复杂度]: O(n)
    """

def auth_revoke(ctx: dict, entity_id: str, reason: str, immediate: bool) -> dict:
    """
    撤销授权。
    [复杂度]: O(1)
    """

def auth_unload(ctx: dict, entity_id: str) -> dict:
    """
    硬卸载：回收代码段SHM、清理授权索引。
    [复杂度]: O(1)
    """

def auth_list_active(ctx: dict) -> dict:
    """
    返回当前所有有效授权的摘要列表。
    [复杂度]: O(n)
    """

def auth_get_metadata(ctx: dict, entity_id: str) -> dict:
    """
    返回授权文件的元数据副本（不含代码）。
    [复杂度]: O(1)
    """
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | auth_blob非空 | `ERR_AUTH_BLOB_EMPTY` | `error` |
| PRE-002 | auth_blob结构合法 | `ERR_AUTH_CORRUPTED` | `error` |
| PRE-003 | auth_blob哈希匹配 | `ERR_HASH_MISMATCH` | `error` |
| PRE-004 | auth_blob签名有效 | `ERR_SIGNATURE_INVALID` | `error` |
| PRE-005 | SHM空间充足 | `ERR_SHM_INSUFFICIENT` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status必须为"ok"或"error" | 不允许其他值 |
| POST-002 | 注入成功后entry_count递增 | 原子操作 |
| POST-003 | 实体在注册表中注册 | 调用registry_register |

### 4.3 不变量

- **零全局状态**: 不读写任何全局变量
- **零堆分配**: 不使用malloc/free/new/delete
- **零系统调用**: 不进行网络IO、进程管理
- **零副作用**: 不修改输入参数
- **确定性**: 相同输入必得相同输出

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_AUTH_BLOB_EMPTY` | auth_blob为空 | `error` | "Auth blob is empty" |
| `ERR_AUTH_CORRUPTED` | 结构解析失败 | `error` | "Auth file corrupted" |
| `ERR_HASH_MISMATCH` | 代码段哈希不符 | `error` | "Hash mismatch, possible tampering" |
| `ERR_SIGNATURE_INVALID` | AICoder签名无效 | `error` | "Invalid AICoder signature" |
| `ERR_SHM_INSUFFICIENT` | SHM空间不足 | `error` | "Insufficient SHM space" |
| `ERR_INJECTION_FAILED` | 注入失败 | `error` | "Injection failed" |
| `ERR_REVOCATION_RACE` | 撤销时实体正在处理消息 | `error` | "Revocation race condition" |

---

## 5. 测试场景（验收标准）

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | 合法授权文件 | 默认 | status="ok", validation_result="passed" |
| TC-002 | 正常 | 热加载新授权 | 商场运行中 | status="ok", entity_id正确返回 |
| TC-003 | 边界 | 授权文件大小=最小值 | 默认 | status="ok" |
| TC-004 | 边界 | 授权文件大小=最大值 | 默认 | status="ok" |
| TC-005 | 异常 | 哈希错误的授权文件 | 默认 | status="error", error="Hash mismatch" |
| TC-006 | 异常 | 签名伪造的授权文件 | 默认 | status="error", error="Invalid signature" |
| TC-007 | 异常 | 结构损坏的授权文件 | 默认 | status="error", error="Auth file corrupted" |
| TC-008 | 性能 | 1秒内提交100份授权 | 默认 | 全部处理完成，auth_index不溢出 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | SHA-256校验 | Python 3.8+ |
| `json` | 授权文件解析 | Python 3.8+ |
| `struct` | 二进制打包 | Python 3.8+ |
| `time` | 时间戳 | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**: 函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n) | 验证操作线性时间 |
| 内存 | <= 256KB | 授权索引区 |
| 栈帧 | <= 8KB | 无深层递归 |
| 执行时限 | <= 100ms | 单次验证 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ⚠️ | 仅读取授权文件目录，不写入 |
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

### 授权索引头矢量（`auth_index_header`，固定32B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `magic` | uint32 | `0x41555448` ("AUTH") |
| 0x04 | `entry_count` | uint32 | 已加载授权数 |
| 0x08 | `max_entries` | uint32 | 上限 |
| 0x0C | `trust_root_addr` | uint64 | AICoder公钥存储矢量基址 |

### 授权条目矢量（`auth_entry_vector`，固定256B）

| 偏移 | 字段 | 类型 | 说明 |
|-----|------|------|------|
| 0x00 | `entity_id` | vector[16] | 关联实体ID |
| 0x10 | `code_segment_addr` | uint64 | 代码段SHM基址 |
| 0x18 | `code_segment_size` | uint32 | 代码段大小 |
| 0x1C | `entry_point_count` | uint32 | 公开函数入口数 |
| 0x20 | `entry_points_array` | vector[64] | 最多8个入口点 |
| 0x60 | `auth_hash` | vector[32] | 授权文件整体哈希 |
| 0x80 | `load_timestamp` | uint64 | 加载时间 |
| 0x88 | `aicoder_id` | vector[16] | 签发AICoder实例ID |
| 0x98 | `flags` | uint32 | 位标志 |
| 0x9C | `reserved` | vector[36] | 保留 |

---

## 附录B：生命周期状态机

```
FILE_ARRIVED ──► VALIDATING ──► VALIDATED ──► INJECTING ──► REGISTERED ──► ACTIVE
                                     │
                                     ▼
                              REJECTED ──► 写入拒绝日志
ACTIVE ──► SUSPEND ──► UNLOADING ──► DEAD ──► SHM_RECLAIMED
```

---

## 附录C：与商场组件接口关系

| 组件 | 调用方向 | 接口 |
|-----|---------|------|
| 外置AICoder | → 授权加载 | 通过文件系统或SHM注入点提交`.auth`文件 |
| 进程注册表 | ← 授权加载 | `auth_inject_functions()`内部调用`registry_register()` |
| SHM矢量管理 | ← 授权加载 | 申请`target_shm_segment`用于存放代码段 |
| 消息路由 | ← 授权加载 | 加载成功/失败/撤销时广播`auth_event` topic |
| 日志生产者 | ← 授权加载 | 每个关键步骤调用`log_emit()` |
| 配置消费者 | → 授权加载 | 动态调整`poll_interval_ms`或信任根公钥 |

---

*本蓝图由创世Worker提交，待AICoder审核后生成授权文件。*
