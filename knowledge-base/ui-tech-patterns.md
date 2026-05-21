# UI 技术模式知识库

> 来源：Kimi_Agent_生成ui.zip 项目分析
> 技术栈：React + TypeScript + Tailwind CSS + Three.js

---

## 一、Canvas 波形渲染技术

### 1.1 双层 Canvas 架构
```typescript
// 静态层（网格）+ 动态层（波形）分离
const gridCanvasRef = useRef<HTMLCanvasElement | null>(null);  // 离屏缓存
const canvasRef = useRef<HTMLCanvasElement>(null);             // 主画布

// 网格只绘制一次，缓存到离屏 canvas
if (config.enableOffscreenCache) {
  if (!gridCanvasRef.current) {
    gridCanvasRef.current = document.createElement('canvas');
  }
  // ... 绘制网格到 gridCanvasRef
}

// 每帧渲染时直接 drawImage 缓存的网格
if (config.enableOffscreenCache && gridCanvasRef.current) {
  ctx.drawImage(gridCanvasRef.current, 0, 0, width, height);
}
```

### 1.2 DPR 适配
```typescript
const dpr = Math.min(window.devicePixelRatio, 2);  // 限制最大 DPR 避免性能问题
canvas.width = width * dpr;
canvas.height = height * dpr;
ctx.scale(dpr, dpr);
```

### 1.3 波形发光效果
```typescript
ctx.strokeStyle = '#38bdf8';
ctx.lineWidth = 1.5;
ctx.shadowColor = 'rgba(56, 189, 248, 0.5)';  // 发光颜色
ctx.shadowBlur = 8;                            // 发光半径
ctx.beginPath();
// ... 绘制路径
ctx.stroke();
ctx.shadowBlur = 0;  // 重置避免影响其他绘制
```

### 1.4 视口变换系统
```typescript
interface ViewportState {
  offsetX: number;  // 水平偏移（数据点索引）
  scaleX: number;   // 水平缩放倍数
  offsetY: number;  // 垂直偏移（像素）
  scaleY: number;   // 垂直缩放倍数
}

// 数据到屏幕坐标转换
const visibleStart = Math.max(0, Math.floor(vp.offsetX));
const visibleEnd = Math.min(data.length, Math.ceil(data.length / vp.scaleX + visibleStart));
const x = ((i - visibleStart) / (visibleEnd - visibleStart - 1)) * width;
const y = centerY - data[i] * scaleY;
```

---

## 二、LTTB 降采样算法

### 2.1 算法实现
```typescript
function lttbDownsample(data: Float32Array, threshold: number): Float32Array {
  const n = data.length;
  if (threshold >= n || threshold <= 0) return data;

  const sampled = new Float32Array(threshold);
  sampled[0] = data[0];

  for (let i = 1; i < threshold - 1; i++) {
    const avgRangeStart = Math.floor((i - 1) * (n - 1) / (threshold - 1)) + 1;
    const avgRangeEnd = Math.floor(i * (n - 1) / (threshold - 1)) + 1;
    
    // 计算桶内平均值
    let avgY = 0;
    const count = avgRangeEnd - avgRangeStart;
    for (let j = avgRangeStart; j < avgRangeEnd; j++) {
      avgY += data[j];
    }
    avgY /= count;

    // 寻找最大三角形面积的点
    const pointAX = (i - 1) * (n - 1) / (threshold - 1);
    const pointAY = sampled[i - 1];
    let maxArea = -1;
    let maxIdx = avgRangeStart;

    for (let j = avgRangeStart; j < avgRangeEnd; j++) {
      const area = Math.abs(
        (pointAX - j) * (avgY - pointAY) - 
        (pointAX - (i * (n - 1) / (threshold - 1))) * (data[j] - pointAY)
      );
      if (area > maxArea) {
        maxArea = area;
        maxIdx = j;
      }
    }
    sampled[i] = data[maxIdx];
  }

  sampled[threshold - 1] = data[n - 1];
  return sampled;
}
```

### 2.2 自适应阈值计算
```typescript
const getDownsampledData = useCallback((
  data: Float32Array, 
  canvasWidth: number, 
  scaleX: number, 
  enableL3: boolean
): Float32Array => {
  if (!enableL3) return data;

  const visiblePoints = Math.floor(data.length / scaleX);
  // 目标：每个像素 2 个数据点
  const threshold = Math.min(Math.max(Math.floor(canvasWidth * 2), 200), visiblePoints);

  if (threshold >= visiblePoints || threshold >= data.length) return data;

  return lttbDownsample(data, threshold);
}, []);
```

