# 蓝图文档（定稿版）：BP-0039-SHM-LIFECYCLE -- SHM矢量生命周期管理器

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0039-SHM-LIFECYCLE |
| 蓝图名称 | SHM矢量生命周期管理器 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-12 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-0（创世纪验证级） |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

SHM矢量生命周期管理器负责管理SHM矢量中产品的完整生命周期，实现产品从铸造到终结的八阶段状态机。核心职责包括：

1. **创建（MINT）**：为新产品分配SHM矢量槽位，初始化产品元数据，状态设为MINTED
2. **封印（SEAL）**：对已完成写入的SHM矢量执行SHA-256校验和计算，将校验和写入矢量头部，状态冻结为SEALED
3. **投递（DELIVER）**：将封印后的产品路由至目标消费者的SHM区，更新状态为DELIVERED
4. **归档（ARCHIVE）**：将已结算的产品标记为归档状态，保留审计周期内的只读访问
5. **销毁（VOID）**：回收SHM槽位，将产品从注册表永久注销

本管理器严格遵循《产品本体论与产品法》（law-product.md）定义的产品六阶段生命周期和八状态状态机。

### 2.2 数据来源

| 数据项 | 来源 | 说明 |
|--------|------|------|
| SHM矢量空间 | SHM vec域 | 产品数据实体 |
| 产品注册表 | SHM auth域 | 产品元数据与状态 |
| 消费者注册表 | SHM auth域 | 消费者SHM区位置 |
| 审计日志 | SHM bus域 | 生命周期事件记录 |

### 2.3 数据去向

| 数据项 | 去向 | 说明 |
|--------|------|------|
| 产品状态更新 | SHM auth域 | 状态机转换记录 |
| SHA-256校验和 | SHM vec域 | 写入矢量头部 |
| 生命周期事件 | SHM bus域 | 审计日志 |
| SHM槽位回收 | SHM vec域 | 释放矢量空间 |

### 2.4 来源锁定

本管理器的数据来源锁定为SHM矢量空间，不接收外部输入。

---

## 3. 函数设计

### 3.1 函数签名

```python
def shm_lifecycle_init(
    ctx: dict,
    config: dict
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: WRITE_SHM_CONFIG
    初始化生命周期管理器，加载产品状态机配置。
    """

def shm_product_seal(
    ctx: dict,
    product_id: str
) -> dict:
    """
    [复杂度]: O(N) 其中N为产品数据长度
    [安全等级]: READ_SHM + WRITE_SHM_VEC
    封印产品：计算SHA-256校验和，冻结状态为SEALED。
    """

def shm_product_deliver(
    ctx: dict,
    product_id: str,
    consumer_id: str
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_SHM + WRITE_SHM_VEC + WRITE_SHM_BUS
    投递产品：将封印产品路由至目标消费者SHM区。
    """

def shm_product_archive(
    ctx: dict,
    product_id: str
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_SHM + WRITE_SHM_AUTH
    归档产品：标记为ARCHIVED状态，保留只读访问。
    """

def shm_product_destroy(
    ctx: dict,
    product_id: str
) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_SHM + WRITE_SHM_AUTH + WRITE_SHM_VEC
    销毁产品：回收SHM槽位，永久注销ProductID。
    """
```

### 3.2 参数规格表

#### shm_lifecycle_init

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空，含shm_root, product_registry_ref | 上下文字典 |
| config | dict | 是 | 含archive_ttl_ms, max_products | 生命周期配置 |

**config结构**:

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| archive_ttl_ms | int | [1000, 86400000] | 归档保留时间（毫秒） |
| max_products | int | [1, 100000] | 最大产品数 |
| seal_algorithm | str | 固定"SHA-256" | 封印校验算法 |
| audit_enabled | bool | - | 是否启用生命周期审计 |

#### shm_product_seal

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| product_id | str | 是 | 长度 <= 64，已注册 | 产品唯一标识 |

