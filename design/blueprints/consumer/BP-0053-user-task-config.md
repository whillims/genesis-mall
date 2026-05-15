# 蓝图：用户任务生成与静态配置系统

> **蓝图编号**: BP-0053  
> **蓝图名称**: User Task Generation & Static Config  
> **版本**: v1.0.0  
> **提交时间**: 2026-05-13  
> **提交者**: 创世Worker  
> **目标语言**: Python  
> **关联蓝图**: BP-0045-task-management-worker, BP-0028-task-parameterization, BP-0048-task-conception  
> **功能域**: consumer  

---

## 一、蓝图概述

### 1.1 设计目标

本蓝图定义消费者层的**用户任务生成与静态配置系统**，赋予用户以下能力：

1. **任务生成**：用户创建任务，支持子任务逻辑链编排和交易链异常处理
2. **任务静态配置**：为单个任务定制个性数据和交易参数
3. **静态配置序列**：批量配置+批量执行多任务
4. **资源冲突检测**：系统自动检测多任务间的资源冲突，生成决策报告供用户裁决

### 1.2 系统定位

```
┌──────────────────────────────────────────────────────────────────┐
│                        消费者层 (Consumer)                        │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  用户任务生成  │                                                │
│  │  ┌────────┐  │                                                │
│  │  │子任务链 │  │    ┌──────────────┐    ┌──────────────────┐   │
│  │  │编排引擎 │──┼───▶│ 任务静态配置  │───▶│  静态配置序列    │   │
│  │  └────────┘  │    │ (个性数据/    │    │  (批量执行器)    │   │
│  │  ┌────────┐  │    │  交易参数)    │    └────────┬─────────┘   │
│  │  │交易链   │  │    └──────────────┘             │             │
│  │  │异常处理 │  │                                │             │
│  │  └────────┘  │                                ▼             │
│  └──────────────┘    ┌──────────────────────────────────┐       │
│                      │      资源冲突检测器               │       │
│                      │  ┌─────────┐  ┌──────────────┐  │       │
│                      │  │SHM冲突  │  │ 算力资源冲突  │  │       │
│                      │  │检测     │  │ 检测         │  │       │
│                      │  └─────────┘  └──────────────┘  │       │
│                      │  ┌─────────────────────────┐    │       │
│                      │  │ 冲突决策报告 → 用户裁决  │    │       │
│                      │  └─────────────────────────┘    │       │
│                      └──────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 核心原则

1. **用户主权**：用户拥有任务定义权、配置权、冲突裁决权
2. **配置与执行分离**：静态配置是声明式的，不直接触发执行
3. **冲突透明**：所有资源冲突必须检测并报告，禁止静默覆盖
4. **链式编排**：子任务通过DAG逻辑链连接，支持条件分支和异常回退

---

## 二、需求定义

### 2.1 功能需求

| 编号 | 需求描述 | 优先级 |
|------|----------|--------|
| R001 | 用户创建任务，指定任务标签(task_colle/task_a/task_sa) | 必须 |
| R002 | 支持子任务DAG逻辑链：串行、并行、条件分支 | 必须 |
| R003 | 交易链异常处理：超时、失败、回退策略 | 必须 |
| R004 | 任务静态配置：为任务绑定个性数据和交易参数 | 必须 |
| R005 | 静态配置序列：有序的配置列表，支持批量执行 | 必须 |
| R006 | 资源冲突检测：SHM路径冲突、算力资源竞争 | 必须 |
| R007 | 冲突决策报告：生成冲突报告，等待用户裁决 | 必须 |
| R008 | 配置继承：子任务继承父任务配置，可覆盖 | 应该 |
| R009 | 配置模板：常用配置可保存为模板复用 | 应该 |
| R010 | 配置校验：执行前验证配置完整性和合法性 | 应该 |

### 2.2 非功能需求

| 类别 | 要求 | 指标 |
|------|------|------|
| 并发 | 同时检测的任务数 | ≥1000 |
| 延迟 | 冲突检测延迟 | ≤10ms/任务 |
| 容量 | 单个配置序列最大任务数 | ≤10000 |
| 精度 | DAG依赖解析 | 无环保证 |

---

## 三、架构设计

### 3.1 模块划分

```
用户任务生成与静态配置系统
├── 任务生成模块 (Task Generation)
│   ├── 子任务链编排器 (Subtask Chain Orchestrator)
│   ├── 交易链异常处理器 (Trade Chain Exception Handler)
│   └── 任务标签生成器 (Task Label Generator)
├── 静态配置模块 (Static Config)
│   ├── 配置定义器 (Config Definer)
│   ├── 配置校验器 (Config Validator)
│   └── 配置模板管理器 (Config Template Manager)
├── 配置序列模块 (Config Sequence)
│   ├── 序列构建器 (Sequence Builder)
│   ├── 序列执行器 (Sequence Executor)
│   └── 序列状态追踪器 (Sequence State Tracker)
└── 资源冲突检测模块 (Conflict Detection)
    ├── SHM冲突检测器 (SHM Conflict Detector)
    ├── 算力冲突检测器 (Compute Conflict Detector)
    ├── 冲突报告生成器 (Conflict Report Generator)
    └── 用户决策收集器 (User Decision Collector)
