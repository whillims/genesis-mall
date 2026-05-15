# 蓝图文档（定稿版）：BP-0031 -- 用户任务编辑器

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0031-TASK-EDITOR |
| 蓝图名称 | 用户任务编辑器 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-12 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-2 |
| 设计范式 | 纯函数级，禁止OOP封装 |

## 2. 功能概述

### 2.1 功能描述

用户任务编辑器是用户意图到消费者执行的"编译器"。用户通过编辑器定义任务（包含子任务），编辑器将任务编译为交易序列——即Worker/消费者通过商场进行产品提交和订阅的有序操作链。用户可设定条件控制交易序列的执行流程。

### 2.2 核心概念

**任务 (Task)**: 用户定义的高层目标，包含多个子任务。

**子任务 (SubTask)**: 任务的可执行单元，对应一个交易序列。

**交易 (Transaction)**: 一个执行单元（Worker或消费者）向商场提交产品的过程。其他执行单元通过订阅获取产品。发送者不关心接收者是否收到。

**条件控制 (Condition)**: 用户设定的执行条件。检测交易状态/产品内容，决定交易序列的继续/等待/终止/分支。

**产品 (Product)**: 交易中提交的数据实体，存储在商场中，可被订阅。

### 2.3 交易模型

```
Worker/消费者 ──产品──→ 商场(发布点) ──订阅──→ Worker/消费者

规则:
  1. 提交方只向商场提交产品，不直接发送给接收方
  2. 订阅方从商场获取产品
  3. 发送者不关心接收者是否收到（解耦）
  4. 用户通过条件控制交易序列执行
```

### 2.4 数据来源

- `shm_vector` 用户配置矢量（user_config域）
- `shm_vector` 消费者/Worker注册表（registry域）
- `ctx` 用户编辑指令

### 2.5 数据去向

- `stdout` 编辑器界面渲染
- `shm_vector` 任务定义矢量（task_def域）
- `shm_vector` 交易序列矢量（tx_sequence域）

### 2.6 来源锁定

- 用户ID锁定：任务归属必须为已认证用户
- 注册表锁定：消费者/Worker列表必须从registry域读取

## 3. 函数设计

### 3.1 主函数签名

```python
def task_editor_compile(
    ctx: dict,
    user_id: str,
    task_definition: dict
) -> dict:
    """
    [复杂度]: O(n*m), n为子任务数, m为平均交易数
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 编辑器渲染函数签名

```python
def task_editor_render(
    ctx: dict,
    user_id: str,
    task_definition: dict = None
) -> dict:
    """
    [复杂度]: O(n*m)
    [安全等级]: READ_ONLY_SHM
    """
