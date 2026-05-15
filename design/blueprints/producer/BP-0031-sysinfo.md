# 蓝图文档（定稿版）：BP-0031 -- 计算机系统信息采集生产者

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | `BP-0031-SYSINFO` |
| 蓝图名称 | 计算机系统信息采集生产者 |
| 生产者 | `Genesis-Worker-001` |
| 提交时间 | `2026-05-11` |
| 版本 | `1.0.0` |
| 目标语言 | Python |
| 复杂度等级 | `Level-0`（创世纪验证级） |
| 设计范式 | `纯函数级，禁止OOP封装` |
| 关联消费者 | `BP-0032-SYSDISPLAY` |

---

## 2. 功能概述

本蓝图定义一个**计算机系统信息采集生产者**。它周期性采集当前运行环境的系统指标（CPU使用率、内存占用、磁盘IO、网络流量、进程列表、系统负载），将采集结果封装为标准SHM数据包写入商场共享内存。

### 2.1 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 系统指标采集、数据格式化、SHM写入、采集周期控制 |
| **不负责** | 指标阈值告警、历史数据存储、跨节点聚合、可视化渲染 |
| **输入** | 操作系统提供的系统调用接口（/proc、psutil、WMIC等） |
| **输出** | SHM矢量 `/shm/system/info` |

### 2.2 采集指标清单

| 指标类别 | 具体指标 | 单位 | 采集方式 |
|----------|----------|------|----------|
| CPU | 使用率（总体+每核） | % | psutil.cpu_percent / cpu_times |
| CPU | 系统负载（1min/5min/15min） | - | os.getloadavg (Linux) |
| 内存 | 物理内存总量/已用/可用 | MB | psutil.virtual_memory |
| 内存 | 交换分区总量/已用 | MB | psutil.swap_memory |
| 磁盘 | 各分区总量/已用/使用率 | GB, % | psutil.disk_partitions / disk_usage |
| 磁盘 | IO读写速率 | MB/s | psutil.disk_io_counters |
| 网络 | 各网卡发送/接收字节数 | bytes | psutil.net_io_counters |
| 网络 | 各网卡发送/接收速率 | bytes/s | 两次采样差值计算 |
| 进程 | 进程总数 | count | len(psutil.pids()) |
| 进程 | Top-N CPU/内存进程 | - | psutil.process_iter排序 |
| 系统 | 系统启动时间 | timestamp | psutil.boot_time |
| 系统 | 当前时间戳 | ns | time.time_ns |

---

## 3. 函数设计（函数级，无类）

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `sysinfo_init(ctx)` | 初始化采集器，注册SHM写入权限 | 商场main()调用一次 |
| `sysinfo_loop(ctx)` | 主采集循环，按周期采集并写入SHM | 独立Worker进程持续运行 |
| `sysinfo_collect_cpu(ctx)` | 采集CPU指标 | `sysinfo_loop()`内调用 |
| `sysinfo_collect_memory(ctx)` | 采集内存指标 | `sysinfo_loop()`内调用 |
| `sysinfo_collect_disk(ctx)` | 采集磁盘指标 | `sysinfo_loop()`内调用 |
| `sysinfo_collect_network(ctx)` | 采集网络指标 | `sysinfo_loop()`内调用 |
| `sysinfo_collect_process(ctx)` | 采集进程指标 | `sysinfo_loop()`内调用 |
| `sysinfo_collect_system(ctx)` | 采集系统级指标 | `sysinfo_loop()`内调用 |
| `sysinfo_pack(ctx, data)` | 将采集数据打包为SHM标准格式 | 采集完成后调用 |
| `sysinfo_cleanup(ctx)` | 释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `sysinfo_init(ctx) -> dict`

```
参数:
  ctx : dict - 商场上下文，包含：
    ctx["shm_vector"]     : string  - 目标SHM矢量路径，默认 "/shm/system/info"
    ctx["collect_interval"]: int    - 采集周期（秒），默认 2
    ctx["top_n"]           : int    - Top-N进程数，默认 5
    ctx["network_interfaces"]: list - 采集的网卡列表，默认 None（全部）

返回:
  {"status": "ok", "state": {...}}   : 成功
  {"status": "error", "error": "..."} : 失败

行为:
  1. 验证psutil模块可用性
  2. 记录初始网络IO计数器（用于后续速率计算）
  3. 记录初始磁盘IO计数器
  4. 返回包含初始状态的ctx副本
```

#### `sysinfo_loop(ctx) -> dict`

