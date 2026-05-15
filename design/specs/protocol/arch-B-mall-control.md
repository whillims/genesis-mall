# 商场主控层接口深化 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档定位**：商场主控层（Mall Main）的**函数级精确契约**。  
> **核心原则**：每个函数都是纯计算与显式副作用的边界；副作用仅限于 SHM 矢量写入和日志记录。  
> **并发模型**：商场主控层为单线程事件循环（epoll/kqueue），通过 SHM 原子操作与多 Worker 进程交互，无共享内存锁。

---

## 一、接口总览

```
mall_main()                          [入口，无返回]
├── mall_init()                      [初始化]
├── mall_auth_verify()               [验签]
├── mall_auth_parse_mandate()        [权限解析]
├── mall_aicoder_dispatch()          [任务分发]
│   ├── mall_token_generate()        [令牌生成]
│   └── mall_task_enqueue()          [任务入队]
├── mall_poll_producer_channel()     [生产者通道轮询]
├── mall_task_state_update()         [状态更新]
├── mall_pipeline_stage_gate()       [阶段门禁]
├── mall_pipeline_rollback()         [回滚]
├── mall_oop_scanner()               [OOP扫描]
├── mall_report_generate()           [报告生成]
└── mall_notify_producer()           [生产者通知]
```

---

## 二、商场初始化函数

### 2.1 `mall_init`

```c
/* 函数：商场全局初始化
 * 契约：
 *   前置条件：进程以 root 或 mall 专用用户启动；/dev/shm 已挂载 tmpfs
 *   后置条件：SHM 根区已分配；算力进程池已预创建；状态机处于 IDLE
 *   副作用：创建 SHM 区域 shm://mall/global_state；创建 N 个 Worker 进程
 *   并发安全：必须在任何其他 mall_* 函数之前调用，且仅调用一次
 *   时间复杂度：O(N)，N 为 Worker 数量
 *   空间复杂度：O(M)，M 为 SHM 总预分配大小
 */
int mall_init(
    const struct MallConfig* config,    /* [in] 商场配置参数 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* MallConfig 结构体定义 */
typedef struct {
    u32 worker_pool_size;              /* 算力池 Worker 数量 */
    u64 shm_total_size;                /* SHM 总预分配大小（字节） */
    char compiler_path[MALL_MAX_PATH_LEN]; /* 受控编译器路径 */
    u32 default_ttl_seconds;           /* 默认任务 TTL */
    u32 max_concurrent_tasks;          /* 最大并发任务数 */
    mall_bool enable_oop_scan;        /* 是否启用 OOP 扫描 */
} MallConfig;
```

**返回值**：
- `MALL_OK` — 初始化成功
- `MALL_ERR_SHM_ALLOC_FAIL` — SHM 分配失败
- `MALL_ERR_UNKNOWN` — 其他不可恢复错误

**副作用清单**：
1. 创建 SHM 区域 `shm://mall/global_state`，大小为 `sizeof(MallGlobalState)`
2. 创建 SHM 区域 `shm://mall/task_queue/aicoder_pending`，环形缓冲区
3. 创建 SHM 区域 `shm://mall/auth_state/registry`，生产者注册表
4. fork() N 个 Worker 进程，每个进入 `mall_worker_main()` 等待循环
5. 写入系统日志：`mall_log(MALL_LOG_INFO, "mall_init completed, workers=%u", config->worker_pool_size)`

---

## 三、授权验证函数族

### 3.1 `mall_auth_verify`

```c
/* 函数：授权文档数字签名验证
 * 契约：
 *   前置条件：auth_doc 指向有效的 AuthDocument 结构体；签名算法字段为 "ed25519"
 *   后置条件：若返回 MALL_OK，则签名验证通过，文档完整性得到保证
 *   副作用：无 SHM 写入；仅 CPU 计算
 *   并发安全：纯函数，线程安全，可并发执行
 *   时间复杂度：O(1) —— Ed25519 验签为固定时间操作
 *   空间复杂度：O(1) —— 栈上临时缓冲区 < 1KB
 */
int mall_auth_verify(
    const AuthDocument* auth_doc,       /* [in] 授权文档 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 内部实现路径：
 * 1. 计算 Header + Mandate + FunctionSpec + Sandbox 的 SHA-256 摘要
 * 2. 使用商场公钥（硬编码于 /opt/mall/keys/mall_pubkey.ed25519）验证 signature
 * 3. 比对文档 checksum 字段与 CRC32(文档整体)
 * 4. 检查 timestamp_utc 是否在合理范围内（±5分钟）
 * 5. 检查 ttl_seconds > 0
 */
```

