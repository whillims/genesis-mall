# BP-0008-aicoder-gateway.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**：BP-0008  
> **产品名称**：aicoder-gateway（AICoder外置网关）  
> **产品类型**：核心服务产品  
> **优先级**：P1  
> **依赖**：BP-0004-shm-manager, BP-0007-error-guard  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`aicoder-gateway` 是商场与**外置AICoder**之间的唯一官方接口。由于AICoder目前外置（非嵌入商场进程），本网关负责：

1. **蓝图提交**：将生产者撰写的蓝图文件传输给AICoder
2. **授权文件接收**：接收AICoder审查、评估、测试后生成的授权文件（含代码）
3. **编译结果回调**：接收AICoder编译结果（成功/失败/警告）
4. **状态同步**：监控AICoder进程健康状态，处理AICoder失联或崩溃

通信方式采用**文件系统监视 + 信号通知**的混合模型：
- 商场将蓝图写入约定目录，`aicoder-gateway` 通过 `inotify`/`watchdog` 检测新文件
- AICoder将授权文件写入输出目录，网关检测并加载
- 紧急状态通过Unix信号或SHM状态位同步

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 通信隔离 | 商场与AICoder之间禁止直接socket/管道连接，仅通过文件系统与SHM状态位通信 |
| 文件安全 | 蓝图文件与授权文件必须存储于约定路径，文件名符合命名规范 |
| 权限控制 | 只有持有有效生产者ID的Worker才能提交蓝图，商场主进程拥有最终加载权 |
| 失败策略 | AICoder崩溃或超时时，网关标记其为不可用，错误守卫记录事件，商场继续运行 |
| 不可回退 | 授权文件一旦加载，旧版本自动归档，不支持运行时回退（需重新提交蓝图） |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **蓝图目录** | 约定路径：`{mall_workspace}/blueprints/incoming/` |
| **授权目录** | 约定路径：`{mall_workspace}/blueprints/authorized/` |
| **归档目录** | 约定路径：`{mall_workspace}/blueprints/archive/` |
| **AICoder状态段** | SHM子段，记录AICoder进程PID、最后活跃时间、当前任务 |
| **授权文件** | JSON格式，包含 `blueprint_id`、`code_payload`、`test_results`、`signature` |
| **签名验证** | 商场使用预置公钥验证授权文件完整性（防止篡改） |
| **编译队列** | SHM中的FIFO队列，记录待编译的授权文件标识 |
| **AICoder心跳** | AICoder定期写入SHM的时间戳，超时则视为失联 |

---

## 4. 接口定义

### 4.1 蓝图提交（生产者 → 商场 → AICoder）

```c
// 生产者提交蓝图文件
int aicoder_submit_blueprint(
    SharedMemory* root,
    uint16_t producer_id,         // 提交者Worker ID
    const char* blueprint_path,   // 蓝图文件绝对路径（已存在于文件系统）
    uint8_t priority              // 审查优先级：0=普通, 1=紧急
);

// 商场查询蓝图审查进度
int aicoder_query_progress(
    SharedMemory* root,
    const char* blueprint_id,
    uint8_t* out_status,          // 0=PENDING, 1=REVIEWING, 2=TESTING, 3=APPROVED, 4=REJECTED
    uint32_t* out_progress_pct,   // 进度百分比
    char* out_message             // 状态描述（如 "单元测试通过 7/10"）
);
```

### 4.2 授权文件接收（AICoder → 商场）

```c
// 网关检测并加载新授权文件（由商场调度器周期性调用）
int aicoder_poll_authorized(
    SharedMemory* root,
    uint32_t* out_new_files,     // 本周期新接收的授权文件数
    uint32_t* out_loaded_files    // 本周期成功加载数
);

// 验证授权文件签名与完整性
int aicoder_verify_authfile(
    SharedMemory* root,
    const char* authfile_path,
    uint8_t* out_valid            // 0=无效, 1=有效
);

// 加载授权文件到商场（编译并注册函数）
int aicoder_load_authfile(
    SharedMemory* root,
    const char* authfile_path,
    uint16_t* out_product_id      // 出参：分配的产品ID
);
```

