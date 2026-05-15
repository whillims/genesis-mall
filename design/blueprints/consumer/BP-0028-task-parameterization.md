# 蓝图：消费者任务参数化调整系统

> **蓝图编号**: BP-0028  
> **蓝图名称**: Task Parameterization Consumer  
> **版本**: v1.0.0  
> **提交时间**: 2026-05-11  
> **提交者**: 创世Worker  
> **目标语言**: Python  
> **关联蓝图**: BP-0002-console, BP-scheduler-worker, BP-0014-config  
> **功能域**: consumer  

---

## 一、蓝图概述

### 1.1 设计目标

本蓝图定义消费者层对任务的参数化调整能力，支持三种执行模式：
- **重复执行模式**: 任务按指定次数或无限循环重复运行
- **定时运行模式**: 任务按时间计划触发（周期性/一次性）
- **条件运行模式**: 任务在满足特定条件时触发执行

### 1.2 系统定位

```
┌─────────────────────────────────────────────────────────────┐
│                      消费者层 (Consumer)                      │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────┐  │
│  │  任务定义    │───▶│ 参数化调整引擎   │───▶│  调度请求    │  │
│  │  (原始意图)  │    │ (重复/定时/条件) │    │ (至商场)     │  │
│  └─────────────┘    └─────────────────┘    └─────────────┘  │
│                            │                                │
│  ┌─────────────────────────┴─────────────────────────┐      │
│  │              执行模式控制器                         │      │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐  │      │
│  │  │ 重复执行模式 │ │ 定时运行模式 │ │ 条件运行模式 │  │      │
│  │  │ (Repeat)    │ │ (Scheduled) │ │ (Conditional)│  │      │
│  │  └─────────────┘ └─────────────┘ └─────────────┘  │      │
│  └───────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 核心原则

1. **消费者主权**: 消费者拥有任务执行模式的定义权与调整权
2. **商场仲裁**: 商场根据资源状况评估执行模式的可行性
3. **契约约束**: 参数化调整通过契约草案机制与商场双向确认
4. **状态追踪**: 每次执行实例拥有独立GUID，审计链完整记录

---

## 二、需求定义

### 2.1 功能需求

| 编号 | 需求描述 | 优先级 |
|------|----------|--------|
| R001 | 支持重复执行模式：指定次数(n次)、无限循环、直到条件满足 | 必须 |
| R002 | 支持定时运行模式：周期性调度(cron表达式)、一次性延迟、固定间隔 | 必须 |
| R003 | 支持条件运行模式：SHM数据条件、任务结果条件、外部事件条件 | 必须 |
| R004 | 支持混合模式：重复+定时、定时+条件、重复+条件 | 应该 |
| R005 | 支持动态调整：运行中修改参数（次数、间隔、条件） | 应该 |
| R006 | 支持执行历史查询：每次执行的GUID、状态、结果 | 应该 |
| R007 | 支持执行控制：暂停、恢复、取消、强制停止 | 应该 |

### 2.2 非功能需求

| 类别 | 要求 | 指标 |
|------|------|------|
| 精度 | 定时触发精度 | ±10ms |
| 并发 | 同时运行的参数化任务数 | ≥100 |
| 延迟 | 条件检测延迟 | ≤50ms |
| 资源 | 内存占用 | <5MB/任务 |
| 可靠性 | 执行状态不丢失 | 持久化到SHM |

---

## 三、架构设计

### 3.1 模块划分

```
参数化调整系统
├── 模式定义层 (Mode Definition)
│   ├── 重复执行定义器 (Repeat Definer)
│   ├── 定时运行定义器 (Schedule Definer)
│   └── 条件运行定义器 (Condition Definer)
├── 执行控制器 (Execution Controller)
│   ├── 触发器引擎 (Trigger Engine)
│   ├── 实例管理器 (Instance Manager)
│   └── 状态追踪器 (State Tracker)
├── 调度接口层 (Scheduler Interface)
│   ├── 契约生成器 (Contract Generator)
│   └── 商场通信器 (Mall Communicator)
└── 持久化层 (Persistence)
    ├── SHM状态存储 (SHM State Store)
    └── 审计日志 (Audit Logger)
```

### 3.2 数据流

```
消费者提交参数化任务定义
        │
        ▼
