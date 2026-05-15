# 蓝图文档（定稿版）：BP-0030 -- 任务发布控制

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0030-TASK-PUBLISHER |
| 蓝图名称 | 任务发布控制 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-11 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

## 2. 功能概述

### 2.1 功能描述

用户通过此模块操控任务的发布，支持三种发布模式：
1. **手动发布**：用户即时触发单个任务
2. **序列发布**：按预设序列依次发布多个任务
3. **定时发布**：按时间计划自动发布任务

### 2.2 数据来源

- `shm_vector` 任务队列矢量（task_queue域）
- `shm_vector` 用户配置矢量（user_config域）
- `ctx` 用户发布指令

### 2.3 数据去向

- `shm_vector` 任务调度矢量（scheduler域）
- `shm_vector` 发布日志矢量（audit域）

### 2.4 来源锁定

- 用户ID锁定：任务发布者必须为已认证用户
- 消费者ID锁定：任务目标必须为用户拥有的消费者

## 3. 函数设计

### 3.1 主函数签名

```python
def task_publisher_dispatch(
    ctx: dict,
    user_id: str,
    publish_mode: str,
    task_spec: dict
) -> dict:
    """
    [复杂度]: O(1) 单任务, O(n) 序列任务
    [安全等级]: READ_WRITE_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | `dict` | 是 | 非空 | 上下文对象 |
| `user_id` | `str` | 是 | 长度 <= 64 | 用户唯一标识 |
| `publish_mode` | `str` | 是 | 枚举值 | 发布模式 |
| `task_spec` | `dict` | 是 | 非空 | 任务规格 |

### 3.3 发布模式枚举

| 模式值 | 说明 | task_spec要求 |
|--------|------|---------------|
| `manual` | 手动发布 | 单任务规格 |
| `sequence` | 序列发布 | 任务序列规格 |
| `scheduled` | 定时发布 | 任务+时间规格 |

### 3.4 任务规格结构

```python
# 单任务规格（manual模式）
task_spec = {
    "consumer_id": str,       # 目标消费者ID
    "task_type": str,         # 任务类型
    "task_data": dict,        # 任务数据
    "priority": int           # 优先级 (0-9)
}

# 序列规格（sequence模式）
task_spec = {
    "sequence_name": str,     # 序列名称
    "tasks": [                # 任务列表
        {"consumer_id": str, "task_type": str, "task_data": dict, "delay_ms": int}
    ],
    "stop_on_error": bool     # 遇错是否停止
}

# 定时规格（scheduled模式）
task_spec = {
    "consumer_id": str,
    "task_type": str,
    "task_data": dict,
    "schedule": {
        "type": str,          # "once" / "repeat"
        "trigger_time": float,  # 触发时间戳
        "interval_ms": int      # 重复间隔（repeat模式）
    }
}
```

### 3.5 返回值规格

```python
{
    "status": str,           # "ok" / "error" / "partial"
    "displayed": str,        # 发布结果摘要
    "timestamp": float,      # 操作时间戳
    "source": str,           # "task_publisher_dispatch"
    "metadata": {
        "user_id": str,
        "publish_mode": str,
        "published_count": int,    # 已发布任务数
        "failed_count": int,       # 失败任务数
        "publish_ids": [str],      # 发布ID列表
        "sequence_status": str     # 序列状态（sequence模式）
    },
    "error": str             # 错误信息
}
```

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | user_id 非空 | `ERR_EMPTY_USER_ID` | `error` |
| PRE-002 | publish_mode 有效 | `ERR_INVALID_PUBLISH_MODE` | `error` |
| PRE-003 | task_spec 非空 | `ERR_EMPTY_TASK_SPEC` | `error` |
| PRE-004 | consumer_id 属于该用户 | `ERR_CONSUMER_NOT_OWNED` | `error` |
| PRE-005 | task_type 有效 | `ERR_INVALID_TASK_TYPE` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" / "error" / "partial" | 三态返回 |
| POST-002 | 任务已写入调度矢量 | 持久化保证 |
| POST-003 | 发布日志已记录 | 审计保证 |

### 4.3 不变量

- 零全局状态：不读写任何全局变量
- 零堆分配：不使用 malloc/free/new/delete
- 零系统调用：不进行文件IO、网络IO、进程管理
- 确定性：相同输入必得相同输出
- 原子性：单任务发布为原子操作

### 4.4 任务类型映射

| 任务类型 | 适用消费者 | 说明 |
|----------|------------|------|
| `render` | waveform, table, chart | 渲染任务 |
| `update` | table, form | 数据更新 |
| `control` | form | 控制指令 |
| `query` | 所有消费者 | 状态查询 |
| `reset` | 所有消费者 | 重置状态 |

### 4.5 错误契约表

| 错误码 | 触发条件 | 返回status | 返回displayed | 返回error |
|--------|----------|------------|---------------|-----------|
| `ERR_EMPTY_USER_ID` | user_id为空 | `error` | `""` | "用户ID不能为空" |
| `ERR_INVALID_PUBLISH_MODE` | 发布模式无效 | `error` | `""` | "无效的发布模式" |
| `ERR_EMPTY_TASK_SPEC` | task_spec为空 | `error` | `""` | "任务规格不能为空" |
| `ERR_CONSUMER_NOT_OWNED` | 消费者不属于用户 | `error` | `""` | "无权访问该消费者" |
| `ERR_INVALID_TASK_TYPE` | 任务类型无效 | `error` | `""` | "无效的任务类型" |
| `ERR_SEQUENCE_EXECUTION` | 序列执行失败 | `partial` | 部分结果 | "序列执行部分失败" |
| `ERR_SCHEDULER_FULL` | 调度队列已满 | `error` | `""` | "调度队列已满" |

## 5. 测试场景

### 5.1 测试用例

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | manual模式, 单任务 | 默认 | status="ok", published_count=1 |
| TC-002 | 正常 | sequence模式, 3任务序列 | 默认 | status="ok", published_count=3 |
| TC-003 | 正常 | scheduled模式, 定时任务 | 5秒后触发 | status="ok", 任务已调度 |
| TC-004 | 边界 | manual模式, 优先级最高 | priority=9 | status="ok" |
| TC-005 | 边界 | sequence模式, 空序列 | tasks=[] | status="error", ERR_EMPTY_TASK_SPEC |
| TC-006 | 边界 | scheduled模式, 过去时间 | trigger_time=过去 | status="error" |
| TC-007 | 异常 | manual模式, 无效consumer_id | consumer_id="invalid" | status="error", ERR_CONSUMER_NOT_OWNED |
| TC-008 | 异常 | manual模式, 无效task_type | task_type="invalid" | status="error", ERR_INVALID_TASK_TYPE |
| TC-009 | 异常 | sequence模式, 中间任务失败 | stop_on_error=true | status="partial" |
| TC-010 | 性能 | sequence模式, 100任务 | 默认 | 执行时间 <= 200ms |

### 5.2 测试数据准备

```python
# 用户1的消费者
user_001_consumers = ["cons_001", "cons_002", "cons_003"]

