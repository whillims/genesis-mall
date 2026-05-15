# 创世纪UI画廊消费者蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0056-UI-GALLERY
> **版本**: v1.2.0
> **日期**: 2026-05-14
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0056-UI-GALLERY |
| 函数名 | ui_gallery_init / ui_gallery_tick |
| 生产者名称 | UIGalleryConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | SHM矢量订阅 (VEC-DISPLAY) + 静态配置 |
| 输出协议 | SHM矢量发布 (VEC-CONTROL) |
| 安全等级 | P1 (内部工具) |
| 版本 | v1.2.0 |

---

## 二、设计意图

### 2.1 核心目标

创建创世纪项目的统一UI画廊入口，以沉浸式展厅的形式展示所有UI界面资产。用户通过画廊可以：

1. **总览** — 一目了然地浏览项目全部UI资产
2. **预览** — 在画廊内实时预览各UI的缩略图/截图
3. **访问** — 一键跳转到任意UI的完整运行实例
4. **分类** — 按功能域、技术栈、蓝图状态等维度筛选
5. **状态** — 实时显示各UI的蓝图状态、构建状态、运行状态

### 2.2 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | 画廊作为消费者域，只读取UI资产元数据，不修改任何UI |
| 函数范式 | 画廊组件纯函数设计，状态外置Pinia Store |
| SHM矢量空间 | 通过VEC-DISPLAY订阅各UI运行状态 |
| 授权文件体系 | 画廊消费者需AUTH-BP-0056授权文件激活 |
| 基因图谱风格 | 深色主题(#0A0E17) + 金色(#D4A843) + 翡翠绿(#00C9A7) |
| 嵌入式仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| Nginx钢铁脊椎 | 画廊由Nginx直接服务，各UI通过Nginx反向代理访问 |
| UI实验室 | 画廊作为UI实验室的门户，实验室产出的UI自动注册到画廊 |

### 2.3 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vue 3 SPA而非纯HTML | 画廊需要动态路由、状态管理、组件复用 | 纯HTML静态页 | 需要构建流程 |
| iframe预览而非新窗口 | 画廊内直接预览，不离开画廊上下文 | 新标签页打开 | 需处理跨域和尺寸适配 |
| 卡片网格而非列表 | 视觉冲击力强，适合展示UI截图 | 列表视图 | 响应式布局更复杂 |
| 静态配置+动态发现 | UI元数据静态配置，运行状态动态获取 | 纯动态发现 | 需维护配置文件 |
| 画廊置于ui-lab内 | 画廊是UI实验室的门户组件 | 独立项目 | 路径统一管理 |

### 2.4 v1.1.0 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vite自定义中间件代理 | 开发模式下需跨项目访问各UI的dist/public目录 | Vite server.proxy (不支持目录映射) | 需自定义serveStaticPlugin |
| Vue SPA设置base path | iframe中加载SPA时，资源路径需包含前缀才能被代理正确服务 | 修改iframe src为绝对URL | admin-profile-ui和spectrum-ui需设置`base`并重新构建 |
| Vue Router传入BASE_URL | SPA在画廊代理中运行时，路由base需与Vite base一致 | 硬编码base path | `createWebHistory(import.meta.env.BASE_URL)` |
| SPA路由fallback | Vue SPA子路由请求需返回index.html | 404 | 中间件检测spaPrefixes做fallback |
| 画廊内弹窗预览+独立预览页 | 双模式：GalleryView内PreviewPanel弹窗 + PreviewView全屏 | 仅弹窗或仅路由 | 两种预览体验 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 画廊 | UI资产展厅 | 主页面，展示所有UI卡片 |
| 展厅 | 功能域分类 | 按域分组的卡片区域 |
| 展品 | 单个UI界面 | 展品卡片（缩略图+元数据+状态） |
| 展品标签 | UI元数据 | 蓝图编号/技术栈/状态/路径 |
| 预览窗 | UI实时预览 | iframe嵌入的UI运行实例 |
| 导览图 | 画廊导航 | 侧边栏/顶部导航 |
| 展品目录 | UI资产清单 | 画廊配置数据源 |

### 3.2 画廊与UI资产关系

```
┌─────────────────────────────────────────────────────────────┐
│                     UI 画廊 (Gallery)                        │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 静态页面  │ │ 嵌入式UI │ │ 仪器类UI │ │ 瀑布流UI │       │
│  │  展厅     │ │  展厅     │ │  展厅     │ │  展厅     │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │            │            │            │              │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐       │
│  │index    │  │spectrum │  │tr-test  │  │waterfall│       │
│  │dashboard│  │admin    │  │         │  │         │       │
│  │tasks    │  │profile  │  │         │  │         │       │
│  │debug    │  │         │  │         │  │         │       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              预览窗 (Preview Panel)                    │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │          iframe: 目标UI运行实例                  │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 数据流向

```
┌─────────────────┐   静态配置    ┌─────────────────┐
│  gallery.json   │ ────────────▶ │  UI Gallery     │
│  (UI资产清单)    │   初始化加载  │    消费者        │
└─────────────────┘              │                 │
                                 │  - 卡片渲染      │
┌─────────────────┐   SHM矢量    │  - 状态轮询      │
│   商场本体       │ ────────────▶│  - 预览加载      │
│  (运行状态)      │  VEC-DISPLAY │  - 命令发送      │
└─────────────────┘              └─────────────────┘
                                         │
                                         ▼
                                  ┌─────────────┐
                                  │  Nginx      │
                                  │ 钢铁脊椎     │
                                  │ (反向代理)   │
                                  └─────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 静态配置 (gallery.json)

```typescript
interface GalleryConfig {
  version: string
  updated_at: string
  categories: GalleryCategory[]
  exhibits: GalleryExhibit[]
}

interface GalleryCategory {
  id: string
  name: string
  name_zh: string
  icon: string
  description: string
  sort_order: number
}

interface GalleryExhibit {
  id: string
  title: string
  title_zh: string
  description: string
  description_zh: string
  category_id: string
  blueprint_id: string | null
  tech_stack: 'html' | 'vue3' | 'react'
  status: 'active' | 'building' | 'offline' | 'draft'
  path: string
  preview_path: string | null
  thumbnail: string
  tags: string[]
  sort_order: number
  meta: {
    routes?: number
    components?: number
    tests?: number
    build_size_kb?: number
    last_build_at?: string
  }
}
```

#### 4.1.2 SHM矢量订阅 (VEC-DISPLAY)

```typescript
interface UIStatusFrame {
  frame_type: 'ui_status'
  timestamp: number
  exhibits: {
    exhibit_id: string
    is_running: boolean
    health: 'healthy' | 'degraded' | 'down'
    response_ms: number
    active_users: number
  }[]
}
```

### 4.2 输出契约 (VEC-CONTROL)

```typescript
interface GalleryControlFrame {
  frame_type: 'gallery_command'
  timestamp: number
  command: 'launch' | 'refresh' | 'navigate'
  params: {
    exhibit_id: string
    target_path?: string
  }
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 画廊启动 | LOADING | 加载gallery.json |
| LOADING | 配置加载成功 | READY | 渲染卡片网格 |
| LOADING | 配置加载失败 | ERROR | 显示错误提示 |
| READY | 用户点击展品 | PREVIEWING | 加载iframe预览 |
| PREVIEWING | 用户关闭预览 | READY | 卸载iframe |
| PREVIEWING | 用户点击"全屏访问" | NAVIGATING | 新窗口打开UI |
| NAVIGATING | 导航完成 | READY | 返回画廊 |
| READY | 筛选条件变更 | FILTERING | 重新渲染卡片 |
| FILTERING | 过滤完成 | READY | 显示筛选结果 |

---

## 五、UI设计规范

### 5.1 色彩体系

复用创世纪项目统一视觉语言：

| 色彩角色 | 色值 | 用途 |
|----------|------|------|
| 深空黑 | `#0A0E17` | 主背景色 |
| 暗夜灰 | `#141B2D` | 卡片/面板背景 |
| 炭灰色 | `#1E2A3A` | 边框、分割线 |
| 皇室金 | `#D4A843` | 主强调色、标题、装饰线 |
| 翡翠绿 | `#00C9A7` | 正常状态、活跃指示 |
| 月光白 | `#E8ECF1` | 主文本色 |
| 银灰色 | `#8892A4` | 次要文本 |
| 薄雾灰 | `#3A4556` | 占位符、禁用态 |
| 烈焰红 | `#E53E3E` | 错误状态、离线指示 |

### 5.2 画廊布局

#### 整体布局

```
┌──────────────────────────────────────────────────────────────┐
│  GENESIS UI GALLERY          [搜索框]  [筛选▼]  [视图切换]   │  ← TopBar (56px)
│  ═══════════════════════════════════════════════════════════  │  ← 2px金色装饰线
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─── 展厅: 静态页面 ────────────────────────────────────┐  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐│  │
│  │  │ 📄       │  │ 📊       │  │ 📋       │  │ 🔧     ││  │
│  │  │ 消费者   │  │ 消费者   │  │ 用户     │  │ 调试    ││  │
│  │  │ 登录页   │  │ 操作台   │  │ 任务台   │  │ 控制台  ││  │
│  │  │ HTML     │  │ HTML     │  │ HTML     │  │ HTML    ││  │
│  │  │ ●ACTIVE  │  │ ●ACTIVE  │  │ ●ACTIVE  │  │ ●ACTIVE ││  │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘│  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─── 展厅: 嵌入式UI ────────────────────────────────────┐  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐                           │  │
│  │  │ 📡       │  │ 👤       │                           │  │
│  │  │ 频谱仪   │  │ 管理员   │                           │  │
│  │  │ UI       │  │ 画像UI   │                           │  │
│  │  │ Vue 3    │  │ Vue 3    │                           │  │
│  │  │ ●DONE    │  │ ●DONE    │                           │  │
│  │  │ BP-0034  │  │ BP-0055  │                           │  │
│  │  └──────────┘  └──────────┘                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─── 展厅: 专业仪器 ────────────────────────────────────┐  │
│  │                                                        │  │
│  │  ┌──────────┐  ┌──────────┐                           │  │
│  │  │ 🔬       │  │ 🌊       │                           │  │
│  │  │ TR测试   │  │ 基因     │                           │  │
│  │  │ 工程师UI │  │ 瀑布流   │                           │  │
│  │  │ HTML/JS  │  │ HTML/JS  │                           │  │
│  │  │ ●ACTIVE  │  │ ●ACTIVE  │                           │  │
│  │  │ BP-0054  │  │  —       │                           │  │
│  │  └──────────┘  └──────────┘                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ● 8 UI Assets | 4 Active | 3 Code Done | 1 Draft          │  ← StatusBar (32px)
└──────────────────────────────────────────────────────────────┘
```

#### 预览模式布局

```
┌──────────────────────────────────────────────────────────────┐
│  GENESIS UI GALLERY          [◀ 返回画廊]  [⛶ 全屏访问]     │  ← TopBar
│  ═══════════════════════════════════════════════════════════  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─── TR测试工程师UI ────────────────────────────────────┐  │
│  │  BP-0054 | Vue 3 | ●ACTIVE | 6 routes | 24 tests    │  │  ← 展品信息栏
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │                                                      │   │
│  │              iframe: 目标UI运行实例                    │   │
│  │              (自适应缩放, transform: scale)            │   │
│  │                                                      │   │
│  │                                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ● Preview Mode | tr-test-ui | 1920x1080 | 60fps           │  ← StatusBar
└──────────────────────────────────────────────────────────────┘
```

### 5.3 展品卡片设计

#### 卡片结构 (instrument-panel)

```
┌─────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│  ← 2px金色装饰线
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │                             │   │  ← 缩略图区域 (16:9)
│  │     UI截图 / 动态预览       │   │     渐变占位符或实际截图
│  │                             │   │
│  └─────────────────────────────┘   │
│                                     │
│  TR测试工程师UI                     │  ← 标题 (月光白, 14px, 700)
│  Professional TR Test Engineer      │  ← 英文副标题 (银灰色, 10px)
│                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐       │  ← 标签行
│  │HTML/JS│ │BP-0054│ │6 inst│       │     instrument-btn 风格
│  └──────┘ └──────┘ └──────┘       │
│                                     │
│  ● ACTIVE                     [→]  │  ← 状态LED + 访问按钮
│                                     │
└─────────────────────────────────────┘
```

#### 卡片交互状态

| 状态 | 视觉效果 | 动画 |
|------|----------|------|
| 默认 | 标准显示 | - |
| 悬停 | 边框变金色，微上浮 | `transform: translateY(-4px)`, `border-color: #D4A843` |
| 激活 | 翡翠绿边框+光晕 | `box-shadow: 0 0 12px rgba(0,201,167,0.3)` |
| 离线 | 灰度显示，透明度降低 | `opacity: 0.5`, `filter: grayscale(0.5)` |
| 加载中 | 缩略图区域显示扫描线动画 | `.scanline` 动画 |

### 5.4 展厅分组设计

每个展厅是一个分类区域，使用 instrument-panel 风格：

```
┌─── 展厅标题 (instrument-header) ─────────────────────────┐
│  📡 嵌入式UI  EMBEDDED INTERFACES                   2项  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────┐  ┌──────────┐                             │
│  │ 频谱仪UI │  │ 管理员UI │                             │
│  └──────────┘  └──────────┘                             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.5 筛选与搜索

#### 筛选栏

```
┌──────────────────────────────────────────────────────────┐
│  [🔍 搜索UI名称...]  [全部▼]  [全部技术栈▼]  [全部状态▼] │
│                       类别      技术栈        状态        │
└──────────────────────────────────────────────────────────┘
```

#### 筛选维度

| 维度 | 选项 | 说明 |
|------|------|------|
| 类别 | 全部/静态页面/嵌入式UI/专业仪器/瀑布流 | 按功能域筛选 |
| 技术栈 | 全部/HTML/Vue 3/React | 按实现技术筛选 |
| 状态 | 全部/Active/Code Done/Draft/Offline | 按蓝图状态筛选 |
| 排序 | 默认/名称/最近构建/蓝图编号 | 排序方式 |

### 5.6 视图模式

| 模式 | 布局 | 说明 |
|------|------|------|
| 网格视图 | CSS Grid, 3-4列 | 默认模式，展示缩略图+完整信息 |
| 列表视图 | 单列列表 | 紧凑模式，一行一个展品 |
| 看板视图 | 按状态分列 | 类似看板，按Active/Done/Draft分列 |

---

## 六、展品清单（初始配置）

### 6.1 静态页面展厅

| 展品ID | 标题 | 路径 | 蓝图 | 技术栈 | 状态 |
|--------|------|------|------|--------|------|
| exhibit-index | 消费者登录页 | /static/index.html | - | html | active |
| exhibit-dashboard | 消费者操作台 | /static/dashboard.html | - | html | active |
| exhibit-tasks | 用户任务台 | /static/user_tasks.html | - | html | active |
| exhibit-debug | 调试控制台 | /static/debug.html | - | html | active |

### 6.2 嵌入式UI展厅

| 展品ID | 标题 | 路径 | 蓝图 | 技术栈 | 状态 |
|--------|------|------|------|--------|------|
| exhibit-spectrum | 嵌入式频谱仪UI | /spectrum-ui/ | BP-0034 | vue3 | active |
| exhibit-admin | 管理员画像系统UI | /admin-profile-ui/ | BP-0055 | vue3 | active |

### 6.3 专业仪器展厅

| 展品ID | 标题 | 路径 | 蓝图 | 技术栈 | 状态 |
|--------|------|------|------|--------|------|
| exhibit-tr-test | TR测试工程师UI | /tr-test-ui/ | BP-0054 | html | active |
| exhibit-waterfall | 基因瀑布流 | /mall-daily-waterfall/ | - | html | active |

### 6.4 展厅分类定义

| 分类ID | 名称 | 图标 | 说明 |
|--------|------|------|------|
| cat-static | 静态页面 | FileText | 系统基础页面，纯HTML实现 |
| cat-embedded | 嵌入式UI | Monitor | Vue 3 SPA，SHM矢量集成 |
| cat-instrument | 专业仪器 | Cpu | 专业测试/监控仪器界面 |
| cat-visualization | 可视化 | Waves | 数据可视化与叙事展示 |

---

## 七、技术架构

### 7.1 技术栈

| 层级 | 技术 | 版本 | 选型理由 |
|------|------|------|----------|
| 前端框架 | Vue 3 | 3.4.x | 与项目其他UI统一 |
| 开发语言 | TypeScript | 5.x | 类型安全 |
| 构建工具 | Vite | 5.x | 极速HMR |
| UI样式 | Tailwind CSS | 3.4.x | 原子化CSS |
| 状态管理 | Pinia | 2.x | Vue官方推荐 |
| 路由 | Vue Router | 4.x | 支持预览路由 |
| 图标库 | Lucide Vue | latest | 统一风格 |
| 字体 | JetBrains Mono + Roboto Mono | Google Fonts | 仪器风格 |

### 7.2 项目目录结构

```
ui-gallery/
├── public/
│   ├── gallery.json              # 展品配置数据 (8展品+4分类)
│   └── favicon.svg
├── src/
│   ├── main.ts                   # 入口 (Vue + Pinia + Router)
│   ├── App.vue                   # 根组件 (router-view)
│   ├── style.css                 # 全局instrument样式体系
│   ├── router.ts                 # 路由 (画廊首页+预览页)
│   ├── components/
│   │   ├── GalleryTopBar.vue     # 画廊顶部栏 (GENESIS UI GALLERY + 时钟 + LED)
│   │   ├── GalleryStatusBar.vue  # 画廊底部状态栏 (展品统计)
│   │   ├── ExhibitCard.vue       # 展品卡片 (缩略图+标签+LED+ACCESS按钮)
│   │   ├── HallSection.vue       # 展厅分组 (instrument-header + Grid)
│   │   ├── FilterBar.vue         # 筛选栏 (搜索+类别+技术栈+状态+视图切换)
│   │   └── PreviewPanel.vue      # 预览面板 (iframe弹窗)
│   ├── views/
│   │   ├── GalleryView.vue       # 画廊主页 (4展厅+筛选+弹窗预览)
│   │   └── PreviewView.vue       # 预览页 (全屏iframe)
│   ├── stores/
│   │   └── galleryStore.ts       # Pinia状态 (配置加载+筛选+预览)
│   └── types/
│       └── index.ts              # 类型定义 (GalleryConfig/Exhibit/Category)
├── tailwind.config.js            # 创世纪色彩体系
├── vite.config.ts                # Vite配置 + serveStaticPlugin中间件
├── tsconfig.json
├── index.html                    # JetBrains Mono + Roboto Mono字体
└── package.json
```

### 7.3 路由设计

| 路径 | 组件 | 说明 |
|------|------|------|
| `/` | GalleryView | 画廊首页，展示所有展品卡片 |
| `/preview/:exhibitId` | PreviewView | 预览页，iframe嵌入目标UI |

**路由守卫**: `beforeEach` — 根据路由名称动态设置`document.title`（画廊首页 → "Genesis UI Gallery"，预览页 → "UI Gallery - Preview"）

### 7.4 Vite开发服务器静态代理

开发模式下，Vite只服务ui-gallery自身文件。通过自定义中间件 `serveStaticPlugin` 将各UI路径映射到实际目录：

```typescript
const staticMappings: Record<string, string> = {
  '/static/': path.join(rootDir, 'static'),
  '/spectrum-ui/': path.join(rootDir, 'spectrum-ui', 'dist'),
  '/admin-profile-ui/': path.join(rootDir, 'admin-profile-ui', 'dist'),
  '/tr-test-ui/': path.join(rootDir, 'ui-lab', 'public', 'tr-test-ui'),
  '/mall-daily-waterfall/': path.join(rootDir, 'ui-lab', 'public', 'mall-daily-waterfall'),
}
```

**中间件逻辑**:
1. 请求URL匹配前缀 → 截取相对路径
2. 空路径/根路径 → 默认返回 `index.html`
3. 文件存在 → 设置MIME类型 + 流式响应
4. 文件不存在 + SPA前缀 → fallback返回 `index.html`
5. 文件不存在 + 非SPA → 返回404

**SPA路由fallback前缀**: `/spectrum-ui/`, `/admin-profile-ui/`

**MIME类型映射**: `.html`/`.css`/`.js`/`.json`/`.svg`/`.png`/`.jpg`/`.ico`/`.map`/`.woff`/`.woff2`/`.ttf`

### 7.5 Vue SPA base path配置

当Vue SPA通过画廊代理访问时，其构建产物的资源路径必须包含前缀，否则浏览器会从画廊根目录请求资源导致404。

**配置要求**:

| 项目 | vite.config.ts | router.ts |
|------|---------------|-----------|
| admin-profile-ui | `base: '/admin-profile-ui/'` | `createWebHistory(import.meta.env.BASE_URL)` |
| spectrum-ui | `base: '/spectrum-ui/'` | 无路由（单页） |

> **⚠️ 关键约束**: 修改`base`后必须重新构建(`npx vite build`)，否则dist中的资源路径仍为旧值。`createWebHistory()`必须传入`import.meta.env.BASE_URL`，否则SPA内部路由无法在画廊代理中正确匹配。

### 7.6 Nginx配置

```nginx
server {
    listen 80;
    server_name gallery.genesis.local;

    # 画廊本体
    location / {
        root /path/to/ui-gallery/dist;
        try_files $uri $uri/ /index.html;
    }

    # 静态页面代理
    location /static/ {
        alias /path/to/genesis/static/;
    }

    # 频谱仪UI代理
    location /spectrum-ui/ {
        alias /path/to/spectrum-ui/dist/;
    }

    # 管理员画像UI代理
    location /admin-profile-ui/ {
        alias /path/to/admin-profile-ui/dist/;
    }

    # TR测试工程师UI代理
    location /tr-test-ui/ {
        alias /path/to/ui-lab/public/tr-test-ui/;
    }

    # 基因瀑布流代理
    location /mall-daily-waterfall/ {
        alias /path/to/ui-lab/public/mall-daily-waterfall/;
    }

    # API代理
    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 7.7 gallery.json 配置格式

```json
{
  "version": "1.0.0",
  "updated_at": "2026-05-14",
  "categories": [
    {
      "id": "cat-static",
      "name": "Static Pages",
      "name_zh": "静态页面",
      "icon": "FileText",
      "description": "System base pages, pure HTML",
      "sort_order": 1
    },
    {
      "id": "cat-embedded",
      "name": "Embedded UI",
      "name_zh": "嵌入式UI",
      "icon": "Monitor",
      "description": "Vue 3 SPA with SHM vector integration",
      "sort_order": 2
    },
    {
      "id": "cat-instrument",
      "name": "Professional Instruments",
      "name_zh": "专业仪器",
      "icon": "Cpu",
      "description": "Professional test and monitoring instruments",
      "sort_order": 3
    },
    {
      "id": "cat-visualization",
      "name": "Visualization",
      "name_zh": "可视化",
      "icon": "Waves",
      "description": "Data visualization and narrative displays",
      "sort_order": 4
    }
  ],
  "exhibits": [
    {
      "id": "exhibit-index",
      "title": "Consumer Login",
      "title_zh": "消费者登录页",
      "description": "System entry point with dark theme login form",
      "description_zh": "系统入口，深色主题登录表单，背景网格+光晕效果",
      "category_id": "cat-static",
      "blueprint_id": null,
      "tech_stack": "html",
      "status": "active",
      "path": "/static/index.html",
      "preview_path": "/static/index.html",
      "thumbnail": "/thumbnails/index.png",
      "tags": ["login", "entry", "static"],
      "sort_order": 1,
      "meta": {}
    },
    {
      "id": "exhibit-dashboard",
      "title": "Consumer Dashboard",
      "title_zh": "消费者操作台",
      "description": "Main operation interface with system status and consumer control",
      "description_zh": "主操作界面，顶部导航栏，展示系统状态与消费者控制",
      "category_id": "cat-static",
      "blueprint_id": null,
      "tech_stack": "html",
      "status": "active",
      "path": "/static/dashboard.html",
      "preview_path": "/static/dashboard.html",
      "thumbnail": "/thumbnails/dashboard.png",
      "tags": ["dashboard", "operation", "static"],
      "sort_order": 2,
      "meta": {}
    },
    {
      "id": "exhibit-tasks",
      "title": "User Tasks",
      "title_zh": "用户任务台",
      "description": "Task management with status tracking and console output",
      "description_zh": "任务管理界面，任务列表、状态追踪、控制台输出",
      "category_id": "cat-static",
      "blueprint_id": null,
      "tech_stack": "html",
      "status": "active",
      "path": "/static/user_tasks.html",
      "preview_path": "/static/user_tasks.html",
      "thumbnail": "/thumbnails/tasks.png",
      "tags": ["tasks", "management", "static"],
      "sort_order": 3,
      "meta": {}
    },
    {
      "id": "exhibit-debug",
      "title": "Debug Console",
      "title_zh": "调试控制台",
      "description": "Development debugging with test execution and health checks",
      "description_zh": "开发调试界面，测试执行、覆盖率查看、健康检查、性能指标",
      "category_id": "cat-static",
      "blueprint_id": null,
      "tech_stack": "html",
      "status": "active",
      "path": "/static/debug.html",
      "preview_path": "/static/debug.html",
      "thumbnail": "/thumbnails/debug.png",
      "tags": ["debug", "development", "static"],
      "sort_order": 4,
      "meta": {}
    },
    {
      "id": "exhibit-spectrum",
      "title": "Spectrum Analyzer UI",
      "title_zh": "嵌入式频谱仪UI",
      "description": "SHM-driven FFT spectrum visualization with touch/key/encoder input",
      "description_zh": "商场内核的相呈现载体，将SHM推送的FFT数据渲染为频谱可视化界面",
      "category_id": "cat-embedded",
      "blueprint_id": "BP-0034",
      "tech_stack": "vue3",
      "status": "active",
      "path": "/spectrum-ui/",
      "preview_path": "/spectrum-ui/",
      "thumbnail": "/thumbnails/spectrum-ui.png",
      "tags": ["spectrum", "fft", "shm", "instrument"],
      "sort_order": 1,
      "meta": {
        "routes": 1,
        "components": 1,
        "tests": 24,
        "build_size_kb": 45
      }
    },
    {
      "id": "exhibit-admin",
      "title": "Admin Profile UI",
      "title_zh": "管理员画像系统UI",
      "description": "Admin profile visualization with instrument-style panels and SHM data",
      "description_zh": "管理员画像可视化界面，嵌入式仪器风格，展示管理员角色信息与平台模块关联",
      "category_id": "cat-embedded",
      "blueprint_id": "BP-0055",
      "tech_stack": "vue3",
      "status": "active",
      "path": "/admin-profile-ui/",
      "preview_path": "/admin-profile-ui/",
      "thumbnail": "/thumbnails/admin-profile-ui.png",
      "tags": ["admin", "profile", "shm", "instrument"],
      "sort_order": 2,
      "meta": {
        "routes": 5,
        "components": 8,
        "tests": 0,
        "build_size_kb": 120
      }
    },
    {
      "id": "exhibit-tr-test",
      "title": "TR Test Engineer UI",
      "title_zh": "TR测试工程师UI",
      "description": "Professional multi-instrument TR test interface with 6 instruments and chat",
      "description_zh": "专业TR测试工程师多仪器集成界面，6仪器同屏+Chat指令+数据表格",
      "category_id": "cat-instrument",
      "blueprint_id": "BP-0054",
      "tech_stack": "html",
      "status": "active",
      "path": "/tr-test-ui/",
      "preview_path": "/tr-test-ui/",
      "thumbnail": "/thumbnails/tr-test-ui.png",
      "tags": ["tr-test", "instrument", "multi-screen", "chat"],
      "sort_order": 1,
      "meta": {
        "components": 6,
        "build_size_kb": 35
      }
    },
    {
      "id": "exhibit-waterfall",
      "title": "Gene Waterfall",
      "title_zh": "基因瀑布流",
      "description": "Six-phase gene waterfall cards with narrative scrolling and audio",
      "description_zh": "六境卡片瀑布流展示，滚动叙事引擎+Web Audio音效+视差系统",
      "category_id": "cat-visualization",
      "blueprint_id": null,
      "tech_stack": "html",
      "status": "active",
      "path": "/mall-daily-waterfall/",
      "preview_path": "/mall-daily-waterfall/",
      "thumbnail": "/thumbnails/waterfall.png",
      "tags": ["waterfall", "narrative", "visualization", "genesis"],
      "sort_order": 1,
      "meta": {
        "components": 6,
        "build_size_kb": 28
      }
    }
  ]
}
```

---

## 八、组件规范

### 8.1 ExhibitCard

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| exhibit | GalleryExhibit | 展品数据 |

**事件**:

| 事件 | 参数 | 说明 |
|------|------|------|
| preview | exhibitId: string | 点击卡片进入预览 |

**内部计算属性**:

| 属性 | 说明 |
|------|------|
| accentColor | 按category_id映射分类强调色 |
| thumbnailStyle | 缩略图样式：`height: 140px` + `linear-gradient(135deg, {accentColor}15, #0F1623)` |
| techLabel | tech_stack映射显示文本：html→HTML, vue3→Vue 3, react→React |
| statusLed | status映射LED类：active→led green, building→led yellow, offline→led red, draft→led yellow |
| statusColor | status映射文本色：active→text-emerald, building→text-royal-gold, offline→text-flame-red, draft→text-silver-gray |

**分类强调色映射**:

| category_id | 色值 | 说明 |
|-------------|------|------|
| cat-static | `#8892A4` | 银灰色 |
| cat-embedded | `#00C9A7` | 翡翠绿 |
| cat-instrument | `#D4A843` | 皇室金 |
| cat-visualization | `#3B82F6` | 蓝色 |

**缩略图占位符**: 无实际截图时，渐变背景 + 展品中文名首字母（`title_zh.charAt(0)`，20%透明度，悬停30%）

**ACCESS按钮**: `@click.stop` 阻止冒泡，直接调用 `window.open(exhibit.path, '_blank')` 新窗口打开

### 8.2 HallSection

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| category | GalleryCategory | 展厅分类 |
| exhibits | GalleryExhibit[] | 该分类下的展品列表 |
| viewMode | ViewMode | 视图模式（grid/list） |

**事件**:

| 事件 | 参数 | 说明 |
|------|------|------|
| preview | exhibitId: string | 透传ExhibitCard的preview事件 |

**分类图标字符**:

| category_id | 字符 | 说明 |
|-------------|------|------|
| cat-static | ◈ | 菱形 |
| cat-embedded | ◉ | 实心圆 |
| cat-instrument | ⬡ | 六边形 |
| cat-visualization | ◎ | 靶心 |

**网格响应式断点**:

| 视图模式 | CSS类 | 说明 |
|----------|-------|------|
| grid | `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4` | 响应式1-4列 |
| list | `grid-cols-1` | 单列列表 |

**空展厅**: 展品为空时显示 "NO EXHIBITS IN THIS HALL"

### 8.3 PreviewPanel

**Props**:

| 属性 | 类型 | 说明 |
|------|------|------|
| exhibit | GalleryExhibit \| null | 要预览的展品，null时组件不渲染（v-if控制） |

**事件**:

| 事件 | 参数 | 说明 |
|------|------|------|
| close | - | 关闭预览面板 |

**布局结构**:
- **header** (h-14): BACK按钮 + 展品信息（中文名/英文名/蓝图ID/技术栈/LED状态） + FULLSCREEN按钮
- **main** (flex-1): iframe容器（max-w-1920px + border-charcoal），iframe使用 `sandbox="allow-scripts allow-same-origin"`
- **footer** (h-8): "PREVIEW MODE" + 展品路径 + 路由数/组件数/构建大小

**FULLSCREEN按钮**: `window.open(exhibit.path, '_blank')` 新窗口打开

### 8.4 FilterBar

**功能**: 搜索框 + 类别下拉 + 技术栈下拉 + 状态下拉 + 视图切换（网格/列表）

**筛选控件**:

| 控件 | 绑定 | 选项 |
|------|------|------|
| 搜索输入框 | store.searchQuery | placeholder: "Search UI name, tag, blueprint..." |
| 类别下拉 | store.filterCategory | ALL CATEGORIES + 动态加载categories |
| 技术栈下拉 | store.filterTechStack | ALL TECH / HTML / Vue 3 / React |
| 状态下拉 | store.filterStatus | ALL STATUS / Active / Building / Offline / Draft |
| 视图切换 | store.viewMode | 网格图标(grid) / 列表图标(list)，选中态为翡翠绿高亮 |

**样式**: 搜索框使用instrument-panel风格（bg-space-black + border-charcoal + focus:border-royal-gold），下拉使用instrument-btn风格

### 8.5 GalleryTopBar

**布局**: 固定顶部，h-14，bg-night-gray + border-b-charcoal，z-50

**左侧**: "Genesis UI Gallery" (text-royal-gold, tracking-widest, uppercase) + "v1.0.0" (text-silver-gray, 9px)

**右侧**: LED green + "CONNECTED" (text-emerald) + 实时时钟 (digital-readout, text-royal-gold, HH:MM:SS格式)

**底部装饰**: 1px金色渐变线（from-transparent via-royal-gold/40 to-transparent）

**时钟**: `setInterval` 每秒更新，`onUnmounted` 清理定时器

### 8.6 GalleryStatusBar

**布局**: 固定底部，h-8，bg-night-gray + border-t-charcoal，z-50

**左侧**: LED green + "{active} Active" (text-emerald) + "{total} UI Assets" (text-silver-gray) + "{done} Code Done" (text-royal-gold)

**右侧**: "BP-0056" (text-mist-gray)

**顶部装饰**: 1px金色渐变线（from-transparent via-royal-gold/30 to-transparent）

**数据来源**: `store.stats` computed（total/active/done）

### 8.7 GalleryView

**布局**: pt-14 pb-8 min-h-screen grid-bg（为TopBar/StatusBar留空间）

**结构**:
1. **标题区**: "▎UI Gallery" (text-2xl text-moon-white) + "Genesis Project · All UI Assets Exhibition" (text-silver-gray)
2. **FilterBar**: 筛选控件
3. **加载态**: 旋转边框圆环 + "LOADING GALLERY..."
4. **错误态**: LED red + "ERROR: {message}"
5. **展厅列表**: 遍历categories，每个category渲染HallSection
6. **空结果**: "NO MATCHING EXHIBITS" + RESET FILTERS按钮
7. **PreviewPanel**: v-if="store.previewExhibit" 弹窗预览

**生命周期**: `onMounted` 时若config未加载则调用 `store.loadConfig()`

### 8.8 PreviewView

**布局**: 全屏覆盖，fixed inset-0 z-[100] bg-space-black

**结构**:
- **header** (h-14): GALLERY按钮（router.push('/')）+ 展品信息 + OPEN IN NEW TAB按钮
- **main** (flex-1): iframe（v-if="exhibit"，sandbox="allow-scripts allow-same-origin"）
- **footer** (h-8): "PREVIEW MODE" + 展品路径 + 路由数/组件数

**展品查找**: `route.params.exhibitId` → `store.exhibits.find()`

**生命周期**: `onMounted` 时若config未加载则调用 `store.loadConfig()`

### 8.9 galleryStore (Pinia)

**状态**:

| 属性 | 类型 | 初始值 | 说明 |
|------|------|--------|------|
| config | GalleryConfig \| null | null | 画廊配置数据 |
| loading | boolean | true | 加载状态 |
| error | string \| null | null | 错误信息 |
| searchQuery | string | '' | 搜索关键词 |
| filterCategory | string | 'all' | 类别筛选 |
| filterTechStack | string | 'all' | 技术栈筛选 |
| filterStatus | string | 'all' | 状态筛选 |
| viewMode | ViewMode | 'grid' | 视图模式 |
| previewExhibit | GalleryExhibit \| null | null | 当前预览展品 |

**计算属性**:

| 属性 | 说明 |
|------|------|
| categories | config.categories按sort_order排序 |
| exhibits | config.exhibits按sort_order排序 |
| filteredExhibits | 搜索+类别+技术栈+状态四维筛选 |
| exhibitsByCategory | Map<categoryId, Exhibit[]>，按分类分组 |
| stats | { total, active, done } 统计 |

**方法**:

| 方法 | 说明 |
|------|------|
| loadConfig() | fetch('/gallery.json')加载配置，设置loading/error状态 |
| setPreview(exhibit) | 设置/清除预览展品 |
| getCategoryById(id) | 按ID查找分类 |

**搜索逻辑**: 同时匹配 title/title_zh/description/description_zh/tags/blueprint_id（不区分大小写）

### 8.10 类型定义 (types/index.ts)

| 类型 | 定义 | 说明 |
|------|------|------|
| GalleryCategory | { id, name, name_zh, icon, description, sort_order } | 展厅分类 |
| GalleryExhibit | { id, title, title_zh, description, description_zh, category_id, blueprint_id, tech_stack, status, path, preview_path, thumbnail, tags, sort_order, meta } | 展品数据 |
| GalleryConfig | { version, updated_at, categories[], exhibits[] } | 画廊配置 |
| ViewMode | 'grid' \| 'list' | 视图模式 |
| FilterCategory | string | 类别筛选值 |
| FilterTechStack | string \| 'all' | 技术栈筛选值 |
| FilterStatus | string \| 'all' | 状态筛选值 |

### 8.11 全局样式体系 (style.css)

**仪器核心类 (10类)**:

| CSS类 | 用途 | 特征 |
|-------|------|------|
| `.instrument-panel` | 仪器面板容器 | 渐变背景(#141B2D→#0F1623) + 2px金色装饰线(::before) + 1px border-charcoal + 2px圆角 |
| `.instrument-header` | 仪器头部 | 渐变底色(#1E2A3A→#141B2D) + 大写等宽11px + 1px字间距 + 金色#D4A843 |
| `.digital-readout` | 数字读数 | JetBrains Mono + tabular-nums + 金色光晕(0 0 12px rgba(212,168,67,0.4)) |
| `.instrument-btn` | 仪器按钮 | 渐变背景 + 1px border-#3A4556 + 大写等宽11px + hover金色 + active下沉1px |
| `.instrument-btn.active` | 激活态按钮 | 翡翠绿15%背景 + 翡翠绿边框 + 翡翠绿8px光晕 |
| `.led` | LED基类 | 8px圆形 + inset阴影 |
| `.led.green` | 绿色LED | radial-gradient(#33FFD4→#00C9A7) + 8px外发光 |
| `.led.yellow` | 黄色LED | radial-gradient(#F0D78C→#D4A843) + 8px外发光 |
| `.led.red` | 红色LED | radial-gradient(#FF6B6B→#E53E3E) + 8px外发光 + pulse动画1.5s |
| `.instrument-divider` | 分割线 | 1px + 渐变(transparent→#3A4556→transparent) |
| `.instrument-scale` | 刻度尺 | flex space-between + 9px + #3A4556 |
| `.instrument-grid-bg` | 仪器网格背景 | 20px网格 + rgba(30,42,58,0.3) |

**辅助类 (2类)**:

| CSS类 | 用途 | 特征 |
|-------|------|------|
| `.scanline` | 扫描线动画 | ::after 2px线 + 翡翠绿0.4透明度 + 3s线性无限 |
| `.grid-bg` | 页面网格背景 | 20px网格 + rgba(30,42,58,0.5) |

**关键帧动画**: `pulse` — 0%/100% opacity:1, 50% opacity:0.4

**CSS加载顺序**: `@import url(fonts)` → `@tailwind base` → `@tailwind components` → `@tailwind utilities` → 自定义样式

---

## 九、安全约束

### 9.1 iframe安全

| 约束 | 实现 |
|------|------|
| 同源策略 | 所有UI通过Nginx同域代理，避免跨域 |
| sandbox属性 | `sandbox="allow-scripts allow-same-origin"` |
| 点击劫持防护 | 目标UI设置`X-Frame-Options`白名单 |

### 9.2 资源限制

| 资源 | 限制 | 超限处理 |
|------|------|----------|
| 同时预览数 | 1个 | 关闭前一个再打开 |
| 缩略图大小 | <= 200KB | 自动压缩 |
| gallery.json | <= 50KB | 分页加载 |
| iframe内存 | <= 150MB | 自动卸载不可见iframe |

---

## 十、实现状态追踪

### 10.1 模块实现状态

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 画廊主页 | GalleryView.vue | ✅ 完成 | 4展厅+筛选+弹窗预览 |
| 预览页 | PreviewView.vue | ✅ 完成 | 全屏iframe+展品信息栏 |
| 展品卡片 | ExhibitCard.vue | ✅ 完成 | instrument-panel+渐变占位符+LED+ACCESS按钮 |
| 展厅分组 | HallSection.vue | ✅ 完成 | instrument-header+CSS Grid |
| 筛选栏 | FilterBar.vue | ✅ 完成 | 搜索+3维筛选+网格/列表切换 |
| 预览面板 | PreviewPanel.vue | ✅ 完成 | iframe弹窗+BACK/FULLSCREEN按钮 |
| 顶部栏 | GalleryTopBar.vue | ✅ 完成 | GENESIS UI GALLERY+时钟+LED |
| 状态栏 | GalleryStatusBar.vue | ✅ 完成 | 展品统计+BP-0056标识 |
| 状态管理 | galleryStore.ts | ✅ 完成 | 配置加载+搜索筛选+预览 |
| 配置数据 | gallery.json | ✅ 完成 | 8展品+4分类 |
| 路由 | router.ts | ✅ 完成 | 首页+预览页+beforeEach标题守卫 |
| 类型定义 | types/index.ts | ✅ 完成 | GalleryConfig/Exhibit/Category/ViewMode/FilterCategory/FilterTechStack/FilterStatus |
| Vite代理 | vite.config.ts | ✅ 完成 | serveStaticPlugin中间件+SPA fallback |
| 全局样式 | style.css | ✅ 完成 | instrument体系10类+辅助2类+pulse动画 |
| 缩略图 | thumbnails/ | ❌ 未实现 | 当前使用渐变占位符+首字母 |
| Nginx配置 | nginx-gallery.conf | ❌ 未实现 | 生产部署时需要 |

---

## 十一、已知问题与修复记录

### 11.1 已修复问题

| 问题 | 根因 | 修复 | 影响 |
|------|------|------|------|
| 点击展品链接看不到网页 | Vite开发服务器只服务ui-gallery自身文件，无法访问其他UI项目目录 | 添加serveStaticPlugin自定义中间件，将5个路径前缀映射到实际目录 | 所有展品无法预览 |
| Vue SPA资源路径404 | admin-profile-ui/spectrum-ui构建时base为`/`，资源路径为`/assets/xxx`，但画廊代理中应为`/admin-profile-ui/assets/xxx` | 两个项目vite.config.ts添加`base`配置并重新构建 | SPA类展品白屏 |
| Vue Router路由不匹配 | `createWebHistory()`未传入base path，SPA内部路由从`/`解析而非`/admin-profile-ui/` | `createWebHistory(import.meta.env.BASE_URL)` | SPA子路由404 |
| SPA子路由fallback缺失 | 中间件未处理SPA路由fallback，如`/admin-profile-ui/profile`返回404 | 中间件检测spaPrefixes做fallback返回index.html | SPA内部导航失败 |
| CSS @import顺序错误 | `@import url(...)`放在`@tailwind`之后，构建报错 | 将@import移到@tailwind之前 | 构建失败 |
| TypeScript对象键名含连字符 | `cat-static`等键名未加引号，TS编译报错 | 添加引号`'cat-static'` | 构建失败 |

### 11.2 已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 缩略图缺失 | 所有展品使用渐变占位符+首字母，无实际截图 | 后续添加截图功能 |
| Nginx配置未实现 | 生产部署配置尚未创建 | 部署时创建 |
| 看板视图未实现 | v1.0仅网格/列表两种视图 | v1.2添加 |
| 动态状态未接入 | 展品运行状态为静态配置，未接入SHM矢量 | SHM矢量层就绪后接入 |

---

## 十二、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-14 | 初始版本，8展品4展厅，Vue 3 + Tailwind + iframe预览 |
| v1.1.0 | 2026-05-14 | 实现完成+调试修复：serveStaticPlugin代理中间件、Vue SPA base path配置、SPA路由fallback、Router BASE_URL、6个构建修复；更新目录结构和实现状态 |
| v1.2.0 | 2026-05-14 | 蓝图精化：修复章节编号重复（7.3→7.4~7.7）、组件规范精确匹配代码（ExhibitCard/HallSection/PreviewPanel/FilterBar修正Props/Events）、新增8.5~8.11组件规范（TopBar/StatusBar/GalleryView/PreviewView/Store/类型定义/样式体系）、路由守卫beforeEach、style.css类数修正（6+6→10+2）、类型定义补充FilterCategory/FilterTechStack/FilterStatus |

---

## 附录

### A. 参考文档

- [ui-blueprint-overview.md](../../specs/ui-blueprint-overview.md) - UI蓝图总览
- [BP-0034-spectrum-ui.md](./BP-0034-spectrum-ui.md) - 频谱仪UI蓝图
- [BP-0054-tr-test-ui.md](./BP-0054-tr-test-ui.md) - TR测试工程师UI蓝图
- [BP-0055-admin-profile-ui.md](./BP-0055-admin-profile-ui.md) - 管理员画像UI蓝图
- [PROJECT_CONTEXT.md](../../PROJECT_CONTEXT.md) - 项目上下文

### B. 术语表

| 术语 | 说明 |
|------|------|
| Gallery | 画廊，UI资产展厅 |
| Exhibit | 展品，单个UI界面 |
| Hall | 展厅，按功能域分类的展品组 |
| Preview | 预览，iframe嵌入的UI运行实例 |
| Thumbnail | 缩略图，展品卡片的预览图 |
