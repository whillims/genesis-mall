# AICoder 执行层接口深化 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档定位**：AICoder Worker 进程内部的**函数级精确契约**。  
> **核心原则**：AICoder 是商场的"外脑"，其所有行为通过显式函数调用暴露；Worker 进程内禁止任何全局可变状态，所有中间数据通过栈分配或 SHM 传递。  
> **隔离模型**：每个 Worker 进程通过 `clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS)` 创建，SHM 访问通过精确映射列表控制。

---

## 一、Worker 生命周期函数

### 1.1 `aicoder_worker_main`

```c
/* 函数：AICoder Worker 主循环
 * 契约：
 *   前置条件：进程已通过 mall_worker_spawn() 创建；已挂载允许的 SHM 区域；
 *             已关闭所有文件描述符（除 stdin/stdout/stderr 和 SHM fd）
 *   后置条件：进程持续运行，直到收到 SIGTERM 或 mall_worker_shutdown() 调用
 *   副作用：
 *     1. 轮询 SHM: shm://mall/task_queue/aicoder_assigned（个人任务队列）
 *     2. 执行分配的任务流水线
 *     3. 定期写入心跳到 shm://aicoder/{token}/state
 *   并发安全：Worker 为单线程事件循环，无并发竞争
 *   时间复杂度：O(infinity) —— 无限循环，单次迭代 O(1)
 *   空间复杂度：O(1) —— 栈上固定缓冲区，无堆分配
 */
void aicoder_worker_main(
    u64 worker_id,                    /* [in] Worker 唯一标识 */
    const struct WorkerShmMap* shm_map  /* [in] 允许的 SHM 映射列表 */
);

/* WorkerShmMap 定义 */
typedef struct {
    char read_keys[MALL_MAX_SHM_KEYS][MALL_MAX_PATH_LEN];
    u8 read_count;
    char write_keys[MALL_MAX_SHM_KEYS][MALL_MAX_PATH_LEN];
    u8 write_count;
} WorkerShmMap;

/* 主循环伪代码：
 * while (!shutdown_signal) {
 *     // 1. 心跳
 *     aicoder_heartbeat(worker_id);
 *
 *     // 2. 任务轮询（非阻塞，超时 100ms）
 *     TaskAssignment assignment;
 *     if (mall_worker_poll_assignment(&assignment, 100) == MALL_OK) {
 *         // 3. 执行任务
 *         aicoder_execute_task(&assignment);
 *     }
 * }
 */
```

---

### 1.2 `aicoder_heartbeat`

```c
/* 函数：Worker 心跳写入
 * 契约：
 *   前置条件：worker_id 有效；当前有活跃任务或处于空闲状态
 *   后置条件：SHM 心跳时间戳已更新
 *   副作用：原子写入 SHM 状态矢量的 last_heartbeat_ns 字段
 *   并发安全：仅当前 Worker 写入自己的状态区，无竞争
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int aicoder_heartbeat(
    u64 worker_id,                    /* [in] Worker 标识 */
    const char* task_token            /* [in] 当前任务令牌（可为空） */
);
```

---

### 1.3 `aicoder_execute_task`

```c
/* 函数：任务执行主控函数
 * 契约：
 *   前置条件：assignment 指向有效的任务分配记录；所有所需 SHM 区域已挂载
 *   后置条件：任务流水线已完成（成功或失败）；产出物已写入 SHM 输出区
 *   副作用：
 *     1. 读取 SHM 输入矢量
 *     2. 根据 task_type 调用对应阶段函数
 *     3. 写入 SHM 输出矢量
 *     4. 更新 SHM 状态矢量
 *     5. 记录审计事件
 *   并发安全：单线程执行，无并发问题
 *   时间复杂度：取决于任务类型和代码规模，通常为 O(Tokens) + O(Tests)
 *   空间复杂度：O(1) —— 中间结果通过栈缓冲区，大小受 FunctionSpec 约束限制
 */
int aicoder_execute_task(
    const struct TaskAssignment* assignment, /* [in] 任务分配记录 */
    struct MallError* out_err                /* [out] 错误详情 */
);

/* TaskAssignment 定义 */
typedef struct {
    char task_token[MALL_UUID_LEN];
    AICoderTaskType task_type;
    char auth_doc_id[MALL_UUID_LEN];     /* 关联的授权文档 ID */
    FunctionSpec function_spec;          /* 目标函数规范（拷贝） */
    SandboxConfig sandbox;             /* 沙箱配置（拷贝） */
} TaskAssignment;

/* 执行流程：
 * Step 1: 状态更新 → PHASE_DESIGN
 * Step 2: 若 task_type 包含 DESIGN:
 *         aicoder_execute_design(shm_in, shm_out, &metrics)
 *         若失败 → 记录审计 → 状态更新 → PHASE_ABORTED → 返回
 * Step 3: 状态更新 → PHASE_DEBUG
 * Step 4: 若 task_type 包含 DEBUG:
 *         aicoder_execute_debug(shm_in, shm_out, &metrics)
 *         若失败 → 记录审计 → 继续（调试失败不终止流水线，仅记录）
 * Step 5: 状态更新 → PHASE_EVALUATE
 * Step 6: 若 task_type 包含 EVALUATE:
 *         aicoder_execute_evaluate(shm_in, shm_out, &metrics)
 *         若失败 → 记录审计 → 状态更新 → PHASE_ABORTED → 返回
 * Step 7: 状态更新 → PHASE_REPORT
 * Step 8: 调用 mall_generate_aicoder_report()（通过 SHM 回调机制）
 * Step 9: 状态更新 → PHASE_COMPLETED
 */
```

