# 商场创世纪七阶段函数级设计文档（AICoder 外置版）

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **版本：** v2.0 — AICoder 外置架构重构  
> **核心约束：** AICoder 作为独立外置服务运行，商场仅通过标准协议代理调用，不嵌入任何 AICoder 运行时。  
> **设计铁律：** 函数级设计，禁止 OOP；人类掌握审核权；商场拥有编译进化权。

---

## 一、顶层架构关系

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           外置 AICoder 集群（独立进程/节点）                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────────┐ │
│  │  /design 端点   │  │  /debug 端点    │  │  /evaluate 端点                 │ │
│  │  设计生成引擎   │  │  调试修复引擎    │  │  质量评估引擎                   │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────────────────────┘ │
│           │                    │                    │                          │
│           └────────────────────┼────────────────────┘                          │
│                                ▼                                               │
│                    ┌─────────────────────┐                                     │
│                    │   AICoder 网关      │  ← 统一协议入口，负载均衡             │
│                    │  (REST/gRPC/SHM桥)  │                                     │
│                    └──────────┬──────────┘                                     │
└───────────────────────────────┼──────────────────────────────────────────────┘
                                │ 标准协议（HTTP/gRPC/SHM桥接）
                                │ 双向 mTLS / 签名认证
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              商  场（Mall）                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────────┐ │
│  │  蓝图接收网关    │  │  AICoder 代理层  │  │  审核编译进化节点                │ │
│  │  (Gateway)      │  │  (Agent Layer)  │  │  (Audit & Compile)              │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────────────────────┘ │
│           │                    │                    │                          │
│           └────────────────────┼────────────────────┘                          │
│                                ▼                                               │
│                    ┌─────────────────────┐                                     │
│                    │   消费者函数池       │  ← 已进化部署的可执行函数集           │
│                    │   (Consumer Pool)   │                                     │
│                    └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────────────────────┘
                                ▲
                                │ 授权蓝图 + 签名
                                │
                    ┌───────────┴───────────┐
                    │     生产者 (Producer)   │
                    └───────────────────────┘
```

**三大铁律：**
1. **物理隔离：** AICoder 进程与商场进程地址空间完全隔离，禁止直接内存共享（除受控 SHM 桥接外）。
2. **协议中介：** 所有 AICoder 交互必须经过 `mall_aicoder_agent_*` 代理层，禁止商场其他模块直接访问外置服务。
3. **失效兜底：** AICoder 集群离线或超时，商场必须降级为「蓝图暂存模式」，拒绝新设计任务但保持已有消费者运行。

---

## 二、全局数据结构定义

```c
/*============================================================
 *  全局标识与基础类型
 *============================================================*/
#define MALL_MAX_PRODUCER_ID_LEN     32
#define MALL_MAX_SESSION_ID_LEN      64
#define MALL_MAX_HASH_LEN            64
#define MALL_MAX_LANG_LEN            16
#define MALL_MAX_PATH_LEN            256
#define MALL_MAX_PROMPT_LEN          4096
#define MALL_MAX_RULES_LEN           2048
#define MALL_MAX_CODE_LEN            65536
#define MALL_MAX_DOC_LEN             32768
#define MALL_MAX_ERROR_MSG_LEN       2048
#define MALL_MAX_REPORT_LEN          16384
#define MALL_MAX_FLAGS_LEN           1024
#define MALL_MAX_ENDPOINT_LEN        128

typedef enum {
    MALL_OK = 0,
    MALL_ERR_INVALID_PARAM = -1,
    MALL_ERR_AUTH_FAILED = -2,
    MALL_ERR_BLUEPRINT_INVALID = -3,
    MALL_ERR_AICODER_OFFLINE = -4,
    MALL_ERR_AICODER_TIMEOUT = -5,
    MALL_ERR_AICODER_RESPONSE_CORRUPT = -6,
    MALL_ERR_AUDIT_REJECTED = -7,
    MALL_ERR_COMPILE_FAILED = -8,
    MALL_ERR_DEPLOY_FAILED = -9,
    MALL_ERR_SHM_FAULT = -10
} MallStatus;

typedef enum {
    AICODER_TASK_DESIGN = 0,
    AICODER_TASK_DEBUG = 1,
    AICODER_TASK_EVALUATE = 2
} AICoderTaskType;

typedef enum {
    AICODER_COMM_HTTP_POST = 0,
    AICODER_COMM_GRPC = 1,
    AICODER_COMM_SHM_BRIDGE = 2
} AICoderCommType;

/*============================================================
 *  生产者身份凭证
 *============================================================*/
typedef struct {
    char producer_id[MALL_MAX_PRODUCER_ID_LEN];
    char auth_signature[MALL_MAX_HASH_LEN];      /* HMAC-SHA256 签名 */
    unsigned long long register_timestamp;
    int privilege_level;                          /* 0=普通 1=创世 2=管理员 */
} ProducerCredential;

/*============================================================
 *  蓝图元数据（商场内部表示）
 *============================================================*/
typedef struct {
    char session_id[MALL_MAX_SESSION_ID_LEN];     /* 商场全局唯一会话 ID */
    char producer_id[MALL_MAX_PRODUCER_ID_LEN];
    char blueprint_hash[MALL_MAX_HASH_LEN];       /* 蓝图文件 SHA-256 */
    char target_language[MALL_MAX_LANG_LEN];      /* "python", "verilog", "c"... */
    char blueprint_path[MALL_MAX_PATH_LEN];
    unsigned long long submit_timestamp;
    int validation_passed;                        /* 0=未校验 1=通过 2=失败 */
} BlueprintMeta;

