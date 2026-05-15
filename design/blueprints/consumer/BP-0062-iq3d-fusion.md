# 时频IQ三维融合可视化系统消费者蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0062-IQ3D-FUSION
> **版本**: v1.1.0
> **日期**: 2026-05-15
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0062-IQ3D-FUSION |
| 函数名 | iq3d_fusion_init / iq3d_fusion_tick |
| 生产者名称 | IQ3DFusionConsumer |
| 语言 | TypeScript + Vue 3 |
| 输入协议 | SHM矢量订阅 (VEC-DISPLAY) + 静态配置 |
| 输出协议 | SHM矢量发布 (VEC-CONTROL) |
| 安全等级 | P1 (内部工具) |
| 版本 | v1.1.0 |

---

## 二、设计意图

### 2.1 核心目标

创建Web端高性能时频IQ三维融合可视化界面，在单窗口内融合展示：

1. **时域波形** — I路/Q路随时间变化的曲线，带标尺和缩放
2. **频域瀑布图** — 频率-时间-幅度三维强度图，纹理逐行追加
3. **IQ星座图** — I-Q平面散点图，EVM实时计算
4. **频谱分析** — FFT频谱/STFT时频热力图双模式切换
5. **参数面板** — 仪器参数+信号参数+状态监控三组控制

### 2.2 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | UI作为消费者域，只订阅SHM矢量，不直接操作商场 |
| 函数范式 | 组件纯函数设计，状态外置Pinia Store |
| SHM矢量空间 | 通过VEC-DISPLAY订阅I/Q/FFT数据，VIEW零拷贝模式 |
| 标志位机制 | 32位无锁互斥，Worker表达域与商场表达域数值隔离 |
| 授权文件体系 | 消费者需AUTH-BP-0062授权文件激活 |
| 仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| Nginx钢铁脊椎 | 静态资源由Nginx直接服务，WebSocket通过Nginx反向代理 |

### 2.3 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Canvas 2D渲染 | 零第三方依赖，符合函数范式，沙箱调试期全透明 | LightningChart JS(商业授权) / Three.js(重依赖) | 需自行实现瀑布图纹理追加和降采样 |
| Cooley-Tukey FFT | O(N log N)性能，1024点实时60fps | 原始DFT O(N²) | 需位反转+蝶形运算实现 |
| 合成数据Demo | 沙箱调试期无真实SHM数据源，用数学函数生成Demo数据 | 真实ADC数据 | Demo模式需标注`preprocessed: true` |
| 五面板布局 | 时域+瀑布图+星座图+频谱图+参数面板五区同屏 | 单面板切换 | 需CSS Grid响应式布局 |
| 降采样在消费者端 | 生产者传输原始产品，消费者根据"相"降采样 | 生产者端预降采样 | 符合商场基因原则 |
| 标尺+缩放共享模块 | zoomRuler.ts统一管理ZoomState和drawRulers | 每组件独立实现 | 减少重复代码，保证交互一致 |
| bufferCanvas替代OffscreenCanvas | 画廊iframe中OffscreenCanvas可能不兼容 | OffscreenCanvas | 需手动管理buffer尺寸 |
| plotW同步Store | 标尺占用36px后绘图区域变窄，waterfallAppend需用plotW | 全canvas宽度 | 需在drawRulers后同步宽度 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| 融合视窗 | 主界面窗口 | 五面板Grid布局 |
| 时域波形 | I/Q随时间变化 | 左上Canvas折线图+标尺+缩放 |
| 瀑布图 | 频谱-时间-幅度 | 左下Canvas热力图+标尺+缩放 |
| 星座图 | I-Q平面散点 | 右上Canvas散点图+标尺+缩放 |
| 频谱分析 | FFT/STFT双模式 | 右中Canvas频谱/热力图+标尺+缩放 |
| 参数面板 | 仪器控制参数 | 右下instrument-panel |
| 标尺系统 | 坐标轴刻度+网格 | 共享zoomRuler.ts模块 |
| 缩放系统 | auto/x/y/overall四模式 | 滚轮缩放+拖拽平移+双击重置 |

