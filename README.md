# 创世纪 (Genesis)

> **微型商场创世纪工程** — 一个基于函数式范式、蓝图驱动、授权执行的分布式任务调度与数据交换系统

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 许可证

本项目采用 [MIT 许可证](LICENSE) 开源。

```
MIT License

Copyright (c) 2026 创世纪项目团队 (Genesis Project Team)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
```

---

## 项目愿景

创世纪是一个实验性的分布式系统框架，探索**代码生成自治**的边界。核心思想：让系统自身成为代码的生产者，人类仅提供设计意图（蓝图），由外置AICoder完成代码精化、审查与授权，最终由商场本体安全加载执行。

## 核心设计理念

| 理念 | 说明 |
|------|------|
| **函数式范式** | 零全局状态、零堆分配、零系统调用 —— 代码即数学 |
| **蓝图驱动** | 所有功能通过标准化蓝图设计，经七维审查后生成授权代码 |
| **三界隔离** | 生产者域、AICoder域、商场域严格分离，互不越界 |
| **SHM矢量空间** | 基于共享内存的零拷贝数据交换，无网络数据库依赖 |
| **授权执行** | 所有代码必须通过授权文件七步验证后方可加载运行 |

## 核心概念模型

本项目围绕**用户-任务-决策**构建完整的分布式计算概念体系：

### 用户 (User)

> **用户是人，系统的操作主体**，与"消费者"概念严格区分。

| 属性 | 说明 |
|------|------|
| 身份 | 自然人或外部系统 |
| 角色 | 可扮演生产者（提交蓝图）或消费者（执行任务） |
| 主页 | 用户拥有个人主页，展示其控制的多个消费者"相" |
| 意图 | 通过任务编辑器定义高层目标，由系统分解为可执行序列 |

### 消费者 (Consumer)

> **消费者是Worker的类型标志，执行实体**，不是人。

- 通过消费者蓝图设计生成
- 具有类型分级（普通消费者、信号处理型消费者等）
- 拥有"相"（外观/界面），在用户主页以卡片形式展示
- 通过SHM矢量空间与商场交互

### 任务 (Task)

> **用户定义的高层目标**，包含多个子任务和交易序列。

| 概念 | 定义 |
|------|------|
| **任务** | 用户定义的高层目标，包含多个子任务 |
| **子任务** | 任务的可执行单元，对应一个交易序列 |
| **交易** | 执行单元向商场提交产品的过程（Worker→商场 或 消费者→商场） |
| **产品** | 交易中提交的数据实体，存储在商场中，可被订阅 |
| **条件控制** | 用户设定的执行条件，检测交易状态决定流程走向 |

**交易模型核心规则：**
1. 提交方只向商场提交产品，不直接发送给接收方
2. 订阅方从商场获取产品
3. 发送者不关心接收者是否收到（解耦）
4. 用户通过条件控制交易序列执行

### 任务群 (Task Group)

> **多个关联任务的集合**，支持批量调度与协同执行。

- 任务群内的任务共享资源配额
- 支持DAG依赖关系定义
- 通过批量同步监督Worker实现屏障同步

### 决策 (Decision)

> **基于实时数据与历史模式的智能建议生成**。

| 组件 | 职责 |
|------|------|
| **决策引擎** | 分析当前状态，生成行动建议 |
| **历史模式挖掘** | 从过往任务执行中提取模式 |
| **需求匹配** | 将消费者需求与生产者能力匹配 |
| **五维评分** | 对生产者进行动态评分（可靠性/效率/质量/安全/创新） |
| **报告生成** | 输出标准化决策报告 |

**决策流程：**
```
数据采集 → 模式挖掘 → 需求匹配 → 评分排序 → 建议生成 → 报告输出
```

## 架构概览

```
浏览器/客户端
    |
    v
[Nginx 钢铁脊椎]  ← 统一入口、安全过滤、SSL终端
    |
    v
[FastAPI 通讯层]  ← HTTP API、Queue转发
    |
    v
[商场本体]        ← 矢量交易引擎、函数注册与调度、状态机
    |
    v
[Workers]         ← 生产者 / 消费者 / 监督者
```

## 文档体系

本项目公开全部**设计文档与蓝图**，源码仅通过授权机制内部流转。