/*============================================================
 *  AICoder 外置请求包（代理层生成）
 *============================================================*/
typedef struct {
    char session_id[MALL_MAX_SESSION_ID_LEN];
    char producer_id[MALL_MAX_PRODUCER_ID_LEN];
    char blueprint_hash[MALL_MAX_HASH_LEN];
    char target_language[MALL_MAX_LANG_LEN];
    char design_prompt[MALL_MAX_PROMPT_LEN];      /* 生产者意图描述 */
    char constraint_rules[MALL_MAX_RULES_LEN];    /* 商场铁律（如禁止 OOP） */
    AICoderTaskType task_type;

    /* 调试/评估专用字段 */
    char existing_code[MALL_MAX_CODE_LEN];        /* 待调试代码 */
    char error_log[MALL_MAX_ERROR_MSG_LEN];       /* 错误日志 */
    char test_cases[MALL_MAX_DOC_LEN];            /* 测试用例 */
} AICoderRequestPacket;

/*============================================================
 *  AICoder 外置响应包（代理层解析）
 *============================================================*/
typedef struct {
    char session_id[MALL_MAX_SESSION_ID_LEN];
    int status;                                   /* 0=成功 1=部分成功 2=失败 */
    char generated_code[MALL_MAX_CODE_LEN];
    char design_doc[MALL_MAX_DOC_LEN];
    char risk_flags[MALL_MAX_FLAGS_LEN];          /* AICoder 自检风险提示 */
    int estimated_complexity;                     /* 1~10 复杂度评分 */
    char raw_response_checksum[MALL_MAX_HASH_LEN]; /* 完整性校验 */
} AICoderResponsePacket;

/*============================================================
 *  设计草案（商场内部审核用）
 *============================================================*/
typedef struct {
    char session_id[MALL_MAX_SESSION_ID_LEN];
    char producer_id[MALL_MAX_PRODUCER_ID_LEN];
    char generated_code[MALL_MAX_CODE_LEN];
    char design_doc[MALL_MAX_DOC_LEN];
    char risk_flags[MALL_MAX_FLAGS_LEN];
    int complexity_score;
    int aicoder_status;
} DesignDraft;

/*============================================================
 *  审核结果
 *============================================================*/
typedef struct {
    int approved;                                 /* 0=驳回 1=通过 */
    char audit_comment[MALL_MAX_DOC_LEN];         /* 审核意见 */
    char compiled_binary_path[MALL_MAX_PATH_LEN]; /* 编译产物路径 */
    int evolved_version;                          /* 进化版本号 */
} AuditResult;

/*============================================================
 *  AICoder 外置服务端点配置
 *============================================================*/
typedef struct {
    char endpoint[MALL_MAX_ENDPOINT_LEN];         /* 如 "https://aicoder.mall.local:8443" */
    AICoderCommType comm_type;
    int timeout_ms;                               /* 默认 30000ms */
    int max_retry;                                /* 默认 3 次 */
    char tls_cert_path[MALL_MAX_PATH_LEN];
    char tls_key_path[MALL_MAX_PATH_LEN];
} AICoderEndpointConfig;
```

---

## 三、创世纪七阶段函数级流程

### 阶段 1：生产者创世注册（Genesis Registration）

**目标：** 生产者在商场登记身份，获取授权令牌，建立信任根。

```c
/*------------------------------------------------------------
 * 函数：mall_producer_register
 * 职责：生产者首次进入商场，提交身份信息，生成唯一凭证
 * 调用方：生产者客户端 / 商场初始化脚本
 *------------------------------------------------------------*/
int mall_producer_register(
    const char* producer_name,           /* 生产者名称（人类可读） */
    const char* public_key_pem,          /* 生产者 RSA 公钥 */
    const char* genesis_contract_hash,   /* 创世合约摘要 */
    ProducerCredential* out_credential   /* 输出：商场签发的凭证 */
);

/* 返回：MALL_OK / MALL_ERR_INVALID_PARAM / MALL_ERR_AUTH_FAILED */


/*------------------------------------------------------------
 * 函数：mall_producer_authorize
 * 职责：校验生产者凭证有效性，确认其提交蓝图权限
 * 调用方：蓝图接收网关（阶段 2 前置检查）
 *------------------------------------------------------------*/
int mall_producer_authorize(
    const ProducerCredential* credential,
    const char* blueprint_hash,          /* 待授权蓝图的摘要 */
    int requested_privilege              /* 申请权限级别 */
);

/* 返回：MALL_OK（授权通过）/ MALL_ERR_AUTH_FAILED */
```

**阶段 1 调用序列：**
```
生产者 → mall_producer_register(public_key, contract) 
       → 商场签发 ProducerCredential 
       → 生产者本地保存 credential
```

---

### 阶段 2：蓝图提交与网关校验（Blueprint Injection）

**目标：** 生产者将授权蓝图注入商场，网关完成格式、签名、哈希三重校验，生成内部会话。

```c
/*------------------------------------------------------------
 * 函数：mall_gateway_accept_blueprint
 * 职责：接收生产者提交的蓝图文件与授权签名，生成商场内部会话
 * 调用方：生产者客户端
 *------------------------------------------------------------*/
