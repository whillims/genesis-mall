# BP-0059 Worker通用交互标志系统（无锁互斥）

> **版本**: v2.0  
> **日期**: 2026-05-15  
> **功能域**: mall  
> **状态**: ACTIVE  
> **说明**: 从单一产品有效标志扩展为通用交互标志系统，支持输入接口标志、输出接口标志、状态标志等

---

## 一、设计概述

本蓝图定义Worker通用交互标志系统，实现**无锁互斥**的多标志位交互机制。

### 核心思想

```
Worker表达域：[0x00000001, 0x7FFFFFFF]（bit31=0）
商场表达域：[0x80000000, 0xFFFFFFFF]（bit31=1）

每个标志位独立遵循表达域分离原则
Worker只能在商场态时写入，写入后设置Worker态
商场只能在Worker态时消费/响应，处理后设置商场态
```

### 多标志位体系

Worker可拥有多个标志：

| 标志类型 | 名称 | 用途 | 偏移 |
|----------|------|------|------|
| 输入接口标志 | INPUT_FLAG | 输入数据就绪标志 | 0x1000 |
| 输出接口标志 | OUTPUT_FLAG | 输出数据就绪标志 | 0x1004 |
| 状态标志 | STATUS_FLAG | Worker状态/命令标志 | 0x1008 |

### 表达域分离原则

- **Worker表达域**：`0x00000001 ~ 0x7FFFFFFF`（Worker可写的状态/命令）
- **商场表达域**：`0x80000000 ~ 0xFFFFFFFF`（商场可写的状态/命令）
- **交集**：无，天然互斥
- **最高位判据**：bit31=0 → Worker态，bit31=1 → 商场态

### 互斥保证

- 每个标志位独立互斥
- Worker只能在商场态（bit31=1）时写入对应标志
- 写入完成后设置Worker态（bit31=0）
- 商场只能在Worker态（bit31=0）时处理对应标志
- 处理完成后设置商场态（bit31=1）
- 无需任何锁

---

## 二、标志位数据结构

### 2.1 SHM矢量头布局

```
┌─────────────────────────────────────────────────────────────────┐
│                        矢量数据区                                │
│  (实际产品数据)                                                  │
├─────────────────────────────────────────────────────────────────┤
│                  环形缓冲头部 (RING_HEADER_SIZE = 8)             │
│  [ring_write_pos:4字节] [ring_read_pos:4字节]                   │
├─────────────────────────────────────────────────────────────────┤
│                  通用交互标志区 (新增: 12字节)                   │
│  [input_flag:4字节] [output_flag:4字节] [status_flag:4字节]     │
│   输入就绪标志        输出就绪标志        状态/命令标志          │
├─────────────────────────────────────────────────────────────────┤
│                  元数据区 (4KB)                                  │
│  (JSON格式元数据)                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 常量定义

```python
# 标志位偏移定义
FLAG_INPUT_OFFSET = 0x1000      # 输入接口标志偏移
FLAG_OUTPUT_OFFSET = 0x1004     # 输出接口标志偏移  
FLAG_STATUS_OFFSET = 0x1008     # 状态标志偏移
FLAG_SIZE = 4                   # 每个标志4字节

# 表达域掩码
WORKER_DOMAIN_MASK = 0x7FFFFFFF     # Worker表达域（bit31=0）
MALL_DOMAIN_MASK = 0x80000000       # 商场表达域（bit31=1）

# ==================== 输入接口标志（INPUT_FLAG） ====================
# Worker态
INPUT_WORKER_READY = 0x00000001      # 输入数据就绪
INPUT_WORKER_EMPTY = 0x00000002      # 输入为空
INPUT_WORKER_STREAMING = 0x00000003  # 流式输入中
INPUT_WORKER_ERROR = 0x00000004      # 输入错误
# 商场态
INPUT_MALL_READY = 0x80000001        # 商场就绪，可接收输入
INPUT_MALL_CONSUMING = 0x80000002    # 正在消费输入
INPUT_MALL_PAUSED = 0x80000003       # 输入暂停

