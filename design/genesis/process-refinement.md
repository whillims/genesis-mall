> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。
创世纪流程细化

---

## 一、概述

创世纪是商场系统的**初始化阶段**，负责从无到有地构建整个生产者生态。该阶段包含**七个相变阶段**，每个阶段都有明确的输入、输出、约束条件和产物。

### 1.1 阶段总览

| 阶段编号 | 阶段名称 | 阶段标识 | 核心目标 | 关键产出 |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 虚空 | PHASE_VOID | 系统自检与配置加载 | genesis_config.bin |
| 1 | 火花 | PHASE_SPARK | Worker实体化与密钥生成 | Worker身份档案 |
| 2 | 蓝图 | PHASE_BLUEPRINT | 设计意图文档提交与预审 | 蓝图文档(BID) |
| 3 | 设计 | PHASE_DESIGN | AICoder函数级设计生成 | 设计产物包 |
| 4 | 试炼 | PHASE_TRIAL | 编译验证、调试修复、量化评估 | 评估报告 |
| 5 | 诞生 | PHASE_BIRTH | 生产者注册与资源分配 | 出生证明 |
| 6 | 初光 | PHASE_FIRSTLIGHT | 首次生产-消费循环验证 | 验收裁决书 |

### 1.2 状态机转换规则

```
PHASE_VOID ──[健康检查通过]──> PHASE_SPARK ──[密钥生成完成]──> PHASE_BLUEPRINT
     ^                                                           │
     │                                                           │
     └────────────────────────────────────────────────────────────┘
                              [试炼失败退回]
```

---

## 二、阶段零：虚空（PHASE_VOID）

### 2.1 目标

系统自检阶段，验证运行环境、依赖服务、权限配置是否就绪。建立创世纪配置句柄，探测AICoder服务健康状态。

### 2.2 函数调用链

```c
/* 2.2.1 创世纪启动自检 */
int genesis_void_bootstrap(
    const char* config_path,       /* 输入：创世配置文件路径（JSON格式） */
    GenesisConfig* out_config      /* 输出：创世配置结构体（内存映射） */
);

/* 2.2.2 探测AICoder服务健康状态 */
int genesis_void_aicoder_probe(
    AICoderEndpoint* endpoint,     /* 输入：AICoder端点（IP:端口） */
    GenesisConfig*    config,       /* 输入：创世配置（含超时配置） */
    int               timeout_ms,   /* 输入：探测超时（默认3000ms） */
    AICoderHealth* out_health      /* 输出：健康状态结构体 */
);

/* 2.2.3 阶段门：只有健康检查全通过才允许相变 */
int genesis_phase_gate(
    enum GenesisPhase current,     /* 输入：当前阶段 */
    enum GenesisPhase target,      /* 输入：目标阶段 */
    void*             phase_evidence, /* 输入：阶段产出证据（文件内存映射） */
    size_t            evidence_size,  /* 输入：证据大小（字节） */
    GenesisAuth*      out_auth       /* 输出：阶段通行证（含数字签名） */
);
```

### 2.3 参数详解

| 函数 | 参数 | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| `genesis_void_bootstrap` | `config_path` | `const char*` | 配置文件路径，JSON格式，包含AICoder端点、超时配置、资源配额上限 |
| | `out_config` | `GenesisConfig*` | 输出配置结构体，后续阶段只读引用 |
| `genesis_void_aicoder_probe` | `endpoint` | `AICoderEndpoint*` | AICoder服务端点，包含协议(http/https)、IP地址、端口 |
| | `timeout_ms` | `int` | 探测超时时间，建议值3000-10000ms |
| | `out_health` | `AICoderHealth*` | 健康状态：{status: PASS/FAIL, latency_ms, version} |

### 2.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `genesis_void_report.md` | Markdown | 虚空阶段自检报告，包含环境检查、依赖验证、权限检查结果 |
| `genesis_config.bin` | 二进制 | 创世配置句柄，后续阶段只读引用，内存映射加载 |

### 2.5 错误处理