| 文档类别 | 路径 | 说明 |
|----------|------|------|
| 项目总览 | [design/genesis/](design/genesis/) | 创世纪七阶段演进文档 |
| 规范宪法 | [design/specs/constitution/](design/specs/constitution/) | 函数范式、SHM宪法、蓝图标准、异常处理等21项核心规范 |
| 协议规范 | [design/specs/protocol/](design/specs/protocol/) | 授权文件规范、AICoder协议、架构文档A-E |
| 工作流程 | [design/specs/workflow/](design/specs/workflow/) | 授权接受流程、蓝图提交流程、IO调试协议 |
| 功能蓝图 | [design/blueprints/](design/blueprints/) | 66+ 蓝图，按 producer/consumer/mall/shm/scheduler/audit/security 分类 |
| 蓝图索引 | [design/blueprints/blueprint-index.md](design/blueprints/blueprint-index.md) | 完整蓝图目录与状态 |

## 七阶段演进

| 阶段 | 主题 | 状态 |
|------|------|------|
| 阶段一 | 商场本体创世 | 已完成 |
| 阶段二 | 授权文件接入协议 | 已完成 |
| 阶段三 | 第一张蓝图落地 | 已完成 |
| 阶段四 | 端到端闭环验证 | 已完成 |
| 阶段五 | 多实例与调度进化 | 已完成 |
| 阶段六 | 开门营业转换 | 已完成 |
| 阶段七 | 商场自治铁律生效 | 待实现 |

## 蓝图示例

本项目已积累 **66+ 功能蓝图**，涵盖：

- **生产者**: 键盘输入、网络遥测、仿真信号、系统信息采集、RTL-SDR频谱等
- **消费者**: Console显示、频谱仪UI、数据可视化、微波TR测试报告、启动动画等
- **商场核心**: 注册表、心跳、消息路由、配置管理、任务管理、IO管理、决策引擎等
- **SHM基础设施**: 矢量管理、双通道架构、生命周期管理等

详见 [design/blueprints/blueprint-index.md](design/blueprints/blueprint-index.md)

## 设计哲学

> **抛砖引玉** — 本仓库仅公开设计文档与架构蓝图，旨在：
> 1. 分享分布式系统设计的范式思考
> 2. 探讨AI辅助代码生成的安全边界
> 3. 为同类项目提供参考框架
>
> 具体实现代码通过授权机制内部流转，不在此仓库提供。

## 为什么不公开源码？

本项目的核心创新在于：**设计文档即源码**。

所有功能蓝图均遵循严格的规范格式（详见 [blueprint-standard.md](design/specs/constitution/blueprint-standard.md)），任何具备AICoder能力的系统均可根据这些设计规则文件**自动生成可执行代码**。

**代码生成流程：**
```
设计蓝图 (BP-xxxx.md)
    |
    v
[外置AICoder] —— 七维审查(D1~D7) → 代码精化 → 授权生成
    |
    v
授权文件 (AUTH-BP-xxxx.json) —— 七步验证 → 安全加载
    |
    v
可执行代码 (PYC编译产物) —— 商场本体运行
```

因此，**设计文档本身就是最高级别的源代码**。公开蓝图即公开了系统的全部设计意图与行为契约，而具体实现可由AICoder根据规范自动推导生成。这也是本项目探索"代码生成自治"的核心实践。

## 如何基于蓝图生成代码

1. **阅读蓝图**: 选择 [design/blueprints/](design/blueprints/) 中的功能蓝图
2. **理解规范**: 遵循 [function-paradigm.md](design/specs/constitution/function-paradigm.md) 的函数式铁律
3. **执行审查**: 按 [auth-acceptance.md](design/specs/workflow/auth-acceptance.md) 的七步验证流程
4. **生成授权**: 使用 [authorization-generation.md](design/specs/constitution/authorization-generation.md) 的5字段结构
5. **安全加载**: 通过商场本体的授权验证机制运行

> 注：本项目已验证1200+测试用例，证明蓝图到代码的自动化生成链路完全可行。

## 参与方式

- **阅读蓝图**: 从 [design/blueprints/blueprint-index.md](design/blueprints/blueprint-index.md) 开始
- **理解规范**: 阅读 [design/specs/constitution/function-paradigm.md](design/specs/constitution/function-paradigm.md) 了解函数范式铁律
- **讨论设计**: 通过 GitHub Issues 讨论架构与蓝图设计
- **提交蓝图**: 参考 [design/specs/constitution/blueprint-standard.md](design/specs/constitution/blueprint-standard.md) 编写符合规范的蓝图

---

## 免责声明

> **本项目纯属头脑风暴与思想实验性质**，旨在探索AI辅助代码生成与分布式系统设计的理论边界。
>
> **不针对任何机构、组织或个人**，所有概念、术语、场景均为技术探讨而设，与现实中的实体无关。
>
> 欢迎基于学术兴趣参与讨论，共同推进相关领域的前沿思考。

---

*创世纪项目 © 2026 | 开源于 MIT 许可证 | 设计文档公开，实现代码授权保护*
