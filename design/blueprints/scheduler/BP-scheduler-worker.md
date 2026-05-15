# 微型商场 · 调度Worker设计蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档编号**: MW-SCH-001  
> **版本**: v1.0  
> **日期**: 2026-05-09  
> **对应规范**: 微型商场设计规范 v1.2  
> **设计层级**: Python函数级（函数签名、参数、返回值、职责说明）  
> **关联文档**: 创世Worker蓝图、SHM设计蓝图、算力法条

---

## 1. Worker职责定义

调度Worker是微型商场的"中央神经系统"，承担以下核心职责：

- **算力进程池管理**: 统一订阅窗口与发布窗口的算力资源池生命周期管理
- **任务路由分发**: 根据Worker能力标签、算力需求曲线、当前负载进行智能调度
- **Worker注册与注销**: 管理Worker的准入、心跳续约、优雅退出
- **动态算力评估**: 建立函数输入数据规模与算力需求曲线，提前预知算力消耗
- **长时间计算续约**: 对高算力任务进行执行期续约管理，保留备份算力

---

## 2. 入口参数设计

```python
def scheduler_worker_entry(
    shm_config: dict,           # SHM共享内存配置（来自SHM设计蓝图）
    mall_state_fd: int,          # 商场状态文件描述符（创世Worker初始化）
    worker_registry_addr: str,   # Worker注册表内存地址
    compute_pool_size: int,      # 算力进程池初始容量
    lease_timeout_ms: int,      # 算力租约默认超时（毫秒）
    dispatch_policy: str,        # 调度策略: "round_robin" | "load_balance" | "affinity"
    audit_callback: callable,    # 审计回调接口（对接审计Worker）
    security_callback: callable  # 安全回调接口（对接安全Worker）
) -> int:
    """
    调度Worker主入口函数

    章节对应:
        - 第1节: 职责定义
        - 第3节: 状态机实现
        - 第4节: 核心函数调用链

    返回值:
        0: 正常退出
        1: SHM初始化失败
        2: 注册表损坏
        3: 算力池耗尽且无法扩容
    """
```

---

## 3. 状态机设计

调度Worker内部状态机（State Machine）：

```
[INIT] --初始化SHM通道--> [REGISTRY_LOAD] --加载Worker注册表--> [POOL_WARMUP]
   |                                                              |
   |<---------------------- 异常重启 -----------------------------|
   v                                                              v
[DISPATCHING] <--任务到来-- [MONITORING] <--心跳检测-- [RUNNING]
   |                           ^    |                              |
   |--无可用算力--> [BACKUP_ALLOC]   |--负载异常--> [REBALANCING]   |
   |                           |    |                              |
   |--任务完成--> [CLEANUP]     |    |--全部空闲--> [IDLE]          |
   |                           |    |                              |
   v                           |    v                              |
[LEASE_RENEW] --续约成功--> [MONITORING] --续约失败--> [FAILOVER]  |
   |                                                              |
   v                                                              v
[GRACEFUL_SHUTDOWN] <----------------- 收到终止信号 -------------|
```

状态说明：
- **INIT**: 初始化本地状态，加载配置
- **REGISTRY_LOAD**: 从SHM读取Worker注册表，验证签名
- **POOL_WARMUP**: 预热算力进程池，建立统一订阅/发布窗口
- **RUNNING**: 正常运行，等待事件
- **MONITORING**: 监控Worker心跳与算力消耗
- **DISPATCHING**: 任务分派中，锁定算力资源
- **BACKUP_ALLOC**: 主算力不足，启用备份算力
- **LEASE_RENEW**: 高算力任务续约管理
- **REBALANCING**: 负载再平衡，迁移低优先级任务
- **CLEANUP**: 任务结束清理，释放算力
- **FAILOVER**: 故障转移，切换备用Worker
- **GRACEFUL_SHUTDOWN**: 优雅关闭，等待所有租约到期

---

## 4. 核心函数设计（函数级）

### 4.1 Worker注册管理

