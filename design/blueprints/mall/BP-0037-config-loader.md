# BP-0037-config-loader.md

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**：BP-0037  
> **产品名称**：config-loader（配置加载与热更新）  
> **产品类型**：基础设施产品  
> **优先级**：P1  
> **依赖**：BP-0023-shm-manager  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`config-loader` 是商场的**配置中枢**，负责将外部配置文件（JSON/YAML/TOML）解析并映射到SHM配置区，实现**配置即内存**——所有运行时参数通过SHM访问，支持热更新而不重启商场。它提供两层抽象：

1. **启动配置**：商场 `main()` 启动时加载的静态参数（如SHM大小、Worker上限、AICoder路径）
2. **动态配置**：运行时可修改的参数（如日志级别、采样频率、告警阈值），修改后即时生效

配置数据一旦进入SHM，所有产品通过统一的 `config_get` / `config_set` 接口访问，禁止直接读取文件系统。

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 配置源 | 仅支持JSON格式配置文件（简化解析器），禁止YAML/TOML以降低复杂度 |
| 存储位置 | 所有配置项存储于SHM根段配置区，禁止本地缓存 |
| 类型系统 | 支持 `int64`、`double`、`bool`、`string`（最大64字节）四种类型 |
| 热更新 | 通过文件系统监视检测配置变更，原子更新SHM配置区 |
| 版本控制 | 每次更新递增配置版本号，消费者可检测版本变化并重新加载 |
| 权限控制 | 启动配置只读（RO），动态配置可写（RW），需校验调用者ID |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **配置项（Config Item）** | 一个键值对，包含名称、类型、值、访问权限、最后修改时间 |
| **配置区** | SHM根段中的固定区域，存储所有配置项的数组 |
| **配置文件** | 外部JSON文件，商场启动时读取，运行时监视变更 |
| **配置版本** | 64位无符号整数，每次任何配置项修改时原子递增 |
| **热更新句柄** | 文件系统监视器返回的变更事件标识 |
| **默认值表** | 编译时嵌入的默认配置，当配置文件缺失项时回退 |
| **配置锁** | 保护配置区写操作的SHM自旋锁 |

---

## 4. 接口定义

### 4.1 配置加载与初始化

```c
// 从文件系统加载配置到SHM（由商场main调用）
int config_load_from_file(
    SharedMemory* root,
    const char* config_path,      // 配置文件路径（如 "mall-config.json"）
    uint8_t* out_items_loaded     // 出参：成功加载的配置项数
);

// 使用默认值初始化配置区（当配置文件不存在时）
int config_load_defaults(
    SharedMemory* root,
    uint8_t* out_items_loaded
);

// 初始化配置区数据结构（创建配置段与默认值表）
int config_init(
    SharedMemory* root,
    uint32_t max_items            // 最大配置项数量
);
```

### 4.2 配置读写

```c
// 读取配置项（任何Worker或商场服务调用）
int config_get_int(
    SharedMemory* root,
    const char* key,
    int64_t* out_value
);
int config_get_double(
    SharedMemory* root,
    const char* key,
    double* out_value
);
int config_get_bool(
    SharedMemory* root,
    const char* key,
    uint8_t* out_value            // 0=false, 1=true
);
int config_get_string(
    SharedMemory* root,
    const char* key,
    char* out_buffer,
    uint32_t buf_cap
);

// 写入动态配置项（仅商场主进程或授权服务调用）
int config_set_int(
    SharedMemory* root,
    const char* key,
    int64_t value,
    uint16_t caller_id
);
// config_set_double / config_set_bool / config_set_string 同理
```

### 4.3 热更新管理