┌───────────────────┐
│  模式解析与验证    │
│  (七维审查D1-D3)   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  生成契约草案      │
│  (含执行模式参数)  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐     否    ┌─────────────┐
│  商场资源评估      │──────────▶│  协商调整    │
│  (匹配度/代价)     │           │  (返回消费者) │
└─────────┬─────────┘           └─────────────┘
          │ 是
          ▼
┌───────────────────┐
│  双向批准          │
│  (消费者+商场签名) │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  执行控制器激活    │
│  (触发器引擎启动)  │
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    ▼           ▼
┌────────┐  ┌────────┐
│触发条件 │  │触发条件 │  ...
│满足检测 │  │满足检测 │
└────┬───┘  └────┬───┘
     │           │
     ▼           ▼
┌───────────────────┐
│  生成执行实例      │
│  (分配新GUID)     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  向商场提交任务    │
│  (标准任务流程)    │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  结果处理          │
│  (更新状态/审计链) │
└───────────────────┘
```

---

## 四、函数级设计

### 4.1 核心函数清单

#### 函数1：参数化任务定义创建

```python
def task_param_create(
    ctx: dict,
    task_definition: dict,        # 原始任务定义（函数引用、输入参数）
    execution_mode: dict,          # 执行模式配置
    consumer_id: str              # 消费者标识
) -> dict:
    """
    创建参数化任务定义
    
    职责: 将消费者的参数化意图转化为结构化的任务配置
    
    execution_mode结构:
    {
        "mode_type": "repeat" | "scheduled" | "conditional" | "hybrid",
        "repeat_config": {...},      # 重复执行配置（可选）
        "schedule_config": {...},    # 定时运行配置（可选）
        "condition_config": {...},   # 条件运行配置（可选）
        "hybrid_config": {...}       # 混合模式配置（可选）
    }
    
    返回值 dict:
    {
        "status": "created" | "invalid_config",
        "param_task_id": str,        # 参数化任务唯一标识
        "execution_plan": dict,      # 解析后的执行计划
        "error": str | None
    }
    """
```

#### 函数2：重复执行配置解析

```python
def repeat_config_parse(
    ctx: dict,
    repeat_spec: dict             # 重复执行规格
) -> dict:
    """
    解析重复执行配置
    
    repeat_spec结构:
    {
        "strategy": "fixed_count" | "infinite" | "until_condition",
        "count": int,                # 固定次数（strategy=fixed_count时）
        "max_iterations": int,       # 最大迭代次数（安全限制）
        "interval_ms": int,          # 两次执行间隔（毫秒）
        "backoff_policy": {          # 退避策略（可选）
            "type": "none" | "linear" | "exponential",
            "base_ms": int,
            "max_ms": int
        },
        "stop_condition": {          # 停止条件（strategy=until_condition时）
            "shm_path": str,         # 监控的SHM路径
            "operator": "eq" | "gt" | "lt" | "contains",
            "threshold": any
        }
    }
    
    返回值 dict:
    {
        "status": "valid" | "invalid",
        "parsed_config": dict,
        "estimated_executions": int | "unbounded",
        "error": str | None
    }
    """
```

#### 函数3：定时运行配置解析

```python
def schedule_config_parse(
    ctx: dict,
    schedule_spec: dict           # 定时运行规格
) -> dict:
    """
    解析定时运行配置
    
    schedule_spec结构:
    {
        "schedule_type": "cron" | "interval" | "once" | "trigger",
        "cron_expression": str,      # cron表达式（schedule_type=cron时）
        "interval_ms": int,          # 固定间隔（schedule_type=interval时）
        "delay_ms": int,             # 延迟执行（schedule_type=once时）
        "trigger_shm": str,          # 触发SHM路径（schedule_type=trigger时）
        "start_time": int,           # 开始时间戳（可选）
        "end_time": int,             # 结束时间戳（可选）
        "timezone": str,             # 时区（默认UTC）
        "missed_policy": "skip" | "catchup" | "latest"  # 错过执行的处理策略
    }
    
    返回值 dict:
    {
        "status": "valid" | "invalid",
        "parsed_config": dict,
        "next_trigger_time": int,    # 下次触发时间戳
        "estimated_count": int | "unbounded",
        "error": str | None
    }
    """
