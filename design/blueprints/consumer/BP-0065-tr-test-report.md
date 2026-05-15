# 微波TR测试报告系统消费者蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0065-TR-TEST-REPORT
> **版本**: v1.0.0
> **日期**: 2026-05-16
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0065-TR-TEST-REPORT |
| 函数名 | tr_test_report_init / tr_test_report_tick |
| 生产者名称 | TRTestReportConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | 模拟数据（Demo模式，微波TR测试数据，动态format字段） |
| 输出协议 | Canvas 2D渲染 + CSV/Markdown/HTML导出 |
| 安全等级 | P2 (内部测试数据平台) |
| 版本 | v1.0.0 |
| 前置蓝图 | BP-0064-DATAVIZ-REPORT |

---

## 二、设计意图

### 2.1 核心目标

构建专业化的微波TR（发射/接收）模块测试报告系统，面向射频工程师与质量保证团队。系统支持动态format字段（S11/RL/NF/S22/S12/S21/AMPM/EVM/IP3/Phase/ACPR/GainFlatness等），从数据自动发现测试指标与规格，提供多维度可视化（直方图+散点图+热力图+Format分布卡片）和Word风格中文测试报告，支持按UUT×Format×Channel×Frequency矩阵展示，异常数据高亮标记，以及多格式导出。

### 2.2 与BP-0064的关系

| 维度 | BP-0064 DataViz Report | BP-0065 TR Test Report |
|------|------------------------|------------------------|
| 定位 | 通用数据可视化与报告 | 微波TR专业测试报告 |
| Format种类 | 6种固定（SINAD/THD/THD+N/IMD/SNR/SFDR） | 动态发现（S11/RL/NF/S22等13+种） |
| 规格来源 | 标称值±公差硬编码 | specLibrary规格库+动态发现 |
| 可视化 | 直方图+散点图 | 直方图+散点图+热力图+Format卡片 |
| 报告 | Word风格中文矩阵表 | Word风格中文矩阵表+结论生成+签章区 |
| 导出 | CSV | CSV+Markdown+HTML+PDF(print) |
| 架构 | 复用 | 继承BP-0064架构，扩展TR域特性 |

### 2.3 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | 前端=消费者域，模拟数据=生产者域，Store=商场域 |
| 函数范式 | Vue 3 Composition API纯函数设计，状态外置Pinia |
| SHM矢量空间 | Pinia Store替代SHM总线，响应式数据流 |
| 产品与相 | 测试数据为产品（客观），图表样式/报告模板为相（主观） |
| 仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| 画廊集成 | Vite构建+base路径+静态映射，iframe嵌入画廊 |

### 2.4 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vue 3 + Pinia | 与画廊+BP-0064统一技术栈 | React + Redux | 生态一致性 |
| Canvas 2D | 零第三方依赖，含热力图 | ECharts | 无交互式图表tooltip |
| 动态format发现 | 不同UUT测试不同指标组合 | 硬编码枚举 | 需uniqueSorted()工具函数 |
| specLibrary规格库 | 13种format的专业规格定义 | 仅自动发现 | 需维护规格数据 |
| 热力图Canvas | Channel×Frequency状态可视化 | ECharts Heatmap | 需自实现色彩映射 |
| Word风格中文报告 | 工业测试报告标准格式 | Markdown风格 | 需白底CSS覆盖暗色主题 |
| 矩阵表+异常标记 | UUT×Format×Channel×Frequency | 扁平表格 | 宽表需overflow-x+sticky |
| 多格式导出 | CSV+Markdown+HTML+PDF | 仅CSV | 需Blob+print API |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 数据商场 | Pinia Store | useTRTestStore |
| 数据产品 | 微波TR测试数据记录 | TRTestData[] |
| 规格库 | specLibrary | SpecLibrary |
| 报告模板 | 报告配置 | ReportTemplate |
| 生成报告 | 多格式文件 | GeneratedReport |
| 数据可视化 | Canvas 2D图表 | DashboardView |
| 热力图 | Channel×Frequency状态 | HeatmapCanvas |
| Format卡片 | 指标分布概览 | FormatCardGrid |

### 3.2 单层架构（纯前端）

```
┌──────────────────────────────────────────────────────────────────────┐
│                      前端应用层（消费者域）                             │
│   Vue 3 SPA │ Canvas 2D可视化 │ 报告生成 │ 多格式导出 │ Pinia Store  │
│                                                                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│   │ dataGenerator│  │ trTestStore  │  │ 10 Vue Components        │  │
│   │ (TR模拟数据)  │→│ (状态+规格库) │→│ (渲染+交互+导出)          │  │
│   └──────────────┘  └──────────────┘  └──────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.3 四区域布局

```
┌──────────────────────────────────────────────────────────────────────┐
│  顶部导航栏 - TR TEST PRO / BP-0065 / DEMO / records / LED / UUT数  │
├────────┬──────────────────────────────────────┬──────────────────────┤
│ 左侧   │                                      │ 右侧控制面板          │
│ 导航   │      主内容区域                        │ - Demo控制            │
│ 菜单   │      - 数据可视化(DashboardView)      │ - 筛选条件(4维度)     │
│        │      - 报告生成(ReportView)           │ - 图表类型            │
│ 📊数据 │      - 历史报告(HistoryView)          │ - 直方图数量          │
│ 📈报告 │      - 数据管理(DataView)             │ - 统计摘要            │
│ 📋历史 │                                      │                      │
│ 💾管理 │                                      │                      │
├────────┴──────────────────────────────────────┴──────────────────────┤
│  底部状态栏 - DEMO/IDLE / records / Pass% / UUTs / BP-0065          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 数据模型