int mall_gateway_accept_blueprint(
    const char* producer_id,
    const char* blueprint_path,          /* 本地文件路径或 SHM 路径 */
    const char* auth_signature,          /* 生产者对蓝图哈希的签名 */
    BlueprintMeta* out_meta              /* 输出：商场内部蓝图元数据 */
);

/* 内部动作：
 *   1. 读取蓝图文件内容
 *   2. 计算 SHA-256 得到 blueprint_hash
 *   3. 调用 mall_producer_authorize() 校验签名与权限
 *   4. 生成 UUID 作为 session_id
 *   5. 将 out_meta 写入商场「待处理池」（Pending Pool）
 */


/*------------------------------------------------------------
 * 函数：mall_blueprint_validate
 * 职责：对蓝图内容进行语法与规则预检
 * 调用方：网关内部 / 商场调度器
 *------------------------------------------------------------*/
int mall_blueprint_validate(
    const BlueprintMeta* meta,
    const char* blueprint_content,       /* 蓝图 JSON/XML 文本 */
    char* out_error_msg,                 /* 输出：校验失败原因 */
    size_t error_msg_max_len
);

/* 校验项：
 *   - 目标语言是否在支持列表（python, verilog, c, java, js, css）
 *   - 设计意图字段非空
 *   - 不包含商场禁止关键词（如 "class", "self", "extends" —— OOP 禁令）
 *   - JSON 格式合法
 */
```

**阶段 2 调用序列：**
```
生产者 → mall_gateway_accept_blueprint(producer_id, blueprint.sig)
       → 商场网关
            ├── 读取蓝图文件
            ├── 计算 blueprint_hash
            ├── 调用 mall_producer_authorize() → 通过
            ├── 生成 session_id
            └── 写入 Pending Pool
       → 返回 BlueprintMeta

商场调度器 → mall_blueprint_validate(meta, content)
           → 通过：标记 validation_passed=1，进入阶段 3
           → 失败：标记 validation_passed=2，退回生产者
```

---

### 阶段 3：AICoder 外置设计调用（External Design Invocation）

**目标：** 商场通过代理层，将蓝图转译为标准化请求，调用外置 AICoder `/design` 端点，获取设计草案。

**核心变更：** 商场不再直接调用内嵌函数，而是通过 `AICoder Agent` 层发起外置网络/进程间调用。

```c
/*------------------------------------------------------------
 * 函数：mall_aicoder_agent_compose_request
 * 职责：将商场内部 BlueprintMeta 封装为 AICoder 外置协议包
 * 调用方：商场调度器（阶段 3 入口）
 *------------------------------------------------------------*/
int mall_aicoder_agent_compose_request(
    const BlueprintMeta* meta,
    const char* design_prompt,           /* 从蓝图中提取的设计意图 */
    const char* constraint_rules,      /* 商场铁律文本 */
    AICoderRequestPacket* out_packet     /* 输出：标准协议包 */
);

/* 内部动作：
 *   1. 将 meta 中的字段映射到 out_packet
 *   2. 将商场铁律（如 "禁止 OOP，全部使用函数级设计"）写入 constraint_rules
 *   3. 设置 task_type = AICODER_TASK_DESIGN
 *   4. 生成请求体校验和
 */


/*------------------------------------------------------------
 * 函数：mall_aicoder_agent_invoke
 * 职责：通过配置的通信协议，向外置 AICoder 发起调用并等待响应
 * 调用方：商场调度器
 * 约束：此函数必须设置硬超时，禁止阻塞商场主循环
 *------------------------------------------------------------*/
int mall_aicoder_agent_invoke(
    const AICoderEndpointConfig* config,
    const AICoderRequestPacket* packet,
    AICoderResponsePacket* out_response,
    int* out_http_status                 /* 输出：原始 HTTP/gRPC 状态码 */
);

/* 内部动作：
 *   1. 根据 config->comm_type 选择通信后端：
 *        HTTP_POST → 构造 JSON 负载，发送 POST，设置 timeout_ms
 *        GRPC      → 通过 protobuf 序列化，调用 stub
 *        SHM_BRIDGE→ 将 packet 写入共享内存环形队列，等待信号量
 *   2. 若超时（> timeout_ms）：返回 MALL_ERR_AICODER_TIMEOUT
 *   3. 若连接失败：返回 MALL_ERR_AICODER_OFFLINE
 *   4. 接收原始响应，校验 checksum
 *   5. 将原始响应反序列化到 out_response
 *   6. 幂等性检查：若 session_id 已存在，直接返回缓存结果
 */


/*------------------------------------------------------------
 * 函数：mall_aicoder_agent_parse_response
 * 职责：将 AICoder 外置响应解析为商场内部 DesignDraft 结构
 * 调用方：商场调度器（mall_aicoder_agent_invoke 之后）
 *------------------------------------------------------------*/
int mall_aicoder_agent_parse_response(
    const AICoderResponsePacket* response,
    const BlueprintMeta* original_meta,    /* 用于交叉校验 session_id */
    DesignDraft* out_draft
);

/* 校验项：
 *   - response->session_id 必须与 original_meta->session_id 一致
 *   - 校验 response->raw_response_checksum
 *   - 若 status == 2（失败），将 risk_flags 写入日志
 *   - 提取 generated_code, design_doc, complexity_score
 */
