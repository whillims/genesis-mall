# 创世纪基因图谱页面消费者蓝图

> **蓝图编号**: BP-0061-GENOME-VIEWER
> **版本**: v1.0.0
> **日期**: 2026-05-15
> **功能域**: consumer
> **状态**: DRAFT

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0057-GENOME-VIEWER |
| 函数名 | genome_viewer_init / genome_viewer_tick |
| 生产者名称 | GenomeViewerConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | 静态配置 (genome.json) |
| 输出协议 | SHM矢量发布 (VEC-CONTROL) |
| 安全等级 | P1 (内部工具) |
| 版本 | v1.0.0 |

---

## 二、设计意图

### 2.1 核心目标

创建创世纪项目的基因图谱可视化页面，以沉浸式单页长滚动方式展示项目基因体系和蓝图资料。用户通过该页面可以：

1. **总览** — 一目了然地浏览项目6层基因图谱
2. **深入** — 每层基因的详细内容（架构图、流程图、状态机）
3. **导航** — 快速跳转到任意基因层或蓝图分类
4. **检索** — 按功能域/状态/编号筛选蓝图
5. **追溯** — 查看规范文档清单和宪法级文件

### 2.2 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | 页面作为消费者域，只读取静态配置，不修改任何数据 |
| 函数范式 | 组件纯函数设计，状态外置Pinia Store |
| 基因图谱6层 | 6大展区一一对应6层基因 |
| 蓝图7域分类 | 蓝图卡片按7功能域分组 |
| 规范文档3类 | constitution/protocol/workflow三类文档列表 |
| 仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| UI画廊 | 作为画廊的新页面，通过画廊路由访问 |

### 2.3 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| 单页长滚动 | 6层基因连续叙事，沉浸式体验 | 多Tab切换 | 需要锚点导航和回到顶部 |
| 静态配置+静态嵌入 | 基因数据变化频率低，蓝图数据从index提取 | 动态API | 需维护genome.json |
| 6色编码 | 每层基因独立色彩标识 | 统一金色 | 视觉区分度更高 |
| 蓝图卡片复用风格 | 与画廊ExhibitCard视觉一致 | 独立设计 | 组件可复用 |
| DNA装饰元素 | 呼应"基因"主题 | 无装饰 | 增加CSS动画复杂度 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 基因图谱 | 项目6层约束体系 | 6大展区，纵向滚动 |
| 基因层 | 单层约束体系 | instrument-panel展区 |
| 基因片段 | 单个约束规则 | 展区内的卡片/图表 |
| DNA链 | 基因层之间的连接 | 侧边DNA装饰动画 |
| 蓝图基因库 | 全部蓝图资产 | 蓝图卡片网格 |
| 宪法文献 | 规范文档体系 | 文档列表 |

### 3.2 基因6层与展区映射

```
┌─────────────────────────────────────────────────────────────┐
│                  基因图谱 (Genome Viewer)                     │
│                                                              │
│  ┌─── 哲学基因 (Philosophy) ─── 金色 #D4A843 ──────────────┐ │
│  │  三界隔离架构 │ 函数范式铁律 │ 零原则体系               │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链                              │
│  ┌─── 授权基因 (Authorization) ─ 翡翠绿 #00C9A7 ──────────┐ │
│  │  五字段结构 │ 七步验证 │ 七维审查                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链                              │
│  ┌─── 运行时基因 (Runtime) ─ 蓝色 #3B82F6 ────────────────┐ │
│  │  SHM六状态机 │ 内存布局 │ 门禁校验                      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链                              │
│  ┌─── 架构基因 (Architecture) ─ 橙色 #F59E0B ─────────────┐ │
│  │  三进程架构 │ Queue通讯 │ Nginx钢铁脊椎                 │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链                              │
│  ┌─── 质量基因 (Quality) ─ 紫色 #8B5CF6 ─────────────────┐ │
│  │  测试分层 │ 异常分类 │ 4阶段处置                        │ │
│  └──────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链                              │
│  ┌─── 演进基因 (Evolution) ─ 红色 #EF4444 ───────────────┐ │
│  │  七阶段里程碑 │ 函数注册表 │ 核心价值观                  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─── 蓝图基因库 (Blueprint Genome) ──────────────────────┐ │
│  │  7功能域分组 │ 50+蓝图卡片 │ 状态筛选                    │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─── 宪法文献 (Constitution) ────────────────────────────┐ │
│  │  宪章23份 │ 协议10份 │ 流程8份                           │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 数据流向

```
┌─────────────────┐   静态配置    ┌─────────────────┐
│  genome.json    │ ────────────▶ │  Genome Viewer  │
│  (基因+蓝图数据) │   初始化加载  │    消费者        │
└─────────────────┘              │                 │
                                 │  - 6层基因渲染   │
                                 │  - 蓝图卡片网格   │
                                 │  - 文档列表       │
                                 │  - 锚点导航       │
                                 └─────────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 静态配置 (genome.json)