# ==================== 输出接口标志（OUTPUT_FLAG） ====================
# Worker态
OUTPUT_WORKER_READY = 0x00000001     # 输出数据就绪
OUTPUT_WORKER_EMPTY = 0x00000002     # 输出缓冲区空
OUTPUT_WORKER_STREAMING = 0x00000003 # 流式输出中
OUTPUT_WORKER_ERROR = 0x00000004     # 输出错误
# 商场态
OUTPUT_MALL_READY = 0x80000001       # 商场就绪，可接收输出
OUTPUT_MALL_CONSUMING = 0x80000002   # 正在消费输出
OUTPUT_MALL_FULL = 0x80000003        # 输出缓冲区满

# ==================== 状态标志（STATUS_FLAG） ====================
# Worker态
STATUS_WORKER_RUNNING = 0x00000001   # 运行中
STATUS_WORKER_PAUSED = 0x00000002    # 暂停
STATUS_WORKER_ERROR = 0x00000003     # 错误
STATUS_WORKER_STOPPED = 0x00000004   # 已停止
STATUS_WORKER_IDLE = 0x00000005      # 空闲
STATUS_WORKER_INIT = 0x00000006      # 初始化中
# 商场态
STATUS_MALL_ACTIVE = 0x80000001      # 商场活跃
STATUS_MALL_STANDBY = 0x80000002     # 商场待命
STATUS_MALL_SHUTDOWN = 0x80000003    # 商场关闭中
STATUS_MALL_ERROR = 0x80000004       # 商场错误
```

---

## 三、完整工作流程

### 3.1 输入接口标志流程

```
时间线 ──────────────────────────────────────────────────────────────────→

[Worker端]                            [商场端]

input_flag=INPUT_MALL_READY
    │
    ├─ 1. Worker检查input_flag
    │   → bit31=1（商场态），可写入输入
    │
    ├─ 2. Worker写入输入数据到矢量数据区
    │
    ├─ 3. Worker设置input_flag=INPUT_WORKER_READY
    │
    │                                   4. 商场轮询检查input_flag
    │                                   → bit31=0（Worker态），有输入
    │
    │                                   5. 商场读取输入数据
    │
    │                                   6. 商场设置input_flag=INPUT_MALL_READY
    │
input_flag=INPUT_MALL_READY
    │
    └─ 7. Worker检查标志，循环回到步骤1
```

### 3.2 输出接口标志流程

```
时间线 ──────────────────────────────────────────────────────────────────→

[Worker端]                            [商场端]

output_flag=OUTPUT_MALL_READY
    │
    ├─ 1. Worker检查output_flag
    │   → bit31=1（商场态），可写入输出
    │
    ├─ 2. Worker写入输出数据到矢量数据区
    │
    ├─ 3. Worker设置output_flag=OUTPUT_WORKER_READY
    │
    │                                   4. 商场轮询检查output_flag
    │                                   → bit31=0（Worker态），有输出
    │
    │                                   5. 商场读取输出数据
    │
    │                                   6. 商场设置output_flag=OUTPUT_MALL_READY
    │
output_flag=OUTPUT_MALL_READY
    │
    └─ 7. Worker检查标志，循环回到步骤1
```

### 3.3 状态标志流程

```
时间线 ──────────────────────────────────────────────────────────────────→

[Worker端]                            [商场端]

status_flag=STATUS_MALL_ACTIVE
    │
    ├─ 1. Worker更新自身状态
    │
    ├─ 2. Worker设置status_flag=STATUS_WORKER_RUNNING
    │
    │                                   3. 商场轮询检查status_flag
    │                                   → 获取Worker状态
    │
    │                                   4. 商场根据状态做出响应
    │
    │                                   5. 商场设置status_flag=STATUS_MALL_ACTIVE
    │
status_flag=STATUS_MALL_ACTIVE
    │
    └─ 6. Worker继续运行/更新状态
```

---

## 四、互斥逻辑保证

### 4.1 原子性保证

标志位读写是**4字节对齐**的原子操作：
- x86/x64架构：32位对齐读写天然原子
- 无需 `Lock`, `RLock`, 或其他同步机制

### 4.2 互斥状态机

```
┌───────────────────────────────────────────────────────────┐
│  状态转换图                                                 │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  [MALL态] ──(Worker写入)──> [WORKER态]                   │
│  (bit31=1)                  (bit31=0)                    │
│      ^                            │                      │
│      │                            │                      │
│      │                            │                      │
│      └───(商场消费)───────────────┘                      │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 4.3 权限分离

