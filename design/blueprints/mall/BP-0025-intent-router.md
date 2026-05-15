# BP-0006-intent-router.md

> **蓝图编号**：BP-0006  
> **产品名称**：intent-router（意图路由与订单匹配）  
> **产品类型**：核心服务产品  
> **优先级**：P1  
> **依赖**：BP-0004-shm-manager, BP-0005-worker-spawn  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`intent-router` 是商场的**中枢神经系统**，负责解析消费者的**意图（Intent）**，匹配生产者的**能力（Capability）**，建立**订单（Order）**并完成数据路由。它实现了一个基于SHM的**发布-订阅市场机制**：

- **消费者**不直接知道生产者存在，只表达"我需要什么数据"
- **生产者**不直接知道消费者存在，只声明"我能提供什么数据"
- **商场**通过 `intent-router` 完成供需匹配，建立SHM数据通道

本产品禁止任何形式的硬编码点对点连接，所有关系必须通过SHM中的**意图表**与**能力表**动态注册。

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 匹配算法 | 基于精确标签匹配 + 优先级排序，禁止模糊匹配导致不确定性 |
| 状态存储 | 意图、能力、订单全部存储于SHM，禁止调度器内部缓存 |
| 并发安全 | 意图表与能力表更新必须持有SHM锁，防止竞争条件 |
| 循环检测 | 禁止生产者-消费者形成循环依赖（A订阅B，B订阅A），检测到即拒绝 |
| 动态性 | 支持运行时注册/注销，支持热插拔生产者与消费者 |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **意图（Intent）** | 消费者提交的需求描述，格式为标签集合 `{key=value, ...}` |
| **能力（Capability）** | 生产者提交的产品描述，格式为标签集合 `{key=value, ...}` |
| **订单（Order）** | 匹配成功后生成的契约，记录生产者ID、消费者ID、数据通道地址、生命周期 |
| **意图表** | SHM中的消费者意图注册数组 |
| **能力表** | SHM中的生产者能力注册数组 |
| **订单表** | SHM中的匹配结果数组 |
| **标签匹配度** | 计算意图标签与能力标签的交集比例，100%为完全匹配 |
| **通道地址** | 订单中指定的SHM子段名与矢量偏移，消费者从此处读取 |

---

## 4. 接口定义

### 4.1 消费者侧：意图注册

```c
// 消费者注册意图（声明需要什么数据）
int intent_register(
    SharedMemory* root,           // SHM根段
    uint16_t consumer_id,         // 消费者Worker ID
    const char* intent_tags,      // 标签字符串（如 "type=text source=keyboard"）
    uint8_t priority,             // 优先级：0=最低, 255=最高
    uint32_t buffer_size,         // 期望的缓冲区大小
    uint16_t* out_intent_id       // 出参：意图ID
);

// 消费者注销意图
int intent_unregister(
    SharedMemory* root,
    uint16_t intent_id,
    uint16_t consumer_id        // 校验：只能注销自己的意图
);

// 消费者查询已匹配的订单
int intent_query_orders(
    SharedMemory* root,
    uint16_t consumer_id,
    uint16_t* out_order_ids,     // 出参：订单ID数组
    uint32_t* out_order_count    // 出参：订单数量
);
```

### 4.2 生产者侧：能力注册

```c
// 生产者注册能力（声明能提供什么数据）
int capability_register(
    SharedMemory* root,
    uint16_t producer_id,         // 生产者Worker ID
    const char* capability_tags,  // 标签字符串（如 "type=text source=keyboard"）
    uint8_t priority,             // 优先级
    const char* output_seg_name,  // 输出数据所在的SHM子段名
    uint64_t output_offset,       // 输出数据偏移
    uint16_t* out_capability_id   // 出参：能力ID
);

// 生产者注销能力
int capability_unregister(
    SharedMemory* root,
    uint16_t capability_id,
    uint16_t producer_id
);

// 生产者查询已建立的订单（谁订阅了我）
int capability_query_subscribers(
    SharedMemory* root,
    uint16_t producer_id,
    uint16_t* out_order_ids,
    uint32_t* out_subscriber_count
);
```

### 4.3 商场侧：匹配与路由

