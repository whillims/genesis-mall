# BP-0060 驻留Worker管理架构：进程池/线程池分离 + SHM通道交互

> **版本**: v1.0  
> **日期**: 2026-05-15  
> **功能域**: mall  
> **状态**: ACTIVE  

---

## 一、设计概述

### 1.1 核心问题

**当前架构的问题**：
- 驻留Worker轮询、Task状态检查等逻辑直接在商场主循环中运行
- 商场主程序可能因Worker相关操作阻塞
- 无法充分利用多核CPU能力

**核心要求**：
- **商场主程序零阻塞**：驻留Worker管理逻辑完全从商场主循环剥离
- **进程池与线程池分离**：不同类型任务使用不同的池
- **SHM通道交互**：管理进程与商场通过SHM矢量通信
- **并行运行**：管理进程独立运行，不影响商场主循环

### 1.2 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        商场主进程 (Mall Core)                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  主循环: 仅处理核心业务 (请求/响应/矢量交易)              │  │
│  │  - 零阻塞，高速稳定                                       │  │
│  │  - 不直接管理Worker/Task                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                          │
                          │ SHM矢量通信
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Worker管理进程 (Manager)                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Thread Pool Manager (线程池管理器)                       │  │
│  │  - 驻留Worker轮询                                          │  │
│  │  - Task状态检查                                            │  │
│  │  - 轻量级IO密集型任务                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Process Pool Manager (进程池管理器)                      │  │
│  │  - 重计算型Task执行                                        │  │
│  │  - 隔离型Worker运行                                         │  │
│  │  - 可崩溃，不影响主进程                                     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                          │
                          │ SHM矢量通信
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                        驻留Worker进程                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Worker 1 (键盘采集)                                      │  │
│  │  Worker 2 (系统监控)                                      │  │
│  │  Worker 3 (频谱采集)                                      │  │
│  │  ...                                                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、SHM通信通道设计

### 2.1 核心控制矢量

| 矢量ID | 用途 | 方向 | 大小 |
|--------|------|------|------|
| `VEC-MGR-CMD` | 商场→管理器：命令通道 | 单向 | 64KB |
| `VEC-MGR-RESP` | 管理器→商场：响应通道 | 单向 | 64KB |
| `VEC-TASK-REQ` | 商场→管理器：Task提交 | 单向 | 1MB |
| `VEC-TASK-RESP` | 管理器→商场：Task结果 | 单向 | 1MB |
| `VEC-WORKER-EVENT` | 管理器→商场：Worker事件 | 单向 | 256KB |

### 2.2 命令协议格式

**命令帧结构**：
```python
{
    "cmd_id": "uuid",           # 命令唯一ID
    "cmd_type": "string",       # 命令类型
    "timestamp": 1234567890.0,  # 时间戳
    "payload": {},              # 命令数据
}
```

**命令类型**：
| 命令类型 | 说明 |
|---------|------|
| `REGISTER_WORKER` | 注册驻留Worker |
| `UNREGISTER_WORKER` | 注销驻留Worker |
| `START_WORKER` | 启动Worker |
| `STOP_WORKER` | 停止Worker |
| `SUBMIT_TASK` | 提交Task |
| `QUERY_STATUS` | 查询状态 |
| `SHUTDOWN` | 关闭管理器 |

---

## 三、管理器架构

### 3.1 管理器状态机

```
[INIT] → [RUNNING] → [SHUTTING_DOWN] → [TERMINATED]
           │
           ├─ 线程池管理
           ├─ 进程池管理
           └─ SHM通信处理
```

### 3.2 线程池管理器 (Thread Pool Manager)

**职责**：
- 驻留Worker轮询（检查数据、生成Task）
- Task状态检查（超时检测、状态更新）
- 轻量级IO密集型任务
- 低延迟，高吞吐

