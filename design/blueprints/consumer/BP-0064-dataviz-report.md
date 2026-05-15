# 数据可视化与报告生成系统消费者蓝图

> **蓝图编号**: BP-0064-DATAVIZ-REPORT
> **版本**: v1.7.0
> **日期**: 2026-05-15
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0064-DATAVIZ-REPORT |
| 函数名 | dataviz_report_init / dataviz_report_tick |
| 生产者名称 | DataVizReportConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | 模拟数据（Demo模式，高斯噪声+标称值+公差） |
| 输出协议 | Canvas 2D渲染 + CSV导出（UTF-8 BOM） |
| 安全等级 | P2 (内部测试数据平台) |
| 版本 | v1.7.0 |

---

## 二、设计意图

### 2.1 核心目标

构建专业化的前端数据可视化与报告生成系统，采用模拟测试数据驱动，支持6种测试格式（SINAD/THD/THD+N/IMD/SNR/SFDR）的多维度可视化、报告模板化生成和CSV导出。提供从数据生成到报告交付的完整前端工作流，零后端依赖，专为测试工程师、数据分析师和质量保证团队设计。

### 2.2 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | 前端=消费者域，模拟数据=生产者域，Store=商场域 |
| 函数范式 | Vue 3 Composition API纯函数设计，状态外置Pinia |
| SHM矢量空间 | Pinia Store替代SHM总线，响应式数据流 |
| 产品与相 | 测试数据为产品（客观），图表样式/报告模板为相（主观） |
| 仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| 画廊集成 | Vite构建+base路径+静态映射，iframe嵌入画廊 |

### 2.3 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vue 3 + Pinia | 与画廊其他项目统一技术栈 | React + Redux | 生态一致性 |
| Canvas 2D | 零第三方依赖，轻量渲染 | Plotly.js / ECharts | 无交互式图表 |
| 模拟数据 | 零后端依赖，画廊即开即用 | PostgreSQL+Flask | 无持久化存储 |
| 高斯噪声 | 真实模拟测试数据分布 | 均匀随机 | 数据分布更真实 |
| CSV导出 | 前端Blob下载，UTF-8 BOM兼容Excel | PDF(需jsPDF) | 无PDF导出 |
| 四区域布局 | 导航+内容+控制面板+状态栏 | 单页切换 | 需CSS Grid布局 |
| Pinia Store | 响应式状态管理，computed自动派生 | Vuex | 更轻量 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 数据商场 | Pinia Store | useDataVizStore |
| 数据产品 | 模拟测试数据记录 | TestData[] |
| 报告模板 | 报告配置 | ReportTemplate |
| 生成报告 | CSV文件 | GeneratedReport |
| 数据可视化 | Canvas 2D图表 | DashboardView |
| 分项导出 | 选择性数据导出 | DataView ExportCSV |

### 3.2 单层架构（纯前端）

```
┌─────────────────────────────────────────────────────────────┐
│                    前端应用层（消费者域）                       │
│   Vue 3 SPA │ Canvas 2D可视化 │ 报告生成 │ CSV导出 │ Pinia  │
│                                                             │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│   │ dataGenerator│  │ datavizStore │  │ 8 Vue Components │  │
│   │ (模拟数据)   │→│ (状态管理)    │→│ (渲染+交互)       │  │
│   └─────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 四区域布局

```
┌─────────────────────────────────────────────────────────────┐
│  顶部导航栏 - DATAVIZ PRO / BP-0064 / DEMO / records / LED  │
├────────┬────────────────────────────────┬───────────────────┤
│ 左侧   │                                │ 右侧控制面板       │
│ 导航   │    主内容区域                    │ - Demo控制         │
│ 菜单   │    - 数据可视化(DashboardView)  │ - 筛选条件(4维度)  │
│        │    - 报告生成(ReportView)       │ - 图表类型         │
│ 📊数据 │    - 历史报告(HistoryView)      │ - 统计摘要         │
│ 📈报告 │    - 数据管理(DataView)         │                   │
│ 📋历史 │                                │                   │
│ 💾管理 │                                │                   │
├────────┴────────────────────────────────┴───────────────────┤
│  底部状态栏 - DEMO/IDLE / records / Pass% / Reports / BP-0064│
└─────────────────────────────────────────────────────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 数据模型