```c
// 执行意图-能力匹配（由商场调度器周期性调用）
int router_match_cycle(
    SharedMemory* root,
    uint32_t* out_new_orders,    // 出参：本周期新建订单数
    uint32_t* out_broken_orders  // 出参：本周期断裂订单数（生产者/消费者死亡）
);

// 强制断开订单（如消费者取消订阅或生产者下线）
int router_break_order(
    SharedMemory* root,
    uint16_t order_id,
    uint8_t reason               // 断开原因：0=消费者注销, 1=生产者注销, 2=商场关闭
);

// 获取路由统计
int router_get_stats(
    SharedMemory* root,
    uint32_t* out_total_intents,
    uint32_t* out_total_capabilities,
    uint32_t* out_active_orders,
    uint32_t* out_match_rate     // 匹配率（百分比）
);
```

---

## 5. SHM数据结构设计

### 5.1 意图表（位于根段 + 0x6000）

```c
struct intent_entry {
    uint16_t intent_id;          // 意图唯一ID
    uint16_t consumer_id;        // 所属消费者
    char     tags[64];           // 标签字符串（如 "type=text source=keyboard"）
    uint8_t  priority;           // 优先级
    uint32_t buffer_size;        // 期望缓冲区大小
    uint8_t  status;             // 0=FREE, 1=PENDING, 2=MATCHED, 3=CANCELLED
    uint16_t matched_order_id;   // 匹配的订单ID（若已匹配）
    uint64_t register_time;      // 注册时间戳
    uint8_t  reserved[6];
};
// 默认支持 128 个意图（128 × 96 = 12KB）
```

### 5.2 能力表（位于根段 + 0x9000）

```c
struct capability_entry {
    uint16_t capability_id;      // 能力唯一ID
    uint16_t producer_id;        // 所属生产者
    char     tags[64];           // 标签字符串
    uint8_t  priority;           // 优先级
    char     output_seg_name[16];// 输出SHM子段名
    uint64_t output_offset;      // 输出偏移
    uint8_t  status;             // 0=FREE, 1=ACTIVE, 2=UNAVAILABLE, 3=REVOKED
    uint16_t subscriber_count;   // 当前订阅者数量
    uint64_t register_time;
    uint8_t  reserved[6];
};
// 默认支持 128 个能力（128 × 112 = 14KB）
```

### 5.3 订单表（位于根段 + 0xD000）

```c
struct order_entry {
    uint16_t order_id;           // 订单唯一ID
    uint16_t intent_id;          // 关联意图ID
    uint16_t capability_id;      // 关联能力ID
    uint16_t consumer_id;        // 消费者ID
    uint16_t producer_id;        // 生产者ID
    char     channel_seg[16];    // 数据通道子段名（可能是新创建的）
    uint64_t channel_offset;     // 通道偏移
    uint32_t channel_size;       // 通道大小
    uint8_t  status;             // 0=FREE, 1=NEGOTIATING, 2=ACTIVE, 3=BROKEN, 4=ARCHIVED
    uint64_t create_time;
    uint64_t last_activity;      // 最后数据活动时间戳
    uint8_t  reserved[8];
};
// 默认支持 256 个订单（256 × 64 = 16KB）
```

---

## 6. 函数详细设计

### 6.1 `intent_register`

**逻辑流程**：
1. 校验 `root` 非空，`intent_tags` 非空，`consumer_id` 有效
2. 在意图表中查找第一个 `status == FREE` 的槽位
3. 若已满，返回 `-ENOSPC`
4. 校验标签字符串长度 <= 63，格式为 `key=value` 对，空格分隔
5. 写入意图项：`consumer_id`、`tags`、`priority`、`buffer_size`
6. 设置 `status = PENDING`，`register_time = now`
7. 返回 `intent_id`

**返回值**：
- `0`：成功
- `-ENOSPC`：意图表已满
- `-EINVAL`：标签格式非法

### 6.2 `capability_register`

**逻辑流程**：
1. 校验 `root` 非空，`capability_tags` 非空，`producer_id` 有效
2. 在能力表中查找第一个 `status == FREE` 的槽位
3. 校验 `output_seg_name` 存在于SHM目录表
4. 写入能力项：`producer_id`、`tags`、`priority`、`output_seg_name`、`output_offset`
5. 设置 `status = ACTIVE`
6. 返回 `capability_id`

**返回值**：
- `0`：成功
- `-ENOSPC`：能力表已满
- `-ENOENT`：输出段不存在

### 6.3 `router_match_cycle`

**逻辑流程**：
1. 获取意图表锁与能力表锁
2. 遍历所有 `status == PENDING` 的意图
3. 对每个意图，遍历所有 `status == ACTIVE` 的能力：
   a. 解析意图标签与能力标签为键值对
   b. 计算匹配度：交集标签数 / 意图标签总数
   c. 若匹配度 == 100%，记录为候选
