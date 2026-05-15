# FFT信号分解与合成演示系统消费者蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0063-FFT-DECOMPOSITION
> **版本**: v1.3.0
> **日期**: 2026-05-15
> **功能域**: consumer
> **状态**: CODE_DONE

---

## 一、元数据

| 属性 | 值 |
|------|-----|
| 蓝图ID | BP-0063-FFT-DECOMPOSITION |
| 函数名 | fft_decomposition_init / fft_decomposition_tick |
| 生产者名称 | FFTDecompositionConsumer |
| 语言 | Vanilla JavaScript (ES2022) |
| 输入协议 | SharedArrayBuffer + Atomics标志位 |
| 输出协议 | Canvas 2D渲染 + DOM控制面板 |
| 安全等级 | P1 (内部教学演示工具) |
| 版本 | v1.3.0 |

---

## 二、设计意图

### 2.1 核心目标

在浏览器环境中，设计并实现一个交互式FFT信号分解与合成演示系统。通过动画形式展示任意单调曲线信号的频域分解过程与时域合成过程，帮助用户直观理解傅里叶变换的数学原理。同时，将商场模式的四域架构完整映射到纯前端技术栈，验证浏览器内SHM总线的可行性。

### 2.2 时间哲学

动画不是越快越好，也不是越慢越好。人类视觉暂留约100ms，意识感知阈值约500ms。一个谐波分量的加入，必须在屏幕上停留足够长的时间，让用户的视网膜捕捉到"新的涟漪"，但又不能长到让耐心耗尽。最佳动画时长，是生理极限与认知耐心的黄金交叉点。

**核心公式**：
```
delta_k = speed / actual_fps        // 每帧增加的分量数
总时间(秒) = target_k / speed       // 动画总时长仅由目标分量数与速度决定
speed = target_k / duration         // 由目标时长反推速度（方案A）
```

### 2.3 创世纪项目适配

| 项目特点 | 设计适配 |
|----------|----------|
| 三界隔离 | Worker=生产者域，主线程协调器=商场域，Canvas UI=消费者域 |
| 函数范式 | Worker脚本内纯函数设计，禁止DOM/this/闭包状态 |
| SHM矢量空间 | SharedArrayBuffer实现跨线程零拷贝数据通道 |
| 标志位机制 | Int32Array + Atomics实现无锁状态同步，高16位商场域/低16位Worker域 |
| 产品与相 | Float32Array为产品（客观数据），Canvas渲染风格为相（主观呈现） |
| 仪器风格 | 复用instrument-panel/header/btn/LED全局样式体系 |
| Nginx钢铁脊椎 | 生产部署需配置COOP/COEP响应头 |

### 2.4 前端技术映射

| 商场概念 | 前端实现 | 说明 |
|----------|----------|------|
| SHM总线 | `SharedArrayBuffer` | 浏览器原生共享内存，跨线程零拷贝数据通道 |
| 标志位 | `Int32Array` + `Atomics` | `Atomics.load/store/wait/notify`实现无锁状态同步 |
| Worker | `new Worker('xxx.js')` | 浏览器Web Worker，独立线程运行生产者函数 |
| 矢量 | `Float32Array`视图 | 基于SAB的TypedArray视图，同一内存多线程共享 |
| 交易员 | 主线程调度逻辑 | `postMessage`轻量控制信令 + SAB数据总线 |
| 消费者域 | 主线程 + Canvas 2D | `requestAnimationFrame`驱动渲染循环 |
| 商场域 | 主线程协调器 | 负责Worker生命周期、SAB分配、交易仲裁 |
| 沙箱 | 开发服务器 / localhost | SAB要求安全上下文（COOP/COEP） |

### 2.5 设计决策