```typescript
interface TestData {
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

interface ReportSection {
  id: string
  title: string
  type: 'table' | 'chart' | 'chart_table'
  data: TestData[]
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
  format: 'csv' | 'pdf'
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
  fail_count: number
  pass_rate: number
}

type ViewMode = 'dashboard' | 'report' | 'history' | 'data'
type ChartType = 'histogram' | 'line' | 'scatter' | 'box'
```

#### 4.1.2 模拟数据生成参数

```typescript
interface DataGenConfig {
  UUT_LIST: string[]       // ['DUT-A01', 'DUT-A02', 'DUT-B01', 'DUT-B02', 'DUT-C01']
  FORMAT_LIST: string[]    // ['SINAD', 'THD', 'THD+N', 'IMD', 'SNR', 'SFDR']
  CHANNEL_LIST: string[]   // ['CH1', 'CH2', 'CH3', 'CH4']
  FREQ_LIST: string[]      // ['1kHz', '10kHz', '100kHz', '1MHz', '10MHz']
  MODE_LIST: string[]      // ['CW', 'Swept', 'Pulsed']
  NOMINALS: Record<string, { nominal: number; tolerance: number; unit: string }>
}
```

#### 4.1.3 标称值与公差

| 格式 | 标称值 | 公差 | 单位 |
|------|--------|------|------|
| SINAD | 80 | ±3 | dB |
| THD | -70 | ±5 | dBc |
| THD+N | -65 | ±5 | dBc |
| IMD | -60 | ±8 | dBc |
| SNR | 75 | ±3 | dB |
| SFDR | 70 | ±5 | dB |

### 4.2 输出契约

```typescript
interface CSVOutput {
  filename: string
  mime_type: 'text/csv;charset=utf-8'
  content: Blob
  encoding: 'UTF-8 with BOM'
}

interface ReportPreview {
  sections: ReportSection[]
  stats: StatResult
  formatStats: Record<string, StatResult>
  channelStats: Record<string, StatResult>
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | IDLE | 初始化Store+空数据 |
| IDLE | 点击Demo | DEMO | 生成模拟数据，isDemo=true |
| DEMO | 点击Stop | IDLE | isDemo=false，数据保留 |
| DEMO | 修改筛选条件 | DEMO | filteredData自动更新 |
| DEMO | 切换图表类型 | DEMO | Canvas重绘 |
| DEMO | 生成报告 | DEMO | 添加GeneratedReport+CSV下载 |
| * | 切换视图模式 | * | viewMode变更 |

### 4.4 状态判定逻辑

```typescript
function getStatus(value, nominal, tolerance): 'pass' | 'warn' | 'fail'
  diff = |value - nominal|
  diff <= tolerance * 0.8 → pass
  diff <= tolerance       → warn
  diff >  tolerance       → fail
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

### 5.2 页面布局规范

| 区域 | 宽度 | 高度 | 组件 |
|------|------|------|------|
| 顶部导航 | 100% | 48px | TopBar |
| 左侧导航 | 64px | 100% | NavMenu |
| 主内容区 | flex-1 | 100% | Dashboard/Report/History/Data |
| 右侧控制面板 | 224px | 100% | ControlPanel |
| 底部状态栏 | 100% | 28px | StatusBar |

### 5.3 DashboardView

#### 5.3.1 两列布局

| 属性 | 值 |
|------|-----|
| 布局 | 左列=直方图区，右列=散点图区 |
| 左列宽度 | 60% |
| 右列宽度 | 40% |

#### 5.3.2 直方图区（左列）

| 属性 | 值 |
|------|-----|
| 数量可选 | 1(单)/2(双)/4(四)，Store.histogramCount |
| 1个布局 | 1×1全高 |
| 2个布局 | 2×1纵向 |
| 4个布局 | 2×2网格 |
| 格式选择 | 每窗口左上角格式下拉选择器 |
| 图表类型 | 每窗口右上角histogram/line/scatter切换按钮 |
| 窗口标题 | 格式名·记录数·合格率% |

#### 5.3.3 散点图区（右列）

| 属性 | 值 |
|------|-----|
| 数量 | 2个，纵向排列 |
| 散点图A | Y轴=通道(CH1-CH4)，X轴=测量值，频率多选，6格式6色 |
| 散点图B | Y轴=频率(1kHz-10MHz)，X轴=测量值，通道多选，6格式6色 |