```typescript
interface TRTestData {
  id: string
  uut: string
  meastime: string
  format: string
  value: number
  setting_raw: string
  test_mode: string
  channel: string
  frequency: string
  tolerance: number
  nominal: number
  created_at: string
}

interface FormatSpec {
  title: string
  unit: string
  min: number | null
  max: number | null
  warn_delta: number
}

type SpecLibrary = Record<string, FormatSpec>

interface ReportSection {
  id: string
  title: string
  type: 'table' | 'chart' | 'chart_table' | 'conclusion'
  data: TRTestData[]
  columns: { field: string; header: string }[]
  chartConfig?: { type: string; xField: string; yField: string; title: string }
  exportable: boolean
}

interface ReportTemplate {
  id: string
  name: string
  description: string
  sections: ReportSection[]
}

interface GeneratedReport {
  id: string
  template_id: string
  report_name: string
  format: 'csv' | 'markdown' | 'html' | 'pdf'
  status: 'pending' | 'processing' | 'completed' | 'failed'
  created_at: string
  completed_at: string | null
  sections_count: number
  rows_count: number
}

interface StatResult {
  mean: number
  min: number
  max: number
  std_dev: number
  count: number
  pass_count: number
  warn_count: number
  fail_count: number
  pass_rate: number
}

type ViewMode = 'dashboard' | 'report' | 'history' | 'data'
type ChartType = 'histogram' | 'line' | 'scatter' | 'box'
type HistogramCount = 1 | 2 | 4
```

#### 4.1.2 规格库（specLibrary）

```typescript
const SPEC_LIBRARY: SpecLibrary = {
  "S11":  { title: "回波损耗",     unit: "dB",     min: null,    max: -10,  warn_delta: 2 },
  "S21":  { title: "增益",         unit: "dB",     min: 15,      max: null, warn_delta: 2 },
  "S12":  { title: "反向隔离",     unit: "dB",     min: null,    max: -20,  warn_delta: 3 },
  "S22":  { title: "输出回波",     unit: "dB",     min: null,    max: -10,  warn_delta: 2 },
  "NF":   { title: "噪声系数",     unit: "dB",     min: null,    max: 3.0,  warn_delta: 1 },
  "P1dB": { title: "1dB压缩点",    unit: "dBm",    min: 20,      max: null, warn_delta: 2 },
  "IP3":  { title: "三阶交调点",   unit: "dBm",    min: 30,      max: null, warn_delta: 3 },
  "Phase":{ title: "相位偏差",     unit: "deg",    min: -5,      max: 5,    warn_delta: 2 },
  "AMPM": { title: "AM-PM转换",    unit: "deg/dB", min: null,    max: 5,    warn_delta: 1 },
  "EVM":  { title: "误差矢量幅度", unit: "%",      min: null,    max: 5,    warn_delta: 1 },
  "ACPR": { title: "邻道功率比",   unit: "dBc",    min: 40,      max: null, warn_delta: 3 },
  "RL":   { title: "驻波比",       unit: "",       min: null,    max: 2.0,  warn_delta: 0.3 },
  "GainFlatness": { title: "增益平坦度", unit: "dB", min: null,  max: 2,    warn_delta: 0.5 }
}
```

#### 4.1.3 动态规格发现

```typescript
function discoverSpecs(records: TRTestData[]): SpecLibrary {
  const formatGroups = groupBy(records, 'format')
  const specs: SpecLibrary = {}
  for (const [fmt, recs] of Object.entries(formatGroups)) {
    if (SPEC_LIBRARY[fmt]) {
      specs[fmt] = SPEC_LIBRARY[fmt]
    } else {
      const values = recs.map(r => r.value).sort((a, b) => a - b)
      specs[fmt] = {
        title: fmt,
        unit: '',
        min: null,
        max: null,
        warn_delta: Math.abs(values[values.length - 1] - values[0]) * 0.05
      }
    }
  }
  return specs
}
```

#### 4.1.4 模拟数据生成参数

```typescript
interface TRDataGenConfig {
  UUT_LIST: string[]
  FORMAT_LIST: string[]
  CHANNEL_LIST: string[]
  FREQ_LIST: string[]
  MODE_LIST: string[]
  NOMINALS: Record<string, { nominal: number; tolerance: number; unit: string }>
}
```

#### 4.1.5 测试数据规模

| UUT | 测试Format种类 | 记录数 | 测试时间 |
|-----|---------------|--------|---------|
| UUT-A001 | S11, RL, NF | 144 | 05-10 |
| UUT-A002 | S22, S12, S21, AMPM | 192 | 05-11 |
| UUT-A003 | EVM, IP3, S11 | 144 | 05-12 |
| UUT-A004 | S21, S22, RL | 144 | 05-13 |
| UUT-A005 | AMPM, S22, ACPR | 144 | 05-14 |
| UUT-A006 | S22, Phase, EVM, NF, S11, S12 | 288 | 05-15 |

总记录数：1,056条（6个UUT × 各3~6种Format × 6频点 × 8通道）

### 4.2 输出契约