#### shm_product_deliver

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| product_id | str | 是 | 长度 <= 64，已注册 | 产品唯一标识 |
| consumer_id | str | 是 | 长度 <= 64，已注册 | 目标消费者标识 |

#### shm_product_archive

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| product_id | str | 是 | 长度 <= 64，已注册 | 产品唯一标识 |

#### shm_product_destroy

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| ctx | dict | 是 | 非空 | 上下文字典 |
| product_id | str | 是 | 长度 <= 64，已注册 | 产品唯一标识 |

### 3.3 数据来源定义

| SHM域 | 数据类型 | 预期规模 | 访问模式 |
|-------|----------|----------|----------|
| vec (矢量域) | bytes | 按产品大小 | 读写 |
| auth (授权域) | json | <= max_products条产品记录 | 读写 |
| bus (总线域) | json | 审计事件 | 只写 |

### 3.4 返回值规格

#### shm_lifecycle_init 返回值

```python
{
    "status": str,              # "ok" / "error"
    "manager_id": str,          # 管理器实例ID
    "config_hash": str,         # 配置摘要
    "timestamp": float,
    "error": str
}
```

#### shm_product_seal 返回值

```python
{
    "status": str,              # "ok" / "error"
    "product_id": str,          # 产品ID
    "previous_state": str,      # 封印前状态
    "new_state": str,           # "SEALED"
    "integrity_hash": str,      # SHA-256校验和（十六进制）
    "data_length": int,         # 数据长度（字节）
    "timestamp": float,
    "error": str
}
```

#### shm_product_deliver 返回值

```python
{
    "status": str,              # "ok" / "error"
    "product_id": str,
    "consumer_id": str,
    "previous_state": str,
    "new_state": str,           # "DELIVERED"
    "shm_target_ref": str,      # 目标SHM区引用
    "timestamp": float,
    "error": str
}
```

#### shm_product_archive 返回值

```python
{
    "status": str,              # "ok" / "error"
    "product_id": str,
    "previous_state": str,
    "new_state": str,           # "ARCHIVED"
    "archive_expiry": float,    # 归档过期时间戳
    "timestamp": float,
    "error": str
}
```

#### shm_product_destroy 返回值

```python
{
    "status": str,              # "ok" / "error"
    "product_id": str,
    "previous_state": str,
    "shm_slot_recovered": bool, # SHM槽位是否已回收
    "registry_removed": bool,   # 注册表是否已移除
    "timestamp": float,
    "error": str
}
```

### 3.5 关键设计：产品状态机

本管理器实现的产品状态机与law-product.md第7条完全对齐：

```
合法状态转换：

[UNBORN] ───铸造───> [MINTED]
[MINTED] ───提交───> [SUBMITTED]
[SUBMITTED] ──封印──> [SEALED]
[SEALED] ───投递───> [DELIVERED]
[DELIVERED] ──确认──> [SETTLED]
[DELIVERED] ──超时──> [FORCED]
[SETTLED] ──归档───> [ARCHIVED]
[FORCED] ──归档───> [ARCHIVED]
[ARCHIVED] ──销毁──> [VOID]
```

**状态转换铁律**：
1. **封印不可逆**：SEALED状态不可回退到SUBMITTED或MINTED
2. **结算不可撤**：SETTLED状态不可回退到DELIVERED
3. **终结不可复**：VOID状态不可恢复，ProductID永久注销
4. **错误向前修正**：若产品数据有误，只能由另一个生产者生成修正产品

### 3.6 关键设计：封印操作

封印操作是产品获得不可变性的关键时刻：

```
封印流程：
  1. 校验产品当前状态为SUBMITTED
  2. 读取SHM矢量body数据的完整内容
  3. 计算body数据的SHA-256校验和
  4. 将校验和写入SHM矢量头部checksum字段
  5. 将产品状态从SUBMITTED更新为SEALED
  6. 写入审计日志（含product_id, hash, timestamp）
  7. 返回封印结果（含integrity_hash）
```