```

### 3.2 数据流

```
用户创建任务 + 静态配置
        │
        ▼
┌───────────────────┐
│  子任务链编排      │
│  (DAG构建/验证)   │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  静态配置绑定      │
│  (个性数据/参数)   │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐     ┌───────────────────┐
│  配置序列构建      │────▶│  资源冲突检测      │
│  (批量配置列表)    │     │  (SHM/算力/依赖)   │
└────────┬──────────┘     └────────┬──────────┘
         │                         │
         │                  ┌──────┴──────┐
         │                  │ 无冲突      │ 有冲突
         │                  ▼             ▼
         │            ┌──────────┐  ┌──────────────┐
         │            │ 直接执行  │  │ 冲突决策报告  │
         │            └──────────┘  │ → 用户裁决    │
         │                          └──────┬───────┘
         │                                 │
         ▼                                 ▼
┌──────────────────────────────────────────────┐
│  序列执行器 (逐任务提交至BP-0045任务管理Worker) │
└──────────────────────────────────────────────┘
```

---

## 四、函数级设计

### 4.1 任务生成模块

#### 函数1：用户任务创建

```python
def user_task_create(
    ctx: dict,
    task_spec: dict,              # 任务规格
    static_config: dict,          # 静态配置
    consumer_id: str
) -> dict:
    """
    用户创建任务（含静态配置）
    
    task_spec结构:
    {
        "task_label": "task_colle/task_a/task_sa",
        "function_ref": str,
        "subtask_chain": {           # 子任务链（可选）
            "nodes": [
                {"id": "sub_1", "function_ref": str, "config_override": dict},
                {"id": "sub_2", "function_ref": str, "depends_on": ["sub_1"]}
            ],
            "edges": [
                {"from": "sub_1", "to": "sub_2", "type": "serial"}
            ]
        },
        "exception_policy": {       # 交易链异常策略
            "timeout_ms": int,
            "retry_count": int,
            "retry_backoff_ms": int,
            "on_failure": "abort" | "skip" | "fallback",
            "fallback_config": dict | None
        }
    }
    
    static_config结构:
    {
        "personal_data": dict,      # 个性数据（用户自定义）
        "trade_params": dict,       # 交易参数
        "shm_paths": {              # SHM路径声明
            "input": str,
            "output": str
        },
        "compute_requirement": {    # 算力需求声明
            "cpu_cores": int,
            "memory_mb": int,
            "time_budget_ms": int
        }
    }
    
    返回值 dict:
    {
        "status": "created" | "invalid",
        "task_id": str,
        "task_label": str,
        "subtask_count": int,
        "config_hash": str,
        "error": str | None
    }
    """
```

#### 函数2：子任务链验证

```python
def subtask_chain_validate(
    ctx: dict,
    chain_spec: dict               # 子任务链规格
) -> dict:
    """
    验证子任务链的DAG合法性
    
    检查项:
    - 无环检测（拓扑排序）
    - 孤立节点检测
    - 依赖完整性
    - 配置继承一致性
    
    返回值 dict:
    {
        "status": "valid" | "invalid",
        "execution_order": list,    # 拓扑排序后的执行顺序
        "parallel_groups": list,    # 可并行执行的分组
        "issues": list,             # 发现的问题
        "error": str | None
    }
    """