| 决策 | 原因 | 备选 | 影响 |
|------|------|------|------|
| Vanilla JavaScript | SAB/Atomics/Worker是底层API，框架封装增加复杂度 | Vue 3 + TypeScript | 无法复用画廊Vue组件体系 |
| SharedArrayBuffer | 零拷贝跨线程数据共享，是SHM总线的浏览器肉身 | postMessage + ArrayBuffer拷贝 | 需COOP/COEP安全上下文 |
| Atomics标志位 | 无锁仲裁，高16位/低16位表达域分离 | Mutex锁 | 需Int32Array对齐 |
| Canvas 2D渲染 | 零第三方依赖，符合函数范式 | WebGL / Three.js | 需自行实现降采样 |
| Express开发服务器 | 最简COOP/COEP头配置 | Vite(CORS限制) | 生产部署需Nginx |
| 三屏垂直布局 | 时域+频域+合成三区同屏 | 单面板切换 | 需CSS flex布局 |
| **自适应时长控制(方案A)** | 固定speed范围(1~60fps)与target_k动态范围(5~512)不匹配，同一speed对少量分量过快、对全频谱过慢 | 方案B:动态speed范围(认知负担重) / 方案C:三段预设(零灵活性) | UI移除speed滑条，改为duration滑条(3~30秒)，内部自动计算speed |
| **帧率动态修正** | rAF实际频率因显示器刷新率(60/120/144Hz)而异，固定delta_k=speed/60导致高刷屏动画加速 | 无修正 | 需每帧测量实际fps并修正delta_k |
| **后台标签页暂停** | rAF在后台标签页被节流至1fps或暂停，导致动画时长膨胀60倍 | 无处理 | 需Page Visibility API检测后台并暂停 |
| **主线程降级模式** | 画廊iframe中SAB可能不可用，导致页面空白 | 仅SAB模式 | main.js内联FFT/信号生成，SAB不可用时自动降级到主线程计算 |
| **窗函数时域压缩** | 窗函数占满N点时频域仅2~3个bin，合成动画瞬间完成 | 载波调制(频域效果弱) | Hanning压缩到8%/Hamming 6%/Blackman 5%，两侧补零，频谱展宽12~20倍 |
| **FFT Size仅2的幂** | Cooley-Tukey位反转+蝶形运算要求N=2^k，非2的幂导致索引越界或结果错误 | slider连续值 | FFT Size改为select下拉，仅允许64/128/256/512/1024/2048/4096 |
| **参数变更自动重跑** | 改FFT Size/信号类型后旧数据残留，新旧波形叠加 | 手动重置 | 参数change事件200ms防抖后自动清缓存+重跑pipeline |
| **股票K线信号** | 纯随机游走缺乏趋势特征，教学价值有限 | 保持random | stock信号添加趋势段+波动率聚集+均值回归；stock-diff差分信号揭示隐藏趋势 |

---

## 三、概念模型

### 3.1 核心概念映射

| 概念借喻 | 实际含义 | UI映射 |
|----------|----------|--------|
| FFT商场 | 主线程协调器 | SAB内存池+Worker池+交易仲裁 |
| 信号工厂 | signal_worker.js / main.js内联 | 7种信号生成纯函数 |
| 频域工厂 | fft_worker.js | Cooley-Tukey FFT + 信号重建 |
| 时域波形 | 原始信号可视化 | 左上Canvas折线图 |
| 频域频谱 | FFT幅度谱可视化 | 右上Canvas柱状图 |
| 合成动画 | 分量逐次叠加还原 | 下方Canvas叠加波形 |
| 控制面板 | 参数与交易控制 | DOM下拉/滑条/按钮 |
| 播放时长 | 动画总时长(3~30秒) | duration滑条(替代speed滑条) |

### 3.2 四域架构（浏览器内映射）

```
┌─────────────────────────────────────────────────────────────┐
│                    AICoder 域（设计时）                        │
│         蓝图生成 │ 代码编译 │ 文档维护 │ 基因提取              │
└─────────────────────────────────────────────────────────────┘
                              ↓ 编译交付
┌─────────────────────────────────────────────────────────────┐
│                    生产者域（Web Worker）                      │
│   signal_worker.js ──┐                                       │
│   fft_worker.js    ──┼──→ 写入 SharedArrayBuffer（产品）       │
└─────────────────────────────────────────────────────────────┘
                              ↓ Atomics 标志位同步
┌─────────────────────────────────────────────────────────────┐
│                    商场域（主线程协调器）                       │
│   SAB 内存池 │ 矢量状态机 │ Worker 池 │ 交易仲裁 │ 安全铁律    │
└─────────────────────────────────────────────────────────────┘
                              ↓ 主线程读取 SAB
┌─────────────────────────────────────────────────────────────┐
│                    消费者域（主线程 UI）                        │
│   Canvas 2D 渲染 │ DOM 控制面板 │ rAF 动画循环 │ 相的投影      │
│   duration自适应 │ 帧率动态修正 │ 后台暂停恢复                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 三屏垂直布局

```
┌─────────────────────────────────────────────────────────────┐
│  控制面板区 (15%高度)                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 信号选择  │ │ 参数调节  │ │ 时长控制  │ │ 状态显示  │       │
│  │ type     │ │ N/FFT/K │ │ duration │ │ FPS/进度 │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├──────────────────────────┬──────────────────────────────────┤
│                          │                                  │
│   时域波形视图 (48%宽度)   │   频域频谱视图 (48%宽度)         │
│                          │                                  │
│   Canvas 2D 直接绘制       │   Canvas 2D 柱状图               │
│   青色 #00f0ff            │   紫色 #bd00ff                   │
│                          │                                  │
├──────────────────────────┴──────────────────────────────────┤
│                                                             │
│   信号合成动画区 (40%高度)                                    │
│                                                             │
│   频谱分量逐次叠加还原原始信号                                  │
│   原始信号：暗青色 (透明度0.3)                                 │
│   重建信号：亮绿色 #00ff9d (透明度1.0)                        │
│   进度指示：k/N 分量 · 剩余 ~s                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、行为契约

### 4.1 输入契约

#### 4.1.1 静态配置