```python
def worker_register(
    worker_id: str,              # Worker唯一标识（UUID）
    worker_type: str,            # Worker类型: "genesis" | "compute" | "probe" | "consumer"
    capability_tags: list,       # 能力标签列表，如 ["fft", "spectrum", "verilog_compile"]
    max_compute_units: int,      # 最大算力单元数
    shm_segment_id: int,         # 该Worker分配的SHM段ID
    auth_token: bytes,           # 安全Worker签发的认证令牌
    heartbeat_interval_ms: int   # 心跳间隔（毫秒）
) -> dict:
    """
    Worker注册函数

    职责: 将Worker纳入商场统一管理，分配算力配额，建立心跳通道

    返回值 dict:
        {
            "status": "approved" | "rejected" | "pending_audit",
            "assigned_pool_id": int,
            "lease_quota": int,          # 算力租约配额
            "registry_slot": int,        # 注册表槽位号
            "dispatch_priority": int     # 调度优先级
        }

    测试方法:
        1. 模拟10个Worker并发注册，验证注册表无竞态条件
        2. 注入损坏的auth_token，验证安全Worker回调被触发
        3. 注册后断开Worker，验证心跳超时后自动注销
    """
```

### 4.2 任务调度分发

```python
def task_dispatch(
    task_id: str,                # 任务唯一标识
    function_ref: str,           # 函数库引用（行为模型标识）
    input_data_meta: dict,       # 输入数据元信息（规模、类型、SHM地址）
    compute_demand_curve: list,  # 算力需求曲线 [(data_size, compute_units), ...]
    priority: int,               # 优先级 0-9（0最高）
    consumer_id: str,            # 发起任务的消费者Worker ID
    deadline_ms: int            # 任务截止时间（毫秒）
) -> dict:
    """
    任务调度分发函数

    职责: 根据算力需求曲线预测算力消耗，选择最优Worker执行，锁定算力资源

    返回值 dict:
        {
            "status": "dispatched" | "queued" | "rejected",
            "assigned_worker_id": str,
            "lease_id": str,             # 算力租约ID
            "estimated_cost": float,     # 预估算力费用
            "queue_position": int,       # 排队位置（如queued）
            "shm_result_addr": int       # 结果写入的SHM地址
        }

    测试方法:
        1. 提交超过算力池容量的任务，验证进入队列而非崩溃
        2. 模拟Worker运行中崩溃，验证任务自动迁移到备份Worker
        3. 验证低算力函数不会被错误地放入算力进程池
        4. 测试截止时间到期前任务未完成时触发告警
    """
```

### 4.3 算力租约续约

```python
def compute_lease_renewal(
    lease_id: str,               # 租约ID
    worker_id: str,              # 持有租约的Worker ID
    progress_ratio: float,       # 任务进度 0.0-1.0
    additional_compute_units: int,  # 额外申请的算力单元
    reason: str                  # 续约原因（用于审计）
) -> dict:
    """
    算力租约续约函数

    职责: 对长时间计算任务进行租约续约，防止算力被回收导致任务中断
          保留备份算力以应对突发负载

    返回值 dict:
        {
            "status": "renewed" | "denied" | "backup_allocated",
            "new_lease_deadline": int,   # 新的租约到期时间戳
            "backup_worker_id": str,     # 分配的备份Worker（如适用）
            "cost_increment": float      # 新增算力费用
        }

    测试方法:
        1. 模拟任务运行超过默认租约时长，验证自动续约流程
        2. 备份算力耗尽时拒绝续约，验证任务优雅降级
        3. 验证续约记录实时写入审计日志
    """
```

### 4.4 动态负载均衡

