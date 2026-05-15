# BP-0046 IO管理Worker

## 元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0046 |
| 功能域 | mall |
| 函数名 | io_manager_init / io_manager_submit / io_manager_cancel / io_manager_query |
| 版本 | 1.1.0 |
| 状态 | DRAFT |
| 依赖 | BP-0004-exception, BP-0009-registry |
| 关联规范 | SPEC-IO-DEBUG-0001 v1.1（第六章） |

---

## 1. 设计目标

为商场Worker提供统一的IO操作管理能力，解决IO操作的四大核心问题：

1. **不可切割性**: IO操作是原子性的，必须完整执行或完全回滚
2. **自闭环**: IO操作必须自包含，不依赖外部可变状态
3. **超时控制**: 所有IO操作必须规定最大等待时间
4. **进程隔离**: 可能阻塞的IO操作必须放入子进程隔离执行
5. **唯一性标识**: 每个IO Worker实例必须具有全局唯一性标识（IO-Worker-UID），确保可追溯性和并发安全（新增）

---

## 2. 核心概念

### 2.1 IO操作类型

| 类型 | 描述 | 示例 | 隔离要求 |
|------|------|------|----------|
| `DISK_READ` | 磁盘读取 | 文件读取、配置加载 | 可选 |
| `DISK_WRITE` | 磁盘写入 | 日志持久化、数据保存 | 可选 |
| `NETWORK_RX` | 网络接收 | HTTP请求、Socket读取 | 必须隔离 |
| `NETWORK_TX` | 网络发送 | HTTP响应、Socket写入 | 可选 |
| `EXTERNAL_PROC` | 外部进程 | 系统命令、子程序调用 | 必须隔离 |
| `DEVICE_IO` | 设备IO | 串口、USB、硬件访问 | 必须隔离 |

### 2.2 IO句柄 (IOHandle)

```
IOHandle结构:
├── handle_id: str          # 唯一标识符 (IO-{timestamp}-{random})
├── worker_uid: str         # IO Worker唯一性标识 (IO-WORKER-{TYPE}-{TIMESTAMP}-{RANDOM})
├── operation_type: str     # 操作类型 (DISK_READ/NETWORK_RX/...)
├── state: str             # 状态 (PENDING/RUNNING/COMPLETED/FAILED/CANCELLED)
├── timeout_ms: int         # 超时时间(毫秒)
├── start_time: float      # 开始时间戳
├── deadline: float        # 截止时间戳
├── isolation_mode: str    # 隔离模式 (NONE/THREAD/PROCESS)
├── worker_pid: int        # 执行进程ID
├── result: dict          # 执行结果
└── error_info: dict     # 错误信息
```

### 2.3 IO状态机

```
                    +-----------+
                    |  PENDING  |  <-- io_manager_submit()
                    +-----------+
                          |
                          v
                    +-----------+
            +-----> |  RUNNING  |  <-- 资源分配完成
            |       +-----------+
            |             |
      cancel|       +-----+-----+
            |       |           |
            v       v           v
      +-----------+     +-----------+
      | CANCELLED |     | COMPLETED |  <-- 成功完成
      +-----------+     +-----------+
                              |
                        timeout|
                              v
                        +-----------+
                        |   FAILED  |  <-- 超时/异常/错误
                        +-----------+
```

---

## 3. 函数契约

### 3.1 io_manager_init

**功能**: 初始化IO管理器

**输入**: 
```python
{
    "max_concurrent_io": int,      # 最大并发IO数 (默认: 10)
    "default_timeout_ms": int,     # 默认超时(毫秒) (默认: 5000)
    "enable_isolation": bool       # 是否启用进程隔离 (默认: True)
}
```

**输出**:
```python
{
    "status": "success" | "error",
    "displayed": str,
    "timestamp": float,
    "source": "io_manager_init",
    "metadata": {
        "manager_id": str,
        "manager_uid": str,        # IO Manager唯一性标识
        "config": dict
    },
    "error": dict | None
}
```

**异常**: 
- `MALL_INIT_FAILED`: 初始化失败
- `MALL_RESOURCE_EXHAUSTED`: 资源不足

**唯一性标识生成**:
```python
def _generate_manager_uid() -> str:
    timestamp = int(time.time() * 1000)
    random_part = os.urandom(4).hex()
    return f"IO-MANAGER-{timestamp}-{random_part}"
```

---

### 3.2 io_manager_submit

**功能**: 提交IO操作请求

**输入**:
```python
{
    "operation_type": str,         # IO操作类型
    "operation_params": dict,      # 操作参数
    "timeout_ms": int,             # 超时时间(可选,默认使用全局配置)
    "isolation_mode": str,         # 隔离模式 (NONE/THREAD/PROCESS)
    "callback_vector": str | None  # 完成通知矢量(可选)
}
```

**输出**:
```python
{
    "status": "submitted" | "error",
    "displayed": str,
    "timestamp": float,
    "source": "io_manager_submit",
    "metadata": {
        "handle_id": str,
        "worker_uid": str,            # IO Worker唯一性标识
        "estimated_duration_ms": int,
        "queue_position": int
    },
    "error": dict | None
}
```