```

#### 函数3：交易链异常策略配置

```python
def exception_policy_configure(
    ctx: dict,
    task_id: str,
    policy: dict                   # 异常策略
) -> dict:
    """
    配置交易链异常处理策略
    
    policy结构:
    {
        "timeout_ms": int,           # 单步超时
        "total_timeout_ms": int,     # 链总超时
        "retry_count": int,          # 重试次数
        "retry_backoff_ms": int,     # 退避间隔
        "on_failure": "abort" | "skip" | "fallback" | "retry",
        "fallback_task_id": str | None,
        "alert_on_failure": bool
    }
    
    返回值 dict:
    {
        "status": "configured" | "invalid",
        "task_id": str,
        "policy_hash": str,
        "error": str | None
    }
    """
```

### 4.2 静态配置模块

#### 函数4：静态配置定义

```python
def static_config_define(
    ctx: dict,
    task_id: str,
    config: dict                   # 静态配置
) -> dict:
    """
    为任务定义静态配置
    
    返回值 dict:
    {
        "status": "defined" | "invalid",
        "task_id": str,
        "config_hash": str,
        "config_size_bytes": int,
        "error": str | None
    }
    """
```

#### 函数5：静态配置校验

```python
def static_config_validate(
    ctx: dict,
    task_id: str,
    config: dict
) -> dict:
    """
    校验静态配置的完整性和合法性
    
    校验项:
    - 必填字段完整性
    - 数据类型匹配
    - SHM路径格式
    - 算力需求合理性
    - 与任务规格的兼容性
    
    返回值 dict:
    {
        "status": "valid" | "invalid",
        "validation_results": [
            {"field": str, "valid": bool, "message": str}
        ],
        "error": str | None
    }
    """
```

#### 函数6：配置模板保存/加载

```python
def config_template_manage(
    ctx: dict,
    action: str,                   # "save" | "load" | "list" | "delete"
    template_id: str,
    template_data: dict | None
) -> dict:
    """
    配置模板管理
    
    返回值 dict:
    {
        "status": "saved" | "loaded" | "listed" | "deleted" | "error",
        "template_id": str,
        "template_data": dict | None,
        "available_templates": list | None,
        "error": str | None
    }
    """
```

### 4.3 配置序列模块

#### 函数7：配置序列构建

```python
def config_sequence_build(
    ctx: dict,
    sequence_name: str,
    task_configs: list             # 任务配置列表
) -> dict:
    """
    构建静态配置序列
    
    task_configs中每项:
    {
        "task_spec": dict,
        "static_config": dict,
        "execution_order": int,    # 执行顺序（可选，默认按列表顺序）
        "enabled": bool,           # 是否启用
        "condition": dict | None   # 前置条件（可选）
    }
    
    返回值 dict:
    {
        "status": "built" | "build_failed",
        "sequence_id": str,
        "sequence_name": str,
        "task_count": int,
        "enabled_count": int,
        "error": str | None
    }
    """
```

#### 函数8：配置序列执行

```python
def config_sequence_execute(
    ctx: dict,
    sequence_id: str,
    execution_params: dict         # 执行参数
) -> dict:
    """
    执行配置序列
    
    execution_params:
    {
        "mode": "sequential" | "parallel" | "condition_based",
        "stop_on_failure": bool,
        "max_parallel": int,
        "conflict_resolution": "auto" | "manual"
    }
    
    返回值 dict:
    {
        "status": "submitted" | "conflict_detected" | "execution_failed",
        "sequence_id": str,
        "submitted_tasks": list,
        "conflict_report": dict | None,
        "error": str | None
    }
    """
```

#### 函数9：序列状态查询

```python
def config_sequence_status(
    ctx: dict,
    sequence_id: str
) -> dict:
    """
    查询配置序列执行状态
    
    返回值 dict:
    {
        "status": "success",
        "sequence_id": str,
        "sequence_state": str,     # PENDING/RUNNING/COMPLETED/FAILED/PAUSED
        "tasks": [
            {
                "task_id": str,
                "execution_order": int,
                "state": str,
                "result_summary": dict | None
            }
        ],
        "progress": float,         # 0.0-1.0
        "error": str | None
    }
    """