| 角色 | 可读 | 可写 | 说明 |
|------|------|------|------|
| Worker | ✓ | ✓ | 只写WORKER域（bit31=0） |
| 商场 | ✓ | ✓ | 只写MALL域（bit31=1） |

### 4.4 表达域检查函数

```python
def is_worker_domain(flag: int) -> bool:
    """检查是否在Worker表达域"""
    return (flag & MALL_DOMAIN_MASK) == 0


def is_mall_domain(flag: int) -> bool:
    """检查是否在商场表达域"""
    return (flag & MALL_DOMAIN_MASK) != 0
```

---

## 五、API设计

### 5.1 标志类型枚举

```python
from enum import Enum

class FlagType(Enum):
    INPUT = "input"      # 输入接口标志
    OUTPUT = "output"    # 输出接口标志
    STATUS = "status"    # 状态标志
```

### 5.2 Worker端API

```python
def flag_read(handle: ShmVectorHandle, flag_type: FlagType) -> int:
    """读取指定类型的标志"""
    offset = _get_flag_offset(flag_type)
    buf = handle.shm.buf
    raw = buf[offset:offset+FLAG_SIZE]
    return struct.unpack('<I', raw)[0]


def flag_write(handle: ShmVectorHandle, flag_type: FlagType, value: int):
    """写入指定类型的标志（原子操作）"""
    offset = _get_flag_offset(flag_type)
    buf = handle.shm.buf
    raw = struct.pack('<I', value)
    buf[offset:offset+FLAG_SIZE] = raw


def _get_flag_offset(flag_type: FlagType) -> int:
    """获取标志偏移"""
    offsets = {
        FlagType.INPUT: FLAG_INPUT_OFFSET,
        FlagType.OUTPUT: FLAG_OUTPUT_OFFSET,
        FlagType.STATUS: FLAG_STATUS_OFFSET,
    }
    return handle.offset + offsets[flag_type]


# ==================== 输入接口标志操作 ====================
def input_flag_check_writable(handle: ShmVectorHandle) -> bool:
    """检查输入接口是否可写入（商场态）"""
    flag = flag_read(handle, FlagType.INPUT)
    return is_mall_domain(flag)


def input_write_with_flag(handle: ShmVectorHandle, input_data: dict) -> bool:
    """写入输入数据并设置输入就绪标志"""
    if not input_flag_check_writable(handle):
        return False
    
    # 写入输入数据
    success = write_json(handle, input_data)
    if not success:
        return False
    
    # 设置输入就绪标志
    flag_write(handle, FlagType.INPUT, INPUT_WORKER_READY)
    return True


# ==================== 输出接口标志操作 ====================
def output_flag_check_writable(handle: ShmVectorHandle) -> bool:
    """检查输出接口是否可写入（商场态）"""
    flag = flag_read(handle, FlagType.OUTPUT)
    return is_mall_domain(flag)


def output_write_with_flag(handle: ShmVectorHandle, output_data: dict) -> bool:
    """写入输出数据并设置输出就绪标志"""
    if not output_flag_check_writable(handle):
        return False
    
    # 写入输出数据
    success = write_json(handle, output_data)
    if not success:
        return False
    
    # 设置输出就绪标志
    flag_write(handle, FlagType.OUTPUT, OUTPUT_WORKER_READY)
    return True


# ==================== 状态标志操作 ====================
def status_flag_update(handle: ShmVectorHandle, worker_state: int) -> bool:
    """更新Worker状态标志"""
    if not is_worker_domain(worker_state):
        return False
    
    # 设置状态标志
    flag_write(handle, FlagType.STATUS, worker_state)
    return True
```

### 5.3 商场端API

