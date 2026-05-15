# BP-0005-worker-spawn.md

> **蓝图编号**：BP-0005  
> **产品名称**：worker-spawn（Worker进程孵化）  
> **产品类型**：基础设施产品  
> **优先级**：P0  
> **依赖**：BP-0004-shm-manager  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`worker-spawn` 是商场的**进程生命中枢**，负责生产者、消费者及内部服务进程（统称为Worker）的创建、监控、调度与销毁。它建立在 `BP-0004-shm-manager` 之上，将进程元数据（PID、状态、心跳、退出码）全部存储于SHM，实现**无状态调度器**——调度逻辑本身不维护任何进程表，所有状态从SHM实时读取。

Worker分为两类：
- **生产者Worker**：持有产品蓝图授权，向SHM写入数据
- **消费者Worker**：持有订阅授权，从SHM读取数据
- **服务Worker**：商场内部服务（如监控、日志收集），无生产消费行为

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 进程模型 | 仅使用 `multiprocessing.Process`，禁止 `os.fork()` 直接调用 |
| 状态存储 | 所有Worker状态存储于SHM进程表，禁止调度器内部字典/列表缓存 |
| 信号策略 | 优雅退出使用 `SIGTERM`，强制终止使用 `SIGKILL`（仅当优雅超时后） |
| 隔离原则 | 每个Worker为独立OS进程，崩溃不影响商场主进程与其他Worker |
| 资源回收 | Worker退出后，其SHM段由 `worker-spawn` 标记清理，延迟回收 |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **Worker表** | SHM中的数组，每项描述一个Worker的元数据 |
| **PID槽位** | Worker表中的固定索引，0号槽位保留给商场主进程 |
| **心跳段** | 每个Worker拥有的专用SHM小段，定期写入时间戳 |
| **退出码段** | Worker退出时写入退出状态，供商场收割 |
| **优雅期** | 收到终止信号后，Worker必须在 `GRACE_PERIOD_MS` 内自行清理 |
| **进程池** | 预创建的空白Worker进程池，用于快速启动（可选优化） |
| **蓝图句柄** | Worker启动时传入的授权文件标识，决定其行为 |

---

## 4. 接口定义

### 4.1 Worker生命周期

```c
// 创建并启动一个Worker进程
int worker_spawn(
    SharedMemory* root,           // SHM根段
    const char* blueprint_id,     // 蓝图标识（如 "BP-0001-keyboard"）
    uint8_t worker_type,          // 0=生产者, 1=消费者, 2=服务
    const char* entry_func,       // 入口函数名（如 "keyboard_main"）
    uint16_t* out_worker_id,      // 出参：分配的Worker ID（即PID槽位号）
    pid_t* out_os_pid             // 出参：操作系统PID
);

// 向Worker发送优雅终止信号
int worker_terminate(
    SharedMemory* root,
    uint16_t worker_id,
    uint32_t grace_period_ms      // 等待自行退出的最大毫秒数
);

// 强制终止Worker（SIGKILL）
int worker_kill(
    SharedMemory* root,
    uint16_t worker_id,
    uint8_t reason_code          // 终止原因：0=正常, 1=心跳超时, 2=内存违规, 3=蓝图违规
);

// 等待Worker退出并收割资源
int worker_reap(
    SharedMemory* root,
    uint16_t worker_id,
    int* out_exit_code,           // 出参：退出码
    uint8_t* out_reason           // 出参：实际退出原因
);
```

### 4.2 监控与心跳

```c
// 注册Worker心跳段（由Worker自身调用）
int worker_heartbeat_register(
    SharedMemory* root,
    uint16_t worker_id,
    uint32_t interval_ms          // 心跳间隔
);

// 更新心跳（由Worker定期调用）
int worker_heartbeat_pulse(
    SharedMemory* root,
    uint16_t worker_id
);

// 商场调度器检查所有Worker心跳
int worker_health_scan(
    SharedMemory* root,
    uint32_t timeout_ms,          // 心跳超时阈值
    uint16_t* out_dead_workers,   // 出参：死亡Worker ID数组
    uint32_t* out_dead_count      // 出参：死亡数量
);
```