# 标准任务规格
standard_task = {
    "consumer_id": "cons_001",
    "task_type": "render",
    "task_data": {"data": [1, 2, 3, 4, 5]},
    "priority": 5
}

# 序列任务规格
sequence_task = {
    "sequence_name": "daily_report",
    "tasks": [
        {"consumer_id": "cons_001", "task_type": "render", "task_data": {}, "delay_ms": 0},
        {"consumer_id": "cons_002", "task_type": "update", "task_data": {}, "delay_ms": 100},
        {"consumer_id": "cons_003", "task_type": "control", "task_data": {}, "delay_ms": 200}
    ],
    "stop_on_error": False
}
```

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `json` | JSON解析 | Python 3.8+ |
| `time` | 时间戳生成 | Python 3.8+ |
| `uuid` | 发布ID生成 | Python 3.8+ |

### 6.2 外部依赖声明

> **零外部依赖**：函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n) | n为序列任务数 |
| 内存 | <= 128KB | 任务数据缓存 |
| 栈帧 | <= 8KB | 递归深度限制 |
| 执行时限 | <= 500ms | 序列发布时间 |

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 不进行任何文件读写 |
| 网络零IO | ✅ | 不进行任何网络通讯 |
| 零全局副作用 | ✅ | 不修改任何全局状态 |
| 输入消毒 | ✅ | 对所有输入进行边界校验 |
| 确定性输出 | ✅ | 相同输入产生相同输出 |
| SHM权限合规 | ✅ | 仅访问授权的SHM矢量 |
| 栈帧安全 | ✅ | 不存在栈溢出风险 |
| 零堆分配 | ✅ | 不使用动态内存分配 |

### 7.2 权限验证

- 用户所有权验证：consumer_id必须属于user_id
- 任务类型验证：task_type必须在允许列表内
- 优先级范围验证：priority必须在0-9范围内

## 8. 授权文件生成要求

### 8.1 致AICoder指令

| 指令项 | 内容 |
|--------|------|
| 契约审查 | 依据本蓝图第4节行为契约进行逐条验证 |
| 代码生成 | 将蓝图意图转化为纯函数级Python实现 |
| 测试覆盖 | 基于第5节测试场景生成验收测试用例 |
| 安全扫描 | 验证三大零原则、SHM权限、输入消毒 |
| 授权文件输出 | 按5字段JSON结构生成授权文件 |

### 8.2 蓝图到授权文件的映射

| 授权文件字段 | 对应蓝图内容 |
|-------------|-------------|
| `auth_header` | 第1节蓝图元数据 |
| `design_meta` | 第3节函数设计 |
| `source_code` | AICoder生成的纯函数实现 |
| `compiled_binary` | PYC编译产物 |
| `verification_report` | 第5节测试场景验收报告 |

---

## 附录：发布流程示意

```
用户指令
    │
    ▼
┌─────────────────────────────────────┐
│        task_publisher_dispatch       │
├─────────────────────────────────────┤
│  1. 验证用户身份                      │
│  2. 验证消费者所有权                  │
│  3. 解析发布模式                      │
│  4. 执行发布逻辑                      │
│     ├── manual: 立即写入调度队列      │
│     ├── sequence: 按序列依次发布      │
│     └── scheduled: 写入定时队列       │
│  5. 记录发布日志                      │
│  6. 返回发布结果                      │
└─────────────────────────────────────┘
    │
    ▼
调度队列 / 定时队列
```

## 附录：序列发布状态机

```
[INIT] ──► [RUNNING] ──► [COMPLETED]
               │
               ├──(错误, stop_on_error=true)──► [FAILED]
               │
               └──(错误, stop_on_error=false)──► [PARTIAL]
```
