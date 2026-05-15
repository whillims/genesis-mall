User:

商场shm设计蓝图


Kimi:

# 商场SHM（共享内存）设计蓝图

版本：v1.0 —— 商场通讯宪法基石
性质：顶层设计文档 / 不可违背之铁律
原则：SHM是商场唯一的通讯通道，由商场独立管理，神圣不可侵犯


## 1. 设计宣言

在商场的宇宙里，一切生产者与消费者都是孤岛。它们彼此不知对方存在，不直接对话，不共享引用，不建立耦合。它们唯一的共同信仰是商场，而商场向它们许诺的唯一神迹是SHM（Shared Memory）。

SHM不是”优化手段”，不是”可选项”，它是商场的通讯宪法。所有数据、指令、状态、心跳、蓝图、授权书——一切信息的流动，必须在SHM中完成显式声明和轨迹记录。商场独立创建、独立命名、独立回收每一块共享内存。任何进程——无论是生产者、消费者、Worker还是AICoder代理——对SHM的访问都必须经过商场的矢量管理闸口。

此文档为商场SHM宪法之起源，任何后续实现不得违背。


## 2. 关键词定义


## 3. 架构定位

┌─────────────────────────────────────────────────────────────┐
│                        商场主进程                            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │  矢量管理器   │  │   闸口控制    │  │   状态机引擎     │  │
│  │  (函数集合)   │  │  (函数集合)   │  │   (函数集合)     │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │
│         └─────────────────┴─────────────────────┘           │
│                           │                                 │
│                    ┌──────▼──────┐                        │
│                    │   SHM宇宙    │  ← 商场独立管理          │
│                    │  (物理内存)   │                        │
│                    └──────┬──────┘                        │
└───────────────────────────┼─────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────▼────┐      ┌─────▼─────┐      ┌────▼────┐
    │ 生产者P1 │      │  消费者C1  │      │ Worker  │
    │ (键盘)   │      │ (Console) │      │ 探针   │
    └─────────┘      └───────────┘      └─────────┘

核心铁律： 1. 唯一通道原则：生产者与消费者之间不存在任何直接连接，所有数据交换必须经过SHM。 2. 商场主权原则：只有商场主进程有权调用SharedMemory(create=True)创建新的SHM块。生产者与消费者只能以create=False模式附加（attach）由商场预先创建的SHM。 3. 显式矢量原则：SHM中的每一段数据必须注册为”矢量”，未注册的数据视为宇宙尘埃，商场有权随时清除。 4. 禁止替代原则：严禁使用multiprocessing.Queue、Pipe、socket、mmap（非shared_memory模块）作为商场内部或商场与进程间的通讯手段。SHM是唯一的神。


## 4. SHM宪法（核心原则）

### 4.1 命名空间宪法

SHM名称采用三段式命名法：

mall_<domain>_<id>_<timestamp>

mall_：商场前缀，神圣不可更改

<domain>：域标识，vec（矢量）、hb（心跳）、bp（蓝图）、auth（授权）、bus（总线）

<id>：UUID短码（8字符），由商场状态机引擎生成

<timestamp>：创建时间戳，用于生命周期审计

示例： - mall_vec_p1_input_1715302800 —— 生产者P1的输入矢量 - mall_hb_p1_1715302800 —— 生产者P1的心跳槽 - mall_bp_aicoder_inbox_1715302800 —— AICoder蓝图收件箱 - mall_auth_p1_keyboard_1715302800 —— 键盘生产者授权书存储区

### 4.2 所有权宪法

每一块SHM在创建时即被赋予所有者（Owner）属性：

所有权变更必须由商场状态机引擎显式执行，任何进程不得私自转让所有权。


## 5. 矢量管理状态机

SHM中的每一个矢量必须处于以下五种状态之一。状态转换由商场状态机引擎驱动，函数级原子操作保证。

### 5.1 状态定义

[UNBORN] ──create──► [ACTIVE] ──seal──► [SEALED] ──archive──► [ARCHIVED] ──destroy──► [VOID]
                          │
                          └──error──► [CORRUPTED] ──destroy──► [VOID]

### 5.2 状态转换函数（函数级接口预览）

# 状态机转换函数 —— 由商场状态机引擎调用
def shm_vec_create(name: str, size: int, owner: str) -> int:
    """UNBORN -> ACTIVE：创建矢量并分配SHM"""
    pass

def shm_vec_seal(name: str, checksum: bytes) -> int:
    """ACTIVE -> SEALED：封印矢量，生成校验"""
    pass

def shm_vec_archive(name: str) -> int:
    """SEALED -> ARCHIVED：归档，供审计"""
    pass