```typescript
type SignalType = 'hanning' | 'hamming' | 'blackman' | 'stock' | 'stock-diff' | 'narrowband' | 'high_prf'

interface FFTDecompositionConfig {
  signal_type: SignalType
  sample_count: number
  fft_size: number
  components: number
  duration: number
  narrowband_bw: number
  narrowband_fc: number
  high_prf_prf: number
  high_prf_duty: number
  volatility: number
}

interface MallSystemConfig {
  pool_size: number
  slot_count: number
  max_vector_size: number
}

interface AnimEngineState {
  target_k: number
  current_k: number
  speed: number
  actual_fps: number
  is_paused: boolean
  is_background: boolean
}
```

#### 4.1.2 SAB矢量订阅

```typescript
interface SABVector {
  vector_id: number
  base_offset: number
  max_size: number
  owner: string | null
  flag: number
}

interface FFTResult {
  magnitude: Float32Array
  phase: Float32Array
  real: Float32Array
  imag: Float32Array
}
```

### 4.2 输出契约

```typescript
interface RenderFrame {
  time_signal: Float32Array
  spectrum: Float32Array
  reconstructed: Float32Array
  components_used: number
  progress: number
  estimated_remaining_ms: number
}
```

### 4.3 状态转换

| 当前状态 | 触发条件 | 下一状态 | 动作 |
|----------|----------|----------|------|
| INIT | 页面加载 | IDLE | 初始化SAB内存池+Worker池 |
| IDLE | 点击播放/参数变更 | COMPUTING | 分配SAB矢量→派发Worker |
| COMPUTING | Worker完成(READY标志) | RENDERING | 仲裁循环分发矢量给消费者 |
| RENDERING | 消费者拷贝完成 | COMPUTING | 释放矢量→请求下一帧 |
| RENDERING | 动画完成(current_k≥target_k) | COMPLETED | 停止rAF，显示完成状态 |
| COMPLETED | 点击重播 | COMPUTING | 动画进度归零，重新派发 |
| * | 点击暂停 | PAUSED | cancelAnimationFrame |
| PAUSED | 点击播放 | RENDERING | 恢复rAF |
| * | 点击重置 | IDLE | 动画进度归零 |
| * | 标签页切后台 | BACKGROUND_PAUSED | 暂停rAF+计时 |
| BACKGROUND_PAUSED | 标签页回前台 | PAUSED | 保持暂停，等待用户手动继续 |

### 4.4 标志位仲裁协议

SAB中预留`Int32Array`作为全局标志区。每个矢量占用一个32位标志字。

```
Bit 31 ──────────────────────────────────────────────── Bit 0
┌──────────────────────────┬──────────────────────────┐
│   商场表达域 (高16位)      │   Worker表达域 (低16位)    │
│   写入权力：主线程(商场)    │   写入权力：Worker         │
├──────────────────────────┼──────────────────────────┤
│  bit08: DISPATCHED       │  bit00: READY (0x0001)    │
│  bit09: RELEASED         │  bit01: BUSY  (0x0002)    │
│                          │  bit02: ERROR (0x0004)    │
└──────────────────────────┴──────────────────────────┘
```

**矢量状态机**: IDLE(0x00) → BUSY(0x02) → READY(0x01) → DISPATCHED(0x01|0x0100) → RELEASED(0x0200) → IDLE(0x00)

---

## 五、UI设计规范

### 5.1 色彩体系

| 角色 | 色值 | 用途 |
|------|------|------|
| 深空黑 | `#0a0a0f` | 主背景 |
| 面板底色 | `#12121a` | 控制区背景 |
| 边框 | `#1e1e2e` | 分隔线 |
| 时域波形 | `#00f0ff` | 原始信号折线 |
| 频谱柱 | `#bd00ff` | FFT幅度柱状图 |
| 原始信号 | `#00f0ff33` | 合成区背景参考 |
| 重建信号 | `#00ff9d` | 合成区前景重建 |
| 主文本 | `#e0e0e0` | 全局文字 |
| 次要文本 | `#8899a6` | 标签文字 |
| 按钮激活 | `#00ff9d` | 播放按钮 |
| 按钮默认 | `#1e1e2e` | 暂停/重置按钮 |
| 进度条 | `#00ff9d` | 动画进度指示 |

### 5.2 Canvas渲染规范

| 面板 | Canvas区域 | 刷新率 | 数据类型 |
|------|-----------|--------|----------|
| 时域波形 | 左上48%宽×45%高 | 60fps | Float32Array信号 |
| 频域频谱 | 右上48%宽×45%高 | 60fps | Float32Array幅度 |
| 合成动画 | 下方100%宽×40%高 | 60fps | Float32Array重建信号 |

### 5.3 信号类型