```python
def load_balance_rebalance(
    overload_threshold: float,   # 负载过载阈值 0.0-1.0
    migration_policy: str        # 迁移策略: "gentle" | "urgent" | "force"
) -> dict:
    """
    动态负载再平衡函数

    职责: 监控各Worker算力负载，当某Worker超过阈值时，迁移低优先级任务
          确保频谱仪等实时Worker运行时不会阻塞其他设备

    返回值 dict:
        {
            "rebalanced": bool,
            "migrations": list,          # 迁移记录 [{"task_id", "from", "to"}, ...]
            "affected_workers": list,    # 受影响的Worker ID列表
            "shm_sync_status": str       # SHM同步状态
        }

    测试方法:
        1. 模拟某Worker负载达到95%，验证低优先级任务被迁移
        2. 迁移过程中原Worker崩溃，验证任务不丢失（SHM双缓冲）
        3. 验证频谱仪Worker运行时，其他设备Worker未被阻塞
    """
```

### 4.5 Worker优雅注销

```python
def worker_unregister(
    worker_id: str,              # 待注销Worker ID
    reason: str,                 # 注销原因: "completed" | "error" | "evicted" | "shutdown"
    grace_period_ms: int         # 优雅等待期（毫秒）
) -> dict:
    """
    Worker优雅注销函数

    职责: 安全地将Worker从注册表移除，回收算力资源，确保无遗留任务

    返回值 dict:
        {
            "status": "unregistered" | "pending_tasks" | "force_killed",
            "released_compute_units": int,
            "orphan_tasks": list,        # 遗留任务列表（需重新调度）
            "audit_log_id": str          # 审计日志条目ID
        }

    测试方法:
        1. 注销持有活跃租约的Worker，验证任务被重新调度
        2. 验证注销后该Worker的SHM段被标记为可回收
        3. 模拟恶意Worker注销攻击，验证安全Worker拦截
    """
```

---

## 5. 与其他Worker的交互协议

### 5.1 与创世Worker交互
- **INIT阶段**: 从创世Worker获取商场状态FD和初始注册表
- **SHM通道**: 通过创世Worker建立的SHM矢量管理机制通信
- **故障时**: 若创世Worker失联，调度Worker进入自维持模式（只读注册表）

### 5.2 与安全Worker交互
- **注册时**: 调用 `security_callback(auth_token)` 验证Worker身份
- **异常时**: 检测到Worker行为异常（如心跳伪造）时，向安全Worker申请吊销令牌
- **调度前**: 高优先级任务调度前，请求安全Worker进行运行时权限复核

### 5.3 与审计Worker交互
- **调度后**: 每次task_dispatch成功后，调用 `audit_callback()` 记录算力分配
- **续约时**: compute_lease_renewal的reason字段直接写入审计日志
- **注销时**: worker_unregister生成完整的生命周期审计报告

### 5.4 与探针Worker交互
- **MONITORING状态**: 接收探针Worker的实时健康数据，作为调度决策输入
- **REBALANCING状态**: 请求探针Worker提供各Worker的详细性能指标

---

## 6. 测试方法（设计检验标准）

| 测试场景 | 检验标准 | 通过条件 |
|---------|---------|---------|
| 并发注册压力测试 | 100个Worker同时注册 | 注册表无数据竞争，所有Worker在500ms内完成注册 |
| 算力耗尽场景 | 提交超过总容量3倍的任务 | 任务进入优先级队列，系统不崩溃，高优先级优先执行 |
| Worker故障转移 | 运行中强制kill一个Worker | 该Worker上的任务在1秒内迁移到备份Worker，结果无丢失 |
| 频谱仪隔离性 | 频谱仪Worker满负荷运行 | 其他设备Worker（示波器、网分）响应延迟 < 10ms |
| 租约续约风暴 | 50个任务同时请求续约 | 续约处理吞吐量 > 100次/秒，无死锁 |
| 优雅关闭 | 发送SIGTERM信号 | 所有活跃租约完成或超时后退出，无孤儿进程 |

---

## 7. 代码注释与文档章节映射

```python
# [对应第2节: 入口参数设计]
def scheduler_worker_entry(...):
    # [对应第3节: INIT状态]
    state = State.INIT
    # [对应第5.1节: 与创世Worker交互]
    mall_fd = shm_connect(shm_config["genesis_channel"])
    ...
```

---

*本蓝图由创世Worker通过AICoder生成，待人类架构师审核后进入函数库。*