**唯一性标识生成**:
```python
def _generate_worker_uid(operation_type: str) -> str:
    timestamp = int(time.time() * 1000)
    random_part = os.urandom(4).hex()
    return f"IO-WORKER-{operation_type}-{timestamp}-{random_part}"
```

**异常**:
- `MALL_INVALID_PARAM`: 参数无效
- `MALL_RESOURCE_EXHAUSTED`: 并发数超限
- `MALL_ISOLATION_FAILED`: 隔离进程创建失败
- `IO_UID_COLLISION`: 唯一性标识冲突（重新生成）

---

### 3.3 io_manager_cancel

**功能**: 取消正在执行的IO操作

**输入**:
```python
{
    "handle_id": str,              # IO句柄ID
    "force": bool                  # 是否强制终止 (默认: False)
}
```

**输出**:
```python
{
    "status": "cancelled" | "not_found" | "error",
    "displayed": str,
    "timestamp": float,
    "source": "io_manager_cancel",
    "metadata": {
        "handle_id": str,
        "previous_state": str,
        "termination_method": str  # "graceful" | "forced"
    },
    "error": dict | None
}
```

**异常**:
- `MALL_NOT_FOUND`: 句柄不存在
- `MALL_TERMINATION_FAILED`: 终止失败

---

### 3.4 io_manager_query

**功能**: 查询IO操作状态

**输入**:
```python
{
    "handle_id": str | None,       # 指定句柄ID(可选)
    "filter_state": str | None,    # 状态过滤(可选)
    "filter_type": str | None      # 类型过滤(可选)
}
```

**输出**:
```python
{
    "status": "success" | "error",
    "displayed": str,
    "timestamp": float,
    "source": "io_manager_query",
    "metadata": {
        "total_count": int,
        "filtered_count": int,
        "operations": list[dict]  # IOHandle列表
    },
    "error": dict | None
}
```

---

## 4. 隔离执行模型

### 4.1 隔离决策矩阵

| 操作类型 | 预估阻塞时间 | 推荐隔离模式 | 理由 |
|----------|-------------|-------------|------|
| DISK_READ < 1MB | < 10ms | NONE | 本地磁盘快速读取 |
| DISK_WRITE < 1MB | < 10ms | NONE | 本地磁盘快速写入 |
| DISK_READ > 10MB | > 100ms | THREAD | 大文件读取可能阻塞 |
| NETWORK_RX | 不确定 | PROCESS | 网络延迟不可控 |
| NETWORK_TX | < 100ms | THREAD | 发送通常较快 |
| EXTERNAL_PROC | 不确定 | PROCESS | 外部进程可能死锁 |
| DEVICE_IO | 不确定 | PROCESS | 硬件响应不可控 |

### 4.2 子进程隔离协议

```
父进程                    子进程(IO执行器)
   |                            |
   |--- fork() ---------------->|
   |                            |
   |--- IPC Channel ----------->|  (pipe/queue/shm)
   |                            |
   |  {"cmd": "EXECUTE",       |
   |   "params": {...},         |
   |   "timeout_ms": 5000}      |
   |--------------------------->|
   |                            |
   |                            |---> 执行IO操作
   |                            |     (带超时控制)
   |                            |
   |  {"status": "COMPLETED",   |
   |   "result": {...},         |
   |   "duration_ms": 123}      |
   |<---------------------------|
   |                            |
   |--- waitpid() --------------|  (回收子进程)
   |                            |
```

### 4.3 超时处理策略

| 层级 | 机制 | 适用场景 |
|------|------|----------|
| 信号(SIGALRM) | Unix信号超时 | 单线程Python |
| 线程定时器 | threading.Timer | 多线程环境 |
| 异步超时 | asyncio.wait_for | 异步IO |
| 进程监控 | 父进程轮询 | 子进程隔离 |

---

## 5. 错误处理体系

### 5.1 IO错误类型

| 错误码 | 描述 | 处置动作 |
|--------|------|----------|
| `IO_TIMEOUT` | 操作超时 | 取消操作,返回超时错误 |
| `IO_CANCELLED` | 操作被取消 | 清理资源,返回取消状态 |
| `IO_PERMISSION_DENIED` | 权限不足 | 返回权限错误 |
| `IO_NOT_FOUND` | 资源不存在 | 返回404类错误 |
| `IO_CONNECTION_REFUSED` | 连接被拒绝 | 返回连接错误 |
| `IO_ISOLATION_FAILED` | 隔离进程创建失败 | 降级到线程模式 |
| `IO_RESOURCE_BUSY` | 资源被占用 | 排队或返回忙错误 |
| `IO_UNKNOWN_ERROR` | 未知错误 | 记录日志,返回通用错误 |

### 5.2 错误传播

```
IO执行层错误
      |
      v
+---------------+
| 错误分类器     |  <-- 映射到标准错误码
+---------------+
      |
      +---> IO_TIMEOUT --------> 通知调用者
      +---> IO_CANCELLED ------> 状态更新
      +---> IO_SYSTEM_ERROR ---> 记录审计日志
      +---> IO_NETWORK_ERROR --> 触发重试逻辑
```

---