| 信号类型 | 公式 | 参数 | 频谱特征 |
|----------|------|------|----------|
| Hanning窗(压缩) | w(n) = 0.5(1-cos(2πn/(W-1)))，W=N×8%，居中补零 | N | sinc型，主瓣+旁瓣，展宽~12倍 |
| Hamming窗(压缩) | w(n) = 25/46-21/46·cos(2πn/(W-1))，W=N×6%，居中补零 | N | sinc型，更低旁瓣，展宽~17倍 |
| Blackman窗(压缩) | w(n) = a₀-a₁cos(φ)+a₂cos(2φ)，W=N×5%，居中补零 | N | sinc型，极低旁瓣，展宽~20倍 |
| 股票K线 | P[n]=P[n-1]+drift+shock+reversion，趋势段+波动率聚集+均值回归 | N, volatility | 1/f型，低频能量集中 |
| 差分K线 | diff[n]=P[n]-P[n-1]，高通滤波去除累积漂移 | N, volatility | 宽带，趋势段正负分明 |
| 窄带脉冲 | sinc(bw·t)·cos(2π·fc·t/N) | N, bw, fc | 以fc为中心的窄带 |
| 高重频调制 | 脉冲序列: period=1/prf, duty控制 | N, prf, duty | 线状谱，prf间隔 |

**窗函数时域压缩原理**：傅里叶时频对偶性——时域越窄，频域越宽。原始窗函数占满N点，FFT后仅2~3个非零bin。压缩到5%~8%后两侧补零，频谱展宽12~20倍，需要更多分量才能完成重建，合成动画更有观赏性。

**股票信号教学价值**：FFT截断重建在L2范数意义下等价于最优低通趋势拟合。K=1为水平均值线，K=5为主趋势轮廓，K=N为完美还原。差分信号去除累积漂移后，趋势拟合的层次感更清晰——低频分量建立涨跌方向，高频分量添加日内波动。

### 5.4 动画时长控制（方案A：自适应时长）

#### 5.4.1 核心原理

将`speed`从"固定绝对值"改为"基于target_k的相对值"。UI上不再显示`speed`，而是显示**目标总时长**`duration`（秒）。系统内部自动计算：`speed = target_k / duration`。

#### 5.4.2 可感知性判定

| 时间区间 | 视觉体验 | 适用场景 | 判定 |
|----------|----------|----------|------|
| < 0.5秒 | 一闪而过，无法分辨单个谐波的加入 | 不适用教学 | ❌ 过快 |
| 0.5~3秒 | 快速浏览，隐约感知生长过程 | 快速预览 | ⚡ 快速 |
| 3~15秒 | 舒适观察，每个新谐波清晰可见 | **标准教学演示** | ✅ 最佳 |
| 15~60秒 | 慢速品味，适合逐帧分析 | 深度教学 | 🐢 慢速 |
| > 60秒 | 严重拖沓，用户注意力涣散 | 仅特殊需求 | ❌ 过慢 |

#### 5.4.3 全参数时间矩阵

| target_k \ duration | 3s | 5s | 8s | 10s | 15s | 20s | 30s |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **5** | 1.67 | 1.0 | 0.63 | 0.5 | 0.33 | 0.25 | 0.17 |
| **10** | 3.33 | 2.0 | 1.25 | 1.0 | 0.67 | 0.5 | 0.33 |
| **50** | 16.67 | 10.0 | 6.25 | 5.0 | 3.33 | 2.5 | 1.67 |
| **100** | 33.33 | 20.0 | 12.5 | 10.0 | 6.67 | 5.0 | 3.33 |
| **200** | 66.67 | 40.0 | 25.0 | 20.0 | 13.33 | 10.0 | 6.67 |
| **512** | 170.67 | 102.4 | 64.0 | 51.2 | 34.13 | 25.6 | 17.07 |

（表中数值为内部speed = target_k / duration，单位：分量/秒）

#### 5.4.4 推荐配置

**按target_k范围推荐**：

| target_k范围 | 推荐duration | 内部speed | 体验描述 |
|:---|:---:|:---:|:---|
| 1~10 | 3~5秒 | 2~3 fps | 少量分量，慢速品味每个谐波 |
| 11~50 | 5~8秒 | 6~10 fps | 中等分量，快速浏览主要频率 |
| 51~200 | 8~15秒 | 13~25 fps | 较多分量，观察频谱收敛过程 |
| 201~512 | 10~20秒 | 25~50 fps | 全频谱，高速生长避免拖沓 |

**按使用场景推荐**：

| 场景 | target_k | duration | speed | 说明 |
|:---|:---:|:---:|:---:|:---|
| 课堂教学 | 10 | 8秒 | 1.25 fps | 教师讲解每个谐波的物理意义 |
| 自学探索 | 50 | 10秒 | 5 fps | 用户自由拖动分量数，观察变化 |
| 快速验证 | 512 | 5秒 | 102.4 fps | 工程师验证FFT正确性 |
| 演示汇报 | 20 | 6秒 | 3.33 fps | 投影大屏，观众距离远，需更慢 |
| 移动端 | 20 | 4秒 | 5 fps | 屏幕小，注意力碎片化 |

#### 5.4.5 控制面板UI