### 4.3 AICoder进程管理

```c
// 启动外置AICoder进程（由商场main调用）
int aicoder_process_spawn(
    SharedMemory* root,
    const char* aicoder_executable, // AICoder可执行文件路径
    const char* workspace_path,     // 工作目录
    pid_t* out_aicoder_pid
);

// 监控AICoder心跳
int aicoder_health_check(
    SharedMemory* root,
    uint32_t timeout_ms,
    uint8_t* out_alive             // 0=死亡, 1=存活
);

// 优雅关闭AICoder进程
int aicoder_process_shutdown(
    SharedMemory* root,
    uint32_t grace_period_ms
);
```

---

## 5. SHM数据结构设计

### 5.1 AICoder状态段（专用子段：{mall_name}-aicoder-status）

```c
struct aicoder_status {
    uint64_t magic;              // "AICODER" = 0x4149434F44455200
    uint32_t version;
    pid_t    aicoder_pid;        // AICoder操作系统PID
    uint8_t  status;             // 0=DOWN, 1=STARTING, 2=READY, 3=BUSY, 4=ERROR
    uint64_t last_heartbeat;     // 最后心跳时间戳
    uint64_t start_time;         // 启动时间戳
    char     current_task[32];   // 当前处理任务（蓝图ID）
    uint32_t tasks_completed;    // 历史完成任务数
    uint32_t tasks_failed;       // 历史失败任务数
    uint8_t  reserved[8];
};
```

### 5.2 编译队列（位于根段 + 0x12000）

```c
struct compile_queue_entry {
    char     blueprint_id[32];   // 蓝图标识
    uint16_t producer_id;        // 提交者
    uint8_t  priority;           // 优先级
    uint64_t submit_time;        // 提交时间戳
    uint8_t  status;             // 0=PENDING, 1=COMPILING, 2=DONE
};
// 环形队列，默认容量 64 项
```

### 5.3 授权文件元数据缓存（位于根段 + 0x14000）

```c
struct authfile_cache_entry {
    char     blueprint_id[32];
    char     authfile_path[64];  // 授权文件路径
    uint64_t load_time;
    uint16_t product_id;         // 加载后分配的产品ID
    uint8_t  status;             // 0=LOADED, 1=ACTIVE, 2=ARCHIVED, 3=REVOKED
    uint8_t  reserved[5];
};
// 默认容量 128 项
```

---

## 6. 函数详细设计

### 6.1 `aicoder_submit_blueprint`

**逻辑流程**：
1. 校验 `root` 非空，`blueprint_path` 存在且可读，`producer_id` 有效
2. 校验蓝图文件名符合规范 `BP-{编号}-{功能名}.md`
3. 读取AICoder状态段，若 `status == DOWN` 或 `ERROR`，返回 `-EAGAIN`（AICoder不可用）
4. 将蓝图文件复制到 `blueprints/incoming/` 目录，文件名保持不变
5. 在编译队列中追加新项：`blueprint_id`、`producer_id`、`priority`、`submit_time`
6. 更新AICoder状态 `current_task`
7. 可选：向AICoder进程发送 `SIGUSR1` 通知新任务（若AICoder支持）
8. 返回0

**返回值**：
- `0`：成功提交
- `-EINVAL`：参数非法或文件名不规范
- `-EAGAIN`：AICoder不可用
- `-ENOSPC`：编译队列已满

### 6.2 `aicoder_poll_authorized`

**逻辑流程**：
1. 扫描 `blueprints/authorized/` 目录，按修改时间排序
2. 对每个新文件（未在授权缓存中）：
   a. 调用 `aicoder_verify_authfile` 验证签名
   b. 若无效，移动到 `blueprints/rejected/` 并记录错误
   c. 若有效，调用 `aicoder_load_authfile` 编译加载
   d. 加载成功后，移动原蓝图到 `blueprints/archive/`，授权文件保留
3. 更新统计计数器
4. 返回新文件数与加载成功数

**返回值**：
- `0`：轮询完成
- 出参包含统计

### 6.3 `aicoder_verify_authfile`

