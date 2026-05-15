# BP-0007-error-guard.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**：BP-0007  
> **产品名称**：error-guard（错误守卫与异常恢复）  
> **产品类型**：基础设施产品  
> **优先级**：P0  
> **依赖**：BP-0004-shm-manager, BP-0005-worker-spawn  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`error-guard` 是商场的**免疫系统**，负责捕获、记录、分类、传播与恢复所有运行时异常。它建立了一个覆盖全商场的**错误日志SHM化体系**——所有错误信息（包括信号、退出码、断言失败、内存违规）必须写入专用SHM日志段，而非本地文件或标准错误流。

核心原则：
- **错误即数据**：错误是商场的原生数据产品，可被消费者订阅分析
- **隔离优先**：单个Worker的崩溃不得传染其他Worker或商场主进程
- **可恢复性**：区分可恢复错误（如临时溢出）与致命错误（如内存宪法违反）
- **审计追踪**：每个错误携带完整上下文（时间、Worker ID、蓝图、SHM地址、调用栈摘要）

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 信号处理 | 使用 `signal.signal()` 注册处理器，禁止覆盖商场主进程关键信号 |
| 日志存储 | 仅写入SHM错误日志段，禁止直接写文件或打印到stdout/stderr |
| 传播策略 | 错误向上传播通过SHM状态位，禁止跨进程异常抛出 |
| 恢复策略 | 可恢复错误自动重置状态机；致命错误触发Worker隔离与蓝图吊销 |
| 幂等性 | 同一错误的重复报告必须去重，防止日志溢出 |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **错误等级** | `INFO=0`, `WARNING=1`, `ERROR=2`, `FATAL=3` |
| **错误代码** | 32位无符号整数，高16位为产品ID，低16位为具体错误 |
| **错误日志段** | 专用SHM子段，循环存储错误记录 |
| **错误记录** | 固定128字节的结构化错误描述 |
| **信号映射表** | 将Unix信号（SIGSEGV、SIGTERM等）映射为商场错误代码 |
| **隔离指令** | 写入Worker表的强制终止标记，由 `worker-spawn` 执行 |
| **恢复句柄** | 针对特定错误代码注册的恢复函数指针（存储于SHM配置区） |
| **去重窗口** | 5秒内同一Worker同一错误代码只记录一次 |

---

## 4. 接口定义

### 4.1 错误报告

```c
// 报告错误（由任何Worker或商场服务调用）
int error_report(
    SharedMemory* root,           // SHM根段
    uint16_t reporter_id,         // 报告者Worker ID（0=商场主进程）
    uint32_t error_code,          // 错误代码
    uint8_t severity,             // 等级：0=INFO, 1=WARNING, 2=ERROR, 3=FATAL
    const char* message,          // 错误描述（最大96字符）
    uint64_t context_addr         // 相关SHM地址（可选，0=无）
);

// 批量查询错误日志（由监控消费者调用）
int error_query(
    SharedMemory* root,
    uint8_t min_severity,         // 最小等级过滤
    uint32_t max_records,         // 最大返回记录数
    uint8_t* out_buffer,          // 输出缓冲区（由调用者分配）
    uint32_t* out_returned_count  // 实际返回记录数
);

// 清除错误日志（仅商场主进程可调用）
int error_clear(
    SharedMemory* root,
    uint16_t caller_id            // 必须为0（主进程）
);
```

### 4.2 信号与异常捕获

```c
// 注册商场全局信号处理器（由main调用）
int error_signal_register(
    SharedMemory* root,
    uint8_t* signal_mask          // 位掩码：bit0=SIGINT, bit1=SIGTERM, bit2=SIGSEGV, bit3=SIGABRT, bit4=SIGCHLD
);

// 注册Worker级信号处理器（由worker-spawn在创建Worker后调用）
int error_worker_signal_register(
    SharedMemory* root,
    uint16_t worker_id,
    uint8_t* signal_mask
);

// 信号分发回调（内部函数）
void error_signal_dispatch(int sig_num, siginfo_t* info, void* context);
```

### 4.3 错误处理与恢复

```c
// 注册可恢复错误的恢复策略（由商场或AICoder配置）
int error_recovery_register(
    SharedMemory* root,
    uint32_t error_code,          // 针对特定错误代码
    uint8_t strategy,             // 0=忽略, 1=重试, 2=重置状态机, 3=重启Worker, 4=吊销蓝图
    uint32_t max_retries          // 最大重试次数
);

// 执行恢复策略（由错误守卫自动调用）
int error_recovery_execute(
    SharedMemory* root,
    uint32_t error_code,
    uint16_t worker_id,
    uint8_t* out_result           // 出参：0=恢复成功, 1=恢复失败, 2=达到最大重试
);

// 吊销蓝图（致命错误时调用，禁止该蓝图再次加载）
int error_revoke_blueprint(
    SharedMemory* root,
    const char* blueprint_id,
    uint32_t error_code,
    const char* reason
);
```