```html
<div class="panel-section">
    <label>播放时长</label>
    <input type="range" id="play-duration" min="3" max="30" step="1" value="8">
    <span id="duration-display">8 秒</span>
</div>
<div class="panel-section">
    <label>预计帧数</label>
    <span id="frame-estimate">480 帧</span>
</div>
```

### 5.5 帧率动态修正

#### 5.5.1 显示器刷新率影响

| 显示器 | rAF频率 | delta_k偏差 | 修正公式 |
|:---|:---|:---|:---|
| 60Hz | 60fps | 基准，无偏差 | `delta_k = speed / 60` |
| 120Hz | 120fps | 动画快2倍 | `delta_k = speed / 120` |
| 144Hz | 144fps | 动画快2.4倍 | `delta_k = speed / 144` |
| 30Hz(省电) | 30fps | 动画慢2倍 | `delta_k = speed / 30` |

**修正策略**：在rAF回调中动态测量帧间隔，修正delta_k：

```javascript
let last_frame_time = 0
let actual_fps = 60

function frame(timestamp) {
    if (last_frame_time > 0) {
        const delta_ms = timestamp - last_frame_time
        actual_fps = 1000 / delta_ms
        anim_engine.set_actual_fps(actual_fps)
    }
    last_frame_time = timestamp
}
```

#### 5.5.2 后台标签页节流

| 场景 | rAF行为 | 时间影响 |
|:---|:---|:---|
| Chrome后台 | 暂停，回到前台恢复 | 动画"冻结"，不丢失进度 |
| Firefox后台 | 降至1fps | 动画极慢播放，时长膨胀60倍 |
| 最小化窗口 | 取决于浏览器策略 | 不可预测 |

**处理策略**：使用Page Visibility API检测后台并暂停：

```javascript
document.addEventListener('visibilitychange', () => {
    if (document.hidden) {
        anim_engine.pause()
    }
})
```

回到前台后保持暂停状态，等待用户手动点击播放继续。

### 5.6 极端边界处理

| 边界情况 | 时间表现 | 处理策略 |
|:---|:---|:---|
| target_k=1, duration=3s | 1个分量播放3秒 | 允许，观察直流分量建立过程 |
| target_k=512, duration=3s | speed=170.7fps | 每帧增加2.84个分量，frac插值保证丝滑 |
| duration=30s, 用户切后台 | 动画暂停 | 回来后保持暂停，用户手动继续 |
| 用户疯狂拖动components | 每次释放触发重新计算speed | 防抖200ms，避免频繁重置动画 |
| 4K显示器(60Hz) vs 手机(90Hz) | 同参数下手机快1.5倍 | 动态测量actual_fps修正delta_k |

---

## 六、技术架构

### 6.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 运行环境 | 浏览器 (ES2022) | SharedArrayBuffer + Atomics + Web Worker |
| 开发语言 | Vanilla JavaScript | 零框架，纯函数设计 |
| 开发服务器 | Express.js | COOP/COEP响应头配置 |
| 渲染引擎 | Canvas 2D API | 零依赖，沙箱透明 |
| 跨线程通信 | SharedArrayBuffer | 零拷贝SHM总线 |
| 状态同步 | Atomics | 无锁标志位仲裁 |
| 动画控制 | requestAnimationFrame | 帧率动态修正+后台暂停 |
| 生产部署 | Nginx | COOP/COEP + 静态资源服务 |

### 6.2 项目目录结构

```
fft-decomposition-ui/
├── server.js                      # Express开发服务器（COOP/COEP头）
├── public/
│   ├── index.html                 # 入口页面
│   ├── main.js                    # 系统入口（初始化商场+Canvas+控制面板）
│   ├── mall_coordinator.js        # 商场域·主线程协调器（SAB池+Worker池+仲裁循环）
│   ├── render_engine.js           # 消费者域·Canvas 2D渲染引擎（帧率修正+后台暂停）
│   ├── anim_engine.js             # 消费者域·动画引擎（duration自适应+进度追踪）
│   ├── ui_controls.js             # 消费者域·DOM控制面板（duration滑条+防抖）
│   ├── signal_worker.js           # 生产者域·信号生成Worker（6种信号纯函数）
│   └── fft_worker.js              # 生产者域·FFT计算Worker（Cooley-Tukey+重建）
```

### 6.3 渲染函数接口

```javascript
// render_engine.js — 消费者域渲染引擎
function render_frame(ctx, time_signal, spectrum, reconstructed, config): void
function draw_waveform(ctx, data, x, y, w, h, color, alpha): void
function draw_spectrum(ctx, magnitudes, x, y, w, h, color): void
function interpolate_signals(from_signal, to_signal, t): Float32Array
function start_consumer_loop(mall_system, canvas, config): { stop, set_target }

// anim_engine.js — 消费者域动画引擎
function create_anim_engine(target_k, duration): AnimEngineHandle
function calculate_speed(target_k, duration): number
function get_speed_range(target_k): { min: number, max: number }
function anim_engine_tick(engine, delta_ms): { current_k, progress, is_complete }
function anim_engine_set_actual_fps(engine, fps): void
function anim_engine_set_target(engine, components, speed): void
function anim_engine_pause(engine): void
function anim_engine_resume(engine): void
function anim_engine_reset(engine): void

// ui_controls.js — 消费者域控制面板
function create_control_panel(container_id, initial_config): panel_handle
function bind_control_events(panel_handle, callbacks): void
function bind_duration_control(panel_handle, anim_engine): void
```

