# 创世纪项目 UI 蓝图总览

> **版本**: 1.0  
> **日期**: 2026-05-13  
> **用途**: 汇总项目所有 UI 相关蓝图与界面资产，提供全局视角与快速导航  
> **范围**: 消费者域 UI 蓝图、静态页面、嵌入式 UI 组件、瀑布流可视化

---

## 一、UI 体系架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        用户界面层                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  入口页面    │  │  操作台页面  │  │  任务台页面  │             │
│  │  index.html │  │dashboard.html│  │user_tasks   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  调试控制台  │  │  频谱仪 UI  │  │  基因瀑布流  │             │
│  │  debug.html │  │ spectrum-ui │  │  waterfall   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              消费者相构建系统 (BP-0035)                │       │
│  │  用户画像 → 消费者集合 → 消费者相模板 → AICoder编译   │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**设计原则**：
- 所有 UI 页面通过 Nginx 钢铁脊椎提供，不可直接访问 FastAPI Worker
- 统一视觉语言：深色主题（`--bg-deep: #060D18`）、翡翠绿主色（`--accent: #00E5A0`）
- 消费者范式：UI 组件本质是消费者，从 SHM 读取数据、向 stdout/DOM 渲染输出

---

## 二、蓝图清单

### 2.1 终端显示类

| 蓝图ID | 名称 | 状态 | 简介 |
|--------|------|------|------|
| BP-0002 | Console 消费者 | ACTIVE | 商场生态末端节点，将商场分发的数据转化为终端可读输出。接收 SHM/队列数据，格式化渲染至 stdout |
| BP-0032 | 系统状态显示消费者 | ACTIVE | 订阅 BP-0031 系统信息采集生产者的 SHM 数据，格式化为终端文本面板：CPU/内存/磁盘/网络/进程六区域布局，含进度条与单位自动转换 |

### 2.2 用户交互类

| 蓝图ID | 名称 | 状态 | 简介 |
|--------|------|------|------|
| BP-0029 | 用户主页定制 | ACTIVE | 根据用户拥有的消费者列表，动态生成个性化主页布局。以"相"的形式展示每个消费者（波形/表格/表单/控制台/图表五类），支持多列网格排列 |
| BP-0035 | 用户画像与消费者相构建器 | DRAFT | 用户 ≠ 消费者的概念体系。用户是消费者集合的持有者和编排者。提供用户画像数据结构、消费者相模板库（终端/分析/文件/控制/容器五型）、八步构建流程（创建身份→选择相→定制→提交蓝图→AICoder编译→用户审批→迭代→归档） |

### 2.3 任务操控类

| 蓝图ID | 名称 | 状态 | 简介 |
|--------|------|------|------|
| BP-0030 | 任务发布控制 | ACTIVE | 用户操控任务发布的三种模式：手动发布（即时触发）、序列发布（按序多任务）、定时发布（时间计划自动触发）。发布者必须为已认证用户，目标必须为用户拥有的消费者 |
| BP-0031 | 用户任务编辑器 | DRAFT | 用户意图到消费者执行的"编译器"。将任务编译为交易序列（Worker→商场→消费者的有序操作链），支持条件控制（继续/等待/终止/分支）。核心概念：任务→子任务→交易→条件控制→产品 |
| BP-0028 | 消费者任务参数化调整 | DRAFT | 消费者层对任务的参数化调整能力：重复执行模式（n次/无限/条件满足）、定时运行模式（cron/延迟/固定间隔）、条件运行模式（SHM数据/任务结果/外部事件）。支持混合模式与运行中动态调整 |
| BP-0053 | 用户任务生成与静态配置 | DRAFT | 消费者层任务生成与静态配置系统。支持子任务 DAG 逻辑链编排、交易链异常处理、任务静态配置（个性数据+交易参数）、静态配置序列（批量执行）、资源冲突检测（SHM路径+算力资源） |

### 2.4 专业仪器类