**逻辑流程**：
1. 打开授权文件（JSON格式）
2. 校验必填字段：`blueprint_id`、`code_payload`、`test_results`、`signature`
3. 校验 `blueprint_id` 存在于商场已注册蓝图列表
4. 使用预置公钥验证 `signature`（签名内容为 `code_payload` + `test_results` 的哈希）
5. 校验 `test_results` 中所有测试用例状态为 `PASS`
6. 返回验证结果

**返回值**：
- `0`：验证完成，`out_valid` 指示结果
- `-EIO`：文件读取失败

### 6.4 `aicoder_load_authfile`

**逻辑流程**：
1. 读取授权文件 `code_payload` 字段（Base64编码的函数代码）
2. 解码为字节码/源代码（取决于商场编译策略）
3. 在商场进程空间中动态编译加载（如使用 `dlopen` 或Python `exec` 的受控版本）
4. 分配新 `product_id`
5. 在授权缓存中注册
6. 调用 `intent_router` 注册产品能力（若蓝图包含能力标签）
7. 返回 `product_id`

**返回值**：
- `0`：成功
- `-ENOEXEC`：代码编译失败
- `-EACCES`：权限不足

---

## 7. 测试场景

### 7.1 单元测试：蓝图提交与接收

```
测试名：test_aicoder_submit_and_poll
步骤：
  1. 启动AICoder进程（模拟器）
  2. 生产者提交蓝图 BP-TEST-0001.md
  3. 模拟AICoder生成授权文件 AUTH-BP-TEST-0001.json
  4. 网关轮询授权目录
断言：
  - aicoder_submit_blueprint 返回0
  - aicoder_poll_authorized 检测到1个新文件
  - aicoder_verify_authfile 返回 valid=1
  - 授权缓存中存在对应项
```

### 7.2 异常测试：AICoder失联

```
测试名：test_aicoder_crash_recovery
步骤：
  1. 启动AICoder进程
  2. 提交蓝图
  3. 强制杀死AICoder进程（SIGKILL）
  4. 等待超时阈值
  5. 调用 aicoder_health_check
断言：
  - out_alive == 0
  - 错误日志记录 FATAL 事件：AICoder心跳超时
  - 商场主进程继续运行（未崩溃）
  - 新蓝图提交返回 -EAGAIN
```

### 7.3 安全测试：授权文件篡改

```
测试名：test_aicoder_tampered_authfile
步骤：
  1. 生成有效授权文件
  2. 篡改 code_payload 中的某个函数名
  3. 保持原签名不变
  4. 调用 aicoder_verify_authfile
断言：
  - out_valid == 0
  - 错误日志记录安全警告：签名验证失败
  - 篡改文件被移动到 rejected 目录
```

### 7.4 集成测试：端到端蓝图加载

```
测试名：test_aicoder_e2e_blueprint_loading
步骤：
  1. 商场main启动，加载BP-0008
  2. 启动外置AICoder
  3. 生产者提交BP-0003-stdio-pipe.md
  4. AICoder审查、测试、生成授权文件
  5. 网关轮询并加载
  6. 检查商场是否注册了新函数
断言：
  - 授权文件加载成功
  - 新函数可通过SHM调用
  - 产品ID在有效范围
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager, BP-0007-error-guard |
| 后置蓝图 | 所有需要AICoder生成代码的BP（实际上所有业务BP） |
| AICoder授权 | 本BP本身也需AICoder生成授权文件 `AUTH-BP-0008.json`，形成自举 |
| 商场加载 | 由商场 `main()` 在创世第5阶段加载，AICoder进程启动后激活 |

---

## 9. 附录：文件系统约定

```
{mall_workspace}/
├── blueprints/
│   ├── incoming/          # 生产者提交的原始蓝图（*.md）
│   ├── authorized/        # AICoder生成的授权文件（*.json）
│   ├── archive/           # 已加载蓝图的归档（*.md 原始文件）
│   └── rejected/          # 未通过审查的文件（*.md + *.rej）
├── aicoder/
│   ├── executable/        # AICoder可执行文件
│   └── logs/              # AICoder本地日志（商场不直接读取）
└── keys/
    └── mall_public.pem    # 商场公钥（用于验证授权文件签名）
```

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