### 6.4 商场域函数接口

```javascript
// mall_coordinator.js — 商场域主线程协调器
function sab_pool_init(pool_size, slot_count, max_vector_size): mall_handle
function sab_alloc(mall, size_bytes, worker_id): number
function sab_write(sab, vector_id, base_offset, data, offset): void
function sab_copy(sab, base_offset, length): Float32Array
function sab_view(sab, base_offset, length): Float32Array
function sab_free(mall, vector_id): void
function trade_dispatch(mall, consumer_id): number
function init_mall_system(config): mall_system
function trade_arbiter_tick(mall, consumers): void
```

### 6.5 生产者域函数接口

```javascript
// signal_worker.js — 信号生成纯函数基因库
function generate_hanning_window(length): Float32Array
function generate_hamming_window(length): Float32Array
function generate_blackman_window(length): Float32Array
function generate_random_monotonic(length, step_scale): Float32Array
function generate_narrowband_pulse(length, bw, fc): Float32Array
function generate_high_prf_modulated(length, prf, duty): Float32Array

// fft_worker.js — FFT计算纯函数基因库
function compute_fft(signal, fft_size): { magnitude, phase, real, imag }
function reconstruct_signal(magnitude, phase, components, fft_size): Float32Array
```

### 6.6 SAB内存布局

| 区域 | 偏移 | 大小 | 用途 |
|------|------|------|------|
| 标志区 | 0 | slot_count × 4 | 所有矢量状态字 |
| 元数据区 | slot_count × 4 | slot_count × 8 | 每槽 base + max_size |
| 长驻数据区 | 元数据区末 | 固定 64KB | 全局配置、用户偏好、相的参数 |
| 交易数据区 | 长驻区末 | 剩余全部 | 动态矢量数据 |

**铁律**: 长驻数据区与交易数据区物理隔离。交易矢量不得溢出到长驻区。

---

## 七、组件规范

### 7.1 signal_worker.js

**职责**: 信号生成生产者

**消息协议**: `postMessage({ cmd: 'generate', sab, vector_id, base_offset, params })` → 计算并写入SAB → `Atomics.store(READY)` → `postMessage({ type: 'done', vector_id, length })`

**纯函数基因库**: 7种信号生成函数，禁止DOM/this/闭包状态

**信号类型**: hanning(压缩8%) / hamming(压缩6%) / blackman(压缩5%) / stock / stock-diff / narrowband / high_prf

### 7.2 fft_worker.js

**职责**: FFT计算与信号重建生产者

**消息协议**:
- FFT: `postMessage({ cmd: 'fft', sab, vector_id, base_offset, fft_size, signal_length })` → 结果写回SAB → `Atomics.store(READY)`
- 重建: `postMessage({ cmd: 'reconstruct', sab, vector_id, base_offset, fft_size, components })` → 重建信号写回SAB → `Atomics.store(READY)`

**FFT算法**: Cooley-Tukey O(N log N)，位反转+蝶形运算

**重建算法**: 选取前N个分量，IFFT还原时域信号

### 7.3 mall_coordinator.js

**职责**: 商场域主线程协调器

**核心功能**: SAB内存池管理、Worker生命周期、交易仲裁循环(~60fps)、矢量状态机

**公共API**: `request_signal()` / `request_fft()` / `request_reconstruct()` / `destroy()`

**仲裁循环**: `setInterval(trade_arbiter_tick, 16)`，扫描READY矢量分发给消费者

### 7.4 anim_engine.js

**职责**: 消费者域动画引擎，管理合成动画的进度、速度和帧率修正

**核心机制**:
- `calculate_speed(target_k, duration)` — 由目标时长反推speed
- `anim_engine_tick(engine, delta_ms)` — 每帧推进current_k，返回进度和完成状态
- `anim_engine_set_actual_fps(engine, fps)` — 动态修正帧率偏差
- `anim_engine_pause/resume/reset` — 暂停/恢复/重置动画

**进度追踪**: `current_k`从0增长到`target_k`，`progress = current_k / target_k`

**帧率修正**: `delta_k = speed / actual_fps`，actual_fps由rAF回调动态测量

**后台处理**: Page Visibility API检测后台，自动暂停，回前台保持暂停等待用户

### 7.5 render_engine.js

**职责**: 消费者域Canvas 2D渲染引擎

**渲染布局**: 时域波形(左上) + 频域频谱(右上) + 合成动画(下方)