```

**阶段 3 调用序列（时序图）：**
```
商场调度器
    │
    ▼
mall_aicoder_agent_compose_request(meta, prompt, rules)
    │
    ▼
生成 AICoderRequestPacket (task_type=DESIGN)
    │
    ▼
mall_aicoder_agent_invoke(config, packet, &response, &http_status)
    │
    ├───────────────────────────────────────────────┐
    │  HTTP POST /design                              │
    │  Header: X-Mall-Session, X-Producer-Id          │  →  外置 AICoder 网关
    │  Body: JSON(AICoderRequestPacket)               │     （独立进程/节点）
    │                                                 │
    │  ← 返回 AICoderResponsePacket (JSON/Protobuf)   │
    ├───────────────────────────────────────────────┘
    │
    ▼
校验 http_status == 200 且 response.status != 2
    │
    ▼
mall_aicoder_agent_parse_response(response, meta, &draft)
    │
    ▼
生成 DesignDraft → 写入商场「设计草案池」（Draft Pool）
    │
    ▼
触发阶段 4（调试）或阶段 5（评估）—— 根据商场配置
```

**错误处理路径：**
| 错误码 | 触发条件 | 商场行为 |
|--------|----------|----------|
| `MALL_ERR_AICODER_OFFLINE` | TCP 连接拒绝 / gRPC UNAVAILABLE | 标记 AICoder 为 DOWN，蓝图回退到 Pending Pool，启动指数退避重试 |
| `MALL_ERR_AICODER_TIMEOUT` | 30 秒内无响应 | 中断当前请求，记录超时日志，生产者收到「服务繁忙」通知 |
| `MALL_ERR_AICODER_RESPONSE_CORRUPT` | Checksum 不匹配 / JSON 解析失败 | 丢弃响应，请求 AICoder 重新生成（max_retry 次） |

---

### 阶段 4：外置调试迭代（External Debug Iteration）

**目标：** 若阶段 3 生成的代码在商场沙箱编译或单元测试失败，将错误上下文通过代理层提交给外置 AICoder `/debug` 端点，获取修复补丁。

```c
/*------------------------------------------------------------
 * 函数：mall_aicoder_agent_debug_invoke
 * 职责：封装调试请求，调用外置 AICoder /debug 端点
 * 调用方：商场沙箱测试模块
 *------------------------------------------------------------*/
int mall_aicoder_agent_debug_invoke(
    const AICoderEndpointConfig* config,
    const char* session_id,              /* 关联原设计会话 */
    const char* failing_code,            /* 当前失败的代码 */
    const char* compiler_error,          /* 编译器错误信息 */
    const char* runtime_log,             /* 运行时日志 */
    AICoderResponsePacket* out_patch     /* 输出：修复后的代码与说明 */
);

/* 内部动作：
 *   1. 构造 AICoderRequestPacket：
 *        task_type = AICODER_TASK_DEBUG
 *        existing_code = failing_code
 *        error_log = compiler_error + runtime_log
 *   2. 调用 mall_aicoder_agent_invoke() 指向 /debug 端点
 *   3. 解析返回的 generated_code 作为补丁
 */


/*------------------------------------------------------------
 * 函数：mall_debug_patch_apply
 * 职责：将 AICoder 返回的修复补丁应用到设计草案
 * 调用方：商场调度器
 *------------------------------------------------------------*/
int mall_debug_patch_apply(
    DesignDraft* draft,
    const AICoderResponsePacket* patch,
    int max_patch_rounds,                /* 最大补丁轮次，防止无限循环 */
    int* out_applied_rounds
);

/* 约束：
 *   - 每轮补丁后必须在商场沙箱重新编译测试
 *   - 若 max_patch_rounds 耗尽仍未通过，标记为「不可修复」，退回阶段 6 人工审核
 *   - 禁止自动无限迭代，防止 AICoder 陷入循环生成
 */
```

**阶段 4 调用序列：**
```
商场沙箱
    │
    ▼
编译/测试 DesignDraft.generated_code → 失败
    │
    ▼
提取 compiler_error + runtime_log
    │
    ▼
mall_aicoder_agent_debug_invoke(config, session_id, code, error, log, &patch)
    │
    ├───────────────────────────────────────────────┐
    │  HTTP POST /debug                               │
    │  Body: {task_type: DEBUG, existing_code, error_log} │ → 外置 AICoder
    │  ← 返回 {generated_code: "修复后代码", risk_flags}    │
    ├───────────────────────────────────────────────┘
    │
    ▼
mall_debug_patch_apply(draft, patch, max_rounds=5, &rounds)
    │
    ▼
重新编译测试 → 通过：进入阶段 5
              → 失败且 rounds < 5：再次调用 debug_invoke
              → 失败且 rounds >= 5：标记「调试耗尽」，进入阶段 6 人工审核
```

---

### 阶段 5：外置评估与报告（External Evaluation）

**目标：** 商场将已通过调试的代码与测试用例提交给外置 AICoder `/evaluate` 端点，获取质量评估报告，作为阶段 6 审核的量化依据。

```c
/*------------------------------------------------------------
 * 函数：mall_aicoder_agent_evaluate_invoke
 * 职责：封装评估请求，调用外置 AICoder /evaluate 端点
 * 调用方：商场调度器（调试通过后自动触发）
 *------------------------------------------------------------*/