| 错误码 | 含义 | 处理策略 |
| :--- | :--- | :--- |
| `GENESIS_ERR_CONFIG_NOT_FOUND` | 配置文件不存在 | 终止创世纪，输出错误日志 |
| `GENESIS_ERR_AICODER_UNREACHABLE` | AICoder服务不可达 | 重试3次后终止 |
| `GENESIS_ERR_PERMISSION_DENIED` | 权限不足 | 检查运行用户权限，终止 |

---

## 三、阶段一：火花（PHASE_SPARK）

### 3.1 目标

创世Worker从虚空中被实例化。这是第一个生产者的降世，但它此时只是**裸Worker**，尚未获得任何生产能力，仅具备元能力（提交蓝图、请求AICoder）。

### 3.2 函数调用链

```c
/* 3.2.1 创世Worker实体化 */
int genesis_spark_worker_incarnate(
    GenesisConfig*    config,            /* 输入：创世配置（只读） */
    const char*       worker_name,       /* 输入：Worker命名（唯一标识，如"progenitor_alpha"） */
    WorkerClass       wclass,            /* 输入：Worker阶级（创世者固定为CLASS_PROGENITOR） */
    SHMVector*        shm_state_vector,  /* 输入：预分配的SHM状态矢量槽位 */
    WorkerHandle*     out_handle         /* 输出：Worker句柄（后续所有操作的身份证） */
);

/* 3.2.2 赋予Worker"元能力"——只能提交蓝图，不能生产 */
int genesis_spark_worker_empower(
    WorkerHandle*     handle,            /* 输入/输出：Worker句柄（能力位图将被修改） */
    CapabilityMask    meta_caps,         /* 输入：元能力掩码（BLUEPRINT_SUBMIT | AICODER_REQUEST） */
    GenesisAuth*      phase1_auth        /* 输入：阶段一通行证（来自phase_gate） */
);

/* 3.2.3 生成Worker的创世密钥对（用于后续所有文档签名） */
int genesis_spark_worker_keygen(
    WorkerHandle*     handle,            /* 输入：Worker句柄 */
    CryptoSuite       suite,             /* 输入：加密套件（固定ED25519） */
    WorkerIdentity*   out_identity       /* 输出：包含公钥、私钥、指纹的身份结构 */
);
```

### 3.3 参数详解

| 函数 | 参数 | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| `genesis_spark_worker_incarnate` | `worker_name` | `const char*` | Worker唯一标识，最大64字符，仅允许[a-zA-Z0-9_-] |
| | `wclass` | `WorkerClass` | 枚举值：CLASS_PROGENITOR（创世者）、CLASS_PRODUCER（生产者）等 |
| | `shm_state_vector` | `SHMVector*` | 预分配的SHM段指针，用于存储Worker状态 |
| `genesis_spark_worker_empower` | `meta_caps` | `CapabilityMask` | 位掩码：0x01=BLUEPRINT_SUBMIT, 0x02=AICODER_REQUEST |
| `genesis_spark_worker_keygen` | `suite` | `CryptoSuite` | 固定为ED25519，不支持其他算法 |

### 3.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `worker_{name}_identity.json` | JSON | Worker身份档案，包含公钥、指纹、创建时间戳、阶级标识 |
| `worker_{name}_capabilities.bin` | 二进制 | 能力位图快照，记录当前Worker具备的所有能力 |

### 3.5 SHM预分配规则

每个Worker在实体化时必须预分配以下SHM资源：

| 资源类型 | 最小大小 | 用途 |
| :--- | :--- | :--- |
| 状态矢量 | 128KB | 存储Worker状态机上下文 |
| 输入通道 | 256KB | 接收外部请求数据 |
| 输出通道 | 256KB | 发送生产结果数据 |

---

## 四、阶段二：蓝图（PHASE_BLUEPRINT）

### 4.1 目标

创世Worker提交第一份设计意图文档。这是整个商场宇宙的**第一因**——此后所有的代码、规则、技能集都源自此蓝图。

### 4.2 函数调用链