---

## 二、设计阶段函数族

### 2.1 `aicoder_execute_design`

```c
/* 函数：设计阶段执行函数
 * 契约：
 *   前置条件：shm_in 已挂载且包含有效的 DesignInput；shm_out 已挂载且可写；
 *             function_spec.constraints.oop_forbidden == MALL_TRUE
 *   后置条件：shm_out 包含 DesignOutput；metrics 包含资源消耗统计
 *   副作用：
 *     1. 读取 SHM 输入矢量
 *     2. 写入 SHM 输出矢量（设计产出）
 *     3. 写入审计事件：DESIGN_START, DESIGN_COMPLETE 或 DESIGN_FAIL
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens) —— 取决于 LLM 生成耗时
 *   空间复杂度：O(1) —— 输出受 8KB 代码缓冲区限制
 */
int aicoder_execute_design(
    const ShmVector* shm_in,            /* [in] 输入矢量 */
    ShmVector* shm_out,               /* [out] 输出矢量 */
    struct AICoderMetrics* metrics     /* [out] 资源消耗统计 */
);

/* AICoderMetrics 定义 */
typedef struct {
    u64 tokens_consumed;               /* LLM Token 消耗量 */
    u64 execution_time_ms;             /* 执行耗时 */
    f32 confidence_score;              /* AICoder 置信度 */
    u64 memory_peak_mb;               /* 内存峰值 */
} AICoderMetrics;

/* 内部子函数调用链：
 * aicoder_execute_design()
 * ├── aicoder_parse_requirements()      [需求解析]
 * ├── aicoder_select_algorithm()        [算法选择]
 * ├── aicoder_generate_pseudocode()     [伪代码生成]
 * ├── aicoder_generate_implementation() [代码生成]
 * ├── aicoder_oop_scan_internal()       [内部 OOP 扫描]
 * ├── aicoder_constraint_check()        [约束检查]
 * └── aicoder_format_output()           [格式化输出]
 */
```

---

### 2.2 `aicoder_parse_requirements`

```c
/* 函数：需求解析
 * 契约：
 *   前置条件：input 包含有效的 function_spec 和 context_notes
 *   后置条件：draft 包含解析后的算法需求描述
 *   副作用：无 SHM 写入
 *   并发安全：纯函数
 *   时间复杂度：O(L)，L 为 context_notes 长度
 *   空间复杂度：O(1) —— 输出受 AlgorithmDraft 固定大小限制
 */
int aicoder_parse_requirements(
    const struct DesignInput* input,   /* [in] 设计输入 */
    struct AlgorithmDraft* draft,        /* [out] 算法需求草稿 */
    struct MallError* out_err            /* [out] 错误详情 */
);

/* AlgorithmDraft 定义 */
typedef struct {
    char requirement_summary[512];     /* 需求摘要 */
    char algorithm_hints[512];           /* 算法偏好提示 */
    char constraints_summary[512];       /* 约束条件摘要 */
    DataType primary_input_type;         /* 主输入数据类型 */
    DataType primary_output_type;        /* 主输出数据类型 */
    u32 max_latency_ms;                 /* 延迟约束 */
    u32 max_memory_mb;                  /* 内存约束 */
    mall_bool deterministic;            /* 确定性要求 */
    mall_bool oop_forbidden;            /* OOP 禁止标志 */
} AlgorithmDraft;
```