### 3.2 五面板布局

```
┌──────────────────────────────────────────────────────────────┐
│  IQ3D FUSION VIEWER     [Demo▶] [Stop■] [FPS: 60]  ●LIVE   │  ← TopBar (48px)
│  ═══════════════════════════════════════════════════════════  │
├────────────────────────────┬─────────────────────────────────┤
│                            │                                 │
│   时域波形 (Time Waveform) │   IQ星座图 (Constellation)      │
│   I路 ─── Q路 ───         │       ●  ●                      │
│   ┌──────────────────┐    │     ●    ●  ●                    │
│   │  ~~~~/\~~~~      │    │   ●  ● ●   ●                    │
│   │  ~~~~\/~~~~      │    │     ●   ● ●                     │
│   └──────────────────┘    │       ●  ●                       │
│   Time→ Amp↑  Zoom: AUTO │   I→ Q↑  EVM: 2.3%  Zoom: X     │
│                            │                                 │
├────────────────────────────┼─────────────────────────────────┤
│                            │                                 │
│   瀑布图 (Waterfall)       │   频谱分析 (Spectrum)            │
│   ┌──────────────────┐    │   ┌──────────────────┐          │
│   │ ▓▓░░▓▓░▓▓░░▓▓░  │    │   │ FFT频谱折线+填充  │          │
│   │ ░░▓▓░░▓▓░▓▓░░▓  │    │   │ 或 STFT时频热力图 │          │
│   │ ▓░░▓▓░░▓▓░▓▓░░  │    │   └──────────────────┘          │
│   │ ░▓▓░░▓▓░░▓▓░▓▓  │    │   Freq→ Mag↑  FFT|Hanning      │
│   └──────────────────┘    │                                 │
│   Freq→ Time↓ Zoom: AUTO │   ┌── 参数面板 ──────────────┐  │
│                            │   │ Instrument: FFT/Win/     │  │
│                            │   │   Spectrum/Color/Zoom    │  │
│                            │   │ Signal: Mod/SNR/Chirp    │  │
│                            │   │ Status: State/FPS/Wfall  │  │
│                            │   └──────────────────────────┘  │
├────────────────────────────┴─────────────────────────────────┤
│  ● LIVE | FPS: 60 | FFT: 4096pt | Waterfall: 200rows | BP-0062│  ← StatusBar
└──────────────────────────────────────────────────────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 静态配置

```typescript
type ColorMapName = 'viridis' | 'plasma' | 'inferno' | 'magma' | 'hot'
type ModulationType = 'QPSK' | 'QAM16' | 'BPSK' | 'Chirp-Linear' | 'Chirp-Nonlinear'
type DownsampleMethod = 'lttb' | 'minmax' | 'none'
type FFTPoints = 4096 | 8192 | 16384
type WindowFunction = 'rectangular' | 'hamming' | 'hanning' | 'blackman' | 'flattop'
type SpectrumMode = 'fft' | 'stft'
type ZoomMode = 'auto' | 'x' | 'y' | 'overall'

