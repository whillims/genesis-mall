# BP-0057 SHM双通道架构（控制通道+数据通道）

> **版本**: v2.0  
> **日期**: 2026-05-15  
> **功能域**: shm  
> **状态**: ACTIVE  

---

## 一、背景与动机

### 问题

信号处理Worker（如scipy.periodogram）需要大量内存：
- 输入数据：≥200MB（原始IQ采样数据）
- 输出数据：≥400MB（频谱分析结果）
- 总计：≥600MB

当前SHM架构限制：
- 默认总大小：16MB
- 最大单矢量配额：CC型 1GB（但总空间不够）
- 单一SHM段无法承载大数据交换

### 解决方案

**双通道架构**：
- **控制通道 (SHM)**：创建者通过商场SHM发布环形缓冲的矢量信息给需求者
- **数据通道 (mmap)**：大数据块零拷贝传输，创建者零拷贝写入（套用shm.buf方式）

### 设计原则

1. **双平台**：Windows和Linux各自一套mmap代码，统一接口
2. **单一创建者**：每个mmap块只有一个创建者，创建者维护mmap生命周期，自行管理环形缓冲
3. **零拷贝**：创建者通过mmap buffer接口直接写入 `mm[offset:n]=data`，与SHM的 `shm.buf[offset:n]=data` 方式一致
4. **SHM发布**：创建者通过商场SHM矢量发布环形缓冲信息（mmap_path、size、ring位置、state），需求者订阅获取
5. **进程隔离**：Worker与SHM管理器不在同一进程，避免卡死
6. **配额控制**：按消费者类型限制mmap总使用量

---

## 二、架构设计

### 双通道架构图

```
创建者进程                     商场SHM(控制通道)              消费者进程
┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│ MmapCreator       │     │ SHM矢量(小,≤1MB)     │     │ MmapReader        │
│                   │     │                      │     │                   │
│ 1. create mmap    │     │ 矢量信息:             │     │ 3. 订阅SHM矢量    │
│ 2. 零拷贝写入     │     │   mmap_path          │     │ 4. 获取mmap_path  │
│    mm[offset:n]=  │──→  │   data_size          │──→  │ 5. 只读映射mmap   │
│      data         │     │   ring_write_pos     │     │ 6. 零拷贝读取     │
│ 3. 管理环形缓冲   │     │   ring_read_pos      │     │    mm[offset:n]   │
│ 4. 更新ring位置   │     │   state              │     │ 7. 更新ring位置   │
│                   │     │                      │     │                   │
│    mmap数据通道    │     └──────────────────────┘     │    mmap数据通道   │
│    (200MB+,零拷贝) │←────────────────────────────────→│    (只读,零拷贝)  │
└──────────────────┘    直接访问同一mmap文件            └──────────────────┘
```

### 数据流

```
1. 创建者申请mmap块
   Creator → MmapDataManager.allocate() → 创建mmap文件 → 返回MmapCreator

2. 创建者创建SHM矢量发布信息
   Creator → ShmVectorManager.create_vector(mode="mmap") 
           → SHM矢量包含: mmap_path, size, ring_write_pos, ring_read_pos, state

3. 创建者零拷贝写入数据
   Creator → mm = creator.get_buffer()
           → mm[HEADER+offset : HEADER+offset+len] = data  (零拷贝，与shm.buf方式一致)

4. 创建者管理环形缓冲
   Creator → creator.ring_write(frame)  → 更新mmap中ring_write_pos
           → SHM矢量同步更新ring_write_pos（供商场监控）

5. 消费者订阅SHM矢量
   Consumer → ShmVectorManager.open_vector() → 获取mmap_path, ring位置

6. 消费者只读映射mmap
   Consumer → MmapReader.open() → mm = reader.get_buffer()
           → data = bytes(mm[HEADER+offset : HEADER+offset+len])  (零拷贝读取)

7. 消费者读取环形缓冲
   Consumer → reader.ring_read() → 读取mmap中ring_read_pos → 读取数据 → 更新ring_read_pos

8. 创建者释放mmap
   Creator → MmapDataManager.release() → 删除mmap文件
```

---

## 三、双平台实现

### 平台差异

