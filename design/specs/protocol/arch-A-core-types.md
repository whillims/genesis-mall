# 核心类型与结构体定义 v1.0

> **文档定位**：商场-AICoder 体系的**类型基石**。所有接口共享的枚举、结构体、常量在此定义。  
> **设计原则**：零动态分配、显式内存布局、可序列化至 SHM、禁止指针嵌套（扁平化设计）。

---

## 一、基础类型约定

```c
/* 基础类型别名 —— 显式宽度，跨平台一致 */
typedef uint8_t   u8;
typedef uint16_t  u16;
typedef uint32_t  u32;
typedef uint64_t  u64;
typedef int8_t    i8;
typedef int16_t   i16;
typedef int32_t   i32;
typedef int64_t   i64;
typedef float     f32;
typedef double    f64;

/* 布尔类型 —— 显式避免 stdbool.h 的宏污染 */
typedef u8 mall_bool;
#define MALL_TRUE  1
#define MALL_FALSE 0

/* 字符串定长缓冲区 —— 禁止变长字符串，所有字符串必须预分配固定长度 */
#define MALL_UUID_LEN      37   /* 36字符 + null终止符 */
#define MALL_MAX_NAME_LEN  64   /* 标识符最大长度 */
#define MALL_MAX_PATH_LEN  256  /* SHM 路径最大长度 */
#define MALL_MAX_ERR_MSG   256  /* 错误消息最大长度 */
#define MALL_MAX_SHM_KEYS  16   /* 单个授权文档的 SHM 键数量上限 */
#define MALL_MAX_DEPS      8    /* 单个函数的依赖项上限 */
#define MALL_MAX_TEST_CASES 64  /* 测试用例集上限 */
#define MALL_MAX_AUDIT_EVENTS 128 /* 审计事件上限 */
```

---

## 二、错误码枚举

```c
/* 错误码体系 —— 分层编码，便于路由处理
 * 1xx: 商场基础设施层错误
 * 2xx: AICoder 执行层错误
 * 3xx: 策略/合规层错误
 * 4xx: SHM/内存层错误
 */
typedef enum {
    /* 成功 */
    MALL_OK = 0,

    /* 1xx —— 商场基础设施层 */
    MALL_ERR_AUTH_FAIL = 101,        /* 授权文档验签失败 */
    MALL_ERR_PERM_DENIED = 102,        /* 权限矢量不足 */
    MALL_ERR_POOL_FULL = 103,          /* 算力池无可用 Worker */
    MALL_ERR_POOL_TIMEOUT = 104,       /* 算力池分配超时 */
    MALL_ERR_TASK_NOT_FOUND = 105,     /* 任务令牌不存在 */
    MALL_ERR_TASK_EXPIRED = 106,       /* 任务 TTL 耗尽 */
    MALL_ERR_PRODUCER_BANNED = 107,    /* 生产者被永久吊销 */

    /* 2xx —— AICoder 执行层 */
    AICODER_ERR_DESIGN_FAIL = 201,     /* 设计阶段算法选择失败 */
    AICODER_ERR_DESIGN_OOP = 202,      /* 设计阶段检测到 OOP 语义 */
    AICODER_ERR_DEBUG_FAIL = 203,      /* 调试阶段无法定位根因 */
    AICODER_ERR_DEBUG_PATCH_INVALID = 204, /* 补丁验证失败 */
    AICODER_ERR_EVAL_FAIL = 205,       /* 评估环境异常 */
    AICODER_ERR_EVAL_TEST_CRASH = 206, /* 测试执行崩溃 */
    AICODER_ERR_RESOURCE_EXHAUST = 207, /* Token/内存/时间耗尽 */

    /* 3xx —— 策略/合规层 */
    AICODER_ERR_OOP_DETECTED = 301,    /* OOP 扫描一票否决 */
    AICODER_ERR_CONSTRAINT_VIOLATION = 302, /* 违反 FunctionSpec 约束 */
    AICODER_ERR_DEPENDENCY_REJECTED = 303, /* 依赖风险等级超限 */
    AICODER_ERR_SIGNATURE_MISMATCH = 304, /* 函数签名哈希不一致 */

    /* 4xx —— SHM/内存层 */
    MALL_ERR_SHM_ALLOC_FAIL = 401,     /* SHM 分配失败 */
    MALL_ERR_SHM_ACCESS_DENIED = 402,  /* SHM 键越权访问 */
    MALL_ERR_SHM_CORRUPTION = 403,     /* SHM 数据校验失败 */
    MALL_ERR_SHM_SERIALIZE_FAIL = 404, /* 序列化/反序列化失败 */

    MALL_ERR_UNKNOWN = 999
} MallErrorCode;

/* 错误详情结构体 —— 扁平化，可直接写入 SHM */
typedef struct {
    MallErrorCode code;
    char message[MALL_MAX_ERR_MSG];   /* 人类可读错误描述 */
    u64 timestamp_ns;                  /* 错误发生时间戳（纳秒级） */
    char source_function[MALL_MAX_NAME_LEN]; /* 产生错误的函数名 */
} MallError;
```