```

### 4.4 资源冲突检测模块

#### 函数10：资源冲突检测

```python
def resource_conflict_detect(
    ctx: dict,
    task_configs: list             # 待检测的任务配置列表
) -> dict:
    """
    检测多任务间的资源冲突
    
    检测维度:
    - SHM路径冲突: 多任务读写同一SHM路径
    - 算力资源冲突: 总算力需求超过可用资源
    - 依赖死锁: 子任务链之间的循环依赖
    - 时间窗口冲突: 定时任务时间重叠
    - 标签冲突: 同一task_label被重复定义
    
    返回值 dict:
    {
        "status": "clean" | "conflict_detected",
        "conflict_count": int,
        "conflicts": [
            {
                "conflict_id": str,
                "type": "shm_overlap" | "compute_exceed" | "deadlock" |
                       "time_overlap" | "label_duplicate",
                "severity": "critical" | "warning" | "info",
                "involved_tasks": [str],
                "description": str,
                "suggestion": str
            }
        ],
        "resource_summary": {
            "total_shm_paths": int,
            "unique_shm_paths": int,
            "overlapping_shm_paths": int,
            "total_compute_cores": int,
            "available_cores": int
        },
        "error": str | None
    }
    """
```

#### 函数11：冲突决策报告生成

```python
def conflict_report_generate(
    ctx: dict,
    conflicts: list,               # 冲突列表
    task_configs: list             # 涉及的任务配置
) -> dict:
    """
    生成冲突决策报告
    
    报告结构:
    {
        "report_id": str,
        "generated_at": int,
        "summary": str,
        "conflict_details": list,
        "decision_options": [
            {
                "option_id": str,
                "description": str,
                "action": str,
                "affected_tasks": list,
                "risk_assessment": str
            }
        ],
        "recommendation": str
    }
    
    返回值 dict:
    {
        "status": "generated",
        "report": dict,
        "error": str | None
    }
    """
```

#### 函数12：用户决策收集

```python
def user_decision_collect(
    ctx: dict,
    report_id: str,
    decision: dict                 # 用户决策
) -> dict:
    """
    收集用户对冲突的决策
    
    decision结构:
    {
        "report_id": str,
        "resolutions": [
            {
                "conflict_id": str,
                "selected_option": str,
                "user_note": str
            }
        ],
        "decided_at": int
    }
    
    返回值 dict:
    {
        "status": "applied" | "partial" | "rejected",
        "report_id": str,
        "resolved_conflicts": int,
        "unresolved_conflicts": int,
        "adjusted_configs": list,
        "error": str | None
    }
    """
