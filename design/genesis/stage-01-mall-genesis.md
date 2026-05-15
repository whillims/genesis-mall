User:

创世纪第一阶段


Kimi:

# 创世纪 · 第一阶段设计文档

文档代号： GENESIS-PHASE-01
阶段命名： 原初之光 —— 商场基座的浇筑与创世Worker的觉醒
核心使命： 在绝对虚无中建立第一个可运行的商场实例，完成创世Worker的入驻、授权与首次蓝图提交，验证生产者-商场-AICoder三元链路的贯通性。


## 一、哲学定位与阶段边界

历史唯物主义视角： 任何复杂系统的诞生，都必须经历一个”原始积累”阶段。商场模式3.0的创世纪，不是渐进式演化，而是一次结构性跃迁——从”无商场”到”有商场”的质变。第一阶段不承担繁荣之责，只承担存在之证。

阶段边界宣言： - ✅ 本阶段只关心”第一个生产者能否在商场中存活并提交第一份蓝图” - ❌ 本阶段不关心多生产者竞争、动态评估体系、算力进程池调度 - ❌ 本阶段不启用Worker组调度、不开启SHM矢量管理的完整状态机 - ✅ 本阶段必须建立不可违背的安全隔离铁律雏形，哪怕只隔离一个生产者


## 二、关键词定义（第一阶段特化）


## 三、系统架构图（文本拓扑）

┌─────────────────────────────────────────────────────────────┐
│                      创世Worker (进程外)                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   蓝图生成器   │───→│  授权文档签名  │───→│   提交网关    │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │  UNIX域套接字 / 本地TCP
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      商场基座 (单进程)                         │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                    安全门 (函数级)                      │  │
│  │   verify_worker_identity() → check_whitelist()        │  │
│  │   validate_auth_document() → parse_genesis_contract() │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                  原初SHM管理器                         │  │
│  │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │  │
│  │   │ 授权区   │ │ 蓝图区   │ │ 报告区   │ │ 状态区   │  │  │
│  │   │  1MB    │ │  1MB    │ │  1MB    │ │  1MB    │  │  │
│  │   └─────────┘ └─────────┘ └─────────┘ └─────────┘  │  │
│  │   shm_alloc_region() / shm_write_vector()            │  │
│  │   shm_read_vector()  / shm_lock_region()             │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                  AICoder代理层                         │  │
│  │   aicoder_design()  → 生成设计文档                    │  │
│  │   aicoder_debug()   → 输出调试日志与修复建议            │  │
│  │   aicoder_evaluate() → 产出评估报告与质量评分           │  │
│  └──────────────────────────┬────────────────────────────┘  │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                  创世报告引擎                          │  │
│  │   generate_genesis_report() → 写入报告区               │  │
│  │   notify_worker_completion() → 回调通知                │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘


## 四、核心函数级设计

### 4.1 商场启动函数

/* 函数：商场主入口 —— 创世纪启动器
 * 职责：完成基座初始化、SHM映射、安全门建立、AICoder代理句柄获取
 * 约束：本函数返回0表示创世纪成功，返回负值表示宇宙坍缩
 */
int mall_genesis_bootstrap(void);

/* 子函数拆解 */
int genesis_shm_initialize(size_t region_size);           // 原初SHM分配与清零
int genesis_security_gate_install(const char* whitelist_path); // 安全门建立
int genesis_aicoder_proxy_init(const char* endpoint);   // AICoder代理握手
int genesis_worker_registry_create(void);                 // Worker注册表（容量=1）

参数契约： - region_size: 固定传入 GENESIS_SHM_SIZE (4 * 1024 * 1024) - whitelist_path: 指向创世Worker公钥文件路径，第一阶段允许硬编码为 "./genesis_worker.pub" - endpoint: AICoder服务地址，如 "http://localhost:11434/api" 或本地LLM套接字


### 4.2 安全门函数

/* 函数：生产者身份验证
 * 输入：worker_id (uint64_t), auth_doc (const char* json), auth_sig (const uint8_t* sig)
 * 输出：验证结果码，通过则返回0，同时填充 worker_handle
 */
int security_gate_authenticate(uint64_t worker_id,
                                const char* auth_doc,
                                const uint8_t* auth_sig,
                                worker_handle_t* out_handle);