| 特性 | Linux | Windows |
|------|-------|---------|
| 基础目录 | `/tmp/mall_mmap/` | `%TEMP%\mall_mmap\` |
| 文件创建 | `os.open(O_CREAT\|O_RDWR)` + `os.ftruncate()` | `open('wb')` + `f.seek()` + `f.write()` |
| 写映射 | `mmap.mmap(fd, size, ACCESS_WRITE)` | `mmap.mmap(fileno, size, ACCESS_WRITE)` |
| 读映射 | `mmap.mmap(fd, size, ACCESS_READ)` | `mmap.mmap(fileno, size, ACCESS_READ)` |
| 文件权限 | `0o600` | 默认ACL |
| 删除语义 | unlink后映射仍有效 | 文件打开时不可删除 |

### 平台抽象接口

```python
class _MmapPlatform:
    @staticmethod
    def create_file(path: str, size: int) -> None
    
    @staticmethod
    def map_write(path: str, size: int) -> mmap.mmap
    
    @staticmethod
    def map_read(path: str, size: int) -> mmap.mmap
```

---

## 四、MmapCreator 设计（单一创建者）

### 核心原则

1. **唯一性**：每个mmap块只有一个创建者，创建者拥有完整生命周期
2. **零拷贝**：通过 `get_buffer()` 返回mmap对象，使用buffer协议直接写入
3. **环形缓冲**：创建者自行管理环形缓冲，ring位置存储在mmap文件头中

### mmap文件布局

```
偏移量     大小      内容
[0:4]      4B       magic (0x4D414C4C = "MALL")
[4:8]      4B       data_size (uint32, big-endian)
[8:16]     8B       timestamp (uint64, big-endian)
[16:64]    48B      vec_id (null-padded UTF-8)
[64:72]    8B       ring_write_pos (uint64, little-endian)
[72:80]    8B       ring_read_pos (uint64, little-endian)
[80:]      变长      数据区 (data_size - 16 字节有效环形缓冲区)
```

### 核心接口

```python
class MmapCreator:
    def __init__(self, vec_id: str, size: int, path: str)
    def create(self) -> bool
    def get_buffer(self) -> mmap.mmap          # 零拷贝，类似shm.buf
    def write(self, offset: int, data: bytes) -> bool
    def read(self, offset: int, length: int) -> bytes
    def ring_write(self, frame: bytes) -> bool  # 创建者管理环形缓冲
    def ring_read(self, timeout_ms: int) -> bytes
    def flush(self) -> None
    def close(self) -> None                     # 关闭映射，保留文件
    def release(self) -> None                   # 关闭映射，删除文件
```

### 零拷贝写入方式

```python
# SHM方式（对照）
shm.buf[offset:offset+len(data)] = data

# mmap方式（一致）
mm = creator.get_buffer()
mm[HEADER_SIZE+offset:HEADER_SIZE+offset+len(data)] = data
```

---

## 五、MmapReader 设计（消费者只读）

### 核心原则

1. **只读**：消费者只能读取，不能写入数据区
2. **零拷贝**：通过 `get_buffer()` 返回只读mmap对象
3. **环形读取**：消费者通过ring_read_pos跟踪读取位置

### 核心接口

```python
class MmapReader:
    def __init__(self, vec_id: str, path: str, size: int)
    def open(self) -> bool
    def get_buffer(self) -> mmap.mmap          # 只读零拷贝
    def read(self, offset: int, length: int) -> bytes
    def ring_read(self, timeout_ms: int) -> bytes
    def close(self) -> None
```

---

## 六、MmapDataManager 设计（分配器/注册表）

### 职责

- mmap块的分配与注册（配额检查）
- 创建MmapCreator实例
- 创建MmapReader实例
- 生命周期跟踪
- 过期块清理

### 核心接口

```python
class MmapDataManager:
    def allocate(self, vec_id, size, owner, consumer_type) -> MmapAllocationResult
    def get_creator(self, vec_id) -> MmapCreator
    def open_reader(self, vec_id) -> MmapReader
    def release(self, vec_id) -> dict
    def get_info(self, vec_id) -> dict
    def list_blocks(self) -> list
    def cleanup_expired(self, max_age_seconds) -> int
    def get_stats(self) -> MmapStats