```c
/* 4.2.1 Worker撰写蓝图（本地操作，不经过商场） */
int genesis_blueprint_compose(
    WorkerHandle*     handle,            /* 输入：Worker身份 */
    BlueprintTemplate tmpl,              /* 输入：蓝图模板（固定为TMPL_FUNCTIONAL_DESIGN） */
    const char*       intent_description,/* 输入：自然语言设计意图（最大4096字符） */
    const char*       target_language,   /* 输入：目标语言（Python/Verilog/C/Java/JS/CSS） */
    BlueprintDraft*   out_draft          /* 输出：蓝图草稿结构 */
);

/* 4.2.2 Worker对蓝图进行数字签名（证明"这是我提交的"） */
int genesis_blueprint_sign(
    WorkerHandle*     handle,            /* 输入：Worker身份（含私钥） */
    BlueprintDraft*   draft,             /* 输入/输出：蓝图草稿（签名后变为正式蓝图） */
    Signature*        out_sig            /* 输出：分离式签名（供第三方验证） */
);

/* 4.2.3 Worker向商场提交蓝图（首次与商场交互） */
int genesis_blueprint_submit(
    WorkerHandle*     handle,            /* 输入：Worker身份 */
    BlueprintDraft*   signed_draft,      /* 输入：已签名蓝图 */
    GenesisAuth*      phase2_auth,       /* 输入：阶段二通行证 */
    BlueprintID*      out_bid            /* 输出：蓝图全局唯一ID（BID） */
);

/* 4.2.4 商场对蓝图进行形式化预审（检查完整性，不做设计评估） */
int genesis_blueprint_preflight(
    BlueprintID       bid,               /* 输入：蓝图ID */
    PreflightCheck*   out_check          /* 输出：预审报告（通过/失败/警告） */
);
```

### 4.3 参数详解

| 函数 | 参数 | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| `genesis_blueprint_compose` | `intent_description` | `const char*` | 自然语言描述，如"实现一个FFT频谱分析生产者，支持1024点复数FFT" |
| | `target_language` | `const char*` | 目标语言标识，必须从预定义列表中选择 |
| `genesis_blueprint_sign` | `out_sig` | `Signature*` | 分离式签名，包含签名值、算法标识、时间戳 |
| `genesis_blueprint_submit` | `out_bid` | `BlueprintID*` | 全局唯一标识符，格式：BID-{UUID} |

### 4.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `blueprint_{BID}.json` | JSON | 正式蓝图文档，包含设计意图、目标语言、签名、时间戳 |
| `blueprint_preflight_{BID}.md` | Markdown | 预审报告，包含完整性检查、格式验证、签名验证结果 |

### 4.5 目标语言支持列表

| 语言标识 | 语言名称 | 代码后缀 | 支持状态 |
| :--- | :--- | :--- | :--- |
| Python | Python 3.x | .py | 完全支持 |
| C | C11 | .c/.h | 完全支持 |
| Java | Java 21 | .java | 完全支持 |
| JavaScript | ES2024 | .js | 完全支持 |
| TypeScript | TypeScript 5.x | .ts | 完全支持 |
| Verilog | Verilog 2001 | .v | 完全支持 |
| CSS | CSS3 | .css | 完全支持 |

---

## 五、阶段三：设计（PHASE_DESIGN）

### 5.1 目标

商场以蓝图作为输入，调用AICoder完成函数级设计。这是创世纪的**核心相变**——从自然语言意图到形式化代码的跃迁。

### 5.2 函数调用链