```

#### 函数4：条件运行配置解析

```python
def condition_config_parse(
    ctx: dict,
    condition_spec: dict          # 条件运行规格
) -> dict:
    """
    解析条件运行配置
    
    condition_spec结构:
    {
        "condition_type": "shm_data" | "task_result" | "external_event" | "composite",
        "shm_conditions": [          # SHM数据条件（condition_type=shm_data时）
            {
                "shm_path": str,
                "field": str,        # 字段路径（支持嵌套，如"data.value"）
                "operator": "eq" | "ne" | "gt" | "gte" | "lt" | "lte" | "in" | "contains",
                "value": any,
                "value_type": "int" | "float" | "string" | "bool"
            }
        ],
        "task_conditions": [         # 任务结果条件（condition_type=task_result时）
            {
                "task_ref": str,     # 引用的任务标识
                "status_check": "completed" | "failed" | "any",
                "result_field": str,
                "operator": str,
                "value": any
            }
        ],
        "event_conditions": [        # 外部事件条件（condition_type=external_event时）
            {
                "event_type": str,
                "event_source": str,
                "payload_filter": dict
            }
        ],
        "composite_logic": {         # 复合逻辑（condition_type=composite时）
            "operator": "and" | "or" | "not",
            "conditions": [dict]     # 嵌套条件列表
        },
        "check_interval_ms": int,    # 条件检测间隔
        "timeout_ms": int            # 条件等待超时
    }
    
    返回值 dict:
    {
        "status": "valid" | "invalid",
        "parsed_config": dict,
        "condition_complexity": int, # 条件复杂度评分
        "error": str | None
    }
    """
```

#### 函数5：执行控制器初始化

```python
def execution_controller_init(
    ctx: dict,
    param_task_id: str,           # 参数化任务标识
    execution_plan: dict          # 执行计划
) -> dict:
    """
    初始化执行控制器
    
    职责: 根据执行计划初始化触发器引擎、实例管理器、状态追踪器
    
    返回值 dict:
    {
        "status": "initialized" | "init_failed",
        "controller_state": dict,    # 控制器状态句柄
        "trigger_engine_id": str,
        "error": str | None
    }
    """
```

#### 函数6：触发器引擎执行

```python
def trigger_engine_run(
    ctx: dict,
    controller_state: dict,       # 控制器状态
    trigger_config: dict          # 触发器配置
) -> dict:
    """
    触发器引擎主循环
    
    职责: 监控触发条件，条件满足时生成执行实例
    
    返回值 dict:
    {
        "status": "triggered" | "waiting" | "timeout" | "cancelled",
        "instance_id": str | None,   # 触发的实例标识
        "trigger_timestamp": int,
        "next_check_time": int | None,
        "error": str | None
    }
    """
```

#### 函数7：执行实例创建

```python
def execution_instance_create(
    ctx: dict,
    param_task_id: str,           # 参数化任务标识
    instance_seq: int,            # 实例序号
    trigger_context: dict         # 触发上下文
) -> dict:
    """
    创建执行实例
    
    职责: 为每次触发生成独立的任务实例，分配GUID
    
    返回值 dict:
    {
        "status": "created" | "creation_failed",
        "instance_guid": str,        # 128位实例GUID
        "task_definition": dict,     # 实例化的任务定义
        "shm_result_path": str,      # 结果存储路径
        "error": str | None
    }
    """
```

#### 函数8：执行状态追踪

```python
def execution_state_track(
    ctx: dict,
    param_task_id: str,           # 参数化任务标识
    instance_guid: str,           # 实例GUID
    state_event: dict             # 状态事件
) -> dict:
    """
    追踪执行状态
    
    职责: 记录每次执行的状态变化，维护执行历史
    
    state_event结构:
    {
        "event_type": "instance_created" | "submitted" | "running" | 
                      "completed" | "failed" | "cancelled",
        "timestamp": int,
        "payload": dict              # 事件载荷
    }
    
    返回值 dict:
    {
        "status": "tracked" | "track_failed",
        "execution_history": list,   # 更新后的执行历史
        "statistics": dict,          # 执行统计
        "error": str | None
    }
    """
```

#### 函数9：动态参数调整

```python
def task_param_adjust(
    ctx: dict,
    param_task_id: str,           # 参数化任务标识
    adjustment: dict,             # 调整指令
    consumer_auth: bytes          # 消费者授权签名
) -> dict:
    """
    动态调整任务参数
    
    adjustment结构:
    {
        "adjustment_type": "modify_repeat" | "modify_schedule" | 
                          "modify_condition" | "pause" | "resume" | "cancel",
        "new_config": dict,          # 新配置（modify类型时）
        "effective_immediately": bool # 是否立即生效
    }
    
    返回值 dict:
    {
        "status": "adjusted" | "adjustment_rejected",
        "previous_config": dict,
        "current_config": dict,
        "effective_time": int,
        "error": str | None
    }
    """