int mall_aicoder_agent_evaluate_invoke(
    const AICoderEndpointConfig* config,
    const char* session_id,
    const char* candidate_code,          /* 待评估的代码 */
    const char* test_suite,              /* 商场提供的测试用例集 */
    const char* design_requirements,     /* 原始设计需求 */
    AICoderResponsePacket* out_report    /* 输出：评估报告 */
);

/* 内部动作：
 *   1. 构造 AICoderRequestPacket：
 *        task_type = AICODER_TASK_EVALUATE
 *        existing_code = candidate_code
 *        test_cases = test_suite
 *        design_prompt = design_requirements
 *   2. 调用 mall_aicoder_agent_invoke() 指向 /evaluate 端点
 *   3. 解析返回的 design_doc 作为评估报告（JSON 格式评分）
 */


/*------------------------------------------------------------
 * 函数：mall_evaluate_report_parse
 * 职责：将 AICoder 评估报告解析为结构化评分
 * 调用方：商场审核节点
 *------------------------------------------------------------*/
int mall_evaluate_report_parse(
    const AICoderResponsePacket* report,
    int* out_correctness_score,          /* 正确性 0~100 */
    int* out_performance_score,            /* 性能 0~100 */
    int* out_security_score,             /* 安全性 0~100 */
    int* out_maintainability_score,      /* 可维护性 0~100 */
    char* out_summary,                   /* 评估摘要 */
    size_t summary_max_len
);

/* 说明：
 *   - 若任一维度评分 < 60，在阶段 6 审核时高亮警告
 *   - 评估报告仅作为「参考」，不直接决定通过/驳回
 *   - 人类审核权保留：即使 AICoder 评分 100，人类仍可驳回
 */
```

**阶段 5 调用序列：**
```
阶段 4 调试通过
    │
    ▼
mall_aicoder_agent_evaluate_invoke(config, session_id, code, tests, reqs, &report)
    │
    ├───────────────────────────────────────────────┐
    │  HTTP POST /evaluate                            │
    │  Body: {task_type: EVALUATE, existing_code,    │ → 外置 AICoder
    │         test_cases, design_prompt}              │
    │  ← 返回 {design_doc: "评估报告 JSON",            │
    │          risk_flags, estimated_complexity}      │
    ├───────────────────────────────────────────────┘
    │
    ▼
mall_evaluate_report_parse(report, &c_score, &p_score, &s_score, &m_score, summary)
    │
    ▼
将评分与报告附加到 DesignDraft → 进入阶段 6
```

---

### 阶段 6：商场审核与编译进化（Mall Audit & Compile Evolution）

**目标：** 人类（或授权审核节点）对设计草案行使审核权；审核通过后，商场行使编译进化权，将草案固化为可执行函数集。

```c
/*------------------------------------------------------------
 * 函数：mall_audit_compile_design_draft
 * 职责：审核设计草案，通过后触发编译进化
 * 调用方：商场审核节点（人类触发或自动化 gate）
 *------------------------------------------------------------*/
int mall_audit_compile_design_draft(
    const DesignDraft* draft,
    const BlueprintMeta* original_meta,
    const int* evaluation_scores,        /* 阶段 5 评分数组 [4] */
    int audit_mode,                      /* 0=人工审核 1=自动 gate（仅高信任生产者） */
    AuditResult* out_result
);

/* 审核检查项（函数级清单）：
 *   1. 代码中是否出现 OOP 关键词（class, extends, new, this, self...）
 *   2. 是否包含系统调用禁令（system(), exec(), fork() 等）
 *   3. 函数签名是否与蓝图需求一致
 *   4. 复杂度评分是否在可接受范围
 *   5. 风险标志是否包含 CRITICAL
 *   6. 人工审核意见（audit_mode=0 时强制要求）
 *   
 *   通过 → 调用 mall_compiler_evolve() 生成二进制/字节码
 *   驳回 → out_result->approved = 0，附带 audit_comment
 */


/*------------------------------------------------------------
 * 函数：mall_compiler_evolve
 * 职责：将审核通过的代码编译为商场可加载函数，赋予进化版本号
 * 调用方：mall_audit_compile_design_draft 内部（审核通过后自动调用）
 *------------------------------------------------------------*/
int mall_compiler_evolve(
    const char* source_code,
    const char* target_language,
    const char* session_id,
    int optimization_level,              /* 0=调试 1=标准 2=高性能 */
    char* out_binary_path,
    size_t path_max_len,
    int* out_version
);

/* 编译动作：
 *   - Python → 生成 .pyc 或存入商场内部字节码缓存
 *   - C → 调用 gcc/clang 生成 .so，沙箱中链接
 *   - Verilog → 生成仿真模型或比特流配置（视商场硬件层而定）
 *   - Java → 生成 .class，商场自定义类加载器加载
 *   - JavaScript/CSS → 商场 V8/CSS 引擎直接解析缓存
 *   
 *   进化版本号规则：同一 session 多次编译，version 递增
 *   旧版本保留，支持回滚
 */
```

**阶段 6 调用序列：**
```
DesignDraft + 评估报告
    │
    ▼
人工审核界面 / 自动 Gate
    │
    ▼