**返回值**：
- `MALL_OK` — 验签通过
- `MALL_ERR_AUTH_FAIL` — 签名无效或文档篡改
- `AICODER_ERR_SIGNATURE_MISMATCH` — 函数签名哈希不匹配（内部哈希校验失败）

---

### 3.2 `mall_auth_parse_mandate`

```c
/* 函数：权限矢量解析与校验
 * 契约：
 *   前置条件：auth_doc 已通过 mall_auth_verify；mandate 字段已填充
 *   后置条件：返回填充后的 PermissionVector；校验各维度值域 [0,3]
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int mall_auth_parse_mandate(
    const AuthDocument* auth_doc,       /* [in] 已验签的授权文档 */
    PermissionVector* out_vector,        /* [out] 解析后的权限矢量 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 校验规则：
 * - 任何维度值 > 3 → MALL_ERR_AUTH_FAIL（文档格式损坏）
 * - 所有维度均为 0 → MALL_ERR_PERM_DENIED（空授权）
 * - perm_design == PERM_FORBIDDEN 但 task_type 包含 DESIGN → MALL_ERR_PERM_DENIED
 */
```

---

## 四、任务分发函数

### 4.1 `mall_aicoder_dispatch`

```c
/* 函数：AICoder 任务分发主控函数
 * 契约：
 *   前置条件：mall_init 已调用；auth_doc_blob 指向有效的序列化授权文档；
 *             task_type 与权限矢量兼容；token_buf_size >= MALL_UUID_LEN
 *   后置条件：若返回 MALL_OK，则任务已注入算力池，out_task_token 包含有效令牌
 *   副作用：
 *     1. 写入 SHM: shm://mall/task_queue/aicoder_pending（追加任务记录）
 *     2. 写入 SHM: shm://mall/auth_state/{doc_id}（授权状态 → TOKEN_ISSUED）
 *     3. 写入 SHM: shm://aicoder/{task_token}/state（任务状态 → PHASE_IDLE）
 *     4. 系统日志记录任务创建事件
 *   并发安全：内部使用 SHM 原子比较交换（CAS）操作入队，无需显式锁
 *   时间复杂度：O(1) 均摊
 *   空间复杂度：O(1) —— 不分配动态内存
 */
int mall_aicoder_dispatch(
    const u8* auth_doc_blob,            /* [in] 授权文档序列化字节流 */
    u64 auth_doc_len,                   /* [in] 文档长度（字节） */
    AICoderTaskType task_type,          /* [in] 任务类型枚举 */
    char* out_task_token,               /* [out] 任务令牌缓冲区 */
    u64 token_buf_size,                 /* [in] 缓冲区大小 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 执行流程（严格顺序）：
 * Step 1: 反序列化 auth_doc_blob → AuthDocument 结构体
 *         调用方: deserialize_struct()
 *         失败 → MALL_ERR_SHM_SERIALIZE_FAIL
 *
 * Step 2: 调用 mall_auth_verify(auth_doc, &err)
 *         失败 → 直接返回 err.code
 *
 * Step 3: 调用 mall_auth_parse_mandate(auth_doc, &perm_vector, &err)
 *         失败 → 直接返回 err.code
 *
 * Step 4: 权限兼容性检查
 *         函数: mall_task_type_check_permission(task_type, perm_vector)
 *         规则映射:
 *           TASK_DESIGN       → 要求 perm_design >= PERM_GENERATIVE
 *           TASK_DEBUG        → 要求 perm_debug >= PERM_GENERATIVE
 *           TASK_EVALUATE     → 要求 perm_evaluate >= PERM_GENERATIVE
 *           TASK_DESIGN_DEBUG → 要求 perm_design >= PERM_GENERATIVE && perm_debug >= PERM_ADVISORY
 *           TASK_FULL_PIPELINE→ 要求 perm_design >= PERM_GENERATIVE && perm_debug >= PERM_ADVISORY && perm_evaluate >= PERM_GENERATIVE
 *         失败 → MALL_ERR_PERM_DENIED
 *
 * Step 5: 生成任务令牌
 *         函数: mall_token_generate(out_task_token, token_buf_size)
 *         算法: UUIDv4，使用 /dev/urandom 熵源
 *         格式: "aic-" + 8位随机十六进制 + "-" + 4位 + "-" + 4位 + "-" + 12位
 *
 * Step 6: 任务入队
 *         函数: mall_task_enqueue(task_token, task_type, auth_doc)
 *         写入 SHM: shm://mall/task_queue/aicoder_pending
 *         队列满 → MALL_ERR_POOL_FULL
 *
 * Step 7: 状态初始化
 *         函数: mall_task_state_init(task_token, auth_doc->header.doc_id)
 *         写入 SHM: shm://aicoder/{task_token}/state
 *         初始状态: PHASE_IDLE, progress=0.0, start_time_ns=now(), worker_pid=0
 *
 * Step 8: 授权状态更新
 *         函数: mall_auth_state_update(auth_doc->header.doc_id, TOKEN_ISSUED)
 *         写入 SHM: shm://mall/auth_state/{doc_id}
 *
 * Step 9: 日志记录
 *         mall_log(MALL_LOG_INFO, "DISPATCH task=%s producer=%s type=%d",
 *                  task_token, auth_doc->header.producer_id, task_type)
 */
```