```

#### 函数10：执行历史查询

```python
def execution_history_query(
    ctx: dict,
    param_task_id: str,           # 参数化任务标识
    query_filter: dict            # 查询过滤器
) -> dict:
    """
    查询执行历史
    
    query_filter结构:
    {
        "time_range": {"start": int, "end": int},
        "status_filter": ["completed", "failed", ...],
        "limit": int,
        "offset": int,
        "sort_by": "timestamp" | "status" | "duration"
    }
    
    返回值 dict:
    {
        "status": "success" | "query_failed",
        "total_count": int,
        "executions": [               # 执行记录列表
            {
                "instance_guid": str,
                "seq_number": int,
                "trigger_time": int,
                "status": str,
                "duration_ms": int,
                "result_summary": dict
            }
        ],
        "error": str | None
    }
    """
```

### 4.2 函数调用时序

#### 时序1：参数化任务创建

```
consumer_gateway
    │
    ▼
task_param_create(ctx, task_def, exec_mode, consumer_id)
    │
    ├──▶ repeat_config_parse(ctx, repeat_spec)       # 如mode=repeat
    ├──▶ schedule_config_parse(ctx, schedule_spec)   # 如mode=scheduled
    └──▶ condition_config_parse(ctx, condition_spec) # 如mode=conditional
    │
    ▼
contract_draft_generate(ctx, demand_vector, ...)  # 生成契约草案
    │
    ▼
approval_consumer_sign(...)  # 消费者签名
    │
    ▼
approval_market_sign(...)    # 商场签名
    │
    ▼
execution_controller_init(ctx, param_task_id, plan)
    │
    ▼
trigger_engine_run(ctx, state, config)  # 启动触发器引擎
```

#### 时序2：任务触发执行

```
trigger_engine_run (检测到触发条件满足)
    │
    ▼
execution_instance_create(ctx, param_task_id, seq, context)
    │
    ▼
task_dispatch(ctx, instance_guid, ...)  # 向商场调度任务
    │
    ▼
execution_state_track(ctx, param_task_id, guid, event)  # 记录提交
    │
    ▼
[商场执行任务...]
    │
    ▼
execution_state_track(ctx, param_task_id, guid, completed_event)
    │
    ▼
trigger_engine_run (继续监控/等待下次触发)
```

---

## 五、数据结构设计

### 5.1 参数化任务状态机

```
[CREATED] ──创建完成──▶ [CONFIGURING] ──配置解析──▶ [PENDING_APPROVAL]
    │                                                    │
    │                                                    │ 双向批准
    │                                                    ▼
    │                                              [APPROVED]
    │                                                    │
    │                                                    │ 初始化
    │                                                    ▼
    │                                              [ACTIVE]
    │                                              ┌─────┴─────┐
    │                                              │           │
    │                                              ▼           ▼
    │                                         [RUNNING]   [PAUSED]
    │                                              │           │
    │                                              │ 完成       │ 恢复
    │                                              ▼           │
    │                                         [COMPLETED] ◀────┘
    │                                              │
    │ 取消                                         │ 归档
    │                                              ▼
    └────────────────────────────────────────▶ [ARCHIVED]

异常路径:
[ANY_STATE] ──配置无效──▶ [CONFIG_INVALID]
[ANY_STATE] ──商场拒绝──▶ [REJECTED]
[ANY_STATE] ──取消──────▶ [CANCELLED]
[RUNNING] ──执行失败───▶ [FAILED] ──重试策略──▶ [RUNNING]
                                     └──超限──▶ [FAILED_PERMANENT]