/* 内部调用链（严格顺序，不可绕过） */
int verify_worker_identity(uint64_t worker_id);        // ID白名单检查
int validate_auth_document(const char* auth_doc);      // JSON Schema校验
int verify_signature(const char* auth_doc,
                     const uint8_t* sig,
                     const uint8_t* pubkey);           // 非对称签名验证
int assign_shm_region(worker_handle_t* handle,
                      shm_region_t* region);           // 分配原初SHM子区

第一阶段铁律： - verify_worker_identity() 内嵌硬编码判断：if (worker_id != WORKER_ID_GENESIS) return -EPERM; - 任何试图以非创世ID进入的请求，直接拒绝并记录到原初日志（SHM状态区尾部512字节）


### 4.3 SHM矢量管理函数（第一阶段简化版）

/* 函数：SHM区域操作 —— 创世纪特化
 * 哲学：SHM不是堆，而是"公共广场"，每个矢量必须有边界、有类型、有生命周期
 */
shm_vector_t* shm_vector_create(shm_region_t* region,
                                const char* vec_name,
                                size_t elem_size,
                                size_t capacity);       // 创建命名矢量

int shm_vector_append(shm_vector_t* vec,
                      const void* data,
                      size_t count);                   // 追加数据

int shm_vector_read(shm_vector_t* vec,
                    size_t offset,
                    void* out_buf,
                    size_t count);                     // 读取数据

int shm_vector_seal(shm_vector_t* vec);                // 封印矢量（只读化，不可逆）

/* 第一阶段映射 */
// 授权区 → 存储 auth_doc 的 shm_vector，命名 "auth_genesis"
// 蓝图区 → 存储 blueprint 的 shm_vector，命名 "blueprint_001"
// 报告区 → 存储 aicoder_report 的 shm_vector，命名 "report_001"
// 状态区 → 存储 mall_status 的 shm_vector，命名 "status_genesis"


### 4.4 AICoder代理函数

/* 函数：AICoder设计代理
 * 输入：blueprint (shm_vector_t*), 包含目标函数描述与契约
 * 输出：design_doc (shm_vector_t*), 结构化设计文档
 * 行为：同步调用，阻塞直到AICoder返回或超时
 */
int aicoder_proxy_design(const shm_vector_t* blueprint,
                         shm_vector_t* design_doc,
                         uint32_t timeout_ms);

/* 函数：AICoder调试代理
 * 输入：code_vector (shm_vector_t*), 待调试代码片段
 * 输出：debug_log (shm_vector_t*), 修复建议与问题定位
 */
int aicoder_proxy_debug(const shm_vector_t* code_vector,
                        shm_vector_t* debug_log,
                        uint32_t timeout_ms);

/* 函数：AICoder评估代理
 * 输入：design_doc + test_scenarios (shm_vector_t*)
 * 输出：eval_report (shm_vector_t*), 含质量评分、风险标记、改进建议
 */
int aicoder_proxy_evaluate(const shm_vector_t* design_doc,
                           const shm_vector_t* test_scenarios,
                           shm_vector_t* eval_report,
                           uint32_t timeout_ms);

第一阶段约束： - 三个函数共享同一个HTTP/套接字连接句柄，不支持并行 - 超时默认 timeout_ms = 300000（5分钟），AICoder思考不设限，但商场必须能感知存活 - 若AICoder无响应，函数返回 -EAICODER_TIMEOUT，商场进入创世悬停状态（不崩溃，等待人工介入）


### 4.5 创世报告引擎函数

/* 函数：生成创世纪最终报告
 * 职责：汇总授权、蓝图、设计、调试、评估全链路信息，形成可审计文档
 */
int genesis_report_compile(const worker_handle_t* worker,
                           const shm_vector_t* auth_vec,
                           const shm_vector_t* blueprint_vec,
                           const shm_vector_t* design_vec,
                           const shm_vector_t* debug_vec,
                           const shm_vector_t* eval_vec,
                           shm_vector_t* final_report);

/* 函数：通知Worker完成
 * 职责：通过回调套接字或信号机制，告知创世Worker"你的蓝图已走完完整生命周期"
 */
int genesis_notify_completion(uint64_t worker_id,
                              const shm_vector_t* final_report,
                              int status_code);