```typescript
interface CSVOutput {
  filename: string
  mime_type: 'text/csv;charset=utf-8'
  content: Blob
  encoding: 'UTF-8 with BOM'
}

interface MarkdownOutput {
  filename: string
  mime_type: 'text/markdown;charset=utf-8'
  content: Blob
}

interface HTMLOutput {
  filename: string
  mime_type: 'text/html;charset=utf-8'
  content: Blob
}

interface PDFOutput {
  method: 'window.print'
  style: '@page { size: A4 landscape; margin: 15mm; }'
}

interface ReportPreview {
  sections: ReportSection[]
  stats: StatResult
  formatStats: Record<string, StatResult>
  channelStats: Record<string, StatResult>
  conclusion: string
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | IDLE | 初始化Store+空数据+加载specLibrary |
| IDLE | 点击Demo | DEMO | 生成模拟数据，isDemo=true |
| DEMO | 点击Stop | IDLE | isDemo=false，数据保留 |
| DEMO | 修改筛选条件 | DEMO | filteredData自动更新 |
| DEMO | 切换图表类型 | DEMO | Canvas重绘 |
| DEMO | 切换直方图数量 | DEMO | 网格布局重排 |
| DEMO | 生成报告 | DEMO | 添加GeneratedReport+导出 |
| DEMO | 切换视图模式 | DEMO | viewMode变更 |

### 4.4 状态判定逻辑

```typescript
function getStatus(value: number, spec: FormatSpec): 'pass' | 'warn' | 'fail' {
  if (spec.min !== null && value < spec.min - spec.warn_delta) return 'fail'
  if (spec.max !== null && value > spec.max + spec.warn_delta) return 'fail'
  if (spec.min !== null && value < spec.min) return 'warn'
  if (spec.max !== null && value > spec.max) return 'warn'
  return 'pass'
}
```

---

## 五、UI设计规范

### 5.1 色彩体系（创世纪仪器风格）

| 角色 | Tailwind类 | 色值 | 用途 |
|------|-----------|------|------|
| 深空黑 | space-black | #0A0E17 | 全局背景 |
| 夜灰 | night-gray | #141B2D | 面板/导航背景 |
| 炭灰 | charcoal | #1E2A3A | 边框/分隔线 |
| 皇家金 | royal-gold | #D4A843 | 标题/标签 |
| 翡翠绿 | emerald | #00C9A7 | LED/数字读数 |
| 天蓝 | sky-blue | #3B82F6 | 折线图/选中态 |
| 月白 | moon-white | #E8ECF1 | 正文文字 |
| 银灰 | silver-gray | #8892A4 | 次要文字 |
| 雾灰 | mist-gray | #3A4556 | 边框/控件边框 |
| 通过 | pass-green | #10B981 | Pass状态 |
| 警告 | warn-amber | #F59E0B | Warn状态 |
| 失败 | fail-red | #EF4444 | Fail状态 |

### 5.1.1 热力图色彩映射

| 状态 | 色值 | 说明 |
|------|------|------|
| 通过 | #0d3b2e | 极暗灰绿 |
| 警告 | #3d3418 | 暗琥珀 |
| 失败 | #3d1520 | 暗深红 |
| 无数据 | #1a2332 | 深空灰 |

### 5.1.2 Format颜色映射

| 格式 | 颜色 | 色值 |
|------|------|------|
| S11 | 蓝 | #3B82F6 |
| S21 | 绿 | #10B981 |
| S12 | 黄 | #F59E0B |
| S22 | 红 | #EF4444 |
| NF | 紫 | #8B5CF6 |
| AMPM | 粉 | #EC4899 |
| EVM | 青 | #06B6D4 |
| IP3 | 橙 | #F97316 |
| Phase | 靛 | #6366F1 |
| ACPR | 玫红 | #E11D48 |
| RL | 黄绿 | #84CC16 |
| GainFlatness | 棕 | #A16207 |

动态format超出12色时循环复用。

### 5.2 页面布局规范

| 区域 | 宽度 | 高度 | 组件 |
|------|------|------|------|
| 顶部导航 | 100% | 48px | TopBar |
| 左侧导航 | 64px | 100% | NavMenu |
| 主内容区 | flex-1 | 100% | Dashboard/Report/History/Data |
| 右侧控制面板 | 224px | 100% | ControlPanel |
| 底部状态栏 | 100% | 28px | StatusBar |

### 5.3 BootLogo（开机画面）

页面加载时全屏覆盖显示，持续约2秒，完成后自动淡出并触发Demo数据生成。

| 元素 | 说明 |
|------|------|
| 双环旋转 | 外层金色环(4s正向) + 内层翡翠绿环(2.5s反向)，border-radius: 50% |
| 中心TR图标 | 金色字母"TR"，JetBrains Mono字体，18px bold |
| 标题 | "微波TR测试报告系统"，letter-spacing: 6px，18px bold |
| 副标题 | "TR Test Report System"，11px，silver-gray |
| 版本号 | "BP-0065 v1.0.0"，9px，mist-gray |
| 进度条 | 200px宽，2px高，金色填充，0.3s ease过渡 |
| 状态文本 | 6步渐进：Loading spec library → Initializing data channels → Preparing test matrices → Calibrating instruments → Rendering dashboard → Ready |
| 淡出动画 | opacity 0.6s ease + transform scale(1.05) |

### 5.4 DashboardView

#### 5.4.1 滚动策略

DashboardView采用**自然高度+外部滚动**策略，而非填满容器+内部压缩：

| 策略 | 实现 | 原因 |
|------|------|------|
| 根容器 | `flex flex-col gap-2`（无h-full） | 内容自然撑开高度 |
| main区域 | `overflow-auto` | 总高度超视口时整体纵向滚动 |
| 图表行 | `flex gap-2`（无flex-1/min-h-0） | 自然高度，不压缩 |
| 直方图列 | `min-height: 380px` | 保证最小可视高度 |
| 散点图列 | `min-height: 380px` | 保证最小可视高度 |
| 单图窗口 | `min-height: 360px` | 4图模式下每格最小180px |
| 双/四图窗口 | `min-height: 180px` | 保证Canvas有足够绘制空间 |
| 热力图 | `height: 200px` | 固定高度，不伸缩 |
| 统计表格 | `max-height: 280px` + `overflow-auto` | 单表内容过多时内部滚动 |
| 控制面板 | `overflow-y-auto` | 面板内容过多时内部滚动 |

#### 5.4.2 Format分布卡片网格

4-6列响应式网格，每张卡片代表一个动态发现的Format。

**卡片内部结构**：
- 顶部：极小灰色英文标签（如 `RETURN LOSS`）+ 大号格式代号（如 `S11`）
- 中部：Canvas sparkline折线图（40px高），纯线条无背景，颜色映射状态
- 数据区：大字显示最新值（如 `-19.8 dB`）+ 趋势箭头
- 底部：三个LED状态计数器（🔴异常 🟡警告 🟢通过）

#### 5.3.2 两列布局

| 属性 | 值 |
|------|-----|
| 布局 | 左列=直方图区，右列=散点图区 |
| 左列宽度 | 60% |
| 右列宽度 | 40% |

#### 5.3.3 直方图区（左列）

| 属性 | 值 |
|------|-----|
| 数量可选 | 1(单)/2(双)/4(四)，Store.histogramCount |
| 1个布局 | 1×1全高 |
| 2个布局 | 2×1纵向 |
| 4个布局 | 2×2网格 |
| 格式选择 | 每窗口左上角格式下拉选择器 |
| 图表类型 | 每窗口右上角histogram/line/scatter切换按钮 |
| 窗口标题 | 格式名·记录数·合格率% |

#### 5.3.4 散点图区（右列）

| 属性 | 值 |
|------|-----|
| 数量 | 2个，纵向排列 |
| 散点图A | Y轴=通道，X轴=测量值，频率多选，N格式N色 |
| 散点图B | Y轴=频率，X轴=测量值，通道多选，N格式N色 |

#### 5.3.5 热力图

Canvas 2D渲染，X轴=频率，Y轴=通道，当前选中format的状态分布。

| 属性 | 值 |
|------|-----|
| 数据源 | 当前format的filteredData |
| X轴 | 频率（动态提取） |
| Y轴 | 通道（动态提取） |
| 色彩映射 | pass→#0d3b2e / warn→#3d3418 / fail→#3d1520 / 无数据→#1a2332 |
| 交互 | 点击单元格→高亮该通道+频率组合 |

#### 5.3.6 窗口结构

```
┌──────────────────────────────────────────────────────────────────┐
│  Format分布卡片网格（4-6列响应式）                                  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │
│  │ S11  │ │ S21  │ │ NF   │ │ S22  │ │ AMPM │ │ EVM  │         │
│  │spark │ │spark │ │spark │ │spark │ │spark │ │spark │         │
│  │-19.8 │ │ 22.1 │ │ 2.3  │ │-15.2 │ │ 2.8  │ │ 3.1  │         │
│  │🟢🟡🔴│ │🟢🟡🔴│ │🟢🟡🔴│ │🟢🟡🔴│ │🟢🟡🔴│ │🟢🟡🔴│         │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘         │
├──────────────────────────────────────────────────────────────────┤
│  直方图区（左列60%）          │  散点图区（右列40%）                │
│  ┌─────────────────────────┐ │  ┌─────────────────────────────┐ │
│  │ [S11▼] · 📊📈📉        │ │  │ 散点图A: 通道×测量值          │ │
│  │   Canvas 直方图          │ │  │ 频率:☑1G☑2G☑3G...          │ │
│  │                         │ │  │ Y=CH1-CH8  X=测量值          │ │
│  └─────────────────────────┘ │  └─────────────────────────────┘ │
│  ┌─────────────────────────┐ │  ┌─────────────────────────────┐ │
│  │ [S21▼] · 📊📈📉        │ │  │ 散点图B: 频率×测量值          │ │
│  │   Canvas 直方图          │ │  │ 通道:☑CH1☑CH2...            │ │
│  │                         │ │  │ Y=1G-6G  X=测量值            │ │
│  └─────────────────────────┘ │  └─────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│  热力图（Channel×Frequency，当前format状态分布）                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │     1.0G  2.0G  3.0G  4.0G  5.0G  6.0G                    │  │
│  │ CH1  🟢    🟢    🟡    🟢    🟢    🟢                      │  │
│  │ CH2  🟢    🟢    🟢    🟢    🟡    🟢                      │  │
│  │ ...                                                        │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### 5.3.7 统计表

