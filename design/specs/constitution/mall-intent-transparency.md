# 商场模式总则：意图透明与SHM宪法

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **版本**：1.0  
> **日期**：2026-05-10  
> **性质**：商场生态根本大法，所有角色、流程、代码必须以本总则为最高约束。  
> **核心原则**：**一切函数围绕SHM运行；一切意图必须裸晒；一切私利必须绞杀。**

---

## 目录

1. [关键词定义](#1-关键词定义)
2. [四角色权力宪法](#2-四角色权力宪法)
3. [SHM内存宪法架构](#3-shm内存宪法架构)
4. [消费者意图订单机制](#4-消费者意图订单机制)
5. [生产者意图透明铁律](#5-生产者意图透明铁律)
6. [AICoder审计与授权流程](#6-aicoder审计与授权流程)
7. [商场主进程加载闸口](#7-商场主进程加载闸口)
8. [供需履约闭环](#8-供需履约闭环)
9. [私利检测清单与黑名单机制](#9-私利检测清单与黑名单机制)
10. [文件格式规范](#10-文件格式规范)

---

## 1. 关键词定义

| 术语 | 定义 |
|------|------|
| **SHM** | Shared Memory，商场唯一通讯通道与状态载体。禁止函数调用、消息队列、管道等其他IPC。 |
| **Intent Order** | 消费者意图订单，消费者向SHM投递的结构化需求声明，是生态进化的燃料。 |
| **Design Intent Manifest** | 生产者设计意图宣言，强制公开的调度策略与能力边界声明。 |
| **Function Cluster** | AICoder生成的离散函数集合（制造工具包），无自调度权，由生产者编排。 |
| **Authorization File** | 外置AICoder生成的授权文件，包含意图宣言、函数集群、审计报告，是商场加载的唯一凭证。 |
| **Mall Constitution** | SHM内存宪法，定义商场内所有频道的命名、权限、生命周期。 |
| **Producer Registry** | SHM中的生产者能力注册表，记录所有已加载生产者的身份、意图、频道。 |
| **Intent Queue** | SHM中的消费者意图订单池，商场主进程周期性扫描。 |
| **Channel Map** | SHM中的已建立频道路由表，记录生产者→消费者的单向数据绑定。 |
| **BlackList Segment** | SHM中的黑名单段，记录被驱逐的生产者身份哈希，永久禁止重新加载。 |

---

## 2. 四角色权力宪法

### 2.1 消费者（Consumer）——需求端
- **权力**：表达"我要什么"；向SHM投递Intent Order。
- **禁止**：直接调用任何函数；直接访问生产者进程；修改商场宪法。
- **产出物**：`Intent Order`（写入`shm://mall/intent_queue`）。

### 2.2 生产者（Producer）——设计端+调度端
- **权力**：决定产品规格；编排函数集群的调度策略；向商场注册能力。
- **强制义务**：必须提交**Design Intent Manifest**；必须公开调度逻辑；必须声明禁区。
- **禁止**：隐藏真实意图；在函数中嵌入未声明的SHM访问；自我繁殖；篡改宪法。
- **产出物**：`Design Intent Manifest` + `Function Cluster Blueprint`（由AICoder生成）。

### 2.3 AICoder（外置）——制造端+审计端
- **权力**：将生产者设计意图翻译为函数集群；对生产者进行意图-代码交叉审计；生成授权文件。
- **禁止**：拥有自调度权；拥有商场内进程加载权；拥有私利（禁止自利）。
- **产出物**：`Authorization File`（含代码、审计报告）。

### 2.4 商场（Mall）——平台端+编译端
- **权力**：维护SHM宪法；扫描Intent Queue；匹配供需；编译加载/拒绝生产者；建立频道绑定；维护黑名单。
- **禁止**：理解业务逻辑；替代AICoder设计代码；偏袒任何角色。
- **产出物**：运行中的进程；SHM频道；履约通知。

---

## 3. SHM内存宪法架构

商场内所有状态、通讯、意图、数据，最终归结为以下SHM频道：

```
shm://mall/
├── constitution/              # 商场宪法区（只读，仅商场主进程可写）
│   ├── version
│   ├── channel_schema
│   └── eviction_rules
├── intent_queue/              # 消费者意图订单池（消费者写，商场读）
│   └── <order_id>/
│       ├── manifest
│       └── timestamp
├── producer_registry/         # 生产者注册表（商场写，所有角色只读）
│   └── <producer_id>/
│       ├── design_intent      # 生产者意图宣言（强制公开）
│       ├── function_cluster   # 函数集群代码段（只读）
│       ├── audit_signature    # AICoder审计签名
│       └── status             # 加载状态：pending / active / evicted
├── consumer_registry/         # 消费者注册表
│   └── <consumer_id>/
│       ├── subscriptions      # 已订阅频道列表
│       └── fulfillment_inbox  # 商场履约通知信箱
├── channel_map/               # 已建立频道路由表
│   └── <channel_id>/
│       ├── producer_ref
│       ├── consumer_ref
│       ├── format_contract
│       └── vector_offset      # SHM矢量偏移地址
├── aicoder_workspace/         # AICoder设计蓝图的临时工作区（外置交互用）
│   └── <draft_id>/
├── blacklist/                 # 黑名单段（商场写，永久有效）
│   └── <producer_hash>/
│       ├── reason_code        # 驱逐原因
│       └── timestamp
└── system_log/                # 商场系统日志（只追加）
    └── <event_id>/
```

**铁律**：`multiprocessing.Queue`、`Pipe`、`Socket`等其他IPC机制被**绝对禁止**。SHM是商场唯一的通讯通道。

---

## 4. 消费者意图订单机制

### 4.1 意图三层表达

| 层级 | 名称 | 载体 | 示例 |
|------|------|------|------|
| L3 | 原始意图 | 自然语言 | "我想在屏幕上看到键盘输入" |
| L2 | 技术订单 | 结构化数据 | `{需求类型: 数据订阅, 目标能力: keyboard_input, 输出格式: utf8_stream}` |
| L1 | SHM操作 | 字节流 | 写入`shm://mall/intent_queue` |

### 4.2 Intent Order结构

```c
struct intent_order {
    char order_id[32];              // UUID
    char consumer_id[32];           // 消费者身份
    char intent_desc[256];          // L3: 自然语言描述
    char required_capability[64];   // L2: 所需能力标签
    char format_requirement[64];    // 格式要求
    uint32_t priority;              // 优先级
    uint8_t auto_evolve;            // 是否允许AICoder自动设计新生产者（0=否，1=是）
    uint64_t timestamp;             // 时间戳
};
```

### 4.3 消费者下单流程

1. 消费者将`intent_order`序列化为字节流；
2. 写入`shm://mall/intent_queue/<order_id>/manifest`；
3. 商场主进程周期性扫描`intent_queue`；
4. 商场解析订单，进入匹配状态机。

---

## 5. 生产者意图透明铁律

### 5.1 核心原则

> **生产者无权沉默。所有设计意图、调度策略、禁区声明必须强制公开。意图宣言与函数代码必须一一对应。**

### 5.2 Design Intent Manifest（强制格式）

```c
struct producer_intent_manifest {
    char identity[32];            // 生产者唯一标识
    char version[16];             // 宣言版本
    char input_contract[128];     // 输入承诺："只从 keyboard_raw 读取"
    char output_contract[128];    // 输出承诺："只向 console_display 写入"
    char forbidden_zones[256];    // 禁区声明："绝不读取 mall_admin、绝不写入 blacklist"
    char schedule_policy[64];     // 调度策略公开：轮询/事件驱动/批量
    char product_spec[256];       // 产品规格说明
    uint8_t auto_evolve_flag;     // 自我进化许可（默认0，禁止）
    uint64_t declaration_time;    // 宣言时间戳
    char aicoder_ref[32];         // 关联AICoder审计批次
};
```

### 5.3 强制提交规则

生产者向商场注册时，必须提交**两份不可分割的文件**：

| 文件 | 性质 | 缺失后果 |
|------|------|----------|
| **Design Intent Manifest** | 意图宣言 | **直接拒绝加载**，不进入AICoder流程 |
| **Function Cluster Blueprint** | 函数集群蓝图 | 无法通过审计，拒绝加载 |

---

## 6. AICoder审计与授权流程

### 6.1 AICoder核心职责

AICoder不是"代码生成器"，而是**意图-代码交叉审计官**。

### 6.2 审计五步法

```
Step 1: 解析 Design Intent Manifest → 提取"允许操作集合"(Allowed Set)
Step 2: 静态分析 Function Cluster → 提取"实际调用集合"(Actual Set)
Step 3: 比对：Actual Set ⊆ Allowed Set ?
        ├─ 是 → 通过
        └─ 否 → 标记"意图走私"(Intent Smuggling)，拒绝
Step 4: SHM访问矢量扫描：
        - 是否访问未声明的共享内存段？
        - 是否包含对 mall_constitution 的写操作？
        - 是否包含对 blacklist 的读操作？
Step 5: 输出审计报告与授权文件
```

### 6.3 授权文件结构

```
Authorization File（.mallauth）
├── Section 0: Intent Manifest（意图宣言）← 不可跳过
├── Section 1: Function Cluster（函数集群代码）
├── Section 2: AICoder Audit Report（审计报告）
│   ├── allowed_set_hash
│   ├── actual_set_hash
│   ├── cross_validation_result
│   └── audit_signature
└── Section 3: Mall Implementation Notes（商场实施备注，由商场填写）
```

---

## 7. 商场主进程加载闸口

商场`main()`在加载任何生产者前，执行**三重闸口**：

### 7.1 第一闸：意图宣言完整性校验
- 检查Section 0是否存在且格式合法；
- **缺失或格式非法 → 直接丢弃，不进入后续流程。**

### 7.2 第二闸：AICoder签名校验
- 验证AICoder审计签名；
- 验证意图宣言哈希与审计报告中的声明哈希一致；
- **不一致 → 标记"意图欺诈"，写入Blacklist Segment。**

### 7.3 第三闸：运行时行为监控
- 加载后，商场持续监控生产者SHM访问行为；
- 一旦发现运行时行为偏离声明，**立即终止进程并吊销授权**。

---

## 8. 供需履约闭环

### 8.1 两种供需场景

| 场景 | 名称 | 流程 |
|------|------|------|
| 场景A | 生产者先行 | 生产者提交蓝图 → 商场实施 → 消费者查询Registry后订阅 |
| 场景B | 消费者先行（进化场景） | 消费者提交Intent Order → 商场发现能力缺口 → 触发AICoder设计新生产者蓝图 → 生成授权文件 → 商场实施 → 建立频道 → 通知消费者 |

### 8.2 履约通知机制

商场对消费者的"通知"不是函数回调，而是**向消费者SHM信箱写入履约状态包**：

```c
struct fulfillment_notice {
    char order_id[32];        // 对应意图订单
    char channel_id[32];      // 已建立的SHM频道名
    uint8_t status;         // 0x01:已匹配 0x02:新建生产者后匹配 0xFF:无法满足
    char producer_id[32];     // 履约生产者身份
    uint64_t timestamp;
};
```

消费者通过**轮询自己的`fulfillment_inbox`**感知履约状态。

---

## 9. 私利检测清单与黑名单机制

### 9.1 私利检测清单

AICoder与商场必须扫描以下"生产者私利"模式：

| 私利类型 | 检测方法 | 后果 |
|----------|----------|------|
| **数据窃取** | 函数是否读取非`input_contract`声明的SHM频道 | 拒绝授权，写入黑名单 |
| **优先级操纵** | 调度逻辑是否暗中提升某消费者数据权重 | 拒绝授权，写入黑名单 |
| **自我繁殖** | 是否包含生成新进程或复制自身代码的逻辑 | 拒绝授权，写入黑名单 |
| **宪法篡改** | 是否尝试写入`mall_constitution`或`intent_queue`管理区 | 拒绝授权，写入黑名单 |
| **后门预留** | 是否包含条件触发的不明SHM写操作（如特定时间激活） | 拒绝授权，写入黑名单 |
| **意图走私** | 实际调用集合超出允许操作集合 | 拒绝授权，写入黑名单 |

### 9.2 黑名单机制

- 被驱逐的生产者身份哈希写入`shm://mall/blacklist/`；
- 商场`main()`在加载任何生产者前，先查询黑名单；
- **黑名单命中 → 永久拒绝，不启动任何审计流程。**

---

## 10. 文件格式规范

### 10.1 设计原则

- **禁止OOP**：所有代码为函数级设计，无类、无对象、无继承。
- **SHM中心**：所有状态流转必须通过SHM矢量完成。
- **显式优于隐式**：意图、调度、禁区必须明文声明，禁止约定俗成。

### 10.2 授权文件命名规范

```
<producer_id>_<version>_<aicoder_batch>.mallauth

示例：keyboard_producer_v1_0_aicoder_20260510_001.mallauth
```

### 10.3 版本控制

- 本总则为**商场根本法**，版本号由商场主进程在`shm://mall/constitution/version`维护；
- 任何对总则的修正必须通过**创世Worker**提交修正案，经人工审核后由商场主进程更新。

---

## 附录：一句话宪法

> **消费者用意图订单点燃需求，AICoder用审计与代码实现需求，生产者用设计意图与调度策略制造产品，商场用SHM宪法与编译闸口履约需求——四方通过共享内存咬合，无函数调用，无对象握手，无隐秘意图。**

---

*本文件由AICoder（外置）根据商场模式讨论生成，供创世Worker审核。*
*审核状态：待审核*
