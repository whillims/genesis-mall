# 创世纪动画UI重构 - 知识库文档

> **来源**: Kimi_Agent_创世纪动画UI重构.zip
> **创建日期**: 2026-05-14
> **项目路径**: `knowledge-base/genesis-anim-ui/`

---

## 1. 项目概述

创世纪动画UI是创世纪商场系统的**沉浸式启动动画**，以"宇宙大爆炸"到"商场觉醒"的叙事方式，展示商场的基因编码、蓝图拓扑和宪法精神。这是进入商场前的仪式感体验，也是创世纪世界观的核心视觉表达。

### 1.1 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 18.x | UI框架 |
| TypeScript | 5.x | 类型系统 |
| Vite | 7.x | 构建工具 |
| Tailwind CSS | 3.4.x | 样式框架 |
| shadcn/ui | - | 基础组件库 (40+) |
| Framer Motion | - | 动画引擎 |
| lucide-react | - | 图标库 |

### 1.2 项目特点

- **五阶段创世叙事**：虚空→编码→蓝图→觉醒→完成
- **Canvas DNA双螺旋动画**：基因序列可视化
- **SVG蓝图拓扑图**：商场架构组件连接关系
- **Framer Motion编排**：流畅的页面过渡与元素动画
- **OKLCH色彩系统**：统一的颜色空间
- **终端日志面板**：实时显示创世过程
- **可跳过**：点击任意处或按ESC跳过动画

---

## 2. 项目结构

```
genesis-anim-ui/
├── public/
│   ├── hero-bg.jpg           # 英雄区背景图
│   └── data-texture.jpg      # 数据纹理背景
├── src/
│   ├── components/
│   │   ├── ui/               # shadcn/ui组件 (40+)
│   │   └── genesis/          # 创世动画组件
│   │       ├── GenesisCanvas.tsx      # 主Canvas画布
│   │       ├── GenesisController.tsx  # 阶段控制器
│   │       ├── GenesisTerminal.tsx    # 终端日志面板
│   │       ├── PhaseVoid.tsx          # 虚空阶段
│   │       ├── PhaseEncode.tsx        # 编码阶段
│   │       ├── GeneHelix.tsx          # DNA双螺旋
│   │       ├── BlueprintTopology.tsx  # 蓝图拓扑图
│   │       └── PhaseAwake.tsx         # 觉醒阶段
│   ├── hooks/
│   │   └── useGenesisPhase.ts         # 创世阶段Hook
│   ├── types/
│   │   └── genesis.ts                 # 类型定义
│   ├── lib/
│   │   ├── genesis-utils.ts           # 创世工具函数
│   │   └── utils.ts                   # 通用工具
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

## 3. 核心叙事：五阶段创世

### 3.1 阶段流程图

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  PHASE 1 │───▶│  PHASE 2 │───▶│  PHASE 3 │───▶│  PHASE 4 │───▶│  PHASE 5 │
│   VOID   │ 2s │  ENCODE  │ 4s │ BLUEPRINT│ 4s │  AWAKE   │ 2s │   DONE   │
│   虚空   │    │   编码   │    │   蓝图   │    │   觉醒   │    │   完成   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
     ▼               ▼               ▼               ▼               ▼
  呼吸光点        DNA双螺旋      组件拓扑图       商场主界面      路由跳转
  "在0和1之前     "基因编码      "基因已解码       "微型商场       进入商场
   先有光"        旋转展开"       为可执行的相"     ·创世版"       或查看宪法
```

### 3.2 各阶段详解

#### Phase 1: Void（虚空）- 2秒

**文件**: `src/components/genesis/PhaseVoid.tsx`

- 屏幕中央一个呼吸光点（2x2像素）
- 外发光圈（8px，青绿色）
- 呼吸动画：scale [1, 1.8, 1]，opacity [0.3, 0.1, 0.3]
- 底部文字："在 0 和 1 之前，先有光"
- 状态标签：STATE_VOID

#### Phase 2: Encode（编码）- 4秒

**文件**: `src/components/genesis/PhaseEncode.tsx` + `GeneHelix.tsx`

- Canvas绘制的DNA双螺旋动画
- 双链结构：青绿色链 + 紫色链
- 40个点/链，量子相位每100ms跳跃0.15
- 基因字符流：16个字符沿螺旋表面流动
- 底部展示基因序列：`0x3F_A1_2B_8C` 等
- 状态标签：STATE_ENCODE