```python
# ==================== 输入接口标志操作 ====================
def input_flag_check_ready(handle: ShmVectorHandle) -> bool:
    """检查输入接口是否有就绪数据（Worker态）"""
    flag = flag_read(handle, FlagType.INPUT)
    return is_worker_domain(flag)


def input_flag_get_state(handle: ShmVectorHandle) -> int:
    """获取输入接口状态"""
    return flag_read(handle, FlagType.INPUT)


def input_consume_and_reset(handle: ShmVectorHandle) -> Optional[dict]:
    """消费输入数据并复位标志"""
    if not input_flag_check_ready(handle):
        return None
    
    # 读取输入数据
    input_data = read_json(handle)
    
    # 处理输入数据
    # ...
    
    # 复位输入标志
    flag_write(handle, FlagType.INPUT, INPUT_MALL_READY)
    
    return input_data


# ==================== 输出接口标志操作 ====================
def output_flag_check_ready(handle: ShmVectorHandle) -> bool:
    """检查输出接口是否有就绪数据（Worker态）"""
    flag = flag_read(handle, FlagType.OUTPUT)
    return is_worker_domain(flag)


def output_flag_get_state(handle: ShmVectorHandle) -> int:
    """获取输出接口状态"""
    return flag_read(handle, FlagType.OUTPUT)


def output_consume_and_reset(handle: ShmVectorHandle) -> Optional[dict]:
    """消费输出数据并复位标志"""
    if not output_flag_check_ready(handle):
        return None
    
    # 读取输出数据
    output_data = read_json(handle)
    
    # 处理输出数据
    # ...
    
    # 复位输出标志
    flag_write(handle, FlagType.OUTPUT, OUTPUT_MALL_READY)
    
    return output_data


# ==================== 状态标志操作 ====================
def status_flag_get(handle: ShmVectorHandle) -> int:
    """获取Worker状态"""
    return flag_read(handle, FlagType.STATUS)


def status_flag_is_worker_state(handle: ShmVectorHandle) -> bool:
    """检查状态标志是否为Worker态"""
    flag = flag_read(handle, FlagType.STATUS)
    return is_worker_domain(flag)


def status_flag_reset(handle: ShmVectorHandle):
    """复位状态标志为商场态"""
    flag_write(handle, FlagType.STATUS, STATUS_MALL_ACTIVE)
```

### 5.4 表达域检查函数

```python
def is_worker_domain(flag: int) -> bool:
    """检查是否在Worker表达域"""
    return (flag & MALL_DOMAIN_MASK) == 0


def is_mall_domain(flag: int) -> bool:
    """检查是否在商场表达域"""
    return (flag & MALL_DOMAIN_MASK) != 0


def is_input_ready(flag: int) -> bool:
    """检查输入标志是否为就绪状态"""
    return is_worker_domain(flag) and flag == INPUT_WORKER_READY


def is_output_ready(flag: int) -> bool:
    """检查输出标志是否为就绪状态"""
    return is_worker_domain(flag) and flag == OUTPUT_WORKER_READY
```

---

## 六、与现有系统集成

### 6.1 集成位置

`src/shm/vector_manager.py` - `ShmVectorHandle` 和 `ShmVectorManager`

### 6.2 头偏移计算

```
元数据区: 0x0000-0x1000 (4KB)
标志位区: 0x1000-0x100C (12字节)
  - input_flag: 0x1000-0x1004
  - output_flag: 0x1004-0x1008
  - status_flag: 0x1008-0x100C
环形缓冲头: 0x100C-0x1014 (原RING_HEADER_SIZE位置)
数据区: 0x1014+
```

### 6.3 与驻留Worker集成

修改 `resident_worker_poller()`，检查多个标志位：

```python
async def resident_worker_poller():
    while MALL_STATE["running"]:
        for worker_id, config in RESIDENT_WORKERS.items():
            vector_id = config["vector_id"]
            
            # 检查状态标志
            status = status_flag_get(vector_id)
            
            # 如果Worker状态异常，跳过
            if is_worker_domain(status) and status == STATUS_WORKER_ERROR:
                log_error(worker_id, "Worker in error state")
                status_flag_reset(vector_id)
                continue
            
            # 检查输入标志
            if input_flag_check_ready(vector_id):
                input_data = input_consume_and_reset(vector_id)
                if input_data is not None:
                    for subscriber_id in config.get("input_subscribers", []):
                        spawn_task(subscriber_id, input_data)
            
            # 检查输出标志
            if output_flag_check_ready(vector_id):
                output_data = output_consume_and_reset(vector_id)
                if output_data is not None:
                    for subscriber_id in config.get("output_subscribers", []):
                        spawn_task(subscriber_id, output_data)
        
        await asyncio.sleep(0.001)
```