mall_audit_compile_design_draft(draft, meta, scores, audit_mode=0, &result)
    │
    ├── 审核驳回 → result.approved = 0
    │              → 退回生产者，附带 audit_comment
    │
    └── 审核通过 → 内部调用 mall_compiler_evolve()
                   │
                   ▼
                   生成 binary_path + version
                   │
                   ▼
                   写入商场「进化函数库」（Evolved Library）
                   │
                   ▼
                   触发阶段 7
```

---

### 阶段 7：部署与消费者激活（Deployment & Consumer Activation）

**目标：** 将编译进化后的函数部署到消费者层，激活服务，完成创世纪闭环。

```c
/*------------------------------------------------------------
 * 函数：mall_deploy_to_consumer
 * 职责：将编译产物注册到消费者函数池，建立调用句柄
 * 调用方：商场调度器（阶段 6 通过后自动触发）
 *------------------------------------------------------------*/
int mall_deploy_to_consumer(
    const char* session_id,
    const char* binary_path,
    int version,
    const char* consumer_pool_id,        /* 目标消费者池 */
    char* out_function_handle,           /* 输出：消费者调用句柄 */
    size_t handle_max_len
);

/* 部署动作：
 *   1. 将 binary_path 映射到 consumer_pool_id 的函数命名空间
 *   2. 生成唯一 function_handle（如 "mall://consumer_pool/func_v3"）
 *   3. 更新商场路由表：蓝图 session_id → function_handle
 *   4. 若同一 session 已部署旧版本，标记旧版为「待退役」，新版为「活跃」
 */


/*------------------------------------------------------------
 * 函数：mall_consumer_activate
 * 职责：激活消费者函数，使其接受外部调用请求
 * 调用方：商场调度器 / 消费者管理模块
 *------------------------------------------------------------*/
int mall_consumer_activate(
    const char* function_handle,
    int concurrency_limit,               /* 并发限制 */
    int* out_active_pid                  /* 输出：激活的进程/线程 ID */
);

/* 激活动作：
 *   1. 加载 binary 到消费者运行时
 *   2. 初始化函数上下文（SHM 矢量分配、信号量注册）
 *   3. 设置 concurrency_limit 信号量
 *   4. 标记状态为 ACTIVE
 *   5. 向商场事件总线广播：FUNCTION_ACTIVE(session_id, handle, version)
 */


/*------------------------------------------------------------
 * 函数：mall_genesis_close_session
 * 职责：关闭创世纪会话，清理中间态资源
 * 调用方：商场调度器（阶段 7 完成后）
 *------------------------------------------------------------*/
int mall_genesis_close_session(
    const char* session_id,
    int retain_draft                     /* 0=清理草案 1=保留归档 */
);

/* 清理动作：
 *   - 从 Pending Pool 移除
 *   - 从 Draft Pool 移除（若 retain_draft==0）
 *   - 保留 BlueprintMeta 到归档库（审计追溯）
 *   - 释放 AICoder 代理层为该 session 分配的临时缓冲
 */
```

**阶段 7 调用序列：**
```
阶段 6 编译进化完成
    │
    ▼
mall_deploy_to_consumer(session_id, binary_path, version, "consumer_pool_alpha", &handle)
    │
    ▼
函数句柄注册到路由表
    │
    ▼
mall_consumer_activate(handle, concurrency_limit=10, &pid)
    │
    ▼
消费者进程就绪，接受调用
    │
    ▼
mall_genesis_close_session(session_id, retain_draft=1)
    │
    ▼
创世纪闭环完成 —— 生产者蓝图已转化为活跃消费者函数
```

---

## 四、AICoder 外置服务端点规范（商场视角）

AICoder 外置集群必须暴露以下 RESTful/gRPC 端点。商场代理层仅与这些端点交互。

### 4.1 端点清单

| 端点 | 方法 | 请求体 | 响应体 | 商场代理函数 |
|------|------|--------|--------|--------------|
| `/design` | POST | `AICoderRequestPacket` (task_type=DESIGN) | `AICoderResponsePacket` | `mall_aicoder_agent_invoke()` |
| `/debug` | POST | `AICoderRequestPacket` (task_type=DEBUG) | `AICoderResponsePacket` | `mall_aicoder_agent_debug_invoke()` |
| `/evaluate` | POST | `AICoderRequestPacket` (task_type=EVALUATE) | `AICoderResponsePacket` | `mall_aicoder_agent_evaluate_invoke()` |
| `/health` | GET | — | `{status: "up"\|"down", load: 0~100}` | 商场心跳检测（独立线程） |

### 4.2 请求头强制字段

```http
POST /design HTTP/1.1
Host: aicoder.mall.local:8443
Content-Type: application/json
X-Mall-Session:      550e8400-e29b-41d4-a716-446655440000
X-Producer-Id:       producer_alpha_01
X-Blueprint-Hash:    a3f5c8...（64字符SHA-256）
X-Auth-Signature:    （商场对请求体的HMAC签名，防篡改）
X-Request-Checksum:  （请求体SHA-256，防传输损坏）
```

### 4.3 响应体结构示例（JSON）

```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": 0,
  "generated_code": "def compute_spectrum(data, fs):\n    ...",
  "design_doc": "## 函数设计文档\n...",
  "risk_flags": "INFO: Uses floating-point division; No critical risks.",
  "estimated_complexity": 4,
  "raw_response_checksum": "b2d4e8..."
}
```

---

## 五、通信协议适配层

商场代理层支持三种通信后端，通过 `AICoderCommType` 选择。

```c
/*------------------------------------------------------------
 * 函数：mall_aicoder_comm_http_post
 * 职责：HTTP POST 后端实现
 *------------------------------------------------------------*/