```

---

## 五、数据结构设计

### 5.1 子任务链DAG

```python
SUBTASK_CHAIN = {
    "chain_id": str,
    "task_id": str,
    "nodes": [
        {
            "id": "sub_1",
            "function_ref": str,
            "config_override": dict,
            "timeout_ms": int,
            "retry_policy": dict
        }
    ],
    "edges": [
        {"from": "sub_1", "to": "sub_2", "type": "serial" | "parallel" | "conditional"},
        {"from": "sub_1", "to": "sub_3", "type": "conditional", "condition": dict}
    ],
    "exception_policy": {
        "timeout_ms": int,
        "retry_count": int,
        "on_failure": "abort" | "skip" | "fallback",
        "fallback_node": str | None
    },
    "execution_plan": {
        "topological_order": list,
        "parallel_groups": list,
        "critical_path_ms": int
    }
}
```

### 5.2 静态配置

```python
STATIC_CONFIG = {
    "config_id": str,
    "task_id": str,
    "version": int,
    "personal_data": dict,
    "trade_params": {
        "demand_mask": int,
        "latency_class": int,
        "path_level": str,
        "time_budget_ms": int,
        "custom_params": dict
    },
    "shm_declaration": {
        "input_paths": [str],
        "output_paths": [str],
        "read_only": bool
    },
    "compute_declaration": {
        "cpu_cores": int,
        "memory_mb": int,
        "shm_budget_mb": int,
        "gpu_required": bool
    },
    "config_hash": str,
    "created_at": int,
    "source": "manual" | "template" | "inherited"
}
```

### 5.3 配置序列

```python
CONFIG_SEQUENCE = {
    "sequence_id": str,
    "sequence_name": str,
    "consumer_id": str,
    "state": "DRAFT" | "READY" | "RUNNING" | "COMPLETED" | "FAILED" | "PAUSED",
    "items": [
        {
            "item_id": str,
            "execution_order": int,
            "task_spec": dict,
            "static_config": dict,
            "enabled": bool,
            "condition": dict | None,
            "state": "PENDING" | "SUBMITTED" | "RUNNING" | "COMPLETED" | "FAILED" | "SKIPPED",
            "task_id": str | None,
            "submitted_at": int | None,
            "completed_at": int | None,
            "result_summary": dict | None
        }
    ],
    "execution_mode": "sequential" | "parallel" | "condition_based",
    "conflict_report": dict | None,
    "user_decisions": dict | None,
    "progress": float,
    "created_at": int,
    "started_at": int | None,
    "completed_at": int | None
}
```

### 5.4 冲突报告

```python
CONFLICT_REPORT = {
    "report_id": str,
    "sequence_id": str,
    "generated_at": int,
    "status": "pending" | "resolved" | "expired",
    "conflicts": [
        {
            "conflict_id": str,
            "type": str,
            "severity": str,
            "involved_tasks": [str],
            "description": str,
            "suggestion": str,
            "resolution": str | None
        }
    ],
    "decision_options": [
        {
            "option_id": str,
            "description": str,
            "action": "serialize" | "prioritize" | "skip" | "cancel",
            "affected_tasks": list,
            "risk": str
        }
    ],
    "user_decision": dict | None,
    "resolved_at": int | None
}
```

---

## 六、资源冲突检测详解

### 6.1 SHM路径冲突

```
任务A: input=/shm/sensors/temp, output=/shm/results/avg
任务B: input=/shm/sensors/temp, output=/shm/results/max
                    ↑
              SHM读冲突（同一路径两个消费者读取）

任务C: output=/shm/results/avg
                    ↑
              SHM写冲突（任务A和任务C写入同一路径）
```

检测规则：
- **读-读**: 允许（多个消费者可读同一SHM路径）
- **写-写**: 冲突（同一SHM路径只能有一个写入者）
- **读-写**: 冲突（写入时不能有其他读写者）

### 6.2 算力资源冲突

```
可用资源: cpu=8核, memory=32GB

任务A: cpu=4核, memory=16GB
任务B: cpu=6核, memory=8GB
任务C: cpu=2核, memory=4GB