---

### 2.3 `aicoder_select_algorithm`

```c
/* 函数：算法选择
 * 契约：
 *   前置条件：draft 已通过需求解析；constraints 有效
 *   后置条件：draft->algorithm_hints 已填充推荐算法列表
 *   副作用：无 SHM 写入
 *   并发安全：纯函数（基于规则匹配，无外部状态）
 *   时间复杂度：O(R)，R 为规则库大小（固定，约 100 条规则）
 *   空间复杂度：O(1)
 */
int aicoder_select_algorithm(
    struct AlgorithmDraft* draft,        /* [in,out] 算法草稿（填充推荐） */
    const FunctionConstraints* constraints, /* [in] 约束条件 */
    struct MallError* out_err            /* [out] 错误详情 */
);

/* 算法选择规则示例（内部知识库）：
 * IF primary_input_type == TYPE_NDARRAY AND context 包含 "频谱"
 *    THEN recommend "APFFT", "STFT", "Welch"
 * IF max_latency_ms < 10 AND language == "c"
 *    THEN recommend "SIMD", "LUT", "固定点"
 * IF oop_forbidden == MALL_TRUE
 *    THEN exclude 任何需要对象状态的设计模式
 */
```

---

### 2.4 `aicoder_generate_pseudocode`

```c
/* 函数：伪代码生成
 * 契约：
 *   前置条件：draft 包含算法选择结果
 *   后置条件：code 包含结构化伪代码（文本形式）
 *   副作用：调用 LLM 推理接口（外部网络已隔离，使用本地模型）
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens)
 *   空间复杂度：O(1) —— 输出受 GeneratedCode 固定大小限制
 */
int aicoder_generate_pseudocode(
    const struct AlgorithmDraft* draft, /* [in] 算法草稿 */
    struct GeneratedCode* code,          /* [out] 生成的代码结构 */
    struct AICoderMetrics* metrics,      /* [out] 资源统计 */
    struct MallError* out_err            /* [out] 错误详情 */
);

/* GeneratedCode 定义 */
typedef struct {
    char pseudocode[2048];              /* 伪代码文本 */
    char implementation[8192];          /* 最终实现代码 */
    char language[MALL_MAX_NAME_LEN];  /* 目标语言 */
    u32 line_count;                     /* 代码行数 */
    u32 token_count;                    /* 生成 Token 数 */
} GeneratedCode;
```

---

### 2.5 `aicoder_generate_implementation`

```c
/* 函数：最终实现代码生成
 * 契约：
 *   前置条件：code 包含伪代码；language 为支持的语言
 *   后置条件：code->implementation 包含可编译的函数实现
 *   副作用：调用 LLM 推理接口
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens)
 *   空间复杂度：O(1)
 */
int aicoder_generate_implementation(
    struct GeneratedCode* code,        /* [in,out] 代码结构 */
    const char* language,              /* [in] 目标语言 */
    struct AICoderMetrics* metrics,    /* [out] 资源统计 */
    struct MallError* out_err          /* [out] 错误详情 */
);
```

---

### 2.6 `aicoder_oop_scan_internal`

```c
/* 函数：内部 OOP 扫描（AICoder 自检）
 * 契约：
 *   前置条件：code->implementation 非空
 *   后置条件：若发现 OOP 语义，返回 AICODER_ERR_OOP_DETECTED；否则 MALL_OK
 *   副作用：无 SHM 写入
 *   并发安全：纯函数
 *   时间复杂度：O(L)，L 为代码长度
 *   空间复杂度：O(L) —— 语法树栈空间
 */
int aicoder_oop_scan_internal(
    const struct GeneratedCode* code,   /* [in] 生成的代码 */
    struct OopViolation* out_violations, /* [out] 违规列表 */
    u64* out_violation_count,           /* [out] 违规数量 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 扫描规则与 mall_oop_scanner() 一致，但此函数在 AICoder Worker 内部执行，
 * 作为自检机制。若自检失败，Worker 直接返回错误，不将污染代码提交给商场。
 * 这是"第一道防线"，mall_oop_scanner() 是"第二道防线"。
 */
```

---

### 2.7 `aicoder_constraint_check`

