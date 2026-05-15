# 蓝图文档（定稿版）：BP-0034 -- 嵌入式精简频谱仪UI

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0034-SPECTRUM-UI |
| 蓝图名称 | 嵌入式精简频谱仪UI消费者 |
| 生产者 | Genesis-Worker-UI-001 |
| 提交时间 | 2026-05-11 |
| 版本 | 1.3.0 |
| 目标语言 | TypeScript (Vue 3 / React 18) |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

---

## 2. 功能概述

### 2.1 功能描述

嵌入式精简频谱仪UI消费者，作为商场内核的"相"呈现载体，负责将SHM矢量推送的FFT数据、仪器状态、颜色映射表渲染为可视化界面。支持电阻屏/电容屏触摸、实体按键、旋转编码器三模输入，实现频率域观测的最小完备交互集。

### 2.2 数据来源

| 来源类型 | 来源标识 | 说明 |
|----------|----------|------|
| SHM矢量 | `shm://vec/spectrum/fft` | FFT幅度数据（dB） |
| SHM矢量 | `shm://vec/spectrum/status` | 仪器状态矢量 |
| SHM矢量 | `shm://vec/spectrum/colormap` | 颜色映射LUT |
| 用户输入 | `event://button/*` | 按键/触摸/编码器事件 |

### 2.3 数据去向

| 去向类型 | 去向标识 | 说明 |
|----------|----------|------|
| 事件总线 | `event://spectrum/command` | 用户意图指令（频率调节/参数变更） |
| 渲染画布 | `canvas://spectrum/main` | 波形显示区Canvas |
| 状态显示 | `display://spectrum/status` | 顶部状态条/底部信息条 |

### 2.4 来源锁定

- **SHM矢量锁定**：仅订阅 `shm://vec/spectrum/*` 命名空间
- **事件来源锁定**：仅接收 `event://button/*` 和 `event://encoder/*` 事件

---

## 3. 函数设计（函数级，无类）

### 3.1 主函数签名

```typescript
function spectrum_ui_init(
    config: SpectrumUIConfig
): SpectrumUIHandle;

function spectrum_ui_tick(
    handle: SpectrumUIHandle,
    timestamp_ms: number
): void;

function spectrum_ui_on_button(
    handle: SpectrumUIHandle,
    btn_id: ButtonID,
    event_type: ButtonEvent
): void;

function spectrum_ui_on_encoder(
    handle: SpectrumUIHandle,
    delta: number,
    pressed: boolean
): void;

function spectrum_ui_on_fft_frame(
    handle: SpectrumUIHandle,
    magnitude_db: Float32Array,
    bin_count: number,
    start_freq: number,
    stop_freq: number
): void;

function spectrum_ui_on_status_update(
    handle: SpectrumUIHandle,
    status: SpectrumStatus
): void;

function spectrum_ui_on_color_map(
    handle: SpectrumUIHandle,
    color_lut: Uint8Array,
    lut_size: number
): void;

function spectrum_ui_destroy(
    handle: SpectrumUIHandle
): void;
```

### 3.2 参数规格表

#### SpectrumUIConfig

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `container_id` | `string` | 是 | 非空，有效DOM ID | UI容器元素ID |
| `width` | `number` | 是 | 800 <= width <= 1920 | 面板宽度(px) |
| `height` | `number` | 是 | 480 <= height <= 1080 | 面板高度(px) |
| `has_encoder` | `boolean` | 否 | 默认true | 是否支持旋转编码器 |
| `has_touch` | `boolean` | 否 | 默认true | 是否支持触摸输入 |
| `theme` | `'dark' \| 'light'` | 否 | 默认'dark' | 主题模式 |
| `locale` | `string` | 否 | 默认'zh-CN' | 语言区域 |

