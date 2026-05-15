# 蓝图文档：BP-0054 -- 专业TR测试工程师UI（多仪器集成版）

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0054-TR-TEST-UI |
| 蓝图名称 | 专业TR测试工程师UI消费者（多仪器集成版） |
| 功能域 | consumer |
| 提交时间 | 2026-05-14 |
| 版本 | 1.1.0 |
| 目标语言 | TypeScript (Vue 3) |
| 复杂度等级 | Level-3 |
| 设计范式 | 纯函数级，禁止OOP封装 |
| 产品来源 | rtlsdr_worker (BP-RTLSDR-WORKER), network_rx (BP-0016), simulation (BP-0018) |
| 消费者ID | CONSUMER-TR-001 |

---

## 2. 功能概述

### 2.1 功能描述

专业TR测试工程师UI消费者，作为商场内核的"相"呈现载体，负责将SHM矢量推送的频谱/波形/功率数据、仪器状态渲染为多仪器集成可视化界面。支持Chat驱动交互、视图联动、自动化测试序列，实现一站式TR测试操作。

### 2.2 数据来源

| 来源类型 | 来源标识 | 说明 |
|----------|----------|------|
| SHM矢量 | `shm://rtlsdr/spectrum_fft` | FFT频谱数据 |
| SHM矢量 | `shm://rtlsdr/waveform_iq` | IQ时域波形数据 |
| SHM矢量 | `shm://rtlsdr/power_meter` | 功率计数据 |
| SHM矢量 | `shm://rtlsdr/device_status` | 设备状态 |
| 用户输入 | Chat指令输入 | 对话式仪器控制 |

### 2.3 数据去向

| 去向类型 | 去向标识 | 说明 |
|----------|----------|------|
| SHM矢量 | `shm://ui/tr_config` | 仪器配置参数 |
| SHM矢量 | `shm://ui/tr_chat_command` | Chat指令 |
| SHM矢量 | `shm://ui/tr_task_sequence` | 自动化任务序列 |
| 渲染画布 | Canvas绑定 | 频谱/波形/仪表盘 |

### 2.4 来源锁定

- **SHM矢量锁定**：仅订阅 `shm://rtlsdr/*` 和 `shm://ui/tr/*` 命名空间
- **事件来源锁定**：仅接收Chat输入和Canvas交互事件

---

## 3. 函数设计

### 3.1 主函数

```typescript
function tr_test_ui_init(config: TRTestUIConfig): TRTestUIHandle
function tr_test_ui_tick(handle: TRTestUIHandle, timestamp_ms: number): void
function tr_test_ui_render(handle: TRTestUIHandle): TRTestUIVNode
```

### 3.2 频谱渲染

```typescript
function render_spectrum_page(ctx: TRTestCtx): VNode
function render_spectrum_canvas(ctx: TRTestCtx): VNode
function render_spectrum_markers(ctx: TRTestCtx): VNode
function render_spectrum_axes(ctx: TRTestCtx): VNode
```

### 3.3 波形渲染

```typescript
function render_waveform_page(ctx: TRTestCtx): VNode
function render_waveform_canvas(ctx: TRTestCtx): VNode
function render_waveform_cursors(ctx: TRTestCtx): VNode
```

### 3.4 功率计渲染

```typescript
function render_power_page(ctx: TRTestCtx): VNode
function render_power_gauge(ctx: TRTestCtx): VNode
function render_power_stats(ctx: TRTestCtx): VNode
```

### 3.5 Chat面板

```typescript
function render_chat_panel(ctx: TRTestCtx): VNode
function render_chat_input(ctx: TRTestCtx): VNode
function render_dynamic_controls(ctx: TRTestCtx): VNode
function render_chat_history(ctx: TRTestCtx): VNode
function parse_chat_command(input: string): ChatCommand
function generate_dynamic_controls(cmd: ChatCommand): VNode[]
```

### 3.6 数据表格

```typescript
function render_data_table(ctx: TRTestCtx): VNode
function filter_table_data(ctx: TRTestCtx, filter: TableFilter): TableRow[]
function export_table_csv(rows: TableRow[]): string
```

