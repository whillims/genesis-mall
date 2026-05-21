# 微波TR测试报告UI设计蓝图 v3.3
## ——多UUT时间筛选动态字段测试报告系统（生产级实现版）

---

## 变更日志 (v3.2 → v3.3)

| 版本 | 日期 | 变更内容 |
|---|---|---|
| v3.3 | 2026-05-15 | 生产级实现反馈：补充UI设计系统、增加终端启动动效、优化导出实现细节、补充性能优化策略 |
| v3.2 | 2024-05-15 | 初始蓝图 |

---

## 一、数据模型（字段种类不固定）

### 1.1 核心Schema

```
records 表结构
├── uut          : STRING   -- 被测单元批次号
├── meastime     : FLOAT    -- 测量时间戳(Unix秒)
├── format       : STRING   -- 数据格式/指标名称（动态字段，非固定枚举）
├── value        : FLOAT    -- 测量结果值
└── setting      : JSON     -- 测试条件
    ├── frequency: FLOAT    -- 激励频率(Hz)
    └── channel  : INT      -- 射频通道号(1-8)
```

**关键设计**：`format`字段为**动态字段**，不同UUT可能测试不同指标组合。系统不得假设format种类固定，必须从实际数据集中自动发现。

### 1.2 测试数据规模（字段种类不固定示例）

| UUT | 测试Format种类 | 记录数 | 测试时间 |
|---|---|---|---|
| UUT-A001 | S11, RL, NF | 144 | 05-10 |
| UUT-A002 | S22, S12, S21, AMPM | 192 | 05-11 |
| UUT-A003 | EVM, IP3, S11 | 144 | 05-12 |
| UUT-A004 | S21, S22, RL | 144 | 05-13 |
| UUT-A005 | AMPM, S22, ACPR | 144 | 05-14 |
| UUT-A006 | S22, Phase, EVM, NF, S11, S12 | 288 | 05-15 |

**总记录数**：1,056条（6个UUT × 各3~6种Format × 6频点 × 8通道）

### 1.3 动态规格发现机制

```javascript
// 系统启动时从数据中自动发现所有format及其规格
function discoverSpecs(records) {
  const formatGroups = groupBy(records, "format");
  const specs = {};
  for (const [fmt, recs] of Object.entries(formatGroups)) {
    const values = recs.map(r => r.value).sort((a, b) => a - b);
    const min = values[0];
    const max = values[values.length - 1];
    specs[fmt] = {
      title: fmt,
      unit: "",
      min: null,
      max: null,
      warn_delta: Math.abs(max - min) * 0.05
    };
  }
  return specs;
}

// 生产系统中规格应从外部规格库加载
const specLibrary = {
  "S11":  { title: "回波损耗",     unit: "dB",  min: null, max: -10,  warn_delta: 2 },
  "S21":  { title: "增益",         unit: "dB",  min: 15,   max: null, warn_delta: 2 },
  "S12":  { title: "反向隔离",     unit: "dB",  min: null, max: -20,  warn_delta: 3 },
  "S22":  { title: "输出回波",     unit: "dB",  min: null, max: -10,  warn_delta: 2 },
  "NF":   { title: "噪声系数",     unit: "dB",  min: null, max: 3.0,  warn_delta: 1 },
  "P1dB": { title: "1dB压缩点",    unit: "dBm", min: 20,   max: null, warn_delta: 2 },
  "IP3":  { title: "三阶交调点",   unit: "dBm", min: 30,   max: null, warn_delta: 3 },
  "Phase":{ title: "相位偏差",     unit: "deg", min: -5,   max: 5,    warn_delta: 2 },
  "AMPM": { title: "AM-PM转换",    unit: "deg/dB", min: null, max: 5, warn_delta: 1 },
  "EVM":  { title: "误差矢量幅度", unit: "%",   min: null, max: 5,    warn_delta: 1 },
  "ACPR": { title: "邻道功率比",   unit: "dBc", min: 40,   max: null, warn_delta: 3 },
  "RL":   { title: "驻波比",       unit: "",    min: null, max: 2.0,  warn_delta: 0.3 },
  "GainFlatness": { title: "增益平坦度", unit: "dB", min: null, max: 2, warn_delta: 0.5 }
};
```

---

## 二、UI设计系统（v3.3 新增）

### 2.1 设计理念

采用"Dark Mode 极客美学"——受 Bloomberg / Refinitiv 等顶级金融终端启发，通过深邃暗色背景、高饱和度霓虹状态色、以及精确锐利的排版，将枯燥的测试数据转化为"可交互的科技叙事"。

### 2.2 色彩宪法

| 用途 | 色值 | 说明 |
|---|---|---|
| 核心背景 | `#050B14` | 深邃蓝黑，全屏主背景 |
| 面板背景 | `#0A111A` | 悬浮控制面板底色 |
| 卡片背景 | `#131C27` | 数据卡片、表格主背景 |
| 交替行 | `#0E1A25` | 表格斑马线交替色 |
| 主文字 | `#D8DDE4` | 标题与核心指标 |
| 次文字 | `#8A96A3` | 正文与次级信息 |
| 分割线 | `#263746` | 暗海蓝灰，极细分隔线 |
| **通过** | `#00E5A0` | 赛博青绿 |
| **警告** | `#FFB800` | 警示琥珀 |
| **失败** | `#FF3366` | 霓虹深红 |
| **交互** | `#3B82F6` | 数据蓝光，hover/聚焦态 |