#### SpectrumStatus

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `center_freq` | `number` | 是 | >= 0 | 中心频率(Hz) |
| `span` | `number` | 是 | >= 0 | 频率跨度(Hz) |
| `ref_level` | `number` | 是 | -100 <= x <= 100 | 参考电平(dBm) |
| `atten` | `number` | 是 | 0 <= x <= 50 | 输入衰减(dB) |
| `rbw` | `number` | 是 | >= 1 | 分辨率带宽(Hz) |
| `vbw` | `number` | 是 | >= 1 | 视频带宽(Hz) |
| `trigger_mode` | `TriggerMode` | 是 | 枚举值 | 触发模式 |
| `trace_mode` | `TraceMode` | 是 | 枚举值 | 迹线模式 |
| `run_state` | `RunState` | 是 | 枚举值 | 运行状态 |
| `marker_freq` | `number` | 否 | >= 0 | Marker频率(Hz) |
| `marker_amp` | `number` | 否 | 任意 | Marker幅度(dBm) |
| `acq_time` | `number` | 是 | >= 0 | 采集时间(ms) |

#### ButtonID 枚举

| 值 | 说明 |
|----|------|
| `R1` | Run/Stop 按钮 |
| `R2` | Single 按钮 |
| `R3` | Peak Search 按钮 |
| `R4` | Marker 按钮 |
| `R5` | Center 按钮 |
| `R6` | Span 按钮 |
| `R7` | Settings 按钮 |
| `B1` | Freq+ 按钮 |
| `B2` | Freq- 按钮 |
| `B3` | Ampl+ 按钮 |
| `B4` | Ampl- 按钮 |
| `B5` | Step 按钮 |

#### ButtonEvent 枚举

| 值 | 说明 |
|----|------|
| `PRESS` | 按下 |
| `RELEASE` | 释放 |
| `LONG_PRESS` | 长按(>1.5s) |
| `REPEAT` | 连发 |

### 3.3 返回值规格

#### SpectrumUIHandle

```typescript
interface SpectrumUIHandle {
    status: 'ok' | 'error';
    container: HTMLElement;
    canvas: HTMLCanvasElement;
    ctx: CanvasRenderingContext2D;
    state: SpectrumUIState;
    config: SpectrumUIConfig;
    error?: string;
}
```

#### 函数返回值

```typescript
{
    status: 'ok' | 'error',
    displayed: string,
    timestamp: number,
    source: string,
    metadata: {
        frame_count: number,
        last_render_time: number,
        fps: number
    },
    error?: string
}
```

### 3.4 界面布局规范

```
┌────────────────────────────────────────┬──────────┐  ─┐
│ [顶部状态条: CF/Span/RL/Atten/RBW/Trig/时间] │          │   │ 10%
├────────────────────────────────────────┼──────────┤  ─┤
│ [颜色标尺]                             │ [R1] 运行 │   │
│ (瀑布图/密度图                         │ [R2] 单次 │   │
│  模式时显示)                           │ [R3] 峰值 │   │ 62%
│                                        │ [R4] 标记 │   │
│      频谱波形主显示区                   │ [R5] 中心 │   │
│      (Grid + Trace + Marker)           │ [R6] 跨度 │   │
│                                        │ [R7] 设置 │   │
├────────────────────────────────────────┴──────────┤  ─┤
│ [底部信息条: CF │ RBW │ VBW │ Span │ Acq Time]         │   │ 6%
├─────────────────────────────────────────────────────┤  ─┤
│ [B1]  [B2]  [B3]  [B4]  [B5]                         │   │ 22%
│  频+   频-   幅+   幅-   步进                         │   │
└─────────────────────────────────────────────────────┘  ─┘
```

**分区占比**：
- 顶部状态条：10%（全宽）
- 波形主显区：62%（左侧82%宽度，含可选颜色标尺10%）
- 右侧按键区：62%（右侧18%宽度，垂直排列7键）
- 底部信息条：6%（全宽，只读）
- 底部按键区：22%（全宽，水平排列5键）

### 3.5 按键功能映射

#### 右侧按键区（R1-R7）