**动画机制**: `requestAnimationFrame`驱动，`interpolate_signals()`实现分量叠加过渡

**降采样**: 消费者端根据Canvas像素宽度截取数据，保证60fps

**进度显示**: 合成区底部显示 `k/N 分量 · 剩余 ~s` 进度信息

### 7.6 ui_controls.js

**职责**: 消费者域DOM控制面板

**控件**:
- 信号选择(select: 7种信号类型)
- FFT点数(select: N=2^k，仅64/128/256/512/1024/2048/4096)
- 合成分量(slider: 1~512)
- **播放时长**(slider: 3~30秒，替代原anim_speed滑条)
- 预计帧数(只读显示: duration×60)
- 信号参数(Volatility/NB BW/NB Fc/PRF/Duty)
- 播放/暂停/重置按钮

**事件绑定**:
- duration变更 → `calculate_speed(target_k, duration)` → `anim_engine.set_target(target_k, speed)`
- components变更 → 重新计算speed → 防抖200ms
- 信号类型/FFT Size/信号参数变更 → 200ms防抖后自动清缓存+重跑pipeline

---

## 八、安全约束

| 约束 | 实现 |
|------|------|
| COOP/COEP铁律 | 生产环境必须配置响应头，否则SAB被浏览器拦截 |
| Worker隔离 | Worker脚本内禁止DOM操作、禁止this、禁止闭包状态 |
| 表达域分离 | 高16位商场域/低16位Worker域，天然互斥无需锁 |
| SAB限时释放 | 每个矢量从READY到RELEASED最大驻留500ms，超时商场强制回收 |
| 长驻区隔离 | 配置类矢量存放在独立长驻区，与交易总线物理隔离 |
| 内存泄漏防护 | 每个sab_alloc必须有对应sab_free路径，商场定期扫描报告未释放矢量 |
| NaN/Inf拦截 | 商场在SAB视图层拦截非法数据并报告 |
| 动画防抖 | components滑条变更200ms防抖，避免频繁重置动画 |

### 三级环境铁律

| 阶段 | SAB权限 | 目的 |
|------|---------|------|
| 沙箱期 | 全透明读写 | localhost开发，逻辑验证 |
| 保密试运行 | Worker只写，主线程只读 | 验证数据流正确性 |
| 归档期 | 主线程只读，禁止Worker写入 | 系统固化，仅消费者读取历史快照 |

---

## 九、实现状态追踪

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 开发服务器 | server.js | ✅ | Express + COOP/COEP头 (port 3006) |
| 入口页面 | index.html | ✅ | 三屏垂直布局+控制面板+状态栏 |
| 系统入口 | main.js | ✅ | 主线程降级模式+内联FFT/信号生成+自动Demo+参数变更重跑 |
| 商场协调器 | mall_coordinator.js | ✅ | SAB池+Worker池+仲裁循环（SAB可用时） |
| 动画引擎 | anim_engine.js | ✅ | duration自适应+帧率修正+后台暂停 |
| 渲染引擎 | render_engine.js | ✅ | Canvas 2D三区渲染+进度显示+频谱高亮 |
| 控制面板 | ui_controls.js | ✅ | 7种信号+N=2^k下拉+duration滑条+防抖 |
| 信号Worker | signal_worker.js | ✅ | 7种信号纯函数（含窗函数压缩+stock/stock-diff） |
| FFT Worker | fft_worker.js | ✅ | Cooley-Tukey + 信号重建 |
| 画廊集成 | gallery.json | ✅ | cat-instrument分类+COOP/COEP条件头 |

---

## 十、已知限制

| 限制 | 说明 | 计划 |
|------|------|------|
| SAB浏览器兼容 | SharedArrayBuffer需COOP/COEP安全上下文，部分浏览器不支持 | 已实现主线程降级模式 |
| 非Vue项目 | Vanilla JS无法复用画廊Vue组件体系 | 后续可考虑Vue 3 + Worker封装 |
| 无STFT | 仅支持FFT频谱分解，不支持短时傅里叶变换 | v1.4添加 |
| 无缩放/标尺 | Canvas渲染无标尺刻度和缩放交互 | v1.4添加 |
| 无真实信号接入 | 仅支持7种数学信号生成 | 后续接入ADC数据源 |
| 极端speed值 | target_k=512+duration=3s时speed=170.7fps，每帧增加2.84个分量 | frac插值保证丝滑，但视觉上分量不可分辨 |
| 画廊iframe SAB限制 | 画廊iframe中COOP/COEP可能不生效，SAB不可用 | 已实现主线程降级，画廊中自动切换 |

---