4. 对每个意图，从候选中选择 `priority` 最高的能力（若优先级相同，选先注册者）
5. 在订单表中创建新订单：
   - `intent_id`、`capability_id`、`consumer_id`、`producer_id`
   - 若 `buffer_size > 0`，创建新通道段；否则复用 `output_seg_name`
   - `status = ACTIVE`
6. 更新意图 `status = MATCHED`，能力 `subscriber_count++`
7. 检查已存在订单：若关联的Worker已死亡（通过Worker表），置 `status = BROKEN`
8. 释放锁，返回新建订单数与断裂订单数

**返回值**：
- `0`：匹配周期完成
- 出参包含统计信息

### 6.4 `router_break_order`

**逻辑流程**：
1. 校验 `order_id` 有效，`status == ACTIVE`
2. 设置订单 `status = BROKEN`
3. 更新关联意图 `status = PENDING`（允许重新匹配）
4. 递减关联能力 `subscriber_count`
5. 若 `subscriber_count == 0`，能力保持 `ACTIVE`（可再次被匹配）
6. 记录断开原因到订单项
7. 通知错误守卫（BP-0007）记录事件

---

## 7. 测试场景

### 7.1 单元测试：精确匹配

```
测试名：test_router_exact_match
步骤：
  1. 初始化SHM与路由表
  2. 生产者注册能力：tags="type=text source=keyboard"
  3. 消费者注册意图：tags="type=text source=keyboard"
  4. 调用 router_match_cycle
断言：
  - out_new_orders == 1
  - 订单 status == ACTIVE
  - 意图 status == MATCHED
  - 能力 subscriber_count == 1
```

### 7.2 单元测试：不匹配

```
测试名：test_router_no_match
步骤：
  1. 生产者注册能力：tags="type=audio source=mic"
  2. 消费者注册意图：tags="type=text source=keyboard"
  3. 调用 router_match_cycle
断言：
  - out_new_orders == 0
  - 意图 status 保持 PENDING
```

### 7.3 压力测试：大规模匹配

```
测试名：test_router_mass_match
步骤：
  1. 注册 64 个生产者，每个提供不同能力
  2. 注册 64 个消费者，每个需求随机匹配其中一个能力
  3. 调用 router_match_cycle
断言：
  - out_new_orders == 64
  - 无重复匹配（一个能力不被多个消费者独占，除非设计允许多播）
  - 匹配耗时 < 100ms（全内存操作）
```

### 7.4 异常测试：循环依赖检测

```
测试名：test_router_cycle_detection
步骤：
  1. 生产者A注册能力：tags="type=X"
  2. 消费者B注册意图：tags="type=X"（匹配A）
  3. 生产者B注册能力：tags="type=Y"
  4. 消费者A注册意图：tags="type=Y"（匹配B）
  5. 若商场支持双向数据流，需检测循环
断言：
  - 第二次匹配返回 -ECYCLE（或商场策略允许但标记风险）
  - 错误日志记录循环依赖警告
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager, BP-0005-worker-spawn |
| 后置蓝图 | BP-0003-stdio-pipe（stdio作为默认通道产品注册）、BP-0009-mall-monitor（统计采集） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0006.json`，含匹配算法与循环检测规则 |
| 商场加载 | 由商场 `main()` 在创世第4阶段加载，Worker调度器就绪后启动 |

---

## 9. 附录：匹配算法伪代码

```
function router_match_cycle(root):
    new_orders = 0
    broken_orders = 0

    for intent in intent_table where status == PENDING:
        best_capability = NULL
        best_priority = 0

        for cap in capability_table where status == ACTIVE:
            if is_cyclic(intent.consumer_id, cap.producer_id):
                continue
            match_score = calculate_match(intent.tags, cap.tags)
            if match_score == 1.0 and cap.priority > best_priority:
                best_capability = cap
                best_priority = cap.priority

        if best_capability != NULL:
            order = create_order(intent, best_capability)
            if order != NULL:
                intent.status = MATCHED
                best_capability.subscriber_count++
                new_orders++

    for order in order_table where status == ACTIVE:
        if worker_is_dead(order.consumer_id) or worker_is_dead(order.producer_id):
            order.status = BROKEN
            broken_orders++
            router_break_order(root, order.order_id, reason=WORKER_DEATH)

    return (new_orders, broken_orders)
```

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