| 编号 | 按钮名 | 短按功能 | 长按功能(>1.5s) | 状态灯 |
|------|--------|----------|-----------------|--------|
| R1 | Run/Stop | 连续扫描启停切换 | 进入单次序列模式 | 绿=运行，红=停止 |
| R2 | Single | 触发一次扫描并冻结 | 重置触发系统 | 黄闪=等待触发 |
| R3 | Peak Search | 搜寻最高峰放置Marker | 切换次峰/左峰/右峰/峰表 | 白=峰值锁定 |
| R4 | Marker | Marker开/关 | 调出Marker表(最多4个) | 橙=激活 |
| R5 | Center | 进入中心频率编辑态 | 将中心频设为Marker频点 | 黄=编辑态 |
| R6 | Span | 进入跨度编辑态 | 循环Full/Zero Span | 黄=编辑态 |
| R7 | Settings | 展开设置页 | 返回主界面 | 无 |

#### 底部按键区（B1-B5）

| 编号 | 按钮名 | 短按功能 | 长按功能 | 状态灯 |
|------|--------|----------|----------|--------|
| B1 | Freq+ | 中心频率按步进增加 | 连续快速递增(150ms/步) | 无 |
| B2 | Freq- | 中心频率按步进减少 | 连续快速递减 | 无 |
| B3 | Ampl+ | 参考电平按5dB增加 | 连续快速递增 | 无 |
| B4 | Ampl- | 参考电平按5dB减少 | 连续快速递减 | 无 |
| B5 | Step | 循环切换频率步进值 | 进入编码器模式 | 绿=编码器模式 |

### 3.6 波形显示规范

#### 网格系统
- 水平轴：10格，标注起点/中心/终点频率
- 垂直轴：10格，标注参考电平及每格dB值
- 网格线颜色：`#333333`，线宽1px
- 背景色：`#0A0A0A`

#### 迹线规范
- 主迹线(T1)：`#00FF41`，线宽1.5px
- 最大保持迹线(T2)：`#FF6B6B`，虚线，透明度60%
- 刷新率：>= 30fps

#### Marker规范
- 十字线：`#FFFFFF`虚线
- 菱形标记：`#FFFF00`
- 读数精度：频率6位有效数字，幅度0.01dBm

### 3.7 颜色标尺规范

- 宽度：固定48px
- 颜色映射：`#FF0000`→`#FFFF00`→`#00FF00`→`#00FFFF`→`#0000FF`
- 显示条件：Trace模式为Density/Spectrogram时展开

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | container_id 非空且有效 | `ERR_INVALID_CONTAINER` | `error` |
| PRE-002 | width 在 800-1920 范围内 | `ERR_WIDTH_OUT_OF_RANGE` | `error` |
| PRE-003 | height 在 480-1080 范围内 | `ERR_HEIGHT_OUT_OF_RANGE` | `error` |
| PRE-004 | magnitude_db 长度与 bin_count 一致 | `ERR_BIN_COUNT_MISMATCH` | `error` |
| PRE-005 | start_freq < stop_freq | `ERR_FREQ_RANGE_INVALID` | `error` |
| PRE-006 | handle 已初始化 | `ERR_HANDLE_NOT_INIT` | `error` |
| PRE-007 | btn_id 为有效枚举值 | `ERR_INVALID_BUTTON_ID` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | canvas 已绑定到 container | DOM结构正确 |
| POST-003 | 事件监听器已注册 | 输入响应就绪 |
| POST-004 | 渲染帧率 >= 30fps | 性能约束 |
| POST-005 | 内存占用 <= 50MB | 资源约束 |

### 4.3 不变量

- **零全局状态**：所有状态封装在handle内
- **零堆分配**：预分配Canvas缓冲区，避免运行时分配
- **零系统调用**：不进行文件IO、网络IO
- **确定性渲染**：相同状态产生相同画面
- **事件驱动**：仅响应外部事件触发更新

### 4.4 状态机

