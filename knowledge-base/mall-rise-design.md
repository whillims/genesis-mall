# 商场的崛起 — 知识库

> 来源：`Kimi_Agent_商场的崛起.zip`
> 技术栈：React 18 + TypeScript + Vite + Tailwind CSS + Framer Motion

---

## 一、项目概述

### 1.1 核心叙事

这是一个以"科幻史诗"风格呈现的 UI 设计蓝图 PPT，讲述"商场"（Mall）从操作系统帝国中崛起的故事。共 10 个章节，从创世大爆炸到基因蓝图共和国的建立。

### 1.2 章节结构

| 章节 | 英文标题 | 内容 |
|------|---------|------|
| 0 | Genesis Protocol | 创世大爆炸，粒子爆发动画 |
| 1 | The Empire's Code | 操作系统帝国的统治代码（暗色调） |
| 2 | Paradigm Rupture | 范式断裂点，帝国 vs 共和国关键词碰撞 |
| 3 | DNA Core | 商场 DNA 双螺旋可视化 |
| 4 | Gene Blueprint | 基因蓝图规格书（技术文档风格） |
| 5 | The Five Pillars | 五大支柱（函数范式/三界隔离/蓝图流程/审计链/共享内存） |
| 6 | AICoder Evolution | AICoder 进化路径（从脚本小子到架构师） |
| 7 | SHM Republic | 共享内存共和国（去中心化网络可视化） |
| 8 | The Mall Rises | 商场崛起（最终高潮） |
| 9 | Future Vision | 未来愿景 |

---

## 二、设计系统

### 2.1 色彩体系（Sci-Fi Palette）

```css
:root {
  --void-black: #050508;
  --deep-space: #0a0e1a;
  --nebula-core: #12182b;
  --gene-cyan: #00f0ff;
  --gene-blue: #0080ff;
  --gene-purple: #b829dd;
  --gene-gold: #ffd700;
  --empire-red: #ff2a2a;
  --empire-ash: #4a4a4a;
  --data-stream: rgba(0, 240, 255, 0.3);
  --pulse-glow: rgba(0, 128, 255, 0.6);
}
```

**配色逻辑**：
- 青色系（cyan/blue）= 商场/基因/正面力量
- 红色系（red/ash）= 帝国/垄断/反面力量
- 金色 = 王者/终极
- 紫色 = 变异/进化

### 2.2 字体系统

```css
--font-display: 'Orbitron', 'MiSans', monospace;
--font-sans: 'Rajdhani', 'MiSans', system-ui, sans-serif;
--font-mono: 'Fira Code', monospace;
```

| 用途 | 字体 | 特征 |
|------|------|------|
| 大标题 | Orbitron | 几何科幻，宽字距 |
| 正文 | Rajdhani | 现代无衬线，轻量 |
| 代码/标签 | Fira Code | 等宽，编程连字 |

### 2.3 圆角策略

```css
--radius: 0rem;
```

全局零圆角 = 硬朗科幻风。

---

## 三、CSS 动效库

### 3.1 Glitch 故障文字

```css
@keyframes glitch {
  0%, 90%, 100% { transform: translate(0); clip-path: none; }
  91% { transform: translate(-3px, 2px); clip-path: inset(20% 0 60% 0); }
  92% { transform: translate(3px, -2px); clip-path: inset(50% 0 30% 0); }
  93% { transform: translate(0); clip-path: none; }
  94% { transform: translate(-2px, 1px); clip-path: inset(10% 0 70% 0); }
  95% { transform: translate(0); clip-path: none; }
}

.glitch-text { animation: glitch 5s infinite; position: relative; }
.glitch-text::before { color: var(--empire-red); clip-path: inset(0 0 50% 0); }
.glitch-text::after  { color: var(--gene-cyan);  clip-path: inset(50% 0 0 0); }
```

**关键技巧**：90% 时间静止，仅在 91%-95% 帧产生故障，形成"偶尔闪烁"效果。

### 3.2 Pulse Glow 脉冲发光

```css
@keyframes pulse-glow {
  0%, 100% { opacity: 1; box-shadow: 0 0 20px rgba(0, 240, 255, 0.3); }
  50% { opacity: 0.85; box-shadow: 0 0 40px rgba(0, 240, 255, 0.5); }
}

@keyframes crackGlow {
  0%, 100% { opacity: 0.6; box-shadow: 0 0 20px var(--gene-cyan); }
  50% { opacity: 1; box-shadow: 0 0 40px var(--gene-cyan), 0 0 60px var(--gene-gold); }
}
```

