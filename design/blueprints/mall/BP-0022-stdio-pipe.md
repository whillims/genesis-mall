# BP-0003-stdio-pipe.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**：BP-0003  
> **产品名称**：stdio-pipe（标准输入输出管道）  
> **产品类型**：基础通道产品  
> **优先级**：P0  
> **依赖**：BP-0001-keyboard, BP-0002-console, BP-0004-shm-manager  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`stdio-pipe` 是商场内置的**默认字节流通道产品**，承担键盘生产者（BP-0001）与控制台消费者（BP-0002）之间的数据转储与协议适配职责。它不直接操作硬件，而是作为SHM上的**逻辑管道**存在，定义标准输入输出数据的缓冲策略、编码协议、流量控制与EOF语义。

在商场创世阶段，`stdio-pipe` 是首个被加载的通道产品，为后续所有生产者-消费者交互提供**参考实现模板**。

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 通信通道 | 仅通过商场SHM段读写，禁止管道/队列/socket |
| 状态管理 | 所有状态必须显式存储于SHM矢量，禁止函数内隐式静态变量 |
| 错误处理 | 错误码通过返回值传递，异常信息写入SHM错误日志段 |
| 编译权 | 商场拥有最终编译与加载权，本产品代码由AICoder外置生成授权文件 |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **字节流段** | SHM中专门用于存储stdio原始字节的命名段，按环形缓冲区组织 |
| **帧头** | 每批输入数据的前8字节元数据：`[4字节长度 | 2字节来源ID | 1字节类型 | 1字节保留]` |
| **水线标记** | 缓冲区读指针（`rd_ptr`）与写指针（`wr_ptr`），原子更新 |
| **EOF令牌** | 特殊单字节标记 `0x04`（EOT），表示输入流结束 |
| **回显开关** | SHM配置位，控制键盘输入是否立即回显到控制台 |

---

## 4. 接口定义

### 4.1 生产者侧接口（键盘调用）

```c
// 将键盘原始扫描码写入stdio字节流段
int stdio_write_keycode(
    SharedMemory* shm,      // 商场SHM根句柄
    uint16_t producer_id,   // 键盘生产者ID（如 0x0001）
    uint8_t* keycode_buf,   // 扫描码缓冲区
    uint32_t len,           // 字节数
    uint8_t flags           // 位掩码：bit0=回显请求, bit1=立即刷新
);

// 刷新缓冲区，强制将暂存数据提交到SHM并通知消费者
int stdio_flush(SharedMemory* shm, uint16_t producer_id);
```

### 4.2 消费者侧接口（控制台调用）

```c
// 从stdio字节流段读取一批数据
int stdio_read_frame(
    SharedMemory* shm,      // 商场SHM根句柄
    uint16_t consumer_id,   // 控制台消费者ID（如 0x0002）
    uint8_t* out_buf,       // 输出缓冲区（由调用者分配）
    uint32_t buf_cap,       // 缓冲区容量
    uint32_t* out_len,      // 实际读取字节数（出参）
    uint8_t* out_type       // 数据类型（出参）：0=文本, 1=控制码, 2=EOF
);

// 查询当前可读字节数（非阻塞）
int stdio_poll_readable(SharedMemory* shm, uint32_t* available_bytes);
```

### 4.3 商场管理接口（main或调度器调用）

```c
// 初始化stdio管道在SHM中的数据结构
int stdio_pipe_init(SharedMemory* shm, uint32_t buffer_size, uint8_t default_echo);

// 重置stdio管道（清空缓冲区，指针归零）
int stdio_pipe_reset(SharedMemory* shm);

// 获取stdio管道状态快照
int stdio_pipe_status(
    SharedMemory* shm,
    uint32_t* total_written,
    uint32_t* total_read,
    uint8_t* is_overrun
);
```

---

## 5. SHM数据结构设计

### 5.1 stdio控制块（固定偏移）

```c
// 位于SHM基地址 + 0x0100 处
struct stdio_control_block {
    uint32_t magic;              // "STIO" = 0x5354494F
    uint32_t version;            // 0x00010000
    uint32_t buf_size;           // 环形缓冲区总容量
    uint32_t wr_ptr;             // 写指针（原子更新）
    uint32_t rd_ptr;             // 读指针（原子更新）
    uint32_t bytes_in_buf;       // 当前缓冲字节数
    uint8_t  echo_enabled;       // 回显开关
    uint8_t  overrun_flag;       // 溢出标记（写满未读）
    uint8_t  reserved[6];        // 对齐填充
    // 总计 32 字节
};
```

### 5.2 字节流环形缓冲区

```c
// 位于SHM基地址 + 0x0120 处，长度为 buf_size
uint8_t stdio_byte_ring[];       // 原始字节环形缓冲
```

### 5.3 帧格式

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| 长度[3:0] | 长度[3:0] | 来源ID[1:0] | 来源ID[1:0] | 类型 | 保留 |  Payload...  |
+--------+--------+--------+--------+--------+--------+--------+--------+
   4字节      4字节       2字节         2字节       1字节  1字节   N字节