---

## 三、性能监控 Hook 模式

### 3.1 usePerformanceMonitor
```typescript
const MAX_HISTORY = 200;

export function usePerformanceMonitor(enabled: boolean) {
  const [metrics, setMetrics] = useState<PerformanceMetrics>({
    fps: 60,
    frameTime: 16,
    memory: 2.1,
    compressionRatio: 1,
    droppedFrameRate: 0,
  });

  const [history, setHistory] = useState<PerformanceHistory>({
    fps: [], frameTime: [], memory: [], compressionRatio: [], timestamps: [],
  });

  // 使用 ref 避免重渲染
  const frameCountRef = useRef(0);
  const lastTimeRef = useRef(performance.now());
  const droppedFramesRef = useRef(0);
  const totalFramesRef = useRef(0);

  const recordFrame = useCallback(() => {
    const now = performance.now();
    const delta = now - lastTimeRef.current;
    frameCountRef.current++;
    totalFramesRef.current++;

    // 掉帧检测：帧耗时 > 33ms (低于 30fps)
    if (delta > 33) {
      droppedFramesRef.current += Math.floor(delta / 33);
    }

    // 每 500ms 更新一次指标
    if (now - startTimeRef.current >= 500) {
      const elapsed = (now - startTimeRef.current) / 1000;
      const fps = Math.min(Math.round(frameCountRef.current / elapsed), 60);
      const frameTime = Math.round((elapsed * 1000) / frameCountRef.current * 10) / 10;
      const dropRate = totalFramesRef.current > 0 
        ? Math.round((droppedFramesRef.current / totalFramesRef.current) * 1000) / 10 
        : 0;

      setMetrics({ fps, frameTime, memory, compressionRatio, droppedFrameRate: dropRate });
      
      // 滚动历史数组
      setHistory(prev => ({
        fps: [...prev.fps, fps].slice(-MAX_HISTORY),
        // ... 其他指标
      }));

      frameCountRef.current = 0;
      startTimeRef.current = now;
    }

    lastTimeRef.current = now;
  }, []);

  useEffect(() => {
    if (!enabled) return;
    const loop = () => {
      recordFrame();
      rafRef.current = requestAnimationFrame(loop);
    };
    rafRef.current = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(rafRef.current);
  }, [enabled, recordFrame]);

  return { metrics, history, recordFrame };
}
```

---

## 四、SVG 实时图表

### 4.1 渐变面积图
```typescript
// SVG 渐变定义
<defs>
  <linearGradient id="grad-fps" x1="0%" y1="0%" x2="0%" y2="100%">
    <stop offset="0%" style="stop-color:#38bdf8;stop-opacity:0.3" />
    <stop offset="100%" style="stop-color:#38bdf8;stop-opacity:0.02" />
  </linearGradient>
</defs>

// 面积路径
const areaPath = `
  M ${points[0]}
  L ${points.join(' L ')}
  L ${padding.left + chartWidth},${padding.top + chartHeight}
  L ${padding.left},${padding.top + chartHeight}
  Z
`;

<path d={areaPath} fill="url(#grad-fps)" />
<path d={linePath} fill="none" stroke={color} strokeWidth="1.5" />
```

### 4.2 打字机故障效果
```typescript
// 鼠标悬停时的文字打乱效果
const [displayLabel, setDisplayLabel] = useState(label);
const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';

useEffect(() => {
  if (!hoveredLabel) {
    setDisplayLabel(label);
    return;
  }

  let iteration = 0;
  const maxIterations = label.length * 2;

  const interval = setInterval(() => {
    setDisplayLabel(
      label
        .split('')
        .map((_char, idx) => {
          if (idx < Math.floor(iteration / 2)) return label[idx];
          return chars[Math.floor(Math.random() * chars.length)];
        })
        .join('')
    );
    iteration++;
    if (iteration > maxIterations) {
      setDisplayLabel(label);
      clearInterval(interval);
    }
  }, 50);

  return () => clearInterval(interval);
}, [hoveredLabel, label]);
```

---

## 五、Three.js WebGL 背景