```typescript
interface GenomeConfig {
  version: string
  updated_at: string
  gene_layers: GeneLayer[]
  blueprints: BlueprintEntry[]
  documents: DocumentSection[]
}

interface GeneLayer {
  id: string
  name: string
  name_zh: string
  color: string
  icon: string
  sort_order: number
  sections: GeneSection[]
}

interface GeneSection {
  id: string
  title: string
  title_zh: string
  type: 'diagram' | 'table' | 'list' | 'stat' | 'milestone'
  content: Record<string, any>
}

interface BlueprintEntry {
  id: string
  name: string
  domain: string
  functions: string
  status: 'ACTIVE' | 'CODE_DONE' | 'DRAFT' | 'PENDING' | 'DONE'
  blueprint_id: string
}

interface DocumentSection {
  category: string
  category_zh: string
  documents: DocumentEntry[]
}

interface DocumentEntry {
  filename: string
  title: string
  title_zh: string
  path: string
}
```

### 4.2 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | LOADING | 加载genome.json |
| LOADING | 配置加载成功 | READY | 渲染基因图谱 |
| LOADING | 配置加载失败 | ERROR | 显示错误提示 |
| READY | 用户点击锚点导航 | SCROLLING | 平滑滚动到目标展区 |
| READY | 用户筛选蓝图 | FILTERING | 重新渲染蓝图卡片 |
| FILTERING | 过滤完成 | READY | 显示筛选结果 |

---

## 五、UI设计规范

### 5.1 色彩体系

复用创世纪项目统一视觉语言，6层基因各配独立色彩：

| 角色 | 色值 | 用途 |
|------|------|------|
| 深空黑 | `#0A0E17` | 主背景色 |
| 暗夜灰 | `#141B2D` | 卡片/面板背景 |
| 炭灰色 | `#1E2A3A` | 边框、分割线 |
| 哲学金 | `#D4A843` | 哲学基因层 |
| 授权绿 | `#00C9A7` | 授权基因层 |
| 运行时蓝 | `#3B82F6` | 运行时基因层 |
| 架构橙 | `#F59E0B` | 架构基因层 |
| 质量紫 | `#8B5CF6` | 质量基因层 |
| 演进红 | `#EF4444` | 演进基因层 |
| 月光白 | `#E8ECF1` | 主文本色 |
| 银灰色 | `#8892A4` | 次要文本 |

### 5.2 页面整体布局

```
┌──────────────────────────────────────────────────────────────┐
│  GENESIS GENOME    [哲学▼] [授权▼] [运行时▼] ... [蓝图▼]     │  ← 固定导航栏 (48px)
│  ═══════════════════════════════════════════════════════════  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─── GENESIS GENOME ─────────────────────────────────────┐ │
│  │  创世纪项目基因图谱                                     │ │  ← Hero区 (200px)
│  │  6 Layers · 50+ Blueprints · 41 Constitutions           │ │
│  │  DNA双螺旋装饰动画                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─── ◈ 哲学基因 PHILOSOPHY ─── #D4A843 ─────────────────┐ │
│  │                                                        │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │ │
│  │  │ 三界隔离架构  │  │ 函数范式铁律  │  │ 零原则体系   │ │ │  ← GeneSection卡片
│  │  │ [架构图]      │  │ [6禁令表格]   │  │ [3原则图]    │ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链装饰                          │
│  ┌─── ◉ 授权基因 AUTHORIZATION ─ #00C9A7 ────────────────┐ │
│  │  ...                                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                         ↓ DNA链装饰                          │
│  ... (运行时/架构/质量/演进) ...                              │
│                                                              │
│  ┌─── 蓝图基因库 BLUEPRINT GENOME ────────────────────────┐ │
│  │  [全部▼] [producer▼] [consumer▼] [mall▼] ... [状态▼]   │ │  ← 筛选栏
│  │                                                        │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             │ │
│  │  │BP-0001   │  │BP-0002   │  │BP-0003   │             │ │  ← BlueprintCard
│  │  │键盘生产者 │  │Console   │  │SHM矢量   │             │ │
│  │  │producer  │  │consumer  │  │shm       │             │ │
│  │  │●ACTIVE   │  │●ACTIVE   │  │●ACTIVE   │             │ │
│  │  └──────────┘  └──────────┘  └──────────┘             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─── 宪法文献 CONSTITUTIONS ─────────────────────────────┐ │
│  │  ┌── 宪章/总纲 (23) ──────────────────────────────────┐│ │
│  │  │  genetic-code.md  function-paradigm.md  ...         ││ │
│  │  └────────────────────────────────────────────────────┘│ │
│  │  ┌── 协议规范 (10) ───────────────────────────────────┐│ │
│  │  │  01-authorization-document-spec.md  ...             ││ │
│  │  └────────────────────────────────────────────────────┘│ │
│  │  ┌── 流程规范 (8) ────────────────────────────────────┐│ │
│  │  │  auth-acceptance.md  blueprint-submission.md  ...   ││ │
│  │  └────────────────────────────────────────────────────┘│ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ● 6 Gene Layers | 50+ Blueprints | 41 Documents | BP-0057 │  ← StatusBar
└──────────────────────────────────────────────────────────────┘
```