```c
/* 函数：约束条件检查
 * 契约：
 *   前置条件：code 包含实现代码；constraints 来自 FunctionSpec
 *   后置条件：验证代码是否满足所有约束条件
 *   副作用：无 SHM 写入
 *   并发安全：纯函数
 *   时间复杂度：O(L) + O(1) —— 代码扫描 + 约束比对
 *   空间复杂度：O(1)
 */
int aicoder_constraint_check(
    const struct GeneratedCode* code,   /* [in] 代码 */
    const FunctionConstraints* constraints, /* [in] 约束 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 检查项：
 * - 函数签名是否与 FunctionSpec 一致（参数名、类型、顺序）
 * - 是否包含复杂度注释（时间/空间复杂度）
 * - 代码长度是否在合理范围内（< 8KB）
 * - 是否使用了禁止的依赖（根据 perm_dependency）
 * - 是否包含全局变量（side_effect_free 约束时）
 */
```

---

## 三、调试阶段函数族

### 3.1 `aicoder_execute_debug`

```c
/* 函数：调试阶段执行函数
 * 契约：
 *   前置条件：shm_in 包含 buggy_code + error_trace + test_cases；
 *             若流水线模式，则 buggy_code 为上一阶段生成的代码
 *   后置条件：shm_out 包含 DebugOutput（补丁、根因、回归计划）
 *   副作用：
 *     1. 读取 SHM 输入矢量
 *     2. 写入 SHM 输出矢量
 *     3. 记录审计事件
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens) + O(Tests) —— LLM 分析 + 测试执行
 *   空间复杂度：O(1)
 */
int aicoder_execute_debug(
    const ShmVector* shm_in,            /* [in] 输入矢量 */
    ShmVector* shm_out,               /* [out] 输出矢量 */
    struct AICoderMetrics* metrics     /* [out] 资源统计 */
);

/* 内部子函数调用链：
 * aicoder_execute_debug()
 * ├── aicoder_parse_error_trace()      [错误解析]
 * ├── aicoder_static_analyze()          [静态分析]
 * ├── aicoder_locate_root_cause()      [根因定位]
 * ├── aicoder_generate_patch()          [补丁生成]
 * └── aicoder_verify_patch()            [补丁验证]
 */
```

---

### 3.2 `aicoder_parse_error_trace`

```c
/* 函数：错误追踪解析
 * 契约：
 *   前置条件：error_trace 为以 null 终止的字符串（来自测试框架或运行时）
 *   后置条件：profile 包含结构化的错误信息（类型、位置、堆栈）
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(L)
 *   空间复杂度：O(1)
 */
int aicoder_parse_error_trace(
    const char* error_trace,            /* [in] 原始错误追踪文本 */
    struct ErrorProfile* profile,       /* [out] 解析后的错误画像 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* ErrorProfile 定义 */
typedef struct {
    char error_type[MALL_MAX_NAME_LEN]; /* 错误类型：SyntaxError, RuntimeError, AssertionError 等 */
    u64 line_number;                    /* 错误发生行号 */
    char file_path[MALL_MAX_PATH_LEN];  /* 错误文件路径 */
    char stack_summary[1024];             /* 堆栈摘要 */
    char exception_message[512];        /* 异常消息 */
} ErrorProfile;
```

---

### 3.3 `aicoder_static_analyze`

```c
/* 函数：代码静态分析
 * 契约：
 *   前置条件：buggy_code 为有效源代码；profile 包含错误画像
 *   后置条件：analysis 包含静态分析结果（潜在缺陷、代码异味、复杂度）
 *   副作用：无 SHM 写入
 *   并发安全：纯函数
 *   时间复杂度：O(L) —— 语法树遍历
 *   空间复杂度：O(L)
 */
int aicoder_static_analyze(
    const char* buggy_code,             /* [in] 缺陷代码 */
    const struct ErrorProfile* profile,  /* [in] 错误画像 */
    struct StaticAnalysis* analysis,     /* [out] 分析结果 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* StaticAnalysis 定义 */
typedef struct {
    u32 cyclomatic_complexity;          /* 圈复杂度 */
    u32 function_length;               /* 函数长度（行） */
    u32 comment_coverage;              /* 注释覆盖率（百分比） */
    char potential_bugs[8][128];       /* 潜在缺陷列表（最多8条） */
    u8 potential_bug_count;
    char code_smells[8][128];          /* 代码异味列表 */
    u8 code_smell_count;
} StaticAnalysis;
```