### 4.3 进程池管理（可选）

```c
// 预创建空白Worker池
int worker_pool_init(
    SharedMemory* root,
    uint32_t pool_size,           // 池大小
    uint32_t stack_size           // 每个Worker栈大小
);

// 从池中快速启动Worker（减少fork开销）
int worker_pool_acquire(
    SharedMemory* root,
    const char* blueprint_id,
    uint8_t worker_type,
    const char* entry_func,
    uint16_t* out_worker_id
);
```

---

## 5. SHM数据结构设计

### 5.1 Worker表（位于根段宪法区之后）

```c
// 根段基地址 + 0x2000：Worker表（每项64字节）
struct worker_entry {
    uint16_t worker_id;          // Worker ID（槽位索引）
    uint16_t blueprint_hash;     // 蓝图标识的CRC16校验
    pid_t    os_pid;             // 操作系统PID
    uint8_t  worker_type;        // 0=生产者, 1=消费者, 2=服务
    uint8_t  status;             // 0=FREE, 1=SPAWNING, 2=RUNNING, 3=TERMINATING, 4=ZOMBIE, 5=DEAD
    uint8_t  exit_code;          // 退出码（若已退出）
    uint8_t  kill_reason;        // 终止原因码
    uint64_t spawn_time;         // 启动时间戳（毫秒级Unix时间）
    uint64_t last_heartbeat;     // 最后一次心跳时间戳
    uint32_t heartbeat_interval; // 心跳间隔（毫秒）
    char     entry_func[16];     // 入口函数名
    uint8_t  reserved[8];
};
// 默认支持 256 个Worker（256 × 64 = 16KB）
```

### 5.2 心跳段（每个Worker独立子段）

```c
// 子段名格式：{mall_name}-heartbeat-{worker_id}
struct heartbeat_block {
    uint64_t last_pulse;         // 上次心跳时间戳
    uint32_t pulse_count;        // 总心跳次数
    uint32_t missed_count;       // 漏跳次数（由调度器更新）
    uint8_t  status;             // 0=ALIVE, 1=SUSPECTED, 2=CONFIRMED_DEAD
    uint8_t  reserved[7];
};
```

### 5.3 退出码段（每个Worker独立子段）

```c
// 子段名格式：{mall_name}-exit-{worker_id}
struct exit_block {
    uint8_t  exited;             // 0=运行中, 1=已退出
    uint8_t  exit_code;          // 进程退出码
    uint8_t  signal_num;         // 若被信号终止，记录信号编号
    uint64_t exit_time;          // 退出时间戳
    char     exit_msg[56];       // 退出消息（如异常堆栈摘要）
};
```

---

## 6. 函数详细设计

### 6.1 `worker_spawn`

**逻辑流程**：
1. 校验 `root` 非空，`blueprint_id` 非空，`entry_func` 非空
2. 在Worker表中查找第一个 `status == FREE` 的槽位
3. 若已满，返回 `-ENOSPC`
4. 设置槽位状态为 `SPAWNING`
5. 使用 `multiprocessing.Process(target=entry_func, args=(root, worker_id))` 创建进程
6. 写入 `os_pid`、`spawn_time`、`blueprint_hash`、`worker_type`、`entry_func`
7. 创建心跳段与退出码段（通过 `shm_create_segment`）
8. 启动进程，设置槽位状态为 `RUNNING`
9. 返回 `worker_id` 与 `os_pid`

**返回值**：
- `0`：成功
- `-ENOSPC`：Worker表已满
- `-EINVAL`：参数非法
- `-ENOEXEC`：入口函数不可执行

### 6.2 `worker_terminate`

**逻辑流程**：
1. 校验 `root` 非空，`worker_id` 在有效范围
2. 读取Worker表项，校验 `status == RUNNING`
3. 向 `os_pid` 发送 `SIGTERM`
4. 设置槽位状态为 `TERMINATING`
5. 等待 `grace_period_ms`，轮询退出码段
6. 若Worker自行退出，调用 `worker_reap`
7. 若超时未退出，返回 `-ETIMEOUT`（调用者可继续调用 `worker_kill`）

