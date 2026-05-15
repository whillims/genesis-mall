# 创世纪启动动画消费者蓝图

> **蓝图编号**: BP-0066-GENESIS-BOOT-ANIMATION
> **版本**: v2.0.0
> **日期**: 2026-05-16
> **功能域**: consumer
> **状态**: DRAFT

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0066-GENESIS-BOOT-ANIMATION |
| 函数名 | genesis_boot_init / genesis_boot_tick |
| 生产者名称 | GenesisBootAnimationConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | 无（纯展示，用户点击/按键触发登录） |
| 输出协议 | Canvas 2D渲染 + CSS动画 |
| 安全等级 | P0 (公开入口页面) |
| 版本 | v2.0.0 |
| 前置蓝图 | BP-0056-UI-GALLERY |

---

## 二、设计意图

### 2.1 核心目标

构建创世纪项目的启动动画入口页面，以"历史演进循环剧场"形式展示商场模式在人类文明史中的五次跃迁：原始集市→封建行会→工业流水线→平台垄断→创世纪商场。用户在观看完整历史演进后，通过点击/按键触发"觉醒"，进入登录界面。

### 2.2 哲学定位

v1.0的"一次性创世仪式"已重构为"历史演进循环剧场"。动画不再是单次基因宣读，而是商场模式在人类文明史中的五次跃迁循环展演。用户在观看完整历史演进后，通过主动点击完成从"观察者"到"参与者"的身份转换。

> "商场不是被发明的，而是从人类交换本能中演化出来的。观看历史，即是理解基因。"

### 2.3 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | 前端=消费者域，动画=产品（客观），样式=相（主观） |
| 函数范式 | Vue 3 Composition API纯函数设计 |
| 仪器风格 | 深空黑背景+金色/翡翠绿点缀，与画廊体系一致 |
| 画廊集成 | Vite构建+base路径+静态映射，iframe嵌入画廊 |

### 2.4 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vue 3 + Canvas 2D | 与画廊统一技术栈，零第三方依赖 | React + Framer Motion | 无Framer Motion的spring动画 |
| Canvas 2D粒子系统 | 纪元三/四/五需要粒子效果 | WebGL | 降级方案更简单 |
| CSS关键帧动画 | 纪元一/二简单动画，性能最优 | JS动画 | 复杂度低 |
| 五纪元循环 | 历史唯物主义叙事，25s总循环 | 单一场景 | 需要状态机管理 |
| 点击觉醒触发 | 用户主动参与，符合历史唯物主义 | 自动跳转 | 用户选择权 |
| 登录界面 | 创世纪项目统一登录入口 | 外部登录 | 需要独立实现 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 历史纪元 | 商场模式演进的五个阶段 | Era组件 |
| 循环叙事 | 五纪元按时间轴顺序播放 | EraLoopController |
| 演进熵增 | 视觉复杂度递增 | 粒子数量/动画复杂度 |
| 觉醒触发 | 用户点击/按键进入登录 | AwakeningTrigger |
| 纪元过渡 | 纪元间切换效果 | EraTransition |
| 观察者契约 | 循环期间纯展示，无交互 | 除觉醒触发外无控件 |

### 3.2 五纪元架构

```
┌─────────────────────────────────────────────────────────────┐
│  纪元一 ──→ 纪元二 ──→ 纪元三 ──→ 纪元四 ──→ 纪元五 ──→ [循环]  │
│  (原始)    (行会)    (工业)    (平台)    (创世纪)            │
│   5s        5s        5s        5s        5s        总循环 25s  │
│                                                             │
│  任意时刻 ──→ [用户点击/按键] ──→ 登录界面                      │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 状态机

```
[INIT] ──→ [ERA_1] ──→ [TRANS_1→2] ──→ [ERA_2] ──→ [TRANS_2→3] ──→ [ERA_3]
   ↑                                                              │
   └──────────────────────────────────────────────────────────────┘
   （循环：ERA_5 结束后回到 ERA_1）

中断事件:
  - USER_CLICK/ENTER/SPACE: 任意状态 ──→ [COLLAPSE] ──→ [LOGIN]
  - MOUSE_HOVER: 当前纪元暂停（可选）
  - MOUSE_LEAVE: 继续播放
