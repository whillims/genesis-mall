# 蓝图：任务发起与批准工具族

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **蓝图编号**: BP-0048  
> **蓝图名称**: task-conception  
> **版本**: v1.0.0  
> **提交时间**: 2026-05-12  
> **提交者**: 创世Worker  
> **目标语言**: Python  
> **功能域**: mall  
> **依赖**: BP-0023-shm-manager, BP-0024-worker-spawn, BP-0025-intent-router, BP-0026-error-guard, BP-0030-task-publisher, BP-0036-mall-monitor

---

## 一、蓝图概述

`task-conception` 定义商场域任务发起与契约批准工具族，覆盖消费者任务蓝图编辑、生产者匹配、代价评估、契约生成、双向确认与沙箱预演。

核心目标：
- 将消费者需求转化为可执行、可验证的任务契约
- 让消费者保有发起、修改、批准的主体权
- 让商场对契约合法性与资源可行性负责
- 提供沙箱预演与风险提示，避免低质量任务进入正式执行

---

## 二、设计目标

### 2.1 功能目标

- 提供结构化任务蓝图初始化接口
- 提供生产者匹配探针接口，按能力、信誉、负载与安全过滤候选生产者
- 提供代价评估器接口，预测时间、资源与风险等级
- 提供契约生成器接口，将蓝图与生产者选型转化为可批准契约
- 提供双向批准接口，记录消费者与商场签名并锁定契约
- 提供沙箱预演接口，验证执行序列与兼容性风险

### 2.2 约束条件

- 遵循函数范式：零全局、零堆、零系统调用，状态外置
- 消费者主导契约内容，商场主导技术可行性
- 签名与批准记录需永久写入审计日志
- 沙箱预演仅提供判断结果，不直接执行异常任务

---

## 三、核心模块

### 3.1 任务蓝图编辑器

**职责**：接收并验证消费者提供的结构化任务蓝图。

输入内容：
- `input_definition`：数据来源、格式、触发条件
- `output_definition`：期望产物、精度、交付格式
- `constraints`：最大时长、最大重试、安全等级
- `producer_preferences`：推荐/排除能力标签

### 3.2 生产者匹配探针

**职责**：基于可用生产者池生成候选列表。

匹配维度：
- 能力标签
- 历史成功率
- 当前负载
- 信誉评分
- 安全等级

### 3.3 代价估算器

**职责**：输出乐观、最可能、悲观时间估算，并给出风险等级与置信度。

### 3.4 契约生成器

**职责**：将消费者蓝图与生产者选型转化为可批准契约。

契约结构应包含：
- `contract_id`
- `consumer_blueprint_hash`
- `producers`
- `execution_plan`
- `constraints`
- `approval_required`
- `signature`

### 3.5 双向批准签名器

**职责**：记录消费者与商场的批准签名，并在签名齐备后将契约置为`approved`。

### 3.6 沙箱预演器

**职责**：在非密环境下模拟契约可行性，检测执行序列、格式兼容性、负载风险与生产者死锁。

---

## 四、接口定义

- `task_conception_init(ctx, root, consumer_id, blueprint)`
- `task_conception_match_producers(ctx, root, required_tags=None)`
- `task_conception_estimate_cost(ctx, root, selected_producer_ids=None)`
- `task_conception_generate_contract(ctx, root, selected_producer_ids, constraints=None, execution_plan=None)`
- `task_conception_ratify_contract(ctx, root, contract_id, signer_type, signer_id)`
- `task_conception_sandbox_preview(ctx, root, contract_id)`

---

## 五、验收标准

- 能对结构化蓝图字段进行必填校验
- 能根据生产者池生成可排序候选列表
- 能输出成本估算与风险等级
- 能生成并签名契约结构
- 能完成消费者与商场双向批准
- 能返回沙箱预演结果与问题提示

---

## 六、交付物

- `design/blueprints/mall/BP-0048-task-conception.md`
- `src/workers/task_conception.py`
- `tests/test_task_conception.py`
