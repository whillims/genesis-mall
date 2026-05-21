# 微波TR测试UI - 知识库文档

> **来源**: Kimi_Agent_微波TR测试UI完成.zip
> **创建日期**: 2026-05-14
> **项目路径**: `knowledge-base/tr-test-ui/`

---

## 1. 项目概述

微波TR模块测试报告系统（TR Sentinel），用于微波收发模块的自动化测试数据采集、可视化分析与报告生成。

### 1.1 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 18.x | UI框架 |
| TypeScript | 5.x | 类型系统 |
| Vite | 7.x | 构建工具 |
| Tailwind CSS | 3.4.x | 样式框架 |
| shadcn/ui | - | 基础组件库 (40+) |
| ECharts | - | 热力图/迷你图 |
| recharts | - | 图表库 |
| xlsx | - | Excel导出 |
| file-saver | - | 文件下载 |
| lucide-react | - | 图标库 |

### 1.2 项目特点

- 终端风格启动动画（TerminalIntro）
- 暗色科技主题（深蓝/青绿配色）
- 13种微波参数规格库
- 频率-通道热力图可视化
- 四格式报告导出（PDF/XLSX/Markdown/HTML）
- 扫描线动效

---

## 2. 项目结构

```
tr-test-ui/
├── public/
│   ├── data-texture.jpg      # 数据纹理背景
│   └── hero-bg.jpg           # 英雄区背景
├── src/
│   ├── components/
│   │   ├── ui/               # shadcn/ui组件 (40+)
│   │   ├── FilterBar.tsx     # 筛选工具栏
│   │   ├── FormatCard.tsx    # 指标卡片（含迷你图）
│   │   ├── Heatmap.tsx       # 频率-通道热力图
│   │   ├── ReportView.tsx    # 测试报告视图
│   │   ├── ScanLine.tsx      # 扫描线动效
│   │   └── TerminalIntro.tsx # 终端启动动画
│   ├── hooks/
│   │   └── use-mobile.ts     # 移动端检测
│   ├── lib/
│   │   ├── declarations.d.ts # 类型声明
│   │   ├── export.ts         # 四格式导出引擎
│   │   ├── specs.ts          # 13种参数规格库
│   │   ├── testData.ts       # 模拟数据生成器
│   │   ├── types.ts          # 类型定义
│   │   └── utils.ts          # 工具函数
│   ├── pages/
│   │   └── Home.tsx          # 主页
│   ├── App.tsx               # 根组件
│   ├── App.css               # 应用样式
│   ├── index.css             # 全局样式
│   └── main.tsx              # 入口文件
├── package.json
├── tailwind.config.js
├── vite.config.ts
└── tsconfig.json
```

---

## 3. 核心页面与视图

### 3.1 视图模式

系统有三种视图模式：

```typescript
type ViewMode = 'intro' | 'dashboard' | 'report';
```

| 模式 | 说明 |
|------|------|
| `intro` | 终端启动动画（自动播放后进入dashboard） |
| `dashboard` | 数据分析仪表盘（筛选+卡片+热力图） |
| `report` | 测试报告视图（表格+结论+签名区） |

### 3.2 仪表盘视图 (Dashboard)