```

---

## 七、消费者类型扩展

### 新增SP类型（Signal Processing）

| 类型 | 名称 | SHM配额 | mmap配额 | 说明 |
|------|------|---------|---------|------|
| SP | 信号处理型 | 1MB | 1GB | 频谱分析、信号处理 |

### 更新后的完整配额表

| 类型 | SHM配额 | mmap配额 | 典型场景 |
|------|---------|---------|---------|
| CT | 1MB | 10MB | 终端型 |
| CF | 10MB | 100MB | 文件型 |
| CN | 100MB | 500MB | 网络型 |
| CA | 100MB | 500MB | 分析型 |
| CC | 1GB | 2GB | 控制型 |
| CCn | 10MB | 50MB | 容器型 |
| **SP** | **1MB** | **1GB** | **信号处理型** |

---

## 八、与ShmVectorManager集成

### SHM控制通道矢量内容

创建者在SHM中发布的矢量信息（JSON格式）：

```json
{
    "mmap_path": "/tmp/mall_mmap/sp_in_001.mmap",
    "mmap_size": 209715200,
    "mode": "mmap",
    "state": "processing",
    "ring_write_pos": 1024,
    "ring_read_pos": 0,
    "consumer_type": "SP"
}
```

### 集成方式

ShmVectorManager新增`mode="mmap"`矢量类型：
- `mode="direct"`: 传统SHM直接内存（小数据）
- `mode="ring"`: 环形缓冲区（流数据）
- `mode="mmap"`: mmap数据通道（大数据，零拷贝）

### 矢量创建流程

```python
# 小数据: 传统SHM
vm.create_vector("small_vec", size=1024, mode="direct")

# 流数据: 环形缓冲区
vm.create_vector("stream_vec", size=65536, mode="ring")

# 大数据: mmap零拷贝（auto模式，>8MB自动切换）
vm.create_vector("periodogram_input", size=200*1024*1024, mode="auto")
vm.create_vector("periodogram_output", size=400*1024*1024, mode="mmap")
```

---

## 九、安全约束

1. **路径限制**：mmap文件只能在指定基础目录下
2. **大小限制**：单块mmap不超过消费者类型配额
3. **单一创建者**：只有创建者可以写入和释放mmap块
4. **只读消费者**：消费者只能只读映射，不能修改数据
5. **自动清理**：超过max_age_seconds的孤立块自动清理
6. **权限控制**：Linux下mmap文件权限0600，Windows下默认ACL

---

## 十、性能指标

| 指标 | 目标值 |
|------|--------|
| 200MB mmap分配时间 | < 100ms |
| 200MB mmap映射时间 | < 10ms |
| 零拷贝写入延迟 | < 1μs (buffer赋值) |
| 零拷贝读取延迟 | < 1μs (buffer切片) |
| 400MB mmap释放时间 | < 50ms |
| 自动清理周期 | 60秒 |

---

## 十一、asyncio事件循环架构

### 设计原则

商场核心采用asyncio事件循环实现"可控固定任务运行序列"：
- 单线程执行，协程在`await`点让出控制权
- 执行顺序确定性，不需要任何锁
- I/O操作不阻塞，`await`替代`time.sleep()`
- 多个I/O源并发监听，但串行处理

### 主循环架构

```python
async def mall_main_loop():
    await asyncio.gather(
        shm_request_listener(),
        queue_request_listener(),
        vector_trade_listener(),
        worker_result_collector(),
        health_check_loop(),
    )

async def shm_request_listener():
    while running:
        await asyncio.sleep(0.001)
        if req_ready.value == 0:
            continue
        req_ready.value = 0
        requests = shm_read_json(req_shm)
        for req in requests:
            result = await dispatch_request(req)
            responses.append(result)
        shm_write_json(resp_shm, responses)
        resp_ready.value = 1

async def queue_request_listener():
    while running:
        req = await loop.run_in_executor(None, req_queue.get, True, 0.1)
        result = await dispatch_request(req)
        resp_queue.put(result)

async def dispatch_request(req):
    action = req.get("action", "")
    params = req.get("params", {})
    handler = HANDLER_REGISTRY.get(action)
    if handler is None:
        return {"status": "error", "message": f"未知操作: {action}"}
    if asyncio.iscoroutinefunction(handler):
        return await handler(params)
    return handler(params)
```

### 与禁令第七条的关系

| 同步轮询（违规） | asyncio（合规） |
|---|---|
| `time.sleep(0.005)` 阻塞 | `await asyncio.sleep(0.001)` 让出 |
| `queue.get(timeout)` 阻塞 | `await loop.run_in_executor()` 非阻塞 |
| 多个I/O源需要多线程 | `asyncio.gather()` 协程并发 |
| 需要Lock保护共享状态 | 单线程事件循环，无并发访问 |