### 3.7 关键设计：销毁操作

```
销毁流程：
  1. 校验产品当前状态为ARCHIVED
  2. 检查归档TTL是否已过期
  3. 从产品注册表中移除ProductID
  4. 回收SHM矢量槽位（标记为可用）
  5. 写入最终审计日志
  6. 返回销毁结果
```

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | ctx 非空且包含必需键 | `ERR_CTX_EMPTY` | `error` |
| PRE-002 | config 非空且包含必需字段 | `ERR_CONFIG_INVALID` | `error` |
| PRE-003 | archive_ttl_ms 在 [1000, 86400000] 范围内 | `ERR_ARCHIVE_TTL_INVALID` | `error` |
| PRE-004 | max_products 在 [1, 100000] 范围内 | `ERR_MAX_PRODUCTS_INVALID` | `error` |
| PRE-005 | product_id 长度 <= 64 | `ERR_PRODUCT_ID_INVALID` | `error` |
| PRE-006 | product_id 已在产品注册表中注册 | `ERR_PRODUCT_NOT_FOUND` | `error` |
| PRE-007 | 封印时产品状态必须为SUBMITTED | `ERR_SEAL_STATE_INVALID` | `error` |
| PRE-008 | 投递时产品状态必须为SEALED | `ERR_DELIVER_STATE_INVALID` | `error` |
| PRE-009 | consumer_id 已在消费者注册表中注册 | `ERR_CONSUMER_NOT_FOUND` | `error` |
| PRE-010 | 归档时产品状态必须为SETTLED或FORCED | `ERR_ARCHIVE_STATE_INVALID` | `error` |
| PRE-011 | 销毁时产品状态必须为ARCHIVED | `ERR_DESTROY_STATE_INVALID` | `error` |
| PRE-012 | 销毁时归档TTL必须已过期 | `ERR_ARCHIVE_TTL_NOT_EXPIRED` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | 封印后integrity_hash为64字符十六进制字符串 | SHA-256输出长度 |
| POST-003 | 封印后产品状态为SEALED | 状态转换正确 |
| POST-004 | 投递后产品状态为DELIVERED | 状态转换正确 |
| POST-005 | 归档后产品状态为ARCHIVED | 状态转换正确 |
| POST-006 | 销毁后产品从注册表移除 | 永久注销 |
| POST-007 | 销毁后SHM槽位标记为可用 | 资源回收 |
| POST-008 | 所有操作均写入审计日志 | 可追溯性 |

### 4.3 不变量

- **零全局状态**：不读写任何全局变量，所有状态通过ctx参数传入
- **零堆分配**：不使用动态内存分配
- **零系统调用**：不进行文件IO、网络IO、进程管理
- **零副作用**：不修改输入参数，所有变更通过返回值和SHM写入传递
- **确定性**：相同输入必得相同输出
- **封印不可逆**：SEALED状态的产品永远不可修改
- **终结不可复**：VOID状态的产品永远不可恢复
- **审计完整性**：每次状态转换必须产生审计记录

### 4.4 错误契约表