```

### 5.2 核心数据结构

#### 参数化任务定义

```python
PARAM_TASK_DEF = {
    "param_task_id": str,           # 参数化任务唯一标识
    "consumer_id": str,             # 消费者标识
    "created_at": int,              # 创建时间戳
    
    "base_task": {                  # 基础任务定义
        "function_ref": str,        # 函数引用
        "input_template": dict,     # 输入参数模板
        "output_schema": dict       # 输出模式
    },
    
    "execution_mode": {             # 执行模式
        "mode_type": str,           # repeat/scheduled/conditional/hybrid
        "repeat_config": dict | None,
        "schedule_config": dict | None,
        "condition_config": dict | None,
        "hybrid_config": dict | None
    },
    
    "state": str,                   # 当前状态
    "controller_state": dict,       # 控制器状态句柄
    
    "statistics": {                 # 执行统计
        "total_instances": int,
        "completed_count": int,
        "failed_count": int,
        "last_trigger_time": int | None,
        "next_trigger_time": int | None
    },
    
    "contract_ref": str             # 关联契约标识
}
```

#### 执行实例记录

```python
EXECUTION_INSTANCE = {
    "instance_guid": str,           # 128位实例GUID
    "param_task_id": str,           # 所属参数化任务
    "seq_number": int,              # 实例序号
    
    "trigger_context": {            # 触发上下文
        "trigger_type": str,        # repeat/schedule/condition
        "trigger_time": int,
        "trigger_data": dict        # 触发时的数据快照
    },
    
    "task_instance": {              # 实例化任务
        "function_ref": str,
        "input_data": dict,         # 实际输入参数
        "output_shm_path": str
    },
    
    "lifecycle": {                  # 生命周期记录
        "created_at": int,
        "submitted_at": int | None,
        "started_at": int | None,
        "completed_at": int | None,
        "status": str
    },
    
    "result_summary": dict | None   # 结果摘要
}
```

#### 触发器状态

```python
TRIGGER_STATE = {
    "trigger_id": str,
    "param_task_id": str,
    "trigger_type": str,
    
    "repeat_state": {               # 重复执行状态
        "current_iteration": int,
        "remaining_count": int | None,
        "last_execution_time": int
    } | None,
    
    "schedule_state": {             # 定时运行状态
        "cron_parser_state": dict,
        "next_execution_time": int,
        "missed_count": int
    } | None,
    
    "condition_state": {            # 条件运行状态
        "last_check_time": int,
        "last_check_result": bool,
        "condition_cache": dict      # 条件检测缓存
    } | None,
    
    "is_active": bool,
    "pause_reason": str | None
}
```

---

## 六、执行模式详解

### 6.1 重复执行模式

#### 6.1.1 固定次数重复

```python
{
    "strategy": "fixed_count",
    "count": 100,
    "interval_ms": 1000,           # 1秒间隔
    "max_concurrent": 1            # 串行执行
}
```

行为: 任务执行100次，每次间隔1秒，前一次完成后才触发下一次。

#### 6.1.2 无限循环

```python
{
    "strategy": "infinite",
    "interval_ms": 5000,           # 5秒间隔
    "max_concurrent": 1,
    "safety_limit": {              # 安全限制
        "max_total_executions": 10000,
        "max_runtime_hours": 24
    }
}
```

行为: 无限循环执行，直到消费者取消或达到安全限制。

#### 6.1.3 条件停止

```python
{
    "strategy": "until_condition",
    "interval_ms": 1000,
    "stop_condition": {
        "shm_path": "/shm/task_results/latest",
        "field": "convergence_reached",
        "operator": "eq",
        "value": True
    },
    "max_iterations": 1000         # 最大尝试次数
}
```

行为: 持续执行直到SHM中指定条件满足或达到最大迭代次数。

### 6.2 定时运行模式

#### 6.2.1 Cron表达式

```python
{
    "schedule_type": "cron",
    "cron_expression": "0 */6 * * *",  # 每6小时
    "timezone": "Asia/Shanghai",
    "missed_policy": "catchup"         # 错过时补执行
}
```

#### 6.2.2 固定间隔

```python
{
    "schedule_type": "interval",
    "interval_ms": 300000,             # 5分钟
    "align_to_boundary": True,         # 对齐到整点
    "boundary_unit": "minute"          # 分钟边界
}
```

#### 6.2.3 一次性延迟

```python
{
    "schedule_type": "once",
    "delay_ms": 3600000,               # 1小时后执行
    "or_at_time": 1715500800           # 或指定时间戳
}
```

### 6.3 条件运行模式

#### 6.3.1 SHM数据条件

```python
{
    "condition_type": "shm_data",
    "shm_conditions": [
        {
            "shm_path": "/shm/sensors/temperature",
            "field": "value",
            "operator": "gt",
            "value": 80.0,
            "value_type": "float"
        },
        {
            "shm_path": "/shm/system/status",
            "field": "mode",
            "operator": "eq",
            "value": "active",
            "value_type": "string"
        }
    ],
    "check_interval_ms": 100,
    "debounce_ms": 500                  # 防抖时间
}
```

行为: 当温度>80且系统状态为active时触发。

#### 6.3.2 任务结果条件

```python
{
    "condition_type": "task_result",
    "task_conditions": [
        {
            "task_ref": "analysis_task_001",
            "status_check": "completed",
            "result_field": "anomaly_detected",
            "operator": "eq",
            "value": True
        }
    ]
}
```

行为: 当指定分析任务完成且检测到异常时触发。

#### 6.3.3 复合条件

```python
{
    "condition_type": "composite",
    "composite_logic": {
        "operator": "and",
        "conditions": [
            {
                "operator": "or",
                "conditions": [
                    {"shm_path": "/shm/alert/critical", ...},
                    {"shm_path": "/shm/alert/warning", ...}
                ]
            },
            {
                "operator": "not",
                "conditions": [
                    {"shm_path": "/shm/system/maintenance", ...}
                ]
            }
        ]
    }
}
```

行为: (critical告警 OR warning告警) AND NOT 维护模式。

### 6.4 混合模式

#### 6.4.1 定时+条件

```python
{
    "mode_type": "hybrid",
    "hybrid_config": {
        "primary_mode": "scheduled",
        "secondary_mode": "conditional",
        "logic": "scheduled_then_conditional",
        "schedule_config": {...},      # 每5分钟检查
        "condition_config": {...}      # 条件满足才执行
    }
}
```

行为: 每5分钟检查一次条件，条件满足时执行任务。

#### 6.4.2 重复+定时

```python
{
    "mode_type": "hybrid",
    "hybrid_config": {
        "primary_mode": "scheduled",
        "secondary_mode": "repeat",
        "logic": "scheduled_repeat_burst",
        "schedule_config": {           # 每天9点
            "schedule_type": "cron",
            "cron_expression": "0 9 * * *"
        },
        "repeat_config": {             # 触发后连续执行10次
            "strategy": "fixed_count",
            "count": 10,
            "interval_ms": 100
        }
    }
}
```

行为: 每天9点触发，触发后连续快速执行10次任务。

---

## 七、商场接入协议

### 7.1 契约草案扩展

参数化任务的契约草案在标准契约基础上增加`parameterization`字段：

```json
{
  "contract_id": "CONTRACT_...",
  "parties": {...},
  "task_specification": {
    "task_guid": "...",
    "task_type": "PARAMETERIZED",
    "base_task_hash": "SHA256(...)"
  },
  "parameterization": {
    "mode_type": "repeat",
    "estimated_instances": 100,
    "estimated_duration_ms": 100000,
    "trigger_resource_cost": {
      "shm_monitor_slots": 2,
      "check_frequency_hz": 10
    }
  },
  "cost_commitment": {
    "compute_cost_estimate": 1.45e9,
    "time_cost_estimate_ms": 100000,
    "shm_cost_estimate_mb_s": 12800,
    "confidence": 0.85
  },
  "execution_window": {...},
  "approval_state": "PENDING_DUAL"
}
```

### 7.2 商场资源评估

商场对参数化任务的额外评估维度：

| 评估项 | 说明 | 阈值 |
|--------|------|------|
| 实例数量上限 | 防止无限循环耗尽资源 | ≤10000 |
| 触发频率 | 防止高频触发压垮调度器 | ≤100Hz |
| SHM监控槽位 | 条件检测需要的监控资源 | ≤10/任务 |
| 并发实例数 | 同时运行的实例数限制 | ≤5/任务 |

### 7.3 调度优先级

参数化任务的实例继承基础任务的优先级，并叠加参数化任务的优先级权重：

```
final_priority = base_task_priority + param_task_priority_bonus