```c
/* 5.2.1 商场构建AICoder授权文档（核心接口） */
int genesis_design_aicoder_authdoc_build(
    BlueprintID       bid,               /* 输入：被授权的蓝图ID */
    WorkerHandle*     requester,         /* 输入：请求者（创世Worker） */
    AICoderScope      scope,             /* 输入：授权范围（DESIGN | DEBUG | EVAL） */
    AuthDocument*     out_authdoc        /* 输出：授权文档（含三重签名） */
);

/* 5.2.2 商场调用AICoder进行设计（核心函数） */
int genesis_design_aicoder_invoke(
    AuthDocument*     authdoc,           /* 输入：授权文档（AICoder验证此文档才执行） */
    AICoderEndpoint*  endpoint,          /* 输入：选定的AICoder服务端点 */
    DesignTask*       task,              /* 输入：设计任务参数包 */
    AICoderSession*   out_session        /* 输出：AICoder会话句柄 */
);

/* 5.2.3 AICoder设计任务参数包结构（关键！） */
typedef struct {
    BlueprintID     source_bid;          /* 关联的蓝图ID */
    LanguageTarget  lang;                /* 目标语言枚举 */
    DesignDepth     depth;               /* 设计深度：ARCHITECTURE/DETAIL/CODE */
    int             max_functions;       /* 最大函数数量（防止爆炸，建议16-64） */
    int             require_tests;       /* 是否强制生成测试（创世强制为1） */
    const char*     design_ruleset;      /* 设计规则集ID（如"func_design_v1.2"） */
    const char*     skillset_hint;       /* 技能集提示（如"simd_optimization"） */
} DesignTask;

/* 5.2.4 轮询AICoder设计进度（异步设计过程的脉搏） */
int genesis_design_aicoder_poll(
    AICoderSession*   session,           /* 输入：会话句柄 */
    int               timeout_ms,        /* 输入：单次轮询超时（默认5000ms） */
    DesignProgress*   out_progress       /* 输出：进度结构 */
);

/* 5.2.5 获取AICoder最终设计产出 */
int genesis_design_aicoder_collect(
    AICoderSession*   session,           /* 输入：会话句柄 */
    GenesisAuth*      phase3_auth,       /* 输入：阶段三通行证 */
    DesignArtifact*   out_artifact       /* 输出：设计产物包 */
);
```

### 5.3 参数详解（重点）

#### 5.3.1 授权范围（AICoderScope）

| 枚举值 | 含义 | 权限描述 |
| :--- | :--- | :--- |
| `SCOPE_DESIGN` | 设计 | 允许AICoder生成设计文档和代码 |
| `SCOPE_DEBUG` | 调试 | 允许AICoder分析和修复代码缺陷 |
| `SCOPE_EVAL` | 评估 | 允许AICoder评估代码质量和性能 |

#### 5.3.2 设计深度（DesignDepth）

| 枚举值 | 深度描述 | 产出内容 |
| :--- | :--- | :--- |
| `DEPTH_ARCHITECTURE` | 架构级 | 仅生成架构设计文档 |
| `DEPTH_DETAIL` | 详细设计 | 架构+详细设计文档 |
| `DEPTH_CODE` | 代码级 | 完整设计文档+可编译代码 |

#### 5.3.3 授权文档结构（AuthDocument）

```c
typedef struct {
    uint32_t        version;            /* 版本号 */
    BlueprintID     bid;                /* 关联蓝图ID */
    WorkerID        requester_id;       /* 请求者ID */
    AICoderScope    scope;              /* 授权范围 */
    uint64_t        timestamp;          /* 创建时间戳 */
    uint64_t        expire_at;          /* 过期时间戳 */
    Signature       worker_sig;         /* Worker私钥签名 */
    Signature       mall_sig;           /* 商场私钥签名 */
    uint8_t         blueprint_hash[32]; /* 蓝图SHA256哈希 */
} AuthDocument;
```

### 5.4 产出文档（AICoder设计包）

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `design_{BID}_req.md` | Markdown | 需求分析文档，包含功能需求、非功能需求 |
| `design_{BID}_arch.md` | Markdown | 架构设计文档，包含模块划分、接口设计 |
| `design_{BID}_detail.md` | Markdown | 详细设计文档，包含函数调用图、参数表 |
| `design_{BID}_api.md` | Markdown | 接口定义文档，包含API规范、数据结构 |
| `code_{BID}/` | 目录 | 代码目录，按函数分文件组织 |
| `test_{BID}/` | 目录 | 测试代码目录，包含单元测试、集成测试 |
| `rules_{BID}.json` | JSON | 设计规则校验文件，用于后续验证 |