### 6.4 向后兼容

保留原有API接口，内部映射到新的多标志位系统：

```python
# 向后兼容API
def product_flag_check_writable(handle: ShmVectorHandle) -> bool:
    """检查是否可写入（映射到输入标志）"""
    return input_flag_check_writable(handle)


def product_write_with_flag(handle: ShmVectorHandle, product: dict, worker_state: int = INPUT_WORKER_READY) -> bool:
    """写入产品并设置标志（映射到输入标志）"""
    return input_write_with_flag(handle, product)


def product_flag_check_has_product(handle: ShmVectorHandle) -> bool:
    """检查是否有产品（映射到输入标志）"""
    return input_flag_check_ready(handle)


def product_consume_and_reset(handle: ShmVectorHandle, mall_state: int = INPUT_MALL_READY) -> Optional[dict]:
    """消费产品并复位标志（映射到输入标志）"""
    return input_consume_and_reset(handle)
```

---

## 七、键盘Worker示例流程（使用多标志位系统）

### 7.1 键盘Worker写入输入数据

```
键盘Worker:
1. input_flag_check_writable("VEC-KEYBOARD") → True (INPUT_MALL_READY)
2. keyboard_echo("hello") → {"key":"h", "timestamp":...}
3. input_write_with_flag("VEC-KEYBOARD", input_data)
   → 写入矢量，设置input_flag=0x00000001 (INPUT_WORKER_READY)
4. status_flag_update("VEC-KEYBOARD", STATUS_WORKER_RUNNING)
   → 设置status_flag=0x00000001 (运行中)
```

### 7.2 商场处理输入数据

```
商场:
5. status = status_flag_get("VEC-KEYBOARD") → 0x00000001 (STATUS_WORKER_RUNNING)
6. input_flag_check_ready("VEC-KEYBOARD") → True (INPUT_WORKER_READY)
7. input_data = input_consume_and_reset("VEC-KEYBOARD")
   → 读取输入数据
   → 更新相关矢量
   → 通知 "console-echo", "key-logger"
   → 设置input_flag=0x80000001 (INPUT_MALL_READY)
8. status_flag_reset("VEC-KEYBOARD")
   → 设置status_flag=0x80000001 (STATUS_MALL_ACTIVE)
```

### 7.3 输出数据流程示例

```
Worker端（产生输出）:
1. output_flag_check_writable("VEC-SPECTRUM") → True (OUTPUT_MALL_READY)
2. generate_spectrum_data() → {"freq": [...], "power": [...]}
3. output_write_with_flag("VEC-SPECTRUM", output_data)
   → 设置output_flag=0x00000001 (OUTPUT_WORKER_READY)

商场端（消费输出）:
4. output_flag_check_ready("VEC-SPECTRUM") → True
5. output_data = output_consume_and_reset("VEC-SPECTRUM")
   → 设置output_flag=0x80000001 (OUTPUT_MALL_READY)
6. 转发给下游消费者...
```

### 7.4 状态标志错误处理示例

```
Worker端（异常情况）:
1. status_flag_update("VEC-XXX", STATUS_WORKER_ERROR)
   → 设置status_flag=0x00000003 (ERROR)

商场端:
2. status = status_flag_get("VEC-XXX") → 0x00000003 (STATUS_WORKER_ERROR)
3. log_error("Worker reported error")
4. status_flag_reset("VEC-XXX")
   → 设置status_flag=0x80000001 (STATUS_MALL_ACTIVE)
```

---

## 八、性能指标

| 指标 | 目标值 |
|------|--------|
| 标志位读写延迟 | < 10ns |
| 完整产品写入周期 | < 100μs |
| 完整产品消费周期 | < 100μs |
| 标志位检查轮询开销 | 可忽略 |
| 无锁保证 | ✓ |

---

## 九、审计与验证

所有标志位操作需记录审计日志：

```
[audit] product_flag_check_writable(vec=VEC-KEYBOARD, result=True)
[audit] product_write_with_flag(vec=VEC-KEYBOARD, success=True)
[audit] product_flag_check_has_product(vec=VEC-KEYBOARD, result=True)
[audit] product_consume_and_reset(vec=VEC-KEYBOARD, product_consumed=True)
```