---

## 5. SHM数据结构设计

### 5.1 错误日志段（专用子段：{mall_name}-error-log）

```c
// 段头（64字节）
struct error_log_header {
    uint64_t magic;              // "ERRLOG" = 0x4552524C4F470000
    uint32_t version;
    uint32_t record_capacity;    // 最大记录数（如1024条）
    uint32_t write_index;        // 循环写指针
    uint32_t total_recorded;     // 历史总记录数（可能溢出）
    uint32_t fatal_count;        // 致命错误计数
    uint32_t error_count;        // 错误计数
    uint32_t warning_count;      // 警告计数
    uint8_t  reserved[12];
};

// 错误记录（128字节每项）
struct error_record {
    uint64_t timestamp;        // 毫秒级Unix时间戳
    uint16_t reporter_id;        // 报告者Worker ID
    uint32_t error_code;         // 错误代码
    uint8_t  severity;           // 等级
    uint8_t  recovered;          // 0=未恢复, 1=已恢复, 2=恢复失败
    uint8_t  retry_count;        // 已重试次数
    uint64_t context_addr;       // 相关SHM地址
    char     message[96];        // 错误描述（null-terminated）
};
```

### 5.2 信号映射表（位于根段配置区）

```c
struct signal_mapping {
    uint8_t  sig_num;            // Unix信号编号
    uint32_t error_code;         // 映射的商场错误代码
    uint8_t  default_strategy;   // 默认恢复策略
    uint8_t  reserved[3];
};
// 支持 16 个信号映射（64字节）
```

### 5.3 恢复策略表（位于根段配置区）

```c
struct recovery_entry {
    uint32_t error_code;         // 错误代码
    uint8_t  strategy;             // 恢复策略
    uint32_t max_retries;        // 最大重试次数
    uint32_t current_retries;    // 当前重试计数（全局）
    uint8_t  reserved[3];
};
// 支持 64 个恢复策略（64 × 16 = 1024字节）
```

### 5.4 吊销蓝图表（位于根段配置区）

```c
struct revoked_blueprint {
    char     blueprint_id[32];   // 被吊销的蓝图标识
    uint32_t error_code;         // 吊销原因错误代码
    uint64_t revoke_time;        // 吊销时间戳
    char     reason[48];         // 吊销原因描述
};
// 支持 32 个吊销记录（32 × 88 = 2816字节）
```

---

## 6. 函数详细设计

### 6.1 `error_report`

**逻辑流程**：
1. 校验 `root` 非空，`message` 非空，`severity <= 3`
2. 读取错误日志段头，校验 `magic`
3. **去重检查**：读取最近5条记录，若同一 `reporter_id` 与 `error_code` 已存在且时间差 < 5秒，跳过写入
4. 计算写入位置：`write_index % record_capacity`
5. 填充错误记录：`timestamp`、`reporter_id`、`error_code`、`severity`、`message`、`context_addr`
6. 原子递增 `write_index` 与对应等级计数器（`fatal_count` / `error_count` / `warning_count`）
7. 若 `severity == FATAL`：
   a. 触发恢复策略查询
   b. 若策略为 `吊销蓝图`，调用 `error_revoke_blueprint`
   c. 向Worker表写入隔离指令
8. 返回0

**返回值**：
- `0`：成功（或被去重）
- `-ENOSPC`：错误日志段未初始化
- `-EINVAL`：参数非法

### 6.2 `error_signal_register`

**逻辑流程**：
1. 校验 `root` 非空
2. 遍历 `signal_mask` 的每一位：
   a. 若位被设置，使用 `sigaction()` 注册 `error_signal_dispatch`
   b. 在信号映射表中查找对应映射，若不存在使用默认映射
3. 设置 `SA_SIGINFO` 标志以获取完整信号信息
4. 返回0

**返回值**：
- `0`：成功
- `-EPERM`：权限不足（非主进程调用）

### 6.3 `error_recovery_execute`