---

## 三、授权文档相关结构体

### 3.1 权限矢量

```c
/* 权限等级 —— 0~3 整数标量 */
typedef enum {
    PERM_FORBIDDEN = 0,   /* 禁止 */
    PERM_ADVISORY = 1,    /* 只读/建议 */
    PERM_GENERATIVE = 2,  /* 生成/设计 */
    PERM_FULL = 3         /* 生成+评估+调试 */
} PermissionLevel;

/* 权限矢量结构体 —— 固定布局，可直接 memcmp 比对 */
typedef struct {
    PermissionLevel design;       /* 函数设计权限 */
    PermissionLevel debug;        /* 代码调试权限 */
    PermissionLevel evaluate;     /* 质量评估权限 */
    PermissionLevel document;     /* 文档生成权限 */
    PermissionLevel dependency;   /* 依赖引入权限 */
} PermissionVector;
```

### 3.2 函数规范原子

```c
/* 数据类型枚举 —— 严格限定，禁止自定义类型 */
typedef enum {
    TYPE_I8, TYPE_I16, TYPE_I32, TYPE_I64,
    TYPE_U8, TYPE_U16, TYPE_U32, TYPE_U64,
    TYPE_F32, TYPE_F64,
    TYPE_BOOL,
    TYPE_COMPLEX64, TYPE_COMPLEX128,
    TYPE_NDARRAY,    /* 多维数组，需配合 shape_hint */
    TYPE_STRING,     /* 定长字符串 */
    TYPE_VOID        /* 无返回值 */
} DataType;

/* 参数/返回值模式定义 */
typedef struct {
    char name[MALL_MAX_NAME_LEN];
    DataType type;
    char shape_hint[MALL_MAX_NAME_LEN];  /* 例如 "(N,)" 或 "(M,N)" */
    char unit[MALL_MAX_NAME_LEN];         /* 物理单位，例如 "Hz", "s" */
    char shm_key[MALL_MAX_PATH_LEN];     /* 关联的 SHM 矢量键（可选） */
} ParamSchema;

/* 函数约束条件 */
typedef struct {
    u32 max_latency_ms;       /* 最大延迟约束 */
    u32 max_memory_mb;        /* 最大内存约束 */
    mall_bool deterministic;  /* 是否要求确定性输出 */
    mall_bool side_effect_free; /* 是否禁止副作用 */
    mall_bool oop_forbidden;  /* 是否禁止 OOP 语义 */
} FunctionConstraints;

/* FunctionSpec —— 目标函数原子定义 */
typedef struct {
    char function_name[MALL_MAX_NAME_LEN];
    char language[MALL_MAX_NAME_LEN];     /* "python", "c", "verilog" 等 */
    char signature_hash[65];              /* SHA-256 十六进制字符串 */
    ParamSchema inputs[8];                  /* 输入参数模式（最多8个） */
    u8 input_count;
    ParamSchema outputs[8];                 /* 输出参数模式（最多8个） */
    u8 output_count;
    FunctionConstraints constraints;
    char context_notes[1024];               /* 业务上下文描述 */
} FunctionSpec;
```

### 3.3 沙箱配置

```c
/* 沙箱隔离级别 */
typedef enum {
    SANDBOX_LIGHT = 0,    /* 仅语法检查，无进程隔离 */
    SANDBOX_STANDARD = 1, /* 容器级隔离（namespace + cgroups） */
    SANDBOX_HEAVY = 2     /* 硬件级虚拟化（KVM/VMX） */
} SandboxLevel;

/* 沙箱配置结构体 */
typedef struct {
    SandboxLevel level;
    char shm_read_keys[MALL_MAX_SHM_KEYS][MALL_MAX_PATH_LEN];
    u8 shm_read_count;
    char shm_write_keys[MALL_MAX_SHM_KEYS][MALL_MAX_PATH_LEN];
    u8 shm_write_count;
    mall_bool network_access;
    char compiler_path[MALL_MAX_PATH_LEN];
} SandboxConfig;
```

