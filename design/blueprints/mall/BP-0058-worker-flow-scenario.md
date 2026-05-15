# BP-0058 Worker流程场景：驻留Worker+环形缓冲+任务调度

> **版本**: v1.0  
> **日期**: 2026-05-15  
> **功能域**: mall  
> **状态**: ACTIVE  

---

## 一、场景概述

本文档定义商场核心的Worker运行时调度逻辑，以键盘Worker为典型场景，描述完整的数据流转和任务生命周期。

### 核心概念

| 概念 | 定义 |
|------|------|
| 驻留Worker | 常驻线程，持续采集/产生数据，不退出 |
| 环形缓冲 | SHM环形矢量，驻留Worker写入，商场读取 |
| 产品 | 驻留Worker写入环形缓冲的一帧数据 |
| 消费Task | 处理驻留Worker产品的异步任务 |
| 非阻塞检查 | 商场轮询Task状态，未完成则跳过 |
| 超时杀死 | Task超过时限自动终止 |

---

## 二、键盘Worker场景（完整流程）

```
时间线 ──────────────────────────────────────────────────────────→

[驻留Worker]     [商场主循环]                [消费Task]
键盘Worker线程    asyncio事件循环              异步任务
    │                 │                         │
    │ 1.用户按键      │                         │
    │ 2.写入环形缓冲  │                         │
    │ 3.更新wpos      │                         │
    │                 │                         │
    │          4.检查环形缓冲                    │
    │          5.发现新数据(rpos<wpos)           │
    │          6.读取产品帧                      │
    │          7.更新rpos                        │
    │          8.查找订阅该产品的消费者           │
    │          9.为每个消费者创建Task             │
    │                 │                         │
    │                 │            10.Task开始执行
    │                 │            11.处理键盘数据
    │                 │            12.写入输出矢量
    │                 │                         │
    │          13.检查Task状态                    │
    │          14.未完成→跳过                    │
    │          15.已完成→读取输出                │
    │          16.超时→杀死Task                  │
    │                 │                         │
    │          17.处理Task输出                    │
    │          18.通知下游消费者                  │
```

---

## 三、驻留Worker模型

### 3.1 驻留Worker生命周期

```
[SPAWN] → [RUNNING] → [RUNNING] → ... → [SHUTDOWN]
              │
              │ 持续循环:
              │   1. 采集/接收数据
              │   2. 写入环形缓冲
              │   3. 更新ring_write_pos
              │   4. await asyncio.sleep(interval)
              │
              └──→ 商场检查环形缓冲
```

### 3.2 驻留Worker注册

```python
RESIDENT_WORKERS = {
    "keyboard-echo": {
        "worker_id": "keyboard-echo",
        "worker_type": "producer",
        "vector_id": "VEC-KEYBOARD",
        "interval_ms": 10,
        "subscribers": ["console-echo", "key-logger"],
        "state": "running",
    },
    "rtlsdr-spectrum": {
        "worker_id": "rtlsdr-spectrum",
        "worker_type": "producer",
        "vector_id": "VEC-SPECTRUM",
        "interval_ms": 500,
        "subscribers": ["spectrum-display", "frequency-analyzer"],
        "state": "running",
    },
    "sysinfo-collector": {
        "worker_id": "sysinfo-collector",
        "worker_type": "producer",
        "vector_id": "VEC-SYSINFO",
        "interval_ms": 2000,
        "subscribers": ["sysdisplay"],
        "state": "running",
    },
}
```

### 3.3 驻留Worker线程函数

```python
async def resident_worker_loop(worker_id, vector_id, produce_fn, interval_ms):
    while MALL_STATE["running"]:
        product = produce_fn()
        if product is not None:
            ring_write(vector_id, product)
        await asyncio.sleep(interval_ms / 1000.0)
```

---

## 四、商场主循环（asyncio）

### 4.1 主循环架构

```python
async def mall_main_loop():
    await asyncio.gather(
        resident_worker_poller(),      # 轮询驻留Worker环形缓冲
        task_status_checker(),         # 检查Task状态
        queue_request_listener(),      # 处理外部请求
        health_check_loop(),           # 健康检查
    )
```

### 4.2 驻留Worker轮询器

```python
async def resident_worker_poller():
    while MALL_STATE["running"]:
        for worker_id, config in RESIDENT_WORKERS.items():
            if config["state"] != "running":
                continue
            vector_id = config["vector_id"]
            new_frames = ring_read_new(vector_id)
            if not new_frames:
                continue
            for frame in new_frames:
                for subscriber_id in config["subscribers"]:
                    task_id = spawn_task(subscriber_id, frame)
        await asyncio.sleep(0.001)
```