```
[UNINIT] ──init()──► [READY] ──start()──► [RUNNING]
                          │                    │
                          │                    ├──stop()──► [STOPPED]
                          │                    │                │
                          │                    │                └──start()──► [RUNNING]
                          │                    │
                          └──destroy()──► [DESTROYED]
```

### 4.5 错误契约表

| 错误码 | 触发条件 | 返回status | 返回displayed | 返回error |
|--------|----------|------------|---------------|-----------|
| `ERR_INVALID_CONTAINER` | 容器ID无效 | `error` | `""` | "Invalid container ID" |
| `ERR_WIDTH_OUT_OF_RANGE` | 宽度超限 | `error` | `""` | "Width must be 800-1920" |
| `ERR_HEIGHT_OUT_OF_RANGE` | 高度超限 | `error` | `""` | "Height must be 480-1080" |
| `ERR_BIN_COUNT_MISMATCH` | FFT数据长度不匹配 | `error` | `""` | "Bin count mismatch" |
| `ERR_FREQ_RANGE_INVALID` | 频率范围无效 | `error` | `""` | "Start freq must < stop freq" |
| `ERR_HANDLE_NOT_INIT` | handle未初始化 | `error` | `""` | "Handle not initialized" |
| `ERR_INVALID_BUTTON_ID` | 按钮ID无效 | `error` | `""` | "Invalid button ID" |
| `ERR_CANVAS_CONTEXT` | Canvas上下文获取失败 | `error` | `""` | "Failed to get canvas context" |

---

## 5. 测试场景（验收标准）

### 5.1 正常路径测试

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 初始化 | 标准配置 | width=1024, height=600 | handle.status='ok', canvas已创建 |
| TC-002 | FFT渲染 | 1024点FFT数据 | 默认配置 | 波形正确渲染，帧率>=30fps |
| TC-003 | 状态更新 | 完整状态对象 | 默认配置 | 顶部状态条正确显示所有字段 |
| TC-004 | 按钮响应 | R1短按 | 运行状态 | 切换为停止状态，状态灯变红 |
| TC-005 | 编码器输入 | delta=+1 | Center编辑态 | 中心频率增加一个步进值 |
| TC-006 | 颜色映射 | 256色LUT | Density模式 | 颜色标尺正确显示 |

### 5.2 边界路径测试

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-010 | 最小分辨率 | width=800, height=480 | 最小配置 | UI正确缩放，按钮可点击 |
| TC-011 | 最大分辨率 | width=1920, height=1080 | 最大配置 | UI正确扩展，无溢出 |
| TC-012 | FFT边界 | bin_count=1 | 最小FFT | 单点显示，无崩溃 |
| TC-013 | FFT边界 | bin_count=65536 | 最大FFT | 正确渲染，内存<=50MB |
| TC-014 | 频率边界 | start_freq=0, stop_freq=1e12 | 极限频率 | 正确显示单位切换 |

### 5.3 异常路径测试

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-020 | 无效容器 | container_id="nonexistent" | 默认配置 | status='error', error='ERR_INVALID_CONTAINER' |
| TC-021 | 宽度超限 | width=500 | 默认配置 | status='error', error='ERR_WIDTH_OUT_OF_RANGE' |
| TC-022 | 高度超限 | height=200 | 默认配置 | status='error', error='ERR_HEIGHT_OUT_OF_RANGE' |
| TC-023 | 空FFT数据 | magnitude_db=[] | 默认配置 | 波形区显示空状态，无崩溃 |
| TC-024 | 无效按钮 | btn_id="X99" | 默认配置 | status='error', error='ERR_INVALID_BUTTON_ID' |

### 5.4 性能路径测试

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-030 | 高帧率渲染 | 60fps持续10秒 | 默认配置 | 平均帧率>=30fps，无内存泄漏 |
| TC-031 | 大数据量 | 65536点FFT持续100帧 | 默认配置 | 渲染时间<=33ms/帧 |
| TC-032 | 快速输入 | 100次按钮点击/秒 | 默认配置 | 所有事件正确处理，无丢失 |
| TC-033 | 长时间运行 | 持续运行1小时 | 默认配置 | 内存稳定，无泄漏 |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `canvas` | 2D渲染 | HTML5 Canvas API |
| `requestAnimationFrame` | 动画帧调度 | 浏览器原生 |
| `Float32Array` | FFT数据存储 | ES6 TypedArray |
| `Uint8Array` | 颜色映射存储 | ES6 TypedArray |