def shm_vec_mark_corrupted(name: str, reason: str) -> int:
    """ACTIVE/SEALED -> CORRUPTED：标记损坏"""
    pass

def shm_vec_destroy(name: str) -> int:
    """任意状态 -> VOID：销毁并回收SHM"""
    pass


## 6. 函数级接口定义（SHM矢量管理器）

禁止任何形式的类封装。以下所有接口均为顶层函数，直接操作SHM宇宙。

### 6.1 宇宙管理（商场专属）

from multiprocessing import shared_memory
import struct
import hashlib
from typing import Dict, List, Tuple, Optional

# ─────────────────────────────────────────────
# 全局状态（商场主进程初始化时构建）
# ─────────────────────────────────────────────
SHM_REGISTRY: Dict[str, dict] = {}      # 矢量注册表：name -> metadata
SHM_NAME_INDEX: Dict[str, str] = {}     # 所有者索引：owner -> list of names

# ─────────────────────────────────────────────
# 1. SHM宇宙初始化
# ─────────────────────────────────────────────
def shm_universe_init() -> int:
    """
    商场启动时调用。
    初始化SHM命名空间、注册表、索引。
    返回：0 成功，-1 失败
    """
    pass

# ─────────────────────────────────────────────
# 2. 矢量创建（商场主权）
# ─────────────────────────────────────────────
def shm_vec_birth(
    domain: str,
    owner: str,
    size: int,
    dtype: str,           # 'bytes', 'int32', 'float64', 'json'
    label: str            # 人类可读标签
) -> Tuple[str, shared_memory.SharedMemory]:
    """
    诞生一个新的SHM矢量。
    自动生成符合命名空间宪法的三段式名称。
    返回：(full_name, shm_object)
    """
    pass

# ─────────────────────────────────────────────
# 3. 闸口校验（任何进程访问SHM前必须通过）
# ─────────────────────────────────────────────
def shm_gate_check(
    caller_id: str,
    vec_name: str,
    intent: str           # 'read', 'write', 'attach', 'detach'
) -> Tuple[bool, str]:
    """
    闸口控制函数。
    校验caller_id是否拥有对vec_name的intent权限。
    返回：(是否允许, 拒绝原因)
    """
    pass

# ─────────────────────────────────────────────
# 4. 矢量写入（带原子性保证）
# ─────────────────────────────────────────────
def shm_vec_write(
    caller_id: str,
    vec_name: str,
    data: bytes,
    offset: int = 0
) -> int:
    """
    向指定矢量写入数据。
    前置条件：通过shm_gate_check(caller_id, vec_name, 'write')
    返回：写入字节数，-1表示失败
    """
    pass

# ─────────────────────────────────────────────
# 5. 矢量读取
# ─────────────────────────────────────────────
def shm_vec_read(
    caller_id: str,
    vec_name: str,
    offset: int = 0,
    length: Optional[int] = None
) -> bytes:
    """
    从指定矢量读取数据。
    前置条件：通过shm_gate_check(caller_id, vec_name, 'read')
    返回：数据字节串
    """
    pass

# ─────────────────────────────────────────────
# 6. 封印与校验（生产者完成写入后调用）
# ─────────────────────────────────────────────
def shm_vec_seal(
    caller_id: str,
    vec_name: str
) -> bytes:
    """
    封印矢量：冻结写入，计算SHA-256校验和。
    状态转换：ACTIVE -> SEALED
    返回：校验和
    """
    pass

# ─────────────────────────────────────────────
# 7. 心跳槽管理
# ─────────────────────────────────────────────
def shm_heartbeat_beat(
    caller_id: str,
    hb_vec_name: str
) -> int:
    """
    进程心跳写入：写入当前时间戳。
    由生产者/消费者定期调用。
    """
    pass

def shm_heartbeat_check(
    hb_vec_name: str,
    timeout_sec: float = 5.0
) -> bool:
    """
    商场调用：检查指定心跳槽是否超时。
    返回：True存活，False死亡
    """
    pass

# ─────────────────────────────────────────────
# 8. 宇宙清理（商场主权）
# ─────────────────────────────────────────────
def shm_universe_sweep() -> Tuple[int, int]:
    """
    扫描所有SHM，回收CORRUPTED和VOID状态的矢量。
    返回：(回收数量, 剩余数量)
    """
    pass

def shm_universe_collapse() -> int:
    """
    商场关闭时调用：销毁所有SHM，清空注册表。
    返回：0 成功
    """
    pass


## 7. 内存布局规范

### 7.1 标准矢量布局