### 4.3 Task状态检查器

```python
async def task_status_checker():
    while MALL_STATE["running"]:
        for task_id, task in list(MALL_STATE["tasks"].items()):
            if task["status"] == "running":
                if time.time() - task["started_at"] > task["timeout_seconds"]:
                    kill_task(task_id)
                    task["status"] = "timeout"
                continue
            if task["status"] == "completed":
                output = read_task_output(task_id)
                notify_downstream(task_id, output)
                cleanup_task(task_id)
            if task["status"] in ("failed", "timeout"):
                cleanup_task(task_id)
        await asyncio.sleep(0.01)
```

---

## 五、Task生命周期

### 5.1 Task状态机

```
[CREATED] → [RUNNING] → [COMPLETED] → [CLEANED]
                │
                ├──→ [FAILED] → [CLEANED]
                │
                └──→ [TIMEOUT] → [CLEANED]
```

### 5.2 Task结构

```python
{
    "task_id": "task-001",
    "consumer_id": "console-echo",
    "input_vector_id": "VEC-KEYBOARD",
    "output_vector_id": "VEC-CONSOLE-OUT",
    "status": "running",
    "started_at": 1715769600.0,
    "timeout_seconds": 5,
    "worker_fn": "keyboard_echo_handler",
    "input_data": {...},
    "output_data": None,
}
```

### 5.3 非阻塞检查规则

| Task状态 | 商场行为 |
|----------|----------|
| RUNNING | 跳过，下次再查 |
| COMPLETED | 读取输出，通知下游，清理 |
| FAILED | 记录错误，清理 |
| TIMEOUT | 杀死Task，清理 |

### 5.4 超时杀死

- 每个Task创建时指定 `timeout_seconds`
- `task_status_checker()` 每轮检查运行时间
- 超时则调用 `kill_task()` 终止
- 默认超时：5秒（可按消费者类型配置）

---

## 六、键盘Worker场景详细流程

### 6.1 步骤1-3：驻留Worker采集数据

```
键盘Worker线程:
  1. keyboard_echo("hello") → {"status":"success", "displayed":True, ...}
  2. ring_write("VEC-KEYBOARD", frame)
  3. 更新 ring_write_pos
```

### 6.2 步骤4-9：商场检测并分发

```
商场主循环 (resident_worker_poller):
  4. ring_read_new("VEC-KEYBOARD") → [frame1, frame2, ...]
  5. 查找 RESIDENT_WORKERS["keyboard-echo"]["subscribers"]
     → ["console-echo", "key-logger"]
  6. spawn_task("console-echo", frame1) → task-001
  7. spawn_task("key-logger", frame1) → task-002
```

### 6.3 步骤10-12：Task执行

```
Task task-001 (console-echo):
  10. 从输入矢量读取数据
  11. console_consumer(data) → {"displayed": True, ...}
  12. 写入输出矢量 VEC-CONSOLE-OUT
```

### 6.4 步骤13-18：商场检查Task

```
商场主循环 (task_status_checker):
  13. 检查 task-001: status=running → 跳过
  14. 下一轮: task-001: status=completed
  15. 读取输出: read_task_output("task-001")
  16. 通知下游消费者
  17. cleanup_task("task-001")
```

---

## 七、与现有架构的集成

### 7.1 与asyncio主循环集成

`resident_worker_poller()` 和 `task_status_checker()` 作为协程加入 `mall_main_loop()`：

```python
async def mall_main_loop(mode="queue", **kwargs):
    tasks = []
    tasks.append(resident_worker_poller())
    tasks.append(task_status_checker())
    # ... 其他监听器
    await asyncio.gather(*tasks)
```

### 7.2 与SHM矢量空间集成

- 驻留Worker通过SHM环形矢量写入数据
- 商场通过 `ring_read_new()` 非阻塞读取
- ring位置在SHM元数据中，零延迟协调

### 7.3 与消费者类型集成

| 消费者类型 | 典型驻留Worker | Task超时 |
|-----------|---------------|---------|
| CT | 键盘采集 | 1秒 |
| CF | 文件监控 | 10秒 |
| CN | 网络接收 | 5秒 |
| CA | 数据分析 | 30秒 |
| SP | 频谱处理 | 60秒 |

---

## 八、性能指标

| 指标 | 目标值 |
|------|--------|
| 驻留Worker轮询延迟 | < 1ms |
| Task创建到启动 | < 0.5ms |
| Task状态检查周期 | 10ms |
| 键盘输入到显示 | < 5ms |
| 超时杀死延迟 | < 15ms |