```
参数:
  ctx : dict - 包含采集器运行状态

返回:
  每次迭代返回 {"status": "ok", "timestamp_ns": ..., "metrics": {...}}

行为:
  while ctx["running"]:
      data = {}
      data["cpu"]      = sysinfo_collect_cpu(ctx)
      data["memory"]   = sysinfo_collect_memory(ctx)
      data["disk"]     = sysinfo_collect_disk(ctx)
      data["network"]  = sysinfo_collect_network(ctx)
      data["process"]  = sysinfo_collect_process(ctx)
      data["system"]   = sysinfo_collect_system(ctx)
      packed = sysinfo_pack(ctx, data)
      mall_shm_write(ctx["shm_handle"], packed)
      sleep(ctx["collect_interval"])
```

#### `sysinfo_collect_cpu(ctx) -> dict`

```
返回:
  {
    "percent": float,          // 总体CPU使用率 (0-100)
    "per_core": [float, ...],  // 每核使用率
    "count_logical": int,      // 逻辑核心数
    "count_physical": int,     // 物理核心数
    "freq_current_mhz": float  // 当前频率
  }

约束:
  - 首次调用cpu_percent需interval=None预热，后续使用interval=0
  - per_core列表长度必须等于count_logical
```

#### `sysinfo_collect_memory(ctx) -> dict`

```
返回:
  {
    "total_mb": float,
    "used_mb": float,
    "available_mb": float,
    "percent": float,
    "swap_total_mb": float,
    "swap_used_mb": float,
    "swap_percent": float
  }
```

#### `sysinfo_collect_disk(ctx) -> dict`

```
返回:
  {
    "partitions": [
      {
        "mountpoint": string,
        "device": string,
        "fstype": string,
        "total_gb": float,
        "used_gb": float,
        "percent": float
      }, ...
    ],
    "io_read_mb": float,       // 累计读取MB
    "io_write_mb": float,      // 累计写入MB
    "io_read_bytes_per_sec": float,
    "io_write_bytes_per_sec": float
  }

约束:
  - 排除虚拟文件系统（tmpfs, proc, sys, devtmpfs）
```

#### `sysinfo_collect_network(ctx) -> dict`

```
返回:
  {
    "interfaces": {
      "eth0": {
        "bytes_sent": int,
        "bytes_recv": int,
        "packets_sent": int,
        "packets_recv": int,
        "send_rate_per_sec": float,
        "recv_rate_per_sec": float
      }, ...
    },
    "total_sent_rate": float,
    "total_recv_rate": float
  }

行为:
  - 通过与上次采样值的差值/时间差计算速率
  - 首次采样速率为0.0
  - ctx中存储上次采样值和采样时间戳
```

#### `sysinfo_collect_process(ctx) -> dict`

```
返回:
  {
    "total_count": int,
    "top_cpu": [
      {"pid": int, "name": string, "cpu_percent": float, "memory_mb": float}, ...
    ],
    "top_memory": [
      {"pid": int, "name": string, "cpu_percent": float, "memory_mb": float}, ...
    ]
  }

约束:
  - top_cpu和top_memory各返回ctx["top_n"]条
  - 排除已终止进程（try/except保护）
```

#### `sysinfo_collect_system(ctx) -> dict`

```
返回:
  {
    "boot_time": string,       // ISO格式
    "uptime_seconds": float,
    "current_time": string,    // ISO格式
    "timestamp_ns": int
  }
```

#### `sysinfo_pack(ctx, data) -> dict`

```
参数:
  data : dict - 所有采集子系统的合并数据

返回:
  {
    "source": "sysinfo_producer",
    "version": "1.0.0",
    "timestamp_ns": int,
    "collect_interval": int,
    "payload": data
  }
```

#### `sysinfo_cleanup(ctx) -> dict`

```
返回:
  {"status": "ok"}

行为:
  - 清空ctx中的采样缓存
  - 返回成功
```

---

## 4. 行为契约

### 4.1 前置条件

| 条件ID | 条件描述 |
|--------|----------|
| PRE-1 | psutil模块已安装且可导入 |
| PRE-2 | 商场SHM系统已初始化 |
| PRE-3 | 采集周期 >= 1秒（避免高频采集导致系统负载） |

### 4.2 后置条件

| 条件ID | 条件描述 |
|--------|----------|
| POST-1 | 每个采集周期产出一个完整的系统信息数据包 |
| POST-2 | 数据包写入SHM后，消费者可立即读取 |
| POST-3 | 采集异常不影响后续周期的采集（容错继续） |

### 4.3 不变量

| 不变量ID | 不变量描述 |
|----------|-----------|
| INV-1 | 返回值始终为dict结构 |
| INV-2 | 所有浮点指标值域有界（CPU: 0-100, 内存: 0-100, 磁盘: 0-100） |
| INV-3 | 不修改任何全局状态，状态通过ctx传递 |

### 4.4 错误处理