int mall_aicoder_comm_http_post(
    const AICoderEndpointConfig* config,
    const char* endpoint_path,           /* "/design", "/debug", "/evaluate" */
    const char* json_payload,
    char* out_response_buffer,
    size_t buffer_max_len,
    int* out_http_status
);

/*------------------------------------------------------------
 * 函数：mall_aicoder_comm_grpc
 * 职责：gRPC 后端实现（需 proto 定义）
 *------------------------------------------------------------*/
int mall_aicoder_comm_grpc(
    const AICoderEndpointConfig* config,
    const AICoderRequestPacket* packet,
    AICoderResponsePacket* out_response
);

/*------------------------------------------------------------
 * 函数：mall_aicoder_comm_shm_bridge
 * 职责：同机 SHM 桥接后端（非嵌入！仅共享环形队列）
 * 约束：仅适用于 AICoder 与商场部署在同一物理节点
 *------------------------------------------------------------*/
int mall_aicoder_comm_shm_bridge(
    const char* shm_segment_name,        /* 如 "/dev/shm/mall_aicoder_bridge" */
    const AICoderRequestPacket* packet,
    AICoderResponsePacket* out_response,
    int timeout_ms
);
```

---

## 六、商场 AICoder 心跳与熔断机制

因 AICoder 外置，商场必须独立维护健康状态，防止将蓝图投递到已死节点。

```c
/*------------------------------------------------------------
 * 函数：mall_aicoder_health_check
 * 职责：定期探测外置 AICoder 健康状态
 * 调用方：商场守护线程（每 10 秒执行）
 *------------------------------------------------------------*/
int mall_aicoder_health_check(
    const AICoderEndpointConfig* config,
    int* out_status,                     /* 0=UP 1=DOWN */
    int* out_load_percentage             /* 0~100，用于负载均衡 */
);

/*------------------------------------------------------------
 * 函数：mall_aicoder_circuit_breaker_eval
 * 职责：熔断器评估，连续失败超过阈值则断开
 * 调用方：mall_aicoder_agent_invoke 内部
 *------------------------------------------------------------*/
int mall_aicoder_circuit_breaker_eval(
    const char* endpoint,
    int consecutive_failures,            /* 连续失败次数 */
    int failure_threshold,               /* 熔断阈值，默认 5 */
    int* out_state                       /* 0=CLOSED 1=OPEN 2=HALF_OPEN */
);

/* 熔断规则：
 *   CLOSED   → 正常投递请求
 *   OPEN     → 直接拒绝请求，返回 MALL_ERR_AICODER_OFFLINE
 *   HALF_OPEN → 每 30 秒允许 1 个探测请求，成功则恢复 CLOSED
 */