每一块SHM矢量（除纯原始数据区外）采用头-体-尾三段布局：

┌─────────────────────────────────────────────────────────────┐
│  头部（Header） 64字节                                       │
│  ├─ magic:      4字节  0x4D414C4C ("MALL")                  │
│  ├─ version:    2字节  布局版本                             │
│  ├─ state:      1字节  当前状态 (0=ACTIVE,1=SEALED,2=ARCHIVED,3=CORRUPTED) │
│  ├─ dtype:      1字节  数据类型编码                          │
│  ├─ owner_len:  2字节  所有者ID长度                          │
│  ├─ owner_id:   32字节 所有者ID（不足补零）                  │
│  ├─ data_len:   8字节  实际数据长度（大端）                  │
│  ├─ checksum:   32字节 SHA-256校验和（封印时写入）           │
│  └─ reserved:   8字节  保留                                 │
├─────────────────────────────────────────────────────────────┤
│  体部（Body） 可变长度                                        │
│  ├─ 实际业务数据                                              │
├─────────────────────────────────────────────────────────────┤
│  尾部（Tail） 16字节                                          │
│  ├─ magic_end:  4字节  0x4C4C414D ("LLAM")                  │
│  ├─ timestamp:  8字节  最后操作时间戳                         │
│  └─ reserved:   4字节                                       │
└─────────────────────────────────────────────────────────────┘

### 7.2 特殊SHM区域定义


## 8. 安全隔离铁律

### 8.1 进程隔离

地址空间隔离：生产者与消费者进程不得持有指向对方SHM的SharedMemory对象引用，除非通过shm_gate_check显式授权。

写隔离：消费者默认对任何SHM矢量处于只读状态。若需回写（如ACK确认），必须申请独立的ACK矢量，不得写回生产者的原始数据区。

命名空间隔离：进程只能看到与自己owner_id匹配的SHM名称列表，商场通过闸口函数过滤全局注册表。

### 8.2 内存安全

越界熔断：任何shm_vec_write/shm_vec_read操作必须检查offset + length <= header.data_len，越界立即返回错误并标记调用者CORRUPTED状态。

类型熔断：dtype字段必须严格匹配，尝试以错误类型解析数据将触发闸口拒绝。

封印不可违：SEALED状态的矢量任何写入尝试都将被闸口拒绝，返回错误码SHM_ERR_SEALED_VIOLATION。

### 8.3 心跳监控

每个生产者/消费者连接商场时，商场为其诞生一个专属hb矢量。

进程必须每N秒（默认3秒）调用shm_heartbeat_beat。

商场主循环每M秒（默认5秒）扫描所有hb矢量。

连续两次检测失败（即超时2*N），商场状态机自动将该进程所有关联矢量标记为CORRUPTED，并启动回收流程。


## 9. 生命周期管理

### 9.1 SHM创建流（商场启动）

shm_universe_init()
    ├── 创建 mall_bus_master (ACTIVE)
    ├── 创建 mall_registry_mirror (ACTIVE)
    ├── 创建 mall_bp_inbox (ACTIVE, owner=MALL, 权限：生产者写/AICoder读)
    ├── 创建 mall_auth_vault (ACTIVE, owner=MALL, 权限：AICoder写/商场实施读)
    └── 返回 0

### 9.2 生产者接入流

生产者P1请求接入
    ├── 商场：shm_vec_birth(domain='hb', owner='PRODUCER_P1', ...) → mall_hb_p1_xxx
    ├── 商场：shm_vec_birth(domain='vec', owner='PRODUCER_P1', label='input') → mall_vec_p1_input_xxx
    ├── 商场：注册到 SHM_REGISTRY 与 SHM_NAME_INDEX
    ├── 商场：更新 mall_registry_mirror
    └── 生产者P1：通过shm_gate_check获取attach权限，开始心跳与数据写入

### 9.3 蓝图提交流程（经SHM）

生产者P1完成蓝图
    ├── P1调用 shm_vec_write 写入 mall_bp_inbox
    ├── P1调用 shm_vec_seal 封印 inbox 中的蓝图数据
    ├── 商场状态机检测到 mall_bp_inbox 状态变为 SEALED
    ├── 商场：将蓝图转交外置AICoder（通过SHM或约定的外部通道，但商场内部全程使用SHM）
    ├── AICoder审核完成，生成授权书
    ├── AICoder调用 shm_vec_write 写入 mall_auth_vault
    ├── AICoder调用 shm_vec_seal 封印授权书
    └── 商场实施模块读取 mall_auth_vault，执行授权书指令


## 10. 与AICoder的协作边界

AICoder作为外置审核智能体，其与商场的交互点严格限定在SHM边界：