**返回值**：
- `MALL_OK` — 任务成功分发
- `MALL_ERR_AUTH_FAIL` — 验签失败
- `MALL_ERR_PERM_DENIED` — 权限不足
- `MALL_ERR_POOL_FULL` — 算力池满载
- `MALL_ERR_SHM_SERIALIZE_FAIL` — 文档反序列化失败

---

### 4.2 `mall_token_generate`

```c
/* 函数：任务令牌生成
 * 契约：
 *   前置条件：buf_size >= MALL_UUID_LEN (37)
 *   后置条件：buf 包含以 null 终止的 ASCII UUID 字符串
 *   副作用：读取 /dev/urandom（系统调用）
 *   并发安全：线程安全（urandom 为内核级线程安全）
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int mall_token_generate(
    char* buf,                          /* [out] 令牌缓冲区 */
    u64 buf_size                        /* [in] 缓冲区大小 */
);
```

---

### 4.3 `mall_task_enqueue`

```c
/* 函数：任务入队（SHM 环形缓冲区）
 * 契约：
 *   前置条件：mall_init 已调用；token 为有效 UUID 字符串
 *   后置条件：任务记录已追加到 pending 队列尾部
 *   副作用：原子写入 SHM: shm://mall/task_queue/aicoder_pending
 *   并发安全：使用 CAS 操作更新队列尾指针，无锁并发
 *   时间复杂度：O(1) 均摊
 *   空间复杂度：O(1)
 */
int mall_task_enqueue(
    const char* task_token,             /* [in] 任务令牌 */
    AICoderTaskType task_type,          /* [in] 任务类型 */
    const AuthDocument* auth_doc,       /* [in] 授权文档 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 队列结构（SHM 布局）：
 * struct TaskQueue {
 *     u64 head;          // 读取索引（调度器消费）
 *     u64 tail;          // 写入索引（dispatch 生产）
 *     u64 capacity;      // 队列容量
 *     u64 version;       // 全局版本号（乐观锁）
 *     TaskRecord records[capacity]; // 固定大小数组
 * }
 *
 * 入队算法（伪代码）：
 * do {
 *     old_tail = atomic_load(&queue->tail);
 *     new_tail = (old_tail + 1) % queue->capacity;
 *     if (new_tail == atomic_load(&queue->head)) return MALL_ERR_POOL_FULL;
 *     // 预写记录
 *     queue->records[old_tail] = record;
 * } while (!atomic_compare_exchange_weak(&queue->tail, &old_tail, new_tail));
 */
```

---

## 五、状态机管理函数

### 5.1 `mall_task_state_update`