### 5.3 Hero区设计

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│              ╔══╗                                            │
│             ╔╝  ╚╗       GENESIS GENOME                     │
│            ╔╝    ╚╗      创世纪项目基因图谱                   │
│           ╔╝  ◈  ╚╗                                          │
│          ╔╝  ◉ ⬡ ╚╗     6 Layers · 50+ Blueprints           │
│         ╔╝  ◎ ◈ ◉ ╚╗    41 Constitutions · 1235 Tests       │
│          ╚╗  ⬡ ◎ ╔╝                                          │
│           ╚╗ ◉ ◈╔╝       [▼ EXPLORE GENOME]                 │
│            ╚╗  ╔╝                                            │
│             ╚══╝                                             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

DNA双螺旋用CSS动画实现，6个基因层图标（◈◉⬡◎）在螺旋上缓慢旋转。

### 5.4 基因展区设计 (GeneSection)

每个基因展区使用instrument-panel风格：

```
┌─── ◈ 哲学基因 PHILOSOPHY ─── #D4A843 ─────────────────────┐
│  ═══════════════════════════════════════════════════════════ │  ← 2px层色装饰线
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌── 三界隔离架构 ────────────────────────────────────────┐ │
│  │  instrument-panel                                      │ │
│  │  ┌──────────┬──────────┬──────────┐                   │ │
│  │  │ 生产者域  │ AICoder域 │ 商场域   │                   │ │  ← diagram类型
│  │  │ Worker   │ 外置审查  │ 主进程   │                   │ │
│  │  │ ● 输入   │ ● 输入   │ ● 输入   │                   │ │
│  │  │ ● 输出   │ ● 输出   │ ● 输出   │                   │ │
│  │  │ ✕ 禁区   │ ✕ 禁区   │ ✕ 禁区   │                   │ │
│  │  └──────────┴──────────┴──────────┘                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌── 函数范式铁律 ────────────────────────────────────────┐ │
│  │  instrument-panel                                      │ │
│  │  | 禁令     | 原因         | 替代方案     |            │ │  ← table类型
│  │  |----------|-------------|-------------|            │ │
│  │  | 禁止类定义 | 编译权私有化 | 纯函数       |            │ │
│  │  | 禁止继承链 | 封建血统论   | 函数组合     |            │ │
│  │  | ...      | ...         | ...         |            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.5 蓝图卡片设计 (BlueprintCard)

```
┌─────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│  ← 2px域色装饰线
├─────────────────────────────────────┤
│  BP-0001                            │  ← 蓝图编号 (层色, 14px, 700)
│  键盘生产者                          │  ← 中文名 (月光白, 12px)
│  Keyboard Echo                      │  ← 英文名 (银灰色, 9px)
│                                     │
│  ┌──────────┐ ┌──────────┐         │  ← 标签行
│  │producer  │ │keyboard  │         │     instrument-btn风格
│  └──────────┘ └──────────┘         │
│                                     │
│  keyboard_echo                      │  ← 函数名 (digital-readout, 10px)
│                                     │
│  ● ACTIVE                           │  ← 状态LED + 文本
└─────────────────────────────────────┘
```

**域色映射**:

| 域 | 色值 | 说明 |
|----|------|------|
| producer | `#00C9A7` | 翡翠绿 |
| consumer | `#3B82F6` | 蓝色 |
| mall | `#D4A843` | 皇室金 |
| shm | `#8B5CF6` | 紫色 |
| scheduler | `#F59E0B` | 橙色 |
| security | `#EF4444` | 红色 |
| audit | `#8892A4` | 银灰色 |

