# 任务流程操作总纲领

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

**版本**：AICoder-TaskFlow-1.1  
**编纂日期**：2026-05-10  
**适用范围**：生产者-商场-消费者全栈体系  
**核心原则**：函数级原子操作 · 人类审核权至上 · 商场编译进化权 · SHM矢量宪法 · 双向契约批准

---

## 目录

- [卷一：任务操作的技术手段](#卷一任务操作的技术手段)
  - [一、纲领宗旨与哲学根基](#一纲领宗旨与哲学根基)
  - [二、核心概念定义](#二核心概念定义)
  - [三、任务体系架构](#三任务体系架构)
  - [四、任务操作的技术手段](#四任务操作的技术手段)
  - [五、任务群编排规范](#五任务群编排规范)
  - [六、流程生命周期详解](#六流程生命周期详解)
  - [七、安全铁律与隔离机制](#七安全铁律与隔离机制)
  - [八、测试验证体系](#八测试验证体系)
  - [九、附录：全局标志速查表](#九附录全局标志速查表)
- [增补卷二：商场-消费者契约匹配与代价评估机制](#增补卷二商场-消费者契约匹配与代价评估机制)
  - [一、机制宗旨](#一机制宗旨从单向提交到双向契约)
  - [二、核心概念定义（增补）](#二核心概念定义增补)
  - [三、三层架构中的契约匹配拓扑](#三三层架构中的契约匹配拓扑)
  - [四、技术手段详解：匹配-评估-批准全链路](#四技术手段详解匹配-评估-批准全链路)
  - [五、状态机扩展：契约态融入任务生命周期](#五状态机扩展契约态融入任务生命周期)
  - [六、安全与审计](#六安全与审计)
  - [七、测试验证体系（增补）](#七测试验证体系增补)
  - [八、审核通过标准（增补）](#八审核通过标准增补)
  - [九、结语：契约即秩序](#九结语契约即秩序)

---

# 卷一：任务操作的技术手段

## 一、纲领宗旨与哲学根基

本纲领是**任务宇宙**的宪法性文件。一切任务与任务群的诞生、演化、消亡，必须在此纲领划定的技术轨道内运行。

> **第一性原理**：任务不是代码的附属品，而是智能体生态的**最小可调度原子**。对任务的操作手段，直接决定整个AICoder文明的运行效率与伦理边界。

我们拒绝以传统操作系统的进程/线程模型来理解任务。在商场体系中，任务是**携带设计意图的矢量信息包**，其操作技术手段必须满足：
- **物理隔离性**：任务之间不可通过隐式共享状态耦合
- **可审计性**：每一个状态跃迁必须留下不可篡改的追踪链
- **可编译进化性**：商场有权对任务进行函数级重组与优化
- **人类终审性**：任何跨任务群的编排决策，必须经由人类架构师审核

---

## 二、核心概念定义

| 术语 | 定义 | 技术形态 |
|---|---|---|
| **任务（Task）** | 最小可执行设计单元，携带输入矢量、操作函数、输出契约 | 函数级原子包，禁止OOP封装 |
| **任务群（TaskGroup）** | 具有拓扑依赖关系的任务集合，构成一次完整的业务闭环 | DAG（有向无环图）结构 |
| **全局标志（Global Flag）** | 跨生产者、商场、消费者三层唯一识别的身份指纹 | 128位复合标识符 |
| **SHM矢量** | 任务间数据交换的唯一合法载体，基于共享内存段 | 定长矢量块，带版本戳 |
| **算力进程池** | 任务执行的物理承载单元，动态评估负载后分配 | 无状态Worker进程组 |
| **编译进化权** | 商场对任务函数进行重组、内联、SIMD优化的专有权力 | 行为级模型转换器 |

---

## 三、任务体系架构

### 3.1 三层拓扑

```
┌─────────────────────────────────────────┐
│           消费者层 (Consumer)            │  ← 任务发起与结果消费
│         人类架构师审核终端                │
├─────────────────────────────────────────┤
│           商场层 (Market)                │  ← 任务编译进化与全局调度
│    算力评估 · 依赖解析 · SHM宪法仲裁      │
├─────────────────────────────────────────┤
│           生产者层 (Producer)            │  ← 任务执行与原始输出
│      Worker探针 · 函数级原子引擎          │
└─────────────────────────────────────────┘
```

### 3.2 任务与任务群的关系律

- **单任务不可跨越两层**：一个任务实例在同一时刻只能处于生产者执行态或商场调度态
- **任务群是商场的管理边界**：商场仅对任务群进行拓扑优化，不对单个任务内部逻辑做语义干预
- **消费者只订阅任务群结果**：消费者通过任务群全局标志获取最终矢量输出，不可直接监听单任务内部SHM

---

## 四、任务操作的技术手段（核心章节）

本章节定义对任务进行**全生命周期操作**的精确技术手段。任何实现层（Python/C/Verilog/JS）必须提供以下手段的原子映射。

### 4.1 全局唯一标识系统（GUID）

**技术手段**：128位复合标志，结构如下：

```
[31:0]   时间戳域（秒级，自Epoch起）
[63:32]  随机熵域（硬件噪声源或CSPRNG）
[79:64]  任务类型编码（16位枚举）
[95:80]  优先级权重（16位，0000为最高）
[127:96] CRC32校验域（覆盖前96位）
```

**操作接口**：
- `task_guid_generate(type_enum, priority)` → 返回原始128位标志
- `task_guid_validate(guid)` → 校验CRC与类型合法性
- `task_guid_extract_priority(guid)` → 提取调度优先级

**任务群标志扩展**：
在任务GUID前附加**群前缀**（32位），格式为 `GROUP_{PREFIX}_{MASTER_GUID}`。同一任务群内所有任务共享群前缀，但保持独立GUID。

### 4.2 状态机驱动引擎

**技术手段**：原子状态跃迁，禁止中间态暴露。

```
状态枚举（严格顺序）：
CREATED(0) → SUBMITTED(1) → AUDITING(2) → COMPILED(3) → SCHEDULED(4) → RUNNING(5) → COMPLETED(6) / FAILED(7) / CANCELLED(8)

跃迁规则：
- CREATED → SUBMITTED：消费者提交触发
- SUBMITTED → AUDITING：商场接收，进入人类审核队列
- AUDITING → COMPILED：人类架构师审核通过（关键控制点）
- COMPILED → SCHEDULED：商场完成函数编译进化
- SCHEDULED → RUNNING：算力进程池分配Worker
- RUNNING → {COMPLETED, FAILED}：生产者返回终态矢量
- 任意态 → CANCELLED：人类架构师或熔断机制强制终止
```

**操作接口**：
- `task_state_transition(guid, target_state, auth_token)` → 原子CAS操作，失败返回当前态
- `task_state_query(guid)` → 返回当前状态码与跃迁时间戳
- `task_state_subscribe(group_guid, callback)` → 异步状态变更推送（仅消费者层可用）

### 4.3 SHM矢量内存管理

**技术手段**：共享内存段的分块矢量分配，替代传统堆/栈/消息队列。

**SHM块结构**：
```
[0:15]      块标志（BlockFlag：关联的Task GUID前32位）
[16:19]     矢量维度（N，最大65535）
[20:23]     元素字节宽（1/2/4/8）
[24:31]     版本戳（单调递增，用于CAS乐观锁）
[32:39]     有效载荷长度（字节）
[40:...]    原始数据区（定长预分配，禁止动态扩展）
[...:末尾]  校验和（Adler32）
```

**操作接口**：
- `shm_vector_alloc(guid, dim, width)` → 分配SHM块，返回段ID
- `shm_vector_write(seg_id, offset, data, expected_version)` → CAS写，版本不匹配则失败
- `shm_vector_read(seg_id, offset, len)` → 只读映射，不触发拷贝
- `shm_vector_reclaim(seg_id)` → 引用计数归零后回收，归零判定由商场仲裁器执行

**宪法铁律**：
- 任务间数据交换**必须**通过SHM矢量，禁止socket/管道/全局变量等隐式通道
- SHM块生命周期**严格绑定**任务GUID，任务进入`COMPLETED`或`FAILED`态后，商场启动延迟回收（默认300秒冷却期，供消费者异步读取）

### 4.4 依赖拓扑解析器（DAG引擎）

**技术手段**：任务群内部的有向无环图构建与拓扑排序。

**边定义**：
```
edge := {
    upstream:   Task GUID,
    downstream: Task GUID,
    shm_link:   SHM段ID,      // 上游输出矢量映射为下游输入
    trigger:    {ALL, ANY, COUNT(n)},  // 下游触发条件
    timeout_ms: 非负整数       // 边级超时，覆盖群默认超时
}
```

**操作接口**：
- `dag_build(group_guid, edge_list)` → 合法性校验（环检测、SHM可达性验证）
- `dag_topological_sort(group_guid)` → 返回可并行执行的层级序列
- `dag_critical_path(group_guid)` → 返回最长路径任务序列，用于算力预分配
- `dag_edge_status(group_guid, edge_id)` → 返回`PENDING / FIRED / EXPIRED`

### 4.5 动态算力调度器

**技术手段**：基于Worker探针实时反馈的进程池调度。

**调度因子向量**：
```
score = w1·(1/cpu_load) + w2·(memory_avail/shm_demand) + w3·(network_latency^-1) + w4·(historical_success_rate)
```

**操作接口**：
- `scheduler_register_worker(worker_guid, capability_vector)` → Worker上线注册
- `scheduler_evaluate_task(task_guid)` → 返回最优Worker候选列表
- `scheduler_dispatch(task_guid, worker_guid, preemptive_flag)` → 下发执行，若`preemptive_flag`为真，允许抢占低优先级任务
- `scheduler_rebalance(group_guid)` → 任务群执行中动态迁移（仅允许对`SCHEDULED`态未启动任务）

### 4.6 编译进化接口（商场专有）

**技术手段**：行为级模型到目标代码的函数级转换与优化。

**进化操作集**：
- `evo_inline(task_guid, target_function)` → 函数内联，消除调用开销
- `evo_simd(task_guid, vector_width)` → SIMD矢量化（128/256/512位）
- `evo_fuse(task_guid_list)` → 多任务函数融合，减少SHM中间态
- `evo_strip(task_guid)` → 死代码消除与常量折叠

**操作接口**：
- `evo_compile(source_design_doc, target_lang)` → 输出编译后函数包
- `evo_verify(compiled_package, test_vector_set)` → 回归测试，行为一致性校验
- `evo_deploy(compiled_package, group_guid)` → 原子替换，旧版本进入`DEPRECATED`态

**权力边界**：商场拥有编译进化权，但**不可**修改任务的设计契约（输入/输出矢量维度与语义）。契约变更必须通过人类架构师审核。

### 4.7 熔断、重试与降级

**技术手段**：任务群级故障隔离与自愈。

**熔断规则**：
- 单任务连续失败次数 ≥ 3，触发`TASK_CIRCUIT_OPEN`，该任务GUID进入黑名单
- 任务群内失败任务占比 ≥ 30%，触发`GROUP_CIRCUIT_OPEN`，整个群进入`FAILED`态
- 熔断开启后，新提交任务直接返回`FAILED`，不再调度

**重试策略**：
- `IMMEDIATE`：失败后立即重试（适用于瞬态网络错误）
- `EXPONENTIAL_BACKOFF`：退避间隔 = 2^n · base_ms（适用于资源争用）
- `REDIRECT`：更换Worker重试（适用于算力节点故障）

**降级策略**：
- `SHM_FALLBACK`：输出空矢量（维度正确，元素为零），保证下游任务可继续执行
- `PARTIAL_RESULT`：输出已计算的部分矢量，标记`is_partial`标志位

**操作接口**：
- `fault_inject_retry(task_guid, strategy)` → 注册重试策略
- `fault_circuit_status(guid)` → 查询熔断器状态
- `fault_degrade(task_guid, mode)` → 手动触发降级（人类架构师权限）

### 4.8 审计追踪链（Audit Chain）

**技术手段**：任务全生命周期事件的不可篡改日志。

**事件结构**：
```
event := {
    timestamp_ns: 64位纳秒时间戳,
    guid:         Task/Group GUID,
    actor:        {HUMAN, MARKET, WORKER, SYSTEM},
    action:       状态跃迁或操作名称,
    pre_hash:     上一事件哈希（SHA256前32位）,
    payload_hash: 事件载荷哈希,
    signature:    actor私钥签名（人类与商场使用非对称密钥）
}
```

**操作接口**：
- `audit_append(guid, event)` → 追加事件，自动链接哈希
- `audit_verify(guid, from_timestamp)` → 校验链完整性
- `audit_export(group_guid, format)` → 导出任务群完整审计报告

---

## 五、任务群编排规范

### 5.1 任务群全局标志格式

```
标准格式：GROUP_{PREFIX}_{STRATEGY}_{MASTER_GUID}

PREFIX（8位十六进制）：群类型标识
  - 0x01：顺序流水线（Sequential Pipeline）
  - 0x02：并行广播（Parallel Broadcast）
  - 0x03：竞争优选（Race Winner）
  - 0x04：冗余表决（Redundant Vote）
  - 0x05：动态图（Dynamic DAG）

STRATEGY（8位十六进制）：调度策略
  - 0xA0：严格拓扑（不允许抢占）
  - 0xA1：优先级抢占（高优先级可中断低优先级）
  - 0xA2：弹性伸缩（根据负载动态增减Worker）

MASTER_GUID：群首任务的128位GUID
```

### 5.2 任务群生命周期

1. **创世（Genesis）**：消费者提交群设计文档，商场生成`MASTER_GUID`
2. **拓扑固化（Solidification）**：`dag_build`成功，群标志写入全局注册表
3. **审核门（Audit Gate）**：人类架构师确认拓扑与SHM分配方案
4. **编译波（Compile Wave）**：商场对群内全部任务执行`evo_compile`
5. **执行域（Execution Domain）**：算力进程池按拓扑层级调度
6. **终结算（Finalization）**：所有任务达`COMPLETED`或`FAILED`，审计链归档
7. **资源释（Release）**：SHM冷却期结束，段回收，群标志注销

---

## 六、流程生命周期详解

### 6.1 单任务操作流程

```
消费者层
    │
    ▼
[1] 设计文档提交 ──→ task_guid_generate(TYPE, PRIORITY)
    │
    ▼
商场层
    │
    ▼
[2] 注册与审核 ──→ task_state_transition(guid, SUBMITTED, CONSUMER_TOKEN)
    │                task_state_transition(guid, AUDITING, SYSTEM_TOKEN)
    ▼
[3] 人类审核 ──→ 人工确认设计契约（输入/输出矢量维度、SHM需求）
    │              通过：task_state_transition(guid, COMPILED, HUMAN_SIG)
    │              拒绝：task_state_transition(guid, CANCELLED, HUMAN_SIG)
    ▼
[4] 编译进化 ──→ evo_compile(design_doc, target_lang)
    │              evo_verify(compiled_pkg, test_vectors)
    ▼
[5] 调度分配 ──→ scheduler_evaluate_task(guid) → worker_guid
    │              shm_vector_alloc(guid, dim, width)
    │              task_state_transition(guid, SCHEDULED, MARKET_TOKEN)
    ▼
生产者层
    │
    ▼
[6] 执行 ──→ task_state_transition(guid, RUNNING, WORKER_TOKEN)
    │          函数级原子引擎执行，结果写入SHM矢量
    ▼
[7] 终态 ──→ task_state_transition(guid, COMPLETED/FAILED, WORKER_TOKEN)
    │          shm_vector_reclaim 延迟启动
    ▼
商场层
    │
    ▼
[8] 审计归档 ──→ audit_append(guid, FINAL_EVENT)
    │              消费者回调触发
    ▼
消费者层
    │
    ▼
[9] 结果消费 ──→ shm_vector_read(seg_id, 0, len) → 获取输出矢量
```

### 6.2 任务群操作流程

在单任务流程基础上，增加：

- **步骤2'**：`dag_build(group_guid, edges)` 校验拓扑
- **步骤4'**：按拓扑层级分批执行`evo_compile`，识别融合机会
- **步骤5'**：`dag_topological_sort`生成分层调度计划，`scheduler_rebalance`动态优化
- **步骤7'**：`dag_edge_status`监控边触发，失败任务按`fault_inject_retry`策略处理
- **步骤8'**：群级`audit_export`生成完整报告

---

## 七、安全铁律与隔离机制

### 7.1 权限矩阵

| 操作 | 人类架构师 | 商场系统 | Worker探针 | 消费者 |
|---|---|---|---|---|
| 任务创建 | ✓ | ✗ | ✗ | ✓ |
| 审核通过 | ✓ | ✗ | ✗ | ✗ |
| 编译进化 | ✗ | ✓ | ✗ | ✗ |
| 状态跃迁（RUNNING） | ✗ | ✗ | ✓ | ✗ |
| SHM读写 | ✗ | ✓（仲裁） | ✓（限定段） | ✓（只读输出段） |
| 熔断操作 | ✓ | ✓ | ✗ | ✗ |
| 审计查询 | ✓ | ✓ | ✗ | ✓（仅限自身任务） |

### 7.2 隔离铁律

1. **内存隔离**：Worker进程的地址空间与SHM段通过`mprotect`或等价机制隔离，越界访问触发`SIGSEGV`并由商场捕获为`FAILED`
2. **时间隔离**：单任务执行设置硬超时（由`timeout_ms`定义），超时强制`SIGKILL`并标记`FAILED`
3. **编译隔离**：`evo_compile`在沙箱进程中进行，编译产物需通过行为级等价验证后方可部署
4. **网络隔离**：Worker探针仅允许与商场层通信，禁止直接访问消费者层或外部网络

---

## 八、测试验证体系

**技术手段是设计的检验标准，测试是技术手段的终审法庭。**

### 8.1 任务级测试

- **单元矢量测试**：输入预定义测试矢量，校验输出矢量的数值精度与维度
- **边界SHM测试**：模拟SHM段满、版本冲突、延迟回收场景
- **状态跃迁测试**：强制注入非法跃迁，验证状态机拒绝与回滚
- **编译等价测试**：对比进化前后函数包对同一测试矢量的输出差异（允许浮点误差<1e-6）

### 8.2 任务群级测试

- **拓扑环检测测试**：提交含环边的任务群，验证`dag_build`拒绝
- **并发压力测试**：1000+任务群同时提交，验证算力调度器与SHM仲裁器无死锁
- **故障注入测试**：随机`KILL` Worker进程，验证熔断、重试、降级链路的自愈能力
- **审计链完整性测试**：篡改中间事件哈希，验证`audit_verify`检测并告警

### 8.3 审核通过标准

任何技术手段的实现代码，必须通过以下审核方可进入商场体系：
1. **无OOP设计**：纯函数级实现，无类、无继承、无多态
2. **SHM宪法合规**：所有跨任务数据交换显式使用SHM矢量
3. **状态机完备**：覆盖全部枚举状态与跃迁路径
4. **测试覆盖率**：分支覆盖率≥95%，关键路径（审核门、编译接口、熔断器）100%

---

## 九、附录：全局标志速查表

### 9.1 任务类型编码（16位）

| 编码 | 类型 | 说明 |
|---|---|---|
| 0x0001 | COMPUTE | 纯计算型（信号处理、数值分析） |
| 0x0002 | IO | 仪器仪表交互型（VNA、频谱仪、示波器） |
| 0x0003 | NETWORK | 网络通信型（RDMA/SHM跨节点） |
| 0x0004 | UI | 界面渲染型（任务编辑控制器、数据可视化） |
| 0x0005 | DOC | 文档生成型（设计文档、代码、规则输出） |
| 0x0006 | EVO | 编译进化型（商场内部专用） |
| 0x0007 | TEST | 测试验证型（探针、回归、压力） |

### 9.2 优先级权重（16位）

| 权重 | 级别 | 调度策略 |
|---|---|---|
| 0x0000 | CRITICAL | 独占算力，可抢占所有非CRITICAL任务 |
| 0x0001-0x00FF | HIGH | 优先分配，允许抢占LOW任务 |
| 0x0100-0x7FFF | NORMAL | 默认权重，按拓扑顺序调度 |
| 0x8000-0xFEFF | LOW | 空闲算力填充，可被任意抢占 |
| 0xFFFF | BACKGROUND | 仅当算力进程池全空闲时执行 |

### 9.3 状态编码（8位）

| 编码 | 状态 | 可跃迁目标 |
|---|---|---|
| 0x00 | CREATED | SUBMITTED, CANCELLED |
| 0x01 | SUBMITTED | AUDITING, CANCELLED |
| 0x02 | AUDITING | COMPILED, CANCELLED |
| 0x03 | COMPILED | SCHEDULED, CANCELLED |
| 0x04 | SCHEDULED | RUNNING, CANCELLED |
| 0x05 | RUNNING | COMPLETED, FAILED, CANCELLED |
| 0x06 | COMPLETED | （终态） |
| 0x07 | FAILED | （终态，可人工重置为CREATED） |
| 0x08 | CANCELLED | （终态，可人工重置为CREATED） |

---

**本纲领自发布之日起生效。任何技术手段的实现，若与本纲领冲突，以本纲领为准。人类架构师保留对本纲领的最终解释权与修订权。**

> *任务即原子，流程即律法，SHM即血液，审核即灵魂。*  
> *商场编译进化，Worker忠实执行，消费者安心设计——此三者，共筑AICoder之基石。*

---

---

# 增补卷二：商场-消费者契约匹配与代价评估机制

**版本**：AICoder-TaskFlow-1.1  
**增补日期**：2026-05-10  
**核心新增**：需求匹配引擎 · 代价评估模型 · 双向批准契约

---

## 一、机制宗旨：从"单向提交"到"双向契约"

在先前的任务宇宙宪法中，任务由消费者单向提交，经人类审核后进入商场编译。这一模型在微型商场创世初期足以运转，但随着算力民主化与任务复杂度指数级增长，**单向提交流**已暴露出根本性缺陷：

> **缺陷一：需求模糊性**——消费者提交的设计意图与商场可编译的函数级契约之间存在语义鸿沟  
> **缺陷二：资源盲目性**——商场在未评估算力代价的情况下接受任务，导致调度器过载、Worker探针崩溃、SHM内存宪法被击穿  
> **缺陷三：权责不对等**——消费者享有发起权却无需承担资源承诺，商场拥有编译权却无法拒绝不合理需求

**本增补卷宣告任务流程进入"契约时代"**：

**商场不再是被动接收任务的容器，而是主动评估、匹配、定价、谈判的算力市场中枢。消费者不再是任务的绝对主人，而是与商场平等缔约的契约方。双方共同批准，方可令任务获得合法出生权。**

---

## 二、核心概念定义（增补）

| 术语 | 定义 | 技术形态 |
|---|---|---|
| **需求矢量（Demand Vector）** | 消费者对任务能力的结构化描述，包含功能语义、性能期望、资源上限 | N维浮点数组，带约束边界 |
| **供给矩阵（Supply Matrix）** | 商场当前可调度的全量算力资源快照，包含Worker负载、SHM余量、编译队列深度 | 稀疏矩阵，实时刷新周期≤100ms |
| **代价评估模型（Cost Model）** | 将需求矢量映射为可量化资源消耗的函数，输出算力代价、时间代价、SHM代价 | 三输出回归模型，支持在线学习 |
| **匹配度指数（Match Score）** | 需求矢量与供给矩阵的相容性度量，0为完全不可行，1为完美匹配 | 归一化标量，保留6位小数 |
| **契约草案（Contract Draft）** | 经匹配与评估后生成的双向约束文件，含任务规格、资源配额、时间窗口、违约条款 | 结构化文档，哈希签名锁定 |
| **双向批准（Dual Approval）** | 消费者以数字签名确认需求接受，商场以系统签名确认供给承诺，二者缺一不可 | 非对称密钥双签机制 |
| **契约锁（Contract Lock）** | 批准后的资源预占状态，SHM段与算力进程池进入保留态，直至任务终态或超时释放 | 带TTL的原子锁 |

---

## 三、三层架构中的契约匹配拓扑

```
┌──────────────────────────────────────────────────────────────┐
│                     消费者层 (Consumer)                        │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│  │  需求意图    │ ──→ │ 需求解构引擎 │ ──→ │ 需求矢量生成 │   │
│  │  (自然语言/  │      │ (语义规范化) │      │ (结构化输出) │   │
│  │   设计草图)  │      └─────────────┘      └──────┬──────┘   │
│  └─────────────┘                                    │          │
│                                                     ▼          │
│                                              ┌─────────────┐   │
│                                              │ 代价评估阅读 │   │
│                                              │  消费者确认  │   │
│                                              │  (数字签名)  │   │
│                                              └──────┬──────┘   │
└─────────────────────────────────────────────────────┼──────────┘
                                                      │
                                                      ▼ 契约草案上行
┌─────────────────────────────────────────────────────┼──────────┐
│                     商场层 (Market)                   │          │
│                                                     ▼          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│  │  供给矩阵    │ ←── │ 资源感知探针 │ ←── │ 算力进程池   │   │
│  │  (实时快照)  │      │ (Worker/SHM) │      │  全量Worker  │   │
│  └──────┬──────┘      └─────────────┘      └─────────────┘   │
│         │                                           ▲          │
│         ▼                                           │          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐   │
│  │  匹配引擎    │ ←── │ 代价评估模型 │ ←── │  契约生成器   │   │
│  │ (相容性计算)│      │ (三代价输出) │      │ (约束锁定)  │   │
│  └──────┬──────┘      └─────────────┘      └──────┬──────┘   │
│         │                                          │          │
│         └──────────────────┬───────────────────────┘          │
│                            ▼                                   │
│                     ┌─────────────┐                            │
│                     │  双向批准门   │                            │
│                     │ 商场系统签名  │                            │
│                     │  契约生效    │                            │
│                     └──────┬──────┘                            │
│                            ▼ 契约锁下行                         │
└────────────────────────────┼───────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────┼───────────────────────────────────┐
│                     生产者层 (Producer)                          │
│                            ▼                                   │
│                     ┌─────────────┐                            │
│                     │  契约执行器   │                            │
│                     │ 资源预占确认  │                            │
│                     │  任务原子执行  │                            │
│                     └─────────────┘                            │
└────────────────────────────────────────────────────────────────┘
```

---

## 四、技术手段详解：匹配-评估-批准全链路

### 4.1 需求解构引擎（消费者层）

**技术手段**：将消费者的非结构化意图转化为机器可解析的需求矢量。

**输入域**：
- 设计文档（函数级规格、输入/输出契约）
- 性能期望（延迟上限、吞吐量下限、精度要求）
- 资源上限（最大SHM占用、最大Worker占用、最大执行时长）
- 优先级声明（CRITICAL / HIGH / NORMAL / LOW / BACKGROUND）

**解构过程**：

```
需求意图
    │
    ▼
[语义规范化] ──→ 提取关键约束词（如"实时"、"批处理"、"高精度"）
    │
    ▼
[功能映射] ──→ 匹配任务类型编码（COMPUTE / IO / NETWORK / UI / DOC / EVO / TEST）
    │
    ▼
[性能量化] ──→ 将"低延迟"转化为 `max_latency_ms ≤ 50`
    │            将"高吞吐"转化为 `min_throughput_qps ≥ 1000`
    ▼
[资源边界] ──→ 设定 `max_shm_bytes`、`max_worker_count`、`max_exec_time_ms`
    │
    ▼
[需求矢量生成] ──→ 输出标准化N维数组
```

**需求矢量结构（Demand Vector）**：
```
[0]      任务类型编码（float32映射）
[1]      优先级权重（归一化到[0,1]）
[2]      期望延迟（ms，倒数归一化）
[3]      期望吞吐（QPS，对数归一化）
[4]      精度要求（有效位数/最大误差）
[5]      SHM预算（MB）
[6]      Worker预算（个数）
[7]      执行时长预算（秒）
[8]      容错等级（0=严格一致，1=允许降级，2=允许失败）
[9..N-1] 扩展语义槽（预留）
```

**操作接口**：
- `demand_deconstruct(intent_doc)` → 返回需求矢量与置信度分数
- `demand_validate(demand_vector)` → 校验边界合理性（如SHM预算不可超过系统上限）
- `demand_diff(demand_v1, demand_v2)` → 计算两个需求矢量的语义距离

### 4.2 供给矩阵实时感知（商场层）

**技术手段**：商场通过分布式探针聚合全量算力资源状态。

**供给矩阵结构（Supply Matrix）**：

```
行：资源维度（与需求矢量同维度）
列：Worker节点（每个活跃Worker一列）

核心行定义：
Row 0:  任务类型支持掩码（该Worker支持哪些类型）
Row 1:  当前优先级承载能力（剩余权重容量）
Row 2:  可用延迟能力（当前负载下的预估响应延迟）
Row 3:  可用吞吐能力（当前负载下的预估处理吞吐）
Row 4:  精度能力（硬件支持的浮点精度等级）
Row 5:  可用SHM容量（MB）
Row 6:  可用Worker槽位（该节点可并行任务数）
Row 7:  可用执行时长窗口（该节点最长可连续执行时间）
Row 8:  容错支持等级（该节点是否支持降级执行）
Row 9:  健康评分（历史成功率、负载稳定性综合）
```

**刷新机制**：
- **心跳周期**：每个Worker探针每100ms上报一次状态增量
- **全量同步**：商场每5秒执行一次全量矩阵重建
- **突变触发**：Worker负载变化>20%或SHM变化>10%时，立即推送增量更新

**操作接口**：
- `supply_snapshot()` → 返回当前供给矩阵副本
- `supply_delta(worker_guid, delta_vector)` → 应用增量更新
- `supply_forecast(horizon_ms)` → 基于趋势预测未来`horizon_ms`时刻的供给矩阵

### 4.3 匹配引擎（相容性计算）

**技术手段**：需求矢量与供给矩阵的逐列相容性计算，输出匹配度指数。

**匹配度计算公式**：

对于每一列Worker `j`，计算：

```
match_score_j = Σ_{i=0}^{N-1} w_i · compatibility(demand[i], supply[i][j])

其中 compatibility 函数：
- 若 supply[i][j] == 0 且 demand[i] > 0 → 0（完全不兼容）
- 若 supply[i][j] ≥ demand[i] → 1（完全满足）
- 若 0 < supply[i][j] < demand[i] → supply[i][j] / demand[i]（部分满足）

权重 w_i 由任务类型编码查表确定：
- COMPUTE型：延迟权重0.3，吞吐权重0.3，SHM权重0.2，Worker权重0.1，时长权重0.1
- IO型：延迟权重0.4，精度权重0.2，SHM权重0.2，健康评分权重0.2
- UI型：延迟权重0.5，Worker权重0.2，SHM权重0.2，容错权重0.1
- （其余类型依此类推）
```

**匹配结果分级**：
- `match_score ≥ 0.95`：**完美匹配**——可直接进入代价评估
- `0.80 ≤ match_score < 0.95`：**可匹配**——需进入协商阶段（商场可建议调整需求）
- `0.60 ≤ match_score < 0.80`：**勉强匹配**——商场需明确标注风险项
- `match_score < 0.60`：**不可匹配**——直接拒绝，返回消费者重构需求

**操作接口**：
- `match_compute(demand_vector, supply_matrix)` → 返回所有Worker的匹配度向量
- `match_rank(match_vector, top_k)` → 返回Top-K候选Worker列表
- `match_explain(demand_vector, worker_guid)` → 返回不匹配维度的详细诊断报告

### 4.4 代价评估模型（三代价输出）

**技术手段**：将匹配成功的候选方案量化为消费者与商场均可理解的代价指标。

**三代价体系**：

| 代价类型 | 计量单位 | 计算依据 | 说明 |
|---|---|---|---|
| **算力代价（Compute Cost）** | FLOP等价单元 | 任务函数级操作数 × 输入数据规模 × 循环复杂度 × SIMD加速系数 | 反映CPU/GPU/FPGA的真实计算消耗 |
| **时间代价（Time Cost）** | 毫秒 | 预估执行时长 + 调度排队时长 + SHM传输时长 + 编译进化时长 | 从契约生效到结果交付的全链路延迟 |
| **SHM代价（Memory Cost）** | MB·秒 | 峰值SHM占用 × 持有时间（含冷却期） | 反映共享内存资源的时空占用 |

**综合代价指数**：

```
total_cost = α·compute_cost + β·time_cost + γ·shm_cost

其中 α, β, γ 为商场动态调节系数：
- 算力紧张期：α ↑（抑制计算密集型任务）
- 延迟敏感场景：β ↑（抑制长耗时任务）
- SHM内存压力大：γ ↑（抑制大数据量任务）
```

**代价评估流程**：

```
匹配成功的Worker候选
    │
    ▼
[静态分析] ──→ 基于设计文档AST估算FLOP与内存需求
    │
    ▼
[动态采样] ──→ 查询该Worker历史同类型任务的实际消耗均值
    │
    ▼
[仿真推演] ──→ 在商场沙箱中运行微缩版任务（1%数据量），获取基准曲线
    │
    ▼
[三代价合成] ──→ 输出 (compute_cost, time_cost, shm_cost, confidence)
    │
    ▼
[风险标注] ──→ 若 confidence < 0.8，标记"预估不确定性高"
```

**操作接口**：
- `cost_estimate(task_design, worker_guid)` → 返回三代价与置信度
- `cost_compare(candidate_list)` → 返回代价最优候选排序
- `cost_breakdown(task_guid)` → 任务完成后返回实际代价与预估代价的偏差分析

### 4.5 契约草案生成

**技术手段**：将需求矢量、匹配结果、代价评估封装为具有法律效力的结构化契约。

**契约草案结构**：

```json
{
  "contract_id": "CONTRACT_{timestamp}_{hash_prefix}",
  "version": "1.0",
  "parties": {
    "consumer": {
      "guid": "消费者GUID",
      "demand_vector": [0.0001, 0.85, ..., 0.5],
      "signature_pending": true
    },
    "market": {
      "guid": "商场系统GUID",
      "supply_commitment": {
        "worker_pool": ["W001", "W003", "W007"],
        "shm_quota_mb": 512,
        "compile_slot": true,
        "max_queue_depth": 3
      },
      "signature_pending": true
    }
  },
  "task_specification": {
    "task_guid": "待分配的128位GUID",
    "task_type": "COMPUTE",
    "priority": "HIGH",
    "design_doc_hash": "SHA256(...)"
  },
  "cost_commitment": {
    "compute_cost_estimate": 1.45e9,
    "time_cost_estimate_ms": 2500,
    "shm_cost_estimate_mb_s": 1280,
    "confidence": 0.87,
    "penalty_clause": "超时200%自动降级，超时500%自动熔断"
  },
  "execution_window": {
    "earliest_start": "2026-05-10T22:35:00Z",
    "latest_deadline": "2026-05-10T22:40:00Z",
    "shm_cooldown_sec": 300
  },
  "matching_evidence": {
    "match_score": 0.93,
    "top_candidate": "W003",
    "risk_flags": ["预估置信度低于0.9"]
  },
  "approval_state": "PENDING_DUAL"
}
```

**操作接口**：
- `contract_draft_generate(demand_vector, match_result, cost_result)` → 生成契约草案
- `contract_validate_draft(contract_doc)` → 校验草案结构完整性与逻辑一致性
- `contract_diff(draft_v1, draft_v2)` → 对比两个草案的差异项

### 4.6 双向批准机制（核心铁律）

**技术手段**：消费者与商场必须独立、对等、不可撤销地批准契约，任务方可获得出生权。

**批准流程**：

```
契约草案生成
    │
    ▼
[消费者审阅门] ──→ 商场将草案推送至消费者终端
    │                消费者阅读：需求矢量是否准确反映意图
    │                消费者阅读：代价评估是否可接受
    │                消费者阅读：执行窗口是否合理
    │
    ▼
{消费者决策}
    │
    ├─→ [拒绝] ──→ 消费者提交修改意见（需求调整/代价异议/窗口重议）
    │              返回需求解构引擎重新生成矢量
    │              循环直至双方达成一致或消费者放弃
    │
    └─→ [批准] ──→ 消费者使用私钥对契约哈希签名
                   consumer_signature = ECDSA_SIGN(consumer_sk, SHA256(contract_body))
                   契约状态更新为 CONSUMER_APPROVED
    │
    ▼
[商场审阅门] ──→ 商场系统校验消费者签名合法性
    │              商场系统复核：供给矩阵是否仍有效（防止批准期间资源漂移）
    │              商场系统复核：代价预估是否仍成立
    │
    ▼
{商场决策}
    │
    ├─→ [拒绝] ──→ 商场提交拒绝理由（资源漂移/系统过载/代价模型更新）
    │              返回匹配引擎重新评估
    │              或建议消费者调整需求后重试
    │
    └─→ [批准] ──→ 商场系统使用系统私钥对契约哈希签名
                   market_signature = ECDSA_SIGN(market_sk, SHA256(contract_body + consumer_signature))
                   契约状态更新为 DUAL_APPROVED
    │
    ▼
[契约锁生效] ──→ 资源预占原子化执行
    │              SHM段预分配（未写入数据，仅锁定容量）
    │              Worker槽位预占（标记为RESERVED）
    │              编译队列预留（标记为PENDING_COMPILE）
    │
    ▼
[任务正式出生] ──→ 分配128位GUID
                   进入任务状态机：CREATED → SUBMITTED → AUDITING...
                   后续流程遵循总纲领卷一
```

**批准状态枚举**：

| 状态 | 编码 | 说明 |
|---|---|---|
| PENDING_DUAL | 0x00 | 草案刚生成，等待双方审阅 |
| CONSUMER_APPROVED | 0x01 | 消费者已签，等待商场签 |
| MARKET_APPROVED | 0x02 | 商场已签，等待消费者签（理论上不会出现，商场不主动预签） |
| DUAL_APPROVED | 0x03 | 双签完成，契约锁生效 |
| CONSUMER_REJECTED | 0x04 | 消费者拒绝，附带理由 |
| MARKET_REJECTED | 0x05 | 商场拒绝，附带理由 |
| EXPIRED | 0x06 | 草案在TTL内未获双签，自动失效 |
| REVOKED | 0x07 | 双签后、任务出生前，任一方发起撤销（需双方同意） |

**操作接口**：
- `approval_consumer_sign(contract_id, consumer_sk, decision)` → 消费者签名或拒绝
- `approval_market_sign(contract_id, market_sk, decision)` → 商场签名或拒绝
- `approval_verify(contract_id)` → 校验双签完整性与哈希链
- `approval_revoke(contract_id, dual_auth)` → 双签状态下发起撤销（需双方授权）

### 4.7 契约锁与资源预占

**技术手段**：双向批准后，商场原子化锁定资源，防止调度竞争。

**契约锁结构**：

```
lock := {
    contract_id:    契约唯一标识,
    task_guid:      预分配的128位任务GUID,
    worker_pool:    预占的Worker候选列表（按优先级排序）,
    shm_reservation:{
        seg_id_pool: 预分配的SHM段ID列表,
        total_mb:    总预占容量,
        ttl_ms:      锁有效期（默认=执行窗口时长 + 冷却期 + 缓冲期）
    },
    compile_slot:   编译队列预留槽位,
    lock_timestamp: 双签完成时间戳,
    expiration:     锁自动失效时间戳,
    owner:          {consumer_guid, market_guid}
}
```

**锁生命周期**：

```
DUAL_APPROVED
    │
    ▼
[资源预占] ──→ 原子操作：shm_vector_reserve() + scheduler_reserve_worker()
    │
    ▼
{任务出生}
    │
    ▼
[任务执行中] ──→ 锁持续有效，资源不可被其他契约抢占
    │              （除非本任务优先级为CRITICAL且发生紧急调度）
    ▼
[任务终态] ──→ COMPLETED / FAILED / CANCELLED
    │
    ▼
[锁释放] ──→ 延迟释放：SHM冷却期结束后回收
    │          Worker槽位立即标记为AVAILABLE
    │          编译队列槽位立即释放
    │
    ▼
[契约归档] ──→ 审计链追加"CONTRACT_FULFILLED"或"CONTRACT_BREACHED"事件
```

**违约处理**：

| 违约方 | 违约情形 | 处理手段 |
|---|---|---|
| 消费者 | 双签后、任务出生前撤销契约 | 扣除信用积分，影响未来优先级权重 |
| 商场 | 双签后、无法按契约提供资源 | 自动升级任务为CRITICAL，从备用池调度，记录商场过失 |
| 双方 | 执行窗口内任务未完成 | 按契约`penalty_clause`执行：降级、熔断或延期 |

**操作接口**：
- `lock_acquire(contract_id, resource_plan)` → 原子预占资源
- `lock_refresh(contract_id, extension_ms)` → 延长锁有效期（需双方同意）
- `lock_release(contract_id, reason)` → 释放契约锁并回收资源
- `lock_status(contract_id)` → 查询锁当前状态与剩余TTL

---

## 五、状态机扩展：契约态融入任务生命周期

将契约匹配机制嵌入原有任务状态机，形成**前置契约态**：

```
┌─────────────────────────────────────────┐
│           新增：契约前置域               │
├─────────────────────────────────────────┤
│                                         │
│  DEMAND_CREATED(0x10)                   │
│      │                                  │
│      ▼                                  │
│  DEMAND_DECONSTRUCTED(0x11)             │
│      │                                  │
│      ▼                                  │
│  MATCHED(0x12) ──→ match_score ≥ 0.80 │
│      │                                  │
│      ▼                                  │
│  COST_ESTIMATED(0x13)                   │
│      │                                  │
│      ▼                                  │
│  CONTRACT_DRAFTED(0x14)                 │
│      │                                  │
│      ▼                                  │
│  CONSUMER_APPROVED(0x15) ──┐             │
│      │                    │             │
│      ▼                    ▼             │
│  DUAL_APPROVED(0x16) ←── MARKET_APPROVED(0x17) [理论上不出现]
│      │                                  │
│      ▼                                  │
│  LOCK_ACQUIRED(0x18) ──→ 资源预占完成    │
│      │                                  │
│      ▼                                  │
│  ┌─────────────────────────────────────┐ │
│  │  接入原有任务状态机                 │ │
│  │  CREATED(0x00) → SUBMITTED(0x01)  │ │
│  │  ... → COMPLETED/FAILED/CANCELLED  │ │
│  └─────────────────────────────────────┘ │
│      │                                  │
│      ▼                                  │
│  CONTRACT_FULFILLED(0x19) /             │
│  CONTRACT_BREACHED(0x1A)                │
│                                         │
└─────────────────────────────────────────┘
```

**跃迁规则**：
- `DEMAND_CREATED` → `DEMAND_DECONSTRUCTED`：消费者提交原始意图
- `DEMAND_DECONSTRUCTED` → `MATCHED`：需求矢量生成且匹配成功
- `MATCHED` → `COST_ESTIMATED`：代价评估完成
- `COST_ESTIMATED` → `CONTRACT_DRAFTED`：契约草案生成
- `CONTRACT_DRAFTED` → `CONSUMER_APPROVED`：消费者签名
- `CONSUMER_APPROVED` → `DUAL_APPROVED`：商场签名
- `DUAL_APPROVED` → `LOCK_ACQUIRED`：资源预占成功
- `LOCK_ACQUIRED` → `CREATED`：任务正式出生，接入原有状态机
- 任意契约态 → `CONTRACT_EXPIRED`：TTL超时
- 任务终态 → `CONTRACT_FULFILLED`（成功）或 `CONTRACT_BREACHED`（失败/取消）

---

## 六、安全与审计

### 6.1 签名与不可否认性

- **消费者密钥对**：每个消费者在注册时生成ECDSA P-256密钥对，私钥本地安全存储，公钥注册至商场身份链
- **商场系统密钥对**：商场持有系统级密钥对，用于所有契约的商场端签名
- **签名内容**：对契约正文（不含签名字段本身）的SHA-256哈希进行签名
- **链式验证**：消费者签名覆盖契约正文；商场签名覆盖`契约正文 + 消费者签名`，形成不可拆解的签名链

### 6.2 审计事件扩展

在原有审计追踪链基础上，新增契约相关事件：

| 事件名称 | Actor | 载荷 |
|---|---|---|
| DEMAND_SUBMITTED | CONSUMER | 原始需求意图哈希 |
| DEMAND_DECONSTRUCTED | MARKET | 需求矢量、置信度 |
| MATCH_COMPUTED | MARKET | 匹配度向量、Top-K候选 |
| COST_ESTIMATED | MARKET | 三代价、置信度、风险标注 |
| CONTRACT_DRAFTED | MARKET | 契约草案全文哈希 |
| CONSUMER_APPROVED | CONSUMER | 消费者签名 |
| CONSUMER_REJECTED | CONSUMER | 拒绝理由、修改建议 |
| MARKET_APPROVED | MARKET | 商场签名 |
| MARKET_REJECTED | MARKET | 拒绝理由、资源漂移证据 |
| DUAL_APPROVED | SYSTEM | 双签哈希、契约锁ID |
| LOCK_ACQUIRED | MARKET | 资源预占详情 |
| LOCK_RELEASED | MARKET | 释放原因、资源回收详情 |
| CONTRACT_FULFILLED | SYSTEM | 实际代价 vs 预估代价 |
| CONTRACT_BREACHED | SYSTEM | 违约方、违约类型、处理结果 |

---

## 七、测试验证体系（增补）

### 7.1 需求解构测试

- **模糊意图测试**：输入"帮我算个快速傅里叶变换，数据量不大，尽快出结果"，验证解构引擎能否正确映射为`COMPUTE`类型、`NORMAL`优先级、合理SHM预算
- **边界约束测试**：输入SHM预算超过系统上限的需求，验证`demand_validate`拒绝并提示
- **语义距离测试**：提交两个仅优先级不同的需求，验证`demand_diff`正确识别差异维度

### 7.2 匹配引擎测试

- **完美匹配测试**：需求矢量与某Worker列完全对齐，验证`match_score = 1.0`
- **不可匹配测试**：需求为`IO`类型但某Worker不支持IO，验证`match_score = 0`
- **部分匹配测试**：需求SHM预算100MB，Worker仅剩80MB，验证`match_score`按80%折算
- **动态漂移测试**：匹配完成后Worker状态突变，验证商场在批准门前复核并拒绝

### 7.3 代价评估测试

- **静态分析精度测试**：对已知FLOP的基准任务，验证`compute_cost`误差<15%
- **历史采样测试**：对重复执行的任务，验证预估代价收敛于历史均值
- **沙箱推演测试**：对比微缩版任务与全量任务的代价比例，验证线性外推合理性

### 7.4 双向批准测试

- **单签无效测试**：仅消费者签或仅商场签，验证任务无法出生
- **签名伪造测试**：篡改消费者公钥或签名内容，验证`approval_verify`拒绝
- **TTL超时测试**：双签前等待超过TTL，验证契约自动进入`CONTRACT_EXPIRED`
- **资源漂移测试**：消费者签后、商场签前，Worker宕机，验证商场拒绝并重新匹配

### 7.5 契约锁测试

- **原子预占测试**：并发多个契约申请同一Worker，验证只有一个获得锁
- **违约扣减测试**：消费者撤销契约，验证信用积分扣减记录写入审计链
- **延迟释放测试**：任务完成后，验证SHM段在冷却期结束后才回收

---

## 八、审核通过标准（增补）

任何实现本增补卷技术手段的代码，除满足卷一标准外，还必须：

1. **契约完整性**：完整实现需求解构、匹配、代价评估、草案生成、双向批准、契约锁六阶段
2. **签名安全性**：使用标准ECDSA库，私钥不可泄露至日志或内存转储
3. **资源原子性**：契约锁的预占与释放必须是原子操作，禁止中间态资源悬空
4. **商场主权**：商场在批准门拥有**最终否决权**，即使消费者已签，商场仍可基于资源现实拒绝
5. **消费者知情权**：代价评估结果必须以人类可读形式呈现，禁止隐藏或模糊化
6. **测试覆盖**：契约前置域全部状态跃迁路径100%覆盖，双签组合（签/拒/超时）全部覆盖

---

## 九、结语：契约即秩序

> **单向提交流是封建制——消费者为君主，商场为附庸，任务为贡品。**  
> **双向契约制是共和制——消费者与商场为平等缔约方，任务为共同意志的结晶。**

本增补卷将商场从被动的任务接收器，提升为**主动的算力市场仲裁者**。商场不再盲目接受一切需求，而是以供给矩阵为尺、以代价模型为秤，衡量每一份需求的重量。消费者不再盲目提交一切意图，而是以需求矢量为镜、以代价评估为鉴，审视自身需求的合理性。

**双向批准，是信任的仪式，更是权力的制衡。**

当消费者的数字签名与商场的系统签名在契约草案上交汇，那一刻，任务不再是孤立的代码片段，而是一个被双方共同承认的、具有资源边界与时空约束的**算力生命体**。它将在契约锁的庇护下诞生，在SHM矢量的滋养下生长，在审计链的注视下终老。

**此机制生效之日，即为微型商场从"作坊"迈向"市场"的成人礼。**

---

**本增补卷与卷一共同构成《任务流程操作总纲领》完整版。**  
**人类架构师保留对本增补卷的最终解释权与修订权。**  
**商场编译进化权与消费者需求发起权，在此契约框架下达成永恒平衡。**

> *任务即原子，流程即律法，SHM即血液，审核即灵魂，契约即秩序。*  
> *商场编译进化，Worker忠实执行，消费者安心设计——此三者，共筑AICoder之基石。*
