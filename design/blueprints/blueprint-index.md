# 创世纪项目 · 蓝图索引

> **版本**: v5.0
> **日期**: 2026-05-10
> **说明**: 蓝图按功能域分类，统一编号前缀

---

## 已激活蓝图

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0001 | producer | blueprints/producer/BP-0001-keyboard.md | keyboard_echo | ACTIVE |
| BP-0002 | consumer | blueprints/consumer/BP-0002-console.md | console_consumer_render | ACTIVE |
| BP-0003 | shm | blueprints/shm/BP-0003-shm-vector.md | shm_universe_init | ACTIVE |
| BP-0004 | mall | blueprints/mall/BP-0004-exception.md | mall_exception_handler | ACTIVE |
| BP-0009 | mall | blueprints/mall/BP-0009-registry.md | registry_register | DONE |
| BP-0010 | mall | blueprints/mall/BP-0010-auth-loader.md | auth_validate | DONE |
| BP-0011 | mall | blueprints/mall/BP-0011-heartbeat.md | tick_emit | DONE |
| BP-0012 | mall | blueprints/mall/BP-0012-message-router.md | route_publish | DONE |
| BP-0013 | mall | blueprints/mall/BP-0013-log-producer.md | log_emit | DONE |
| BP-0014 | mall | blueprints/mall/BP-0014-config.md | config_apply | DONE |
| BP-0015 | mall | blueprints/mall/BP-0015-persistence.md | persist_subscribe | DONE |

---

## 待处理蓝图（SC场景引用）

> **来源**: 从根目录整合，原编号与已激活蓝图冲突，重新编号为BP-0016~BP-0021
> **关联场景**: SC-0002~SC-0005

| 编号 | 原编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|--------|----------|--------|------|
| BP-0016 | BP-0003 | producer | blueprints/producer/BP-0016-network-rx.md | network_rx_init / network_rx_loop | PENDING |
| BP-0017 | BP-0004 | mall | blueprints/mall/BP-0017-analysis-worker.md | analysis_worker_init / analysis_worker_loop | PENDING |
| BP-0018 | BP-0005 | producer | blueprints/producer/BP-0018-simulation.md | simulation_init / simulation_loop | PENDING |
| BP-0019 | BP-0006 | mall | blueprints/mall/BP-0019-fusion-worker.md | fusion_worker_init / fusion_worker_loop | PENDING |
| BP-0020 | BP-0007 | mall | blueprints/mall/BP-0020-control-worker.md | control_worker_init / control_worker_loop | PENDING |
| BP-0021 | BP-0008 | producer | blueprints/producer/BP-0021-network-tx.md | network_tx_init / network_tx_loop | PENDING |

---

## 待处理蓝图（基础设施）

> **来源**: 从根目录整合，原编号与已激活蓝图冲突，重新编号为BP-0022~BP-0027

| 编号 | 原编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|--------|----------|--------|------|
| BP-0022 | BP-0003 | mall | blueprints/mall/BP-0022-stdio-pipe.md | stdio_pipe | PENDING |
| BP-0023 | BP-0004 | shm | blueprints/shm/BP-0023-shm-manager.md | shm_manager | PENDING |
| BP-0024 | BP-0005 | mall | blueprints/mall/BP-0024-worker-spawn.md | worker_spawn | PENDING |
| BP-0025 | BP-0006 | mall | blueprints/mall/BP-0025-intent-router.md | intent_router | PENDING |
| BP-0026 | BP-0007 | mall | blueprints/mall/BP-0026-error-guard.md | error_guard | PENDING |
| BP-0027 | BP-0008 | mall | blueprints/mall/BP-0027-aicoder-gateway.md | aicoder_gateway | PENDING |

---