```
┌─────────────────────────────────────────────────────────┐
│  TR Sentinel v3.2    |  仪表盘  |  测试报告            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── FilterBar ────────────────────────────────────┐   │
│  │ 时间范围: [2024-05-10] ~ [2024-05-15]           │   │
│  │ UUT批次:  [A001] [A002] [A003] [A004] [A005]   │   │
│  │ 通道筛选: [CH1] [CH2] ... [CH8]                 │   │
│  │ [生成报告] [导出▼]                               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  选择指标: [S11] [S21] [S12] [S22] [NF] [P1dB] ...    │
│                                                         │
│  ┌─FormatCard─┐ ┌─FormatCard─┐ ┌─FormatCard─┐         │
│  │ S11 回波损耗│ │ S21 增益   │ │ S12 反向隔离│        │
│  │ ~~~迷你图~~~│ │ ~~~迷你图~~│ │ ~~~迷你图~~│         │
│  │ -19.2 dB ↓ │ │ 18.5 dB ↑ │ │ -25.1 dB   │         │
│  │ ●0 ●2 ●46  │ │ ●0 ●1 ●47 │ │ ●0 ●0 ●48  │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
│                                                         │
│  ┌─── Heatmap ──────────────────────────────────────┐   │
│  │ S11 —— 回波损耗 异常分布热力图                   │   │
│  │     1GHz  2GHz  3GHz  4GHz  5GHz  6GHz          │   │
│  │ CH1  ■     ■     ■     ■     ■     ■            │   │
│  │ CH2  ■     ■     ■     ■     ■     ■            │   │
│  │ ...                                               │   │
│  │ ■通过  ■警告  ■失败  ■无数据                     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─统计─┐ ┌─统计─┐ ┌─统计─┐ ┌─统计─┐                  │
│  │总测试│ │ 通过  │ │ 警告 │ │ 失败 │                  │
│  │ 1056 │ │ 980  │ │  52  │ │  24  │                  │
│  └──────┘ └──────┘ └──────┘ └──────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 3.3 测试报告视图 (ReportView)

```
┌─────────────────────────────────────────────────────────┐
│  ← 返回分析仪表盘                                      │
│                                                         │
│         微 波 T R 模 块 测 试 报 告                     │
│  报告编号：RPT-xxx | 生成时间 | 测试批次 | 时间范围     │
│  [总测试项] [通过 xx%] [警告 xx%] [失败 xx%]           │
│  ████████████████████████████████████████               │
│                                                         │
│  ── UUT-A001 测试详情 ──────────────────────────────    │
│  ┌─ S11 ── 回波损耗 (dB) ── 判定: <= -10dB ──────┐   │
│  │ 通道  │ 1.0GHz │ 2.0GHz │ 3.0GHz │ ...        │   │
│  │ CH1   │ -19.2✓ │ -18.5✓ │ -20.1✓ │ ...        │   │
│  │ CH2   │ -9.8✗  │ -11.2✓ │ -10.5⚠ │ ...        │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─── 测试结论 ─────────────────────────────────────┐   │
│  │ 综合判定：🟢 合格 / 🟡 条件通过 / 🔴 不合格     │   │
│  │                                                   │   │
│  │ 测试工程师 _______  审核 _______  日期 _______    │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. 核心组件详解

### 4.1 TerminalIntro - 终端启动动画

**文件**: `src/components/TerminalIntro.tsx`

终端风格逐行打字动画，模拟系统初始化过程：

```typescript
const LOGS: LogEntry[] = [
  { text: 'TR Sentinel v3.2.1 — Microwave Test Report System', color: 'system' },
  { text: 'Initializing secure connection to test server...', color: 'default' },
  { text: '  > SSL handshake complete (TLS 1.3)', color: 'success' },
  { text: 'Discovering UUT devices on test bench...', color: 'default' },
  { text: '  [OK] UUT-A001 — S11/RL/NF module detected', color: 'success' },
  // ... 6个UUT设备发现
  { text: 'Running format spec discovery...', color: 'default' },
  { text: '  S11: threshold <= -10 dB (warn_delta: 2)', color: 'warning' },
  // ... 规格加载
  { text: 'System ready. Rendering dashboard...', color: 'system' },
];
```

**特性**:
- 逐字符打字效果（25-65ms/字符）
- 行间延迟（100-800ms）
- 光标闪烁动画
- 自动滚动
- 可跳过

### 4.2 FilterBar - 筛选工具栏

**文件**: `src/components/FilterBar.tsx`

**筛选维度**:

| 维度 | 类型 | 说明 |
|------|------|------|
| timeRange | 日期范围 | 测试时间区间 |
| uuts | 多选 | UUT批次（6个） |
| channels | 多选 | 通道号（CH1-CH8） |
| freqRange | 范围 | 频率区间 |

**操作按钮**:
- 生成报告（带加载动画）
- 导出下拉菜单（PDF/XLSX/Markdown/HTML）

### 4.3 FormatCard - 指标卡片

**文件**: `src/components/FormatCard.tsx`

每个卡片展示一种微波参数指标：

```
┌──────────────────────┐
│ 回波损耗          48点│
│ S11                  │
│ ~~~~~迷你趋势图~~~~~~│
│ -19.2 dB  ↓         │
│ ●0 ●2 ●46           │
└──────────────────────┘
```

**视觉特性**:
- ECharts迷你趋势图（sparkline）
- 趋势箭头（↑/↓/—）
- 状态颜色编码（绿/黄/红）
- 悬停发光效果

### 4.4 Heatmap - 频率-通道热力图

**文件**: `src/components/Heatmap.tsx`

基于ECharts的热力图，展示选定指标在频率-通道矩阵上的通过/警告/失败分布：

```typescript
interface HeatmapCell {
  channel: number;
  frequency: number;
  status: JudgeResult;  // 'pass' | 'warning' | 'fail' | 'nodata'
  value: number | null;
}
```