### 5.6 宪法文献列表设计

```
┌─── 宪章/总纲 CONSTITUTIONS ─ 23 items ──────────────────────┐
│  ═══════════════════════════════════════════════════════════ │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌ genetic-code.md ────────────────────────────────────────┐ │
│  │  项目基因提取              新Session必读                  │ │  ← instrument-btn行
│  └─────────────────────────────────────────────────────────┘ │
│  ┌ function-paradigm.md ──────────────────────────────────┐ │
│  │  函数范式铁律              零OOP/零全局/零堆             │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ...                                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5.7 固定导航栏

```
┌──────────────────────────────────────────────────────────────┐
│  GENESIS GENOME  │ 哲学 │ 授权 │ 运行时 │ 架构 │ 质量 │ 演进 │ 蓝图 │ 文献 │
│                  └── 点击锚点平滑滚动到对应展区 ──┘           │
└──────────────────────────────────────────────────────────────┘
```

导航项使用instrument-btn风格，当前可视展区对应的按钮高亮（active态）。

---

## 六、展品内容（基因6层详细内容）

### 6.1 哲学基因 (Philosophy) — #D4A843

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| phil-isolation | 三界隔离架构 | diagram | 生产者/AICoder/商场三域隔离图，输入/输出/禁区 |
| phil-paradigm | 函数范式铁律 | table | 6禁令表格（禁止类/继承/私有/多态/副作用/网络DB） |
| phil-zero | 零原则体系 | diagram | 三大零原则（零全局/零堆/零系统调用） |

### 6.2 授权基因 (Authorization) — #00C9A7

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| auth-5field | 五字段授权文件 | diagram | auth_header/design_meta/source_code/compiled_binary/verification_report |
| auth-7step | 七步验证流程 | list | Step1格式→Step2签名→Step3哈希→Step4静态→Step5沙箱→Step6回归→Step7注册 |
| auth-7dim | 七维审查清单 | table | D1形式/D2安全/D3语义/D4性能/D5测试/D6编译/D7生态 |

### 6.3 运行时基因 (Runtime) — #3B82F6

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| rt-statemachine | SHM矢量六状态机 | diagram | UNBORN→ACTIVE→SEALED→ARCHIVED/CORRUPTED/VOID |
| rt-memory | 矢量内存布局 | diagram | 82B头部+N数据体+16B尾部 |
| rt-gate | 门禁校验机制 | list | 存在性/封印/损坏/销毁/权限5步校验 |

### 6.4 架构基因 (Architecture) — #F59E0B

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| arch-3proc | 三进程架构 | diagram | Manager+MallCore+FastAPI三进程+Queue |
| arch-queue | Queue通讯协议 | table | 请求格式/响应格式/Queue参数 |
| arch-nginx | Nginx钢铁脊椎 | diagram | 浏览器→Nginx→FastAPI→商场本体 |

### 6.5 质量基因 (Quality) — #8B5CF6

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| qual-test | 测试分层统计 | stat | 1227+单元/57联合/4场景/10+压力边界 |
| qual-exception | 异常分类体系 | table | AICODER/MALL_AUTH/MALL_RUNTIME/FUNCTION四阶段 |
| qual-handling | 4阶段处置流程 | diagram | REJECT→SILENT_DROP→REVOKE_HANDLE→FALLBACK |

### 6.6 演进基因 (Evolution) — #EF4444

| 片段ID | 标题 | 类型 | 内容 |
|--------|------|------|------|
| evo-milestone | 七阶段里程碑 | milestone | 阶段一~六✅，阶段七🔲 |
| evo-registry | 函数注册表 | stat | 38个已注册函数 |
| evo-values | 核心价值观 | list | 透明即信任/授权即法律/隔离即安全/函数即原子/SHM即血脉/进化即生长 |

---

## 七、技术架构

### 7.1 技术栈

复用ui-gallery项目现有技术栈，作为画廊的新路由页面：

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | Vue 3 | 与画廊统一 |
| 开发语言 | TypeScript | 类型安全 |
| 构建工具 | Vite | 画廊项目内 |
| UI样式 | Tailwind CSS + instrument体系 | 复用 |
| 状态管理 | Pinia | 可选，数据量小 |

### 7.2 项目目录结构（新增文件）

```
ui-gallery/src/
├── views/
│   └── GenomeView.vue            # 基因图谱页面（新增）
├── components/
│   ├── GenomeHero.vue            # Hero区（DNA动画+统计）
│   ├── GeneSection.vue           # 基因展区（层色+片段卡片）
│   ├── GeneDiagram.vue           # 基因图表组件（diagram/table/list/stat/milestone）
│   ├── BlueprintCard.vue         # 蓝图卡片（域色+编号+函数名+状态）
│   ├── BlueprintFilter.vue       # 蓝图筛选栏（域+状态）
│   ├── DocumentList.vue          # 文献列表（分类折叠）
│   └── GenomeNavBar.vue          # 固定导航栏（锚点+高亮）
├── stores/
│   └── genomeStore.ts            # 基因数据Store（可选）
├── types/
│   └── genome.ts                 # 基因类型定义
└── public/
    └── genome.json               # 基因+蓝图+文档配置数据
