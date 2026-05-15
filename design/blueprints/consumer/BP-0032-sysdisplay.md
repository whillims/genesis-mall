# 蓝图文档（定稿版）：BP-0032 -- 计算机系统状态显示消费者

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | `BP-0032-SYSDISPLAY` |
| 蓝图名称 | 计算机系统状态显示消费者 |
| 生产者 | `Genesis-Worker-001` |
| 提交时间 | `2026-05-11` |
| 版本 | `1.0.0` |
| 目标语言 | Python |
| 复杂度等级 | `Level-0`（创世纪验证级） |
| 设计范式 | `纯函数级，禁止OOP封装` |
| 关联生产者 | `BP-0031-SYSINFO` |
| 消费者类型 | 终端型（CT）— 实时渲染至标准输出 |

---

## 2. 功能概述

本蓝图定义一个**计算机系统状态显示消费者**。它订阅系统信息采集生产者（BP-0031）写入SHM的系统指标数据，将原始指标格式化为人类可读的文本布局，渲染至标准输出（终端）。

### 2.1 功能边界

| 范围 | 说明 |
|------|------|
| **负责** | 从SHM读取系统信息、格式化为文本布局、渲染至stdout |
| **不负责** | 系统信息采集、阈值告警、历史数据存储、Web可视化 |
| **输入** | SHM矢量 `/shm/system/info`（由BP-0031写入） |
| **输出** | 格式化文本 → stdout |

### 2.2 显示布局

```
╔══════════════════════════════════════════════════════════╗
║  系统状态监控  |  2026-05-11 16:40:00  |  间隔: 2s     ║
╠══════════════════════════════════════════════════════════╣
║  CPU  使用率: 23.5%  |  核心: 4C/8T  |  频率: 3200MHz ║
║  ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ║
╠══════════════════════════════════════════════════════════╣
║  内存  已用: 4.2GB / 16.0GB (26.3%)                     ║
║  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ║
║  交换  已用: 0.0GB / 4.0GB (0.0%)                       ║
╠══════════════════════════════════════════════════════════╣
║  磁盘  C:  120GB/256GB (46.9%)  D: 340GB/512GB (66.4%) ║
╠══════════════════════════════════════════════════════════╣
║  网络  eth0: ↑1.2MB/s ↓3.4MB/s  |  lo: ↑0.0MB/s ↓0.0  ║
╠══════════════════════════════════════════════════════════╣
║  进程  总数: 245  |  运行时间: 3d 12h 30m               ║
║  Top CPU: python(1234) 12.3% chrome(5678) 8.1% ...     ║
║  Top MEM: chrome(5678) 1024MB python(1234) 512MB ...    ║
╚══════════════════════════════════════════════════════════╝
```

---

## 3. 函数设计（函数级，无类）

### 3.1 核心函数清单

| 函数名 | 职责 | 调用关系 |
|--------|------|----------|
| `sysdisplay_init(ctx)` | 初始化消费者，订阅SHM矢量 | 商场main()调用一次 |
| `sysdisplay_loop(ctx)` | 主渲染循环，读取SHM并格式化输出 | 独立Worker进程持续运行 |
| `sysdisplay_render_cpu(ctx, data)` | 格式化CPU区域 | `sysdisplay_loop()`内调用 |
| `sysdisplay_render_memory(ctx, data)` | 格式化内存区域 | `sysdisplay_loop()`内调用 |
| `sysdisplay_render_disk(ctx, data)` | 格式化磁盘区域 | `sysdisplay_loop()`内调用 |
| `sysdisplay_render_network(ctx, data)` | 格式化网络区域 | `sysdisplay_loop()`内调用 |
| `sysdisplay_render_process(ctx, data)` | 格式化进程区域 | `sysdisplay_loop()`内调用 |
| `sysdisplay_render_bar(ctx, percent, width)` | 生成进度条字符串 | 各render函数内调用 |
| `sysdisplay_format_uptime(ctx, seconds)` | 格式化运行时间 | `sysdisplay_loop()`内调用 |
| `sysdisplay_cleanup(ctx)` | 释放资源 | 商场终止时调用 |

### 3.2 函数详细设计

#### `sysdisplay_init(ctx) -> dict`

```
参数:
  ctx : dict - 商场上下文，包含：
    ctx["shm_vector"]      : string  - 订阅的SHM矢量路径，默认 "/shm/system/info"
    ctx["refresh_interval"] : int    - 刷新周期（秒），默认 2
    ctx["bar_width"]        : int    - 进度条宽度（字符数），默认 50
    ctx["color_enabled"]    : bool   - 是否启用ANSI颜色，默认 False

返回:
  {"status": "ok", "state": {...}}   : 成功
  {"status": "error", "error": "..."} : 失败

行为:
  1. 注册SHM读取权限
  2. 清屏（ANSI escape: \033[2J\033[H）
  3. 返回包含初始状态的ctx副本
```

#### `sysdisplay_loop(ctx) -> dict`