```

---

## 七、安全隔离铁律（外置架构强化版）

| 铁律编号 | 内容 | 技术实现 |
|----------|------|----------|
| **S1** | **网络隔离** | 商场与 AICoder 之间强制 mTLS 双向认证，证书由商场 CA 签发 |
| **S2** | **输入消毒** | 商场网关层校验 `blueprint_hash` 与 `auth_signature`，AICoder 只接收商场代理转发的已校验包 |
| **S3** | **沙箱返回** | AICoder 返回的 `generated_code` 必须在商场沙箱先编译/模拟执行，确认无恶意指令（如 `rm -rf`、`__import__('os').system`） |
| **S4** | **拒绝循环依赖** | AICoder 外置服务禁止回调商场内部接口（禁止反向 HTTP 调用商场） |
| **S5** | **会话隔离** | 每个 `session_id` 的 AICoder 上下文独立，禁止跨会话泄露代码片段 |
| **S6** | **超时熔断** | 任何 AICoder 调用超过 `timeout_ms` 立即中断，防止商场主循环阻塞 |
| **S7** | **权限最小化** | AICoder 进程以无特权用户运行（nobody/aicoder），禁止访问商场文件系统 |

---

## 八、创世纪完整调用时序总图

```
生产者                商场网关              商场调度器           AICoder代理层         外置AICoder集群        审核节点             消费者池
  │                      │                      │                      │                      │                      │                      │
  │  mall_producer_register                    │                      │                      │                      │                      │
  │─────────────────────>│                      │                      │                      │                      │                      │
  │  返回 ProducerCredential                   │                      │                      │                      │                      │
  │<─────────────────────│                      │                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │  提交蓝图 + 签名      │                      │                      │                      │                      │                      │
  │  mall_gateway_accept_blueprint            │                      │                      │                      │                      │
  │─────────────────────>│                      │                      │                      │                      │                      │
  │                      │  校验签名/权限        │                      │                      │                      │                      │
  │                      │  生成 session_id     │                      │                      │                      │                      │
  │                      │  写入 Pending Pool   │                      │                      │                      │                      │
  │  返回 BlueprintMeta   │                      │                      │                      │                      │                      │
  │<─────────────────────│                      │                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │  mall_blueprint_validate                   │                      │                      │                      │
  │                      │─────────────────────>│                      │                      │                      │                      │
  │                      │  通过：进入阶段 3     │                      │                      │                      │                      │
  │                      │<─────────────────────│                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  mall_aicoder_agent_compose_request          │                      │                      │
  │                      │                      │─────────────────────>│                      │                      │                      │
  │                      │                      │  生成 RequestPacket   │                      │                      │                      │
  │                      │                      │<─────────────────────│                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  mall_aicoder_agent_invoke(config, packet)    │                      │                      │
  │                      │                      │─────────────────────>│                      │                      │                      │
  │                      │                      │                      │  HTTP POST /design   │                      │                      │
  │                      │                      │                      │─────────────────────>│                      │                      │
  │                      │                      │                      │                      │  生成设计草案         │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │                      │  返回 ResponsePacket │                      │                      │
  │                      │                      │                      │<─────────────────────│                      │                      │
  │                      │                      │  返回 response        │                      │                      │                      │
  │                      │                      │<─────────────────────│                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  mall_aicoder_agent_parse_response            │                      │                      │
  │                      │                      │─────────────────────>│                      │                      │                      │
  │                      │                      │  生成 DesignDraft    │                      │                      │                      │
  │                      │                      │<─────────────────────│                      │                      │                      │
  │                      │                      │  写入 Draft Pool      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  【阶段 4：调试】（若编译失败）              │                      │                      │
  │                      │                      │  mall_aicoder_agent_debug_invoke             │                      │                      │
  │                      │                      │─────────────────────>│  HTTP POST /debug    │                      │                      │
  │                      │                      │<─────────────────────│<─────────────────────│                      │                      │
  │                      │                      │  mall_debug_patch_apply                      │                      │                      │
  │                      │                      │  （循环至多 5 轮）    │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  【阶段 5：评估】     │                      │                      │                      │
  │                      │                      │  mall_aicoder_agent_evaluate_invoke           │                      │                      │
  │                      │                      │─────────────────────>│  HTTP POST /evaluate │                      │                      │
  │                      │                      │<─────────────────────│<─────────────────────│                      │                      │
  │                      │                      │  mall_evaluate_report_parse                   │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  【阶段 6：审核编译】  │                      │                      │                      │
  │                      │                      │  mall_audit_compile_design_draft             │                      │                      │
  │                      │                      │─────────────────────────────────────────────────────────────────────>│                      │
  │                      │                      │                      │                      │                      │  人工审核 / 自动 Gate │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │  内部调用            │
  │                      │                      │                      │                      │                      │  mall_compiler_evolve│
  │                      │                      │                      │                      │                      │──────────────────────┤
  │                      │                      │                      │                      │                      │  返回 binary_path    │
  │                      │                      │                      │                      │                      │<─────────────────────│
  │                      │                      │  返回 AuditResult     │                      │                      │                      │
  │                      │                      │<─────────────────────────────────────────────────────────────────────│                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  【阶段 7：部署激活】  │                      │                      │                      │
  │                      │                      │  mall_deploy_to_consumer                      │                      │                      │
  │                      │                      │──────────────────────────────────────────────────────────────────────────────────────────>│
  │                      │                      │  注册句柄到路由表     │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  mall_consumer_activate                       │                      │                      │
  │                      │                      │──────────────────────────────────────────────────────────────────────────────────────────>│
  │                      │                      │                      │                      │                      │                      │  进程就绪
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  mall_genesis_close_session                   │                      │                      │
  │                      │                      │  清理中间态资源       │                      │                      │                      │
  │                      │                      │                      │                      │                      │                      │
  │                      │                      │  【创世纪闭环完成】    │                      │                      │                      │
```

---

## 九、关键设计决策记录（ADR）

| 决策 | 选择 | 理由 |
|------|------|------|
| AICoder 部署方式 | **外置独立服务** | 商场稳定性优先，AICoder 崩溃/升级不得影响商场核心交易 |
| 通信协议 | **HTTP POST 为主，gRPC/SHM桥为选配** | HTTP 最通用，gRPC 用于高性能场景，SHM 桥仅限同机部署 |
| 超时策略 | **硬超时 30s，指数退避重试** | 防止 AICoder 僵死拖垮商场调度器 |
| 熔断机制 | **连续 5 次失败熔断，30s 半开探测** | 快速失败，自动恢复，人工免干预 |
| 审核权归属 | **人类强制审核（audit_mode=0）** | 即使自动 gate 未来开放，人类仍保留最终否决权 |
| 代码安全 | **沙箱编译 + 关键词黑名单** | AICoder 返回的代码不可直接部署，必须过商场沙箱 |
| 幂等性 | **session_id 全局唯一，结果缓存** | 防止网络抖动导致重复生成，浪费算力 |

---

## 十、版本变更日志

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0 | 2026-05-09 | 初始版本，AICoder 内嵌商场进程 |
| **v2.0** | **2026-05-10** | **AICoder 外置重构：** 新增代理层、协议封装、心跳熔断、错误降级、完整端点规范 |

---

> **结语：** 此文档将 AICoder 从商场的「内脏」剥离为「外脑」，商场通过标准化协议持遥控器指挥，既保留了 AICoder 的设计智能，又确保了商场主权的绝对独立。生产者的蓝图流程因此增加了一层「协议转译」与「失效兜底」，但这是为「商场永不宕机」支付的必要架构税。