| 蓝图ID | 名称 | 状态 | 简介 |
|--------|------|------|------|
| BP-0034 | 嵌入式精简频谱仪 UI | CODE_DONE | 商场内核的"相"呈现载体，将 SHM 推送的 FFT 数据渲染为频谱可视化界面。支持触摸/按键/编码器三模输入，含顶部状态条、波形主显区（网格+迹线+Marker）、颜色标尺、右侧7键+底部5键面板。Vue 3 + TypeScript 实现 |
| BP-0054 | TR测试工程师UI | DRAFT | 专业TR测试工程师多仪器集成界面。6仪器同屏（频谱/波形/功率/信号源/脉冲/电源），CSS Grid 4布局模式（单/双/四/六），Chat指令引擎（9指令类型），数据表格+CSV导出，视图联动，4场景预设。纯HTML/CSS/JS实现 |
| BP-0055 | 管理员画像系统UI | CODE_DONE | 管理员画像可视化界面，嵌入式仪器风格。5页面（画像/总线/引擎/风控/命令），8组件，6 Pinia Store，instrument-panel/header/btn/LED/digital-readout全局样式体系。Vue 3 + TypeScript + Pinia 实现 |

### 2.5 UI画廊类

| 蓝图ID | 名称 | 状态 | 简介 |
|--------|------|------|------|
| BP-0056 | 创世纪UI画廊 | CODE_DONE | 统一UI资产展厅入口，8展品4展厅（静态页面/嵌入式UI/专业仪器/可视化），iframe弹窗+全屏双模式预览，instrument-panel卡片风格，gallery.json静态配置，serveStaticPlugin Vite代理，Vue SPA base path适配，Nginx反向代理统一访问 |
| BP-0061 | 基因图谱页面 | DRAFT | 项目基因图谱可视化，6层基因展区（哲学/授权/运行时/架构/质量/演进）6色编码，蓝图基因库7域分组卡片，宪法文献3类列表，单页长滚动+锚点导航，DNA双螺旋装饰，genome.json静态配置 |
| BP-0062 | 时频IQ三维融合 | CODE_DONE | 四面板融合可视化（时域波形+瀑布图+星座图+参数面板），Canvas 2D渲染，5种色映射，QPSK/QAM16/BPSK Demo数据，LTTB降采样，instrument风格 |

---

## 三、静态页面资产

### 3.1 页面清单