---

## 六、阶段四：试炼（PHASE_TRIAL）

### 6.1 目标

对AICoder产出的设计进行编译验证、调试修复、量化评估。这是商场的**质量门**，未通过试炼的设计将被焚毁（删除），蓝图退回重审。

### 6.2 函数调用链

```c
/* 6.2.1 编译试炼：将代码包编译为目标文件 */
int genesis_trial_compile(
    DesignArtifact*   artifact,          /* 输入：设计产物 */
    CompileTarget     target,            /* 输入：编译目标（OS/ARCH/ABI） */
    CompileReport*    out_report         /* 输出：编译报告 */
);

/* 6.2.2 调试试炼：运行测试函数，捕获崩溃与断言失败 */
int genesis_trial_debug(
    DesignArtifact*   artifact,          /* 输入：设计产物（含测试代码） */
    DebugRuntime*     runtime,           /* 输入：调试运行时环境（沙箱） */
    int               max_iterations,    /* 输入：最大调试迭代次数（默认3） */
    DebugReport*      out_report        /* 输出：调试报告 */
);

/* 6.2.3 评估试炼：量化评分，决定生产者是否合格 */
int genesis_trial_evaluate(
    CompileReport*    compile_rpt,       /* 输入：编译报告 */
    DebugReport*      debug_rpt,         /* 输入：调试报告 */
    EvaluateMetric*   metrics,           /* 输入：评估指标权重配置 */
    EvaluateReport*   out_eval_report    /* 输出：评估报告 */
);

/* 6.2.4 AICoder调试修复回调（当测试失败时自动触发） */
int genesis_trial_aicoder_repair(
    AICoderSession*   design_session,    /* 输入：原始设计会话（上下文保留） */
    DebugReport*      failure_evidence,  /* 输入：失败证据 */
    RepairStrategy    strategy,          /* 输入：修复策略 */
    DesignArtifact*   inout_artifact     /* 输入/输出：设计产物 */
);
```

### 6.3 参数详解

#### 6.3.1 编译目标（CompileTarget）

| 参数 | 说明 | 示例值 |
| :--- | :--- | :--- |
| `os` | 操作系统 | linux/windows/macos |
| `arch` | 架构 | x86_64/arm64/riscv64 |
| `abi` | ABI版本 | gnu/musl/msvc |

#### 6.3.2 修复策略（RepairStrategy）

| 策略 | 描述 | 适用场景 |
| :--- | :--- | :--- |
| `STRATEGY_CONSERVATIVE` | 保守修复，仅修复语法错误和明显bug | 编译错误、简单逻辑错误 |
| `STRATEGY_AGGRESSIVE` | 激进修复，可重构代码结构 | 测试失败、性能问题 |

#### 6.3.3 评估指标权重（EvaluateMetric）

| 指标 | 权重 | 说明 |
| :--- | :--- | :--- |
| `metric_compile` | 20% | 编译是否通过，警告数量 |
| `metric_test_pass` | 30% | 测试通过率 |
| `metric_coverage` | 20% | 代码覆盖率 |
| `metric_performance` | 15% | 性能基准测试 |
| `metric_complexity` | 15% | 代码复杂度评估 |

### 6.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `trial_compile_{BID}.md` | Markdown | 编译报告，包含编译结果、警告信息、耗时 |
| `trial_debug_{BID}.md` | Markdown | 调试报告，包含测试覆盖率、崩溃日志 |
| `trial_eval_{BID}.md` | Markdown | 评估报告，量化评分卡（总分≥80为通过） |
| `trial_repair_{BID}.log` | 文本 | 修复日志，记录修复次数、修复内容 |

### 6.5 试炼通过条件

```
总分 >= 80 分 AND 测试通过率 >= 95% AND 编译无错误
```

---

## 七、阶段五：诞生（PHASE_BIRTH）

### 7.1 目标

