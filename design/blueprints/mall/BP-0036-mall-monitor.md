# BP-0036-mall-monitor.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**：BP-0036  
> **产品名称**：mall-monitor（商场运行监控）  
> **产品类型**：服务产品  
> **优先级**：P2  
> **依赖**：BP-0023-shm-manager, BP-0024-worker-spawn, BP-0025-intent-router, BP-0026-error-guard  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`mall-monitor` 是商场的**可观测性基础设施**，将商场内部所有运行时状态（SHM健康、Worker存活、订单流量、错误统计）转化为**结构化数据产品**，供外部监控消费者订阅。它本身不直接操作显示器或网络，而是通过SHM暴露监控数据，由专门的监控消费者（如控制台监控面板、网络遥测Exporter）读取并渲染。

核心设计原则：
- **监控即产品**：监控数据与普通业务数据同等对待，通过意图路由器发布
- **零开销采样**：监控采集通过SHM原子读完成，不引入额外锁竞争
- **分级暴露**：基础指标（心跳、状态）高频更新；详细指标（调用栈、内存映射）按需采样
- **自监控**：监控服务自身状态也被纳入监控范围（递归可观测）

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 数据来源 | 仅读取SHM中的已有数据结构（Worker表、订单表、错误日志），禁止直接访问/proc或系统调用 |
| 更新频率 | 基础指标每100ms采样一次，详细指标每1s采样一次 |
| 存储位置 | 监控数据写入专用SHM监控段，禁止本地文件缓存 |
| 告警机制 | 通过SHM告警队列发布告警事件，由消费者订阅处理 |
| 性能约束 | 单次采样周期CPU耗时 < 1ms（避免影响业务Worker） |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **指标（Metric）** | 一个可量化的运行时数值，如 `worker_alive_count` |
| **指标段** | SHM子段，存储所有指标的名称-值对 |
| **采样周期** | 两次连续采样的时间间隔，基础指标100ms，详细指标1000ms |
| **告警队列** | SHM中的环形队列，存储触发阈值的异常事件 |
| **仪表盘快照** | 某一时刻所有指标的完整视图，固定大小便于消费者原子读取 |
| **吞吐量计数器** | 统计SHM字节读写速率的64位原子计数器 |
| **延迟直方图** | 预分桶的延迟分布统计（如订单匹配耗时） |

---

## 4. 接口定义

### 4.1 监控采集

```c
// 初始化监控系统（创建指标段与告警队列）
int monitor_init(
    SharedMemory* root,
    uint32_t metric_capacity,     // 最大指标数量
    uint32_t alert_capacity       // 最大告警队列长度
);

// 执行一次完整采样周期（由商场调度器周期性调用）
int monitor_sample_cycle(
    SharedMemory* root,
    uint64_t timestamp_ms         // 当前时间戳
);

// 获取指定指标当前值
int monitor_get_metric(
    SharedMemory* root,
    const char* metric_name,
    int64_t* out_value,
    uint8_t* out_type            // 0=GAUGE, 1=COUNTER, 2=HISTOGRAM_BUCKET
);
```

### 4.2 告警管理

```c
// 注册告警阈值规则
int monitor_alert_register(
    SharedMemory* root,
    const char* metric_name,
    uint8_t condition,            // 0=GT, 1=LT, 2=EQ, 3=CHANGE_RATE
    int64_t threshold,
    uint8_t severity,             // 0=INFO, 1=WARNING, 2=ERROR, 3=CRITICAL
    const char* alert_message
);

// 查询当前活跃告警
int monitor_alert_query(
    SharedMemory* root,
    uint8_t min_severity,
    uint32_t max_alerts,
    uint8_t* out_buffer,
    uint32_t* out_alert_count
);

// 清除已处理的告警（由监控消费者调用）
int monitor_alert_ack(
    SharedMemory* root,
    uint32_t alert_id,
    uint16_t consumer_id
);
```

### 4.3 仪表盘快照