---

### 3.4 `aicoder_locate_root_cause`

```c
/* 函数：根因定位
 * 契约：
 *   前置条件：buggy_code、profile、analysis 均有效
 *   后置条件：cause 包含根因描述和代码位置
 *   副作用：调用 LLM 推理（本地模型）
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens)
 *   空间复杂度：O(1)
 */
int aicoder_locate_root_cause(
    const char* buggy_code,             /* [in] 缺陷代码 */
    const struct ErrorProfile* profile,  /* [in] 错误画像 */
    const struct StaticAnalysis* analysis, /* [in] 静态分析结果 */
    struct RootCause* cause,             /* [out] 根因 */
    struct AICoderMetrics* metrics,      /* [out] 资源统计 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* RootCause 定义 */
typedef struct {
    char description[1024];             /* 根因自然语言描述 */
    u64 line_number;                    /* 根因代码行号 */
    char code_snippet[256];             /* 相关代码片段 */
    char category[MALL_MAX_NAME_LEN];   /* 根因类别：LogicError, BoundaryError, TypeError 等 */
    f32 confidence;                     /* 根因置信度 */
} RootCause;
```

---

### 3.5 `aicoder_generate_patch`

```c
/* 函数：补丁生成
 * 契约：
 *   前置条件：buggy_code 和 cause 均有效
 *   后置条件：patch 包含 unified diff 格式的补丁
 *   副作用：调用 LLM 推理
 *   并发安全：单线程执行
 *   时间复杂度：O(Tokens)
 *   空间复杂度：O(1) —— 补丁受 4KB 限制
 */
int aicoder_generate_patch(
    const char* buggy_code,             /* [in] 缺陷代码 */
    const struct RootCause* cause,      /* [in] 根因 */
    struct Patch* patch,                /* [out] 补丁 */
    struct AICoderMetrics* metrics,      /* [out] 资源统计 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* Patch 定义 */
typedef struct {
    char unified_diff[4096];            /* unified diff 文本 */
    u32 added_lines;                    /* 新增行数 */
    u32 removed_lines;                  /* 删除行数 */
    char target_file[MALL_MAX_PATH_LEN]; /* 目标文件路径 */
} Patch;
```

---

### 3.6 `aicoder_verify_patch`

```c
/* 函数：补丁验证
 * 契约：
 *   前置条件：patch 包含有效 diff；test_cases 包含回归测试用例
 *   后置条件：验证补丁应用后是否通过所有测试
 *   副作用：
 *     1. 在临时 SHM 区域创建代码副本
 *     2. 应用补丁到副本
 *     3. 执行测试用例
 *     4. 清理临时区域
 *   并发安全：单线程执行
 *   时间复杂度：O(Tests) —— 测试执行时间
 *   空间复杂度：O(1) —— 临时区域大小固定
 */
int aicoder_verify_patch(
    const struct Patch* patch,          /* [in] 补丁 */
    const struct TestCase* test_cases,  /* [in] 测试用例数组 */
    u64 test_case_count,                /* [in] 用例数量 */
    mall_bool* out_valid,               /* [out] 补丁是否有效 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 验证流程：
 * 1. 分配临时 SHM: shm://aicoder/{token}/tmp/patch_verify
 * 2. 将 buggy_code 写入临时区
 * 3. 应用 unified_diff 到临时区（使用内部 diff 应用器）
 * 4. 编译临时代码（使用 sandbox.compiler_path 指定的编译器）
 * 5. 逐条执行 test_cases
 * 6. 统计通过率
 * 7. 若通过率 == 100%，*out_valid = MALL_TRUE
 * 8. 删除临时 SHM 区域
 */
```

---

## 四、评估阶段函数族

### 4.1 `aicoder_execute_evaluate`