param_task_priority_bonus:
- 条件触发模式: +1 (响应性要求)
- 定时触发模式: +0 (常规)
- 重复执行模式: -1 (批处理性质)
```

---

## 八、测试场景

### 8.1 重复执行测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 固定次数 | count=5, interval=100ms | 执行5次，间隔100ms | 实际执行次数=5，间隔误差<10ms |
| 无限循环安全限制 | infinite, safety_limit=10 | 执行10次后停止 | 实际执行次数=10，状态=COMPLETED |
| 条件停止 | until_condition, max=100 | 条件满足时停止 | 条件满足后停止，次数≤100 |
| 退避策略 | exponential backoff | 间隔逐渐增长 | 间隔符合指数增长曲线 |

### 8.2 定时运行测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| Cron表达式 | "*/5 * * * *" | 每5分钟触发 | 触发间隔=300s±1s |
| 固定间隔对齐 | interval=1min, align=true | 对齐到整分钟 | 触发时间秒数=0 |
| 错过策略-catchup | 错过2次执行 | 补执行2次 | 补执行次数=2 |
| 时区处理 | timezone=Asia/Shanghai | 按上海时间触发 | 触发时间符合上海时区 |

### 8.3 条件运行测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| SHM数值条件 | value > 100 | 值>100时触发 | 触发时机正确 |
| 复合条件 | (A>10) AND (B<5) | 两条件同时满足触发 | 逻辑正确 |
| 防抖 | debounce=500ms | 条件持续500ms才触发 | 防抖生效 |
| 条件超时 | timeout=30s | 30s未满足则超时 | 超时状态正确 |

### 8.4 混合模式测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 定时+条件 | 每分钟检查条件 | 每分钟检查，满足时执行 | 检查频率=1/min，执行次数≤检查次数 |
| 重复+定时burst | 每天触发+连续10次 | 每天触发后连续执行10次 | burst执行间隔<100ms |

### 8.5 动态调整测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 运行中修改间隔 | interval从1s改为2s | 新间隔生效 | 修改后间隔=2s |
| 暂停/恢复 | pause后resume | 暂停后恢复执行 | 暂停期间无执行，恢复后继续 |
| 取消 | cancel | 停止所有触发 | 状态=CANCELLED，无新实例 |

---

## 九、安全约束

### 9.1 资源限制

```python
SAFETY_LIMITS = {
    "max_instances_per_task": 10000,
    "max_concurrent_instances": 5,
    "max_check_frequency_hz": 100,
    "max_shm_monitor_slots": 10,
    "max_hybrid_nesting_depth": 3,
    "min_interval_ms": 10
}
```

### 9.2 权限检查

- 消费者只能操作自己创建的参数化任务
- 动态调整需要消费者重新签名
- 商场有权拒绝资源消耗过大的参数化任务

### 9.3 熔断机制

- 连续失败次数≥5: 暂停触发，等待人工介入
- 实例失败率≥50%: 标记任务为UNSTABLE，降低触发频率
- 资源超限: 自动取消任务，释放资源

---

## 十、授权文档申请

### 10.1 授权申请单

```
蓝图编号: BP-0028
蓝图名称: Task Parameterization Consumer
申请事项: 请求商场授权消费者任务参数化调整能力