| 文件 | 页面名称 | 简介 |
|------|----------|------|
| [index.html](file:///d:\创世纪\static\index.html) | 消费者登录页 | 系统入口，深色主题登录表单，背景网格+光晕效果 |
| [dashboard.html](file:///d:\创世纪\static\dashboard.html) | 消费者操作台 | 主操作界面，顶部导航栏，展示系统状态与消费者控制 |
| [user_tasks.html](file:///d:\创世纪\static\user_tasks.html) | 用户任务台 | 任务管理界面，任务列表、状态追踪、控制台输出 |
| [debug.html](file:///d:\创世纪\static\debug.html) | 调试控制台 | 开发调试界面，测试执行、覆盖率查看、健康检查、性能指标 |

### 3.2 统一视觉规范

```css
:root {
    --bg-deep: #060D18;       /* 深层背景 */
    --bg-card: #0D1B2A;       /* 卡片背景 */
    --bg-input: #1B2838;      /* 输入框背景 */
    --accent: #00E5A0;        /* 翡翠绿主色 */
    --accent-dim: #00B87D;    /* 主色暗调 */
    --accent-glow: rgba(0,229,160,0.15);  /* 主色光晕 */
    --text-primary: #E8ECF1;  /* 主文字 */
    --text-secondary: #7A8BA0;/* 次文字 */
    --border: #1E2D3D;        /* 边框 */
    --danger: #FF5C5C;        /* 危险色 */
    --warning: #FFB347;       /* 警告色 */
}
```

---

## 四、嵌入式 UI 组件

### 4.1 频谱仪 UI（spectrum-ui）

| 属性 | 值 |
|------|-----|
| 路径 | [spectrum-ui/](file:///d:\创世纪\spectrum-ui) |
| 技术栈 | Vue 3 + TypeScript + Vite |
| 蓝图 | BP-0034-SPECTRUM-UI |
| 测试 | 24项通过 |
| 状态 | CODE_DONE |

**核心函数**：
- `spectrum_ui_init` — 初始化 UI，创建 Canvas
- `spectrum_ui_tick` — 主渲染循环，30fps+
- `spectrum_ui_on_button` — 按键事件处理（R1-R7, B1-B5）
- `spectrum_ui_on_encoder` — 旋转编码器输入
- `spectrum_ui_on_fft_frame` — FFT 数据帧渲染
- `spectrum_ui_on_status_update` — 仪器状态更新
- `spectrum_ui_on_color_map` — 颜色映射 LUT 更新

**界面布局**：
```
┌────────────────────────────────────┬──────────┐
│ [顶部状态条: CF/Span/RL/Atten...]  │          │  10%
├────────────────────────────────────┼──────────┤
│ [颜色标尺]                         │ [R1] 运行 │
│      频谱波形主显示区               │ [R2] 单次 │  62%
│      (Grid + Trace + Marker)       │ [R3-R7]  │
├────────────────────────────────────┴──────────┤
│ [底部信息条: CF │ RBW │ VBW │ Span]           │   6%
├────────────────────────────────────────────────┤
│ [B1]  [B2]  [B3]  [B4]  [B5]                  │  22%
└────────────────────────────────────────────────┘
```

### 4.2 基因瀑布流（mall-daily-waterfall）

| 属性 | 值 |
|------|-----|
| 路径 | [mall-daily-waterfall/](file:///d:\创世纪\mall-daily-waterfall) |
| 技术栈 | 纯 HTML + CSS + JavaScript |
| 功能 | P0-P3 全路径实现，六境卡片瀑布流展示 |

**六境阶段**：
1. **Genesis**（创世）— 商场本体诞生
2. **Forge**（锻造）— 授权文件锻造
3. **Covenant**（契约）— 蓝图契约签订
4. **Manifestation**（显现）— 函数激活运行
5. **Stroll**（巡游）— 商场自治运行
6. **Archive**（归档）— 历史沉淀归档

**特色功能**：
- 滚动叙事引擎（narrative.js）
- Web Audio API 音效引擎（audio.js）
- 视差系统（parallax.js）
- 安全生命周期管理（security.js）
- 粒子 Canvas 背景

---

## 五、UI 设计文档资产

### 5.1 消费者注册 UI 设计（UI-Design-Consumer-Registration-V1.0）

| 文档 | 内容 |
|------|------|
| 01-设计说明文档 | 设计目标与风格定义 |
| 02-信息架构图 | 页面结构与导航关系 |
| 03-高保真视觉稿 | 像素级设计稿 |
| 04-组件规格表 | UI 组件参数与状态 |
| 05-交互状态机 | 交互流程状态转换 |
| 06-CSS变量表 | 设计令牌定义 |
| 07-交付检查表 | 验收标准 |
| 08-响应式适配规则 | 多端适配方案 |

### 5.2 波形显示 UI 设计（UI-Design-Waveform-V1.0）

| 文档 | 内容 |
|------|------|
| 01-设计说明文档 | 波形显示设计原则 |
| 02-信息架构图 | 波形页面结构 |
| 03-高保真视觉稿 | 波形渲染视觉稿 |
| 04-组件规格表 | 波形组件参数 |
| 05-交互状态机 | 波形交互流程 |
| 06-CSS变量表 | 波形专用设计令牌 |
| 07-交付检查表 | 验收标准 |
| 08-SHM绑定规范 | 数据源 SHM 绑定规则 |

---

## 六、UI 蓝图关系图

```
                        ┌──────────────────┐
                        │   用户 (User)     │
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │  BP-0056 UI画廊   │  ← 统一入口
                        └────────┬─────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ BP-0029  │ │ BP-0035  │ │ BP-0030  │
            │ 用户主页 │ │ 用户画像 │ │ 任务发布 │
            └────┬─────┘ └────┬─────┘ └────┬─────┘
                 │            │            │
                 │     ┌──────┴──────┐     │
                 │     │ 消费者相     │     │
                 │     │ 模板库       │     │
                 │     └──────┬──────┘     │
                 │            │            │
         ┌───────┴────┐      │      ┌─────┴─────┐
         │            │      │      │           │
         ▼            ▼      ▼      ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ BP-0002  │ │ BP-0032  │ │ BP-0034  │ │ BP-0031  │
   │ Console  │ │ 系统显示 │ │ 频谱仪UI │ │ 任务编辑 │
   └──────────┘ └──────────┘ └──────────┘ └──────────┘
                                       │
                    ┌──────────────────┤
                    │                  │
                    ▼                  ▼
             ┌──────────┐       ┌──────────┐
             │ BP-0054  │       │ BP-0055  │
             │TR测试UI  │       │管理员UI  │
             └──────────┘       └──────────┘
                                       │
                                       ▼
                                ┌──────────┐
                                │ BP-0028  │
                                │ 任务参数 │
                                └──────────┘
                                       │
                                       ▼
                                ┌──────────┐
                                │ BP-0053  │
                                │ 任务配置 │
                                └──────────┘
```

**数据流方向**：用户 → UI画廊(统一入口) → 主页/画像 → 消费者相选择 → 任务发布/编辑 → 参数化配置 → 消费者执行 → 数据渲染

---

## 七、实现状态统计

| 类别 | 总数 | ACTIVE | CODE_DONE | DRAFT |
|------|------|--------|-----------|-------|
| 终端显示类 | 2 | 2 | 0 | 0 |
| 用户交互类 | 2 | 1 | 0 | 1 |
| 任务操控类 | 4 | 1 | 0 | 3 |
| 专业仪器类 | 3 | 0 | 2 | 1 |
| UI画廊类 | 1 | 0 | 0 | 1 |
| **合计** | **14** | **4** | **2** | **8** |

| 页面资产 | 数量 | 状态 |
|----------|------|------|
| 静态 HTML 页面 | 4 | 已实现 |
| 嵌入式 UI 组件 | 4 | 已实现 |
| UI 设计文档集 | 2 | 已完成 |

---

## 八、蓝图文件索引

| 蓝图ID | 文件路径 |
|--------|----------|
| BP-0002 | [design/blueprints/consumer/BP-0002-console.md](file:///d:\创世纪\design\blueprints\consumer\BP-0002-console.md) |
| BP-0028 | [design/blueprints/consumer/BP-0028-task-parameterization.md](file:///d:\创世纪\design\blueprints\consumer\BP-0028-task-parameterization.md) |
| BP-0029 | [design/blueprints/mall/BP-0029-user-homepage.md](file:///d:\创世纪\design\blueprints\mall\BP-0029-user-homepage.md) |
| BP-0030 | [design/blueprints/mall/BP-0030-task-publisher.md](file:///d:\创世纪\design\blueprints\mall\BP-0030-task-publisher.md) |
| BP-0031 | [design/blueprints/mall/BP-0031-task-editor.md](file:///d:\创世纪\design\blueprints\mall\BP-0031-task-editor.md) |
| BP-0032 | [design/blueprints/consumer/BP-0032-sysdisplay.md](file:///d:\创世纪\design\blueprints\consumer\BP-0032-sysdisplay.md) |
| BP-0034 | [design/blueprints/consumer/BP-0034-spectrum-ui.md](file:///d:\创世纪\design\blueprints\consumer\BP-0034-spectrum-ui.md) |
| BP-0035 | [design/blueprints/consumer/BP-0035-user-profile.md](file:///d:\创世纪\design\blueprints\consumer\BP-0035-user-profile.md) |
| BP-0053 | [design/blueprints/consumer/BP-0053-user-task-config.md](file:///d:\创世纪\design\blueprints\consumer\BP-0053-user-task-config.md) |
| BP-0054 | [design/blueprints/consumer/BP-0054-tr-test-ui.md](file:///d:\创世纪\design\blueprints\consumer\BP-0054-tr-test-ui.md) |
| BP-0055 | [design/blueprints/consumer/BP-0055-admin-profile-ui.md](file:///d:\创世纪\design\blueprints\consumer\BP-0055-admin-profile-ui.md) |
| BP-0056 | [design/blueprints/consumer/BP-0056-ui-gallery.md](file:///d:\创世纪\design\blueprints\consumer\BP-0056-ui-gallery.md) |
| BP-0061 | [design/blueprints/consumer/BP-0061-genome-viewer.md](file:///d:\创世纪\design\blueprints\consumer\BP-0061-genome-viewer.md) |
| BP-0062 | [design/blueprints/consumer/BP-0062-iq3d-fusion.md](file:///d:\创世纪\design\blueprints\consumer\BP-0062-iq3d-fusion.md) |