```
参数:
  ctx : dict - 包含消费者运行状态

返回:
  每次迭代返回 {"status": "ok", "displayed": true, "timestamp_ns": ...}

行为:
  while ctx["running"]:
      raw = mall_shm_read(ctx["shm_handle"])
      if raw is None:
          sleep(0.5)
          continue
      data = raw["payload"] if "payload" in raw else raw
      output = ""
      output += sysdisplay_render_header(ctx, data)
      output += sysdisplay_render_cpu(ctx, data.get("cpu", {}))
      output += sysdisplay_render_memory(ctx, data.get("memory", {}))
      output += sysdisplay_render_disk(ctx, data.get("disk", {}))
      output += sysdisplay_render_network(ctx, data.get("network", {}))
      output += sysdisplay_render_process(ctx, data.get("process", {}))
      output += sysdisplay_render_footer(ctx)
      print(output, flush=True)
      sleep(ctx["refresh_interval"])
```

#### `sysdisplay_render_cpu(ctx, cpu_data) -> str`

```
参数:
  cpu_data : dict - CPU采集数据

返回:
  格式化的CPU区域字符串（2行：信息行 + 进度条行）

行为:
  1. 提取percent、count_physical、count_logical、freq_current_mhz
  2. 生成进度条：sysdisplay_render_bar(ctx, percent, ctx["bar_width"])
  3. 拼接为格式化字符串
  4. 缺失字段用 "N/A" 填充
```

#### `sysdisplay_render_memory(ctx, mem_data) -> str`

```
参数:
  mem_data : dict - 内存采集数据

返回:
  格式化的内存区域字符串（3行：物理内存+进度条、交换分区）

行为:
  1. 格式化物理内存：已用/总量 (百分比)
  2. 生成进度条
  3. 格式化交换分区
  4. 单位自动转换：MB→GB（>=1024MB时）
```

#### `sysdisplay_render_disk(ctx, disk_data) -> str`

```
参数:
  disk_data : dict - 磁盘采集数据

返回:
  格式化的磁盘区域字符串（1行，多分区横向排列）

行为:
  1. 遍历partitions列表
  2. 每个分区格式化为 "挂载点: 已用/总量 (百分比)"
  3. 分区间用 " | " 分隔
  4. 超过终端宽度时换行
```

#### `sysdisplay_render_network(ctx, net_data) -> str`

```
参数:
  net_data : dict - 网络采集数据

返回:
  格式化的网络区域字符串（1行）

行为:
  1. 遍历interfaces
  2. 每个网卡格式化为 "网卡名: ↑发送速率 ↓接收速率"
  3. 速率自动转换单位：B/s → KB/s → MB/s
  4. 排除lo回环接口（可选）
```

#### `sysdisplay_render_process(ctx, proc_data) -> str`

```
参数:
  proc_data : dict - 进程采集数据

返回:
  格式化的进程区域字符串（3行：总数+运行时间、Top CPU、Top MEM）

行为:
  1. 格式化进程总数
  2. 从system数据获取uptime，调用sysdisplay_format_uptime
  3. Top CPU: 格式化为 "进程名(PID) CPU%"
  4. Top MEM: 格式化为 "进程名(PID) 内存MB"
  5. 内存自动转换：MB→GB（>=1024MB时）
```

#### `sysdisplay_render_bar(ctx, percent, width) -> str`

```
参数:
  percent : float - 百分比值 (0-100)
  width   : int    - 进度条总宽度

返回:
  进度条字符串，如 "████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░"

行为:
  1. 将percent限制在[0, 100]
  2. 计算填充字符数: filled = int(width * percent / 100)
  3. 用 "█" 表示已用，"░" 表示未用
  4. 返回长度为width的字符串
```

#### `sysdisplay_format_uptime(ctx, seconds) -> str`

```
参数:
  seconds : float - 运行秒数

返回:
  格式化字符串，如 "3d 12h 30m" 或 "45m 12s"

行为:
  1. 计算天、时、分、秒
  2. 省略为零的高位单位
  3. 最多显示3个单位
```

#### `sysdisplay_cleanup(ctx) -> dict`

```
返回:
  {"status": "ok"}

行为:
  - 清空ctx中的渲染状态
  - 重置终端（ANSI reset）
  - 返回成功
```

---

## 4. 行为契约

### 4.1 前置条件

| 条件ID | 条件描述 |
|--------|----------|
| PRE-1 | BP-0031生产者已启动并写入SHM |
| PRE-2 | 商场SHM系统已初始化 |
| PRE-3 | stdout可写（非重定向到closed fd） |

### 4.2 后置条件

| 条件ID | 条件描述 |
|--------|----------|
| POST-1 | 每个刷新周期产出一帧完整的系统状态文本 |
| POST-2 | 输出文本可通过终端正确显示 |
| POST-3 | SHM读取失败时显示 "等待数据..." 而非崩溃 |

### 4.3 不变量

| 不变量ID | 不变量描述 |
|----------|-----------|
| INV-1 | 返回值始终为dict结构 |
| INV-2 | 渲染输出为纯文本（str），不含二进制数据 |
| INV-3 | 进度条长度恒等于ctx["bar_width"] |
| INV-4 | 不修改任何全局状态，状态通过ctx传递 |