```

### 3.3 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | `dict` | 是 | 非空 | 上下文对象，含SHM访问器 |
| `user_id` | `str` | 是 | 长度 <= 64 | 用户唯一标识 |
| `task_definition` | `dict` | 是(compile)/否(render) | 非空 | 任务定义 |

### 3.4 任务定义结构

```python
task_definition = {
    "task_name": str,           # 任务名称
    "description": str,         # 任务描述
    "subtasks": [               # 子任务列表
        {
            "subtask_id": str,      # 子任务ID
            "name": str,            # 子任务名称
            "condition": dict|None, # 执行条件（可为空，表示立即执行）
            "transactions": [       # 交易列表
                {
                    "tx_id": str,           # 交易ID
                    "type": str,            # 交易类型: submit/subscribe
                    "from_worker": str|None, # 提交方Worker ID（submit类型）
                    "consumer": str|None,    # 消费者ID（subscribe类型）
                    "product": dict,         # 产品定义
                    "subscribe_to": str|None,# 订阅的交易ID（subscribe类型）
                }
            ]
        }
    ]
}
```

### 3.5 条件结构

```python
condition = {
    "type": str,             # 条件类型
    "ref_tx": str,           # 引用的交易ID
    "check": str,            # 检测项
    "operator": str,         # 操作符: eq/ne/gt/lt/gte/lte/contains
    "value": any,            # 期望值
    "on_fail": str,          # 失败动作: wait/skip/abort/branch
    "branch_to": str|None,   # 分支目标子任务ID（on_fail=branch时）
    "timeout_ms": int|None,  # 超时时间（on_fail=wait时）
}
```

### 3.6 条件类型枚举

| 类型 | 说明 | 检测项 |
|------|------|--------|
| `product_available` | 产品是否可用 | 交易ID对应的产品状态 |
| `product_content` | 产品内容检测 | 产品数据的具体字段 |
| `tx_completed` | 交易是否完成 | 交易执行状态 |
| `tx_failed` | 交易是否失败 | 交易错误状态 |
| `threshold` | 阈值检测 | 产品数据中的数值 |
| `user_manual` | 用户手动触发 | 用户操作信号 |
| `time_elapsed` | 时间条件 | 从任务启动经过的时间 |

### 3.7 交易类型枚举

| 类型 | 说明 | 必填字段 |
|------|------|----------|
| `submit` | Worker向商场提交产品 | from_worker, product |
| `subscribe` | 消费者/Worker订阅商场产品 | consumer, subscribe_to |

### 3.8 返回值规格（compile）

```python
{
    "status": str,              # "ok" / "error"
    "displayed": str,           # 编译结果摘要
    "timestamp": float,
    "source": str,              # "task_editor_compile"
    "metadata": {
        "user_id": str,
        "task_id": str,             # 生成的任务ID
        "task_name": str,
        "subtask_count": int,
        "transaction_count": int,
        "condition_count": int,
        "transaction_sequence": [   # 编译后的交易序列
            {
                "seq_order": int,       # 执行顺序
                "tx_id": str,
                "type": str,
                "from_worker": str|None,
                "consumer": str|None,
                "product": dict,
                "condition": dict|None,
                "dependencies": [str],  # 依赖的交易ID列表
            }
        ],
        "dag_valid": bool,         # DAG是否有环
        "compilation_errors": [],  # 编译错误列表
    },
    "error": str
}
```

### 3.9 返回值规格（render）

```python
{
    "status": str,
    "displayed": str,           # 编辑器界面渲染输出
    "timestamp": float,
    "source": str,              # "task_editor_render"
    "metadata": {
        "user_id": str,
        "available_workers": [str],    # 可用Worker列表
        "available_consumers": [str],  # 可用消费者列表
        "task_tree": dict,             # 任务树形结构
        "tx_dag": dict,                # 交易DAG结构
    },
    "error": str
}
```

## 4. 行为契约

### 4.1 前置条件（compile）

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | user_id 非空 | `ERR_EMPTY_USER_ID` | `error` |
| PRE-002 | user_id 长度 <= 64 | `ERR_USER_ID_TOO_LONG` | `error` |
| PRE-003 | task_definition 非空 | `ERR_EMPTY_TASK_DEF` | `error` |
| PRE-004 | task_definition 含 subtasks | `ERR_NO_SUBTASKS` | `error` |
| PRE-005 | 每个 subtask 含 transactions | `ERR_NO_TRANSACTIONS` | `error` |
| PRE-006 | 交易DAG无环 | `ERR_DAG_HAS_CYCLE` | `error` |
| PRE-007 | 交易ID全局唯一 | `ERR_DUPLICATE_TX_ID` | `error` |
| PRE-008 | subscribe引用的tx_id存在 | `ERR_INVALID_SUBSCRIBE_REF` | `error` |
| PRE-009 | 条件引用的ref_tx存在 | `ERR_INVALID_CONDITION_REF` | `error` |

### 4.2 前置条件（render）

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-R01 | user_id 非空 | `ERR_EMPTY_USER_ID` | `error` |
| PRE-R02 | ctx 包含 shm_accessor | `ERR_MISSING_SHM_ACCESSOR` | `error` |

### 4.3 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | transaction_sequence 按拓扑排序 | DAG合法排序 |
| POST-003 | 每个交易包含依赖列表 | 依赖关系完整 |
| POST-004 | dag_valid 为 True 时无编译错误 | 一致性保证 |

### 4.4 不变量

- 零全局状态：不读写任何全局变量
- 零堆分配：不使用 malloc/free/new/delete
- 零系统调用：不进行文件IO、网络IO、进程管理
- 确定性：相同输入必得相同输出
- DAG合法性：编译输出必须是无环有向图

### 4.5 错误契约表

| 错误码 | 触发条件 | 返回status | 返回displayed | 返回error |
|--------|----------|------------|---------------|-----------|
| `ERR_EMPTY_USER_ID` | user_id为空 | `error` | `""` | "用户ID不能为空" |
| `ERR_USER_ID_TOO_LONG` | user_id超长 | `error` | `""` | "用户ID长度超过限制" |
| `ERR_EMPTY_TASK_DEF` | task_definition为空 | `error` | `""` | "任务定义不能为空" |
| `ERR_NO_SUBTASKS` | 无子任务 | `error` | `""` | "任务必须包含至少一个子任务" |
| `ERR_NO_TRANSACTIONS` | 子任务无交易 | `error` | `""` | "子任务必须包含至少一个交易" |
| `ERR_DAG_HAS_CYCLE` | 交易DAG有环 | `error` | `""` | "交易序列存在循环依赖" |
| `ERR_DUPLICATE_TX_ID` | 交易ID重复 | `error` | `""` | "交易ID必须唯一" |
| `ERR_INVALID_SUBSCRIBE_REF` | subscribe引用无效 | `error` | `""` | "订阅引用的交易ID不存在" |
| `ERR_INVALID_CONDITION_REF` | 条件引用无效 | `error` | `""` | "条件引用的交易ID不存在" |
| `ERR_MISSING_SHM_ACCESSOR` | ctx缺少SHM | `error` | `""` | "缺少SHM访问器" |

## 5. 测试场景

### 5.1 测试用例（compile）

| 编号 | 类别 | 输入 | 预期输出 |
|------|------|------|----------|
| TC-001 | 正常 | 单子任务+单交易(submit) | status="ok", tx_count=1 |
| TC-002 | 正常 | 单子任务+submit+subscribe | status="ok", tx_count=2, subscribe依赖submit |
| TC-003 | 正常 | 多子任务+条件控制 | status="ok", condition_count>0 |
| TC-004 | 正常 | 3子任务链式依赖 | status="ok", dag_valid=True |
| TC-005 | 边界 | 空任务定义 | status="error", ERR_EMPTY_TASK_DEF |
| TC-006 | 边界 | 无子任务 | status="error", ERR_NO_SUBTASKS |
| TC-007 | 边界 | 子任务无交易 | status="error", ERR_NO_TRANSACTIONS |
| TC-008 | 异常 | DAG有环 | status="error", ERR_DAG_HAS_CYCLE |
| TC-009 | 异常 | 重复交易ID | status="error", ERR_DUPLICATE_TX_ID |
| TC-010 | 异常 | subscribe引用不存在的tx | status="error", ERR_INVALID_SUBSCRIBE_REF |
| TC-011 | 异常 | 条件引用不存在的tx | status="error", ERR_INVALID_CONDITION_REF |
| TC-012 | 性能 | 50子任务+500交易 | 执行时间 <= 500ms |

### 5.2 测试用例（render）

| 编号 | 类别 | 输入 | 预期输出 |
|------|------|------|----------|
| TC-R01 | 正常 | user_id + 可用消费者 | status="ok", 含available列表 |
| TC-R02 | 正常 | user_id + 已有任务 | status="ok", 含task_tree |
| TC-R03 | 边界 | 空user_id | status="error", ERR_EMPTY_USER_ID |
| TC-R04 | 异常 | ctx缺少shm | status="error", ERR_MISSING_SHM_ACCESSOR |

### 5.3 测试数据

```python
# 简单任务: 传感器数据采集 → 波形显示
simple_task = {
    "task_name": "传感器监控",
    "description": "采集传感器A数据并波形显示",
    "subtasks": [
        {
            "subtask_id": "st_001",
            "name": "数据采集",
            "condition": None,
            "transactions": [
                {
                    "tx_id": "tx_001",
                    "type": "submit",
                    "from_worker": "worker_sensor_a",
                    "product": {"data_type": "float64", "source": "sensor_a"},
                    "consumer": None,
                    "subscribe_to": None,
                }
            ]
        },
        {
            "subtask_id": "st_002",
            "name": "波形显示",
            "condition": {
                "type": "product_available",
                "ref_tx": "tx_001",
                "check": "status",
                "operator": "eq",
                "value": "ready",
                "on_fail": "wait",
                "timeout_ms": 5000,
            },
            "transactions": [
                {
                    "tx_id": "tx_002",
                    "type": "subscribe",
                    "from_worker": None,
                    "consumer": "cons_waveform",
                    "product": {},
                    "subscribe_to": "tx_001",
                }
            ]
        }
    ]
}