| 表格 | 列 |
|------|-----|
| 格式统计表 | 格式×中文名×数量×均值×标准差×合格率 |
| 通道统计表 | 通道×数量×均值×最小×最大×合格率 |

### 5.5 ReportView

#### 5.5.1 报告配置

| 区域 | 内容 |
|------|------|
| 报告名称 | input，默认"微波TR模块测试报告 + 日期" |
| 时间区间 | datetime-local start / datetime-local end |
| UUT多选 | checkbox列表，动态从数据提取 |
| 导出格式 | CSV / Markdown / HTML / PDF(print) |
| Generate按钮 | 生成报告+导出 |

#### 5.5.2 Word风格中文报告

| 属性 | 值 |
|------|-----|
| 背景 | 白色(#FFFFFF) |
| 文字 | 深色(#1a1a1a) |
| 报告大标题 | "微 波 T R 模 块 测 试 报 告"，居中，letter-spacing: 8px |
| 章节标题 | 深蓝(#1e40af)，加粗，14px |
| 表头背景 | 浅蓝(#F0F9FF) |
| 表头文字 | 深蓝(#1e40af)，加粗 |
| 表格边框 | 灰色(#E5E7EB) |
| 交替行 | 白色/浅灰(#F9FAFB) |
| 状态色 | 合格绿(#10B981)/警告黄(#F59E0B)/超差红(#EF4444) |
| 异常标记 | warn=浅黄背景(#FEF3C7)，fail=红色背景(#EF4444)+白色文字 |

#### 5.5.3 报告章节（中文）

| 章节 | 标题 | 内容 |
|------|------|------|
| 封面 | 微波TR模块测试报告 | 大标题+报告编号+生成时间+测试批次+时间范围 |
| 一 | 测试概览 | 总记录数/均值/标准差/最小最大值/合格率/全局统计卡片 |
| 二~N | 按UUT分节 | 每个UUT一个章节，章节标题="被测单元：UUT-A001" |
| 二~N.1 | 按Format分表 | 每个Format一张矩阵表 |
| 二~N.1表 | Channel×Frequency矩阵 | 行=通道，列=频率，单元格=测量值+状态色 |
| 末章 | 测试结论 | 自动生成结论文本+签章区 |

#### 5.5.4 矩阵表结构

```
被测单元：UUT-A001
  S11（回波损耗）  合格范围：≤ -10 dB
  ┌──────┬───────┬───────┬───────┬───────┬───────┬───────┐
  │ 通道  │ 1.0G  │ 2.0G  │ 3.0G  │ 4.0G  │ 5.0G  │ 6.0G  │
  ├──────┼───────┼───────┼───────┼───────┼───────┼───────┤
  │ CH1  │-19.8✓ │-18.2✓ │-11.5⚠ │-15.6✓ │-17.3✓ │-20.1✓ │
  │ CH2  │-16.4✓ │-14.7✓ │-13.2⚠ │-18.9✓ │-15.1✓ │-12.8⚠ │
  │ CH3  │-21.3✓ │-19.6✓ │-16.8✓ │-14.5⚠ │-18.7✓ │-17.2✓ │
  │ CH4  │-15.9✓ │-13.1⚠ │-9.8✗  │-17.4✓ │-16.2✓ │-14.8✓ │
  │ CH5  │-18.7✓ │-16.5✓ │-15.3✓ │-19.1✓ │-13.7⚠ │-17.8✓ │
  │ CH6  │-20.5✓ │-18.3✓ │-17.1✓ │-15.8✓ │-19.4✓ │-16.6✓ │
  │ CH7  │-14.2⚠ │-17.9✓ │-16.4✓ │-18.6✓ │-21.2✓ │-15.5✓ │
  │ CH8  │-19.1✓ │-15.7✓ │-13.9⚠ │-17.5✓ │-18.8✓ │-20.3✓ │
  └──────┴───────┴───────┴───────┴───────┴───────┴───────┘
                                              [导出CSV]
```

#### 5.5.5 合格范围显示

每张Format矩阵表标题居中，下方显示合格范围：

```
S11（回波损耗）  合格范围：≤ -10 dB
S21（增益）      合格范围：≥ 15 dB
NF（噪声系数）   合格范围：≤ 3.0 dB
Phase（相位偏差） 合格范围：-5 ~ +5 deg
```

规则：
- 仅有min：`≥ {min} {unit}`
- 仅有max：`≤ {max} {unit}`
- 双限：`{min} ~ {max} {unit}`
- 无限：`无限制`

#### 5.5.6 宽表溢出处理

| 属性 | 值 |
|------|-----|
| 容器 | overflow-x: auto, border: 1px solid #E5E7EB |
| 首列 | position: sticky, left: 0, z-index: 1, background: #F0F9FF |
| 最小宽度 | min-width: max-content |

#### 5.5.7 测试结论生成

```typescript
function generateConclusion(
  filteredRecords: TRTestData[],
  specs: SpecLibrary
): string {
  const total = filteredRecords.length
  const failCount = countByStatus(filteredRecords, specs, 'fail')
  const warnCount = countByStatus(filteredRecords, specs, 'warn')
  const passCount = total - failCount - warnCount
  const failRate = (failCount / total * 100).toFixed(1)
  const failFormats = getFormatsByStatus(filteredRecords, specs, 'fail')
  const warnFormats = getFormatsByStatus(filteredRecords, specs, 'warn')

  if (failCount === 0 && warnCount === 0) {
    return `本次测试共涉及${uniqueSorted(filteredRecords, 'uut').length}个被测单元，` +
           `${total}项测试数据全部通过，判定为合格。`
  } else if (failCount === 0) {
    return `本次测试共${total}项测试数据，其中通过${passCount}项，` +
           `警告${warnCount}项。警告项集中在${warnFormats.join('、')}指标，建议关注但可放行。`
  } else {
    return `本次测试共${total}项测试数据，其中通过${passCount}项，` +
           `警告${warnCount}项，失败${failCount}项（失败率${failRate}%）。` +
           `失败项主要集中在${failFormats.join('、')}指标。建议返修或报废处理。`
  }
}
```

#### 5.5.8 签章区

```
测试工程师：_______________    审核：_______________    日期：_______________
```

#### 5.5.9 中文标签映射

| 英文 | 中文 |
|------|------|
| Test Report | 测试报告 |
| Total Records | 总记录数 |
| Mean | 均值 |
| Std Deviation | 标准差 |
| Min / Max | 最小值 / 最大值 |
| Pass Rate | 合格率 |
| Format | 格式 |
| Channel | 通道 |
| Frequency | 频率 |
| Count | 数量 |
| PASS | 合格 |
| WARN | 警告 |
| FAIL | 超差 |
| Nominal | 标称值 |
| Tolerance | 公差 |
| UUT | 被测单元 |
| Status | 状态 |
| Conclusion | 测试结论 |
| Qualified Range | 合格范围 |
| Export CSV | 导出CSV |

### 5.6 导出规范

#### 5.6.1 CSV导出

| 属性 | 值 |
|------|-----|
| 编码 | UTF-8 with BOM (\uFEFF) |
| 分隔符 | 逗号(,) |
| 字段引用 | 双引号(") |
| 文件名 | 报告名称.csv / all_data.csv / filtered_data.csv |
| 每表独立导出 | 每张Format矩阵表右上角"导出CSV"按钮 |

#### 5.6.2 Markdown导出

| 属性 | 值 |
|------|-----|
| 编码 | UTF-8 |
| 格式 | 标准Markdown表格语法 |
| 状态符号 | ✓(通过) ⚠(警告) ✗(失败) |
| 文件名 | 报告名称.md |

#### 5.6.3 HTML导出

| 属性 | 值 |
|------|-----|
| 编码 | UTF-8 |
| 格式 | 自包含HTML，内联CSS，浏览器直接打开 |
| 样式 | Word风格白底报告样式 |
| 文件名 | 报告名称.html |

#### 5.6.4 PDF导出

| 属性 | 值 |
|------|-----|
| 方式 | window.print() + print-optimized CSS |
| 页面 | @page { size: A4 landscape; margin: 15mm; } |
| 字体 | SimSun / Microsoft YaHei |
| 状态色 | .pass { background: #E8F5E9; } / .warn { background: #FFF8E1; } / .fail { background: #FFEBEE; } |

---

## 六、技术架构

### 6.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | Vue 3 + Composition API | SPA组件化开发 |
| 状态管理 | Pinia | 响应式Store，computed自动派生 |
| 图表渲染 | Canvas 2D | 零第三方依赖，histogram/line/scatter/heatmap/sparkline |
| 样式系统 | Tailwind CSS v3 | 创世纪仪器色彩体系 |
| 构建工具 | Vite | base: '/tr-test-report-ui/' |
| 模拟数据 | dataGenerator.ts | 微波TR测试数据，动态format+specLibrary |
| CSV导出 | Blob + URL.createObjectURL | 前端纯下载 |
| Markdown导出 | Blob | 前端纯下载 |
| HTML导出 | Blob | 自包含HTML |
| PDF导出 | window.print() | 浏览器打印API |

### 6.2 项目目录结构

```
tr-test-report-ui/
├── public/
│   ├── favicon.svg
│   └── icons.svg
├── src/
│   ├── App.vue
│   ├── main.ts
│   ├── style.css
│   ├── components/
│   │   ├── TopBar.vue
│   │   ├── NavMenu.vue
│   │   ├── StatusBar.vue
│   │   ├── ControlPanel.vue
│   │   ├── DashboardView.vue
│   │   ├── FormatCardGrid.vue
│   │   ├── HeatmapCanvas.vue
│   │   ├── ReportView.vue
│   │   ├── HistoryView.vue
│   │   └── DataView.vue
│   ├── stores/
│   │   └── trTestStore.ts
│   ├── types/
│   │   └── index.ts
│   └── utils/
│       ├── dataGenerator.ts
│       ├── specLibrary.ts
│       └── conclusionGenerator.ts
├── tailwind.config.js
├── vite.config.ts
└── package.json
```

### 6.3 数据流

```
dataGenerator.ts              trTestStore.ts                      Vue Components
┌──────────────┐             ┌─────────────────────┐             ┌──────────────┐
│generateTestData│─TRTestData[]→│ allData: ref          │             │ DashboardView │
│discoverSpecs   │             │ filteredData: comp    │──computed──→│ FormatCardGrid│
│computeStats    │             │ stats: comp           │──computed──→│ HeatmapCanvas │
│filterData      │             │ formatStats: comp     │──computed──→│ ReportView    │
│getStatus       │             │ channelStats: comp    │──computed──→│ HistoryView   │
│exportCSV       │             │ discoveredSpecs: comp │──computed──→│ DataView      │
│exportMarkdown  │             │ activeFormats: comp   │──computed──→│ ControlPanel  │
│exportHTML      │             │ activeChannels: comp  │──computed──→│ TopBar        │
│gaussRandom     │             │ activeFreqs: comp     │──computed──→│ StatusBar     │
└──────────────┘             │ activeUuts: comp      │             └──────────────┘
                              │ reportFormats: comp   │
specLibrary.ts               │ reportChannels: comp  │
┌──────────────┐             │ reportFreqs: comp     │
│SPEC_LIBRARY   │──SpecLibrary→│ reportUuts: comp      │
│discoverSpecs  │             │ uutGroupedData: comp  │
└──────────────┘             │ histogramCount: ref   │
                              │ chartTypes: reactive  │
conclusionGenerator.ts       │ scatterAFreqs: ref    │
┌──────────────────────┐     │ scatterBChannels: ref │
│generateConclusion     │←───│ reportTimeStart: ref  │
│countByStatus          │     │ reportTimeEnd: ref    │
│getFormatsByStatus     │     │ reportUutFilter: ref  │
└──────────────────────┘     └─────────────────────┘
```

### 6.4 模拟数据生成算法

```
generateTRTestData(uutConfigs):
  baseDate = 2026-05-10T08:00:00
  for each uutConfig in uutConfigs:
    for format in uutConfig.formats:
      spec = SPEC_LIBRARY[format] or discoverSpec
      for channel in CHANNEL_LIST:
        for freq in FREQ_LIST:
          noise = gaussRandom() * spec.warn_delta * 0.4
          value = computeNominal(spec) + noise
          meastime = baseDate + offset
          → TRTestData{ id, uut, meastime, format, value, setting_raw,
                        test_mode, channel, frequency, tolerance, nominal, created_at }

computeNominal(spec):
  if spec.min !== null and spec.max !== null:
    return (spec.min + spec.max) / 2
  if spec.min !== null:
    return spec.min + spec.warn_delta * 2
  if spec.max !== null:
    return spec.max - spec.warn_delta * 2
  return 0

gaussRandom():
  u, v = uniform(0,1)
  return sqrt(-2*ln(u)) * cos(2*π*v)

computeStats(data, specs):
  values = data.map(d => d.value)
  mean = avg(values)
  min, max = minmax(values)
  std_dev = sqrt(variance(values))
  pass_count = count(getStatus(v, spec) === 'pass')
  warn_count = count(getStatus(v, spec) === 'warn')
  fail_count = count(getStatus(v, spec) === 'fail')
  → StatResult{ mean, min, max, std_dev, count, pass_count, warn_count, fail_count, pass_rate }
```

### 6.5 热力图渲染算法

```
renderHeatmap(ctx, data, width, height):
  channels = uniqueSorted(data, 'channel')
  freqs = uniqueSorted(data, 'frequency')
  cellW = width / freqs.length
  cellH = height / channels.length

  for ch in channels:
    for freq in freqs:
      records = data.filter(d => d.channel === ch && d.frequency === freq)
      status = worstStatus(records, currentSpec)
      color = HEATMAP_COLORS[status]
      ctx.fillStyle = color
      ctx.fillRect(freqIdx * cellW, chIdx * cellH, cellW, cellH)

  renderAxisLabels(ctx, channels, freqs, cellW, cellH)
```

---

## 七、组件规范

### 7.1 TopBar

**职责**: 顶部导航栏

**内容**: TR TEST PRO / BP-0065 / DEMO标签 / records计数 / UUT数 / LED指示灯

**样式**: h-12, bg-night-gray, border-b, flex, px-4

### 7.2 NavMenu

**职责**: 左侧4图标导航

**菜单项**: 📊数据(dashboard) / 📈报告(report) / 📋历史(history) / 💾管理(data)

**样式**: w-16, bg-sidebar-bg, flex-col, items-center, gap-1

### 7.3 DashboardView

**职责**: 数据可视化主视图

**子组件**: FormatCardGrid + 直方图区 + 散点图区 + HeatmapCanvas + 统计表

**图表类型**: histogram(30bins, pass/warn/fail色) / scatter(点图, 状态色) / line(排序折线)

**技术**: Canvas 2D, devicePixelRatio适配, resize监听

### 7.4 FormatCardGrid

**职责**: Format分布卡片网格

**内容**: 动态发现的Format卡片，sparkline+最新值+状态计数

**布局**: 4-6列响应式网格

**技术**: Canvas sparkline(40px高)

### 7.5 HeatmapCanvas

**职责**: Channel×Frequency热力图

**内容**: 当前选中format的通道×频率状态分布

**色彩**: pass=#0d3b2e / warn=#3d3418 / fail=#3d1520 / nodata=#1a2332

**交互**: 点击单元格→高亮该组合

### 7.6 ReportView

**职责**: 报告生成+预览+导出

**配置**: 报告名称(input) + 时间区间(datetime-local×2) + UUT多选(checkbox) + 导出格式(select) + Generate按钮

**报告章节**: 封面 / 测试概览 / UUT分节(Format矩阵表) / 测试结论 / 签章区

**导出**: CSV(每表独立) / Markdown / HTML / PDF(print)

### 7.7 ControlPanel

**职责**: 右侧控制面板

**5区域**: Demo控制(▶/■+记录数) / Filter(4维度select) / Chart类型 / 直方图数量(1/2/4) / Stats摘要

**筛选维度**: Format(动态) / Channel(动态) / Frequency(动态) / UUT(动态)

### 7.8 HistoryView

**职责**: 报告历史列表

**列**: ID / Name / Format / Records / Status / Created

### 7.9 DataView

**职责**: 数据管理

**功能**: 全量数据表(前200条) / Export CSV(全部) / Export Filtered(筛选)

### 7.10 StatusBar

**职责**: 底部状态栏

**内容**: LED + DEMO/IDLE + records + Pass% + UUTs + BP-0065

---

## 八、安全约束

| 约束 | 实现 |
|------|------|
| 无后端通信 | 纯前端，零API调用 |
| 无敏感数据 | 模拟测试数据，不含个人隐私 |
| CSV安全 | Blob URL下载后立即revokeObjectURL |
| XSS防护 | Vue模板自动转义 |
| 数据隔离 | 浏览器内存，页面关闭即清除 |
| 规格库安全 | specLibrary内置于前端，不含生产密钥 |
| PDF安全 | window.print()不涉及外部服务 |

---

## 九、实现状态追踪

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 类型定义 | types/index.ts | ✅ | TRTestData+FormatSpec+SpecLibrary+ReportSection+GeneratedReport+StatResult+FormatCardData+HeatmapCell+ConclusionData |
| 规格库 | utils/specLibrary.ts | ✅ | 13种format规格+getSpec()+formatFrequency()+formatThreshold()+getStatus() |
| 数据生成器 | utils/dataGenerator.ts | ✅ | 6UUT动态format+高斯噪声(Box-Muller)+统计+筛选+CSV导出 |
| 结论生成器 | utils/conclusionGenerator.ts | ✅ | generateConclusion()+countByStatus()+getFormatsByStatus() |
| Pinia Store | stores/trTestStore.ts | ✅ | 13状态+16计算属性+10动作+formatCards+heatmapData |
| 顶部导航 | TopBar.vue | ✅ | TR TEST PRO + BP-0065 + DEMO标签 + records + UUT数 + LED |
| 左侧导航 | NavMenu.vue | ✅ | 4图标导航(仪表/报告/历史/数据) + active态 |
| 底部状态栏 | StatusBar.vue | ✅ | DEMO/IDLE + records + Pass% + UUTs + BP-0065 |
| 控制面板 | ControlPanel.vue | ✅ | Demo控制+4维度动态筛选+Chart类型+直方图数量+Stats |
| Format卡片 | FormatCardGrid.vue | ✅ | 动态Format卡片网格(2-6列响应式)+Canvas sparkline+状态计数 |
| 热力图 | HeatmapCanvas.vue | ✅ | Channel×Frequency热力图+Canvas 2D色彩映射+ResizeObserver |
| 数据可视化 | DashboardView.vue | ✅ | 两列布局(60/40)+直方图(1/2/4可选)+散点图(Y轴主轴)+热力图+统计表+自然高度滚动策略 |
| 报告生成 | ReportView.vue | ✅ | Word风格中文报告+UUT分节+Format矩阵表+合格范围+异常标记+结论生成+签章区+4格式导出 |
| 报告历史 | HistoryView.vue | ✅ | 历史列表 |
| 数据管理 | DataView.vue | ✅ | 数据表(前200条)+CSV导出(全部/筛选) |
| 开机Logo | BootLogo.vue | ✅ | 双环旋转动画+TR图标+进度条+6步状态+淡出动画 |
| 样式系统 | style.css | ✅ | Tailwind + instrument体系 + LED + data-table + status颜色 |
| 构建配置 | vite.config.ts | ✅ | base: '/tr-test-report-ui/' |
| 画廊集成 | gallery.json | ✅ | exhibit-tr-test-report (cat-instrument, sort_order: 5) |
| 画廊映射 | ui-gallery/vite.config.ts | ✅ | '/tr-test-report-ui/' → tr-test-report-ui/dist |

---

## 十、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 无持久化 | 数据仅存内存，刷新丢失 | v2.0接入后端API |
| 热力图无tooltip | Canvas 2D静态渲染 | v1.1添加hover tooltip |
| 规格库硬编码 | 13种format规格内置于前端 | v2.0从后端API加载 |
| PDF无页眉页脚 | window.print()限制 | v1.2添加CSS @page页眉 |
| 无实时数据流 | 仅Demo模拟数据 | v1.1添加WebSocket |
| 图表无交互 | Canvas 2D静态渲染 | v1.2添加tooltip/hover |
| 无协作功能 | 单用户 | v2.0 |
| 数据量限制 | 前端内存限制，建议≤10000条 | v2.0分页+后端 |

---

## 十一、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-16 | CODE_DONE：基于微波TR测试报告UI设计蓝图v3.3改写，技术栈从React+ECharts+SheetJS迁移至Vue 3+Pinia+Canvas 2D，继承BP-0064架构（两列布局+直方图数量可选+散点图Y轴主轴+Word风格中文报告+矩阵表+异常标记），新增specLibrary规格库+动态format发现+热力图+Format卡片+结论生成+签章区+多格式导出(CSV/Markdown/HTML/PDF)。新增BootLogo开机动画（双环旋转+进度条+6步状态）。DashboardView采用自然高度+外部滚动策略，解决组件干涉问题。画廊已集成（ui-gallery/vite.config.ts静态映射+gallery.json exhibit条目） |

---

## 附录

### A. 参考文档

- [微波TR测试报告UI设计蓝图_v3_3.md](../../微波TR测试报告UI设计蓝图_v3_3.md) - 原始设计文档（React+ECharts+SheetJS）
- [BP-0064-dataviz-report.md](./BP-0064-dataviz-report.md) - 数据可视化与报告生成系统消费者蓝图（架构参考）
- [BP-0054-tr-test-ui.md](./BP-0054-tr-test-ui.md) - TR测试工程师UI消费者蓝图（TR域参考）
- [BP-0062-iq3d-fusion.md](./BP-0062-iq3d-fusion.md) - IQ3D Fusion消费者蓝图（Canvas可视化参考）
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图

### B. 术语表

| 术语 | 说明 |
|------|------|
| UUT | Unit Under Test，被测单元 |
| TR | Transmit/Receive，发射/接收 |
| meastime | Measurement Time，测量时间 |
| S11 | 输入回波损耗 |
| S21 | 正向增益 |
| S12 | 反向隔离 |
| S22 | 输出回波损耗 |
| NF | Noise Figure，噪声系数 |
| IP3 | Third-order Intercept Point，三阶交调点 |
| EVM | Error Vector Magnitude，误差矢量幅度 |
| ACPR | Adjacent Channel Power Ratio，邻道功率比 |
| AMPM | AM-PM Conversion，调幅调相转换 |
| RL | Return Loss / 驻波比 |
| GainFlatness | 增益平坦度 |
| specLibrary | 规格库，定义各format的合格范围 |
| warn_delta | 警告余量，超限但未超warn_delta为警告 |
| Box-Muller | 高斯随机数生成变换 |
| BOM | Byte Order Mark，UTF-8文件头标识 |

### C. v3_3→BP-0065迁移映射

| v3_3特性 | BP-0065适配 |
|----------|-------------|
| React + TypeScript | Vue 3 + Composition API + TypeScript |
| ECharts Heatmap | Canvas 2D HeatmapCanvas组件 |
| ECharts sparkline | Canvas 2D sparkline(40px) |
| SheetJS (xlsx) | 移除，仅CSV/Markdown/HTML/PDF |
| file-saver | Blob + URL.createObjectURL |
| shadcn/ui | Tailwind CSS + instrument体系 |
| Dark Mode极客美学 | 创世纪仪器风格(深空黑+皇家金+翡翠绿) |
| 终端启动动画 | 不纳入v1.0，v1.1可选 |
| 雷达扫描线 | 不纳入v1.0，v1.1可选 |
| 卡片Hover发光 | 不纳入v1.0，v1.1可选 |
| specLibrary | 完整保留，13种format规格 |
| 动态format发现 | 完整保留，discoverSpecs() |
| 热力图 | Canvas 2D实现，Channel×Frequency |
| Format分布卡片 | FormatCardGrid组件 |
| 结论生成 | conclusionGenerator.ts |
| 签章区 | ReportView内HTML占位 |
| 批量导出(PDF/XLSX/MD/HTML) | CSV+Markdown+HTML+PDF(print)，移除XLSX |

### D. 与BP-0064的组件复用

| BP-0064组件 | BP-0065复用 | 差异点 |
|-------------|-------------|--------|
| TopBar.vue | 复用 | 标题改为TR TEST PRO，增加UUT数 |
| NavMenu.vue | 复用 | 无差异 |
| StatusBar.vue | 复用 | 增加UUTs显示 |
| ControlPanel.vue | 复用 | 增加直方图数量选择，筛选维度动态化 |
| DashboardView.vue | 扩展 | 增加FormatCardGrid+HeatmapCanvas |
| ReportView.vue | 扩展 | 增加结论生成+签章区+多格式导出 |
| HistoryView.vue | 复用 | 无差异 |
| DataView.vue | 复用 | 增加Markdown/HTML导出 |
| datavizStore.ts | 扩展 | 增加specLibrary+discoveredSpecs+uutGroupedData |
| dataGenerator.ts | 扩展 | 6UUT动态format+specLibrary规格 |