### 2.3 字体体系

| 用途 | 字体 | 说明 |
|---|---|---|
| 界面与正文 | `Noto Sans SC` / `PingFang SC` | 暗色模式中文极佳阅读体验 |
| 数据与指标 | `JetBrains Mono` | 所有S11/NF/dB等终端数据，字距紧凑 |

### 2.4 间距与边框

- 卡片间距：`24px`
- 边框圆角：极度克制，`2px` 或 `4px`，拒绝圆滑SaaS风格
- 分割线：`1px solid #263746`，取代阴影
- 卡片顶部微光：`1px solid rgba(255,255,255,0.05)` 提升材质层次感

---

## 三、全局筛选栏（时间 + UUT多选）

### 3.1 筛选器布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  [FILTER ICON] 测试报告筛选                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  测试时间范围：                                                               │
│  [📅 2024-05-10 08:00]  至  [2024-05-15 23:59]                              │
│                                                                             │
│  UUT批次（可多选）：                    [全选] [清空]                          │
│  [● A001] [● A002] [● A003] [○ A004] [○ A005] [○ A006]                     │
│                                                                             │
│  通道筛选：                    [全选] [清空]                                  │
│  [● CH1] [● CH2] [● CH3] [● CH4] [● CH5] [● CH6] [● CH7] [● CH8]           │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  [🔄 生成报告]  [📥 导出 ▼]                              UUT: 3/6  CH: 8/8  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 筛选状态机

```javascript
const filterState = {
  timeRange: { start: null, end: null },
  uuts: [],        // 多选数组
  channels: [],    // 多选数组 [1-8]
  freqRange: { min: null, max: null }
};

function filterRecords(records, filters) {
  return records.filter(r => {
    if (filters.timeRange.start && r.meastime < filters.timeRange.start) return false;
    if (filters.timeRange.end && r.meastime > filters.timeRange.end) return false;
    if (filters.uuts.length > 0 && !filters.uuts.includes(r.uut)) return false;
    if (filters.channels.length > 0 && !filters.channels.includes(r.setting.channel)) return false;
    if (filters.freqRange.min && r.setting.frequency < filters.freqRange.min) return false;
    if (filters.freqRange.max && r.setting.frequency > filters.freqRange.max) return false;
    return true;
  });
}
```

---

## 四、核心特效与动效（v3.3 新增）

### 4.1 终端启动动画（Terminal Intro Sequence）

首次加载时覆盖全屏的CRT终端窗口，逐字打印系统自检日志。

**实现要点**：
- 纯黑背景 `#000000`，macOS风格三色圆点（红/黄/绿）
- 字体：`JetBrains Mono` 14px
- 文字颜色分层：普通 `#E0E0E0` / 成功 `#00E5A0` / 警告 `#FFB800` / 系统 `#3B82F6`
- 打字速度：25-65ms/字符，行间延迟100-600ms模拟网络延迟
- 自动滚动到底部，完成后自动进入仪表盘

### 4.2 雷达扫描线（Radar Sweep）

固定定位的Canvas/CSS叠加层，`pointer-events: none` 不阻挡交互。

**CSS实现**：
- 主线：2px宽霓虹青绿渐变，3s线性无限循环，从左到右扫描
- 副线：1px宽蓝色，5s周期，2s延迟
- 仅作为氛围特效，不直接绑定热力图数据

### 4.3 卡片Hover发光

鼠标悬停时：
- 边框变为 `#3B82F6` 40%透明度
- 微弱的发光阴影：`0 0 20px rgba(59,130,246,0.08)`
- 卡片内发出与当前状态对应的颜色微光（pass=青绿, warn=琥珀, fail=深红）

---

## 五、第一阶段：异常分析仪表盘

### 5.1 Format分布总览（动态卡片网格）

4-6列响应式网格，每张卡片代表一个动态发现的Format。

**卡片内部结构**：
- 顶部：极小灰色英文标签（如 `RETURN LOSS`）+ 大号格式代号（如 `S11`）
- 中部：Echarts sparkline折线图（40px高），纯线条无背景，颜色映射状态
- 数据区：`JetBrains Mono` 大字显示最新值（如 `-19.8 dB`）+ 趋势箭头
- 底部：三个LED状态计数器（🔴异常 🟡警告 🟢通过）

### 5.2 热力图（按频率为主轴，8通道×6频点）

Echarts Heatmap渲染，X轴=6频点(1.0-6.0GHz)，Y轴=8通道(CH1-CH8)。