#### 5.3.4 窗口结构

```
┌──────────────────────┬──────────────────────┐
│  直方图区（左列60%）   │  散点图区（右列40%）   │
│  ┌─────────────────┐ │  ┌─────────────────┐ │
│  │ [SINAD▼] · 📊📈📉│ │  │ 散点图A: 通道×值  │ │
│  │   Canvas 图表    │ │  │ 频率:☑1k☑10k...  │ │
│  │                 │ │  │ Y=CH1-CH4 X=值   │ │
│  └─────────────────┘ │  └─────────────────┘ │
│  ┌─────────────────┐ │  ┌─────────────────┐ │
│  │ [THD▼] · 📊📈📉 │ │  │ 散点图B: 频率×值  │ │
│  │   Canvas 图表    │ │  │ 通道:☑CH1☑CH2... │ │
│  │                 │ │  │ Y=1k-10M X=值    │ │
│  └─────────────────┘ │  └─────────────────┘ │
└──────────────────────┴──────────────────────┘
```

#### 5.3.3 图表类型（per-format独立）

| 类型 | 说明 |
|------|------|
| histogram | 20bins直方图，pass/warn/fail色 |
| scatter | 散点图，状态色点 |
| line | 排序折线图 |

#### 5.3.4 多格式散点图

在6窗口网格下方，新增2个横跨全宽的散点图：

| 散点图 | Y轴(主轴) | X轴 | 多选维度 | 颜色区分 |
|--------|-----------|-----|----------|----------|
| 散点图A | 通道(CH1-CH4) | 测量值 | 频率多选(checkbox) | 6种格式6种颜色 |
| 散点图B | 频率(1kHz-10MHz) | 测量值 | 通道多选(checkbox) | 6种格式6种颜色 |

格式颜色映射：

| 格式 | 颜色 | 色值 |
|------|------|------|
| SINAD | 蓝 | #3B82F6 |
| THD | 绿 | #10B981 |
| THD+N | 黄 | #F59E0B |
| IMD | 红 | #EF4444 |
| SNR | 紫 | #8B5CF6 |
| SFDR | 粉 | #EC4899 |

#### 5.3.5 统计表

| 表格 | 列 |
|------|-----|
| 格式统计表 | 格式×数量×均值×标准差×合格率 |
| 通道统计表 | 通道×数量×均值×最小×最大×合格率 |

### 5.4 ReportView

#### 5.4.1 报告配置

| 区域 | 内容 |
|------|------|
| 报告名称 | input，默认"测试报告 + 日期" |
| 格式选择 | CSV / PDF |
| Generate按钮 | 生成报告+CSV下载 |

#### 5.4.2 Word风格中文报告