```c
/* 函数：任务状态原子更新
 * 契约：
 *   前置条件：task_token 对应任务已存在；new_phase 为合法状态转换目标
 *   后置条件：SHM 状态矢量已更新；旧状态被覆盖
 *   副作用：原子写入 SHM: shm://aicoder/{task_token}/state
 *   并发安全：使用 SHM 版本号乐观锁；更新失败时自动重试
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int mall_task_state_update(
    const char* task_token,             /* [in] 任务令牌 */
    PipelinePhase new_phase,            /* [in] 新阶段 */
    f32 progress,                      /* [in] 阶段进度 0.0~1.0 */
    u64 worker_pid,                    /* [in] 当前 Worker PID（0 表示无） */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 合法状态转换矩阵：
 *                    TO →
 * FROM ↓           IDLE  DESIGN  DEBUG  EVAL  REPORT  COMP  ABORT  EXP
 * IDLE              ✗     ✓      ✗     ✗     ✗      ✗     ✗     ✗
 * DESIGN            ✗     ✗      ✓     ✗     ✗      ✗     ✓     ✗
 * DEBUG             ✗     ✗      ✗     ✓     ✗      ✗     ✓     ✗
 * EVALUATE          ✗     ✗      ✗     ✗     ✓      ✗     ✓     ✗
 * REPORT            ✗     ✗      ✗     ✗     ✗      ✓     ✓     ✗
 * COMPLETED         ✗     ✗      ✗     ✗     ✗      ✗     ✗     ✗
 * ABORTED           ✗     ✗      ✗     ✗     ✗      ✗     ✗     ✗
 * EXPIRED           ✗     ✗      ✗     ✗     ✗      ✗     ✗     ✗
 *
 * 任何非法转换 → MALL_ERR_UNKNOWN，记录安全审计日志
 */
```

---

### 5.2 `mall_pipeline_stage_gate`

```c
/* 函数：流水线阶段质量门禁
 * 契约：
 *   前置条件：前一阶段已完成；artifacts 指向有效产出物
 *   后置条件：返回 GATE_PASS 或 GATE_FAIL；若 PASS，则状态自动推进到下一阶段
 *   副作用：可能调用 mall_task_state_update() 推进状态；可能调用 mall_pipeline_rollback()
 *   并发安全：单线程调用（由商场主循环串行执行）
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
typedef enum {
    GATE_PASS = 0,
    GATE_FAIL = 1,
    GATE_CONDITIONAL = 2   /* 通过但附带警告 */
} GateResult;

GateResult mall_pipeline_stage_gate(
    const char* task_token,             /* [in] 任务令牌 */
    PipelinePhase from_phase,           /* [in] 当前完成阶段 */
    PipelinePhase to_phase,           /* [in] 目标阶段 */
    const void* artifacts,            /* [in] 阶段产出物指针 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 门禁规则：
 * DESIGN → DEBUG:
 *   - 必须: design_output.code 非空
 *   - 必须: design_output.confidence_score >= 0.5
 *   - 必须: OOP 扫描通过（mall_oop_scanner() == MALL_OK）
 *   - 可选: dependency_count <= perm_dependency 允许的最大值
 *
 * DEBUG → EVALUATE:
 *   - 必须: debug_output.patch_valid == MALL_TRUE（如果生成了补丁）
 *   - 必须: 无致命错误码
 *
 * EVALUATE → REPORT:
 *   - 必须: evaluate_output.score_card.total >= 60.0
 *   - 必须: evaluate_output.score_card.oop_compliance == 10.0（一票否决）
 *   - 可选: total >= 80 → GATE_PASS; 60 <= total < 80 → GATE_CONDITIONAL
 */
```

---

### 5.3 `mall_pipeline_rollback`

```c
/* 函数：流水线回滚
 * 契约：
 *   前置条件：task_token 对应任务存在且未处于 COMPLETED/ARCHIVED 状态
 *   后置条件：任务所有临时 SHM 区域已清空；Worker 已释放；状态置为 ABORTED
 *   副作用：
 *     1. 清空 shm://aicoder/{task_token}/design/*
 *     2. 清空 shm://aicoder/{task_token}/debug/*
 *     3. 清空 shm://aicoder/{task_token}/evaluate/*
 *     4. 发送 SIGTERM 到 worker_pid（如果 > 0）
 *     5. 更新状态为 PHASE_ABORTED
 *     6. 记录审计事件 ROLLBACK_EXECUTED
 *   并发安全：发送 SIGTERM 后等待 Worker 退出（waitpid），避免僵尸进程
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int mall_pipeline_rollback(
    const char* task_token,             /* [in] 任务令牌 */
    MallErrorCode reason,              /* [in] 回滚原因 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

## 六、OOP 扫描函数

### 6.1 `mall_oop_scanner`

```c
/* 函数：OOP 语义扫描器
 * 契约：
 *   前置条件：code_blob 指向以 null 终止的源代码字符串；language 为有效语言标识
 *   后置条件：若返回 MALL_OK，则代码通过 OOP 扫描；若返回 AICODER_ERR_OOP_DETECTED，
 *             则 out_violations 包含检测到的违规列表
 *   副作用：无 SHM 写入；纯 CPU 计算
 *   并发安全：线程安全（无共享状态）
 *   时间复杂度：O(L)，L 为代码长度
 *   空间复杂度：O(L) —— 语法树栈空间
 */