**色彩映射**：
| 状态 | 热力图色值 | 说明 |
|---|---|---|
| 通过 | `#0d3b2e` | 极暗灰绿 |
| 警告 | `#3d3418` | 暗琥珀 |
| 失败 | `#3d1520` | 暗深红 |
| 无数据 | `#1a2332` | 深空灰 |

**Tooltip设计**：深黑底(`#0A111A`) + 青绿边框(`#00E5A0`)，显示通道、频率、实测值、判定结果。

---

## 六、第二阶段：UUT测试报告

### 6.1 报告总览栏

- **大标题**："微 波 T R 模 块 测 试 报 告"，居中，letter-spacing: 8px，下方贯穿线
- **元数据**：`JetBrains Mono` 居中列出报告编号、生成时间、测试批次、时间范围
- **全局统计栏**：四大指标卡片并排（总测试项/通过/警告/失败），下方彩色进度条直观显示比例

### 6.2 UUT分栏数据矩阵

- **UUT分栏头**：深灰粗边框标题栏（如 `UUT-A001 测试详情`），强烈分隔
- **Format表格**：深灰表头 + 交替行色(`#131C27`/`#0E1A25`)
- **状态渲染**：单元格数值右侧带状态符号（✓/⚠/✗），通过行绿色左边框，失败行红色左边框

### 6.3 测试结论与签章

**自动生成逻辑**：
```javascript
function generateConclusion(filteredRecords, specs) {
  const total = filteredRecords.length;
  const failCount = /* ... */;
  const warnCount = /* ... */;
  const passCount = total - failCount - warnCount;
  const failRate = (failCount / total * 100).toFixed(1);
  
  // 三种模板：全通过 / 仅警告 / 有失败
  if (failCount === 0 && warnCount === 0) {
    return "本次测试共涉及N个被测单元...全部通过，判定为合格。";
  } else if (failCount === 0) {
    return "...警告项集中在XX指标，建议关注但可放行。";
  } else {
    return "...失败项主要集中在XX指标。建议返修或报废处理。";
  }
}
```

**签章区**：三个下划线占位符（测试工程师/审核/日期），体现正式工业报告质感。

---

## 七、批量导出功能

### 7.1 导出菜单

| 格式 | 技术方案 | 特点 |
|---|---|---|
| PDF | 浏览器print API + A4 landscape样式 | 打印样式，适合归档 |
| XLSX | SheetJS (xlsx) | 每UUT每Format一Sheet |
| Markdown | 原生Blob | 纯文本，适合版本管理 |
| HTML | 原生Blob自包含 | 浏览器直接打开 |

### 7.2 导出实现要点（v3.3 优化）

**PDF导出**：
- 使用 `window.open()` 打开新窗口
- 写入完整HTML含print-optimized CSS
- `@page { size: A4 landscape; margin: 15mm; }`
- 字体：`SimSun` / `Microsoft YaHei`
- 状态颜色类：`.pass { background: #E8F5E9; }` 等

**XLSX导出**：
- 每个Sheet名称限制31字符（Excel限制）
- 列宽自适应：`ws['!cols'] = [{wch: 10}, ...]`
- 独立"结论"Sheet含统计摘要

**Markdown导出**：
- 标准Markdown表格语法
- 状态符号：✓(通过) ⚠(警告) ✗(失败)

---

## 八、技术栈（v3.3 生产确认）

| 层级 | 库 | 版本 | 职责 |
|---|---|---|---|
| 框架 | React + TypeScript + Vite | 18.x / 5.x | UI框架与构建 |
| 样式 | Tailwind CSS 3.4 | 3.4.x | 原子化CSS |
| 组件 | shadcn/ui | latest | 基础UI组件 |
| 图表 | Apache ECharts + echarts-for-react | 5.5+ | 热力图与sparkline |
| XLSX导出 | SheetJS (xlsx) | 0.20+ | Excel生成 |
| 文件保存 | file-saver | 2.0+ | 浏览器下载触发 |
| 字体 | JetBrains Mono + Noto Sans SC | - | 数据/界面双字体栈 |

---

## 九、性能优化策略（v3.3 新增）

1. **数据层面**：测试数据在初始化时一次性生成，后续筛选基于内存计算，避免重复I/O
2. **图表层面**：Echarts使用canvas渲染，sparkline关闭animation降低CPU占用
3. **渲染层面**：React `useMemo` 缓存filterRecords/formatCards/heatmapData，避免重复计算
4. **打包层面**：Vite tree-shaking + code splitting，chunk大小约1.75MB（含Echarts完整库）

---

## 十、状态流转图（v3.3 新增）

```
[终端启动动画] ──完成──> [异常分析仪表盘] ──点击"生成报告"──> [UUT测试报告]
                                                               │
                                                               └── 点击"返回仪表盘" ──> [异常分析仪表盘]
```

---

*本蓝图遵循商场三权分立宪法：数据库记录所有权归生产者，消费者仅拥有可视化"相"的定制权与数据迁移自由。*

*UI设计系统采用 Dark Mode 极客美学——深邃、精确、未来感。所有色值与字体选择服务于"让枯燥的射频测试数据变得可交互且充满控制力"这一核心目标。*