A+B = 10核 > 8核  → 算力冲突
A+C = 6核 ≤ 8核   → 无冲突
B+C = 8核 ≤ 8核   → 无冲突（但内存=12GB ≤ 32GB）
```

### 6.3 冲突严重等级

| 等级 | 说明 | 处理 |
|------|------|------|
| **critical** | 写-写SHM冲突、算力严重超限 | 必须用户裁决 |
| **warning** | 读-写SHM冲突、算力接近上限 | 建议用户裁决 |
| **info** | 读-读SHM共享、标签重复 | 自动处理，通知用户 |

### 6.4 冲突解决策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **serialize** | 串行执行冲突任务 | SHM写冲突 |
| **prioritize** | 按优先级排序，低优先级等待 | 算力不足 |
| **skip** | 跳过低优先级任务 | 非关键任务冲突 |
| **cancel** | 取消冲突任务 | 无法调和的冲突 |
| **partition** | 分时段执行 | 时间窗口冲突 |

---

## 七、与现有体系的关系

### 7.1 与BP-0045任务管理Worker

| BP-0045概念 | 本蓝图对应 |
|-------------|-----------|
| trader_intent_receive | user_task_create → 生成意图后提交给交易员 |
| trader_order_create | 配置序列执行 → 每个任务配置转化为订单 |
| task_label | 任务标签直接传递给BP-0045的task_label机制 |
| supervisor_trade_audit | 冲突检测作为审计的前置检查 |

### 7.2 与BP-0028参数化任务

| BP-0028概念 | 本蓝图对应 |
|-------------|-----------|
| execution_mode.repeat | 配置序列的循环执行模式 |
| execution_mode.scheduled | 配置序列的定时执行模式 |
| execution_mode.conditional | 配置序列的条件执行模式 |

### 7.3 与BP-0048任务构想

| BP-0048概念 | 本蓝图对应 |
|-------------|-----------|
| task_conception_generate_contract | 静态配置 → 契约生成 |
| task_conception_estimate_cost | 算力声明 → 代价预估 |

---

## 八、测试场景

### 8.1 任务生成测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 创建简单任务 | task_spec + config | created | task_id非空 |
| 创建含子任务链的任务 | 3节点DAG | created | subtask_count=3 |
| 无效DAG（有环） | 循环依赖 | invalid | issues包含环检测 |
| 孤立节点 | 无依赖的节点 | valid | execution_order包含孤立节点 |
| 异常策略配置 | fallback策略 | configured | policy_hash非空 |

### 8.2 静态配置测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 定义完整配置 | 所有必填字段 | defined | config_hash非空 |
| 缺少必填字段 | 不完整配置 | invalid | validation_results有错误 |
| 模板保存/加载 | 保存后加载 | saved→loaded | 数据一致 |
| 配置继承 | 父→子继承 | valid | 子配置包含父配置字段 |

### 8.3 配置序列测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 构建序列 | 5个任务配置 | built | task_count=5 |
| 顺序执行 | sequential模式 | submitted | submitted_tasks=5 |
| 并行执行 | parallel模式 | submitted | submitted_tasks=5 |
| 禁用某任务 | enabled=false | built | enabled_count=4 |
| 条件执行 | condition_based | submitted | 按条件跳过 |

### 8.4 资源冲突检测测试

| 场景 | 输入 | 预期 | 通过条件 |
|------|------|------|----------|
| 无冲突 | 独立SHM路径 | clean | conflict_count=0 |
| SHM写冲突 | 同一路径两个写入者 | conflict_detected | type=shm_overlap |
| 算力超限 | 总cpu>可用 | conflict_detected | type=compute_exceed |
| 标签重复 | 同一task_label | conflict_detected | type=label_duplicate |
| 冲突报告生成 | 有冲突 | generated | decision_options非空 |
| 用户决策 | 选择serialize | applied | resolved_conflicts>0 |

---

## 九、安全约束

```python
SYSTEM_LIMITS = {
    "max_subtasks_per_chain": 100,
    "max_sequence_length": 10000,
    "max_config_size_bytes": 1048576,
    "max_templates_per_user": 100,
    "max_parallel_detection": 1000,
    "conflict_report_ttl_ms": 3600000,
    "max_personal_data_fields": 50,
    "max_custom_params": 20
}
```

---

## 十、授权文档申请

```
蓝图编号: BP-0053
蓝图名称: User Task Generation & Static Config
申请事项: 请求商场授权消费者层任务生成与静态配置能力

依赖蓝图:
  - BP-0045-task-management-worker (任务管理Worker)
  - BP-0028-task-parameterization (参数化任务)
  - BP-0048-task-conception (任务构想)

授权范围:
  - 任务生成: 支持
  - 子任务链编排: 支持
  - 静态配置: 支持
  - 配置序列: 支持
  - 资源冲突检测: 支持
  - 用户决策收集: 支持

安全约束:
  - 冲突检测必须执行，禁止跳过
  - 写-写SHM冲突必须用户裁决
  - 配置序列最大长度: 10000
  - 子任务链最大节点: 100

签名: [创世Worker]
```

---

## 十一、附录

### 11.1 术语定义

| 术语 | 定义 |
|------|------|
| 任务静态配置 | 声明式的任务参数绑定，不直接触发执行 |
| 配置序列 | 有序的任务配置列表，支持批量执行 |
| 子任务链 | DAG结构的子任务依赖关系 |
| 资源冲突 | 多任务竞争同一资源（SHM/算力/时间窗口） |
| 冲突决策报告 | 向用户展示冲突详情和解决选项的报告 |

### 11.2 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v1.0.0 | 2026-05-13 | 初始版本 |

---

**审核状态**: 待审核  
**下一步动作**: 提交AICoder进行七维审查与代码生成
