# 管理员画像系统UI消费者蓝图

> **蓝图编号**: BP-0055-ADMIN-PROFILE-UI
> **版本**: v1.1.0
> **日期**: 2026-05-14
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0055-ADMIN-PROFILE-UI |
| 函数名 | admin_profile_ui_init / admin_profile_ui_tick |
| 生产者名称 | AdminProfileUIConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | SHM矢量订阅 (VEC-DISPLAY) |
| 输出协议 | SHM矢量发布 (VEC-CONTROL) |
| 安全等级 | P1 (内部工具) |
| 版本 | v1.1.0 |

---

## 二、设计意图

### 2.1 核心目标

为高性能数据处理平台提供管理员画像可视化界面，展示管理员角色信息、权限范围、操作行为统计，以及与平台核心模块（数据总线、撮合引擎、风控中心）的关联映射。

### 2.2 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | UI作为消费者域，只订阅SHM矢量，不直接操作商场 |
| 函数范式 | UI组件纯函数设计，状态外置Pinia Store |
| SHM矢量空间 | 通过VEC-DISPLAY订阅数据，VEC-CONTROL发布命令 |
| 授权文件体系 | UI消费者需AUTH-BP-0055授权文件激活 |
| 基因图谱风格 | 深色主题(#0A0E17) + 金色(#D4A843) + 翡翠绿(#00C9A7) |
| 嵌入式仪器风格 | instrument-panel/header/btn/LED/digital-readout 全局样式体系 |
| Nginx钢铁脊椎 | 静态资源由Nginx直接服务，API通过Nginx反向代理 |
| Worker类型 | 消费者Worker，被动接收数据更新 |

### 2.3 v1.1.0 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| 嵌入式仪器风格替代通用Web UI | 与TR测试工程师UI风格统一，体现工业控制平台定位 | 通用Dashboard风格 | 全局CSS类体系重构，11个组件/视图更新 |
| Tailwind任意值 `w-[200px]` | Tailwind v3默认间距表无50值，`w-50`不生效 | `w-48`(192px)或`w-52`(208px) | 侧边栏/主内容区必须使用任意值语法 |
| LED红色使用 `#E53E3E` | 原设计`#E8B84B`实为金色，语义错误 | 保持原色 | 风控告警等错误状态必须用红色 |
| 全局样式替代scoped重复 | StatCard/LatencyGauge中scoped样式与全局重复 | 保留scoped | 移除scoped后CSS体积减少1.2KB |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 商场 | 高性能数据处理平台 | 整体系统平台展示 |
| 交易 | 数据采集/处理事件 | 管理员操作记录 |
| 撮合 | 数据匹配/关联处理 | 撮合引擎状态监控 |
| SHM数据总线 | 共享内存高速传输通道 | 实时数据可视化 |
| 商户 | 数据源/数据生产者 | 数据通道来源 |
| 顾客 | 数据消费者/订阅者 | 下游处理模块 |
| 仪器面板 | 数据监控模块 | instrument-panel容器 |
| LED指示灯 | 状态信号 | led green/yellow/red |

### 3.2 数据流向

```
┌─────────────────┐     SHM矢量      ┌─────────────────┐
│   商场本体       │ ───────────────▶ │  AdminProfile   │
│  (数据生产者)    │   VEC-DISPLAY    │    UI消费者     │
│                 │                  │                 │
│  - 管理员数据    │ ◀─────────────── │  - 界面渲染      │
│  - 总线状态      │   VEC-CONTROL    │  - 用户交互      │
│  - 引擎状态      │                  │  - 命令发送      │
│  - 风控告警      │                  │                 │
└─────────────────┘                  └─────────────────┘
                                              │
                                              ▼
                                       ┌─────────────┐
                                       │  Nginx      │
                                       │ 钢铁脊椎     │
                                       └─────────────┘
```

---

## 四、行为契约

### 4.1 输入契约 (VEC-DISPLAY订阅)

```typescript
interface DisplayFrame {
  frame_type: 'admin_profile' | 'bus_status' | 'engine_status' | 'risk_alert'
  timestamp: number
  sequence: number
  data: AdminProfileData | BusStatusData | EngineStatusData | RiskAlertData
}

interface AdminProfileData {
  admin_id: string
  name: string
  employee_id: string
  role: string
  platform_id: string
  platform_name: string
  contact: { phone: string; email: string }
  permissions: PermissionModule[]
  responsibilities: string[]
  statistics: {
    login_count: number
    last_login_at: string
    active_hours: { peak: string[] }
    operation_distribution: OperationStat[]
  }
}

interface PermissionModule {
  module: string
  items: string[]
}

interface OperationStat {
  label: string
  percent: number
  count: number
  color: string
}

interface BusStatusData {
  health: 'healthy' | 'warning' | 'critical'
  uptime: number
  channels: { total: number; active: number; inactive: number }
  throughput: {
    current_msg_rate: number
    current_data_rate: number
    peak_msg_rate: number
    peak_data_rate: number
  }
  latency: {
    avg_write: number
    avg_read: number
    p95_write: number
    p99_write: number
    avg_end_to_end: number
    p95_end_to_end: number
    p99_end_to_end: number
  }
  memory: {
    total_size: number
    used_size: number
    usage_percent: number
    fragmentation: number
    free_percent: number
  }
  connections: {
    producers: number
    consumers: number
    abnormal_disconnects: number
  }
  channel_list: Channel[]
}

interface Channel {
  id: string
  name: string
  type: string
  status: 'active' | 'inactive'
  msg_rate: number
  avg_latency: number
}

interface EngineStatusData {
  engine_id: string
  name: string
  status: 'running' | 'stopped' | 'error'
  uptime: number
  performance: {
    match_success_rate: number
    avg_process_time: number
    throughput: number
    queue_depth: number
    max_queue_depth: number
  }
  rules: MatchRule[]
  statistics: {
    total_processed: number
    match_success: number
    match_failed: number
    failure_reasons: FailureReason[]
  }
  resources: { cpu_usage: number; memory_usage: number; thread_count: number }
}

interface MatchRule {
  id: string
  name: string
  enabled: boolean
  time_window: number
  time_window_unit: string
  match_key: string
  conditions: string
}

interface FailureReason {
  reason: string
  count: number
  percent: number
}

interface RiskAlertData {
  alerts: AlertEvent[]
  alert_count: number
  critical_count: number
  warning_count: number
}

interface AlertEvent {
  id: string
  rule_id: string
  rule_name: string
  level: 'critical' | 'warning' | 'info'
  status: 'active' | 'acknowledged' | 'resolved'
  message: string
  triggered_at: string
  duration: string
  timeline: TimelineEvent[]
}

interface TimelineEvent {
  time: string
  action: string
  operator: string
}

interface RiskRule {
  id: string
  name: string
  type: string
  enabled: boolean
  threshold: string
  actions: string
  trigger_count: number
}
```

### 4.2 输出契约 (VEC-CONTROL发布)

```typescript
interface ControlFrame {
  frame_type: 'command' | 'config_update' | 'acknowledge_alert'
  timestamp: number
  sequence: number
  admin_id: string
  command: CommandData
}

interface CommandData {
  type: 'refresh' | 'switch_view' | 'execute_cmd' | 'ack_alert'
  params: Record<string, any>
}

interface ChatMessage {
  id: string
  role: 'user' | 'assistant'
  type: 'text' | 'data_card' | 'chart'
  content: string
  data?: Record<string, any>
  timestamp: number
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | Worker启动 | CONNECTING | 初始化SHM连接 |
| CONNECTING | VEC-DISPLAY订阅成功 | ACTIVE | 开始接收数据帧 |
| ACTIVE | 收到数据帧 | RENDERING | 解析并渲染UI |
| RENDERING | 渲染完成 | ACTIVE | 等待下一帧 |
| ACTIVE | 用户交互 | COMMAND | 构建控制帧 |
| COMMAND | 发布VEC-CONTROL | ACTIVE | 返回等待状态 |
| ACTIVE | 连接断开 | RECONNECTING | 尝试重连 |
| RECONNECTING | 重连成功 | ACTIVE | 恢复数据接收 |
| RECONNECTING | 重连失败 | ERROR | 报告错误 |

---

## 五、UI设计规范

### 5.1 色彩体系 (基因图谱风格 + 嵌入式仪器)

#### 主色板

| 色彩角色 | 色值 | 用途 |
|----------|------|------|
| 深空黑 | `#0A0E17` | 主背景色 |
| 暗夜灰 | `#141B2D` | 卡片/面板背景、instrument-panel渐变终点 |
| 炭灰色 | `#1E2A3A` | 边框、分割线、instrument-header渐变起点 |
| 皇室金 | `#D4A843` | 主强调色、标题、instrument-header文字、digital-readout |
| 浅金色 | `#F0D78C` | 次强调色、LED黄色高光点 |
| 翡翠绿 | `#00C9A7` | 正常状态、LED绿色、增长指标、成功 |
| 深绿色 | `#0A7E5C` | 图表辅助色 |
| 月光白 | `#E8ECF1` | 主文本色、仪表盘指针 |
| 银灰色 | `#8892A4` | 次要文本、说明文字、仪表盘中心点 |
| 薄雾灰 | `#3A4556` | 占位符、禁用态、仪表盘刻度线 |
| 烈焰红 | `#E53E3E` | 错误状态、LED红色、critical告警 |
| 珊瑚红 | `#FF6B6B` | LED红色高光点 |

#### 语义色

| 状态 | 色值 | 用途 | LED实现 |
|------|------|------|---------|
| 成功 | `#00C9A7` | 操作成功、健康状态 | `led.green` radial-gradient(#33FFD4, #00C9A7) |
| 警告 | `#D4A843` | 预警提示、需要关注 | `led.yellow` radial-gradient(#F0D78C, #D4A843) |
| 错误 | `#E53E3E` | 故障、异常、critical告警 | `led.red` radial-gradient(#FF6B6B, #E53E3E) + pulse动画 |
| 信息 | `#00C9A7` | 一般提示信息 | 同成功色 |

### 5.2 嵌入式仪器风格体系

#### 5.2.1 仪器面板 (instrument-panel)

```
┌─────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│  ← 2px金色渐变装饰线 (::before)
├─────────────────────────────────────┤
│  INSTRUMENT HEADER          MOD-001│  ← instrument-header
│─────────────────────────────────────│
│                                     │
│         内容区域                     │
│                                     │
└─────────────────────────────────────┘
```

**CSS规格**:
```css
.instrument-panel {
  position: relative;
  background: linear-gradient(145deg, #141B2D 0%, #0F1623 100%);
  border: 1px solid #1E2A3A;
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.03), 0 2px 8px rgba(0,0,0,0.4);
  border-radius: 2px;
}
.instrument-panel::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 2px;
  background: linear-gradient(90deg, transparent, rgba(212,168,67,0.6), transparent);
}
```

#### 5.2.2 仪器头部 (instrument-header)

**CSS规格**:
```css
.instrument-header {
  background: linear-gradient(180deg, #1E2A3A 0%, #141B2D 100%);
  border-bottom: 1px solid #0A0E17;
  padding: 8px 12px;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 1px;
  color: #D4A843;
  font-family: 'JetBrains Mono', monospace;
}
```

#### 5.2.3 数字读数 (digital-readout)

**CSS规格**:
```css
.digital-readout {
  font-family: 'JetBrains Mono', monospace;
  font-variant-numeric: tabular-nums;
  text-shadow: 0 0 12px rgba(212, 168, 67, 0.4);
  letter-spacing: 0.05em;
}
```

#### 5.2.4 仪器按钮 (instrument-btn)

**CSS规格**:
```css
.instrument-btn {
  background: linear-gradient(180deg, #1E2A3A 0%, #141B2D 100%);
  border: 1px solid #3A4556;
  color: #8892A4;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 1px;
  font-family: 'JetBrains Mono', monospace;
  transition: all 0.15s ease;
  border-radius: 2px;
}
.instrument-btn:hover {
  border-color: #D4A843;
  color: #D4A843;
  background: linear-gradient(180deg, #252F3F 0%, #1A2230 100%);
}
.instrument-btn:active {
  background: #0A0E17;
  transform: translateY(1px);
}
.instrument-btn.active {
  background: rgba(0, 201, 167, 0.15);
  border-color: #00C9A7;
  color: #00C9A7;
  box-shadow: 0 0 8px rgba(0, 201, 167, 0.2);
}
```

#### 5.2.5 LED指示灯 (led)

**CSS规格**:
```css
.led {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  box-shadow: inset 0 1px 2px rgba(0,0,0,0.5);
}
.led.green {
  background: radial-gradient(circle at 30% 30%, #33FFD4, #00C9A7);
  box-shadow: 0 0 8px rgba(0, 201, 167, 0.6), inset 0 1px 2px rgba(0,0,0,0.3);
}
.led.yellow {
  background: radial-gradient(circle at 30% 30%, #F0D78C, #D4A843);
  box-shadow: 0 0 8px rgba(212, 168, 67, 0.6), inset 0 1px 2px rgba(0,0,0,0.3);
}
.led.red {
  background: radial-gradient(circle at 30% 30%, #FF6B6B, #E53E3E);
  box-shadow: 0 0 8px rgba(229, 62, 62, 0.6), inset 0 1px 2px rgba(0,0,0,0.3);
  animation: pulse 1.5s ease-in-out infinite;
}
```

#### 5.2.6 辅助样式

| 类名 | 用途 | 实现 |
|------|------|------|
| `.instrument-divider` | 仪器内分割线 | `linear-gradient(90deg, transparent, #3A4556, transparent)` |
| `.instrument-scale` | 仪表盘刻度标注 | flex + justify-between + 9px字号 |
| `.instrument-grid-bg` | 仪器网格背景 | 20px网格线 `rgba(30,42,58,0.3)` |
| `.scanline` | 扫描线效果 | `::after` 2px绿线 3s下移动画 |
| `.grid-bg` | 页面网格背景 | 20px网格线 `rgba(30,42,58,0.5)` |

### 5.3 布局架构

#### 整体布局

```
┌──────────────────────────────────────────────────────────┐
│  SHM DATA PLATFORM    Admin Profile System               │  ← TopBar (56px, fixed)
│  [LED●] Connected  [CLK 10:00:00]  [🔔] [⚙️] [ADMIN]   │     底部1px金色渐变线
├──────────┬───────────────────────────────────────────────┤
│          │                                               │
│  侧边栏   │              主内容区                          │
│ [200px]  │  (padding-left: 200px, padding: 24px)         │
│  fixed   │                                               │
│          │  ┌─────────────────────────────────────────┐   │
│ ▌PROFILE │  │ ▎ ADMIN PROFILE           [Refresh]    │   │
│ ▌BUS     │  ├─────────────────────────────────────────┤   │
│ ▌ENGINE  │  │                                         │   │
│ ▌RISK    │  │    instrument-panel 卡片/仪表盘/表格     │   │
│ ▌COMMAND │  │                                         │   │
│          │  └─────────────────────────────────────────┘   │
│          │                                               │
│ ┌──────┐ │                                               │
│ │SYS   │ │                                               │
│ │ADMIN │ │                                               │
│ │●Online│ │                                               │
│ └──────┘ │                                               │
├──────────┴───────────────────────────────────────────────┤
│  ● 系统正常 | REF: 10:00 | CPU 23% | MEM 47% | v1.0.0    │  ← StatusBar (32px, fixed)
└──────────────────────────────────────────────────────────┘
```

#### 关键布局参数

| 元素 | Tailwind类 | 实际值 | 说明 |
|------|-----------|--------|------|
| TopBar | `fixed top-0 h-14` | 56px | 底部1px金色渐变线 |
| Sidebar | `fixed top-14 bottom-8 w-[200px]` | 200px | 右侧1px金色渐变线 |
| StatusBar | `fixed bottom-0 h-8` | 32px | 顶部1px金色渐变线 |
| Main | `pt-14 pb-8 pl-[200px]` | 56px+32px+200px | ⚠️ 必须使用任意值语法 |
| Main padding | `p-6` | 24px | 内容区内边距 |

> **⚠️ 关键约束**: Tailwind CSS v3默认间距表无`50`值（只有48=12rem和52=13rem），侧边栏宽度和主内容区左内边距**必须**使用任意值语法 `w-[200px]` / `pl-[200px]`，否则样式不生效导致布局完全错乱。

#### 响应式断点

| 断点 | 代号 | 侧边栏 | 内容网格 |
|------|------|--------|----------|
| >= 1024px | Desktop | 固定展开 200px | 4列KPI / 3列仪表盘 / 2列图表 |
| 768px - 1023px | Tablet | 抽屉菜单(w-64) | 2列KPI / 3列仪表盘 / 2列图表 |
| < 768px | Mobile | 抽屉菜单(w-64) | 2列KPI / 1列仪表盘 / 1列图表 |

### 5.4 页面模块

#### 管理员画像页 (ProfileView)

```
├── 页面标题 (▎ ADMIN PROFILE, 金色竖线+大写等宽)
├── 个人信息卡片 + 权限树 (左右布局, lg:grid-cols-3)
│   ├── instrument-panel: 头像+姓名+ID+联系方式
│   └── instrument-panel: PERMISSION SCOPE + 模块权限标签
├── 操作统计行 (4x StatCard, grid-cols-2 md:grid-cols-4)
│   ├── Login Count (MOD-101)
│   ├── Last Login (MOD-102)
│   ├── Modules (MOD-103)
│   └── Peak Hours (MOD-104)
└── instrument-panel: OPERATION DISTRIBUTION
    ├── LED + 标签 + 百分比列表
    └── 渐变堆叠条形图
```

#### 数据总线监控页 (BusMonitorView)

```
├── 页面标题 (▎ BUS MONITOR) + instrument-btn Refresh
├── KPI行 (4x StatCard, grid-cols-2 md:grid-cols-4)
│   ├── Msg Throughput (MOD-301)
│   ├── Data Rate (MOD-302)
│   ├── Avg Latency (MOD-303)
│   └── Mem Usage (MOD-304, showBar)
├── 延迟仪表盘行 (3x LatencyGauge, grid-cols-1 md:grid-cols-3)
│   ├── Write Latency (P95/P99)
│   ├── Read Latency (P95/P99)
│   └── E2E Latency (P95/P99)
└── instrument-panel: CHANNEL LIST
    ├── instrument-header: 通道数统计
    └── 表格: Channel ID / Name / Type / Status(LED) / Msg Rate / Latency
```

#### 撮合引擎管理页 (EngineManageView)

```
├── 页面标题 (▎ MATCH ENGINE) + instrument-btn Refresh
├── 状态行 (grid-cols-1 md:grid-cols-3)
│   ├── instrument-panel: ENGINE STATUS (LED + StatusBadge + Uptime)
│   ├── StatCard: Match Rate (MOD-201)
│   └── StatCard: Queue Depth (MOD-202)
├── instrument-panel: MATCH RULES
│   └── 表格: Rule Name / Window / Match Key / Condition / 启停开关
└── 统计与资源行 (grid-cols-1 md:grid-cols-2)
    ├── instrument-panel: PROCESS STATISTICS (总数/匹配/失败+原因条形图)
    └── instrument-panel: RESOURCE USAGE (CPU/MEM进度条+线程数)
```

#### 风控中心页 (RiskCenterView)

```
├── 页面标题 (▎ RISK CENTER)
├── 告警摘要行 (4x instrument-panel, grid-cols-2 md:grid-cols-4)
│   ├── Active Alerts (LED red/green + 数字)
│   ├── Critical (LED red/green + 数字)
│   ├── Warnings (LED yellow/green + 数字)
│   └── Resolved (数字)
├── instrument-panel: ACTIVE ALERTS
│   ├── instrument-header: 过滤按钮 (instrument-btn active)
│   └── 告警列表: LED + 规则名 + StatusBadge + 消息 + 时间 + ACK按钮
└── instrument-panel: RISK RULES
    └── 表格: Rule Name / Type / Threshold / Action / Triggers / 启停开关
```

#### 命令交互中心页 (CommandCenterView)

```
├── 页面标题 (命令交互中心) + 清除按钮
├── instrument-panel聊天界面 (flex-col, height: calc(100vh - 180px))
│   ├── 头部 (Bot图标 + 智能助手 + StatusBadge在线)
│   ├── 消息区域 (flex-1, overflow-y-auto)
│   │   ├── data_card: instrument-panel + instrument-header + key-value网格
│   │   └── text: 用户(金边) / 助手(绿底) 气泡
│   ├── 快捷命令面板 (flex-wrap pill按钮)
│   └── 输入区域 (文本输入 + 发送按钮)
```

### 5.5 组件规范

#### 数据指标卡片 (StatCard)

```
┌─────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│  ← 2px金色装饰线
├─────────────────────────────────────┤
│  [📊] MSG THROUGHPUT       MOD-301│  ← instrument-header
│─────────────────────────────────────│
│  125,000                            │  ← digital-readout (金色, 2xl, 700)
│  msg/s                              │  ← 单位 (银灰色, xs, uppercase)
│  ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░             │  ← 进度条 (可选, 渐变+blur光晕)
└─────────────────────────────────────┘
```

**Props**:

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| title | string | - | 标题（大写等宽） |
| value | number | - | 数值 |
| unit | string | - | 单位 |
| icon | Component | - | Lucide图标 |
| decimals | number | 0 | 小数位数 |
| trend | string | undefined | 趋势文字 |
| trendPositive | boolean | true | 趋势方向 |
| max | number | 100 | 进度条最大值 |
| showBar | boolean | false | 显示进度条 |
| modelLabel | string | 'MOD-001' | 仪器型号标签 |

**进度条颜色规则**: <60% 翡翠绿 → <85% 皇室金 → >=85% 错误金

#### 延迟仪表盘 (LatencyGauge)

```
┌─────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
├─────────────────────────────────────┤
│  WRITE LATENCY                      │  ← instrument-header
│─────────────────────────────────────│
│          ╱╲                         │
│     ╱╲╱╲  ╲  ╱╲                    │  ← 21刻度线 (每5%主刻度)
│   ╱            ╲                    │
│  ╱  ▓▓▓▓▓▓▓▓▓▓  ╲                  │  ← 渐变弧 (绿→黄→红)
│ ╱    2.3 μs       ╲                 │  ← digital-readout (28px)
│ │         ↗         │                │  ← 指针 (SVG needleGlow滤镜)
│ └─────○─────────────┘                │  ← 中心金属圆点 (radialGradient)
│ 0    2.5   5   7.5   10             │  ← instrument-scale
│  P95: 4.5μs    P99: 8.2μs          │  ← 百分位读数
└─────────────────────────────────────┘
```

**SVG规格**:
- 弧线: 半径80, stroke-width 8, `linearGradient` (#00C9A7 → #D4A843 → #E8B84B)
- 刻度: 21条 (奇数=主刻度1px, 偶数=次刻度0.5px)
- 指针: stroke-width 2, `needleGlow` SVG滤镜 (feGaussianBlur stdDeviation=2)
- 中心: `radialGradient` 金属纹理 + 双圆 (r=5外圈 + r=2内圈)
- 数值: 28px 700字重 digital-readout

#### 状态标签 (StatusBadge)

| 变体 | 背景 | 边框 | 文字色 | LED |
|------|------|------|--------|-----|
| success | `rgba(0,201,167,0.1)` | `border-2 border-emerald/30` | `#00C9A7` | `led.green` |
| warning | `rgba(212,168,67,0.1)` | `border-2 border-royal-gold/30` | `#D4A843` | `led.yellow` |
| error | `rgba(232,184,75,0.1)` | `border-2 border-error-gold/30` | `#E8B84B` | `led.red` + pulse |
| info | `rgba(0,201,167,0.05)` | `border-2 border-emerald/20` | `#00C9A7` | `bg-emerald/50` |

#### 顶部导航栏 (TopBar)

| 元素 | 样式 | 说明 |
|------|------|------|
| 标题 | `SHM DATA PLATFORM` 大写等宽 | 0.15em字间距 |
| 副标题 | `Admin Profile System` 9px | 银灰色 |
| 连接状态 | `led green` + `CONNECTED` | 10px等宽 |
| 系统时钟 | `digital-readout` HH:MM:SS | 每秒刷新 |
| 通知铃铛 | 红色角标计数 | alertCount > 0时显示 |
| 用户区 | 翡翠绿头像框 + `ADMIN` + `ONLINE` | 等宽字体 |
| 底部线 | 1px金色渐变 | `via-royal-gold/40` |

#### 侧边栏 (Sidebar)

| 元素 | 样式 | 说明 |
|------|------|------|
| 导航项 | `instrument-btn` | 11px大写等宽 |
| 激活态 | `!border-l-2 !border-l-royal-gold` | 金色左边框 |
| 底部用户 | `instrument-panel` | LED在线指示 |
| 右侧线 | 1px金色渐变 | `via-royal-gold/30` |

#### 底部状态栏 (StatusBar)

| 元素 | 样式 | 说明 |
|------|------|------|
| 状态LED | `led green/yellow/red` | 根据系统状态变色 |
| 刷新时间 | `REF: HH:MM` | 等宽字体 |
| CPU使用率 | `digital-readout` 翡翠绿 | 5秒刷新模拟值 |
| MEM使用率 | `digital-readout` 皇室金 | 5秒刷新模拟值 |
| 版本号 | `v1.0.0` | 银灰色 |
| 顶部线 | 1px金色渐变 | `via-royal-gold/30` |

### 5.6 基因图谱背景

背景采用HTML5 Canvas绘制的动态基因图谱效果，营造科技感氛围。

**实现规格**:
- 最大节点数: 80个
- 连接距离: 200px范围内
- 最小节点间距: 80px
- 渲染引擎: `requestAnimationFrame`，目标 60fps
- 每节点连接数: 最多4个最近邻居
- 节点颜色: `rgba(212, 168, 67, 0.25)`，脉动光晕 `rgba(212, 168, 67, 0.08)`
- 连线颜色: `rgba(0, 201, 167, opacity)`，opacity随距离衰减

**动画规范**:

| 动画类型 | 描述 | 时长 | 缓动 |
|----------|------|------|------|
| 节点脉动 | 节点大小在2-4px间缓慢变化 | 3-5s | ease-in-out |
| 节点漂移 | 节点在画布内缓慢移动 | 持续 | linear |
| 边界反弹 | 节点触碰边界时反向 | 即时 | - |

---

## 六、技术架构

### 6.1 技术栈选型

| 层级 | 技术 | 版本 | 选型理由 |
|------|------|------|----------|
| 前端框架 | Vue 3 | 3.4.x | 响应式系统，Composition API，TypeScript支持好 |
| 开发语言 | TypeScript | 5.x | 类型安全，IDE支持好 |
| 构建工具 | Vite | 5.x | 极速冷启动，HMR，生产优化 |
| UI样式 | Tailwind CSS | 3.4.x | 原子化CSS，快速开发 |
| 状态管理 | Pinia | 2.x | Vue官方推荐，TypeScript友好 |
| 路由 | Vue Router | 4.x | Vue官方路由 |
| 图标库 | Lucide Vue | latest | 轻量，统一风格 |
| 动画 | CSS Animations + Canvas | 原生 | 配合Tailwind，无需引入动画库 |
| 字体 | JetBrains Mono + Roboto Mono | Google Fonts | 等宽数字读数，工业仪器感 |

### 6.2 项目目录结构

```
admin-profile-ui/
├── public/
│   └── favicon.svg
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── style.css                        # 全局样式 (instrument体系)
│   ├── router.ts                        # 路由配置
│   ├── components/
│   │   ├── GenomeBackground.vue         # Canvas基因图谱背景
│   │   ├── Layout.vue                   # 全局布局 (TopBar+Sidebar+Main+StatusBar)
│   │   ├── TopBar.vue                   # 顶部导航栏 (SHM DATA PLATFORM + 时钟 + LED)
│   │   ├── Sidebar.vue                  # 侧边栏导航 (instrument-btn)
│   │   ├── StatusBar.vue                # 底部状态栏 (LED + CPU/MEM)
│   │   ├── StatCard.vue                 # 数据指标卡片 (instrument-panel + modelLabel)
│   │   ├── LatencyGauge.vue             # SVG延迟仪表盘 (21刻度 + 渐变弧 + 滤镜)
│   │   └── StatusBadge.vue              # 状态标签 (border-2 + LED)
│   ├── views/
│   │   ├── ProfileView.vue              # 管理员画像页
│   │   ├── BusMonitorView.vue           # 数据总线监控页
│   │   ├── EngineManageView.vue         # 撮合引擎管理页
│   │   ├── RiskCenterView.vue           # 风控中心页
│   │   └── CommandCenterView.vue        # 命令交互中心页
│   ├── stores/
│   │   ├── appStore.ts                  # 全局状态 (连接/移动端/告警数)
│   │   ├── profileStore.ts              # 管理员画像数据
│   │   ├── busStore.ts                  # 数据总线状态 (含channel_list)
│   │   ├── engineStore.ts               # 撮合引擎状态 (含rules/statistics)
│   │   ├── riskStore.ts                 # 风控状态 (含alerts/rules)
│   │   └── commandStore.ts              # 命令交互状态 (含shortcuts/消息生成)
│   ├── types/
│   │   └── index.ts                     # 全部TypeScript类型定义
│   └── utils/
│       └── formatters.ts                # 数据格式化 (7个函数)
├── tailwind.config.js
├── vite.config.ts
├── tsconfig.json
├── index.html                           # JetBrains Mono + Roboto Mono 字体加载
└── package.json
```

### 6.3 SHM矢量集成

```typescript
// composables/useSHMVector.ts (待实现)
import { ref, onMounted, onUnmounted } from 'vue'

export function useSHMVector(vectorId: string) {
  const data = ref<any>(null)
  const isConnected = ref(false)
  const error = ref<Error | null>(null)

  let eventSource: EventSource | null = null

  onMounted(() => {
    eventSource = new EventSource(`/api/vector/${vectorId}/stream`)

    eventSource.onmessage = (event) => {
      try {
        const frame = JSON.parse(event.data)
        data.value = frame.data
        isConnected.value = true
      } catch (e) {
        error.value = e as Error
      }
    }

    eventSource.onerror = () => {
      isConnected.value = false
      error.value = new Error('SHM连接断开')
    }
  })

  onUnmounted(() => {
    eventSource?.close()
  })

  const publishControl = async (command: ControlFrame) => {
    const response = await fetch('/api/vector/control/publish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(command)
    })
    return response.json()
  }

  return { data, isConnected, error, publishControl }
}
```

---

## 七、安全约束

### 7.1 输入验证

| 检查项 | 规则 | 错误处理 |
|--------|------|----------|
| 帧类型 | 必须是预定义枚举值 | 丢弃并记录 |
| 时间戳 | 必须在合理范围内 | 使用本地时间替代 |
| 序列号 | 必须单调递增 | 发出乱序警告 |
| 数据完整性 | HMAC-SHA256验证 | 拒绝处理 |

### 7.2 资源限制

| 资源 | 限制 | 超限处理 |
|------|------|----------|
| 内存使用 | <= 150MB | 触发GC，释放旧数据 |
| Canvas节点数 | <= 80个 | 自适应降级 |
| 消息队列长度 | <= 100条 | 丢弃最旧消息 |
| 重连间隔 | >= 3秒 | 指数退避 |

### 7.3 审计日志

| 事件 | 记录内容 |
|------|----------|
| 连接建立 | 时间戳、矢量ID、客户端信息 |
| 数据接收 | 帧类型、序列号、数据大小 |
| 命令发送 | 命令类型、参数、响应状态 |
| 错误发生 | 错误类型、堆栈、上下文 |
| 连接断开 | 时间戳、原因、持续时间 |

---

## 八、实现状态追踪

### 8.1 模块实现状态

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 全局样式 | style.css | ✅ 完成 | instrument体系6类+辅助6类 |
| 根组件 | App.vue | ✅ 完成 | router-view |
| 路由 | router.ts | ✅ 完成 | 5个子路由+beforeEach标题 |
| 布局 | Layout.vue | ✅ 完成 | TopBar+Sidebar+Main+StatusBar |
| 基因背景 | GenomeBackground.vue | ✅ 完成 | 80节点Canvas动画 |
| 顶部栏 | TopBar.vue | ✅ 完成 | SHM DATA PLATFORM+时钟+LED |
| 侧边栏 | Sidebar.vue | ✅ 完成 | instrument-btn+用户面板 |
| 状态栏 | StatusBar.vue | ✅ 完成 | LED+CPU/MEM模拟 |
| 统计卡片 | StatCard.vue | ✅ 完成 | instrument-panel+modelLabel |
| 延迟仪表 | LatencyGauge.vue | ✅ 完成 | 21刻度+渐变弧+SVG滤镜 |
| 状态标签 | StatusBadge.vue | ✅ 完成 | border-2+LED |
| 画像页 | ProfileView.vue | ✅ 完成 | 个人信息+权限+统计+分布 |
| 总线页 | BusMonitorView.vue | ✅ 完成 | KPI+仪表盘+通道表 |
| 引擎页 | EngineManageView.vue | ✅ 完成 | 状态+规则表+统计+资源 |
| 风控页 | RiskCenterView.vue | ✅ 完成 | 告警摘要+列表+规则表 |
| 命令页 | CommandCenterView.vue | ✅ 完成 | 聊天界面+数据卡片 |
| 全局Store | appStore.ts | ✅ 完成 | 连接/移动端/告警/侧边栏 |
| 画像Store | profileStore.ts | ✅ 完成 | 模拟数据 |
| 总线Store | busStore.ts | ✅ 完成 | 8通道模拟数据 |
| 引擎Store | engineStore.ts | ✅ 完成 | 3规则+统计+资源 |
| 风控Store | riskStore.ts | ✅ 完成 | 告警+4规则 |
| 命令Store | commandStore.ts | ✅ 完成 | 消息+快捷键+响应生成 |
| 类型定义 | types/index.ts | ✅ 完成 | 15个接口/类型 |
| 格式化 | formatters.ts | ✅ 完成 | 7个格式化函数 |
| SHM集成 | composables/useSHMVector.ts | ❌ 未实现 | 待SHM矢量层就绪 |
| 图表主题 | composables/useChartTheme.ts | ❌ 未实现 | 待ECharts集成 |
| 移动端检测 | composables/useMobile.ts | ❌ 未实现 | 当前内联在Layout |

### 8.2 待实现功能

| 功能 | 优先级 | 依赖 | 说明 |
|------|--------|------|------|
| SHM矢量实时数据 | P0 | 矢量管理层 | 替换模拟数据 |
| ECharts图表 | P1 | useChartTheme | 吞吐量趋势/内存饼图/性能折线 |
| 活跃时段热力图 | P2 | ECharts | 7x24网格 |
| 迷你面积图 | P2 | Canvas/ECharts | 命令中心Chat内嵌 |
| 告警统计图表 | P2 | ECharts | 柱状图+饼图 |

---

## 九、已知问题与修复记录

### 9.1 已修复问题

| 问题 | 根因 | 修复 | 影响 |
|------|------|------|------|
| 侧边栏无宽度 | Tailwind v3无`w-50`间距值 | `w-50` → `w-[200px]` | 布局完全错乱 |
| 主内容区无左内边距 | Tailwind v3无`pl-50`间距值 | `pl-50` → `pl-[200px]` | 内容被侧边栏遮挡 |
| LED红色显示为金色 | `led.red`使用`#E8B84B`（实为金色） | 改为`#FF6B6B`/`#E53E3E` | 风控告警状态语义错误 |
| 命令中心高度溢出 | `calc(100vh - 200px)`计算偏大 | 改为`calc(100vh - 180px)` | 底部输入区被截断 |
| 组件样式重复 | StatCard/LatencyGauge scoped样式与全局重复 | 移除scoped样式块 | CSS体积+1.2KB |

### 9.2 已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 数据全部模拟 | 所有Store使用硬编码模拟数据 | SHM矢量集成后替换 |
| 无图表 | ECharts未集成，趋势图/饼图缺失 | v1.2引入ECharts |
| 无SHM连接 | useSHMVector未实现 | 等待矢量管理层 |
| CPU/MEM模拟 | StatusBar中CPU/MEM使用随机数 | 接入系统监控矢量 |
| 命令响应简单 | commandStore仅关键词匹配 | 接入AI命令解析 |

---

## 十、测试策略

### 10.1 测试覆盖

| 测试类型 | 覆盖范围 | 用例数 |
|----------|----------|--------|
| 单元测试 | 组件渲染、状态管理、工具函数 | 30+ |
| 集成测试 | SHM矢量连接、数据流、命令发送 | 15+ |
| 视觉测试 | 响应式布局、主题切换、动画效果 | 10+ |
| 性能测试 | 大数据量渲染、内存占用、FPS | 5+ |

### 10.2 测试命令

```bash
npm run test:unit
npm run test:integration
npm run test:visual
npm run test:performance
npm run test
```

---

## 十一、部署规范

### 11.1 构建流程

```bash
npm install
npm run dev
npm run build
# 产出目录: dist/
#   - index.html
#   - assets/*.js
#   - assets/*.css
```

### 11.2 Nginx配置

```nginx
server {
    listen 80;
    server_name admin-profile.local;

    location / {
        root /path/to/admin-profile-ui/dist;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 十二、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-14 | 初始版本，适配创世纪项目特点 |
| v1.1.0 | 2026-05-14 | 嵌入式仪器风格重构；修复Tailwind间距值bug；修正LED红色；更新类型契约与实现状态追踪 |

---

## 附录

### A. 参考文档

- [PROJECT_CONTEXT.md](../../PROJECT_CONTEXT.md) - 项目上下文
- [function-paradigm.md](../specs/constitution/function-paradigm.md) - 函数范式宪章
- [BP-0032-sysdisplay.md](./BP-0032-sysdisplay.md) - 系统状态显示消费者
- [BP-0034-spectrum-ui.md](./BP-0034-spectrum-ui.md) - 频谱仪UI消费者
- [BP-0054-tr-test-ui.md](./BP-0054-tr-test-ui.md) - TR测试工程师UI消费者

### B. 术语表

| 术语 | 说明 |
|------|------|
| SHM | Shared Memory，共享内存 |
| VEC-DISPLAY | 显示数据矢量通道 |
| VEC-CONTROL | 控制命令矢量通道 |
| P1 | 安全等级P1（内部工具） |
| Pinia | Vue官方状态管理库 |
| instrument-panel | 嵌入式仪器面板CSS类 |
| digital-readout | 数字读数CSS类（等宽+金色光晕） |
| instrument-btn | 仪器按钮CSS类（工业风格） |
| led | LED指示灯CSS类（radial-gradient） |