**基因序列**（8个宪法哈希片段）：
```typescript
const GENESIS_GENE_SEQUENCE = [
  "0x3F_A1_2B_8C",  // 三权分立标识
  "0x7D_E4_91_F2",  // 安全宪法 v1.0
  "0x1A_55_C3_9E",  // SHM 矢量协议
  "0xB8_22_7F_4D",  // 消费者主权
  "0xE6_3C_A0_1B",  // AICoder 职责边界
  "0x4F_88_D5_62",  // 生产者进化权
  "0x9A_11_3E_B7",  // 商场中立性
  "0xC5_77_2A_F9",  // 蓝图自由迁移
];
```

#### Phase 3: Blueprint（蓝图）- 4秒

**文件**: `src/components/genesis/BlueprintTopology.tsx`

- SVG绘制的商场架构拓扑图
- 6个组件节点：交互(button)、容器(card)、输入(input)、标识(badge)、MALL(logo)、节点(node)
- 5条连接虚线（动画流动效果）
- MALL Logo：SVG路径动画绘制
- 状态标签：STATE_BLUEPRINT

**拓扑连接关系**：
```
交互(btn) ──▶ 容器(card) ──▶ 输入(input)
                │
                ▼
标识(badge) ──▶ MALL(logo) ──▶ 节点(node)
```

#### Phase 4: Awake（觉醒）- 2秒

**文件**: `src/components/genesis/PhaseAwake.tsx`

- 商场主界面卡片浮现
- 背景二进制粒子流（20个0/1字符下落）
- 版本徽章：v1.0 GENESIS
- 标题：微型商场 · 创世版
- 副标题：三权分立 · 安全宪法 · 基因驱动
- 两个按钮：进入商场 / 查看宪法
- 宪法弹窗（Dialog组件，7条宪法条款）
- 状态标签：STATE_AWAKE

**宪法内容**：
```
第一条 · 三权分立
第二条 · 安全宪法
第三条 · 基因驱动
第四条 · 消费者主权
第五条 · 商场中立性
第六条 · 蓝图自由迁移
第七条 · AICoder 职责边界
signed: Genesis Block 0x3FA12B8C
```

#### Phase 5: Done（完成）

- 动画结束，可路由跳转至商场主界面
- 进度条满格

---

## 4. 核心组件详解

### 4.1 GenesisController - 阶段控制器

**文件**: `src/components/genesis/GenesisController.tsx`

**职责**：
- 管理五阶段切换逻辑
- 渲染当前阶段的视觉元素
- 处理跳过逻辑（点击/ESC）
- 显示进度条和状态标签

**子组件**：
- `PhaseLabel`：左上角状态标签（STATE_VOID等）
- `SkipHint`：右上角跳过提示（2秒后显示）
- `ProgressBar`：底部进度条

### 4.2 GenesisCanvas - 主Canvas画布

**文件**: `src/components/genesis/GenesisCanvas.tsx`

**职责**：
- 全屏Canvas渲染
- 背景网格线（40px间距，opacity 0.02）
- 背景纹理图（opacity 0.03）
- 扫描线动效（CSS动画）

### 4.3 GeneHelix - DNA双螺旋

**文件**: `src/components/genesis/GeneHelix.tsx`

**技术细节**：
- Canvas 2D渲染
- 双链结构，每链40个点
- 量子相位：每100ms跳跃0.15
- 半径动态扩展：40→140px（1.5秒内）
- 基因字符流：16个字符沿螺旋流动
- 连接线：每5个点绘制横桥

**颜色**：
- 链1：oklch(0.65 0.18 180) 青绿色
- 链2：oklch(0.60 0.15 280) 紫色
- 连接线：oklch(0.65 0.1 230)

### 4.4 BlueprintTopology - 蓝图拓扑图

**文件**: `src/components/genesis/BlueprintTopology.tsx`

**技术细节**：
- SVG动态绘制连接线
- 贝塞尔曲线路径
- 虚线流动动画（CSS dash-flow）
- 组件轮廓：Framer Motion缩放动画
- MALL Logo：SVG路径绘制动画（pathLength 0→1）

### 4.5 GenesisTerminal - 终端日志面板

**文件**: `src/components/genesis/GenesisTerminal.tsx`

**技术细节**：
- 固定左下角
- 可折叠/展开
- 实时日志追加（每100ms检查）
- 自动滚动到底部
- 闪烁光标
- REC录制指示灯