### 3.7 任务序列

```typescript
function render_task_runner(ctx: TRTestCtx): VNode
function load_task_sequence(json_str: string): TaskSequence
function validate_task_sequence(seq: TaskSequence): ValidationResult
function execute_task_step(ctx: TRTestCtx, step: TaskStep): StepResult
```

### 3.8 视图联动

```typescript
function sync_spectrum_to_signal_source(ctx: TRTestCtx, freq_hz: number): void
function sync_pulse_to_waveform(ctx: TRTestCtx, pulse_cfg: PulseConfig): void
function sync_power_alarm(ctx: TRTestCtx, threshold_dbm: number): void
```

### 3.9 布局引擎

```typescript
function apply_layout(ctx: TRTestCtx): void
function set_layout(layout: 'single' | 'dual' | 'quad' | 'hex'): void
function toggle_instrument(ctx: TRTestCtx, instrument: string): void
function resize_all_canvases(ctx: TRTestCtx): void
```

### 3.10 场景预设

```typescript
function apply_scene(ctx: TRTestCtx, scene_id: string): void
function get_scene_config(scene_id: string): SceneConfig | null
```

### 3.11 连接状态管理

```typescript
function toggle_connection(ctx: TRTestCtx): void
function get_connection_state(ctx: TRTestCtx): ConnectionState
```

---

## 4. 设计文档索引

| 编号 | 文件 | 说明 |
|------|------|------|
| 01 | `UI-Design-TR-Test-V1.0/01-设计说明文档.md` | 设计哲学/SHM引用/函数设计/安全合规 |
| 02 | `UI-Design-TR-Test-V1.0/02-信息架构图.md` | 六仪器标签页+Chat面板+数据表格架构 |
| 03 | `UI-Design-TR-Test-V1.0/03-高保真视觉稿.md` | 深色主题仪器渲染规范 |
| 04 | `UI-Design-TR-Test-V1.0/04-组件规格表.md` | Canvas/仪表盘/Chat组件规格 |
| 05 | `UI-Design-TR-Test-V1.0/05-交互状态机.md` | 视图联动+Chat指令+任务执行状态机 |
| 06 | `UI-Design-TR-Test-V1.0/06-CSS变量表.md` | 仪器配色+状态色+深色主题 |
| 07 | `UI-Design-TR-Test-V1.0/07-交付检查表.md` | 验收标准（59项） |
| 08 | `UI-Design-TR-Test-V1.0/08-SHM绑定规范.md` | 各仪器Canvas与SHM矢量数据绑定 |

---

## 5. 实现状态

| 模块 | 状态 | 实现路径 |
|------|------|---------|
| 六仪器集成 | ✅ 已实现 | ui-lab/public/tr-test-ui/ |
| 多仪表同屏 | ✅ 已实现 | CSS Grid + JS布局引擎 |
| Chat指令引擎 | ✅ 已实现 | 9种指令解析+动态控件 |
| 频谱/波形实时渲染 | ✅ 已实现 | Canvas 2D + requestAnimationFrame |
| 功率仪表盘 | ✅ 已实现 | 弧形仪表+指针+数字读数 |
| 数据表格 | ✅ 已实现 | 实时更新+排序+CSV导出 |
| 视图联动 | ✅ 已实现 | 频谱→信号源/功率→频谱 |
| 场景预设 | ✅ 已实现 | 4个预设场景 |
| 安全合规 | 🔶 部分实现 | 参数校验+确认弹窗，审计待集成 |
| SHM矢量绑定 | ❌ 待集成 | 需商场矢量交易架构就绪 |
| 自动化任务序列 | ❌ 待实现 | 任务导入/执行/进度 |

---

## 6. 技术实现

- 前端：纯HTML + CSS + JavaScript（无框架依赖）
- 渲染：Canvas 2D API，requestAnimationFrame驱动
- 布局：CSS Grid，4种布局模式
- 数据：模拟数据，预留SHM矢量接口
- 字体：Orbitron + JetBrains Mono
- 预览：http://127.0.0.1:8765