### 3.4 完整授权文档

```c
/* 授权文档头部 */
typedef struct {
    char doc_id[MALL_UUID_LEN];
    char producer_id[MALL_MAX_NAME_LEN];
    char mall_node_id[MALL_MAX_NAME_LEN];
    char timestamp_utc[32];   /* ISO-8601 格式，固定长度 */
    u32 ttl_seconds;
    char version[8];
} AuthHeader;

/* 数字签名区块 —— Ed25519 */
typedef struct {
    u8 signature[64];           /* Ed25519 签名原始字节 */
    char algorithm[16];         /* 签名算法标识 */
} AuthSignature;

/* 完整授权文档 —— 扁平化结构，总大小约 8KB，可整块写入 SHM */
typedef struct {
    AuthHeader header;
    PermissionVector mandate;
    FunctionSpec function_spec;
    SandboxConfig sandbox;
    AuthSignature signature;
    u32 checksum;               /* 文档完整性校验（CRC32） */
} AuthDocument;
```

---

## 四、SHM 矢量类型系统

### 4.1 SHM 矢量句柄

```c
/* SHM 矢量访问模式 */
typedef enum {
    SHM_MODE_READ = 0,
    SHM_MODE_WRITE = 1,
    SHM_MODE_READWRITE = 2
} ShmAccessMode;

/* SHM 矢量元数据 —— 所有 SHM 区域的前缀头部 */
typedef struct {
    char key[MALL_MAX_PATH_LEN];     /* 矢量键名 */
    u64 capacity;                     /* 总容量（字节） */
    u64 used;                         /* 已使用字节数 */
    u64 version;                      /* 数据版本号（乐观锁） */
    u64 owner_pid;                    /* 当前写入者 PID */
    ShmAccessMode mode;               /* 当前挂载模式 */
    u32 checksum;                     /* 数据区 CRC32 */
} ShmVectorMeta;

/* SHM 矢量句柄 —— 所有 SHM 操作的统一接口 */
typedef struct {
    ShmVectorMeta meta;
    void* data;                       /* 映射后的用户数据区指针 */
    u64 data_offset;                  /* 数据区相对于映射基址的偏移 */
} ShmVector;
```

### 4.2 序列化原语

```c
/* 序列化缓冲区 —— 用于将结构体扁平化为字节流写入 SHM */
typedef struct {
    u8* buffer;           /* 指向 SHM 数据区的指针 */
    u64 capacity;         /* 缓冲区总容量 */
    u64 offset;           /* 当前写入偏移 */
    MallError last_error; /* 最后一次序列化错误 */
} SerializeStream;

/* 序列化函数族 —— 所有操作均为纯函数，不修改外部状态 */
int serialize_u8(SerializeStream* s, u8 val);
int serialize_u32(SerializeStream* s, u32 val);
int serialize_u64(SerializeStream* s, u64 val);
int serialize_bool(SerializeStream* s, mall_bool val);
int serialize_string(SerializeStream* s, const char* str, u64 max_len);
int serialize_struct(SerializeStream* s, const void* data, u64 size);

/* 反序列化函数族 */
int deserialize_u8(const SerializeStream* s, u64* offset, u8* out);
int deserialize_u32(const SerializeStream* s, u64* offset, u32* out);
int deserialize_u64(const SerializeStream* s, u64* offset, u64* out);
int deserialize_bool(const SerializeStream* s, u64* offset, mall_bool* out);
int deserialize_string(const SerializeStream* s, u64* offset, char* out, u64 max_len);
```

---

## 五、任务与状态机类型

### 5.1 任务类型枚举

```c
typedef enum {
    TASK_DESIGN = 0,          /* 仅设计阶段 */
    TASK_DEBUG = 1,           /* 仅调试阶段 */
    TASK_EVALUATE = 2,        /* 仅评估阶段 */
    TASK_DESIGN_DEBUG = 3,    /* 设计+调试 */
    TASK_DEBUG_EVALUATE = 4,  /* 调试+评估 */
    TASK_FULL_PIPELINE = 5    /* 完整三阶段流水线 */
} AICoderTaskType;
```

### 5.2 流水线阶段枚举

```c
typedef enum {
    PHASE_IDLE = 0,
    PHASE_DESIGN = 1,
    PHASE_DEBUG = 2,
    PHASE_EVALUATE = 3,
    PHASE_REPORT = 4,
    PHASE_COMPLETED = 5,
    PHASE_ABORTED = 6,
    PHASE_EXPIRED = 7
} PipelinePhase;
```