| 错误码 | 触发条件 | 返回status | 返回error |
|--------|----------|------------|-----------|
| `ERR_CTX_EMPTY` | ctx为空或缺少必需键 | `error` | "ctx is empty or missing required keys" |
| `ERR_CONFIG_INVALID` | config为空或缺少必需字段 | `error` | "config is empty or missing required fields" |
| `ERR_ARCHIVE_TTL_INVALID` | archive_ttl_ms超出范围 | `error` | "archive_ttl_ms must be in [1000, 86400000]" |
| `ERR_MAX_PRODUCTS_INVALID` | max_products超出范围 | `error` | "max_products must be in [1, 100000]" |
| `ERR_PRODUCT_ID_INVALID` | product_id长度 > 64 | `error` | "product_id length must be <= 64" |
| `ERR_PRODUCT_NOT_FOUND` | product_id未注册 | `error` | "product not found in registry" |
| `ERR_SEAL_STATE_INVALID` | 产品状态不是SUBMITTED | `error` | "product must be in SUBMITTED state to seal" |
| `ERR_DELIVER_STATE_INVALID` | 产品状态不是SEALED | `error` | "product must be in SEALED state to deliver" |
| `ERR_CONSUMER_NOT_FOUND` | consumer_id未注册 | `error` | "consumer not found in registry" |
| `ERR_ARCHIVE_STATE_INVALID` | 产品状态不是SETTLED或FORCED | `error` | "product must be SETTLED or FORCED to archive" |
| `ERR_DESTROY_STATE_INVALID` | 产品状态不是ARCHIVED | `error` | "product must be in ARCHIVED state to destroy" |
| `ERR_ARCHIVE_TTL_NOT_EXPIRED` | 归档TTL未过期 | `error` | "archive TTL has not expired yet" |
| `ERR_SHM_CORRUPTED` | SHM数据完整性校验失败 | `error` | "SHM data integrity check failed" |

---

## 5. 测试场景

### 5.1 正常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-001 | 标准ctx + 标准config | archive_ttl_ms=60000, max_products=1000 | init返回status="ok"，manager_id非空 |
| TC-002 | 已注册产品（状态SUBMITTED） | 任意 | seal返回status="ok"，new_state="SEALED"，integrity_hash为64字符hex |
| TC-003 | 已封印产品（状态SEALED）+ 已注册消费者 | 任意 | deliver返回status="ok"，new_state="DELIVERED" |
| TC-004 | 已投递产品（状态SETTLED） | 任意 | archive返回status="ok"，new_state="ARCHIVED" |
| TC-005 | 已归档产品（TTL已过期） | 任意 | destroy返回status="ok"，shm_slot_recovered=true，registry_removed=true |

### 5.2 边界路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-006 | archive_ttl_ms=1000 | 最小TTL | init返回status="ok" |
| TC-007 | archive_ttl_ms=86400000 | 最大TTL | init返回status="ok" |
| TC-008 | max_products=1 | 最小产品数 | init返回status="ok" |
| TC-009 | max_products=100000 | 最大产品数 | init返回status="ok" |
| TC-010 | product_id长度=64 | 最大长度 | seal返回status="ok" |
| TC-011 | 产品数据为空（0字节） | 任意 | seal返回status="ok"，integrity_hash为空数据的SHA-256 |
| TC-012 | 产品数据为最大允许大小 | 任意 | seal返回status="ok"，计算正确hash |

### 5.3 异常路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-013 | ctx={} | 任意 | init返回status="error"，error含ERR_CTX_EMPTY |
| TC-014 | config={} | 任意 | init返回status="error"，error含ERR_CONFIG_INVALID |
| TC-015 | 未注册的product_id | 任意 | seal返回status="error"，error含ERR_PRODUCT_NOT_FOUND |
| TC-016 | 产品状态为MINTED（非SUBMITTED） | 任意 | seal返回status="error"，error含ERR_SEAL_STATE_INVALID |
| TC-017 | 产品状态为SUBMITTED（非SEALED） | 任意 | deliver返回status="error"，error含ERR_DELIVER_STATE_INVALID |
| TC-018 | 未注册的consumer_id | 任意 | deliver返回status="error"，error含ERR_CONSUMER_NOT_FOUND |
| TC-019 | 产品状态为DELIVERED（非SETTLED/FORCED） | 任意 | archive返回status="error"，error含ERR_ARCHIVE_STATE_INVALID |
| TC-020 | 产品状态为SEALED（非ARCHIVED） | 任意 | destroy返回status="error"，error含ERR_DESTROY_STATE_INVALID |
| TC-021 | 归档TTL未过期 | 任意 | destroy返回status="error"，error含ERR_ARCHIVE_TTL_NOT_EXPIRED |
| TC-022 | product_id长度=65 | 任意 | seal返回status="error"，error含ERR_PRODUCT_ID_INVALID |