**设计**：
```python
THREAD_POOL_CONFIG = {
    "max_workers": 16,          # 最大线程数
    "queue_size": 1000,         # 任务队列大小
    "poll_interval_ms": 1,      # 驻留Worker轮询间隔
    "status_check_interval_ms": 10,  # Task状态检查间隔
}

class ThreadPoolManager:
    def __init__(self):
        self.pool = concurrent.futures.ThreadPoolExecutor(
            max_workers=THREAD_POOL_CONFIG["max_workers"]
        )
        self.task_queue = queue.Queue(maxsize=THREAD_POOL_CONFIG["queue_size"])
    
    def poll_resident_workers(self):
        """轮询所有驻留Worker，检查新数据"""
        while running:
            for worker_id in resident_workers:
                # 检查Worker数据（非阻塞）
                # 生成Task提交到线程池
                pass
            time.sleep(THREAD_POOL_CONFIG["poll_interval_ms"] / 1000.0)
    
    def check_task_status(self):
        """检查所有Task状态，处理超时"""
        while running:
            for task_id in tasks:
                # 检查超时
                # 更新状态
                pass
            time.sleep(THREAD_POOL_CONFIG["status_check_interval_ms"] / 1000.0)
```

### 3.3 进程池管理器 (Process Pool Manager)

**职责**：
- 重计算型Task执行（CPU密集）
- 隔离型Worker运行
- 崩溃隔离（一个进程挂不影响其他）
- 高资源消耗任务

**设计**：
```python
PROCESS_POOL_CONFIG = {
    "max_workers": 4,           # 最大进程数（按CPU核心数）
    "queue_size": 100,          # 任务队列大小
    "task_timeout_seconds": 60, # 默认超时
    "restart_on_crash": True,   # 崩溃自动重启
}

class ProcessPoolManager:
    def __init__(self):
        self.pool = concurrent.futures.ProcessPoolExecutor(
            max_workers=PROCESS_POOL_CONFIG["max_workers"]
        )
        self.task_queue = multiprocessing.Queue(maxsize=PROCESS_POOL_CONFIG["queue_size"])
    
    def submit_compute_task(self, task_data):
        """提交计算密集型Task"""
        future = self.pool.submit(compute_worker_fn, task_data)
        return future
    
    def monitor_processes(self):
        """监控进程状态，处理崩溃"""
        while running:
            # 检查进程健康
            # 崩溃自动重启
            pass
```

### 3.4 管理器主循环

```python
async def manager_main_loop():
    """管理器主循环"""
    # 初始化
    thread_pool = ThreadPoolManager()
    process_pool = ProcessPoolManager()
    
    # 启动后台线程
    threading.Thread(target=thread_pool.poll_resident_workers, daemon=True).start()
    threading.Thread(target=thread_pool.check_task_status, daemon=True).start()
    threading.Thread(target=process_pool.monitor_processes, daemon=True).start()
    
    # SHM通信处理
    while MGR_STATE["running"]:
        # 处理商场命令
        cmd = read_shm_command(VEC_MGR_CMD)
        if cmd:
            handle_command(cmd, thread_pool, process_pool)
        
        # 发送事件到商场
        send_events_to_mall()
        
        await asyncio.sleep(0.001)
```

---

## 四、商场主循环简化

### 4.1 新架构下的商场主循环

```python
async def mall_main_loop():
    """商场主循环 - 零阻塞，仅处理核心业务"""
    await asyncio.gather(
        queue_request_listener(),      # 处理外部请求
        health_check_loop(),           # 健康检查
        manager_event_listener(),      # 监听管理器事件
    )
```

### 4.2 管理器事件监听器

```python
async def manager_event_listener():
    """监听管理器发来的事件"""
    while MALL_STATE["running"]:
        # 从SHM读取管理器事件（非阻塞）
        events = read_new_frames(VEC_WORKER_EVENT)
        
        for event in events:
            event_type = event.get("type")
            if event_type == "TASK_COMPLETED":
                handle_task_completed(event)
            elif event_type == "TASK_FAILED":
                handle_task_failed(event)
            elif event_type == "WORKER_ERROR":
                handle_worker_error(event)
        
        await asyncio.sleep(0.001)
```

### 4.3 向管理器提交命令

```python
def send_command_to_manager(cmd_type, payload):
    """向管理器发送命令"""
    cmd = {
        "cmd_id": str(uuid.uuid4()),
        "cmd_type": cmd_type,
        "timestamp": time.time(),
        "payload": payload,
    }
    write_frame(VEC_MGR_CMD, cmd)

def submit_task_to_manager(consumer_id, input_data, consumer_type="CT"):
    """提交Task到管理器"""
    send_command_to_manager("SUBMIT_TASK", {
        "consumer_id": consumer_id,
        "input_data": input_data,
        "consumer_type": consumer_type,
    })

def register_worker_via_manager(worker_id, worker_type, vector_id, subscribers):
    """通过管理器注册Worker"""
    send_command_to_manager("REGISTER_WORKER", {
        "worker_id": worker_id,
        "worker_type": worker_type,
        "vector_id": vector_id,
        "subscribers": subscribers,
    })
```