**日志内容**：
```
[00:00:00] [GENE] 启动序列已初始化
[00:00:05] [GENE] 写入宪法摘要到内存...
[00:00:12] [INFO] 基因螺旋旋转稳定
[00:00:20] [GENE] 三权分立协议已签署
[00:00:30] [GENE] 拓扑图节点展开中
[00:00:38] [INFO] UI 组件轮廓已生成
[00:00:45] [WARN] 商场觉醒 — 准备接管渲染
[00:00:50] [GENE] 创世完成
```

### 4.6 useGenesisPhase - 创世阶段Hook

**文件**: `src/hooks/useGenesisPhase.ts`

**阶段配置**：
```typescript
const PHASE_CONFIG: Record<GenesisPhase, PhaseConfig> = {
  void:     { duration: 2000,  next: 'encode',    label: '虚空' },
  encode:   { duration: 4000,  next: 'blueprint', label: '编码' },
  blueprint:{ duration: 4000,  next: 'awake',     label: '蓝图' },
  awake:    { duration: 2000,  next: 'done',      label: '觉醒' },
  done:     { duration: 0,     next: null,        label: '完成' },
};
```

**返回值**：
```typescript
{
  phase: GenesisPhase;      // 当前阶段
  progress: number;         // 总进度 (0-1)
  isSkipped: boolean;       // 是否已跳过
  skipAnimation: () => void; // 跳过函数
}
```

---

## 5. 样式系统

### 5.1 设计变量

```css
/* 核心背景 */
--bg-primary: #050508;      /* 深空黑 */

/* 基因链颜色 */
--gene-cyan: oklch(0.65 0.18 180);    /* 青绿色链 */
--gene-purple: oklch(0.60 0.15 280);  /* 紫色链 */

/* 状态颜色 */
--status-warn: oklch(0.70 0.20 50);   /* 警告/觉醒 */
--status-gold: oklch(0.75 0.12 100);  /* 金色/完成 */

/* 文字色 */
--text-primary: #f0f0f5;
--text-secondary: #9ca3af;
--text-muted: oklch(0.95 0.01 260 / 0.3);
```

### 5.2 动画缓动函数

```typescript
// 主缓动：ease-out-expo
[0.16, 1, 0.3, 1]

// 呼吸动画
ease: 'easeInOut'

// 线性流动
ease: 'linear'
```

---

## 6. 与创世纪项目集成

### 6.1 集成场景

该动画UI作为创世纪商场的**启动页/欢迎页**，在以下场景使用：

1. **首次访问**：展示完整五阶段创世动画
2. **日常访问**：可配置跳过动画直接进入商场
3. **演示模式**：完整展示商场世界观

### 6.2 集成方式

```
┌──────────────────────────────────────────────────────┐
│              Genesis Animation UI                     │
│                                                      │
│  Phase 1-4: 动画展示                                  │
│  Phase 5:  "进入商场" 按钮 ──▶ 路由跳转                │
│                                                      │
│  路由: /genesis → /dashboard (admin-office-ui)       │
│       /genesis → /bus-monitor (tr-test-ui)           │
└──────────────────────────────────────────────────────┘
```

### 6.3 可复用组件

| 组件 | 说明 | 复用价值 |
|------|------|----------|
| GenesisTerminal | 终端日志面板 | 高 - 可复用为系统日志查看器 |
| GeneHelix | DNA双螺旋动画 | 中 - 可作为加载动画 |
| BlueprintTopology | 拓扑图 | 中 - 可展示系统架构 |
| PhaseAwake | 觉醒卡片 | 高 - 可作为欢迎页模板 |
| useGenesisPhase | 阶段管理Hook | 高 - 通用多阶段动画管理 |

### 6.4 集成建议

1. **路由配置**：在App.tsx中添加 `/genesis` 路由
2. **跳过配置**：本地存储记录用户是否看过动画
3. **API对接**：将终端日志替换为真实的系统启动日志
4. **宪法内容**：从后端API动态加载宪法条款
5. **基因序列**：与商场授权文件的哈希值关联

---

## 7. 运行方式

```bash
# 进入项目目录
cd knowledge-base/genesis-anim-ui

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

## 8. 相关资源

- [shadcn/ui 官方文档](https://ui.shadcn.com/)
- [Framer Motion 文档](https://www.framer.com/motion/)
- [OKLCH 色彩空间](https://oklch.com/)
- [创世纪项目 Code Wiki](file:///d:\创世纪\CODE_WIKI.md)
- [创世纪项目上下文](file:///d:\创世纪\PROJECT_CONTEXT.md)

---

*本文档由SOLO-AICoder生成*