```

---

## 6. 函数详细设计

### 6.1 `stdio_write_keycode`

**逻辑流程**：
1. 校验 `shm` 非空，`keycode_buf` 非空，`len > 0`
2. 读取SHM控制块，校验 `magic == 0x5354494F`
3. 计算帧总长度 = 8（帧头）+ `len`
4. 检查环形缓冲区剩余空间，若不足则置位 `overrun_flag` 并返回 `-ENOSPC`
5. 按环形缓冲策略写入帧头（小端序）+ Payload
6. 原子更新 `wr_ptr` 与 `bytes_in_buf`
7. 若 `flags & 0x01`（回显请求），触发回显路径（直接复制到控制台输入段）
8. 若 `flags & 0x02`（立即刷新），发送SHM信号通知消费者Worker
9. 返回实际写入字节数

**返回值**：
- `>0`：成功写入的字节数
- `-EINVAL`：参数非法
- `-ENOSPC`：缓冲区溢出
- `-EBADSHM`：SHM控制块损坏

### 6.2 `stdio_read_frame`

**逻辑流程**：
1. 校验 `shm` 非空，`out_buf` 非空，`buf_cap > 0`
2. 读取SHM控制块，校验 `magic`
3. 检查 `bytes_in_buf`，若为0返回 `-EAGAIN`
4. 从 `rd_ptr` 处读取8字节帧头，解析长度 `N`
5. 若 `buf_cap < N`，返回 `-EMSGSIZE`
6. 按环形缓冲策略读取 `N` 字节 Payload 到 `out_buf`
7. 原子更新 `rd_ptr`，递减 `bytes_in_buf`
8. 若读取到 `0x04`（EOF令牌），设置 `*out_type = 2`
9. 返回0

**返回值**：
- `0`：成功
- `-EAGAIN`：暂无数据
- `-EMSGSIZE`：输出缓冲区不足
- `-EBADSHM`：SHM控制块损坏

### 6.3 `stdio_pipe_init`

**逻辑流程**：
1. 在SHM指定偏移处写入控制块，设置 `magic`、`version`、`buf_size`
2. 清零 `wr_ptr`、`rd_ptr`、`bytes_in_buf`
3. 设置 `echo_enabled = default_echo`
4. 清零环形缓冲区
5. 返回0

---

## 7. 测试场景

### 7.1 单元测试：基本读写

```
测试名：test_stdio_basic_loopback
步骤：
  1. 初始化SHM与stdio管道（buf_size=1024）
  2. 键盘生产者写入 "Hello\n"（6字节文本）
  3. 控制台消费者读取帧
断言：
  - out_len == 6
  - out_type == 0（文本）
  - 内容匹配 "Hello\n"
  - bytes_in_buf == 0（完全消费）
```

### 7.2 压力测试：环形缓冲环绕

```
测试名：test_stdio_ring_wraparound
步骤：
  1. 初始化 buf_size=64 的小缓冲区
  2. 循环写入 16字节帧 × 5次（总计80字节 > 64字节）
  3. 每写一次读一次，迫使指针环绕
断言：
  - 无数据损坏
  - wr_ptr 与 rd_ptr 正确模运算
  - overrun_flag 始终为0
```

### 7.3 异常测试：缓冲区溢出

```
测试名：test_stdio_overrun
步骤：
  1. 初始化 buf_size=32
  2. 连续写入 3帧 × 16字节（不读取）
断言：
  - 第3次写入返回 -ENOSPC
  - overrun_flag == 1
  - 商场错误日志段记录溢出事件
```

### 7.4 集成测试：键盘→stdio→控制台端到端

```
测试名：test_stdio_e2e_keyboard_console
步骤：
  1. 商场main加载BP-0001、BP-0002、BP-0003
  2. 模拟键盘输入 "test123"
  3. 等待控制台消费者渲染
断言：
  - 控制台输出缓冲区包含 "test123"
  - 端到端延迟 < 10ms（本地SHM）
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0001-keyboard, BP-0002-console, BP-0004-shm-manager |
| 后置蓝图 | BP-0006-intent-router（stdio作为默认订阅目标） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0003.json`，包含本蓝图全部函数代码与测试用例 |
| 商场加载 | 由商场 `main()` 在创世第3阶段加载，作为默认通道产品注册到意图路由器 |

---

## 9. 附录：状态机

```
[UNINITIALIZED] --stdio_pipe_init()--> [IDLE]
[IDLE] --stdio_write_keycode()--> [BUFFERING]
[BUFFERING] --stdio_flush()--> [READY_TO_READ]
[READY_TO_READ] --stdio_read_frame()--> [IDLE]（若读空）或 [READY_TO_READ]（若仍有数据）
[Any] --stdio_pipe_reset()--> [IDLE]
[Any] --缓冲区溢出--> [OVERRUN] --stdio_pipe_reset()--> [IDLE]
```

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