```c
/* 函数：评估阶段执行函数
 * 契约：
 *   前置条件：shm_in 包含 code_under_test + test_suite + eval_config；
 *             code_under_test 已通过 OOP 扫描
 *   后置条件：shm_out 包含 EvaluateOutput（评分卡、详细报告、改进建议）
 *   副作用：
 *     1. 读取 SHM 输入矢量
 *     2. 写入 SHM 输出矢量
 *     3. 可能创建临时编译产物（在 Worker 临时目录，非 SHM）
 *     4. 记录审计事件
 *   并发安全：单线程执行
 *   时间复杂度：O(Tests * TestComplexity) + O(L) —— 测试执行 + 静态扫描
 *   空间复杂度：O(1) —— 输出受固定大小结构体限制
 */
int aicoder_execute_evaluate(
    const ShmVector* shm_in,            /* [in] 输入矢量 */
    ShmVector* shm_out,               /* [out] 输出矢量 */
    struct AICoderMetrics* metrics     /* [out] 资源统计 */
);

/* 内部子函数调用链：
 * aicoder_execute_evaluate()
 * ├── aicoder_eval_compile()            [语法编译]
 * ├── aicoder_eval_unit_tests()         [单元测试]
 * ├── aicoder_eval_perf_benchmark()     [性能基准]
 * ├── aicoder_eval_security_scan()      [安全扫描]
 * ├── aicoder_eval_complexity()         [复杂度分析]
 * ├── aicoder_eval_oop_final()          [最终 OOP 扫描]
 * ├── aicoder_eval_score()              [评分汇总]
 * └── aicoder_eval_format_report()      [报告格式化]
 */
```

---

### 4.2 `aicoder_eval_compile`

```c
/* 函数：语法编译验证
 * 契约：
 *   前置条件：code 为有效源代码；language 为支持的语言
 *   后置条件：返回编译结果（成功/失败及错误信息）
 *   副作用：在 Worker 临时目录生成编译产物，执行后自动清理
 *   并发安全：单线程执行
 *   时间复杂度：O(1) —— 编译器调用
 *   空间复杂度：O(1)
 */
int aicoder_eval_compile(
    const char* code,                  /* [in] 源代码 */
    const char* language,              /* [in] 语言 */
    const char* compiler_path,         /* [in] 编译器路径 */
    struct CompileResult* out_result,  /* [out] 编译结果 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* CompileResult 定义 */
typedef struct {
    mall_bool success;                 /* 编译是否成功 */
    char error_output[1024];           /* 编译器错误输出 */
    u32 warning_count;                 /* 警告数量 */
    u32 error_count;                   /* 错误数量 */
} CompileResult;
```

---

### 4.3 `aicoder_eval_unit_tests`

```c
/* 函数：单元测试执行
 * 契约：
 *   前置条件：code 已编译通过；test_suite 包含有效测试用例
 *   后置条件：返回测试结果统计
 *   副作用：在隔离环境中执行测试（使用沙箱限制资源）
 *   并发安全：单线程执行
 *   时间复杂度：O(Tests * TestComplexity)
 *   空间复杂度：O(1)
 */
int aicoder_eval_unit_tests(
    const char* compiled_artifact,     /* [in] 编译产物路径 */
    const struct TestSuite* test_suite, /* [in] 测试套件 */
    struct TestResult* out_result,     /* [out] 测试结果 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* TestSuite / TestResult 定义 */
typedef struct {
    char name[MALL_MAX_NAME_LEN];
    char input[512];                   /* 测试输入（序列化） */
    char expected_output[512];         /* 期望输出（序列化） */
    mall_bool is_boundary;             /* 是否为边界测试 */
} TestCase;

typedef struct {
    TestCase cases[MALL_MAX_TEST_CASES];
    u64 case_count;
} TestSuite;

typedef struct {
    u64 total;                         /* 总用例数 */
    u64 passed;                        /* 通过数 */
    u64 failed;                        /* 失败数 */
    f32 pass_rate;                     /* 通过率 */
    char failure_details[8][256];      /* 失败详情（最多8条） */
    u64 failure_count;
} TestResult;
```

---

### 4.4 `aicoder_eval_perf_benchmark`

```c
/* 函数：性能基准测试
 * 契约：
 *   前置条件：code 已编译通过；benchmark_config 包含测试参数
 *   后置条件：返回性能指标（延迟、吞吐量、内存）
 *   副作用：执行多次迭代，消耗 CPU 时间
 *   并发安全：单线程执行
 *   时间复杂度：O(Iterations * FunctionComplexity)
 *   空间复杂度：O(1)
 */
int aicoder_eval_perf_benchmark(
    const char* compiled_artifact,     /* [in] 编译产物路径 */
    const struct BenchmarkConfig* config, /* [in] 基准配置 */
    struct PerfResult* out_result,     /* [out] 性能结果 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* BenchmarkConfig / PerfResult 定义 */
typedef struct {
    u64 iterations;                    /* 迭代次数 */
    u32 warmup_iterations;             /* 预热次数 */
    char input_dataset[MALL_MAX_PATH_LEN]; /* 输入数据集 SHM 路径 */
} BenchmarkConfig;

typedef struct {
    f32 mean_latency_ms;               /* 平均延迟 */
    f32 p50_latency_ms;                /* P50 */
    f32 p99_latency_ms;                /* P99 */
    f32 max_latency_ms;                /* 最大延迟 */
    f32 throughput_ops_per_sec;        /* 吞吐量 */
    u64 peak_memory_mb;                /* 峰值内存 */
    f32 baseline_ratio;                /* 与基准的比值 */
} PerfResult;
```