```c
// 启动配置文件监视（由商场调度器调用）
int config_watch_start(
    SharedMemory* root,
    const char* config_path,
    uint32_t poll_interval_ms     // 轮询间隔（毫秒）
);

// 检查配置变更并执行热更新（由调度器周期性调用）
int config_poll_hotupdate(
    SharedMemory* root,
    uint8_t* out_changed,         // 0=无变更, 1=有变更
    uint32_t* out_changed_items   // 变更项数
);

// 获取当前配置版本号
int config_get_version(
    SharedMemory* root,
    uint64_t* out_version
);
```

---

## 5. SHM数据结构设计

### 5.1 配置区（位于根段 + 0x18000）

```c
// 配置区头（64字节）
struct config_header {
    uint64_t magic;              // "CONFIG" = 0x434F4E4649470000
    uint32_t version;
    uint32_t item_count;         // 当前配置项数
    uint32_t item_capacity;      // 最大配置项数
    uint64_t config_version;     // 全局配置版本号（原子递增）
    uint64_t last_update_time;   // 最后更新时间戳
    uint32_t default_count;      // 默认值表项数
    uint8_t  reserved[8];
};

// 配置项（128字节每项）
struct config_item {
    char     key[32];            // 配置键名
    uint8_t  type;               // 0=INT, 1=DOUBLE, 2=BOOL, 3=STRING
    uint8_t  access;             // 0=RO（启动配置）, 1=RW（动态配置）
    uint16_t owner_id;           // 最后修改者ID（RO项为0）
    uint64_t modify_time;        // 最后修改时间戳
    union {
        int64_t  i_value;
        double   d_value;
        uint8_t  b_value;
        char     s_value[64];
    } value;
    uint8_t  reserved[16];
};
```

### 5.2 默认值表（位于根段 + 0x1C000）

```c
struct default_item {
    char     key[32];
    uint8_t  type;
    uint8_t  access;
    union {
        int64_t  i_value;
        double   d_value;
        uint8_t  b_value;
        char     s_value[64];
    } value;
};
// 默认支持 64 个默认值项
```

---

## 6. 函数详细设计

### 6.1 `config_load_from_file`

**逻辑流程**：
1. 校验 `root` 非空，`config_path` 存在且可读
2. 打开JSON文件，读取全部内容到临时缓冲区
3. 解析JSON为键值对（使用极简递归下降解析器，禁止外部库依赖）
4. 获取 `config_lock`
5. 对每个键值对：
   a. 在配置区查找同名项，若不存在查找默认值表
   b. 校验类型一致性（JSON类型与目标类型匹配）
   c. 写入配置项：`key`、`type`、`value`、`access = RO`（启动配置默认为RO）
   d. 设置 `owner_id = 0`（主进程）
6. 释放 `config_lock`
7. 原子递增 `config_version`
8. 返回加载项数

**返回值**：
- `0`：成功
- `-EIO`：文件读取失败
- `-EINVAL`：JSON格式错误或类型不匹配
- `-ENOSPC`：配置区已满

### 6.2 `config_get_int`

**逻辑流程**：
1. 校验 `root` 非空，`key` 非空
2. 在配置区线性搜索 `key`
3. 若找到，校验 `type == INT`，读取 `i_value` 到 `out_value`
4. 若未找到，在默认值表搜索
5. 若仍未找到，返回 `-ENOENT`
6. 返回0

**返回值**：
- `0`：成功
- `-ENOENT`：配置项不存在
- `-EINVAL`：类型不匹配（如请求INT但配置为STRING）

### 6.3 `config_set_int`

**逻辑流程**：
1. 校验 `root` 非空，`key` 非空
2. 在配置区搜索 `key`
3. 校验 `access == RW`，若 `access == RO` 返回 `-EACCES`
4. 校验 `caller_id` 有权限修改（商场主进程或授权服务）
5. 获取 `config_lock`
6. 写入新值，更新 `modify_time`、`owner_id = caller_id`
7. 释放锁，递增 `config_version`
8. 返回0

**返回值**：
- `0`：成功
- `-ENOENT`：配置项不存在
- `-EACCES`：配置项只读或调用者无权限

### 6.4 `config_poll_hotupdate`

