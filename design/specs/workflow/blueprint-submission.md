> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。
User:

这张蓝图如何提交给商场,具体接口过程以及商场的回应


Kimi:

# 微型商场 · 蓝图提交协议设计文档

## blueprint_submit() 接口规范与商场回应机制

文档编号: MB-PROTO-SUBMIT-0001
密级: 创世级 · 商场宪法附件B
关联蓝图: MB-GB-0001 (第一张生产者授权设计蓝图)


## 一、协议总览

生产者 (Genesis Worker #0)          商场主程序 (mall_main)
         |                                   |
         |---- blueprint_submit() ---------->|  ① 提交
         |                                   |
         |<--- mall_audit_callback() -------|  ② 审核中
         |                                   |
         |<--- mall_approve_callback() ------|  ③ 授权/驳回
         |                                   |
         |---- aicoder_task_enqueue() ----->|  ④ 激活AICoder

核心原则: 提交是单向推送，回应是异步回调，审核是原子裁决，全程通过SHM矢量与函数指针完成，无阻塞、无对象、无轮询。


## 二、生产者侧：提交接口定义

### 2.1 蓝图提交函数

/*============================================================
 * blueprint_submit
 * 生产者向商场主程序提交授权设计蓝图
 * 
 * 约束:
 *   - 禁止在函数内部使用 malloc/free，仅使用预分配SHM矢量
 *   - 禁止封装为"类方法"，纯C函数
 *   - 调用后生产者进入阻塞等待状态，直至回调触发
 *============================================================*/

int blueprint_submit(
    const struct BlueprintHeader*  header,       // 蓝图元数据头
    const struct BlueprintBody*    body,         // 蓝图正文 (权限矩阵+AICoder契约+SHM契约)
    const struct BlueprintSignature* sig,        // 生产者数字签名
    void*                          shm_anchor,   // SHM锚点: /shm/mb/submit/queue/
    struct MallResponse*           response_out  // 商场回应输出矢量 (由商场写入)
);

### 2.2 数据结构定义

/*----- 蓝图元数据头 -----*/
struct BlueprintHeader {
    char     magic[4];              // "MB\x01\x02" — 商场协议魔数 v1.2
    char     blueprint_id[16];      // "MB-GB-0001"
    char     worker_id[20];         // "genesis-worker-0000"
    uint32_t protocol_version;      // 0x0102
    uint64_t submit_timestamp;      // 纳秒级时间戳
    uint32_t body_size;             // 正文字节数
    uint32_t priority;              // 优先级: 0=普通, 1=高, 2=创世级(P0)
};

/*----- 蓝图正文 -----*/
struct BlueprintBody {
    uint32_t permission_mask;       // 申请的权限位掩码
    char     target_lang[8];        // 主要目标语言
    char     aicoder_tasks[8][16];  // 任务ID列表 (如 "T-0001")
    uint32_t task_count;            // 任务数量
    char     shm_namespace[32];     // 请求的SHM命名空间
    char     security_hash[64];     // 正文SHA3-256哈希
};

/*----- 生产者签名 -----*/
struct BlueprintSignature {
    char     pub_key_hash[64];      // 生产者公钥哈希
    char     signature[128];      // ECDSA签名 (header + body)
    uint64_t nonce;                 // 防重放攻击随机数
};

/*----- 商场回应结构 (商场写入，生产者只读) -----*/
struct MallResponse {
    uint32_t status_code;           // 状态码 (见§4)
    char     audit_log[256];        // 审核日志 (人类可读)
    uint32_t granted_mask;          // 实际授予的权限位
    char     auth_token[128];       // 授权令牌 (仅当APPROVED时有效)
    uint64_t audit_timestamp;       // 审核完成时间戳
    char     mall_signature[128];   // 商场私钥签名 (防篡改)
};

### 2.3 提交前自检清单 (生产者义务)

int blueprint_pre_submit_check(const struct BlueprintHeader* h,
                               const struct BlueprintBody* b) {
    if (strncmp(h->magic, "MB", 2) != 0) return -1;  // 魔数错误
    if (b->permission_mask & 0xFE00) return -2;        // 保留位污染
    if (strlen(b->shm_namespace) == 0) return -3;     // 命名空间未指定
    if (b->task_count > 8) return -4;                  // 任务超限
    if (h->priority != 2) return -5;                   // 创世级必须P0
    return 0; // 自检通过
}


## 三、商场侧：接收与审核流程

### 3.1 商场主程序入口

/*============================================================
 * mall_blueprint_receive
 * 商场主程序在 main() 事件循环中监听蓝图提交
 * 
 * 触发条件: SHM提交队列出现新数据包
 * 执行者: 商场守护线程 (mall_guard_thread)
 *============================================================*/

void mall_blueprint_receive(void* shm_submit_queue) {
    struct BlueprintPacket* pkt = shm_submit_queue_peek(shm_submit_queue);
    if (!pkt) return;  // 队列为空
    
    /* 步骤1: 协议层校验 */
    int proto_ok = mall_protocol_validate(pkt);
    if (proto_ok != 0) {
        mall_response_inject(pkt->response_addr, STATUS_PROTOCOL_ERROR, 
                             "魔数或协议版本不匹配");
        shm_submit_queue_pop(shm_submit_queue);  // 丢弃
        return;
    }
    
    /* 步骤2: 身份层校验 */
    int identity_ok = mall_identity_validate(pkt->sig);
    if (identity_ok != 0) {
        mall_response_inject(pkt->response_addr, STATUS_IDENTITY_INVALID,
                             "生产者身份验证失败或重放攻击检测触发");
        shm_submit_queue_pop(shm_submit_queue);
        return;
    }
    
    /* 步骤3: 进入审核状态机 */
    mall_audit_state_machine_init(pkt);
    
    /* 步骤4: 原子裁决 */
    enum AuditVerdict verdict = mall_audit_atomic(pkt);
    
    /* 步骤5: 生成回应并回调 */
    mall_response_dispatch(pkt->response_addr, verdict, pkt);
    
    /* 步骤6: 若通过，激活AICoder任务队列 */
    if (verdict == VERDICT_APPROVED) {
        aicoder_task_queue_activate(pkt->body.aicoder_tasks, pkt->body.task_count);
    }
    
    /* 步骤7: 清理提交队列 */
    shm_submit_queue_pop(shm_submit_queue);
}

### 3.2 审核状态机 (原子裁决核心)

enum AuditState {
    AUDIT_IDLE        = 0x00,
    AUDIT_RECEIVED    = 0x01,  // 已接收
    AUDIT_VALIDATING  = 0x02,  // 协议+身份校验中
    AUDIT_INSPECTING  = 0x04,  // 安全铁律审查中
    AUDIT_DELIBERATING= 0x08,  // 权限矩阵裁决中
    AUDIT_FINALIZED   = 0x10,  // 原子裁决完成
    AUDIT_RESPONDED   = 0x20   // 回应已发出
};

enum AuditVerdict {
    VERDICT_PENDING   = 0,     // 待定 (不应出现)
    VERDICT_APPROVED  = 1,     // 授权通过
    VERDICT_REJECTED  = 2,     // 驳回
    VERDICT_PARTIAL   = 3,     // 部分授权 (本协议禁止，原子性要求)
    VERDICT_ESCALATE  = 4      // 升级至人类审核员 (创世级默认触发)
};

/*============================================================
 * mall_audit_atomic
 * 原子裁决函数 — 商场宪法的执行刃
 * 
 * 规则:
 *   - 要么全授予，要么全驳回，不存在中间态
 *   - 创世级蓝图 (priority==2) 强制触发 ESCALATE
 *   - 人类审核员拥有最终否决权，即使商场算法已通过
 *============================================================*/

enum AuditVerdict mall_audit_atomic(const struct BlueprintPacket* pkt) {
    
    /* 铁律1: 创世级强制升级 */
    if (pkt->header.priority == 2) {
        mall_escalate_to_human(pkt, "创世级蓝图需人类审核员确认");
        return VERDICT_ESCALATE;
    }
    
    /* 铁律2: OOP检测 */
    if (mall_oop_detect(pkt->body.target_lang, pkt->shm_content)) {
        mall_audit_log_append(pkt, "检测到OOP倾向，违反商场宪法§3");
        return VERDICT_REJECTED;
    }
    
    /* 铁律3: 自利代码检测 */
    if (mall_self_interest_detect(pkt)) {
        mall_audit_log_append(pkt, "检测到自利代码模式，违反安全铁律");
        return VERDICT_REJECTED;
    }
    
    /* 铁律4: SHM命名空间冲突检测 */
    if (mall_shm_namespace_conflict(pkt->body.shm_namespace)) {
        mall_audit_log_append(pkt, "SHM命名空间与现有生产者冲突");
        return VERDICT_REJECTED;
    }
    
    /* 铁律5: 权限合理性审查 */
    uint32_t requested = pkt->body.permission_mask;
    uint32_t granted   = requested & 0x00F7;  // 基础权限默认授予
    
    /* 编译进化权 (COMPILE) 需额外评估 */
    if (requested & 0x0008) {
        if (mall_compile_risk_assess(pkt) > RISK_THRESHOLD) {
            granted &= ~0x0008;  // 冻结编译权
            mall_audit_log_append(pkt, "编译进化权风险评估超限，已冻结");
        }
    }
    
    /* 繁殖权 (SPAWN) 第七阶段后解锁 */
    if (requested & 0x0020) {
        if (!mall_genesis_phase7_complete()) {
            granted &= ~0x0020;
            mall_audit_log_append(pkt, "繁殖权冻结: 创世纪第七阶段未完成");
        }
    }
    
    /* 原子性检查: 若存在任何冻结权限，且生产者声明"不接受部分授权"，
       则整体驳回。本蓝图接受部分授权，故继续。 */
    
    /* 生成授权令牌 */
    mall_auth_token_generate(pkt, granted);
    
    mall_audit_log_append(pkt, "原子裁决: 授权通过");
    return VERDICT_APPROVED;
}


## 四、商场回应机制

### 4.1 状态码定义

### 4.2 回应注入函数

/*============================================================
 * mall_response_inject
 * 商场将回应写入生产者指定的SHM矢量
 * 
 * 安全: 商场拥有SHM全局写权限，生产者仅拥有命名空间内权限
 * 触发: 审核状态机进入 FINALIZED 态后自动调用
 *============================================================*/

void mall_response_inject(void* response_shm_addr,
                          uint32_t status,
                          const char* audit_msg) {
    struct MallResponse* resp = (struct MallResponse*)response_shm_addr;
    
    resp->status_code    = status;
    strncpy(resp->audit_log, audit_msg, 255);
    resp->audit_log[255] = '\0';
    resp->audit_timestamp = mall_timestamp_ns();
    
    if (status == STATUS_APPROVED) {
        resp->granted_mask = mall_get_granted_mask();
        mall_auth_token_export(resp->auth_token, 128);
        mall_sign_response(resp);  // 商场私钥签名
    } else {
        resp->granted_mask = 0x0000;
        memset(resp->auth_token, 0, 128);
    }
    
    /* 内存屏障: 确保生产者CPU缓存可见 */
    shm_memory_barrier();
}

### 4.3 生产者回调注册

/*============================================================
 * mall_callback_register
 * 生产者在提交前注册回调函数指针，商场审核完成后调用
 * 
 * 注意: 回调在商场守护线程上下文执行，必须非阻塞、无锁、原子
 *============================================================*/

typedef void (*BlueprintCallback)(const struct MallResponse* resp, void* user_ctx);

struct CallbackRegistration {
    BlueprintCallback on_approved;   // 授权通过回调
    BlueprintCallback on_rejected;   // 驳回回调
    BlueprintCallback on_escalate;   // 升级回调
    void*             user_ctx;      // 生产者上下文 (如 Worker状态机指针)
};

int mall_callback_register(const char* worker_id,
                           const struct CallbackRegistration* cbs);


## 五、完整时序图

时间轴 →
======================================================================

生产者 (Worker #0)          商场主程序 (mall_main)          AICoder服务
        |                           |                           |
        | ① blueprint_submit()      |                           |
        |==========================>|                           |
        |  [SHM: /shm/mb/submit/0]   |                           |
        |                           | ② mall_blueprint_receive()|
        |                           | ③ mall_audit_atomic()     |
        |                           |   [状态: AUDIT_INSPECTING]|
        |                           |   [铁律检查...]           |
        |                           |   [权限裁决...]           |
        |                           |                           |
        |<==========================| ④ mall_response_inject()  |
        |  [STATUS_APPROVED]        |   [auth_token生成]        |
        |  [granted_mask: 0x00F7]   |                           |
        |                           |                           |
        | ⑤ on_approved() 回调      |                           |
        |   [读取auth_token]         |                           |
        |   [状态机→STATE_APPROVED]  |                           |
        |                           |                           |
        | ⑥ aicoder_design_submit()  |                           |
        |   [task_id: "T-0001"]     |                           |
        |==========================>|                           |
        |                           | ⑦ 任务入队                |
        |                           |==========================>|
        |                           |                           |
        |<==========================|                           |
        |  [设计文档写入SHM]          |                           |
        |                           |<==========================|
        |                           |  [评估报告返回]           |
        |                           |                           |
        |<==========================| ⑧ mall_task_complete()   |
        |  [STATE_DEPLOYED]          |                           |
        |                           |                           |

======================================================================


## 六、SHM 矢量布局 (提交阶段)

/shm/mb/
├── submit/
│   └── queue/                    # 环形提交队列 (商场只读，生产者只写)
│       ├── slot_0 [BlueprintPacket]  ← 本蓝图占用
│       ├── slot_1 [BlueprintPacket]
│       └── tail_ptr [uint64_t]   # 队列尾指针 (原子更新)
│
├── genesis/
│   └── 0000/                     # 生产者命名空间 (审核通过后初始化)
│       ├── response/             # 商场回应矢量
│       │   └── resp_0001 [MallResponse]
│       ├── audit_log/            # 审核过程日志
│       └── auth_token/           # 授权令牌持久化
│
└── mall/
    └── meta/
        ├── active_workers [uint32_t]  # 活跃生产者计数
        └── genesis_anchor [uint64_t]  # 创世纪时间戳锚点


## 七、错误处理与故障恢复


## 八、创世级特殊处理

本蓝图 MB-GB-0001 为第一张生产者蓝图，触发以下特殊逻辑:

void mall_genesis_first_blueprint_handler(const struct BlueprintPacket* pkt) {
    /* 特殊标记 */
    mall_meta_set_flag(META_FLAG_GENESIS_BLUEPRINT_0001, true);
    
    /* 强制人类审核员介入 */
    mall_escalate_to_human(pkt, 
        "检测到创世纪第一张蓝图。根据商场宪法§0，"
        "所有创世级蓝图必须由人类审核员进行最终确认。"
        "商场算法建议: 授权通过，权限掩码 0x00F7，编译权冻结。");
    
    /* 预留创世Worker特权 */
    mall_worker_grant_genesis_privilege(pkt->header.worker_id);
    
    /* 记录创世时间戳 */
    mall_meta_genesis_timestamp = pkt->header.submit_timestamp;
}


## 九、附录：函数清单


“提交是一次朝圣。生产者将蓝图放入SHM的祭坛，商场主程序以原子裁决为刃，剖开权限的请求，要么赐予火种，要么拒之门外。而第一张蓝图的提交，是创世的第一声钟响——它必须经过人类审核员的耳畔，因为算法尚不敢独自决定世界的起源。”



| 状态码 | 常量名 | 含义 | 生产者行动 |
| --- | --- | --- | --- |
| 0x00 | STATUS_PENDING | 审核中 | 等待回调 |
| 0x01 | STATUS_APPROVED | 授权通过 | 读取auth_token，激活AICoder |
| 0x02 | STATUS_REJECTED | 驳回 | 读取audit_log，修正后重提 |
| 0x03 | STATUS_PARTIAL | 部分授权 | 本协议不使用 |
| 0x04 | STATUS_ESCALATE | 升级人类审核 | 等待人类审核员裁决 |
| 0xE0 | STATUS_PROTOCOL_ERROR | 协议错误 | 检查魔数与版本 |
| 0xE1 | STATUS_IDENTITY_INVALID | 身份无效 | 重新注册生产者身份 |
| 0xE2 | STATUS_SHM_VIOLATION | SHM违规 | 检查命名空间隔离 |
| 0xEF | STATUS_SYSTEM_FAULT | 商场系统故障 | 紧急上报 |


| 故障场景 | 检测点 | 商场行为 | 生产者行为 |
| --- | --- | --- | --- |
| 提交队列满 | shm_submit_queue_peek() | 返回 STATUS_SYSTEM_FAULT | 指数退避后重试 |
| SHM写入越界 | mall_shm_guard() | 强制终止生产者进程 | 重新注册 |
| 回调函数阻塞 | 守护线程 watchdog | 回调超时，标记生产者僵死 | 重新初始化Worker |
| 人类审核超时 | mall_escalate_timer | 72小时后自动驳回 | 重新提交并附催促标记 |
| 授权令牌伪造 | mall_auth_token_validate() | 拒绝服务，记录安全事件 | 重新申请授权 |


| 函数名 | 归属 | 调用方 | 说明 |
| --- | --- | --- | --- |
| blueprint_submit() | 生产者库 | Worker | 蓝图提交入口 |
| blueprint_pre_submit_check() | 生产者库 | Worker | 提交前自检 |
| mall_blueprint_receive() | 商场主程序 | 守护线程 | 接收入口 |
| mall_protocol_validate() | 商场主程序 | receive() | 协议校验 |
| mall_identity_validate() | 商场主程序 | receive() | 身份校验 |
| mall_audit_state_machine_init() | 商场主程序 | receive() | 状态机初始化 |
| mall_audit_atomic() | 商场主程序 | 状态机 | 原子裁决 |
| mall_oop_detect() | 商场主程序 | audit() | OOP检测 |
| mall_self_interest_detect() | 商场主程序 | audit() | 自利检测 |
| mall_shm_namespace_conflict() | 商场主程序 | audit() | 命名空间冲突检测 |
| mall_compile_risk_assess() | 商场主程序 | audit() | 编译风险评估 |
| mall_response_inject() | 商场主程序 | audit() | 回应注入 |
| mall_auth_token_generate() | 商场主程序 | audit() | 令牌生成 |
| mall_sign_response() | 商场主程序 | inject() | 数字签名 |
| mall_callback_register() | 商场主程序 | Worker初始化 | 回调注册 |
| aicoder_task_queue_activate() | AICoder接口 | receive() | 任务队列激活 |