# 有环的任务（应被拒绝）
cyclic_task = {
    "task_name": "循环任务",
    "description": "测试DAG环检测",
    "subtasks": [
        {
            "subtask_id": "st_001",
            "name": "步骤A",
            "condition": None,
            "transactions": [
                {"tx_id": "tx_001", "type": "submit", "from_worker": "w1",
                 "product": {}, "consumer": None, "subscribe_to": "tx_002"}
            ]
        }
    ]
}
```

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `json` | JSON解析 | Python 3.8+ |
| `time` | 时间戳生成 | Python 3.8+ |
| `uuid` | 任务ID生成 | Python 3.8+ |
| `copy` | 深拷贝 | Python 3.8+ |

### 6.2 外部依赖声明

> **零外部依赖**：函数不得依赖任何第三方库。

### 6.3 关联蓝图

| 蓝图 | 关系 | 说明 |
|------|------|------|
| BP-0029-USER-HOMEPAGE | 读取 | 获取用户可用的消费者/Worker列表 |
| BP-0030-TASK-PUBLISHER | 输出 | 编译后的交易序列通过任务发布控制执行 |

### 6.4 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n*m) | n为子任务数, m为平均交易数 |
| 内存 | <= 256KB | 任务定义+交易序列 |
| 栈帧 | <= 16KB | DAG递归深度 |
| 执行时限 | <= 500ms | 编译时间 |

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 不进行任何文件读写 |
| 网络零IO | ✅ | 不进行任何网络通讯 |
| 零全局副作用 | ✅ | 不修改任何全局状态 |
| 输入消毒 | ✅ | 对user_id/task_definition进行校验 |
| 确定性输出 | ✅ | 相同输入产生相同输出 |
| SHM权限合规 | ✅ | 仅读写授权的SHM矢量 |
| 栈帧安全 | ✅ | DAG深度限制防止栈溢出 |
| 零堆分配 | ✅ | 不使用动态内存分配 |

### 7.2 stdout特许声明

- **特许项目**: 编辑器界面渲染输出
- **输出格式**: 文本任务树 + 交易DAG
- **频率限制**: 每次调用输出一次

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现，两个入口函数 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例 |
| 安全扫描 | 验证三大零原则、SHM权限、输入消毒 |
| 授权文件输出 | 按5字段JSON结构生成授权文件 |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据 |
| `design_meta` | 第3节函数设计（双函数） |
| `source_code` | AICoder生成的纯函数实现 |
| `compiled_binary` | PYC编译产物 |
| `verification_report` | 第5节测试场景验收报告 |

---

## 附录A：交易序列执行流程

```
用户定义任务
    │
    ▼
