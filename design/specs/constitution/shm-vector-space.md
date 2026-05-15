# SHM矢量空间总则

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## The Constitution of Shared Memory Vector Space

**版本**: v1.0
**制定者**: 商场模式架构委员会
**上位法**: 《微型商场创世纪总纲》第17条、《函数范式宪章》第七章、《异常处理总则》铁律一
**核心原则**: SHM是商场的唯一血脉，矢量是数据的唯一合法容器，闸口是访问的唯一合法通道

---

## 目录

1. [总则序言](#一总则序言)
2. [根本原则](#二根本原则)
3. [矢量生命周期宪法](#三矢量生命周期宪法)
4. [矢量创建与分配规范](#四矢量创建与分配规范)
5. [矢量读写规范](#五矢量读写规范)
6. [内存安全与边界铁律](#六内存安全与边界铁律)
7. [并发访问与隔离模型](#七并发访问与隔离模型)
8. [矢量命名空间宪法](#八矢量命名空间宪法)
9. [心跳监控与存活检测](#九心跳监控与存活检测)
10. [封印、校验与归档](#十封印校验与归档)
11. [销毁、清扫与坍缩](#十一销毁清扫与坍缩)
12. [错误码体系](#十二错误码体系)
13. [与函数的交互契约](#十三与函数的交互契约)
14. [SHM污染与熔断](#十四shm污染与熔断)
15. [合规Checklist](#十五合规checklist)

---

## 一、总则序言

> **"SHM是商场的血脉，矢量是血细胞，闸口是血管壁。血脉不可污染，细胞不可越界，管壁不可穿透。"**

共享内存（SHM）是创世纪体系中生产者、消费者、商场之间**唯一合法的数据流通通道**。任何试图绕过SHM的通讯手段——Queue、Pipe、socket、mmap（非shared_memory模块）——皆为违宪。

本总则将散落于《函数范式宪章》第七章、《异常处理总则》铁律一、各协议文档中的SHM规则，统一收编为独立宪法，作为所有SHM相关行为的最高裁判依据。

---

## 二、根本原则

### 原则一：SHM唯一性原则

SHM是创世纪体系中**唯一合法的进程间通讯机制**。严禁使用以下替代手段：

| 禁止手段 | 违宪等级 | 处置 |
|----------|----------|------|
| `multiprocessing.Queue` / `Pipe` | 严重违宪 | AICoder审查拒绝 |
| `socket` / 网络套接字 | 严重违宪 | AICoder审查拒绝 |
| `mmap`（非 `shared_memory` 模块） | 严重违宪 | AICoder审查拒绝 |
| 文件系统中转 | 严重违宪 | AICoder审查拒绝 |
| 全局变量 / 单例模式 | 严重违宪 | 总纲第15条，零全局状态 |

### 原则二：商场主权原则

商场主进程拥有SHM的**绝对主权**：

- **创建权**：只有商场主进程有权创建新的SHM块（`create=True`）
- **销毁权**：只有商场调度器有权调用 `shm_destroy`
- **闸口权**：所有SHM访问必须经过 `shm_gate_check` 审批
- **映射权**：Worker进程的SHM映射由商场通过 `WorkerShmMap` 精确控制

生产者与消费者只能以 `create=False` 模式**附加**到商场已创建的SHM段。

### 原则三：显式矢量原则

SHM中的每一段数据**必须注册为矢量**（Vector）。未注册的数据视为**宇宙尘埃**，商场有权随时清除。

### 原则四：函数盲区原则

函数内部**禁止直接操作共享内存**。函数的输入输出通过调用方传入的缓冲区指针交互，SHM的映射、生命周期管理、矢量编排由商场封装层全权负责。函数保持**纯计算blindness**，对SHM的存在一无所知。

---

## 三、矢量生命周期宪法

### 3.1 六状态定义

| 状态 | 值 | 含义 | 允许操作 |
|------|-----|------|----------|
| **UNBORN** | 0 | 已注册但未分配物理SHM | 无（等待创建） |
| **ACTIVE** | 1 | 正常运行，可读写 | 读、写、封印 |
| **SEALED** | 2 | 数据写入完成，冻结只读 | 读、校验、归档 |
| **ARCHIVED** | 3 | 数据已消费，保留审计 | 读、销毁 |
| **CORRUPTED** | 4 | 校验失败或心跳丢失 | 销毁 |
| **VOID** | 5 | 物理内存已回收，名称注销 | 无（终态） |

### 3.2 合法状态转换

```
                    shm_create()
    [UNBORN] ──────────────────► [ACTIVE]
                                    │
                    shm_seal()      │ shm_mark_corrupted()
                                    ▼                    ▼
                              [SEALED]            [CORRUPTED]
                                │                      │
                    shm_archive()│                      │
                                ▼                      │
                            [ARCHIVED]                 │
                                │                      │
                                └──────────┬───────────┘
                                           │
                                     shm_destroy()
                                           │
                                           ▼
                                       [VOID]

    注：任意非VOID状态均可直接执行 shm_destroy() 进入 VOID
```

### 3.3 状态转换铁律

1. **封印不可逆**：ACTIVE → SEALED 后，禁止任何写入操作
2. **损坏不可恢复**：CORRUPTED 状态的矢量禁止读和写，只能销毁
3. **VOID不可复生**：VOID 是终态，矢量名称从注册表中永久注销
4. **归档只读**：ARCHIVED 状态仅保留审计用途，禁止写入

---

## 四、矢量创建与分配规范

### 4.1 创建参数约束

| 参数 | 约束 | 违反后果 |
|------|------|----------|
| `domain` | 必须为 `vec`/`hb`/`bp`/`auth`/`bus` 之一 | `SHM_ERR_TYPE_MISMATCH` |
| `owner` | 非空字符串 | `SHM_ERR_TYPE_MISMATCH` |
| `size` | 正整数（> 0） | `SHM_ERR_OUT_OF_BOUNDS` |
| `dtype` | 必须为 `bytes`/`int32`/`float64`/`json` 之一 | `SHM_ERR_TYPE_MISMATCH` |

### 4.2 唯一性保证

矢量全名通过 `SHA-256哈希前8字符 + 时间戳 + 序列号` 生成。若已存在于registry中，返回 `SHM_ERR_ALREADY_EXISTS`。

### 4.3 创建即ACTIVE

矢量创建成功后，状态立即为 `STATE_ACTIVE`，可被读写。

### 4.4 系统矢量预创建

`shm_universe_init()` 时自动创建4个系统矢量：

| 系统矢量 | 域 | 用途 | 初始容量 |
|----------|-----|------|----------|
| 商场内部消息总线 | `bus` | 商场内部事件传递 | 1MB, bytes |
| 注册表镜像 | `vec` | 函数目录与状态快照 | 256KB, json |
| 蓝图提交收件箱 | `bp` | 生产者蓝图投递 | 512KB, json |
| 授权书金库 | `auth` | AICoder授权文件存放 | 512KB, json |

### 4.5 容量预分配铁律

创建时必须指定 `capacity`，分配 `capacity + sizeof(ShmVectorMeta)` 字节。**禁止动态扩容**。容量不足时，必须销毁旧矢量并创建新矢量。

### 4.6 创建契约声明

创建矢量时必须声明以下元数据：

```python
vector_id = shm_create(
    size=estimated_size,
    access_mode='READ_ONLY',        # READ_ONLY / WRITE_ONLY / READWRITE
    owner_worker='worker_name',     # 创建者标识
    ttl=compute_time_estimate * 2,  # 生存周期（秒）
    audit_chain=True               # 是否开启访问审计链
)
```

---

## 五、矢量读写规范

### 5.1 闸口前置检查

所有SHM访问**必须先通过闸口**：

```python
allowed, error_code, reason = shm_gate_check(ctx, caller_id, vec_name, "write")
if not allowed:
    return error_code
```

闸口检查项：
- 矢量是否存在
- 矢量状态是否允许该操作
- 调用者是否为矢量所有者（或经授权）
- 访问类型是否与权限模式匹配

### 5.2 写入规范

- 支持**偏移写入**：写入位置为 `_HEADER_SIZE + offset`
- 写入成功返回**实际写入字节数**
- 每次写入自动更新尾部的**最后操作时间戳**
- 非owner且非无主（owner_pid=0）的写入返回 `SHM_ERR_PERMISSION_DENIED`

### 5.3 读取规范

- 支持**偏移读取**和**长度限制**：`length` 为 None 时读取到矢量末尾
- 读取成功返回 `(SHM_OK, data)` 二元组
- 读取失败返回 `(error_code, b"")` 二元组

### 5.4 写隔离原则

消费者默认对任何SHM矢量处于**只读状态**。若需回写（如ACK确认），必须申请**独立的ACK矢量**，不得写回生产者的原始数据区。

### 5.5 乐观锁读取

读取前检查 `version`，读取后校验 `version` 是否变化。若变化则重试（最多3次），超过则返回 `MALL_ERR_SHM_CORRUPTION`。

### 5.6 写入双递增版本号

写入开始时原子递增 `version`（标记写入中），写入完成后再次原子递增 `version`（标记写入完成），确保读者能检测到写入中状态。

---

## 六、内存安全与边界铁律

### 6.1 头-体-尾三段布局

每块SHM采用固定布局：

```
+------------------+------------------------+------------------+
|     Header       |         Body           |      Tail        |
|     82 bytes     |     variable length    |     16 bytes     |
+------------------+------------------------+------------------+
| magic: 0x4D414C4C|                        | magic_end:       |
| version          |     业务数据            |   0x4C4C414D     |
| state            |                        | timestamp        |
| dtype            |                        | reserved         |
| owner            |                        |                  |
| data_size        |                        |                  |
| checksum         |                        |                  |
+------------------+------------------------+------------------+
```

### 6.2 越界访问铁律

- **写入越界**：`offset + len(data) > meta["size"]` 时返回 `SHM_ERR_OUT_OF_BOUNDS`
- **读取越界**：`offset + length > meta["size"]` 时返回 `SHM_ERR_OUT_OF_BOUNDS`
- **越界写即污染**：任何越界写操作，无论意图如何，一律视为SHM污染，触发三级熔断（见第十四章）

### 6.3 类型安全铁律

`dtype` 字段必须严格匹配。尝试以错误类型解析数据将触发闸口拒绝。类型不匹配返回 `SHM_ERR_TYPE_MISMATCH`。

### 6.4 CRC32完整性校验

每个SHM矢量元数据中包含 `checksum` 字段（CRC32），可调用 `shm_vector_checksum_verify` 验证数据完整性。校验失败返回 `SHM_ERR_CHECKSUM_MISMATCH`。

### 6.5 封印不可违

SEALED状态的矢量，任何写入尝试都将被闸口拒绝，返回 `SHM_ERR_SEALED_VIOLATION`。

---

## 七、并发访问与隔离模型

### 7.1 乐观锁并发模型

SHM采用**乐观锁**而非互斥锁：

- **读取**：通过 `version` 检测并发写入，最多重试3次
- **写入**：通过 `version` 双递增标记写入区间
- **创建/销毁**：由商场主控进程串行执行，无并发竞争

### 7.2 命名空间隔离

进程只能看到与自己 `owner_id` 匹配的SHM名称列表。商场通过 `shm_gate_check` 过滤全局注册表，实现命名空间级别的隔离。

### 7.3 地址空间隔离

生产者与消费者进程**不得持有**指向对方SHM的 `SharedMemory` 对象引用，除非通过 `shm_gate_check` 显式授权。

### 7.4 WorkerShmMap精确映射

每个Worker进程通过 `WorkerShmMap` 结构体接收允许的SHM映射列表：

```python
WorkerShmMap = {
    "read_keys": [...],   # 允许读取的矢量键名列表，上限16个
    "write_keys": [...]   # 允许写入的矢量键名列表，上限16个
}
```

超出列表的访问返回 `MALL_ERR_SHM_ACCESS_DENIED`。

### 7.5 单线程事件循环

商场主控层为**单线程事件循环**，通过SHM原子操作与多Worker进程交互，无共享内存锁。

---

## 八、矢量命名空间宪法

### 8.1 命名格式

矢量全名采用三段式命名法：

```
mall_{domain}_{owner}_{short_id}_{timestamp}
```

| 段 | 说明 | 示例 |
|----|------|------|
| `mall_` | 固定前缀，神圣不可更改 | `mall_` |
| `{domain}` | 域标识 | `vec`/`hb`/`bp`/`auth`/`bus` |
| `{owner}` | 所有者标识 | `keyboard_echo` |
| `{short_id}` | SHA-256哈希前8字符 | `a1b2c3d4` |
| `{timestamp}` | Unix时间戳 | `1715299200` |

### 8.2 URI风格键名规范

SHM矢量键名可采用URI风格：

```
shm://mall/...              商场内部矢量
shm://aicoder/{token}/...   AICoder专用矢量
shm://producer/{id}/...     生产者私有矢量
shm://repo/{func_name}      函数仓库矢量
```

**键名合法性规则**：
- 必须以 `shm://` 开头
- 总长度 <= 256
- 只允许 `[a-zA-Z0-9_/.-]`
- 禁止 `..` 序列（防路径遍历）
- 禁止空段
- 必须以字母或数字结尾

### 8.3 域分类宪法

| 域 | 标识 | 用途 | 访问权限 |
|----|------|------|----------|
| 矢量域 | `vec` | 业务数据矢量 | Worker按需读写 |
| 心跳域 | `hb` | 进程心跳检测 | 所有者写，商场读 |
| 蓝图域 | `bp` | 蓝图提交收件箱 | 生产者写+封印，AICoder读 |
| 授权域 | `auth` | 授权文件金库 | AICoder写+封印，商场读 |
| 总线域 | `bus` | 商场内部消息总线 | 商场独占读写 |
| **指标域** | **`metrics`** | **交易指标数据（宏观/中观/微观）** | **商场写，相关角色只读** |

#### 8.3.1 指标域（metrics）详细规范

指标域用于存储商场产出的结构化交易指标，作为"公共信息产品"供管理者和消费者决策参考。

**指标域子空间**：

| 子空间 | URI | 内容 | 访问权限 |
|--------|-----|------|----------|
| 全局指标 | `shm://mall/metrics/global` | δ_mall、throughput_mall、congestion_index | 所有角色只读 |
| 生产者指标 | `shm://mall/metrics/producer/{id}` | δ_producer、queue_depth per producer | 管理者、相关消费者只读 |
| 交易指标 | `shm://mall/metrics/tx/{tx_id}` | tx_latency per transaction | 当事方只读 |
| 建议事件 | `shm://mall/metrics/suggestions` | 主动建议事件队列 | 目标角色只读 |

**指标数据属性**：
- 指标数据为**SEALED态只读产品**，商场写入后立即封印
- 指标数据TTL：宏观指标保留1小时，中观指标保留24小时，微观指标保留任务终结后1小时
- 指标数据格式：JSON，定长结构，禁止变长嵌套

---

## 九、心跳监控与存活检测

### 9.1 心跳写入

`shm_heartbeat_beat` 以大端double格式写入当前 `time.time()` 到心跳矢量。进程必须每N秒（默认3秒）调用一次。

### 9.2 心跳检测

`shm_heartbeat_check` 读取心跳矢量前8字节，解析为double时间戳，与当前时间比较判断是否超时。商场主循环每M秒（默认5秒）扫描所有hb矢量。

### 9.3 心跳域限制

心跳矢量必须属于 `hb` 域。非hb域的心跳检测返回 `SHM_ERR_TYPE_MISMATCH`。

### 9.4 超时处置

连续两次检测失败（超时 2×N），商场自动将该进程**所有关联矢量**标记为 `CORRUPTED`，启动回收流程。

### 9.5 幽灵进程清除

已失去心跳响应但仍在操作系统进程表中存在的进程为**幽灵进程**。清除函数：

```python
def ghost_process_purge(pid: int, shm_token: str) -> bool:
    """清除幽灵进程并回收SHM令牌"""
    pass
```

---

## 十、封印、校验与归档

### 10.1 封印操作

封印将矢量从 ACTIVE 冻结为 SEALED：

- 计算body数据的 **SHA-256 校验和**
- 将校验和写入header的 `checksum` 字段（32字节）
- 状态字节更新为 SEALED
- 封印只能从 ACTIVE 状态执行，重复封印返回 `SHM_ERR_INVALID_STATE`

### 10.2 校验操作

`shm_vec_verify` 仅对 SEALED 状态矢量有效：
- 重新计算body的SHA-256
- 与存储的checksum比对
- 返回 `(error_code, checksum)` 二元组

### 10.3 归档操作

归档将矢量从 SEALED 转为 ARCHIVED：
- 表示数据已被消费，保留用于审计
- 归档只能从 SEALED 状态执行
- ARCHIVED 状态禁止写入，允许读取

---

## 十一、销毁、清扫与坍缩

### 11.1 销毁操作

`shm_vec_destroy` 从三处同时删除：
- `registry`（注册表）
- `shm_blocks`（物理SHM块）
- `name_index`（名称索引）

若owner在name_index中的矢量列表清空，则删除该owner条目。

### 11.2 销毁权限

只有商场调度器有权调用 `shm_destroy`。销毁时必须记录原因和审计日志。

### 11.3 销毁通知

销毁SHM时向所有持有该矢量的进程发送信号通知映射失效。持有进程收到后必须调用 `shm_vector_close`。

### 11.4 销毁场景策略

| 场景 | 策略 |
|------|------|
| 任务完成 | 销毁临时矢量 |
| 任务回滚 | 销毁该任务所有关联矢量 |
| 报告归档 | 保留报告矢量，销毁其他 |
| TTL过期 | 商场定时任务自动销毁 |
| 心跳超时 | 标记CORRUPTED后销毁 |

### 11.5 宇宙清扫

`shm_universe_sweep` 扫描所有 `STATE_CORRUPTED` 和 `STATE_VOID` 状态的矢量并批量销毁。

### 11.6 宇宙坍缩

`shm_universe_collapse` 销毁所有矢量，清空registry和shm_blocks，用于商场关闭。

---

## 十二、错误码体系

### 12.1 Python层错误码（SHM系列）

| 错误码 | 名称 | 含义 |
|--------|------|------|
| `SHM_OK = 0` | 成功 | 操作正常完成 |
| `SHM_ERR_NOT_FOUND = 1001` | 未找到 | 矢量不存在 |
| `SHM_ERR_PERMISSION_DENIED = 1002` | 权限不足 | 非所有者或未授权 |
| `SHM_ERR_SEALED_VIOLATION = 1003` | 封印违规 | 尝试写入已封印矢量 |
| `SHM_ERR_OUT_OF_BOUNDS = 1004` | 越界 | 偏移+长度超出矢量容量 |
| `SHM_ERR_TYPE_MISMATCH = 1005` | 类型不匹配 | domain/dtype参数非法 |
| `SHM_ERR_ALREADY_EXISTS = 1006` | 已存在 | 矢量名称冲突 |
| `SHM_ERR_INVALID_STATE = 1007` | 状态非法 | 当前状态不允许该操作 |
| `SHM_ERR_CHECKSUM_MISMATCH = 1008` | 校验失败 | CRC32/SHA-256不匹配 |

### 12.2 C层错误码（MALL系列）

| 错误码 | 名称 | 含义 |
|--------|------|------|
| `MALL_ERR_SHM_ALLOC_FAIL = 401` | 分配失败 | SHM物理内存不足 |
| `MALL_ERR_SHM_ACCESS_DENIED = 402` | 越权访问 | 键名不在WorkerShmMap中 |
| `MALL_ERR_SHM_CORRUPTION = 403` | 数据损坏 | version校验失败 |
| `MALL_ERR_SHM_SERIALIZE_FAIL = 404` | 序列化失败 | json/bytes编解码错误 |

### 12.3 AER异常码（SHM相关）

| 编码 | 含义 |
|------|------|
| `F-000-001` | 商场SHM初始化失败 |
| `C-100-002` | 生产者SHM写入越界 |
| `C-100-003` | 生产者幽灵化（心跳超时） |

---

## 十三、与函数的交互契约

### 13.1 SHM通信公理

```
forall input in Input:  input = shm_read(vector_id)
forall output in Output: output = shm_write(data) -> vector_id
```

所有数据交换必须通过SHM矢量空间。函数间通信通过**矢量ID传递**，而非对象引用。

### 13.2 ctx参数传递

所有SHM操作函数接收 `ctx: dict` 作为第一个参数（宇宙上下文），包含三个核心字典：

```python
ctx = {
    "registry": {},      # 矢量注册表
    "name_index": {},    # 名称索引
    "shm_blocks": {}     # 物理SHM块映射
}
```

### 13.3 纯函数接口

所有SHM接口均为**顶层函数**，无类封装。返回值统一为dict结构或错误码。

### 13.4 AICoder交互边界

AICoder在SHM中的身份为 `OWNER_AICODER`，其交互限定为双区域：

| 通道 | 方向 | 权限 |
|------|------|------|
| `mall_bp_inbox` | 商场 → AICoder | 只读（读取蓝图） |
| `mall_auth_vault` | AICoder → 商场 | 写+封印（提交授权文件） |
| `mall_hb_aicoder` | 商场 ↔ AICoder | 心跳与任务调度 |

AICoder不得直接创建SHM，不得直接操作生产者或消费者的私有矢量，不得绕过闸口。

### 13.5 Worker间禁止直接调用

Worker间不得直接调用彼此的函数，必须通过商场调度器间接耦合。数据流：Worker_A → SHM矢量 → 商场调度 → Worker_B。

---

## 十四、SHM污染与熔断

### 14.1 污染定义

以下行为一律视为**SHM污染**：

| 行为 | 污染等级 |
|------|----------|
| 越界写操作 | 致命 |
| 以错误dtype解析数据 | 严重 |
| 写入已封印矢量 | 严重 |
| 非授权跨域访问 | 严重 |
| 心跳超时导致数据孤岛 | 一般 |

### 14.2 三级熔断机制

任何SHM污染触发以下三级熔断：

```
第一级：冻结
  └── 冻结该实体所有SHM写权限

第二级：取证
  └── 启动内存隔离区（Quarantine Zone）
  └── 复制污染数据用于取证

第三级：追责
  └── 通知AICoder重新评估该实体蓝图
  └── 记录AER异常事件记录
```

### 14.3 隔离区取证

```python
def shm_quarantine_zone_alloc(polluted_offset: int, size: int) -> int:
    """在隔离区分配取证内存，返回隔离区新偏移地址"""
    pass
```

### 14.4 污染处置与信用追溯

- 污染事件记入生产者信用档案
- 累计3次严重污染，永久禁止该生产者提交蓝图
- AICoder签发的授权文件若导致SHM污染，AICoder节点信用扣减

---

## 十五、合规Checklist

### 矢量创建阶段

- [ ] domain 为合法值（vec/hb/bp/auth/bus）
- [ ] owner 为非空字符串
- [ ] size 为正整数
- [ ] dtype 为合法值（bytes/int32/float64/json）
- [ ] 容量预分配，禁止动态扩容
- [ ] 创建契约声明完整（access_mode/owner/ttl/audit_chain）

### 矢量读写阶段

- [ ] 所有访问经过 shm_gate_check 闸口
- [ ] 写入偏移 + 数据长度 <= 矢量容量
- [ ] 读取偏移 + 长度 <= 矢量容量
- [ ] 非owner不写入（除非owner_pid=0）
- [ ] SEALED矢量禁止写入
- [ ] CORRUPTED/VOID矢量禁止一切操作

### 矢量生命周期阶段

- [ ] 封印前状态为 ACTIVE
- [ ] 封印后SHA-256校验和已写入
- [ ] 归档前状态为 SEALED
- [ ] 销毁时三处同步清理（registry/shm_blocks/name_index）
- [ ] TTL到期或心跳超时触发自动回收

### 安全与隔离阶段

- [ ] WorkerShmMap 精确映射（read_keys/write_keys 各<=16）
- [ ] 进程间地址空间隔离
- [ ] 命名空间按owner过滤
- [ ] 心跳矢量仅限 hb 域
- [ ] AICoder仅访问双区域（bp_inbox/auth_vault）

---

## 宪章签署

> **"SHM是商场的血脉，矢量是血细胞，闸口是血管壁。血脉不可污染，细胞不可越界，管壁不可穿透。"**

本总则自发布之日起生效，所有进入商场的Worker、AICoder、生产者、消费者，均须遵守上述法条。违反者，商场调度器有权冻结权限，安全Worker有权熔断处置，审计系统有权追溯信用。

**商场模式架构委员会**
**版本**: v1.0
**状态**: 正式生效

---

*"SHM唯一性不可动摇，闸口审批不可绕过，边界铁律不可逾越。违者熔断，无赦。"*