### 5.4 性能路径

| 编号 | 输入 | 配置 | 预期输出 |
|------|------|------|----------|
| TC-023 | 1MB产品数据封印 | 任意 | seal耗时 < 100ms |
| TC-024 | 连续1000次封印操作 | 任意 | 无内存泄漏，每次hash正确 |
| TC-025 | 连续10000次销毁操作 | 任意 | 所有SHM槽位正确回收 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `hashlib` | SHA-256校验和计算 | Python 3.8+ |
| `time` | 时间戳获取与TTL计算 | Python 3.8+ |

### 6.2 外部依赖

**零外部依赖**。不依赖任何第三方库。

### 6.3 蓝图间依赖

| 依赖蓝图 | 依赖类型 | 依赖内容 |
|----------|----------|----------|
| BP-0003（SHM矢量管理器） | 功能依赖 | SHM矢量的读写、创建、销毁接口 |
| BP-0004（异常处理） | 错误处理 | MallException体系与错误码定义 |

### 6.4 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(N) per seal（N为数据长度） | SHA-256计算 |
| 内存 | <= 256KB | 管理器状态与中间数据 |
| 栈帧 | <= 8KB | 单次函数调用最大栈深 |
| 执行时限 | seal <= 100ms（1MB数据） | 封印操作上限 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | YES | 不进行任何文件读写 |
| 网络零IO | YES | 不进行任何网络通讯 |
| 零全局副作用 | YES | 不修改任何全局状态 |
| 输入消毒 | YES | 对所有输入参数进行类型、范围、存在性校验 |
| 确定性输出 | YES | 相同输入产生相同输出 |
| SHM权限合规 | YES | 仅访问授权的SHM矢量段，封印操作仅写入checksum字段 |
| 栈帧安全 | YES | 不存在递归，最大栈深 <= 8KB |
| 零堆分配 | YES | 不使用动态内存分配 |
| 零OOP | YES | 不使用类定义、继承、self引用 |
| 零系统调用 | YES | 不调用操作系统API |

### 7.2 stdout特许

本管理器不直接向stdout输出。所有诊断信息通过返回值dict传递。

---

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证，确保13条前置条件全部有对应错误码 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现，禁止class/struct/self，所有状态通过ctx参数传递 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例，覆盖TC-001至TC-025全部25个用例 |
| 安全扫描 | 验证三大零原则、SHM权限合规、封印不可逆性、终结不可复性 |
| 授权文件输出 | 按5字段JSON结构生成授权文件（auth_header, design_meta, source_code, compiled_binary, verification_report） |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据（BP-0039-SHM-LIFECYCLE, v1.0.0, Genesis-Worker-001） |
| `design_meta` | 第3节函数设计（5个函数签名、参数规格、返回值规格、状态机定义、封印算法） |
| `source_code` | AICoder根据蓝图意图生成的纯函数Python实现 |
| `compiled_binary` | PYC编译产物（base64编码） |
| `verification_report` | 基于第5节25个测试场景的验收测试报告 |

### 8.3 特别审查要求

1. **状态机完整性**：确保实现的状态机与law-product.md第7条完全一致，八状态（UNBORN/MINTED/SUBMITTED/SEALED/DELIVERED/SETTLED/FORCED/ARCHIVED/VOID）全部实现
2. **封印不可逆**：确保SEALED状态的产品数据永远不可被修改，SHA-256校验和写入后不可覆盖
3. **SHA-256正确性**：确保校验和计算使用标准hashlib.sha256，输出为64字符十六进制字符串
4. **审计完整性**：确保每次状态转换都产生审计记录，包含product_id、前后状态、时间戳
5. **销毁安全性**：确保只有ARCHIVED状态且TTL已过期的产品才能被销毁

---

*蓝图审核状态：待审核*
*设计者：Genesis-Worker-001*
*AICoder评估：待外置AICoder审查*