**逻辑流程**：
1. 在恢复策略表中查找 `error_code`
2. 若未找到，使用默认策略 `strategy=3`（重启Worker）
3. 根据策略执行：
   - `0=忽略`：直接返回成功
   - `1=重试`：递增 `current_retries`，若超过 `max_retries` 返回失败
   - `2=重置状态机`：调用相关产品状态机重置函数（通过函数指针表）
   - `3=重启Worker`：调用 `worker_terminate` + `worker_spawn`
   - `4=吊销蓝图`：调用 `error_revoke_blueprint`，终止所有使用该蓝图的Worker
4. 更新错误记录的 `recovered` 与 `retry_count`
5. 返回结果

**返回值**：
- `0`：恢复成功
- `1`：恢复失败
- `2`：达到最大重试次数

### 6.4 `error_revoke_blueprint`

**逻辑流程**：
1. 校验 `root` 非空，`blueprint_id` 非空
2. 在吊销蓝图表中查找第一个 `FREE` 槽位
3. 若已满，覆盖最旧的记录（循环覆盖）
4. 写入 `blueprint_id`、`error_code`、`revoke_time`、`reason`
5. 遍历Worker表，找到所有 `blueprint_hash` 匹配该蓝图的Worker
6. 对每个匹配Worker调用 `worker_kill`，原因码设为 `3=蓝图违规`
7. 向错误日志报告 `INFO` 等级事件："蓝图 {id} 已吊销，{N} 个Worker被终止"
8. 返回0

---

## 7. 测试场景

### 7.1 单元测试：错误报告与查询

```
测试名：test_error_report_query
步骤：
  1. 初始化SHM与错误日志段（capacity=1024）
  2. 报告3个错误：INFO、WARNING、ERROR
  3. 查询 min_severity=WARNING
断言：
  - 查询返回2条记录（WARNING与ERROR）
  - 记录内容匹配报告时的 message
  - total_recorded == 3
```

### 7.2 单元测试：去重机制

```
测试名：test_error_deduplication
步骤：
  1. 初始化错误日志
  2. 同一Worker连续报告同一error_code 10次（间隔1秒）
断言：
  - 错误日志中只记录1条（首次）
  - total_recorded 可能只递增1（取决于实现策略）
```

### 7.3 异常测试：信号捕获

```
测试名：test_error_sigsegv_capture
步骤：
  1. 注册SIGSEGV处理器
  2. 创建一个测试Worker，故意触发段错误（访问非法地址）
  3. 等待信号处理
断言：
  - 错误日志中出现SIGSEGV映射的错误代码
  - reporter_id 匹配测试Worker
  - severity == FATAL
  - Worker被标记为DEAD
```

### 7.4 集成测试：恢复策略执行

```
测试名：test_error_recovery_restart
步骤：
  1. 注册恢复策略：error_code=0x00050001, strategy=3（重启Worker）
  2. 启动一个生产者Worker
  3. 模拟Worker崩溃（通过向其发送SIGSEGV）
  4. 错误守卫自动执行恢复
断言：
  - 原Worker被终止
  - 新Worker被自动创建（相同blueprint_id）
  - 新Worker status == RUNNING
  - 错误记录 recovered == 1
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager, BP-0005-worker-spawn |
| 后置蓝图 | BP-0008-aicoder-gateway（AICoder错误也需被捕获）、BP-0009-mall-monitor（错误统计可视化） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0007.json`，含信号映射表与恢复策略模板 |
| 商场加载 | 由商场 `main()` 在创世第2阶段加载，紧接Worker孵化之后，必须在任何业务Worker启动前完成信号注册 |

---

## 9. 附录：错误代码分配表

| 高16位（产品ID） | 低16位范围 | 含义 |
|------------------|-----------|------|
| 0x0000 | 0x0000-0x00FF | 商场主进程通用错误 |
| 0x0001 | 0x0100-0x01FF | BP-0001-keyboard 错误 |
| 0x0002 | 0x0200-0x02FF | BP-0002-console 错误 |
| 0x0003 | 0x0300-0x03FF | BP-0003-stdio-pipe 错误 |
| 0x0004 | 0x0400-0x04FF | BP-0004-shm-manager 错误 |
| 0x0005 | 0x0500-0x05FF | BP-0005-worker-spawn 错误 |
| 0x0006 | 0x0600-0x06FF | BP-0006-intent-router 错误 |
| 0x0007 | 0x0700-0x07FF | BP-0007-error-guard 错误 |
| 0x0008 | 0x0800-0x08FF | BP-0008-aicoder-gateway 错误 |
| 0xFFFF | 0xFF00-0xFFFF | 系统级致命错误（内存宪法违反等） |

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