关键约束：AICoder不得直接创建SHM，不得直接操作生产者或消费者的私有矢量，不得绕过闸口。AICoder在商场宇宙中的身份是OWNER_AICODER，其权限由商场主进程在shm_universe_init()时显式注册。


## 11. 设计审核检查点


此蓝图一经审核通过，即成为商场SHM宇宙的创世法典。所有代码实现、测试场景、Worker行为必须在此法典框架内运行。任何对SHM的访问若未在此文档中显式定义，视为非法入侵，商场状态机有权直接标记CORRUPTED并启动强制回收。

审核状态：待人类架构师终审



| 关键词 | 定义 |
| --- | --- |
| SHM | multiprocessing.shared_memory.SharedMemory，商场唯一认可的跨进程通讯实体 |
| 矢量（Vector） | SHM中的一段逻辑数据单元，具有名称、类型、长度、状态、所有者五元组属性 |
| 矢量管理器（Vector Manager） | 商场内部的一个函数集合，负责SHM的创建、索引、分配、回收、状态监控 |
| SHM命名空间 | 商场全局维护的命名规则体系，确保所有SHM块名称唯一且可溯源 |
| 闸口（Gate） | 生产者/消费者访问SHM前必须通过的身份校验与权限检查函数 |
| 心跳槽（Heartbeat Slot） | SHM中预留的固定区域，用于进程存活检测 |
| 蓝图缓冲区（Blueprint Buffer） | 专用于接收生产者提交蓝图的SHM矢量区域 |
| 授权书缓冲区（Authorization Buffer） | 专用于存放AICoder审核通过后授权文件的SHM矢量区域 |
| 消息总线SHM | 商场内部进程间（如主进程与Worker进程）通讯的SHM实现，禁止使用Queue/Pipe替代 |


| 所有者类型 | 权限 | 生命周期 |
| --- | --- | --- |
| MALL | 完全控制（创建/读写/销毁） | 与商场主进程同生共死 |
| PRODUCER_<id> | 读写指定矢量区域 | 由商场状态机决定 |
| CONSUMER_<id> | 只读指定矢量区域（默认），写权限需特批 | 由商场状态机决定 |
| WORKER_<id> | 读写指定任务区域 | 任务完成即回收 |
| AICODER | 只读蓝图区 / 写授权书区 | 审核会话期间有效 |


| 状态 | 含义 | 允许操作 |
| --- | --- | --- |
| UNBORN | 已注册但未分配物理SHM | 仅商场：创建SHM |
| ACTIVE | 正常运行，可读写 | 商场：监控；所有者：读写 |
| SEALED | 数据写入完成，冻结只读 | 商场：校验；消费者：读取 |
| ARCHIVED | 数据已消费，保留审计 | 仅商场：读取/最终销毁 |
| CORRUPTED | 校验失败或心跳丢失 | 仅商场：诊断/销毁 |
| VOID | 物理内存已回收，名称注销 | 无 |


| 区域名称 | 用途 | 创建时机 | 大小 |
| --- | --- | --- | --- |
| mall_bus_master | 商场内部消息总线SHM | shm_universe_init() | 1MB |
| mall_registry_mirror | 注册表镜像（供Worker只读） | shm_universe_init() | 256KB |
| mall_bp_inbox | 蓝图提交收件箱（生产者写，AICoder读） | 商场启动后 | 512KB |
| mall_auth_vault | 授权书金库（AICoder写，商场实施读） | 商场启动后 | 512KB |


| 交互方向 | SHM区域 | AICoder权限 | 说明 |
| --- | --- | --- | --- |
| 商场 → AICoder | mall_bp_inbox | 只读 | AICoder读取待审核蓝图 |
| AICoder → 商场 | mall_auth_vault | 写+封印 | AICoder写入授权文件 |
| 商场 → AICoder | mall_hb_aicoder | 商场写/AICoder读 | 心跳与任务调度 |


| 检查项 | 状态 | 说明 |
| --- | --- | --- |
| 禁止OOP | ✅ 通过 | 全函数级接口，无类定义 |
| SHM唯一通道 | ✅ 通过 | 明确禁止Queue/Pipe/socket替代 |
| 商场主权 | ✅ 通过 | 仅商场可create=True |
| 矢量五元组 | ✅ 通过 | 名称/类型/长度/状态/所有者 |
| 状态机完备 | ✅ 通过 | 六状态+显式转换函数 |
| 安全隔离 | ✅ 通过 | 闸口/越界熔断/心跳监控 |
| AICoder边界 | ✅ 通过 | 限定双SHM区域交互 |