```

### 7.3 路由设计

| 路径 | 组件 | 说明 |
|------|------|------|
| `/` | GalleryView | 画廊首页（现有） |
| `/preview/:exhibitId` | PreviewView | 预览页（现有） |
| `/genome` | GenomeView | 基因图谱页（新增） |

**路由守卫**: `beforeEach` — `/genome` 路由设置 `document.title = "Genesis Genome"`

### 7.4 genome.json 配置格式

```json
{
  "version": "1.0.0",
  "updated_at": "2026-05-15",
  "gene_layers": [
    {
      "id": "philosophy",
      "name": "Philosophy",
      "name_zh": "哲学基因",
      "color": "#D4A843",
      "icon": "◈",
      "sort_order": 1,
      "sections": [
        {
          "id": "phil-isolation",
          "title": "Three-Realm Isolation",
          "title_zh": "三界隔离架构",
          "type": "diagram",
          "content": {
            "columns": ["生产者域", "AICoder域", "商场域"],
            "rows": [
              {"label": "身份", "values": ["Worker/探针", "外置审查节点", "主进程/编译器"]},
              {"label": "输入", "values": ["契约蓝图", "蓝图", "授权文件"]},
              {"label": "输出", "values": ["蓝图文件", "授权文件", "函数句柄"]},
              {"label": "禁区", "values": ["不提交代码", "不入场通信", "不修改语义"]}
            ]
          }
        }
      ]
    }
  ],
  "blueprints": [
    {
      "id": "BP-0001",
      "name": "键盘生产者",
      "name_en": "Keyboard Echo",
      "domain": "producer",
      "functions": "keyboard_echo",
      "status": "ACTIVE",
      "blueprint_id": "BP-0001-ORIGIN"
    }
  ],
  "documents": [
    {
      "category": "constitution",
      "category_zh": "宪章/总纲",
      "documents": [
        {"filename": "genetic-code.md", "title": "Project Genetic Code", "title_zh": "项目基因提取", "path": "design/specs/constitution/genetic-code.md"},
        {"filename": "function-paradigm.md", "title": "Function Paradigm", "title_zh": "函数范式铁律", "path": "design/specs/constitution/function-paradigm.md"}
      ]
    }
  ]
}
```

---

## 八、组件规范

### 8.1 GenomeView

**布局**: pt-14 pb-8 min-h-screen grid-bg（与GalleryView一致）

**结构**:
1. **GenomeNavBar**: 固定顶部锚点导航
2. **GenomeHero**: Hero区（DNA动画+统计数字）
3. **GeneSection × 6**: 6层基因展区
4. **BlueprintFilter + BlueprintCard网格**: 蓝图基因库
5. **DocumentList × 3**: 宪法文献（constitution/protocol/workflow）

**生命周期**: `onMounted` 加载genome.json

### 8.2 GenomeHero

**布局**: h-[200px] + 居中 + DNA双螺旋CSS动画背景

**内容**: "GENESIS GENOME" (text-3xl text-royal-gold) + "创世纪项目基因图谱" (text-silver-gray) + 统计数字（6 Layers · 50+ Blueprints · 41 Constitutions）

**DNA动画**: 纯CSS实现，两条正弦曲线 + 6个基因图标在曲线上浮动

### 8.3 GeneSection

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| layer | GeneLayer | 基因层数据 |

**布局**: instrument-panel + 2px层色装饰线（::before使用层色替代金色）+ instrument-header（层色文字）

**内容**: 遍历layer.sections，每个section渲染GeneDiagram

### 8.4 GeneDiagram

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| section | GeneSection | 基因片段数据 |
| layerColor | string | 层色 |

**按type渲染**:

| type | 渲染方式 |
|------|----------|
| diagram | 三列/多列对比表格（instrument-panel子面板） |
| table | 标准表格（表头+行，层色表头） |
| list | 有序列表（数字+文本，层色数字） |
| stat | 大数字+标签（digital-readout风格） |
| milestone | 时间轴/进度条（✅完成/🔲待实现） |

### 8.5 BlueprintCard

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| blueprint | BlueprintEntry | 蓝图数据 |

**布局**: instrument-panel + 2px域色装饰线 + 蓝图编号(层色) + 中文名 + 英文名 + 域标签 + 函数名(digital-readout) + 状态LED

**状态LED映射**: ACTIVE→led green, CODE_DONE→led green, DONE→led green, DRAFT→led yellow, PENDING→led yellow

### 8.6 BlueprintFilter

**功能**: 域下拉(7域+全部) + 状态下拉(5状态+全部)

**样式**: instrument-btn风格下拉，与FilterBar一致

### 8.7 DocumentList

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| section | DocumentSection | 文档分类数据 |

**布局**: instrument-panel + instrument-header(分类名+数量) + 文档行列表

**文档行**: instrument-btn风格，文件名(月光白) + 中文标题(银灰色)

### 8.8 GenomeNavBar

**布局**: 固定顶部，h-12，bg-night-gray/95 + backdrop-blur，z-40

**内容**: "GENESIS GENOME" + 8个锚点按钮（6基因层+蓝图+文献）

**高亮逻辑**: IntersectionObserver检测当前可视展区，对应按钮active态

---

## 九、安全约束

### 9.1 数据安全

| 约束 | 实现 |
|------|------|
| 只读 | 页面只读取genome.json，不修改任何数据 |
| 无API调用 | 纯静态配置，不调用后端API |
| 无敏感数据 | 基因/蓝图/文档均为项目设计文档，不含密钥 |

---

## 十、实现状态追踪

### 10.1 模块实现状态

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 基因图谱页 | GenomeView.vue | ❌ 未实现 | 主页面 |
| Hero区 | GenomeHero.vue | ❌ 未实现 | DNA动画+统计 |
| 基因展区 | GeneSection.vue | ❌ 未实现 | 层色+片段 |
| 基因图表 | GeneDiagram.vue | ❌ 未实现 | 5种类型渲染 |
| 蓝图卡片 | BlueprintCard.vue | ❌ 未实现 | 域色+编号+状态 |
| 蓝图筛选 | BlueprintFilter.vue | ❌ 未实现 | 域+状态筛选 |
| 文献列表 | DocumentList.vue | ❌ 未实现 | 分类折叠 |
| 导航栏 | GenomeNavBar.vue | ❌ 未实现 | 锚点+高亮 |
| 配置数据 | genome.json | ❌ 未实现 | 基因+蓝图+文档 |
| 类型定义 | types/genome.ts | ❌ 未实现 | GeneLayer/Section/Blueprint/Document |
| 路由 | router.ts | ❌ 未实现 | 新增/genome路由 |

---

## 十一、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 数据静态 | genome.json需手动维护 | 后续可从design/目录自动生成 |
| 文档不可点击 | 文献列表仅展示元数据，不链接实际文件 | 后续可集成文件预览 |
| DNA动画简化 | CSS实现基础双螺旋，非3D | 可升级为Three.js |

---

## 十二、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-15 | 初始版本，6层基因展区+蓝图基因库+宪法文献，单页长滚动 |

---

## 附录

### A. 参考文档

- [genetic-code.md](../../specs/constitution/genetic-code.md) - 项目基因提取
- [blueprint-index.md](../blueprint-index.md) - 蓝图索引
- [ui-blueprint-overview.md](../../specs/ui-blueprint-overview.md) - UI蓝图总览
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图
- [PROJECT_CONTEXT.md](../../PROJECT_CONTEXT.md) - 项目上下文

### B. 术语表

| 术语 | 说明 |
|------|------|
| Genome | 基因图谱，项目约束体系的可视化展示 |
| Gene Layer | 基因层，6层约束体系之一 |
| Gene Section | 基因片段，单层内的具体约束规则 |
| Blueprint Genome | 蓝图基因库，全部蓝图资产的可视化 |
| Constitution | 宪法文献，规范文档体系 |