## 新增蓝图（2026-05-11）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0028 | consumer | blueprints/consumer/BP-0028-task-parameterization.md | task_param_create / execution_controller_init | DRAFT |
| BP-0029 | mall | blueprints/mall/BP-0029-user-homepage.md | user_homepage_render | ACTIVE |
| BP-0030 | mall | blueprints/mall/BP-0030-task-publisher.md | task_publisher_dispatch | ACTIVE |
| BP-0048 | mall | blueprints/mall/BP-0048-task-conception.md | task_conception_init / task_conception_match_producers / task_conception_estimate_cost / task_conception_generate_contract / task_conception_ratify_contract / task_conception_sandbox_preview | CODE_DONE |
| BP-0049 | mall | blueprints/mall/BP-0049-task-adjustment.md | task_adjustment_init / task_adjustment_review / task_adjustment_apply_corrections | CODE_DONE |
| BP-0050 | mall | blueprints/mall/BP-0050-alarm-system.md | alarm_system_init / alarm_system_evaluate / alarm_system_notify | CODE_DONE |
| BP-0051 | mall | blueprints/mall/BP-0051-decision-engine.md | decision_engine_init / decision_engine_evaluate_rules / decision_engine_prioritize | CODE_DONE |
| BP-0052 | mall | blueprints/mall/BP-0052-execution-monitor.md | execution_monitor_init / execution_monitor_sample / execution_monitor_report | CODE_DONE |
| BP-0053 | consumer | blueprints/consumer/BP-0053-user-task-config.md | user_task_create / subtask_chain_validate / static_config_define / config_sequence_build / resource_conflict_detect | DRAFT |
| BP-0031 | mall | blueprints/mall/BP-0031-task-editor.md | task_editor_compile / task_editor_render | DRAFT |

---

## 新增蓝图（2026-05-11 系统监控）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0031 | producer | blueprints/producer/BP-0031-sysinfo.md | sysinfo_init / sysinfo_loop | DRAFT |
| BP-0032 | consumer | blueprints/consumer/BP-0032-sysdisplay.md | sysdisplay_init / sysdisplay_loop | DRAFT |

---

## 新增蓝图（2026-05-11 基因编译器）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0033 | mall | blueprints/mall/BP-0033-genesis-compiler.md | genesis_compiler_init / genesis_compiler_run + 14模块函数 | DRAFT |

---

## 新增蓝图（2026-05-12 用户画像）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0035 | consumer | blueprints/consumer/BP-0035-user-profile.md | user_profile_create + user_profile_api_init/run + aicoder_phase_compiler_main + phase_renderer_main + 11函数 | ACTIVE |

---

## 新增蓝图（2026-05-12 核心商场蓝图）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0036 | mall | blueprints/mall/BP-0036-mall-monitor.md | mall_monitor | PENDING |
| BP-0037 | mall | blueprints/mall/BP-0037-config-loader.md | config_loader | PENDING |
| BP-0038 | mall | blueprints/mall/BP-0038-mall-scheduler.md | mall_scheduler_init / mall_scheduler_tick / mall_scheduler_dispatch / mall_scheduler_rebalance | PENDING |
| BP-0039 | mall | blueprints/mall/BP-0039-shm-lifecycle.md | shm_lifecycle_init / shm_product_seal / shm_product_deliver / shm_product_archive / shm_product_destroy | PENDING |
| BP-0040 | mall | blueprints/mall/BP-0040-trade-metrics.md | metrics_init / metrics_collect_global / metrics_collect_producer / metrics_collect_tx / metrics_publish_advisory | PENDING |
| BP-0041 | mall | blueprints/mall/BP-0041-path-router.md | path_router_init / path_router_assign / path_router_upgrade / path_router_conflict_resolve | PENDING |
| BP-0047 | mall | blueprints/mall/BP-0047-task-closure.md | task_closure_init / task_closure_acceptance_panel / task_closure_archive_package / task_closure_evolution_suggest | CODE_DONE |

---

## 新增蓝图（2026-05-12 矢量交易架构）

> **版本**: v1.0
> **架构**: 三进程 + 矢量Pub/Sub + 监督Worker屏障同步

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0042 | mall | blueprints/mall/BP-mall-vector-trade.md | run_vector_loop / init_vector_mode / _process_receipt / _process_feedback | **ACTIVE** |
| BP-0043 | shm | blueprints/shm/BP-0043-vector-manager.md | VectorManager.allocate / publish / subscribe / read_new / init_system_vectors | **ACTIVE** |
| BP-0044 | mall | blueprints/mall/BP-0044-batch-sync-supervisor.md | BatchSyncSupervisor.run / _check_receipts / _publish_feedback / SupervisorManager.create_supervisor | **ACTIVE** |
| BP-0045 | mall | blueprints/mall/BP-0045-task-management-worker.md | task_mgmt_worker_init / task_mgmt_worker_loop + trader(7函数) + supervisor(6函数) | **ACTIVE** |

### 矢量交易核心文件