### 5.3 任务状态结构体

```c
/* 任务状态 —— 存储于 SHM 状态矢量，供轮询使用 */
typedef struct {
    char task_token[MALL_UUID_LEN];
    PipelinePhase current_phase;
    f32 phase_progress;           /* 0.0 ~ 1.0 */
    u64 last_heartbeat_ns;       /* 纳秒级心跳时间戳 */
    u64 worker_pid;              /* 执行 Worker 的 PID */
    MallErrorCode last_error;    /* 最后记录的错误码 */
    u64 start_time_ns;
    u64 elapsed_time_ns;
} TaskState;
```

---

## 六、AICoder 产出物类型

### 6.1 设计阶段产出

```c
/* 依赖项 */
typedef struct {
    char name[MALL_MAX_NAME_LEN];
    char version[MALL_MAX_NAME_LEN];
    u8 risk_level;              /* 0=低风险 1=中风险 2=高风险 */
} Dependency;

/* 设计产出 */
typedef struct {
    char generated_code[8192];       /* 生成的函数代码（8KB 上限） */
    char design_rationale[2048];     /* 设计依据说明 */
    Dependency dependencies[MALL_MAX_DEPS];
    u8 dependency_count;
    f32 confidence_score;            /* AICoder 置信度 0.0~1.0 */
} DesignOutput;
```

### 6.2 调试阶段产出

```c
/* 补丁差异 —— unified diff 格式 */
typedef struct {
    char patch_diff[4096];           /* diff 文本 */
    char root_cause[1024];           /* 根因分析 */
    char regression_plan[1024];      /* 回归测试建议 */
    mall_bool patch_valid;           /* 补丁是否通过验证 */
} DebugOutput;
```

### 6.3 评估阶段产出

```c
/* 评分卡 */
typedef struct {
    f32 correctness;       /* 正确性得分（加权前） */
    f32 performance;       /* 性能得分 */
    f32 security;          /* 安全性得分 */
    f32 maintainability;   /* 可维护性得分 */
    f32 oop_compliance;    /* OOP 合规得分 */
    f32 total;             /* 加权总分 */
} ScoreCard;

/* 评估产出 */
typedef struct {
    ScoreCard score_card;
    char detailed_report[4096];       /* 详细评估报告 */
    char improvements[2048];          /* 改进建议列表 */
    u32 tests_total;
    u32 tests_passed;
    u32 tests_failed;
    f32 p99_latency_ms;
} EvaluateOutput;
```

---

## 七、审计追踪类型

```c
/* 单个审计事件 */
typedef struct {
    u64 timestamp_ns;
    char action[MALL_MAX_NAME_LEN];     /* 例如 "DESIGN_START" */
    PipelinePhase phase;
    char detail[MALL_MAX_ERR_MSG];
} AuditEvent;

/* 审计追踪 —— 循环缓冲区设计 */
typedef struct {
    AuditEvent events[MALL_MAX_AUDIT_EVENTS];
    u32 head;                           /* 写入位置 */
    u32 count;                          /* 当前有效事件数 */
    u64 total_events;                   /* 累计事件数（含溢出） */
} AuditTrail;
```

---

## 八、报告类型

```c
/* 商场裁定建议 */
typedef enum {
    VERDICT_REJECT = 0,           /* 拒绝 */
    VERDICT_CONDITIONAL = 1,      /* 条件接受 */
    VERDICT_ACCEPT = 2            /* 完全接受 */
} MallVerdict;

/* 最终报告结构体 —— 聚合所有阶段产出 */
typedef struct {
    char task_token[MALL_UUID_LEN];
    char producer_id[MALL_MAX_NAME_LEN];
    char function_name[MALL_MAX_NAME_LEN];
    char language[MALL_MAX_NAME_LEN];

    MallVerdict verdict;
    ScoreCard score_card;

    DesignOutput design;
    DebugOutput debug;
    EvaluateOutput evaluate;
    AuditTrail audit;

    u64 total_duration_ms;
    u64 tokens_consumed;
    f32 confidence_aggregate;
} FinalReport;
```

---

## 九、审核宣言

> 类型系统是软件的**宪法**。  
> 所有结构体均为扁平化、定长、可序列化设计，杜绝指针嵌套与动态分配。  
> 这是商场模式在数据层面的根本保障——如果数据结构允许任意嵌套，就等于允许了不可控的复杂性膨胀，等同于 OOP 的幽灵借尸还魂。  
> **审核状态**：待人类架构师最终裁定。