| 错误场景 | 处理方式 | 返回值 |
|----------|----------|--------|
| psutil导入失败 | 首次init时返回错误 | `{"status": "error", "error": "psutil not available"}` |
| 单项采集失败 | 跳过该项，其余继续 | 该项值为 `{"error": "描述"}` |
| SHM写入失败 | 记录错误，下次周期重试 | `{"status": "error", "shm_error": "..."}` |
| 进程信息获取异常 | 跳过该进程 | 不包含在top列表中 |

---

## 5. 测试场景（验收标准）

### 5.1 单元测试

| 测试ID | 测试场景 | 输入 | 预期输出 |
|--------|----------|------|----------|
| T-001 | CPU采集结构验证 | 正常系统 | 返回dict含percent/per_core/count_logical/count_physical |
| T-002 | 内存采集结构验证 | 正常系统 | 返回dict含total_mb/used_mb/available_mb/percent |
| T-003 | 磁盘采集结构验证 | 正常系统 | 返回dict含partitions列表，每项含mountpoint/total_gb/used_gb/percent |
| T-004 | 网络采集结构验证 | 正常系统 | 返回dict含interfaces字典，每项含bytes_sent/bytes_recv |
| T-005 | 进程采集结构验证 | 正常系统 | 返回dict含total_count/top_cpu/top_memory |
| T-006 | 系统采集结构验证 | 正常系统 | 返回dict含boot_time/uptime_seconds/current_time |
| T-007 | 数据打包格式验证 | 完整采集数据 | pack返回含source/version/timestamp_ns/payload |
| T-008 | CPU值域验证 | 正常系统 | percent在[0, 100]范围内 |
| T-009 | 内存值域验证 | 正常系统 | percent在[0, 100]范围内 |
| T-010 | 磁盘过滤虚拟FS | 正常系统 | partitions中不含tmpfs/proc/sys/devtmpfs |
| T-011 | Top-N数量验证 | top_n=3 | top_cpu和top_memory各返回3条 |
| T-012 | 进程异常容错 | 模拟僵尸进程 | 不崩溃，跳过异常进程 |
| T-013 | 初始化成功 | 默认参数 | status="ok"，state含必要字段 |
| T-014 | 初始化无psutil | mock移除psutil | status="error" |
| T-015 | cleanup清空状态 | 含缓存的ctx | status="ok" |
| T-016 | 网络速率计算 | 两次采样 | 第二次返回非零速率 |
| T-017 | 采集周期控制 | interval=1 | 两次采集间隔 >= 0.9s |
| T-018 | 完整loop迭代 | 默认ctx | 一次完整迭代产出含6个子系统的数据 |

### 5.2 集成测试

| 测试ID | 测试场景 | 预期 |
|--------|----------|------|
| T-I01 | 生产者→SHM→消费者链路 | 消费者可读取到完整的系统信息数据包 |

---

## 6. 依赖与资源声明

### 6.1 外部依赖

| 依赖 | 版本要求 | 用途 | 必需/可选 |
|------|----------|------|-----------|
| psutil | >= 5.0 | 系统信息采集 | 必需 |

### 6.2 SHM资源

| 资源 | 路径 | 权限 | 大小估计 |
|------|------|------|----------|
| 系统信息矢量 | `/shm/system/info` | WRITE | ~16KB/次 |

### 6.3 计算资源

| 资源 | 估计值 |
|------|--------|
| 内存占用 | < 5MB |
| CPU占用（采集时） | < 1% |
| 磁盘IO | 无 |

---

## 7. 安全与隔离铁律

| 规则ID | 规则描述 | 检查方式 |
|--------|----------|----------|
| SEC-1 | 仅读取系统信息，禁止修改任何系统配置 | 静态代码审查：无write类系统调用 |
| SEC-2 | 禁止读取其他进程的内存空间 | 静态代码审查：无ptrace/mem_read |
| SEC-3 | 禁止访问网络（仅采集本地信息） | 静态代码审查：无socket/connect |
| SEC-4 | 禁止执行外部命令 | 静态代码审查：无os.system/subprocess/exec |
| SEC-5 | 进程列表采集不包含命令行参数（防信息泄露） | 代码审查：仅采集pid/name/cpu/memory |
| SEC-6 | 所有状态通过ctx传递，零全局变量 | 静态分析 |

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 授权文件字段

| 字段 | 值 |
|------|-----|
| auth_header.blueprint_id | `BP-0031-SYSINFO` |
| auth_header.producer_name | `Genesis-Worker-001` |
| design_meta.function_name | `sysinfo_init / sysinfo_loop` |
| design_meta.language | `Python` |
| design_meta.entry_symbol | `sysinfo_init` |

### 8.2 编译要求

- Python源码编译为PYC（marshal格式）
- entry_symbol解析：`sysinfo_init`
- 备用fallback：`compile(source)` → `exec`

### 8.3 验证要求

- 全部18项单元测试通过
- 1项集成测试通过
- 静态规则检查：SEC-1~SEC-6全部通过