| 文件 | 说明 | 状态 |
|------|------|------|
| `src/mall_core.py` | 商场本体子进程，支持矢量交易模式 | **ACTIVE** |
| `process_manager.py` | 主进程管理器，三进程编排 | **ACTIVE** |
| `fastapi_worker.py` | FastAPI通讯层，Queue转发 | **ACTIVE** |
| `src/vectors/vector_manager.py` | 矢量管理器，环形缓冲区实现 | **ACTIVE** |
| `src/vectors/types.py` | 帧结构定义 | **ACTIVE** |
| `src/supervisor/batch_sync.py` | 批量同步监督Worker | **ACTIVE** |

### 矢量通道

| 通道ID | 方向 | 说明 |
|--------|------|------|
| VEC-RECEIPT | 消费者→商场 | 回执确认（环形缓冲区） |
| VEC-CONTROL | 商场→所有 | 节奏调节/配额通知 |
| VEC-FEEDBACK | 监督Worker→商场 | 屏障同步反馈 |
| VEC-PUB-{producer_id} | 生产者→消费者 | 产品发布（按需分配） |

---

## 新增蓝图（2026-05-12 决策工具内核）

> **版本**: v1.0
> **规范依据**: advisor-kernel-governance.md（决策工具部署规范）
> **铁律**: 分析在商场域，呈现在消费者域，UI只负责"相"

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0045 | mall | blueprints/mall/BP-0045-advisor-kernel.md | AdvisorKernel / mine_patterns / score_producer / match_producer / generate_task_report | **ACTIVE** |

### Advisor Kernel 核心文件

| 文件 | 说明 | 状态 |
|------|------|------|
| `src/advisor/__init__.py` | 模块导出 | **ACTIVE** |
| `src/advisor/kernel.py` | AdvisorKernel 主类（请求调度/缓存/档案管理） | **ACTIVE** |
| `src/advisor/mine.py` | 历史模式挖掘引擎 | **ACTIVE** |
| `src/advisor/score.py` | 生产者五维动态评分引擎 | **ACTIVE** |
| `src/advisor/match.py` | 消费者需求匹配引擎 | **ACTIVE** |
| `src/advisor/report.py` | 标准化报告生成引擎 | **ACTIVE** |
| `design/specs/constitution/advisor-kernel-governance.md` | 决策工具部署规范（宪法级） | **ACTIVE** |

### Advisor API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/advisor/mine` | GET | 历史模式挖掘 |
| `/api/advisor/score` | GET | 生产者评分（可指定producer_id） |
| `/api/advisor/match` | GET | 消费者需求匹配 |
| `/api/advisor/report` | GET | 报告生成（任务报告/商场报告） |
| `/api/advisor/stats` | GET | Advisor 统计信息 |

---

## 商场任务工具全景蓝图（2026-05-12）

> **版本**: v1.0
> **宪法依据**: 三权分立体系、四域架构、商场安全宪法 v1.0–v1.3
> **铁律**: 工具只提供"能力"，不提供"决策"；决策权永远属于人类消费者
> **规范文档**: `design/specs/workflow/mall_tools_outline_v1.md` 等7份

### 工具全景（19个工具，5个阶段）

| 阶段 | 工具 | 状态 | 完成度 |
|------|------|------|--------|
| **一、发起** | 1.任务蓝图编辑器 | 🔶 部分实现 | ~40% |
| | 2.生产者匹配探针 | 🔶 部分实现 | ~60% |
| | 3.代价评估计算器 | ❌ 未实现 | 0% |
| | 4.契约生成器 | ❌ 未实现 | 0% |
| **二、批准** | 5.契约可视化面板 | ❌ 未实现 | 0% |
| | 6.双向确认签名器 | ❌ 未实现 | 0% |
| | 7.沙箱预演器 | ❌ 未实现 | ~5% |
| **三、执行** | 8.实时状态监控台 | 🔶 部分实现 | ~50% |
| | 9.参数动态调节器 | 🔶 部分实现 | ~40% |
| | 10.IO流检视器 | ❌ 未实现 | ~10% |
| | 11.紧急制动阀 | 🔶 部分实现 | ~50% |
| **四、监督** | 12.交易审计日志 | 🔶 部分实现 | ~35% |
| | 13.异常告警器 | ❌ 未实现 | 0% |
| | 14.质量抽检器 | ❌ 未实现 | 0% |
| | 15.进度推演仪 | 🔶 部分实现 | ~30% |
| **五、完结** | 16.成果验收面板 | ❌ 未实现 | ~15% |
| | 17.数据归档打包器 | ❌ 未实现 | 0% |
| | 18.生产者评价器 | 🔶 部分实现 | ~40% |
| | 19.蓝图进化建议器 | ❌ 未实现 | ~10% |