### 3.3 扫描线叠加

```css
body::after {
  content: '';
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 9999;
  background: repeating-linear-gradient(
    0deg,
    transparent, transparent 2px,
    rgba(0, 240, 255, 0.015) 2px,
    rgba(0, 240, 255, 0.015) 4px
  );
}
```

### 3.4 全息卡片

```css
.hologram-card {
  background: linear-gradient(135deg, rgba(18, 24, 43, 0.9), rgba(10, 14, 26, 0.95));
  border: 1px solid rgba(0, 240, 255, 0.2);
  position: relative;
  overflow: hidden;
}
.hologram-card::before {
  content: '';
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, transparent 50%, rgba(0, 240, 255, 0.03) 50%);
  background-size: 100% 4px;
  pointer-events: none;
}
```

### 3.5 六边形裁剪

```css
.hexagon {
  clip-path: polygon(50% 0%, 100% 25%, 100% 75%, 50% 100%, 0% 75%, 0% 25%);
}
```

---

## 四、Canvas 粒子特效

### 4.1 大爆炸粒子系统（Genesis）

```typescript
interface Particle {
  x: number; y: number;
  vx: number; vy: number;
  life: number; color: string;
  size: number; decay: number;
}

const colors = ['#ffffff', '#ffd700', '#00f0ff', '#0080ff'];

for (let i = 0; i < 300; i++) {
  const angle = Math.random() * Math.PI * 2;
  const speed = 2 + Math.random() * 6;
  particles.push({
    x: centerX, y: centerY,
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    life: 1,
    color: colors[Math.floor(Math.random() * colors.length)],
    size: Math.random() * 3 + 1,
    decay: 0.002 + Math.random() * 0.003,
  });
}

// 渲染：粒子本体 + 径向渐变发光
particles.forEach((p) => {
  p.x += p.vx; p.y += p.vy;
  p.vx *= 0.98; p.vy *= 0.98;
  p.life -= p.decay;

  ctx.globalAlpha = p.life * 0.8;
  ctx.beginPath();
  ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2);
  ctx.fillStyle = p.color;
  ctx.fill();

  // 发光层
  const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.size * p.life * 3);
  grad.addColorStop(0, p.color + '40');
  grad.addColorStop(1, 'transparent');
  ctx.fillStyle = grad;
  ctx.fill();
});
```

### 4.2 DNA 双螺旋（DNACore）

```typescript
const numPoints = 200;
const helixRadius = 80;

for (let i = 0; i < numPoints; i++) {
  const t = i / numPoints;
  const angle = t * Math.PI * 8;  // 4 圈螺旋
  const y = t * canvasHeight;
  const xA = centerX + Math.cos(angle) * helixRadius;
  const xB = centerX + Math.cos(angle + Math.PI) * helixRadius;
  const zA = Math.sin(angle);

  // 连接横杆（每隔 10 个点）
  if (i % 10 === 0) {
    ctx.beginPath();
    ctx.moveTo(xA, y);
    ctx.lineTo(xB, y);
    ctx.strokeStyle = `rgba(0, 240, 255, ${0.1 + Math.abs(zA) * 0.2})`;
    ctx.stroke();
  }
}
```

### 4.3 浮动粒子网络（ParticleBackground）

```typescript
// 50 个慢速漂浮点 + 距离连线
const particles = Array.from({ length: 50 }, () => ({
  x: Math.random() * w, y: Math.random() * h,
  vx: (Math.random() - 0.5) * 0.3,
  vy: (Math.random() - 0.5) * 0.3,
  size: Math.random() * 2 + 0.5,
  opacity: Math.random() * 0.5 + 0.1,
}));

// 粒子间连线（距离 < 150px 时绘制）
particles.forEach((a, i) => {
  particles.slice(i + 1).forEach((b) => {
    const dist = Math.hypot(a.x - b.x, a.y - b.y);
    if (dist < 150) {
      ctx.strokeStyle = `rgba(0, 240, 255, ${0.1 * (1 - dist / 150)})`;
      ctx.stroke();
    }
  });
});
```

---

## 五、Framer Motion 动画模式

### 5.1 滚动触发入场