```c
// 生成当前仪表盘快照（原子复制所有指标到快照区）
int monitor_snapshot(
    SharedMemory* root,
    uint64_t* out_snapshot_time   // 出参：快照时间戳
);

// 消费者读取快照（非阻塞，原子读）
int monitor_snapshot_read(
    SharedMemory* root,
    uint8_t* out_buffer,           // 预分配缓冲区（大小由 monitor_init 决定）
    uint64_t* out_snapshot_time
);
```

---

## 5. SHM数据结构设计

### 5.1 指标段（专用子段：{mall_name}-metrics）

```c
// 段头（64字节）
struct metrics_header {
    uint64_t magic;              // "METRICS" = 0x4D45545249435300
    uint32_t version;
    uint32_t metric_count;       // 当前指标数
    uint32_t metric_capacity;    // 最大指标数
    uint64_t last_sample_time;   // 上次采样时间戳
    uint32_t sample_seq;         // 采样序号（递增）
    uint8_t  reserved[8];
};

// 指标项（32字节每项）
struct metric_entry {
    char     name[16];           // 指标名（如 "worker_alive"）
    int64_t  value;              // 当前值
    uint8_t  type;               // 0=GAUGE, 1=COUNTER, 2=HISTOGRAM
    uint8_t  reserved[7];
};
```

### 5.2 告警队列（专用子段：{mall_name}-alerts）

```c
// 告警记录（64字节每项）
struct alert_record {
    uint32_t alert_id;           // 告警唯一ID
    uint64_t timestamp;
    char     metric_name[16];    // 触发指标名
    int64_t  trigger_value;      // 触发时的值
    int64_t  threshold;          // 阈值
    uint8_t  condition;          // 条件类型
    uint8_t  severity;
    uint8_t  acked;              // 0=未确认, 1=已确认
    uint16_t acked_by;           // 确认者ID
    char     message[32];        // 告警消息
};
```

### 5.3 仪表盘快照区（专用子段：{mall_name}-snapshot）

```c
struct snapshot_block {
    uint64_t timestamp;          // 快照时间
    uint32_t metric_count;       // 快照时指标数
    uint8_t  valid;              // 0=无效（采样中）, 1=有效
    uint8_t  reserved[3];
    // 后跟 metric_count 个 metric_entry 的副本
};
```

---

## 6. 函数详细设计

### 6.1 `monitor_sample_cycle`

**逻辑流程**：
1. 校验 `root` 非空
2. 读取当前时间戳，计算与上次采样的间隔
3. **Worker指标**：遍历Worker表，统计：
   - `worker_total`：总槽位数
   - `worker_alive`：status == RUNNING 的数量
   - `worker_spawning`：status == SPAWNINGS 的数量
   - `worker_dead`：status == DEAD 的数量
4. **SHM指标**：调用 `shm_health_check` 获取：
   - `shm_total_segments`
   - `shm_used_segments`
   - `shm_corrupted_segments`
5. **订单指标**：遍历订单表，统计：
   - `orders_active`：status == ACTIVE
   - `orders_broken`：status == BROKEN
   - `orders_total_matched`：历史总匹配数（从计数器读取）
6. **错误指标**：读取错误日志段头：
   - `errors_fatal_1h`：最近1小时致命错误数（需时间过滤）
   - `errors_total`：历史总数
7. **性能指标**：
   - 计算本周期SHM字节读写增量，更新 `shm_throughput_bps`
   - 记录 `monitor_sample_duration_us`（本次采样自身耗时）
8. 原子写入指标段
9. 检查告警阈值，触发者写入告警队列
10. 更新 `last_sample_time` 与 `sample_seq`

**返回值**：
- `0`：采样完成
- `-EAGAIN`：距离上次采样过近（< 50ms）

### 6.2 `monitor_alert_register`

**逻辑流程**：
1. 校验 `metric_name` 存在于指标段或预定义列表
2. 校验 `condition` 在 [0,3] 范围
3. 在SHM配置区追加告警规则（若规则表已满，覆盖最旧规则）
4. 返回规则ID