**逻辑流程**：
1. 获取配置文件的最后修改时间（`stat` 系统调用）
2. 与SHM中存储的上次修改时间比较
3. 若一致，返回 `out_changed = 0`
4. 若不一致：
   a. 调用 `config_load_from_file` 重新加载
   b. 对比新旧值，统计变更项数
   c. 更新 `last_update_time`
   d. 返回 `out_changed = 1`，`out_changed_items = N`
5. 若文件被删除，回退到默认值表

**返回值**：
- `0`：轮询完成
- 出参指示是否有变更

---

## 7. 测试场景

### 7.1 单元测试：配置加载与读取

```
测试名：test_config_load_and_get
步骤：
  1. 创建测试配置文件 mall-test.json：
     {"shm_size": 65536, "max_workers": 16, "debug_mode": false}
  2. 调用 config_init 与 config_load_from_file
  3. 读取 shm_size、max_workers、debug_mode
断言：
  - config_get_int("shm_size") == 65536
  - config_get_int("max_workers") == 16
  - config_get_bool("debug_mode") == 0
  - config_version == 1
```

### 7.2 单元测试：动态配置热更新

```
测试名：test_config_hotupdate
步骤：
  1. 加载初始配置 {"sample_interval_ms": 100}
  2. 将配置文件修改为 {"sample_interval_ms": 200}
  3. 调用 config_poll_hotupdate
  4. 读取 sample_interval_ms
断言：
  - out_changed == 1
  - out_changed_items == 1
  - config_get_int("sample_interval_ms") == 200
  - config_version == 2
```

### 7.3 异常测试：只读配置保护

```
测试名：test_config_readonly_protection
步骤：
  1. 加载配置 {"mall_name": "genesis-01"}（启动配置，默认RO）
  2. 以 caller_id=0x0001（非主进程）尝试修改 mall_name
断言：
  - config_set_string 返回 -EACCES
  - 错误日志记录越权修改事件
  - 原值保持不变
```

### 7.4 集成测试：配置驱动商场行为

```
测试名：test_config_drive_behavior
步骤：
  1. 商场main加载BP-0010，读取 max_workers=4
  2. 尝试创建5个Worker
断言：
  - 前4个Worker创建成功
  - 第5个返回 -ENOSPC（受配置限制）
  - 修改 max_workers=8 后热更新
  - 第5个Worker可成功创建
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | BP-0004-shm-manager（配置区需SHM支持） |
| 后置蓝图 | 所有其他BP（均通过config-loader读取参数） |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0010.json`，含JSON解析器与默认值表 |
| 商场加载 | 由商场 `main()` 在创世第0阶段最先加载（甚至在SHM管理器之前，用于读取SHM大小参数），但依赖SHM管理器完成物理分配 |

---

## 9. 附录：默认配置清单

| 键名 | 类型 | 默认值 | 访问权限 | 说明 |
|------|------|--------|----------|------|
| `mall_name` | STRING | "mall-genesis" | RO | 商场唯一标识 |
| `shm_root_size` | INT | 65536 | RO | SHM根段大小（字节） |
| `max_workers` | INT | 256 | RO | 最大Worker数量 |
| `max_segments` | INT | 128 | RO | 最大SHM子段数 |
| `grace_period_ms` | INT | 5000 | RW | Worker优雅退出等待时间 |
| `heartbeat_timeout_ms` | INT | 3000 | RW | Worker心跳超时阈值 |
| `sample_interval_ms` | INT | 100 | RW | 监控采样间隔 |
| `alert_queue_size` | INT | 64 | RW | 告警队列长度 |
| `aicoder_workspace` | STRING | "./aicoder" | RW | AICoder工作目录 |
| `debug_mode` | BOOL | false | RW | 调试模式开关 |
| `log_level` | INT | 1 | RW | 日志等级：0=DEBUG, 1=INFO, 2=WARNING, 3=ERROR |

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