**总体完成度: ~20%**（9个部分实现，10个未实现）

### 工具设计规范文档

| 文件 | 说明 |
|------|------|
| `design/specs/workflow/mall_tools_outline_v1.md` | 总纲（工具哲学/五阶段/四域映射） |
| `design/specs/workflow/mall_tools_conception_v1.md` | 发起与批准工具设计（工具1~7） |
| `design/specs/workflow/mall_tools_execution_v1.md` | 执行与监督工具设计（工具8~15） |
| `design/specs/workflow/mall_tools_closure_v1.md` | 完结与归档工具设计（工具16~19） |
| `design/specs/workflow/mall_tools_deep_v1.md` | 关键工具深度设计（调节器/告警器/决策引擎） |
| `design/specs/workflow/mall_tools_scenario_v1.md` | 小场景实例：功率测量任务工具串联 |
| `design/specs/workflow/mall_tools_gap_v1.md` | 工具缺口清单与演进路线 |

### 实施路线（按缺口优先级）

| 版本 | 目标 | 工具 |
|------|------|------|
| **v1.1 近期** | 降低任务失败率 | 生产者画像面板 + 生产者自检工具 |
| **v1.2 中期** | 多任务管理 | 任务群调度器 + 蓝图推荐引擎 |
| **v1.3 远期** | 零门槛+质量保障 | 跨商场迁移 + 引导层 + 任务回放器 |

---

## 新增蓝图（2026-05-14 TR测试工程师UI）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0054 | consumer | blueprints/consumer/BP-0054-tr-test-ui.md | tr_test_ui_init / tr_test_ui_tick / tr_test_ui_render + 30+函数 | DRAFT |

### TR测试工程师UI设计文档

| 文件 | 说明 |
|------|------|
| `UI-Design-TR-Test-V1.0/01-设计说明文档.md` | 设计哲学/SHM引用/函数设计/安全合规 |
| `UI-Design-TR-Test-V1.0/02-信息架构图.md` | 六仪器标签页+Chat面板+数据表格架构 |
| `UI-Design-TR-Test-V1.0/03-高保真视觉稿.md` | 深色主题仪器渲染规范 |
| `UI-Design-TR-Test-V1.0/04-组件规格表.md` | Canvas/仪表盘/Chat组件规格 |
| `UI-Design-TR-Test-V1.0/05-交互状态机.md` | 视图联动+Chat指令+任务执行状态机 |
| `UI-Design-TR-Test-V1.0/06-CSS变量表.md` | 仪器配色+状态色+深色主题 |
| `UI-Design-TR-Test-V1.0/07-交付检查表.md` | 验收标准（59项） |
| `UI-Design-TR-Test-V1.0/08-SHM绑定规范.md` | 各仪器Canvas与SHM矢量数据绑定 |

---

## 新增蓝图（2026-05-14 管理员画像系统UI）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0055 | consumer | blueprints/consumer/BP-0055-admin-profile-ui.md | admin_profile_ui_init / admin_profile_ui_tick | CODE_DONE |

### 管理员画像系统UI设计文档

| 文件 | 说明 |
|------|------|
| `BP-0055-admin-profile-ui.md` | 管理员画像系统UI消费者蓝图 v1.1.0（嵌入式仪器风格） |

### 设计特点