**返回值**：
- `0`：成功
- `-ENOENT`：指标名不存在

### 6.3 `monitor_snapshot`

**逻辑流程**：
1. 设置快照区 `valid = 0`（标记采样中）
2. 原子复制指标段所有 `metric_entry` 到快照区
3. 写入 `timestamp` 与 `metric_count`
4. 设置 `valid = 1`
5. 返回时间戳

**设计要点**：消费者读取时先检查 `valid`，若为1则原子读取整个快照区；若为0则等待下次采样。

---

## 7. 测试场景

### 7.1 单元测试：指标采集

```
测试名：test_monitor_metrics_collection
步骤：
  1. 初始化SHM与监控系统
  2. 创建3个Worker（2个RUNNING，1个DEAD）
  3. 创建2个活跃订单
  4. 调用 monitor_sample_cycle
  5. 查询指标 worker_alive、orders_active
断言：
  - worker_alive == 2
  - orders_active == 2
  - monitor_sample_duration_us < 1000
```

### 7.2 单元测试：告警触发

```
测试名：test_monitor_alert_trigger
步骤：
  1. 注册告警规则：worker_alive < 5 时触发WARNING
  2. 初始化5个Worker
  3. 调用 monitor_sample_cycle（不应触发）
  4. 终止3个Worker
  5. 再次采样
断言：
  - 第5步后告警队列中出现1条记录
  - severity == WARNING
  - metric_name == "worker_alive"
  - trigger_value == 2
```

### 7.3 压力测试：高频采样

```
测试名：test_monitor_high_frequency
步骤：
  1. 初始化监控（metric_capacity=256）
  2. 以10ms间隔连续调用100次 monitor_sample_cycle
断言：
  - 前几次可能返回 -EAGAIN（保护机制）
  - 无SHM损坏
  - 商场主进程CPU占用增加 < 5%
```

### 7.4 集成测试：监控消费者端到端

```
测试名：test_monitor_e2e_dashboard
步骤：
  1. 商场main加载BP-0004至BP-0009
  2. 启动监控消费者Worker（订阅 mall-monitor 产品）
  3. 监控消费者定期读取快照
  4. 创建/销毁业务Worker，观察监控数据变化
断言：
  - 监控消费者实时看到 worker_alive 变化
  - 快照读取始终原子一致（无半写状态）
  - 告警事件被正确投递到消费者
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager, BP-0005-worker-spawn, BP-0006-intent-router, BP-0007-error-guard |
| 后置蓝图 | 专门的监控面板消费者（需单独设计，可视为BP-0011） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0009.json`，含指标定义与告警规则模板 |
| 商场加载 | 由商场 `main()` 在创世第6阶段加载，所有核心服务就绪后启动 |

---

## 9. 附录：预定义指标清单

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `worker_total` | GAUGE | Worker表总槽位数 |
| `worker_alive` | GAUGE | 当前存活Worker数 |
| `worker_spawning` | GAUGE | 正在启动的Worker数 |
| `worker_dead` | GAUGE | 已死亡未收割的Worker数 |
| `shm_total_segments` | GAUGE | SHM总段数 |
| `shm_used_segments` | GAUGE | 已使用段数 |
| `shm_corrupted_segments` | GAUGE | 损坏段数 |
| `orders_active` | GAUGE | 活跃订单数 |
| `orders_broken` | GAUGE | 断裂订单数 |
| `orders_match_rate` | GAUGE | 意图匹配率（百分比） |
| `errors_fatal_1h` | GAUGE | 最近1小时致命错误数 |
| `errors_total` | COUNTER | 历史错误总数 |
| `shm_throughput_bps` | GAUGE | SHM字节吞吐量（字节/秒） |
| `monitor_sample_duration_us` | GAUGE | 单次采样耗时（微秒） |
| `aicoder_tasks_pending` | GAUGE | AICoder待处理任务数 |
| `aicoder_alive` | GAUGE | AICoder存活状态（0/1） |

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