## 五、数据结构定义（C风格，无OOP）

/* 创世纪常量 */
#define WORKER_ID_GENESIS      0x1ULL
#define GENESIS_SHM_SIZE       (4 * 1024 * 1024)
#define GENESIS_MAX_WORKERS    1
#define GENESIS_TIMEOUT_MS     300000

/* Worker句柄 —— 纯数据，无方法 */
typedef struct {
    uint64_t    worker_id;
    uint32_t    status;          /* 0=未认证, 1=已入驻, 2=蓝图提交中, 3=处理中, 4=完成 */
    shm_region_t shm_auth;       /* 授权区子区域 */
    shm_region_t shm_blueprint;  /* 蓝图区子区域 */
    shm_region_t shm_report;    /* 报告区子区域 */
    shm_region_t shm_status;     /* 状态区子区域 */
    int         socket_fd;       /* 回调通知套接字 */
} worker_handle_t;

/* SHM区域描述符 */
typedef struct {
    void*       base_addr;
    size_t      size;
    uint32_t    flags;           /* 读/写/封印 */
    char        name[32];
} shm_region_t;

/* SHM矢量 —— 带类型的连续存储 */
typedef struct {
    shm_region_t* parent;
    size_t        elem_size;
    size_t        capacity;
    size_t        count;
    size_t        offset;        /* 在parent中的偏移 */
    uint8_t       sealed;        /* 0=可写, 1=只读 */
    char          name[32];
} shm_vector_t;

/* 商场状态 —— 存于状态区 */
typedef struct {
    uint64_t    boot_timestamp;
    uint32_t    phase;           /* 固定为 1 */
    uint32_t    worker_count;    /* 固定为 0 或 1 */
    uint32_t    last_error_code;
    char        last_error_msg[128];
    uint8_t     aicoder_alive;   /* 心跳标志 */
} mall_status_t;


## 六、调用流程与参数设置（创世纪时序）

[时序0] 系统启动
    mall_genesis_bootstrap()
    ├── genesis_shm_initialize(4194304)
    ├── genesis_security_gate_install("./genesis_worker.pub")
    ├── genesis_aicoder_proxy_init("http://aicoder.local:11434")
    └── genesis_worker_registry_create()
    返回 0 → 商场进入 "WAITING_FOR_GENESIS" 状态

[时序1] 创世Worker连接
    security_gate_authenticate(0x1, auth_json, signature, &handle)
    ├── verify_worker_identity(0x1) → 通过
    ├── validate_auth_document(auth_json) → Schema校验通过
    ├── verify_signature(...) → 签名验证通过
    └── assign_shm_region(&handle, &genesis_shm)
    返回 0 → Worker状态 = 1 (已入驻)

[时序2] 蓝图提交
    shm_vector_create(handle.shm_blueprint, "blueprint_001", 1, 1048576)
    shm_vector_append(blueprint_vec, blueprint_json, blueprint_len)
    shm_vector_seal(blueprint_vec)  ← 封印，Worker不可篡改
    Worker状态 = 2 (蓝图提交中)

[时序3] AICoder设计阶段
    aicoder_proxy_design(blueprint_vec, design_vec, 300000)
    ├── 读取蓝图区 "blueprint_001"
    ├── 调用AICoder /design 接口
    ├── 等待响应
    └── 写入报告区子矢量 "design_doc_001"
    Worker状态 = 3 (处理中)

[时序4] AICoder调试阶段（可选，若蓝图含代码片段）
    aicoder_proxy_debug(code_vec, debug_vec, 300000)

[时序5] AICoder评估阶段
    aicoder_proxy_evaluate(design_vec, test_vec, eval_vec, 300000)

[时序6] 报告生成与通知
    genesis_report_compile(&handle, ..., &final_report)
    genesis_notify_completion(0x1, &final_report, 0)
    Worker状态 = 4 (完成)
    商场状态 → "GENESIS_COMPLETE"


## 七、测试场景与验收标准

### 测试场景1：创世Worker正常入驻

输入： 正确的 worker_id=0x1，有效授权文档，匹配签名
预期： security_gate_authenticate() 返回0，SHM四区正确分配，状态区记录 worker_count=1
验收： 通过

