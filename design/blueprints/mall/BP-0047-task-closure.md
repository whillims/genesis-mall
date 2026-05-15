# 蓝图：任务完结与归档闭环

> **蓝图编号**: BP-0047  
> **蓝图名称**: task-closure  
> **版本**: v1.0.0  
> **提交时间**: 2026-05-12  
> **提交者**: 创世Worker  
> **目标语言**: Python  
> **功能域**: mall  
> **依赖**: BP-0023-shm-manager, BP-0024-worker-spawn, BP-0025-intent-router, BP-0026-error-guard, BP-0030-task-publisher, BP-0036-mall-monitor

---

## 一、蓝图概述

`task-closure` 定义商场域任务完成后的闭环工具族，覆盖成果验收、归档打包、生产者评价、蓝图进化建议与决策权归属。

核心目标：
- 让任务结束结果可验证、可追溯、可归档
- 让消费者拥有验收与归档控制权
- 让商场生成闭环进化建议，辅助下一次蓝图迭代
- 让生产者信誉与奖励体系与任务完结挂钩

---

## 二、设计目标

### 2.1 功能目标

- 提供“成果验收面板”，支持消费者对任务输出进行完全接受、部分接受、完全驳回
- 提供“数据归档打包器”，将任务全过程打包为可验证的归档包
- 提供“生产者评价器”，记录消费者对生产者的信誉反馈
- 提供“蓝图进化建议器”，基于任务数据建议下次蓝图优化
- 定义“决策权归属”，明确消费者、商场、AICoder三方角色

### 2.2 约束条件

- 设计遵循项目函数范式：零全局、零堆、零系统调用
- 归档数据只读不可变，采用 SHA-256 完整性校验
- 验收操作必须由消费者主动触发，禁止自动化“默认验收”
- 建议器输出仅供参考，不越过消费者的最终采纳权

---

## 三、核心模块

### 3.1 成果验收面板

**职责**：提供消费者对任务成果的审核入口。

功能要点：
- 展示结果摘要与关键指标
- 支持“完全接受”、“部分接受”、“完全驳回”
- 强制填写评价理由
- 记录验收哈希与审计事件

### 3.2 数据归档打包器

**职责**：将任务产出、日志、契约、蓝图、审计记录打包成可迁移归档包。

归档结构：
```
task_archive_{contract_id}.tar
├── manifest.json
├── data/
│   ├── raw/
│   ├── processed/
│   └── metadata/
├── logs/
│   ├── audit_log/
│   ├── control_log/
│   └── alarm_log/
├── contracts/
│   ├── original_contract.md
│   └── signed_contract.json
├── blueprints/
│   └── consumer_blueprint.json
└── signatures/
    ├── consumer_approval.sig
    ├── mall_approval.sig
    └── data_integrity.sig
```

### 3.3 生产者评价器

**职责**：让消费者对生产者进行多维评价，并把结果写入信誉系统。

评价维度：稳定性、准确性、时效性、协作性、文档质量。

### 3.4 蓝图进化建议器

**职责**：基于任务执行数据、异常日志、验收结果，生成下次蓝图优化建议。

建议类型：参数优化、流程优化、风险预警、生产者推荐。

### 3.5 决策权归属

| 决策点 | 决策主体 | 说明 |
|--------|----------|------|
| 成果验收 | 消费者 | 依赖验收面板提供的数据证据 |
| 归档确认 | 消费者 + 商场 | 归档包打包并签名 |
| 生产者评价 | 消费者 | 评价后不可修改，仅可补充 |
| 蓝图进化 | 消费者 | 建议器输出供采纳，不可自动生效 |

---

## 四、接口定义

### 4.1 完成流程

- `task_closure_init(root, contract_id, task_context)`
- `task_closure_acceptance_panel(root, task_id, consumer_id, acceptance_result, reason)`
- `task_closure_archive_package(root, task_id, archive_path)`
- `task_closure_evolution_suggest(root, task_id)`

### 4.2 归档摘要

归档包必须包含 `manifest.json` 和每个子文件的 SHA-256 哈希，顶层签名由商场与消费者共同生成。

---

## 五、验收标准

- 验收面板能够展示任务产出摘要、关键指标、异常标记和原始数据预览
- 消费者能够执行“完全接受/部分接受/完全驳回”并生成审计记录
- 归档打包器能生成可解压、校验通过、签名完整的归档包
- 生产者评价结果能写入信誉系统且不可篡改
- 蓝图进化建议器能基于历史数据生成明确优化建议

---

## 六、交付物

- `design/blueprints/mall/BP-0047-task-closure.md`
- `AUTH-BP-0047-TASK-CLOSURE.json`（后续授权文件）
- 对应测试文件：`tests/test_task_closure.py`
- 更新后的 `PROJECT_CONTEXT.md` 和 `design/blueprints/blueprint-index.md`