### 6.2 外部依赖声明

| 依赖 | 用途 | 版本要求 | 说明 |
|------|------|----------|------|
| Vue 3 / React 18 | UI框架 | ^3.0 / ^18.0 | 二选一 |
| TypeScript | 类型系统 | ^5.0 | 可选 |

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n) 每帧渲染 | n为FFT点数 |
| 内存 | <= 50MB | 包含Canvas缓冲区 |
| GPU | 无要求 | 纯CPU渲染 |
| 执行时限 | <= 33ms/帧 | 30fps最低要求 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 不进行任何文件读写 |
| 网络零IO | ✅ | 不进行任何网络通讯 |
| 零全局副作用 | ✅ | 所有状态封装在handle内 |
| 输入消毒 | ✅ | 对所有输入进行边界校验 |
| 确定性输出 | ✅ | 相同状态产生相同画面 |
| SHM权限合规 | ✅ | 仅读取授权的SHM矢量 |
| 栈帧安全 | ✅ | 不存在栈溢出风险 |
| 零堆分配 | ✅ | 预分配缓冲区，无运行时分配 |

### 7.2 DOM操作特许声明

本函数需要操作DOM元素（Canvas），声明以下特许：
- **Canvas创建**：在container内创建Canvas元素
- **事件监听**：注册触摸/鼠标事件监听器
- **样式修改**：修改Canvas样式属性

### 7.3 事件边界

- **输入事件**：仅处理button/encoder事件，不处理其他DOM事件
- **输出事件**：仅发送command事件到事件总线

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 契约审查指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证 |
| 代码生成 | 将蓝图意图转化为TypeScript实现（Vue 3 / React 18） |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例 |
| 安全扫描 | 验证三大零原则、DOM操作特许、输入消毒 |
| 授权文件输出 | 按JSON结构生成授权文件 |

### 8.2 技术栈选择

推荐使用 **Vue 3 + TypeScript** 实现：
- Composition API 适合函数式设计
- 响应式系统简化状态管理
- 更好的TypeScript支持

### 8.3 文件结构建议

```
spectrum-ui/
├── src/
│   ├── components/
│   │   ├── TopStatusBar.vue      # 顶部状态条
│   │   ├── WaveformCanvas.vue    # 波形显示区
│   │   ├── ColorScale.vue        # 颜色标尺
│   │   ├── RightPanel.vue        # 右侧按键区
│   │   ├── BottomInfoBar.vue     # 底部信息条
│   │   └── BottomPanel.vue       # 底部按键区
│   ├── composables/
│   │   ├── useSpectrumUI.ts      # 主Hook
│   │   ├── useButtonInput.ts     # 按键输入
│   │   ├── useEncoderInput.ts    # 编码器输入
│   │   └── useFFTData.ts         # FFT数据处理
│   ├── types/
│   │   └── spectrum.ts           # 类型定义
│   └── utils/
│       ├── format.ts             # 格式化工具
│       └── colorMap.ts           # 颜色映射
├── tests/
│   └── spectrum-ui.test.ts       # 验收测试
└── package.json
```

### 8.4 部署要求

- 构建产物：静态HTML + JS + CSS
- 部署方式：Nginx静态文件服务
- 浏览器兼容：Chrome 90+, Firefox 88+, Safari 14+

---

## 审核状态

**状态**: 待审核  
**版本**: 1.3.0  
**设计者**: AICoder（UI载体层）  
**审核权归属**: 人类架构师（消费者域主权）

---

*"蓝图不载代码，只载意图。格式不存偏好，只存契约。八节齐全则意图明，七维通过则契约立。"*