**返回值**：
- `0`：Worker已优雅退出
- `-ETIMEOUT`：优雅期超时
- `-ESRCH`：Worker不存在

### 6.3 `worker_health_scan`

**逻辑流程**：
1. 获取当前时间戳 `now`
2. 遍历Worker表（跳过 `FREE` 与 `DEAD`）
3. 对每个 `RUNNING` Worker：
   a. 读取其心跳段 `last_pulse`
   b. 计算 `elapsed = now - last_pulse`
   c. 若 `elapsed > timeout_ms`：
      - 递增 `missed_count`
      - 若 `missed_count >= 3`，标记 `status = CONFIRMED_DEAD`
      - 将Worker ID写入 `out_dead_workers` 数组
4. 返回死亡数量

**返回值**：
- `0`：扫描完成
- `>0`：死亡Worker数量（通过出参）

---

## 7. 测试场景

### 7.1 单元测试：Worker创建与退出

```
测试名：test_worker_spawn_exit
步骤：
  1. 创建SHM根段
  2. 调用 worker_spawn 创建测试Worker（类型=服务，入口函数=dummy_main）
  3. 校验 worker_id 在 [1, 255] 范围
  4. 校验 os_pid > 0
  5. 调用 worker_terminate 优雅退出
  6. 调用 worker_reap 收割
断言：
  - spawn 返回0
  - Worker表项 status 最终为 DEAD
  - 退出码为0
  - 心跳段与退出码段已标记清理
```

### 7.2 压力测试：批量Worker创建

```
测试名：test_worker_mass_spawn
步骤：
  1. 创建根段
  2. 循环创建 256 个Worker
  3. 尝试创建第257个
断言：
  - 前256次均返回0
  - 第257次返回 -ENOSPC
  - 所有Worker status == RUNNING
```

### 7.3 异常测试：心跳超时检测

```
测试名：test_worker_heartbeat_timeout
步骤：
  1. 创建根段与Worker（心跳间隔=100ms）
  2. Worker故意停止发送心跳
  3. 调度器每50ms扫描一次，timeout_ms=300ms
断言：
  - 约350ms后，health_scan 检测到死亡
  - missed_count >= 3
  - 错误日志记录心跳超时事件
```

### 7.4 集成测试：生产者-消费者端到端

```
测试名：test_worker_e2e_producer_consumer
步骤：
  1. 商场main加载BP-0004、BP-0005、BP-0003
  2. 启动键盘生产者Worker（BP-0001）
  3. 启动控制台消费者Worker（BP-0002）
  4. 模拟键盘输入
  5. 终止生产者，再终止消费者
断言：
  - 生产者与消费者均正常启动
  - 数据流完整
  - 终止顺序无关，无僵尸进程残留
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager（SHM根段与段创建） |
| 后置蓝图 | BP-0006-intent-router（Worker启动后需注册意图）、BP-0007-error-guard（Worker崩溃处理） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0005.json`，含进程安全隔离规则 |
| 商场加载 | 由商场 `main()` 在创世第2阶段加载，紧接SHM管理器初始化之后 |

---

## 9. 附录：Worker状态机

```
[FREE] --worker_spawn()--> [SPAWNING] --进程启动成功--> [RUNNING]
[RUNNING] --worker_terminate()--> [TERMINATING] --自行退出--> [ZOMBIE] --worker_reap()--> [DEAD]
[RUNNING] --worker_kill()--> [DEAD]（跳过ZOMBIE）
[TERMINATING] --超时未退出--> [TERMINATING]（保持，需人工干预或二次kill）
[RUNNING] --心跳超时--> [SUSPECTED]（心跳段状态） --连续3次--> [CONFIRMED_DEAD] --worker_reap()--> [DEAD]
[DEAD] --延迟清理--> [FREE]（槽位回收，供下次spawn复用）
```

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