### 测试场景2：非法Worker闯入

输入： worker_id=0x2 或任意伪造ID
预期： verify_worker_identity() 返回 -EPERM，状态区记录 last_error_code=0x01，错误信息 "GENESIS: WORKER_ID_NOT_IN_WHITELIST"
验收： 通过（安全铁律验证）

### 测试场景3：蓝图封印不可篡改

输入： 蓝图提交后，Worker尝试再次 shm_vector_append()
预期： shm_vector_seal() 已设置 sealed=1，追加操作返回 -EROFS
验收： 通过（行为边界验证）

### 测试场景4：AICoder超时悬停

输入： AICoder服务端人为断开网络
预期： aicoder_proxy_design() 返回 -EAICODER_TIMEOUT，商场不崩溃，状态区 aicoder_alive=0，进入悬停态等待人工介入
验收： 通过（容错验证）

### 测试场景5：端到端创世纪闭环

输入： 完整蓝图（含函数描述、输入输出契约、测试场景诉求）
预期： 时序0→6全部成功执行，最终报告区包含：设计文档、调试日志、评估报告、创世摘要
验收： 通过（创世纪完成标志）


## 八、第一阶段输出产物清单


## 九、下一阶段预告（Phase-02 边界）

当第一阶段的 GENESIS_COMPLETE 信号亮起，商场便从”存在”迈向”繁殖”。第二阶段将解锁： - Worker组调度：允许多个生产者排队入驻，引入简单的FIFO调度器 - SHM矢量状态机：从简化版升级为完整形式化验证版本 - AICoder流水线：设计/调试/评估三阶段异步化，引入任务队列 - 动态评估体系雏形：对AICoder产出进行首次量化评分

但在此之前，第一阶段必须被严格验收。没有稳固的基座，任何繁荣都是幻象。


文档状态： 待审核
审核要点： 是否存在隐式OOP设计？SHM管理是否足够原子？AICoder代理的同步阻塞策略是否符合创世纪哲学？



| 关键词 | 定义 | 第一阶段约束 |
| --- | --- | --- |
| 创世Worker | 第一个进入商场的生产者，拥有提交蓝图的原始权利，是后续所有Worker的”基因模板” | 仅允许1个实例存活，ID固定为 WORKER_ID_GENESIS = 0x1 |
| 商场基座 | 运行时的最小内核，包含SHM根区、安全门、AICoder代理句柄 | 单进程模式，无守护进程，无故障恢复 |
| 原初SHM | 第一阶段分配的共享内存空间，分为：授权区、蓝图区、报告区、状态区 | 固定大小 GENESIS_SHM_SIZE = 4MB，物理连续 |
| 安全门 | 生产者进入商场的唯一通道，执行编译权验证与行为级模型审查 | 仅支持白名单机制，白名单内唯一条目即创世Worker的公钥指纹 |
| AICoder代理 | 商场内部代理AICoder能力的函数句柄，负责将蓝图转化为设计文档、调试报告、评估结果 | 本阶段为同步阻塞调用，不支持并发流水线 |
| 蓝图 | 创世Worker提交的结构化意图文档，包含：目标函数描述、输入输出契约、测试场景诉求 | 必须为JSON格式，Schema版本 v0.1-genesis |
| 创世纪授权文档 | 生产者与商场之间的契约，明确编译权归属、SHM使用范围、行为边界 | 第一阶段采用硬编码授权，不启用动态协商 |


| 产物名称 | 存储位置 | 格式 | 说明 |
| --- | --- | --- | --- |
| genesis_mall_binary | 编译输出 | ELF/可执行文件 | 商场基座可执行体 |
| genesis_worker.pub | 部署目录 | PEM/二进制 | 创世Worker公钥 |
| genesis_auth_template.json | 文档附件 | JSON | 授权文档Schema v0.1 |
| genesis_blueprint_schema.json | 文档附件 | JSON | 蓝图提交Schema v0.1 |
| shm_layout_genesis.md | 文档附件 | Markdown | 4MB原初SHM布局图 |
| genesis_report_001.bin | SHM报告区 | 二进制矢量 | 第一份创世纪完整报告 |
| genesis_test_log.txt | 测试输出 | 文本 | 五大测试场景执行日志 |