| 属性 | 值 |
|------|-----|
| 背景 | 白色(#FFFFFF) |
| 文字 | 深色(#1a1a1a) |
| 章节标题 | 深蓝(#1e40af)，加粗，14px |
| 表头背景 | 浅蓝(#F0F9FF) |
| 表头文字 | 深蓝(#1e40af)，加粗 |
| 表格边框 | 灰色(#E5E7EB) |
| 交替行 | 白色/浅灰(#F9FAFB) |
| 状态色 | 合格绿(#10B981)/警告黄(#F59E0B)/超差红(#EF4444) |

#### 5.4.3 报告章节（中文）

| 章节 | 标题 | 内容 |
|------|------|------|
| 一 | 测试概览 | 总记录数/均值/标准差/最小最大值/合格率 |
| 二~N | 按UUT分节 | 每个UUT一个章节，章节标题="被测单元：DUT-A01" |
| 二~N.1 | 按Format分表 | 每个Format一张矩阵表 |
| 二~N.1表 | Channel×Frequency矩阵 | 行=通道(CH1-CH4)，列=频率(1kHz-10MHz)，单元格=测量值+状态色 |

矩阵表结构：

```
被测单元：DUT-A01
  SINAD
  ┌──────┬───────┬───────┬───────┬───────┬───────┐
  │ 通道  │ 1kHz  │ 10kHz │100kHz │ 1MHz  │ 10MHz │
  ├──────┼───────┼───────┼───────┼───────┼───────┤
  │ CH1  │ 80.2✓ │ 79.8✓ │ 81.1✓ │ 78.5⚠ │ 80.0✓ │
  │ CH2  │ 79.6✓ │ 80.5✓ │ 79.1⚠ │ 80.3✓ │ 79.9✓ │
  │ CH3  │ 80.8✓ │ 79.4⚠ │ 80.1✓ │ 79.7✓ │ 81.2✓ │
  │ CH4  │ 79.3⚠ │ 80.6✓ │ 79.9✓ │ 80.4✓ │ 78.2✗ │
  └──────┴───────┴───────┴───────┴───────┴───────┘
  THD
  ┌──────┬───────┬───────┬───────┬───────┬───────┐
  │ ...  │  ...  │  ...  │  ...  │  ...  │  ...  │
  └──────┴───────┴───────┴───────┴───────┴───────┘
```

#### 5.4.4 中文标签映射

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
| Count | 数量 |
| PASS | 合格 |
| WARN | 警告 |
| FAIL | 超差 |
| Nominal | 标称值 |
| Tolerance | 公差 |
| UUT | 被测单元 |
| Frequency | 频率 |
| Status | 状态 |

### 5.5 导出规范

| 属性 | 值 |
|------|-----|
| 编码 | UTF-8 with BOM (\uFEFF) |
| 分隔符 | 逗号(,) |
| 字段引用 | 双引号(") |
| 文件名 | 报告名称.csv / all_data.csv / filtered_data.csv |

---

## 六、技术架构

### 6.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | Vue 3 + Composition API | SPA组件化开发 |
| 状态管理 | Pinia | 响应式Store，computed自动派生 |
| 图表渲染 | Canvas 2D | 零第三方依赖，histogram/line/scatter |
| 样式系统 | Tailwind CSS v3 | 创世纪仪器色彩体系 |
| 构建工具 | Vite | base: '/dataviz-report-ui/' |
| 模拟数据 | dataGenerator.ts | 高斯噪声+标称值+公差 |
| CSV导出 | Blob + URL.createObjectURL | 前端纯下载 |

### 6.2 项目目录结构

```
dataviz-report-ui/
├── public/
│   ├── favicon.svg
│   └── icons.svg
├── src/
│   ├── App.vue                        # 四区域布局入口
│   ├── main.ts                        # Vue 3 + Pinia初始化
│   ├── style.css                      # Tailwind + instrument体系
│   ├── components/
│   │   ├── TopBar.vue                 # 顶部导航栏
│   │   ├── NavMenu.vue                # 左侧4图标导航
│   │   ├── StatusBar.vue              # 底部状态栏
│   │   ├── ControlPanel.vue           # 右侧控制面板(Demo+Filter+Chart+Stats)
│   │   ├── DashboardView.vue          # 数据可视化(Canvas图表+统计表)
│   │   ├── ReportView.vue             # 报告生成+预览+CSV导出
│   │   ├── HistoryView.vue            # 报告历史列表
│   │   └── DataView.vue               # 数据管理+CSV导出
│   ├── stores/
│   │   └── datavizStore.ts            # Pinia Store
│   ├── types/
│   │   └── index.ts                   # 类型定义
│   └── utils/
│       └── dataGenerator.ts           # 模拟数据生成器
├── tailwind.config.js                 # 创世纪色彩体系
├── vite.config.ts                     # base路径配置
└── package.json
```

### 6.3 数据流

```
dataGenerator.ts                    datavizStore.ts                    Vue Components
┌──────────────┐                   ┌──────────────────┐              ┌──────────────┐
│generateTestData│──TestData[]──→│ allData: ref      │              │ DashboardView │
│computeStats   │                 │ filteredData: comp│──computed──→│ ReportView    │
│filterData     │                 │ stats: comp       │──computed──→│ HistoryView   │
│getStatus      │                 │ formatStats: comp │──computed──→│ DataView      │
│exportCSV      │                 │ channelStats: comp│──computed──→│ ControlPanel  │
│gaussRandom    │                 │ viewMode: ref     │              │ TopBar        │
└──────────────┘                   │ chartType: ref    │              │ StatusBar     │
                                   │ filter*: ref      │              └──────────────┘
                                   │ isDemo: ref       │
                                   │ reports: ref      │
                                   └──────────────────┘
```

### 6.4 模拟数据生成算法

```
generateTestData(count):
  baseDate = 2026-05-14T08:00:00
  for i in 0..count:
    format = random_choice(FORMAT_LIST)
    spec = NOMINALS[format]
    noise = gaussRandom() * spec.tolerance * 0.4
    value = spec.nominal + noise
    meastime = baseDate + i * 60000 * (1 + random * 3)
    → TestData{ id, uut, meastime, format, value, setting_raw, test_mode, channel, frequency, tolerance, nominal, created_at }

gaussRandom():
  u, v = uniform(0,1)  // Box-Muller变换
  return sqrt(-2*ln(u)) * cos(2*π*v)

computeStats(data):
  values = data.map(d => d.value)
  mean = avg(values)
  min, max = minmax(values)
  std_dev = sqrt(variance(values))
  pass_count = count(|value - nominal| <= tolerance)
  → StatResult{ mean, min, max, std_dev, count, pass_count, fail_count, pass_rate }
```

---

## 七、组件规范

### 7.1 TopBar

**职责**: 顶部导航栏

**内容**: DATAVIZ PRO / BP-0064 / DEMO标签 / records计数 / LED指示灯

**样式**: h-12, bg-night-gray, border-b, flex, px-4

### 7.2 NavMenu

**职责**: 左侧4图标导航

**菜单项**: 📊数据(dashboard) / 📈报告(report) / 📋历史(history) / 💾管理(data)

**样式**: w-16, bg-sidebar-bg, flex-col, items-center, gap-1

### 7.3 DashboardView

**职责**: 数据可视化主视图

**图表类型**: histogram(30bins, pass/warn/fail色) / scatter(点图, 状态色) / line(排序折线)

**统计表**: 格式统计(6行) + 通道统计(4行)

**技术**: Canvas 2D, devicePixelRatio适配, resize监听

### 7.4 ReportView

**职责**: 报告生成+预览

**配置**: 报告名称(input) + 格式(select: CSV/PDF) + Generate按钮

**预览4节**: 测试概览 / 格式性能 / 通道统计 / 详细记录(前50条)

**导出**: CSV自动下载(UTF-8 BOM)

### 7.5 ControlPanel

**职责**: 右侧控制面板

**4区域**: Demo控制(▶/■+记录数) / Filter(4维度select) / Chart类型 / Stats摘要

**筛选维度**: Format(6) / Channel(4) / Frequency(5) / UUT(5)

### 7.6 HistoryView

**职责**: 报告历史列表

**列**: ID / Name / Format / Records / Status / Created

### 7.7 DataView

**职责**: 数据管理

**功能**: 全量数据表(前200条) / Export CSV(全部) / Export Filtered(筛选)

### 7.8 StatusBar

**职责**: 底部状态栏

**内容**: LED + DEMO/IDLE + records + Pass% + Reports + BP-0064

---

## 八、安全约束

| 约束 | 实现 |
|------|------|
| 无后端通信 | 纯前端，零API调用 |
| 无敏感数据 | 模拟测试数据，不含个人隐私 |
| CSV安全 | Blob URL下载后立即revokeObjectURL |
| XSS防护 | Vue模板自动转义 |
| 数据隔离 | 浏览器内存，页面关闭即清除 |

---

## 九、实现状态追踪

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 类型定义 | types/index.ts | ✅ | 7接口+2类型 |
| 数据生成器 | utils/dataGenerator.ts | ✅ | 6函数，高斯噪声+统计+筛选+导出 |
| Pinia Store | stores/datavizStore.ts | ✅ | 10状态+4计算属性+5动作 |
| 顶部导航 | TopBar.vue | ✅ | DATAVIZ PRO + LED |
| 左侧导航 | NavMenu.vue | ✅ | 4图标导航 |
| 底部状态栏 | StatusBar.vue | ✅ | DEMO/IDLE + Pass% |
| 控制面板 | ControlPanel.vue | ✅ | Demo+Filter+Chart+Stats |
| 数据可视化 | DashboardView.vue | ✅ | Canvas 2D + 统计表 |
| 报告生成 | ReportView.vue | ✅ | 配置+预览+CSV导出 |
| 报告历史 | HistoryView.vue | ✅ | 历史列表 |
| 数据管理 | DataView.vue | ✅ | 数据表+导出 |
| 样式系统 | style.css | ✅ | Tailwind + instrument体系 |
| 构建配置 | vite.config.ts | ✅ | base: '/dataviz-report-ui/' |
| 画廊集成 | gallery.json | ✅ | exhibit-dataviz-report |

---

## 十、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 无持久化 | 数据仅存内存，刷新丢失 | v2.0接入后端API |
| 无PDF导出 | 仅CSV格式 | v1.2添加jsPDF |
| 无实时数据流 | 仅Demo模拟数据 | v1.1添加WebSocket |
| 图表无交互 | Canvas 2D静态渲染 | v1.2添加tooltip/hover |
| 无协作功能 | 单用户 | v2.0 |
| 数据量限制 | 前端内存限制，建议≤5000条 | v2.0分页+后端 |

---

## 十一、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-15 | 初始DRAFT版本，基于原始设计文档改写 |
| v1.1.0 | 2026-05-15 | CODE_DONE：Vue 3+Pinia+Canvas 2D实现，模拟数据生成器(6格式+高斯噪声)，8组件四区域布局，CSV导出，画廊集成 |
| v1.2.0 | 2026-05-15 | ReportView Word风格中文报告：白底深色文字+蓝色章节标题+浅蓝表头+中文标签(合格/警告/超差) |
| v1.3.0 | 2026-05-15 | DashboardView多窗口网格：6格式独立窗口(2×3)+per-format图表类型选择+窗口标题(格式·记录数·合格率) |
| v1.4.0 | 2026-05-15 | 窗口格式可选：每窗口左上角格式下拉选择器，多窗口可选同一格式对比不同图表类型 |
| v1.5.0 | 2026-05-15 | 多格式散点图(通道×频率+频率×通道，6格式6色+多选checkbox)+报告UUT×Format×Channel×Frequency矩阵表 |
| v1.6.0 | 2026-05-15 | 两列布局(左列直方图+右列散点图)，直方图数量可选(1/2/4)，散点图Y轴主轴(通道/频率)，X轴=测量值 |
| v1.7.0 | 2026-05-15 | 报告筛选(时间区间+UUT多选)，每张Format矩阵表独立导出CSV，表标题居中+合格范围(标称值±公差) |

---

## 附录

### A. 参考文档

- [数据可视化与报告生成系统UI设计文档.docx](../../数据可视化与报告生成系统UI设计文档.docx) - 原始设计文档
- [BP-0062-iq3d-fusion.md](./BP-0062-iq3d-fusion.md) - IQ3D Fusion消费者蓝图（Canvas可视化参考）
- [BP-0063-fft-decomposition.md](./BP-0063-fft-decomposition.md) - FFT分解消费者蓝图（Vanilla JS参考）
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图

### B. 术语表

| 术语 | 说明 |
|------|------|
| UUT | Unit Under Test，被测单元 |
| meastime | Measurement Time，测量时间 |
| SINAD | Signal to Noise And Distortion，信纳比 |
| THD | Total Harmonic Distortion，总谐波失真 |
| THD+N | THD plus Noise，总谐波失真加噪声 |
| IMD | Intermodulation Distortion，互调失真 |
| SNR | Signal to Noise Ratio，信噪比 |
| SFDR | Spurious Free Dynamic Range，无杂散动态范围 |
| Box-Muller | 高斯随机数生成变换 |
| BOM | Byte Order Mark，UTF-8文件头标识 |

### C. 与BP-0062/BP-0063的对比

| 维度 | BP-0062 IQ3D Fusion | BP-0063 FFT Decomposition | BP-0064 DataViz Report |
|------|---------------------|---------------------------|------------------------|
| 技术栈 | Vue 3 + Canvas 2D | Vanilla JS + Canvas 2D | Vue 3 + Canvas 2D |
| 状态管理 | Pinia | 全局变量 | Pinia |
| 数据源 | Demo合成数据 | 数学信号生成 | 模拟测试数据(高斯噪声) |
| 导出 | 无 | 无 | CSV(UTF-8 BOM) |
| 报告生成 | 无 | 无 | 模板化生成+预览 |
| 部署复杂度 | 低(静态) | 低(静态) | 低(静态) |
| 教学价值 | 仪器可视化 | FFT原理演示 | 测试数据全生命周期 |