```typescript
const [visible, setVisible] = useState(false);
const sectionRef = useRef<HTMLElement>(null);

useEffect(() => {
  const observer = new IntersectionObserver(
    ([entry]) => { if (entry.isIntersecting) setVisible(true); },
    { threshold: 0.3 }
  );
  if (sectionRef.current) observer.observe(sectionRef.current);
  return () => observer.disconnect();
}, []);

<motion.div
  initial={{ opacity: 0, scale: 0.8 }}
  animate={visible ? { opacity: 1, scale: 1 } : {}}
  transition={{ duration: 1.5, ease: 'easeOut' }}
/>
```

### 5.2 级联延迟入场

```typescript
<motion.div transition={{ delay: 0.3 }} />   // 标签
<motion.h2 transition={{ delay: 0.8 }} />   // 标题
<motion.div transition={{ delay: 1.2 }} />  // 分割线
<motion.p   transition={{ delay: 1.5 }} />  // 描述
```

### 5.3 打字机逐行展示

```typescript
const [typewriterIndex, setTypewriterIndex] = useState(0);

useEffect(() => {
  if (!triggered) return;
  const timer = setInterval(() => {
    setTypewriterIndex((prev) => {
      if (prev >= lines.length) { clearInterval(timer); return prev; }
      return prev + 1;
    });
  }, 1000);
  return () => clearInterval(timer);
}, [triggered]);
```

---

## 六、视觉设计模式

### 6.1 章节标签格式

```html
<div class="font-mono text-xs tracking-[0.4em]" style="color: var(--gene-cyan)">
  SECTION 0 // GENESIS PROTOCOL
</div>
```

### 6.2 渐变分割线

```html
<div class="w-48 h-px mx-auto"
  style="background: linear-gradient(90deg, transparent, var(--gene-cyan), transparent)">
</div>
```

### 6.3 径向渐变光晕

```html
<div class="w-[500px] h-[500px] rounded-full"
  style="background: radial-gradient(circle,
    rgba(0,240,255,0.08) 0%, rgba(0,128,255,0.03) 40%, transparent 70%)">
</div>
```

### 6.4 自定义滚动条

```css
::-webkit-scrollbar { width: 4px; }
::-webkit-scrollbar-track { background: var(--void-black); }
::-webkit-scrollbar-thumb { background: var(--gene-cyan); border-radius: 2px; }
```

---

## 七、叙事性 UI 设计理念

### 7.1 对立色彩叙事

| 阵营 | 色彩 | 关键词 |
|------|------|--------|
| 帝国（旧） | 红/灰 | vendor_lock, monopoly, spyware, bloatware, update_or_die |
| 共和国（新） | 青/蓝/金 | blueprint_migration, gene_pool, aicoder_evolution, human_audit, shm_republic |

### 7.2 视觉高潮节奏

```
Genesis(爆发) → Empire(压抑) → Rupture(断裂/高潮) → DNA(生命)
→ Blueprint(技术) → Pillars(稳定) → Evolution(成长)
→ Republic(繁荣) → Mall(巅峰) → Future(展望)
```

### 7.3 情绪色彩映射

- **爆发/创世**：白 + 金 + 青（大爆炸）
- **压抑/统治**：红 + 灰 + 暗黑（帝国）
- **断裂/冲突**：红 vs 青碰撞（裂纹效果）
- **生命/基因**：青 + 蓝 + 紫（DNA 螺旋）
- **繁荣/巅峰**：金 + 青（全息卡片）

---

## 八、可复用技术清单

| 技术 | 适用场景 | 复杂度 |
|------|---------|--------|
| Glitch 文字效果 | 标题/品牌展示 | 低（纯 CSS） |
| 扫描线叠加 | 全局科幻氛围 | 低（纯 CSS） |
| 全息卡片 | 信息展示卡片 | 低（纯 CSS） |
| 脉冲发光 | 状态指示/按钮 | 低（纯 CSS） |
| 六边形裁剪 | 头像/图标容器 | 低（纯 CSS） |
| 大爆炸粒子 | 首屏/过渡动画 | 中（Canvas） |
| DNA 双螺旋 | 数据可视化 | 中（Canvas） |
| 浮动粒子网络 | 背景/氛围 | 中（Canvas） |
| 滚动触发动画 | 章节入场 | 低（Framer Motion） |
| 级联延迟入场 | 内容逐步展示 | 低（Framer Motion） |
| 打字机效果 | 文字/宣言展示 | 低（React State） |
| 对立色彩叙事 | 品牌/故事讲述 | 设计层面 |