interface IQ3DConfig {
  fft_points: FFTPoints
  sample_rate_msps: number
  center_freq_ghz: number
  waterfall_depth: number
  color_map: ColorMapName
  downsample_method: DownsampleMethod
  modulation: ModulationType
  snr_db: number
  symbol_rate_msps: number
  window_function: WindowFunction
  spectrum_mode: SpectrumMode
  stft_window_size: number
  stft_hop_size: number
  chirp_f0_hz: number
  chirp_f1_hz: number
  chirp_duration_ms: number
  zoom_mode: ZoomMode
}
```

#### 4.1.2 SHM矢量订阅 (VEC-DISPLAY)

```typescript
interface IQDataFrame {
  i_data: Float32Array
  q_data: Float32Array
  fft_magnitude: Float32Array
  stft_magnitude: Float32Array | null
  timestamp: number
}
```

### 4.2 输出契约 (VEC-CONTROL)

```typescript
interface IQ3DControlFrame {
  frame_type: 'iq3d_command'
  timestamp: number
  command: 'start_capture' | 'stop_capture' | 'set_params'
  params: Record<string, any>
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | IDLE | 初始化Canvas |
| IDLE | 点击Demo按钮 | DEMO | 启动合成数据生成 |
| DEMO | 点击Stop | IDLE | 停止数据生成 |
| IDLE | SHM数据到达 | LIVE | 渲染真实数据 |
| LIVE | SHM数据中断 | IDLE | 显示断开提示 |
| * | 参数变更 | * | 重新配置渲染管线 |
| * | Zoom模式变更 | * | syncZoomMode同步mode+isAuto |

---

## 五、UI设计规范

### 5.1 色彩体系

| 角色 | 色值 | 用途 |
|------|------|------|
| 深空黑 | `#0A0E17` | 主背景 |
| 暗夜灰 | `#141B2D` | 面板背景 |
| 炭灰色 | `#1E2A3A` | 边框/网格线 |
| 皇室金 | `#D4A843` | 主强调色/FFT频谱 |
| 翡翠绿 | `#00C9A7` | I路波形/LIVE状态/星座点 |
| 天际蓝 | `#3B82F6` | Q路波形 |
| 烈焰红 | `#EF4444` | 错误/告警 |
| 月光白 | `#E8ECF1` | 主文本 |
| 银灰色 | `#8892A4` | 次要文本/标尺刻度 |
| 标尺背景 | `#0D1220` | Y轴标尺区域 |
| 网格线 | `rgba(30,42,58,0.4)` | 绘图区网格 |

### 5.2 Canvas渲染规范

| 面板 | Canvas尺寸 | 刷新率 | 数据类型 | 缩放模式 |
|------|-----------|--------|----------|----------|
| 时域波形 | 自适应 | 60fps | Float32Array I/Q | auto/x/y/overall |
| 瀑布图 | 自适应 | 10fps追加 | Float32Array FFT幅度 | auto/x/y/overall |
| 星座图 | 自适应 | 30fps | Float32Array I/Q散点 | auto/x/y/overall |
| 频谱分析 | 自适应 | 60fps | Float32Array FFT/STFT | auto/x/y/overall |

### 5.3 标尺系统

| 属性 | 值 | 说明 |
|------|-----|------|
| RULER_SIZE | 36px | Y轴标尺宽度 |
| LABEL_SIZE | 14px | X轴标签高度 |
| 刻度字体 | 9px JetBrains Mono | 标尺刻度数字 |
| 标签字体 | 8px JetBrains Mono | 轴标签 |
| 刻度算法 | niceStep | 自适应1/2/5/10序列 |
| SI换算 | G/M/k/m | 自动单位换算 |
| 网格线 | rgba(30,42,58,0.4) | 绘图区背景网格 |

### 5.4 缩放系统

| 模式 | 行为 | 交互 |
|------|------|------|
| auto | 自动适配数据范围，禁止手动缩放 | 双击重置 |
| x | 仅X方向缩放 | 滚轮X缩放+X拖拽平移 |
| y | 仅Y方向缩放 | 滚轮Y缩放+Y拖拽平移 |
| overall | XY同时缩放 | 滚轮整体缩放+XY拖拽平移 |

**缩放交互**：
- 滚轮：以鼠标位置为中心缩放，factor=0.9/1.1
- 拖拽：mousedown→mousemove→mouseup平移
- 双击：resetZoom恢复初始状态
- 范围限制：scaleX/scaleY ∈ [1, 100]

### 5.5 瀑布图色映射

| 色映射名 | 渐变 | 用途 |
|----------|------|------|
| viridis | 深紫→青→黄 | 默认，感知均匀 |
| plasma | 深紫→粉→黄 | 高对比度 |
| inferno | 黑→红→黄 | 热力图 |
| hot | 黑→红→黄→白 | 经典热力 |
| magma | 黑→紫→粉 | 暗色主题 |

### 5.6 FFT窗函数

| 窗函数 | 相干增益 | 特点 |
|--------|----------|------|
| rectangular | 1.0 | 无衰减，最高频率分辨率 |
| hamming | 0.54 | 低旁瓣，通用 |
| hanning | 0.5 | 平滑衰减，频谱分析默认 |
| blackman | 0.42 | 极低旁瓣，高动态范围 |
| flattop | 0.216 | 幅度精度最高，频率分辨率低 |

---

## 六、技术架构

### 6.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端框架 | Vue 3 | 与画廊统一 |
| 开发语言 | TypeScript | 类型安全 |
| 构建工具 | Vite | 画廊项目内 |
| 渲染引擎 | Canvas 2D API | 零依赖，沙箱透明 |
| UI样式 | Tailwind CSS v3 + instrument体系 | 复用 |
| 状态管理 | Pinia | 参数+数据状态 |
| FFT算法 | Cooley-Tukey | O(N log N)位反转+蝶形运算 |

### 6.2 项目目录结构

```
iq3d-fusion-ui/
├── public/
│   └── favicon.svg
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── style.css
│   ├── components/
│   │   ├── FusionTopBar.vue       # 顶部栏（IQ3D FUSION + Demo/Stop + FPS + LED）
│   │   ├── FusionStatusBar.vue    # 状态栏（LIVE + FPS + FFT点数 + 瀑布行数 + BP-0062）
│   │   ├── TimeWaveform.vue       # 时域波形Canvas（I/Q双通道+标尺+缩放）
│   │   ├── WaterfallPlot.vue      # 瀑布图Canvas（纹理追加+标尺+缩放+plotW同步）
│   │   ├── ConstellationPlot.vue  # IQ星座图Canvas（散点+EVM+标尺+缩放）
│   │   ├── SpectrumPlot.vue       # 频谱分析Canvas（FFT频谱/STFT热力图+标尺+缩放）
│   │   └── ParameterPanel.vue     # 参数面板（Instrument/Signal/Status三组）
│   ├── stores/
│   │   └── fusionStore.ts         # 参数+数据+渲染状态+瀑布宽度同步
│   ├── renderers/
│   │   ├── waterfallRenderer.ts   # 瀑布图渲染函数（bufferCanvas纹理追加+色映射LUT）
│   │   └── colorMaps.ts           # 5种色映射LUT生成函数
│   ├── generators/
│   │   └── demoDataGenerator.ts   # 合成Demo数据（5种调制+Cooley-Tukey FFT+STFT+AWGN）
│   ├── types/
│   │   └── index.ts               # 类型定义（8种类型+2种接口）
│   └── utils/
│       ├── zoomRuler.ts           # 共享缩放状态+标尺渲染（ZoomState+drawRulers+syncZoomMode）
│       ├── windowFunctions.ts     # 5种FFT窗函数+相干增益
│       └── downsample.ts          # LTTB/MinMax降采样函数
├── tailwind.config.js
├── vite.config.ts                 # base: '/iq3d-fusion-ui/'
├── tsconfig.json
├── index.html
└── package.json
```

### 6.3 渲染函数接口

```typescript
// zoomRuler.ts — 共享缩放+标尺模块
function createZoomState(): ZoomState
function syncZoomMode(zs: ZoomState, mode: ZoomMode): void
function resetZoom(zs: ZoomState): void
function handleWheel(zs: ZoomState, e: WheelEvent, canvasWidth: number, canvasHeight: number): void
function handleDragStart(zs: ZoomState, e: MouseEvent): DragState | null
function handleDragMove(zs: ZoomState, e: MouseEvent, drag: DragState): void
function drawRulers(ctx: CanvasRenderingContext2D, width: number, height: number, axis: AxisConfig, zs: ZoomState): { plotX: number; plotY: number; plotW: number; plotH: number }

// waterfallRenderer.ts — 瀑布图渲染模块
function waterfallAppend(rowData: Float32Array, colorMap: ColorMapName, canvasWidth: number): void
function waterfallRender(ctx: CanvasRenderingContext2D, width: number, height: number, colorMap: ColorMapName): void
function getWaterfallRows(): number
function resetWaterfall(): void

// colorMaps.ts — 色映射LUT
function generateColorMap(name: ColorMapName, size: number): Uint8ClampedArray

// windowFunctions.ts — FFT窗函数
function applyWindow(data: Float32Array, windowType: WindowFunction): Float32Array
function getWindowCoherenceGain(type: WindowFunction): number

// downsample.ts — 降采样
function downsampleLTTB(data: Float32Array, threshold: number): Float32Array
function downsampleMinMax(data: Float32Array, bins: number): Float32Array
```

### 6.4 Demo数据生成

```typescript
function generateDemoFrame(
  modulation: ModulationType,
  snrDb: number,
  numPoints: number,
  windowFn: WindowFunction,
  spectrumMode: SpectrumMode,
  stftWindowSize: number,
  stftHopSize: number,
  chirpF0: number,
  chirpF1: number,
  chirpDuration: number
): IQDataFrame

// 内部函数
function generateQPSKSymbols(n: number): { i: Float32Array; q: Float32Array }
function generateQAM16Symbols(n: number): { i: Float32Array; q: Float32Array }
function generateBPSKSymbols(n: number): { i: Float32Array; q: Float32Array }
function generateLinearChirp(n: number, f0: number, f1: number, duration: number): { i: Float32Array; q: Float32Array }
function generateNonlinearChirp(n: number, f0: number, f1: number, duration: number): { i: Float32Array; q: Float32Array }
function addAWGN(data: Float32Array, snrDb: number): Float32Array
function fftInPlace(re: Float32Array, im: Float32Array): void  // Cooley-Tukey O(N log N)
function computeFFTMagnitude(iData: Float32Array, qData: Float32Array, windowFn: WindowFunction): Float32Array
function computeSTFT(iData: Float32Array, qData: Float32Array, windowFn: WindowFunction, windowSize: number, hopSize: number): Float32Array
```

### 6.5 信号生成算法

| 信号类型 | 公式 | 参数 |
|----------|------|------|
| QPSK | I/Q ∈ {±0.707} | snr_db |
| QAM16 | I/Q ∈ {±0.316, ±0.105} | snr_db |
| BPSK | I ∈ {±1}, Q ≈ 0 | snr_db |
| Chirp-Linear | φ(t) = 2π(f₀t + (f₁-f₀)/(2T)t²) | f₀, f₁, T |
| Chirp-Nonlinear | f(t) = f₀ + (f₁-f₀)(t/T)² | f₀, f₁, T |
| AWGN | σ = √(P_signal / 10^(SNR/10)) | snr_db |

---

## 七、组件规范

### 7.1 TimeWaveform

**Props**: 无（从Store读取数据）

**Canvas渲染**: 双通道折线图，I路翡翠绿(#00C9A7)，Q路天际蓝(#3B82F6)，标尺+网格背景，Y轴自动缩放

**缩放交互**: syncZoomMode + handleWheel + handleDragStart/Move + resetZoom(双击)

**信息栏**: I■ Q■ | Zoom: AUTO/X/Y/OVERALL

### 7.2 WaterfallPlot

**Props**: 无（从Store读取数据）

**Canvas渲染**: bufferCanvas纹理追加模式，每帧追加1行FFT幅度数据，Y轴自动滚动，色映射LUT查找

**关键机制**: drawRulers后用plotW同步Store.waterfallCanvasWidth，确保buffer宽度与绘图区域匹配；ctx.translate(plotX, plotY)对齐绘图坐标

**缩放交互**: syncZoomMode + handleWheel + handleDragStart/Move + resetZoom(双击)

**信息栏**: color_map | Zoom: AUTO/X/Y/OVERALL

### 7.3 ConstellationPlot

**Props**: 无（从Store读取数据）

**Canvas渲染**: I/Q散点图，渐变透明度（新点亮旧点暗），参考网格线，EVM计算

**缩放交互**: syncZoomMode + handleWheel + handleDragStart/Move + resetZoom(双击)

**信息栏**: modulation | Zoom: AUTO/X/Y/OVERALL

**叠加信息**: EVM: x.x% | Points: N

### 7.4 SpectrumPlot

**Props**: 无（从Store读取数据）

**Canvas渲染双模式**:
- **FFT模式**: 频谱折线(皇室金)+渐变填充，X轴频率(Hz)，Y轴幅度
- **STFT模式**: 时频热力图(putImageData)，X轴时间(s)，Y轴频率(Hz)，色映射LUT

**缩放交互**: syncZoomMode + handleWheel + handleDragStart/Move + resetZoom(双击)

**信息栏**: FFT/STFT | window_function | Zoom: AUTO/X/Y/OVERALL

### 7.5 ParameterPanel

**布局**: instrument-panel容器，三组参数

**Instrument组**: FFT点数(select) / 窗函数(select) / 频谱模式(select) / STFT参数(条件显示) / 采样率(digital-readout) / 中心频率(digital-readout) / 色映射(select) / 降采样方法(select) / 缩放模式(select)

**Signal组**: 调制类型(select: QPSK/QAM16/BPSK/Chirp-Linear/Chirp-Nonlinear) / SNR(slider: 0~40dB) / Chirp参数(条件显示: f₀/f₁/Duration) / 符号率(非Chirp时显示)

**Status组**: State / FPS / Waterfall行数

### 7.6 FusionTopBar

**布局**: 固定顶部，h-12，与画廊TopBar风格一致

**左侧**: "IQ3D FUSION VIEWER" (text-royal-gold)

**右侧**: Demo按钮(instrument-btn) / Stop按钮(instrument-btn) / FPS读数(digital-readout) / LED状态灯

### 7.7 FusionStatusBar

**布局**: 固定底部，h-8

**内容**: LED + "LIVE"/"DEMO" + FPS + FFT点数 + 瀑布行数 + BP-0062

---

## 八、安全约束

| 约束 | 实现 |
|------|------|
| 只读UI | 消费者域只订阅数据，不修改SHM |
| Demo标注 | 合成数据标注`preprocessed: true` |
| 沙箱透明 | 调试期全明文，无加密混淆 |
| 内存限制 | Canvas纹理+缓冲区 ≤ 500MB |
| 缩放安全 | scaleX/scaleY ∈ [1, 100]，防止无限缩放 |

---

## 九、实现状态追踪

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 主页面 | App.vue | ✅ | 五面板Grid: 2列+280px参数列 |
| 顶部栏 | FusionTopBar.vue | ✅ | Demo/Stop/FPS/LED |
| 状态栏 | FusionStatusBar.vue | ✅ | LIVE/DEMO/FPS/FFT/Wfall/BP-0062 |
| 时域波形 | TimeWaveform.vue | ✅ | I/Q双通道+标尺+缩放 |
| 瀑布图 | WaterfallPlot.vue | ✅ | bufferCanvas纹理追加+标尺+缩放+plotW同步 |
| 星座图 | ConstellationPlot.vue | ✅ | 散点+EVM+标尺+缩放 |
| 频谱分析 | SpectrumPlot.vue | ✅ | FFT频谱+STFT热力图+标尺+缩放 |
| 参数面板 | ParameterPanel.vue | ✅ | Instrument/Signal/Status三组 |
| 状态管理 | fusionStore.ts | ✅ | 17个setter+tick+瀑布宽度同步 |
| 瀑布渲染 | waterfallRenderer.ts | ✅ | bufferCanvas+色映射LUT+MAX_ROWS=500 |
| 色映射 | colorMaps.ts | ✅ | 5种LUT |
| Demo生成 | demoDataGenerator.ts | ✅ | 5种调制+Cooley-Tukey FFT+STFT+AWGN |
| 窗函数 | windowFunctions.ts | ✅ | 5种窗+相干增益 |
| 降采样 | downsample.ts | ✅ | LTTB+MinMax |
| 标尺+缩放 | zoomRuler.ts | ✅ | ZoomState+drawRulers+syncZoomMode+4种交互 |
| 类型定义 | types/index.ts | ✅ | 8种类型+2种接口 |
| 全局样式 | style.css | ✅ | instrument体系+Tailwind v3 |
| Vite配置 | vite.config.ts | ✅ | base: '/iq3d-fusion-ui/' |
| 画廊集成 | gallery.json | ✅ | cat-instrument分类 |

---

## 十、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| Canvas 2D非WebGL | 性能上限低于WebGL，百万级数据点需降采样 | 后续升级LightningChart JS |
| 无真实SHM接入 | 沙箱调试期使用合成数据 | SHM矢量层就绪后接入 |
| 无切片联动 | 十字准线跨面板联动未实现 | v1.2添加 |
| STFT缩放未实现 | STFT模式用putImageData直接写入，缩放时需重新计算像素映射 | v1.2优化 |
| 瀑布图缩放仅标尺 | 瀑布图内容为纹理追加模式，缩放只影响标尺刻度，不影响纹理内容 | v1.2优化 |
| Demo点数固定1024 | 为保证60fps帧率，DEMO_POINTS=1024 | 后续可配置 |

---

## 十一、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-15 | 初始版本，四面板布局，Canvas 2D渲染，Demo数据生成，5种色映射 |
| v1.1.0 | 2026-05-15 | 新增SpectrumPlot(FFT/STFT双模式)；新增5种FFT窗函数；新增Chirp-Linear/Nonlinear信号源；新增标尺+缩放系统(auto/x/y/overall)；修复缩放syncZoomMode同步bug；修复瀑布图plotW宽度同步bug；Cooley-Tukey FFT替代O(N²) DFT；五面板Grid布局 |

---

## 附录

### A. 参考文档

- [时频IQ三维融合可视化系统设计蓝图_v2.md](../../时频IQ三维融合可视化系统设计蓝图_v2.md) - 原始设计蓝图v2.0
- [genetic-code.md](../../specs/constitution/genetic-code.md) - 项目基因提取
- [BP-0034-spectrum-ui.md](./BP-0034-spectrum-ui.md) - 频谱仪UI蓝图
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图

### B. 术语表

| 术语 | 说明 |
|------|------|
| IQ | 同相/正交信号对 |
| Waterfall | 瀑布图，频谱随时间演变的强度图 |
| Constellation | 星座图，I-Q平面散点图 |
| EVM | 误差向量幅度，调制质量指标 |
| LTTB | Largest-Triangle-Three-Buckets降采样算法 |
| AWGN | 加性高斯白噪声 |
| STFT | 短时傅里叶变换，分窗FFT输出时频热力图 |
| Chirp | 线性/非线性调频信号 |
| Cooley-Tukey | O(N log N) FFT算法，位反转+蝶形运算 |
| niceStep | 自适应刻度算法，1/2/5/10序列 |
| plotW | 标尺占用后的实际绘图区域宽度 |
| syncZoomMode | 同步ZoomState.mode和isAuto的函数 |