通过试炼的设计产物被注册为正式生产者。Worker从**裸Worker**进化为**生产公民**，获得商场内的生产权限和SHM资源配额。

### 7.2 函数调用链

```c
/* 7.2.1 将设计产物注册为生产者的"灵魂"（可执行代码映像） */
int genesis_birth_soul_imprint(
    WorkerHandle*     handle,            /* 输入：创世Worker句柄 */
    DesignArtifact*   certified_artifact,/* 输入：通过试炼的设计产物（只读） */
    SoulImage*        out_soul           /* 输出：灵魂映像 */
);

/* 7.2.2 为生产者分配SHM资源配额 */
int genesis_birth_resource_grant(
    WorkerHandle*     handle,            /* 输入：生产者 */
    ResourceQuota     quota,             /* 输入：资源配额请求 */
    SHMVector*        out_io_vector      /* 输出：分配的SHM矢量通道 */
);

/* 7.2.3 提升Worker能力位图：从"元能力"扩展到"生产能力" */
int genesis_birth_worker_evolve(
    WorkerHandle*     handle,            /* 输入/输出：Worker句柄 */
    CapabilityMask    production_caps,    /* 输入：新增能力 */
    GenesisAuth*      phase5_auth        /* 输入：阶段五通行证 */
);

/* 7.2.4 生产者在商场公示牌注册（其他Worker可见） */
int genesis_birth_registry_enroll(
    WorkerHandle*     handle,            /* 输入：生产者 */
    SoulImage*        soul,              /* 输入：灵魂映像 */
    RegistryEntry*    out_entry          /* 输出：商场公示牌条目 */
);
```

### 7.3 参数详解

#### 7.3.1 资源配额结构（ResourceQuota）

```c
typedef struct {
    size_t  shm_bytes;          /* SHM内存配额（字节） */
    int     cpu_cores;          /* CPU核心数 */
    int     max_concurrent;     /* 最大并发任务数 */
    int     priority;           /* 调度优先级（1-10） */
} ResourceQuota;
```

#### 7.3.2 生产能力掩码（CapabilityMask）

| 能力位 | 值 | 说明 |
| :--- | :--- | :--- |
| `CAP_PRODUCE` | 0x04 | 允许执行生产任务 |
| `CAP_CONSUME` | 0x08 | 允许消费其他生产者的产出 |
| `CAP_PUBLISH` | 0x10 | 允许发布服务到注册表 |

### 7.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `birth_certificate_{WORKER_ID}.json` | JSON | 出生证明，包含灵魂哈希、资源配额、能力位图 |
| `registry_entry_{WORKER_ID}.json` | JSON | 商场公示牌条目，其他Worker可见 |

---

## 八、阶段六：初光（PHASE_FIRSTLIGHT）

### 8.1 目标

验证生产者的第一次实际生产-消费循环。这是创世纪的**最终验收**——如果初光成功，证明整个创世纪流程闭环，商场进入正常运行态。

### 8.2 函数调用链

```c
/* 8.2.1 创建第一个"消费者"（可以是商场的测试消费者） */
int genesis_firstlight_consumer_spawn(
    GenesisConfig*    config,            /* 输入：创世配置 */
    const char*       consumer_name,     /* 输入：消费者名称 */
    ConsumerProfile profile,            /* 输入：消费者画像 */
    ConsumerHandle*   out_consumer       /* 输出：消费者句柄 */
);

/* 8.2.2 消费者向商场发布需求（触发生产） */
int genesis_firstlight_request_post(
    ConsumerHandle*   consumer,          /* 输入：消费者 */
    RequestTicket*    ticket,            /* 输入：需求票证 */
    SHMVector*        request_vector     /* 输入/输出：需求数据SHM矢量 */
);

/* 8.2.3 商场调度生产者处理需求（首次调度） */
int genesis_firstlight_dispatch(
    RegistryEntry*    producer_entry,    /* 输入：生产者公示牌条目 */
    RequestTicket*    ticket,            /* 输入：需求票证 */
    DispatchContext*  out_context        /* 输出：调度上下文 */
);

/* 8.2.4 生产者执行生产（调用其灵魂映像中的入口函数） */
int genesis_firstlight_produce(
    WorkerHandle*     producer,          /* 输入：生产者 */
    SHMVector*        input_vector,      /* 输入：需求数据 */
    SHMVector*        output_vector,     /* 输出：生产结果 */
    ProduceMetrics*   out_metrics       /* 输出：生产指标 */
);

/* 8.2.5 消费者验收产出 */
int genesis_firstlight_consume(
    ConsumerHandle*   consumer,          /* 输入：消费者 */
    SHMVector*        output_vector,     /* 输入：生产结果 */
    AcceptanceVerdict* out_verdict       /* 输出：验收裁决 */
);
```