```

---

## 四、行为契约

### 4.1 纪元配置

```typescript
interface EraConfig {
  id: string
  name: string
  name_en: string
  period: string
  duration: number
  colors: {
    background: string
    primary: string
    accent: string
  }
  topology: 'point-to-point' | 'star-blackhole' | 'linear-pipeline' | 'centralized-pyramid' | 'decentralized-triangle'
  symbols: string[]
  textContent: string | null
}

const ERAS_CONFIG: EraConfig[] = [
  {
    id: 'primitive',
    name: '原始集市',
    name_en: 'The Primitive Bazaar',
    period: '史前 → 公元前',
    duration: 5000,
    colors: { background: '#1a1510', primary: '#8B7355', accent: '#D4A574' },
    topology: 'point-to-point',
    symbols: ['shell', 'stone', 'bone'],
    textContent: null,
  },
  {
    id: 'guild',
    name: '封建行会',
    name_en: 'The Guild System',
    period: '中世纪 → 文艺复兴',
    duration: 5000,
    colors: { background: '#0f172a', primary: '#64748b', accent: '#94a3b8' },
    topology: 'star-blackhole',
    symbols: ['hexagon', 'lock', 'scroll'],
    textContent: 'GUILD',
  },
  {
    id: 'industrial',
    name: '工业流水线',
    name_en: 'The Industrial Pipeline',
    period: '18世纪 → 20世纪初',
    duration: 5000,
    colors: { background: '#18181b', primary: '#71717a', accent: '#a1a1aa' },
    topology: 'linear-pipeline',
    symbols: ['gear', 'box', 'steam'],
    textContent: '01001000 01101001',
  },
  {
    id: 'platform',
    name: '平台垄断',
    name_en: 'The Platform Hegemony',
    period: '20世纪末 → 21世纪初',
    duration: 5000,
    colors: { background: '#0a0f1c', primary: '#3b82f6', accent: '#60a5fa' },
    topology: 'centralized-pyramid',
    symbols: ['pyramid', 'data', 'algorithm'],
    textContent: '用户画像 · 算法推荐 · 流量变现',
  },
  {
    id: 'genesis',
    name: '创世纪商场',
    name_en: 'The Genesis Mall',
    period: '未来 · 现在',
    duration: 5000,
    colors: { background: '#020617', primary: '#00C9A7', accent: '#D4A843' },
    topology: 'decentralized-triangle',
    symbols: ['dna', 'triangle', 'shm'],
    textContent: '三权分立 · 安全宪法 · 基因驱动',
  },
]
```

### 4.2 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | ERA_1 | 初始化Canvas，开始第一纪元 |
| ERA_N | 纪元时长到达 | TRANS_N→N+1 | 触发过渡动画 |
| TRANS | 过渡完成 | ERA_N+1 | 渲染下一纪元 |
| ERA_5 | 纪元时长到达 | TRANS_5→1 | 触发回到第一纪元的过渡 |
| 任意 | 用户点击/Enter/Space | COLLAPSE | 执行坍缩动画 |
| COLLAPSE | 坍缩完成 | LOGIN | 显示登录界面 |
| 任意 | 鼠标悬停 | PAUSED | 暂停当前纪元动画 |
| PAUSED | 鼠标移出 | 原状态 | 继续播放 |

### 4.3 觉醒触发

```typescript
function useAwakeningTrigger(onTrigger: () => void) {
  const handleTrigger = (e: MouseEvent | KeyboardEvent) => {
    const target = e.target as HTMLElement
    if (target.closest('[data-login-interface]')) return
    if (e instanceof KeyboardEvent && e.key !== 'Enter' && e.key !== ' ') return
    onTrigger()
  }

  window.addEventListener('click', handleTrigger)
  window.addEventListener('keydown', handleTrigger)
  window.addEventListener('touchstart', handleTrigger, { passive: true })

  return () => {
    window.removeEventListener('click', handleTrigger)
    window.removeEventListener('keydown', handleTrigger)
    window.removeEventListener('touchstart', handleTrigger)
  }
}
```

---

## 五、UI设计规范

### 5.1 纪元色彩映射

| 纪元 | 色温基调 | 背景 | 主色 | 辅色 | 象征 |
|------|----------|------|------|------|------|
| 一·原始 | 暖土 | #1a1510 | #8B7355 | #D4A574 | 大地、篝火 |
| 二·行会 | 冷石 | #0f172a | #64748b | #94a3b8 | 石墙、羊皮纸 |
| 三·工业 | 冷钢 | #18181b | #71717a | #a1a1aa | 钢铁、蒸汽 |
| 四·平台 | 冷蓝 | #0a0f1c | #3b82f6 | #60a5fa | 数据、算法 |
| 五·创世纪 | 深空 | #020617 | #00C9A7 | #D4A843 | 基因、宇宙 |

### 5.2 纪元一：原始集市

| 属性 | 值 |
|------|-----|
| 背景 | CSS渐变，土黄+深褐，轻微噪点纹理 |
| 核心元素 | 两个圆点(16px, stone-500)以正弦波轨迹靠近 |
| 交换时刻 | 中央垂直光柱(2px, amber-400) |
| 符号 | 贝壳/石块/兽骨抽象SVG轮廓(24px) |
| 信息载体 | 无文字，纯符号闪烁 |
| 拓扑 | 点对点直连 |
| 动画 | ease-in-out缓动，极慢节奏 |

### 5.3 纪元二：封建行会

| 属性 | 值 |
|------|-----|
| 背景 | 深石板色(#0f172a)，模拟羊皮纸与石墙 |
| 核心元素 | 中央六边形(80px, slate-400描边)代表行会总部 |
| 工匠节点 | 6个矩形(40×24px)环绕公转 |
| 连接 | 节点间虚线连接，不与中心直连 |
| 信息载体 | 哥特式大写字母"G"，stone-300，opacity 0.3 |
| 拓扑 | 星型拓扑，中心黑洞 |
| 动画 | 公转+光点间歇传输 |

### 5.4 纪元三：工业流水线

| 属性 | 值 |
|------|-----|
| 背景 | 冷钢灰色(#18181b)，模拟工厂车间 |
| 核心元素 | 水平流水线贯穿屏幕中央(2px实线, zinc-400) |
| 节点 | 5个方形(32×32px, zinc-500)从左向右匀速移动 |
| 产品 | 小方块从上方落入节点 |
| 蒸汽 | 垂直半透明矩形，opacity 0.1，高度随机变化 |
| 信息载体 | 等宽字体二进制流(01001000...)，zinc-300，14px滚动 |
| 拓扑 | 线性拓扑，单向流动 |
| 动画 | 机械匀速，无缓动 |

### 5.5 纪元四：平台垄断

| 属性 | 值 |
|------|-----|
| 背景 | 深蓝色(#0a0f1c)，模拟数据中心 |
| 核心元素 | 中央金字塔(4层矩形堆叠) |
| 平台巨头 | 顶端菱形(blue-400，强烈glow) |
| 数据汇聚 | 底层两侧小点(4px)向中心汇聚 |
| 数据流 | 粒子向上涌动(blue-300) |
| 返利 | 光点从顶端滴落到底层 |
| 信息载体 | 平台术语漂浮("用户画像""算法推荐"等)，slate-400，12px |
| 拓扑 | 中心化星型拓扑 |
| 动画 | 数据洪流高速涌动 |

### 5.6 纪元五：创世纪商场

| 属性 | 值 |
|------|-----|
| 背景 | 深空黑(#020617)，回归虚空感 |
| 核心元素 | 三权分立拓扑图：三个等距节点(三角形排列) |
| 节点颜色 | 生产者(emerald)、商场(royal-gold)、消费者(moon-white) |
| 连接 | 双向实线，等边三角形 |
| DNA双螺旋 | 三角形中心旋转，accent色点缀 |
| Worker节点 | 6个小节点环绕公转 |
| SHM数据流 | 背景缓慢流动的细线(primary/20) |
| 信息载体 | 中央宣言"三权分立 · 安全宪法 · 基因驱动"，Inter，20px |
| 拓扑 | 去中心化三角拓扑，节点对等互锁 |
| 动画 | 优雅稳定，象征新秩序 |

### 5.7 纪元过渡：历史断层

| 步骤 | 效果 | 时长 |
|------|------|------|
| 画面撕裂 | CSS clip-path多边形动画，斜向裂缝 | 400ms |
| 数据冲刷 | Canvas粒子爆炸，前一纪元符号纹理 | 600ms |
| 拓扑重组 | 新结构从碎片中重组 | 500ms |
| 色温跃迁 | 背景色300ms内完成跳变 | 300ms |

### 5.8 觉醒过渡

| 步骤 | 效果 | 时长 |
|------|------|------|
| 坍缩 | 所有元素向中心汇聚，scale 1→0，opacity 1→0 | 400ms |
| 白光 | 中心光点扩展为白色全屏 | 300ms |
| 展开 | 登录界面从中心缩放进入 | 500ms |

### 5.9 登录界面

| 区域 | 内容 |
|------|------|
| Card | max-w-md，glass变体(bg-background/80 backdrop-blur-xl) |
| Header | 商场徽标(48px) + "微型商场 · 创世版" + "基因已验证，请输入身份密钥" |
| Content | 用户名Input + 密码Input + 记住我Checkbox |
| Footer | "进入商场"主按钮(fullWidth, emerald背景) + "以访客浏览"辅助按钮(ghost) |
| 下方 | 版本信息"v1.0 Genesis · 三权分立宪法已生效" |

---

## 六、技术架构

### 6.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | Vue 3 + Composition API | SPA组件化 |
| 动画渲染 | Canvas 2D + CSS关键帧 | 纪元三/四/五用Canvas，一/二用CSS |
| 样式系统 | Tailwind CSS v3 | 创世纪仪器色彩体系 |
| 构建工具 | Vite | base: '/genesis-boot-animation/' |

### 6.2 项目目录结构

```
genesis-boot-animation/
├── public/
│   └── favicon.svg
├── src/
│   ├── App.vue
│   ├── main.ts
│   ├── style.css
│   ├── components/
│   │   ├── GenesisController.vue
│   │   ├── EraPrimitive.vue
│   │   ├── EraGuild.vue
│   │   ├── EraIndustrial.vue
│   │   ├── EraPlatform.vue
│   │   ├── EraGenesis.vue
│   │   ├── EraTransition.vue
│   │   ├── LoginInterface.vue
│   │   └── CollapseAnimation.vue
│   ├── composables/
│   │   ├── useEraLoop.ts
│   │   ├── useAwakeningTrigger.ts
│   │   └── usePerformanceProfile.ts
│   ├── utils/
│   │   └── genesis-utils.ts
│   └── types/
│       └── index.ts
├── tailwind.config.js
├── vite.config.ts
└── package.json
```

### 6.3 性能策略

| 设备性能 | 策略 |
|----------|------|
| 高性能 | 全特效：粒子+辉光+模糊+多图层 |
| 中性能 | 关闭粒子系统，保留CSS动画+SVG |
| 低性能 | 纯CSS关键帧动画，无Canvas，纪元简化为单色背景+文字 |
| 无障碍(prefers-reduced-motion) | 仅显示静态纪元五画面+登录界面，无动画循环 |

### 6.4 纪元预加载

双缓冲预加载：当前纪元播放时，后台静默预渲染下一纪元初始帧。预渲染使用离屏Canvas或visibility:hidden的DOM节点。过渡触发时，预渲染帧立即替换当前帧。

---

## 七、组件规范

### 7.1 GenesisController

**职责**: 纪元循环控制器+状态机

**状态**: currentEraIndex / eraProgress / isPlaying / isPaused / isLoginVisible

**逻辑**: 管理五纪元循环，处理过渡动画，响应觉醒触发

### 7.2 EraPrimitive

**职责**: 纪元一渲染

**技术**: CSS动画，两个圆点正弦波靠近，中央光柱，符号闪烁

### 7.3 EraGuild

**职责**: 纪元二渲染

**技术**: CSS transform rotate/orbit，六边形+6矩形公转，虚线连接

### 7.4 EraIndustrial

**职责**: 纪元三渲染

**技术**: Canvas 2D，流水线+移动节点+掉落产品+蒸汽柱+二进制流

### 7.5 EraPlatform

**职责**: 纪元四渲染

**技术**: Canvas 2D粒子系统，金字塔+数据汇聚+术语漂浮

### 7.6 EraGenesis

**职责**: 纪元五渲染

**技术**: Canvas 2D，三角拓扑+DNA双螺旋+Worker公转+SHM数据流

### 7.7 EraTransition

**职责**: 纪元间过渡

**技术**: CSS clip-path + Canvas粒子爆炸

### 7.8 LoginInterface

**职责**: 登录界面

**布局**: 居中Card(glass变体)，用户名/密码/记住我/进入按钮/访客按钮

### 7.9 CollapseAnimation

**职责**: 觉醒触发后的坍缩动画

**技术**: CSS动画，scale→0 + opacity→0，然后白光展开

---

## 八、安全约束

| 约束 | 实现 |
|------|------|
| 无后端通信 | 纯前端展示，登录为静态UI |
| 无敏感数据 | 仅展示动画，不含个人隐私 |
| XSS防护 | Vue模板自动转义 |
| 性能安全 | 低性能自动降级，避免卡顿 |
| 无障碍 | 支持prefers-reduced-motion |

---

## 九、实现状态追踪

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 类型定义 | types/index.ts | ⬜ | EraConfig+EraState+TopologyType |
| 纪元配置 | utils/genesis-utils.ts | ⬜ | ERAS_CONFIG+色温映射+拓扑算法 |
| 循环控制器 | composables/useEraLoop.ts | ⬜ | 五纪元循环+状态机 |
| 觉醒触发 | composables/useAwakeningTrigger.ts | ⬜ | 点击/按键/触摸监听 |
| 性能检测 | composables/usePerformanceProfile.ts | ⬜ | 降级策略 |
| 纪元一 | EraPrimitive.vue | ⬜ | CSS动画，点对点拓扑 |
| 纪元二 | EraGuild.vue | ⬜ | CSS动画，星型黑洞拓扑 |
| 纪元三 | EraIndustrial.vue | ⬜ | Canvas 2D，线性流水线拓扑 |
| 纪元四 | EraPlatform.vue | ⬜ | Canvas 2D粒子，中心化金字塔拓扑 |
| 纪元五 | EraGenesis.vue | ⬜ | Canvas 2D，去中心化三角拓扑 |
| 过渡组件 | EraTransition.vue | ⬜ | clip-path+粒子爆炸 |
| 登录界面 | LoginInterface.vue | ⬜ | glass Card+表单 |
| 坍缩动画 | CollapseAnimation.vue | ⬜ | scale+opacity动画 |
| 主控制器 | GenesisController.vue | ⬜ | 状态机+纪元切换 |
| 样式系统 | style.css | ⬜ | Tailwind+纪元色彩+关键帧 |
| 构建配置 | vite.config.ts | ⬜ | base: '/genesis-boot-animation/' |
| 画廊集成 | gallery.json | ⬜ | exhibit-genesis-boot |

---

## 十、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| 无音效 | 未集成音频系统 | v1.1添加Web Audio API |
| 登录无后端 | 纯静态UI，无真实验证 | v2.0接入商场认证系统 |
| Canvas粒子性能 | 低性能设备可能卡顿 | 已设计三级降级 |
| 无触摸手势 | 仅支持点击，无滑动 | v1.1添加手势支持 |
| 固定25s循环 | 不可配置时长 | v1.1添加配置参数 |

---

## 十一、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v2.0.0 | 2026-05-16 | DRAFT：基于genesis-boot-animation-blueprint-v2改写，技术栈从React+Framer Motion+shadcn/ui迁移至Vue 3+Canvas 2D+CSS关键帧+Tailwind CSS，保留五纪元循环架构（原始集市→封建行会→工业流水线→平台垄断→创世纪商场），历史断层过渡，觉醒触发，登录界面，三级性能降级 |

---

## 附录

### A. 参考文档

- [genesis-boot-animation-blueprint-v2.md](../../genesis-boot-animation-blueprint-v2.md) - 原始设计文档（React+Framer Motion+shadcn/ui）
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图

### B. 术语表

| 术语 | 说明 |
|------|------|
| 历史纪元 | 商场模式演进的五个阶段 |
| 循环叙事 | 五纪元按时间轴顺序播放，第五纪元后回到第一纪元 |
| 演进熵增 | 每个纪元视觉复杂度递增 |
| 觉醒触发 | 用户点击/按键，打断循环进入登录 |
| 历史断层 | 纪元间切换效果：画面撕裂+数据冲刷+拓扑重组 |
| 观察者契约 | 循环期间纯展示，无交互控件 |
| 三权分立 | 生产者域、商场域、消费者域 |

### C. v2→BP-0066迁移映射

| v2特性 | BP-0066适配 |
|--------|-------------|
| React + TypeScript | Vue 3 + Composition API + TypeScript |
| Framer Motion | CSS关键帧动画 + Canvas 2D |
| shadcn/ui | Tailwind CSS + 自定义组件 |
| Geist Mono/Inter | JetBrains Mono + system fonts |
| OKLCH色彩 | Hex色彩（与创世纪体系一致） |
| WebGL DNA螺旋 | Canvas 2D简化版 |
| 完整登录表单 | 静态UI（无后端） |
| app/目录结构 | src/components/标准Vue结构 |