### 4.4 错误处理

| 错误场景 | 处理方式 | 返回值 |
|----------|----------|--------|
| SHM无数据 | 显示等待提示，继续轮询 | `{"status": "ok", "displayed": false}` |
| SHM数据格式异常 | 显示原始数据，标记格式错误 | 渲染含 `[FORMAT ERROR]` |
| 数据字段缺失 | 用 "N/A" 填充缺失字段 | 正常渲染，N/A显示 |
| stdout写入失败 | 静默忽略，下次周期重试 | `{"status": "error", "displayed": false}` |
| 百分比值越界 | 钳制到[0, 100] | 正常渲染 |

---

## 5. 测试场景（验收标准）

### 5.1 单元测试

| 测试ID | 测试场景 | 输入 | 预期输出 |
|--------|----------|------|----------|
| T-001 | CPU渲染正常数据 | 完整cpu dict | 包含使用率、核心数、频率、进度条 |
| T-002 | CPU渲染缺失字段 | 空dict | 显示 "N/A" |
| T-003 | 内存渲染正常数据 | 完整memory dict | 包含物理内存+交换分区 |
| T-004 | 内存MB→GB转换 | 2048MB | 显示 "2.0GB" |
| T-005 | 磁盘渲染多分区 | 3个分区 | 3个分区用 "|" 分隔 |
| T-006 | 磁盘渲染空列表 | partitions=[] | 显示 "无磁盘信息" |
| T-007 | 网络渲染正常数据 | 完整network dict | 包含网卡名和速率 |
| T-008 | 网络速率单位转换 | 1536 bytes/s | 显示 "1.5KB/s" |
| T-009 | 进程渲染Top-N | top_n=3 | 各显示3条进程 |
| T-010 | 进程渲染空列表 | top_cpu=[] | 显示 "无进程数据" |
| T-011 | 进度条0% | percent=0 | 全 "░" |
| T-012 | 进度条100% | percent=100 | 全 "█" |
| T-013 | 进度条50% | percent=50, width=10 | 5个"█" + 5个"░" |
| T-014 | 进度条越界钳制 | percent=150 | 等同于100% |
| T-015 | 进度条负值钳制 | percent=-10 | 等同于0% |
| T-016 | 运行时间格式化 | 90061秒 | "1d 1h 1m" |
| T-017 | 运行时间短格式 | 45秒 | "45s" |
| T-018 | 初始化成功 | 默认参数 | status="ok" |
| T-019 | cleanup成功 | 正常ctx | status="ok" |
| T-020 | 完整渲染帧 | 全量模拟数据 | 输出包含所有6个区域 |

### 5.2 集成测试

| 测试ID | 测试场景 | 预期 |
|--------|----------|------|
| T-I01 | BP-0031→SHM→BP-0032链路 | 消费者终端显示完整系统状态面板 |
| T-I02 | 生产者停止后消费者行为 | 显示 "等待数据..." 不崩溃 |

---

## 6. 依赖与资源声明

### 6.1 外部依赖

| 依赖 | 版本要求 | 用途 | 必需/可选 |
|------|----------|------|-----------|
| 无外部依赖 | - | 纯文本渲染 | - |

### 6.2 SHM资源

| 资源 | 路径 | 权限 | 大小估计 |
|------|------|------|----------|
| 系统信息矢量 | `/shm/system/info` | READ | ~16KB/次 |

### 6.3 计算资源

| 资源 | 估计值 |
|------|--------|
| 内存占用 | < 2MB |
| CPU占用（渲染时） | < 0.5% |
| 终端IO | 每帧 ~2KB文本 |

---

## 7. 安全与隔离铁律

| 规则ID | 规则描述 | 检查方式 |
|--------|----------|----------|
| SEC-1 | 仅读取SHM数据，禁止写入任何SHM矢量 | 静态代码审查：无SHM写操作 |
| SEC-2 | 仅输出至stdout，禁止写文件 | 静态代码审查：无open/write/file操作 |
| SEC-3 | 禁止执行外部命令 | 静态代码审查：无os.system/subprocess/exec |
| SEC-4 | 禁止网络访问 | 静态代码审查：无socket/connect |
| SEC-5 | 进程名显示不包含命令行参数 | 代码审查：仅显示name字段 |
| SEC-6 | 所有状态通过ctx传递，零全局变量 | 静态分析 |

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 授权文件字段

| 字段 | 值 |
|------|-----|
| auth_header.blueprint_id | `BP-0032-SYSDISPLAY` |
| auth_header.producer_name | `Genesis-Worker-001` |
| design_meta.function_name | `sysdisplay_init / sysdisplay_loop` |
| design_meta.language | `Python` |
| design_meta.entry_symbol | `sysdisplay_init` |

### 8.2 编译要求

- Python源码编译为PYC（marshal格式）
- entry_symbol解析：`sysdisplay_init`
- 备用fallback：`compile(source)` → `exec`

### 8.3 验证要求

- 全部20项单元测试通过
- 2项集成测试通过
- 静态规则检查：SEC-1~SEC-6全部通过