┌─────────────────────────────────┐
│       task_editor_compile        │
├─────────────────────────────────┤
│  1. 验证任务定义结构              │
│  2. 验证交易ID唯一性              │
│  3. 解析交易依赖关系              │
│  4. DAG环检测（拓扑排序）         │
│  5. 验证条件引用                  │
│  6. 生成有序交易序列              │
│  7. 返回编译结果                  │
└─────────────────────────────────┘
    │
    ▼
交易序列 → BP-0030 发布 → 商场执行
```

## 附录B：DAG拓扑排序算法

```
1. 构建邻接表: submit交易 → subscribe交易
2. 计算入度
3. 入度为0的节点入队
4. BFS遍历，输出拓扑序列
5. 若输出节点数 < 总节点数 → 存在环
```

## 附录C：条件控制状态机

```
[检查条件] ──满足──→ [执行下一步] ──→ [下一个条件/交易]
     │
     ├──不满足 + on_fail=wait ──→ [等待] ──超时──→ [abort/skip]
     │
     ├──不满足 + on_fail=skip ──→ [跳过当前子任务]
     │
     ├──不满足 + on_fail=abort ──→ [终止任务]
     │
     └──不满足 + on_fail=branch ──→ [跳转到指定子任务]
```

## 附录D：编辑器界面渲染示例

```
=== 任务编辑器 (user_001) ===

任务: 传感器监控
  ├── 子任务1: 数据采集 [立即执行]
  │     └── tx_001: [submit] worker_sensor_a → 商场
  │
  └── 子任务2: 波形显示 [条件: tx_001产品就绪]
        └── tx_002: [subscribe] cons_waveform ← 商场(tx_001)

交易DAG:
  tx_001 (submit) ──→ 商场 ──→ tx_002 (subscribe)

条件:
  tx_002前置: product_available(tx_001) == ready
  失败策略: wait, 超时5000ms

=== 编译通过: 2交易, 1条件, DAG无环 ===
```