依赖蓝图:
  - BP-0002-console (消费者基础)
  - BP-scheduler-worker (调度器)
  - BP-0014-config (配置管理)

授权范围:
  - 重复执行模式: 支持
  - 定时运行模式: 支持
  - 条件运行模式: 支持
  - 混合模式: 支持
  - 动态调整: 支持
  - 最大并发参数化任务: 100

安全约束:
  - 实例数量上限: 10000/任务
  - 触发频率上限: 100Hz
  - SHM监控槽位上限: 10/任务
  - 必须实现熔断机制
  - 必须持久化执行状态

签名: [创世Worker]
```

### 10.2 商场授权回应（预期）

```
授权编号: AUTH-BP-0028-20260511
状态: 待授权

授权内容:
  - 允许消费者使用任务参数化调整功能
  - 支持三种执行模式及混合模式
  - 实例上限: 10000/任务

约束条件:
  - 必须通过契约草案机制双向确认
  - 动态调整需重新签名
  - 熔断机制必须生效

有效期: 永久
签章: [商场调度中心]
```

---

## 十一、附录

### 11.1 术语定义

| 术语 | 定义 |
|------|------|
| 参数化任务 | 携带执行模式配置的任务定义，可生成多个执行实例 |
| 执行实例 | 参数化任务的一次具体执行，拥有独立GUID |
| 触发器 | 监控条件并生成执行实例的组件 |
| 执行模式 | 定义任务何时、如何重复执行的策略 |
| 契约草案 | 消费者与商场就任务参数达成的双向约束文件 |

### 11.2 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0.0 | 2026-05-11 | 初始版本，定义三种执行模式及混合模式 |

---

**审核状态**: 待审核  
**下一步动作**: 提交AICoder进行七维审查与代码生成