### 5.1 ShaderMaterial 网格背景
```typescript
const backgroundVertexShader = `
  uniform float uTime;
  uniform vec2 uMouse;
  
  void main() {
    vec3 pos = position;
    // 波浪动画
    float zOffset = sin(pos.x * 0.1 + uTime) * cos(pos.y * 0.1 + uTime) * 2.0;
    pos.z += zOffset;
    
    // 鼠标交互
    float dist = distance(uMouse, pos.xy);
    float circle = smoothstep(5.0, 0.0, dist);
    pos.z += circle * 5.0;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
  }
`;

const backgroundFragmentShader = `
  uniform float uTime;
  uniform vec2 uMouse;
  uniform vec2 uResolution;
  
  void main() {
    vec2 uv = gl_FragCoord.xy / uResolution.xy;
    
    // 扫描线效果
    float rayX = sin(uTime * 0.5) * 0.5 + 0.5;
    float rayWidth = 0.05;
    float rayIntensity = smoothstep(rayWidth, 0.0, abs(uv.x - rayX)) * 0.15;
    
    vec3 color = vec3(10.0, 14.0, 26.0) / 255.0;
    color += rayIntensity;
    
    gl_FragColor = vec4(color, 0.4);
  }
`;
```

### 5.2 鼠标交互网格
```typescript
// 创建网格线
const gridSize = 400;
const numVertical = 100;
const numHorizontal = 80;

const verticalPositions = new Float32Array(numVertical * 2 * 3);
for (let i = 0; i < numVertical; i++) {
  const x = (i - numVertical / 2) * (gridSize / numVertical);
  verticalPositions[i * 6] = x;
  verticalPositions[i * 6 + 1] = -gridSize / 2;
  verticalPositions[i * 6 + 2] = 0;
  verticalPositions[i * 6 + 3] = x;
  verticalPositions[i * 6 + 4] = gridSize / 2;
  verticalPositions[i * 6 + 5] = 0;
}

const vGeo = new THREE.BufferGeometry();
vGeo.setAttribute('position', new THREE.BufferAttribute(verticalPositions, 3));
const vMat = new THREE.LineBasicMaterial({ 
  color: 0x1f2937, 
  transparent: true, 
  opacity: 0.5 
});
const vLines = new THREE.LineSegments(vGeo, vMat);
scene.add(vLines);
```

---

## 六、Tailwind 暗色主题系统

### 6.1 CSS 变量定义
```css
:root {
  --background: 222 47% 5%;      /* #0a0e1a */
  --foreground: 210 40% 98%;     /* #f9fafb */
  --card: 217 33% 8%;            /* #111827 */
  --card-foreground: 210 40% 98%;
  --primary: 199 89% 48%;        /* #38bdf8 */
  --primary-foreground: 222 47% 5%;
  --muted: 217 33% 12%;          /* #1f2937 */
  --muted-foreground: 215 16% 65%; /* #9ca3af */
  --border: 217 33% 15%;         /* #374151 */
  --success: 142 69% 58%;        /* #4ade80 */
  --warning: 38 92% 50%;         /* #fbbf24 */
  --destructive: 0 84% 60%;      /* #f87171 */
}
```

### 6.2 组件样式类
```css
@layer components {
  .card-surface {
    @apply bg-[#111827] border border-[#1f2937] rounded;
  }
  .card-surface:hover {
    border-color: #38bdf8;
    box-shadow: 0 0 15px rgba(56, 189, 248, 0.15);
  }
  
  .btn-primary {
    @apply px-4 py-2 bg-[#1f2937] text-[#f9fafb] text-sm font-medium rounded transition-all duration-200;
  }
  .btn-primary:hover {
    @apply bg-[#38bdf8] text-[#0a0e1a];
    box-shadow: 0 0 15px rgba(56, 189, 248, 0.3);
  }
  
  .label-muted {
    @apply text-[#9ca3af] text-xs font-medium uppercase tracking-wider;
  }
  
  /* 扫描线效果 */
  .scanline-overlay {
    background: repeating-linear-gradient(
      0deg,
      transparent,
      transparent 2px,
      rgba(56, 189, 248, 0.03) 2px,
      rgba(56, 189, 248, 0.03) 4px
    );
    pointer-events: none;
  }
}
```

---

## 七、数据缓存模式