- **三界隔离适配**: UI作为消费者域，只订阅SHM矢量，不直接操作商场
- **函数范式**: UI组件纯函数设计，状态外置Pinia Store
- **SHM矢量空间**: 通过VEC-DISPLAY订阅数据，VEC-CONTROL发布命令
- **基因图谱风格**: 深色主题(#0A0E17) + 金色(#D4A843) + 翡翠绿(#00C9A7)
- **嵌入式仪器风格**: instrument-panel/header/btn/LED/digital-readout 全局样式体系
- **Nginx钢铁脊椎**: 静态资源由Nginx直接服务

### v1.1.0 变更

- 嵌入式仪器风格重构（替代通用Web UI）
- 修复Tailwind `w-50`/`pl-50`间距值bug → `w-[200px]`/`pl-[200px]`
- 修正LED红色（`#E8B84B`金色 → `#E53E3E`红色）
- 更新类型契约（Channel/MatchRule/FailureReason/RiskRule/ChatMessage）
- 新增实现状态追踪（24模块已完成，3模块待实现）
- 新增已知问题与修复记录

---

## 新增蓝图（2026-05-14 UI画廊）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0056 | consumer | blueprints/consumer/BP-0056-ui-gallery.md | ui_gallery_init / ui_gallery_tick | CODE_DONE |

---

## 新增蓝图（2026-05-13 SHM双通道架构）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0057 | shm | blueprints/shm/BP-0057-dual-channel-shm.md | MmapDataManager.allocate / release / get_path / get_info / cleanup_expired | ACTIVE |
| BP-0058 | mall | blueprints/mall/BP-0058-worker-flow-scenario.md | resident_worker_poller / task_status_checker / spawn_task / kill_task | ACTIVE |
| BP-0059 | shm | blueprints/shm/BP-0059-product-valid-flag.md | product_flag_check_writable / product_write_with_flag / product_flag_check_has_product / product_consume_and_reset | ACTIVE |

### 双通道架构核心文件

| 文件 | 说明 | 状态 |
|------|------|------|
| `src/shm/mmap_manager.py` | mmap大数据通道管理器 | ACTIVE |
| `src/shm/vector_manager.py` | ShmVectorManager（集成mmap模式） | ACTIVE |

### 双通道架构

| 通道 | 介质 | 用途 | 大小限制 |
|------|------|------|---------|
| 控制通道 | SHM | 交易信令/元数据/矢量ID | 16MB |
| 数据通道 | mmap | 大数据零拷贝传输 | 按消费者配额 |

### 消费者mmap配额

| 类型 | SHM配额 | mmap配额 |
|------|---------|---------|
| CT | 1MB | 10MB |
| CF | 10MB | 100MB |
| CN | 100MB | 500MB |
| CA | 100MB | 500MB |
| CC | 1GB | 2GB |
| CCn | 10MB | 50MB |
| SP | 1MB | 1GB |

### UI画廊设计文档

| 文件 | 说明 |
|------|------|
| `BP-0056-ui-gallery.md` | 创世纪UI画廊消费者蓝图 v1.2.0 |

### 设计特点

- **统一入口**: 画廊作为所有UI资产的统一展厅入口
- **8展品4展厅**: 静态页面(4) + 嵌入式UI(2) + 专业仪器(1) + 可视化(1)
- **iframe预览**: 画廊内直接预览目标UI，不离开画廊上下文
- **仪器风格**: 复用instrument-panel/header/btn/LED全局样式体系
- **Nginx代理**: 所有UI通过Nginx反向代理统一访问
- **静态配置**: gallery.json定义展品元数据，支持动态发现
- **Vite代理**: serveStaticPlugin中间件开发模式跨项目访问
- **SPA适配**: Vue SPA base path + Router BASE_URL + SPA fallback

---

## 新增蓝图（2026-05-15 基因图谱页面）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0061 | consumer | blueprints/consumer/BP-0061-genome-viewer.md | genome_viewer_init / genome_viewer_tick | DRAFT |

---

## 新增蓝图（2026-05-15 驻留Worker管理架构）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0060 | mall | blueprints/mall/BP-0060-resident-worker-manager.md | ThreadPoolManager / ProcessPoolManager / manager_main_loop | ACTIVE |

### 驻留Worker管理架构设计文档

| 文件 | 说明 |
|------|------|
| `BP-0060-resident-worker-manager.md` | 驻留Worker管理架构：进程池/线程池分离 + SHM通道交互 v1.0.0 |

### 架构特点

- **商场主程序零阻塞**：驻留Worker管理逻辑完全从商场主循环剥离
- **进程池与线程池分离**：Thread Pool管理IO密集型，Process Pool管理CPU密集型
- **SHM通道交互**：VEC-MGR-CMD/VEC-MGR-RESP/VEC-TASK-REQ/VEC-TASK-RESP/VEC-WORKER-EVENT
- **并行运行**：管理进程独立运行，不影响商场主循环
- **配置开关**：USE_WORKER_MANAGER，支持平滑迁移

### 基因图谱页面设计文档

| 文件 | 说明 |
|------|------|
| `BP-0061-genome-viewer.md` | 创世纪基因图谱页面消费者蓝图 v1.0.0 |

### 设计特点

- **6层基因展区**: 哲学/授权/运行时/架构/质量/演进，6色编码
- **蓝图基因库**: 7功能域分组蓝图卡片，域色+状态LED
- **宪法文献**: constitution/protocol/workflow三类文档列表
- **单页长滚动**: 锚点导航+IntersectionObserver高亮
- **DNA装饰**: CSS双螺旋动画
- **仪器风格**: 复用instrument-panel/header/btn/LED全局样式体系
- **静态配置**: genome.json定义基因+蓝图+文档数据

---

## 新增蓝图（2026-05-15 时频IQ三维融合可视化）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0062 | consumer | blueprints/consumer/BP-0062-iq3d-fusion.md | iq3d_fusion_init / iq3d_fusion_tick | CODE_DONE |

### 时频IQ三维融合可视化设计文档

| 文件 | 说明 |
|------|------|
| `BP-0062-iq3d-fusion.md` | 时频IQ三维融合可视化系统消费者蓝图 v1.1.0 |

### 设计特点

- **五面板融合**: 时域波形+频域瀑布图+IQ星座图+频谱分析+参数面板同屏
- **Canvas 2D渲染**: 零第三方依赖，沙箱调试期全透明
- **Cooley-Tukey FFT**: O(N log N)位反转+蝶形运算，替代原始O(N²) DFT
- **FFT/STFT双模式**: 频谱折线+时频热力图切换
- **5种FFT窗函数**: rectangular/hamming/hanning/blackman/flattop
- **5种调制信号**: QPSK/QAM16/BPSK/Chirp-Linear/Chirp-Nonlinear
- **5种色映射**: viridis/plasma/inferno/magma/hot
- **标尺+缩放系统**: 共享zoomRuler.ts，auto/x/y/overall四模式
- **瀑布图纹理追加**: bufferCanvas+plotW宽度同步+色映射LUT
- **降采样算法**: LTTB/MinMax自适应
- **仪器风格**: 复用instrument-panel/header/btn/LED全局样式体系
- **画廊集成**: 已添加至UI画廊cat-instrument分类

---

## 新增蓝图（2026-05-15 FFT信号分解与合成演示）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0063 | consumer | blueprints/consumer/BP-0063-fft-decomposition.md | fft_decomposition_init / fft_decomposition_tick | DRAFT |

### FFT信号分解与合成演示设计文档

| 文件 | 说明 |
|------|------|
| `BP-0063-fft-decomposition.md` | FFT信号分解与合成演示系统消费者蓝图 v1.3.0 |

### 设计特点

- **浏览器内四域架构**: SAB=SHM总线、Worker=生产者域、Atomics=标志位仲裁、Canvas=消费者相
- **SharedArrayBuffer**: 零拷贝跨线程数据共享，SHM总线的浏览器肉身
- **Atomics标志位**: 无锁状态同步，高16位商场域/低16位Worker域天然互斥
- **6种信号生成**: Hanning/Hamming/Blackman窗(时域压缩)+股票K线+差分K线+窄带脉冲+高重频调制
- **Cooley-Tukey FFT**: O(N log N)位反转+蝶形运算，FFT Size仅2的幂(64~4096)
- **信号重建动画**: 频谱分量逐次叠加还原原始信号
- **自适应时长控制(方案A)**: duration滑条(3~30秒)替代固定speed，内部自动计算speed=target_k/duration
- **帧率动态修正**: rAF实测actual_fps修正delta_k，适配60/120/144Hz显示器
- **后台标签页暂停**: Page Visibility API检测后台自动暂停，回前台等待用户手动继续
- **主线程降级模式**: SAB不可用时自动切换到主线程内联FFT/信号生成
- **窗函数时域压缩**: Hanning 8%/Hamming 6%/Blackman 5%，频谱展宽12~20倍
- **股票K线信号**: 趋势段+波动率聚集+均值回归，差分K线揭示隐藏趋势
- **参数变更自动重跑**: 信号/FFT/参数change事件200ms防抖后自动清缓存+重跑pipeline
- **Canvas 2D三屏渲染**: 时域波形+频域频谱+合成动画
- **Vanilla JavaScript**: 零框架依赖，纯函数设计
- **COOP/COEP安全上下文**: SAB运行必要条件

---

## 新增蓝图（2026-05-15 数据可视化与报告生成系统）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0064 | consumer | blueprints/consumer/BP-0064-dataviz-report.md | dataviz_report_init / dataviz_report_tick | CODE_DONE |

### 数据可视化与报告生成系统设计文档

| 文件 | 说明 |
|------|------|
| `BP-0064-dataviz-report.md` | 数据可视化与报告生成系统消费者蓝图 v1.8.0 |

### 设计特点

- **Vue 3 + Pinia + Canvas 2D**: 纯前端实现，零后端依赖，画廊即开即用
- **模拟数据生成器**: 6种测试格式(SINAD/THD/THD+N/IMD/SNR/SFDR)，高斯噪声+标称值+公差
- **两列布局**: 左列直方图区(60%)+右列散点图区(40%)
- **直方图数量可选**: 1(单)/2(双)/4(四)，自适应网格布局
- **四区域布局**: 左侧导航(4图标)+主内容区+右侧控制面板(Demo+Filter+Stats)+底部状态栏
- **报告矩阵表**: 按UUT分节×Format分表×Channel×Frequency矩阵，Word风格中文报告
- **报告筛选**: 时间区间+被测单元多选
- **动态维度**: 通道/频率/格式/UUT均从数据动态提取，零硬编码
- **宽表溢出**: 矩阵表overflow-x横向滚动+sticky首列
- **4视图切换**: Dashboard(多窗口可视化)/Report(中文报告)/History(历史)/Data(数据管理)
- **多维度筛选**: 格式(6)/通道(4)/频率(5)/被测单元(5)
- **CSV导出**: 全量/筛选/报告三种导出模式
- **仪器风格**: 复用instrument-panel/header/btn/LED全局样式体系
- **画廊集成**: 已添加至UI画廊cat-instrument分类

---

## 新增蓝图（2026-05-16 微波TR测试报告系统）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0065 | consumer | blueprints/consumer/BP-0065-tr-test-report.md | tr_test_report_init / tr_test_report_tick | CODE_DONE |

### 微波TR测试报告系统设计文档

| 文件 | 说明 |
|------|------|
| `BP-0065-tr-test-report.md` | 微波TR测试报告系统消费者蓝图 v1.0.0 |

### 设计特点

- **Vue 3 + Pinia + Canvas 2D**: 纯前端实现，零后端依赖，画廊即开即用
- **动态format发现**: 从数据自动发现S11/RL/NF/S22/S12/S21/AMPM/EVM/IP3/Phase/ACPR/GainFlatness等指标，零硬编码
- **specLibrary规格库**: 13种format专业规格定义(title/unit/min/max/warn_delta)+动态发现回退
- **Format分布卡片**: 动态Format卡片网格，sparkline+最新值+状态计数
- **热力图**: Channel×Frequency状态分布，Canvas 2D色彩映射
- **两列布局**: 左列直方图区(60%)+右列散点图区(40%)
- **直方图数量可选**: 1(单)/2(双)/4(四)，自适应网格布局
- **Word风格中文报告**: 白底深色文字+蓝色章节标题+浅蓝表头+中文标签
- **报告矩阵表**: 按UUT分节×Format分表×Channel×Frequency矩阵
- **报告筛选**: 时间区间+被测单元多选
- **合格范围**: 每张表标题居中+显示标称值±公差
- **异常数据标记**: warn=浅黄背景(#FEF3C7)，fail=红色背景(#EF4444)+白色文字
- **结论生成**: 自动生成三模板结论(全通过/仅警告/有失败)
- **签章区**: 测试工程师/审核/日期占位符
- **多格式导出**: CSV+Markdown+HTML+PDF(print)
- **宽表溢出**: overflow-x横向滚动+sticky首列
- **继承BP-0064**: 复用两列布局+直方图数量可选+散点图Y轴主轴+矩阵表+异常标记架构
- **画廊集成**: 已规划添加至UI画廊cat-instrument分类

---

## 新增蓝图（2026-05-16 创世纪启动动画）

| 编号 | 功能域 | 文件路径 | 函数名 | 状态 |
|------|--------|----------|--------|------|
| BP-0066 | consumer | blueprints/consumer/BP-0066-genesis-boot-animation.md | genesis_boot_init / genesis_boot_tick | CODE_DONE |

### 创世纪启动动画设计文档

| 文件 | 说明 |
|------|------|
| `BP-0066-genesis-boot-animation.md` | 创世纪启动动画消费者蓝图 v2.0.0 |

### 设计特点

- **Vue 3 + Canvas 2D + CSS关键帧**: 纯前端实现，零第三方依赖，画廊即开即用
- **五纪元历史演进循环**: 原始集市(5s)→封建行会(5s)→工业流水线(5s)→平台垄断(5s)→创世纪商场(5s)，总循环25s
- **纪元色彩映射**: 暖土→冷石→冷钢→冷蓝→深空，色温渐进变化
- **拓扑可视化**: 点对点→星型黑洞→线性流水线→中心化金字塔→去中心化三角
- **演进熵增**: 每个纪元视觉复杂度递增，从简单CSS动画到Canvas粒子系统
- **历史断层过渡**: clip-path撕裂+粒子爆炸+拓扑重组+色温跃迁，1.5s过渡
- **觉醒触发**: 任意时刻点击/Enter/Space触发坍缩→白光→登录界面
- **登录界面**: glass Card变体，用户名/密码/记住我/进入商场/访客浏览
- **三级性能降级**: 高性能(全特效)→中性能(无粒子)→低性能(纯CSS)→无障碍(静态)
- **纪元预加载**: 双缓冲预渲染下一纪元初始帧
- **鼠标悬停暂停**: 当前纪元暂停，移出继续
- **画廊集成**: 已添加至UI画廊cat-visualization分类

---

## 已驳回蓝图

> **驳回日期**: 2026-05-10
> **驳回报告**: REJECT-2026-05-10-001.md
> **驳回原因**: 蓝图格式与《蓝图标准格式总则v1.0》不符

| 编号 | 功能域 | 文件路径 | 说明 | 状态 |
|------|--------|----------|------|------|
| BP-audit-worker | audit | blueprints/audit/BP-audit-worker.md | 审计Worker设计 | ❌ 驳回 |
| BP-consumer-worker | consumer | blueprints/consumer/BP-consumer-worker.md | 消费者Worker设计 | ❌ 驳回 |
| BP-scheduler-worker | scheduler | blueprints/scheduler/BP-scheduler-worker.md | 调度Worker设计 | ❌ 驳回 |
| BP-security-worker | security | blueprints/security/BP-security-worker.md | 安全Worker设计 | ❌ 驳回 |

---

## 场景文档索引

| 场景编号 | 场景名称 | 文件路径 | 关联蓝图 |
|----------|----------|----------|----------|
| SC-0001 | 基础IO回路 | blueprints/SC-0001-basic-io-loop.md | BP-0001, BP-0002 |
| SC-0002 | 网络遥测回路 | blueprints/SC-0002-network-telemetry-loop.md | BP-0016, BP-0017, BP-0002 |
| SC-0003 | 仿真验证回路 | blueprints/SC-0003-simulation-verification-loop.md | BP-0018, BP-0017, BP-0002 |
| SC-0004 | 多源融合回路 | blueprints/SC-0004-multi-source-fusion-loop.md | BP-0001, BP-0016, BP-0018, BP-0019, BP-0002 |
| SC-0005 | 闭环控制回路 | blueprints/SC-0005-closed-control-loop.md | BP-0016, BP-0017, BP-0021 |

---

## 功能域分类

| 功能域 | 目录 | 说明 |
|--------|------|------|
| producer | blueprints/producer/ | 生产者蓝图（数据输入源） |
| consumer | blueprints/consumer/ | 消费者蓝图（数据输出/渲染） |
| scheduler | blueprints/scheduler/ | 调度器蓝图（任务分发/负载均衡） |
| security | blueprints/security/ | 安全模块蓝图（认证/沙箱/隔离） |
| audit | blueprints/audit/ | 审计模块蓝图（日志/追溯/计费） |
| shm | blueprints/shm/ | 共享内存蓝图（矢量管理） |
| mall | blueprints/mall/ | 商场核心蓝图（异常处理/状态机/基础设施） |

---

## 新增蓝图流程

1. 确定功能域
2. 分配蓝图编号（BP-{四位编号}）
3. 创建文件：`design/blueprints/{功能域}/BP-{编号}-{名称}.md`
4. 更新本索引文件