---

### 4.5 `aicoder_eval_security_scan`

```c
/* 函数：安全静态扫描
 * 契约：
 *   前置条件：code 为有效源代码
 *   后置条件：返回安全扫描结果（漏洞、风险函数、注入点）
 *   副作用：无 SHM 写入
 *   并发安全：纯函数
 *   时间复杂度：O(L)
 *   空间复杂度：O(1)
 */
int aicoder_eval_security_scan(
    const char* code,                  /* [in] 源代码 */
    const char* language,              /* [in] 语言 */
    struct SecurityScan* out_result,     /* [out] 扫描结果 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* SecurityScan 定义 */
typedef struct {
    f32 score;                         /* 安全得分 0.0~100.0 */
    u32 high_risk_count;              /* 高危问题数 */
    u32 medium_risk_count;            /* 中危问题数 */
    u32 low_risk_count;               /* 低危问题数 */
    char high_risk_issues[4][256];    /* 高危问题详情（最多4条） */
    char medium_risk_issues[8][256];  /* 中危问题详情 */
} SecurityScan;
```

---

### 4.6 `aicoder_eval_complexity`

```c
/* 函数：复杂度分析
 * 契约：
 *   前置条件：code 为有效源代码
 *   后置条件：返回圈复杂度、Halstead 指标、可维护性指数
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(L)
 *   空间复杂度：O(L)
 */
int aicoder_eval_complexity(
    const char* code,                  /* [in] 源代码 */
    const char* language,              /* [in] 语言 */
    struct ComplexityMetrics* out_result, /* [out] 复杂度指标 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* ComplexityMetrics 定义 */
typedef struct {
    u32 cyclomatic;                    /* 圈复杂度 */
    u32 lines_of_code;                 /* 代码行数 */
    u32 comment_lines;                 /* 注释行数 */
    f32 comment_ratio;                 /* 注释覆盖率 */
    u32 function_count;                /* 函数数量（应为1） */
    f32 halstead_volume;               /* Halstead 体积 */
    f32 maintainability_index;         /* 可维护性指数 */
} ComplexityMetrics;
```

---

### 4.7 `aicoder_eval_score`

```c
/* 函数：评分汇总计算
 * 契约：
 *   前置条件：所有子评估结果均有效
 *   后置条件：返回加权评分卡
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
ScoreCard aicoder_eval_score(
    const struct TestResult* tests,    /* [in] 测试结果 */
    const struct PerfResult* perf,     /* [in] 性能结果 */
    const struct SecurityScan* sec,    /* [in] 安全扫描 */
    const struct ComplexityMetrics* comp, /* [in] 复杂度 */
    mall_bool oop_clean                /* [in] OOP 扫描是否通过 */
);

/* 评分算法（纯函数，无副作用）：
 * correctness = tests->pass_rate * 40.0
 * performance = (1.0 - clamp((perf->p99_latency_ms - baseline) / baseline, 0, 1)) * 25.0
 * security = sec->score * 0.15
 * maintainability = clamp(comp->maintainability_index / 100.0, 0, 1) * 10.0
 * oop_compliance = oop_clean ? 10.0 : 0.0
 * total = sum of above
 *
 * 其中 baseline 来自 FunctionSpec.constraints.max_latency_ms
 */
```

---

## 五、审核宣言

> AICoder 执行层是商场的**认知末梢**——它接收商场的指令，在隔离的沙箱中执行设计、调试、评估，然后将产出物通过 SHM 矢量回传。  
> Worker 进程不是"服务"，而是**一次性的认知容器**——它启动、执行、产出、消亡，不留下任何状态痕迹。  
> 每个子函数都是纯函数或受控副作用的精确边界，确保 AICoder 的不可控性被严格限制在商场的血管网络中。  
> **审核状态**：待人类架构师最终裁定。