**颜色映射**:
| 状态 | 颜色 |
|------|------|
| 通过 | `#0d3b2e` (深绿) |
| 警告 | `#3d3418` (深黄) |
| 失败 | `#3d1520` (深红) |
| 无数据 | `#1a2332` (深蓝) |

### 4.5 ReportView - 测试报告视图

**文件**: `src/components/ReportView.tsx`

完整的测试报告渲染，包含：
- 报告元信息（编号/时间/批次）
- 全局统计条
- 按UUT分组的格式化表格
- 测试结论与综合判定
- 签名区

### 4.6 ScanLine - 扫描线动效

**文件**: `src/components/ScanLine.tsx`

两条扫描线动效：
- 主扫描线：青绿色，30%宽度
- 副扫描线：蓝色，20%宽度，延迟2秒

---

## 5. 数据模型

### 5.1 核心类型定义

**文件**: `src/lib/types.ts`

```typescript
interface TestRecord {
  uut: string;           // UUT编号
  meastime: number;      // 测量时间戳
  format: string;        // 参数格式 (S11/S21/NF等)
  value: number;         // 测量值
  setting: {
    frequency: number;   // 频率 (Hz)
    channel: number;     // 通道号
  };
}

type JudgeResult = 'pass' | 'warning' | 'fail' | 'nodata';

interface FormatSpec {
  title: string;         // 参数中文名
  unit: string;          // 单位
  min: number | null;    // 最小阈值
  max: number | null;    // 最大阈值
  warn_delta: number;    // 警告裕度
}

interface ReportData {
  reportId: string;
  generatedAt: string;
  uuts: string[];
  timeRange: string;
  uutTables: ReportUUTData[];
  conclusion: ConclusionData;
  globalStats: { total, pass, warning, fail, passRate, warnRate, failRate };
}
```

### 5.2 参数规格库

**文件**: `src/lib/specs.ts`

13种微波参数规格：

| 格式 | 中文名 | 单位 | 判定条件 | 警告裕度 |
|------|--------|------|----------|----------|
| S11 | 回波损耗 | dB | ≤ -10 | 2 |
| S21 | 增益 | dB | ≥ 15 | 2 |
| S12 | 反向隔离 | dB | ≤ -20 | 3 |
| S22 | 输出回波 | dB | ≤ -10 | 2 |
| NF | 噪声系数 | dB | ≤ 3.0 | 1 |
| P1dB | 1dB压缩点 | dBm | ≥ 20 | 2 |
| IP3 | 三阶交调点 | dBm | ≥ 30 | 3 |
| Phase | 相位偏差 | deg | -5 ~ 5 | 2 |
| AMPM | AM-PM转换 | deg/dB | ≤ 5 | 1 |
| EVM | 误差矢量幅度 | % | ≤ 5 | 1 |
| ACPR | 邻道功率比 | dBc | ≥ 40 | 3 |
| RL | 驻波比 | - | ≤ 2.0 | 0.3 |
| GainFlatness | 增益平坦度 | dB | ≤ 2 | 0.5 |

### 5.3 模拟数据生成

**文件**: `src/lib/testData.ts`

6个UUT配置，6个频点（1-6GHz），8个通道：

```typescript
const UUT_CONFIGS = {
  'UUT-A001': ['S11', 'RL', 'NF'],
  'UUT-A002': ['S22', 'S12', 'S21', 'AMPM'],
  'UUT-A003': ['EVM', 'IP3', 'S11'],
  'UUT-A004': ['S21', 'S22', 'RL'],
  'UUT-A005': ['AMPM', 'S22', 'ACPR'],
  'UUT-A006': ['S22', 'Phase', 'EVM', 'NF', 'S11', 'S12'],
};

const FREQUENCIES = [1.0e9, 2.0e9, 3.0e9, 4.0e9, 5.0e9, 6.0e9];
const CHANNELS = [1, 2, 3, 4, 5, 6, 7, 8];
```

**数据生成特性**:
- 确定性随机种子（可复现）
- 高斯分布（Box-Muller变换）
- UUT系统性偏差
- 通道相关变异
- 异常注入（S22高通道恶化、EVM高频恶化、NF整体偏高）

---

## 6. 导出引擎

**文件**: `src/lib/export.ts`

支持四种格式导出：

| 格式 | 函数 | 说明 |
|------|------|------|
| Markdown | `exportMarkdown()` | 生成.md文件，含表格和统计 |
| XLSX | `exportXLSX()` | 生成Excel文件，按UUT+格式分Sheet |
| HTML | `exportHTML()` | 生成独立HTML报告，含打印样式 |
| PDF | `exportPDF()` | 打开新窗口打印为PDF（A4横向） |