## 6. 安全约束

### 6.1 输入验证

- `operation_type` 必须在预定义类型列表中
- `timeout_ms` 必须在 [100, 300000] 范围内 (100ms ~ 5min)
- `isolation_mode` 必须是 NONE/THREAD/PROCESS 之一
- `operation_params` 必须经过参数消毒

### 6.2 资源限制

- 最大并发IO数: 可配置,默认10
- 单操作最大超时: 5分钟
- 子进程最大存活时间: 超时时间 + 10秒(清理缓冲)
- 孤儿进程自动回收: 每30秒扫描一次

### 6.3 隔离沙箱

子进程隔离时:
- 限制文件系统访问 (chroot或路径白名单)
- 限制网络访问 (防火墙规则)
- 限制系统调用 (seccomp-bpf, Linux only)
- 限制资源使用 (CPU/内存配额)

---

## 7. 性能指标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 提交延迟 | < 1ms | 从提交到开始执行的延迟 |
| 进程创建开销 | < 50ms | fork+exec子进程的时间 |
| 超时精度 | ±50ms | 实际超时与设定值的偏差 |
| 并发吞吐量 | > 100 ops/sec | 每秒处理的IO操作数 |
| 内存占用 | < 10MB | IO管理器基础内存占用 |

---

## 8. 测试策略

### 8.1 单元测试

| 测试项 | 数量 | 覆盖点 |
|--------|------|--------|
| 初始化测试 | 5 | 参数验证、重复初始化、资源分配 |
| 提交测试 | 10 | 各类IO操作提交、参数边界 |
| 超时测试 | 8 | 正常完成、超时触发、超时精度 |
| 取消测试 | 6 | 正常取消、强制终止、重复取消 |
| 隔离测试 | 8 | 进程创建、IPC通信、资源回收 |
| 并发测试 | 6 | 并发提交、资源竞争、死锁避免 |
| 错误处理 | 8 | 各类错误码、错误传播、恢复 |

### 8.2 集成测试

- 与异常处理体系集成 (BP-0004)
- 与进程注册表集成 (BP-0009)
- 与SHM矢量系统集成 (BP-0003)
- 端到端IO链路测试

### 8.3 压力测试

- 1000并发IO操作
- 持续运行1小时
- 内存泄漏检测
- 僵尸进程检测

---

## 9. 实现约束

### 9.1 函数范式

- 零全局状态: 所有状态通过ctx参数传递
- 纯函数设计: 相同输入产生相同输出
- 无OOP: 不使用class/继承
- 统一返回: dict结构,含status/displayed/timestamp/source/metadata/error

### 9.2 依赖限制

允许:
- `multiprocessing` (进程隔离)
- `threading` (线程超时)
- `signal` (信号处理)
- `os` (进程管理,仅限fork/waitpid/kill)
- `time` (超时计时)

禁止:
- 直接网络IO (必须通过隔离层)
- 直接文件IO (大文件必须通过隔离层)
- 阻塞式系统调用

---

## 10. 唯一性标识规范

### 10.1 标识格式

IO Manager 和 Worker 实例必须遵循以下唯一性标识格式：

```
IO-MANAGER-{TIMESTAMP}-{RANDOM}      # Manager标识
IO-WORKER-{TYPE}-{TIMESTAMP}-{RANDOM} # Worker标识
```

| 字段 | 说明 | 示例 |
|------|------|------|
| `IO-MANAGER` / `IO-WORKER` | 固定前缀 | `IO-MANAGER` / `IO-WORKER` |
| `{TYPE}` | IO操作类型（仅Worker） | `NETWORK_RX`, `DISK_READ` |
| `{TIMESTAMP}` | 13位毫秒时间戳 | `1747123456789` |
| `{RANDOM}` | 8位随机十六进制 | `A3F2D1C9` |

### 10.2 标识生成规则

- **Manager初始化**: `io_manager_init()` 生成Manager唯一性标识
- **Worker提交**: `io_manager_submit()` 为每次提交生成Worker唯一性标识
- **时间单调**: 同一Manager下的Worker，时间戳必须单调递增
- **随机防碰撞**: 使用 `os.urandom(4).hex()` 生成随机部分

### 10.3 标识存储

| 存储位置 | 字段名 | 用途 |
|----------|--------|------|
| ctx["io_manager"] | `manager_uid` | Manager全局唯一标识 |
| IOHandle | `worker_uid` | Worker实例唯一标识 |

### 10.4 冲突检测

| 错误码 | 场景 | 处置 |
|--------|------|------|
| `IO_UID_COLLISION` | 标识冲突 | 重新生成（最多重试3次） |
| `IO_UID_INVALID` | 格式非法 | 拒绝实例化 |

---

## 11. 验收标准

- [ ] 所有单元测试通过 (51项)
- [ ] 唯一性标识测试通过 (5项新增)
- [ ] 集成测试通过
- [ ] 压力测试通过
- [ ] 零内存泄漏
- [ ] 零僵尸进程
- [ ] 超时精度达标 (±50ms)
- [ ] 代码通过七维审查
- [ ] 唯一性标识格式验证通过
- [ ] 标识冲突检测通过