int mall_oop_scanner(
    const char* code_blob,              /* [in] 源代码字符串 */
    const char* language,               /* [in] 语言标识 */
    struct OopViolation* out_violations, /* [out] 违规列表（固定大小数组） */
    u64* out_violation_count,           /* [out] 违规数量 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 扫描规则（按语言）：
 * Python:
 *   - 禁止关键字: class, self, __init__, super(), @property, @classmethod, @staticmethod
 *   - 禁止模式: "class "（类定义）, "self."（实例访问）, "def __"（魔术方法）
 *   - 禁止模式: 继承语法 "class X(Y)"
 *
 * C:
 *   - 允许 struct（纯数据聚合）
 *   - 禁止: struct 内包含函数指针（模拟虚表）
 *   - 禁止: 复杂的宏模拟 OOP（如 OBJECT_DECLARE, METHOD_DECLARE 等模式）
 *
 * Java:
 *   - 禁止: class, interface, extends, implements, this, new Object()
 *   - 允许: 纯静态函数文件（所有函数为 static，无类实例化）
 *
 * JavaScript:
 *   - 禁止: class, extends, super, this, new, prototype
 *   - 允许: 纯函数定义（function foo() {}）
 *   - 允许: 箭头函数（const foo = () => {}）
 *
 * Verilog:
 *   - 禁止: OOP 扩展（如 SystemVerilog 的 class 关键字）
 *   - 允许: 纯模块级设计（module, always, assign）
 *
 * CSS:
 *   - 无 OOP 语义，通常直接通过
 */

typedef struct {
    u64 line_number;
    char pattern[MALL_MAX_NAME_LEN];    /* 检测到的违规模式 */
    char context[128];                  /* 违规代码上下文 */
} OopViolation;
```

---

## 七、生产者通知函数

### 7.1 `mall_notify_producer`

```c
/* 函数：向生产者发送通知
 * 契约：
 *   前置条件：producer_id 在商场注册表中存在
 *   后置条件：通知消息已写入生产者的 SHM 通知信箱
 *   副作用：写入 SHM: shm://producer/{producer_id}/notifications
 *   并发安全：使用生产者信箱的尾部 CAS 操作
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int mall_notify_producer(
    const char* producer_id,            /* [in] 生产者标识 */
    const struct MallError* event,      /* [in] 通知事件（成功或错误） */
    const char* task_token              /* [in] 关联的任务令牌（可为空） */
);

/* 通知信箱结构（每个生产者一个）：
 * struct ProducerMailbox {
 *     u64 head;                        // 生产者读取索引
 *     u64 tail;                        // 商场写入索引
 *     u64 capacity;                    // 容量（固定 64 条）
 *     Notification messages[64];      // 消息数组
 * }
 *
 * 通知满 → 覆盖最旧消息（循环覆盖策略）
 */

typedef struct {
    u64 timestamp_ns;
    MallErrorCode code;
    char message[MALL_MAX_ERR_MSG];
    char task_token[MALL_UUID_LEN];
} Notification;
```

---

## 八、审核宣言

> 商场主控层是体系的**心脏**——它泵送任务血液，监控状态脉搏，守护安全边界。  
> 每个函数都经过严格的契约设计：前置条件明确假设，后置条件保证结果，副作用清单透明公开。  
> 这不是传统的"框架"或"运行时"，而是**函数级的操作系统**——以纯函数为细胞，以 SHM 矢量为血管，以状态机为神经。  
> **审核状态**：待人类架构师最终裁定。