### 6.1 Markdown导出格式

```markdown
# 微波TR模块测试报告

**报告编号**：RPT-xxx
**测试批次**：UUT-A001、UUT-A002...
**生成时间**：2024-05-15 10:30:00

## 全局统计
- 总测试项：1056
- 通过：980 (92.80%)
- 警告：52 (4.92%)
- 失败：24 (2.27%)

## UUT-A001 测试详情
### S11 —— 回波损耗 (dB)
| 通道 | 1.0GHz | 2.0GHz | ... |
|------|--------|--------|-----|
| CH1  | -19.2 ✓ | -18.5 ✓ | ... |
```

### 6.2 XLSX导出格式

- 每个UUT+格式组合一个Sheet
- Sheet名：`UUT-A001_S11`（最长31字符）
- 含表头、阈值说明、数据矩阵、统计行
- 额外"结论"Sheet

---

## 7. 样式系统

### 7.1 设计变量

```css
/* 主色调 */
--bg-primary: #050B14;      /* 深空蓝背景 */
--bg-card: #131C27;         /* 卡片背景 */
--bg-card-alt: #0E1A25;     /* 交替行/区域 */
--border: #263746;          /* 边框/分隔线 */

/* 强调色 */
--accent-cyan: #00E5A0;     /* 主强调（青绿） */
--accent-blue: #3B82F6;     /* 次强调（蓝） */
--accent-red: #FF3366;      /* 失败/危险 */
--accent-yellow: #FFB800;   /* 警告 */

/* 文字色 */
--text-primary: #D8DDE4;    /* 主文字 */
--text-secondary: #8A96A3;  /* 次要文字 */
--text-muted: #5C6370;      /* 辅助文字 */
```

### 7.2 背景层次

```
Layer 0: hero-bg.jpg (opacity: 0.08)     # 英雄区背景图
Layer 1: data-texture.jpg (opacity: 0.03) # 数据纹理
Layer 2: CSS Grid (opacity: 0.02)         # 60px网格线
Layer 3: ScanLine                          # 扫描线动效
Layer 4: Content (z-index: 10)            # 主内容
```

---

## 8. 与创世纪项目集成

### 8.1 集成场景

该UI可作为创世纪商场的测试报告消费者，对接IO调试沙箱和RTL-SDR频谱仪：

```
┌──────────────────────────────────────────────────────┐
│           TR Sentinel UI (React)                      │
│                                                      │
│  /api/rtlsdr/spectrum → FastAPI → rtlsdr_worker     │
│  /api/vector/read/*   → MallCore → VectorManager    │
│  /api/task/publish    → MallCore → 任务管理          │
└──────────────────────────────────────────────────────┘
```

### 8.2 可复用组件

| 组件 | 说明 | 复用价值 |
|------|------|----------|
| FormatCard | 指标卡片+迷你图 | 高 - 可展示Worker状态 |
| Heatmap | 频率-通道热力图 | 高 - 可展示SHM矢量状态 |
| FilterBar | 多维筛选工具栏 | 中 - 通用筛选 |
| TerminalIntro | 终端启动动画 | 中 - 启动页特效 |
| ReportView | 报告视图 | 高 - 可对接审计报告 |
| export.ts | 四格式导出引擎 | 高 - 通用导出能力 |

### 8.3 集成建议

1. **替换mock数据**: 将 `testData.ts` 的模拟数据替换为API调用
2. **对接RTL-SDR**: 将频谱数据接入热力图展示
3. **对接审计模块**: 将审计日志接入ReportView
4. **矢量数据映射**: 将SHM矢量数据映射为TestRecord格式
5. **规格库扩展**: 将 `specs.ts` 扩展为可配置的规格管理

---

## 9. 运行方式

```bash
# 进入项目目录
cd knowledge-base/tr-test-ui

# 安装依赖
npm install

# 开发模式
npm run dev

# 构建生产版本
npm run build

# 预览生产版本
npm run preview
```

---

## 10. 相关资源

- [微波TR测试报告UI设计蓝图 v3.3](file:///d:\创世纪\微波TR测试报告UI设计蓝图_v3_3.md)
- [shadcn/ui 官方文档](https://ui.shadcn.com/)
- [ECharts 文档](https://echarts.apache.org/)
- [SheetJS (xlsx) 文档](https://docs.sheetjs.com/)

---

*本文档由SOLO-AICoder生成*