### 7.1 波形数据缓存
```typescript
const dataCache = useRef<Map<string, Float32Array>>(new Map());
const dualChannelCache = useRef<Map<string, { i: Float32Array; q: Float32Array }>>(new Map());

const generateWaveform = useCallback((config: WaveformConfig) => {
  const cacheKey = `${config.dataLevel}-${config.waveformType}`;
  
  // 检查缓存
  if (config.channelMode === 'dual') {
    if (dualChannelCache.current.has(cacheKey)) {
      const cached = dualChannelCache.current.get(cacheKey)!;
      return { data: cached.i, dualData: cached };
    }
  } else {
    if (dataCache.current.has(cacheKey)) {
      return { data: dataCache.current.get(cacheKey)! };
    }
  }

  // 生成新数据
  const data = new Float32Array(config.dataLevel);
  // ... 填充数据

  // 存入缓存
  if (config.channelMode === 'dual') {
    dualChannelCache.current.set(cacheKey, { i: dualI, q: dualQ });
  } else {
    dataCache.current.set(cacheKey, data);
  }
  
  return { data };
}, []);
```

---

## 八、交互设计模式

### 8.1 开关按钮组件
```typescript
function ToggleButton({ label, enabled, onClick }: { 
  label: string; 
  enabled: boolean; 
  onClick: () => void;
}) {
  return (
    <button
      onClick={onClick}
      className="flex items-center gap-1.5 text-xs transition-colors"
      style={{ color: enabled ? '#38bdf8' : '#6b7280' }}
    >
      {enabled ? (
        <ToggleRight className="w-4 h-4" style={{ color: '#38bdf8' }} />
      ) : (
        <ToggleLeft className="w-4 h-4" style={{ color: '#6b7280' }} />
      )}
      <span>{label}</span>
    </button>
  );
}
```

### 8.2 状态指示器
```typescript
<div className="flex items-center gap-2">
  <CircleDot 
    className="w-3.5 h-3.5" 
    style={{ color: isRunning ? '#4ade80' : '#6b7280' }} 
  />
  <span className="font-mono text-xs font-semibold" style={{ color: '#f9fafb' }}>
    {isRunning ? '波形渲染中' : '引擎待机'}
  </span>
</div>
```

### 8.3 阈值颜色编码
```typescript
// FPS 颜色
<span style={{ 
  color: fps >= 45 ? '#4ade80' : fps >= 30 ? '#fbbf24' : '#f87171' 
}}>
  {fps}
</span>

// 内存颜色
<span style={{ 
  color: memory < 8 ? '#4ade80' : '#fbbf24' 
}}>
  {memory.toFixed(1)}MB
</span>

// 帧耗时颜色
<span style={{ 
  color: frameTime < 22 ? '#4ade80' : '#fbbf24' 
}}>
  {frameTime.toFixed(1)}ms
</span>
```

---

## 九、字体与排版

### 9.1 字体栈
```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=JetBrains+Mono:wght@400;500;600;700&display=swap');

html {
  font-family: 'Inter', 'PingFang SC', 'Microsoft YaHei', ui-sans-serif, system-ui, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

code, .font-mono {
  font-family: 'JetBrains Mono', ui-monospace, monospace;
}
```

### 9.2 等宽字体用于数据
```typescript
<span className="font-mono text-xs font-bold" style={{ color }}>
  {lastValue.toFixed(1)}{unit}
</span>
```

---

## 十、项目文件结构

```
app/
├── src/
│   ├── sections/           # 页面区块组件
│   │   ├── HeroSection.tsx
│   │   ├── Navigation.tsx
│   │   ├── WaveformCanvas.tsx    # 波形渲染主组件
│   │   ├── PerformanceCharts.tsx # 性能图表
│   │   ├── TestScenarioMatrix.tsx
│   │   └── WebGLGrid.tsx         # Three.js 背景
│   ├── hooks/              # 自定义 Hooks
│   │   ├── usePerformanceMonitor.ts
│   │   ├── useWaveformData.ts
│   │   └── use-mobile.ts
│   ├── components/ui/      # shadcn/ui 组件库
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── chart.tsx
│   │   └── ... (50+ components)
│   ├── lib/
│   │   └── utils.ts        # 工具函数
│   ├── types/
│   │   └── index.ts        # TypeScript 类型定义
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css           # 全局样式 + Tailwind
├── public/
│   └── *.jpg               # 静态资源
├── tailwind.config.js
├── vite.config.ts
└── package.json
```

---

## 参考项目

- **技术栈**: React 18 + TypeScript + Vite + Tailwind CSS + Three.js
- **UI 库**: shadcn/ui (基于 Radix UI)
- **图标**: Lucide React
- **构建**: Vite