---

## 五、驻留Worker管理流程

### 5.1 Worker注册流程

```
商场                      管理器                    SHM矢量
  │                        │                         │
  │─ 注册Worker命令 ──────>│                         │
  │  (VEC-MGR-CMD)         │                         │
  │                        │─ 验证Worker配置 ──────>│
  │                        │─ 初始化Worker状态       │
  │                        │─ 加入轮询列表           │
  │<─── 注册成功响应 ──────│                         │
  │  (VEC-MGR-RESP)        │                         │
  │                        │─ 开始轮询Worker数据 ──>│
```

### 5.2 数据处理流程

```
驻留Worker              管理器(线程池)             管理器(进程池)            商场
    │                      │                         │                      │
    │─ 写入数据 ─────────>│                         │                      │
    │  (VEC-KEYBOARD)      │                         │                      │
    │                      │─ 检测新数据             │                      │
    │                      │─ 生成Task               │                      │
    │                      │─ 提交到线程池/进程池 ──>│                      │
    │                      │                         │─ 执行Task            │
    │                      │<── Task完成通知 ───────│                      │
    │                      │─ 发送结果事件 ───────────────────────────────>│
    │                      │                         │                      │─ 处理结果
```

---

## 六、错误处理与容错

### 6.1 管理器崩溃处理

```
商场检测管理器崩溃 → 重启管理器进程 → 恢复状态 → 继续运行
```

### 6.2 Worker进程崩溃处理

```
管理器检测Worker崩溃 → 记录日志 → 重启Worker → 恢复数据采集
```

### 6.3 Task超时处理

```
管理器检测Task超时 → 标记超时 → 杀死进程/线程 → 通知商场 → 清理资源
```

---

## 七、性能指标

| 指标 | 目标值 |
|------|--------|
| 商场主循环延迟 | < 0.1ms |
| 驻留Worker轮询延迟 | < 1ms |
| Task提交延迟 | < 0.5ms |
| 管理器重启时间 | < 2s |
| 最大并发Task数 | 1000+ |

---

## 八、与现有系统集成

### 8.1 逐步迁移策略

1. **Phase 1**：保留现有架构，新增管理器作为可选
2. **Phase 2**：将驻留Worker轮询迁移到管理器
3. **Phase 3**：将Task状态检查迁移到管理器
4. **Phase 4**：完全移除商场主循环中的Worker管理逻辑

### 8.2 配置开关

```python
USE_WORKER_MANAGER = True  # 开关：是否使用独立管理器

if USE_WORKER_MANAGER:
    # 启动独立管理器进程
    start_worker_manager()
else:
    # 保留原架构（兼容模式）
    pass
```

---

## 九、启动流程

```python
# 主进程
def start_mall_system():
    # 1. 初始化SHM矢量
    init_shm_vectors()
    
    # 2. 启动管理器进程
    manager_process = multiprocessing.Process(
        target=worker_manager_entry,
        daemon=True
    )
    manager_process.start()
    
    # 3. 等待管理器就绪
    wait_for_manager_ready()
    
    # 4. 启动商场主进程
    mall_process = multiprocessing.Process(
        target=mall_core_entry,
        daemon=True
    )
    mall_process.start()
    
    # 5. 监控进程健康
    monitor_processes([manager_process, mall_process])
```

---

## 十、审计与监控

### 10.1 管理器状态SHM矢量

`VEC-MGR-STATUS` 实时暴露管理器状态：
```python
{
    "thread_pool": {
        "active_tasks": 10,
        "queue_size": 50,
        "max_workers": 16,
    },
    "process_pool": {
        "active_tasks": 3,
        "queue_size": 10,
        "max_workers": 4,
    },
    "resident_workers": {
        "count": 8,
        "running": 7,
        "paused": 1,
    },
    "uptime_seconds": 1234.5,
}
```