### 8.3 参数详解

#### 8.3.1 需求票证结构（RequestTicket）

```c
typedef struct {
    RequestID    id;             /* 需求唯一标识 */
    DataType     data_type;      /* 数据类型标识 */
    int          priority;       /* 优先级（1-10） */
    uint64_t     deadline;       /* 截止时间戳 */
    size_t       data_size;      /* 数据大小 */
    uint8_t      flags;          /* 标志位 */
} RequestTicket;
```

#### 8.3.2 验收裁决（AcceptanceVerdict）

| 枚举值 | 含义 | 后续处理 |
| :--- | :--- | :--- |
| `VERDICT_ACCEPT` | 验收通过 | 创世纪完成，进入稳态运行 |
| `VERDICT_REJECT` | 验收失败 | 销毁生产者，蓝图退回重审 |
| `VERDICT_REWORK` | 需要返工 | 重新进入设计阶段 |

### 8.4 产出文档

| 产出文件 | 格式 | 用途 |
| :--- | :--- | :--- |
| `firstlight_dispatch_{TIMESTAMP}.md` | Markdown | 调度记录，包含调度策略、资源分配 |
| `firstlight_produce_{TIMESTAMP}.md` | Markdown | 生产性能报告，包含延迟、吞吐量、资源消耗 |
| `firstlight_verdict_{TIMESTAMP}.md` | Markdown | 验收裁决书，包含验收结果、问题记录 |

---

## 九、创世纪完整调用序列图

```
main()
  └─> genesis_void_bootstrap() ──> [产出 genesis_config.bin]
        └─> genesis_void_aicoder_probe() ──> [健康: PASS]
              └─> genesis_phase_gate(PHASE_VOID→PHASE_SPARK)
                    └─> genesis_spark_worker_incarnate("progenitor_alpha")
                          └─> genesis_spark_worker_empower(BLUEPRINT_SUBMIT|AICODER_REQUEST)
                                └─> genesis_spark_worker_keygen()
                                      └─> genesis_phase_gate(PHASE_SPARK→PHASE_BLUEPRINT)
                                            └─> genesis_blueprint_compose("实现FFT频谱分析生产者", "C")
                                                  └─> genesis_blueprint_sign()
                                                        └─> genesis_blueprint_submit()
                                                              └─> genesis_blueprint_preflight() ──> [PASS]
                                                                    └─> genesis_phase_gate(PHASE_BLUEPRINT→PHASE_DESIGN)
                                                                          └─> genesis_design_aicoder_authdoc_build(DESIGN|DEBUG|EVAL)
                                                                                └─> genesis_design_aicoder_invoke(task={depth:CODE, max_functions:16, require_tests:1})
                                                                                      └─> genesis_design_aicoder_poll() ──> [进度100%]
                                                                                            └─> genesis_design_aicoder_collect() ──> [产出 design_artifact]
                                                                                                  └─> genesis_phase_gate(PHASE_DESIGN→PHASE_TRIAL)
                                                                                                        ├─> genesis_trial_compile() ──> [PASS]
                                                                                                        ├─> genesis_trial_debug() ──> [测试通过率: 100%]
                                                                                                        └─> genesis_trial_evaluate() ──> [总分: 92/100, PASS]
                                                                                                              └─> genesis_phase_gate(PHASE_TRIAL→PHASE_BIRTH)
                                                                                                                    ├─> genesis_birth_soul_imprint()
                                                                                                                    ├─> genesis_birth_resource_grant(shm:1MB, cpu:1)
                                                                                                                    ├─> genesis_birth_worker_evolve(PRODUCE|CONSUME|PUBLISH)
                                                                                                                    └─> genesis_birth_registry_enroll()
                                                                                                                          └─> genesis_phase_gate(PHASE_BIRTH→PHASE_FIRSTLIGHT)
                                                                                                                                ├─> genesis_firstlight_consumer_spawn("test_consumer")
                                                                                                                                ├─> genesis_firstlight_dispatch()
                                                                                                                                ├─> genesis_firstlight_produce() ──> [latency: 12ms]
                                                                                                                                └─> genesis_firstlight_consume() ──> [VERDICT: ACCEPT]
                                                                                                                                      └─> [创世纪完成，商场进入稳态运行]
```