## 十一、版本历史

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| v1.0.0 | 2026-05-15 | 初始版本，SAB+Atomics+Worker四域架构，6种信号生成，Cooley-Tukey FFT，Canvas 2D三屏渲染，动画合成 |
| v1.1.0 | 2026-05-15 | 动画合成模块补充，interpolate_signals插值，anim_engine基础框架 |
| v1.2.0 | 2026-05-15 | 动画播放时间范围分析：自适应时长控制(方案A)替代固定speed；帧率动态修正(actual_fps)；后台标签页暂停(Page Visibility API)；新增anim_engine.js模块；全参数时间矩阵；推荐配置表；极端边界处理 |
| v1.3.0 | 2026-05-15 | 实现完成+调试修复：主线程降级模式(SAB不可用时自动切换)；窗函数时域压缩(Hanning 8%/Hamming 6%/Blackman 5%，频谱展宽12~20倍)；random→stock K线信号(趋势段+波动率聚集+均值回归)；新增stock-diff差分K线(揭示隐藏趋势)；FFT Size改为select仅2的幂(64~4096)；参数变更自动重跑pipeline(200ms防抖)；画廊集成(cat-instrument+COOP/COEP条件头)；自动Demo播放 |

---

## 附录

### A. 参考文档

- [FFT_Signal_Decomposition_Frontend_Blueprint_v1.0.md](../../FFT_Signal_Decomposition_Frontend_Blueprint_v1.0.md) - 原始设计蓝图v1.0
- [FFT_Animation_Time_Range_Analysis_v1.2.md](../../FFT_Animation_Time_Range_Analysis_v1.2.md) - 动画播放时间范围分析v1.2
- [BP-0062-iq3d-fusion.md](./BP-0062-iq3d-fusion.md) - IQ3D Fusion消费者蓝图（Vue 3版FFT可视化参考）
- [BP-0056-ui-gallery.md](./BP-0056-ui-gallery.md) - UI画廊蓝图
- [BP-0057-dual-channel-shm.md](../shm/BP-0057-dual-channel-shm.md) - 双通道SHM架构（SAB映射参考）

### B. 术语表

| 术语 | 说明 |
|------|------|
| SAB | SharedArrayBuffer，浏览器共享内存缓冲区，类比商场SHM总线 |
| Atomics | 浏览器原子操作API，标志位无锁仲裁的物理基础 |
| 产品 | 客观数据实体（时域采样/频域复数/幅度谱/相位谱），以Float32Array(SAB)形式流动 |
| 相 | 主观呈现要求（动画速度/配色/布局/渲染风格），由消费者定义 |
| 矢量 | SAB中的最小数据单元，Float32Array视图，具有唯一ID/基地址/长度/状态标志 |
| Worker | 浏览器Web Worker，生产者域的原子能力单元，禁止DOM/this/闭包状态 |
| 标志位 | SAB中Int32Array的一个元素，高16位商场表达域/低16位Worker表达域 |
| COOP/COEP | Cross-Origin-Opener-Policy / Cross-Origin-Embedder-Policy，启用SAB的必要HTTP响应头 |
| Cooley-Tukey | O(N log N) FFT算法，位反转+蝶形运算 |
| 信号重建 | 选取前N个FFT分量，IFFT还原时域信号 |
| duration | 目标播放时长（秒），替代固定speed的自适应控制参数 |
| actual_fps | rAF回调实测帧率，用于动态修正delta_k |
| delta_k | 每帧增加的分量数 = speed / actual_fps |
| 防抖 | 滑条变更后延迟200ms触发计算，避免频繁重置动画 |
| 时域压缩 | 窗函数在t轴上压缩到5%~8%，两侧补零，利用时频对偶性展宽频谱 |
| stock | 带趋势段+波动率聚集+均值回归的股票K线随机游走信号 |
| stock-diff | 股票K线差分信号 diff[n]=P[n]-P[n-1]，高通滤波揭示隐藏趋势 |
| 主线程降级 | SAB不可用时，FFT/信号生成在主线程内联计算，无需Worker |
| 趋势拟合 | FFT截断重建在L2范数意义下等价于最优低通趋势拟合 |

### C. 与BP-0062 IQ3D Fusion的对比

| 维度 | BP-0062 IQ3D Fusion | BP-0063 FFT Decomposition |
|------|---------------------|---------------------------|
| 技术栈 | Vue 3 + TypeScript + Pinia | Vanilla JavaScript |
| SHM实现 | 无（Demo模式，主线程计算） | 主线程降级 + SAB(可选) |
| FFT位置 | 主线程demoDataGenerator.ts | 主线程内联(降级) / fft_worker.js(SAB) |
| 信号生成 | 主线程generateDemoFrame | 主线程内联(降级) / signal_worker.js(SAB) |
| 跨线程通信 | 无（单线程） | SAB零拷贝 + postMessage控制信令 |
| 标志位 | 无 | Int32Array + Atomics无锁仲裁 |
| 动画控制 | 无（实时流式渲染） | duration自适应+帧率修正+后台暂停 |
| 画廊集成 | 已集成 | 已集成(cat-instrument+COOP/COEP条件头) |
| 教学价值 | 仪器可视化 | FFT原理演示 + 趋势拟合直觉 + 商场架构验证 |
| 信号类型 | 5种调制信号 | 7种(3窗函数压缩+2股票K线+窄带+高重频) |