---

## 十、关键设计铁律（函数级约束）

### 10.1 禁止OOP原则

所有接口均为**C风格函数**，数据结构为`struct`，无类、无继承、无多态。函数命名采用`genesis_{phase}_{action}`的层级命名法。

### 10.2 SHM矢量隔离原则

每个Worker的`SHMVector`必须在`genesis_spark_worker_incarnate`时**预分配**，后续阶段只传递指针，**禁止动态分配**。

### 10.3 文档即证据原则

每个阶段产出必须**持久化为文件**，`genesis_phase_gate()`的`phase_evidence`参数必须指向这些文件的内存映射。

### 10.4 AICoder授权不可伪造原则

`AuthDocument`必须包含**三重签名**：
- Worker私钥签名
- 商场私钥签名  
- 蓝图哈希

AICoder验证失败则拒绝服务。

### 10.5 试炼不过则焚毁原则

`genesis_trial_evaluate()`返回`passed=0`时，自动触发`genesis_blueprint_purge()`，删除该蓝图的所有产物，Worker退回`PHASE_SPARK`状态（保留身份，清空能力）。

### 10.6 状态机严格顺序原则

阶段转换必须严格按照以下顺序：

```
PHASE_VOID → PHASE_SPARK → PHASE_BLUEPRINT → PHASE_DESIGN 
    ↓                                           ↑
    └────────── PHASE_TRIAL → PHASE_BIRTH → PHASE_FIRSTLIGHT ──┘
```

跳过任何阶段或逆序转换将被拒绝。

---

## 十一、错误码体系

| 错误码前缀 | 范围 | 含义 |
| :--- | :--- | :--- |
| `GENESIS_ERR_VOID_*` | 1000-1099 | 虚空阶段错误 |
| `GENESIS_ERR_SPARK_*` | 1100-1199 | 火花阶段错误 |
| `GENESIS_ERR_BLUEPRINT_*` | 1200-1299 | 蓝图阶段错误 |
| `GENESIS_ERR_DESIGN_*` | 1300-1399 | 设计阶段错误 |
| `GENESIS_ERR_TRIAL_*` | 1400-1499 | 试炼阶段错误 |
| `GENESIS_ERR_BIRTH_*` | 1500-1599 | 诞生阶段错误 |
| `GENESIS_ERR_FIRSTLIGHT_*` | 1600-1699 | 初光阶段错误 |

---

## 十二、总结

这份规范将创世纪从"宏大叙事"压缩为可逐行实现的函数清单。每个函数的参数、约束、产出都已明确定义：

- **7个阶段**：虚空→火花→蓝图→设计→试炼→诞生→初光
- **22个核心函数**：覆盖整个创世纪流程
- **严格的状态机**：确保阶段转换的正确性
- **完善的错误处理**：每个阶段都有明确的错误码体系
- **文档即证据**：所有产出持久化，支持审计追溯

> **下一步建议**：可以基于此规范生成C语言头文件框架（`genesis.h`），包含所有结构体定义和函数原型。如需进一步细化某个阶段（如AICoder调用接口或评估算法），请告知。
