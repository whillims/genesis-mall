# 蓝图文档（定稿版）：BP-0029 -- 用户主页定制

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | BP-0029-USER-HOMEPAGE |
| 蓝图名称 | 用户主页定制 |
| 生产者 | Genesis-Worker-001 |
| 提交时间 | 2026-05-11 |
| 版本 | 1.0.0 |
| 目标语言 | Python |
| 复杂度等级 | Level-1 |
| 设计范式 | 纯函数级，禁止OOP封装 |

## 2. 功能概述

### 2.1 功能描述

根据用户拥有的消费者列表，动态生成个性化主页布局。主页以"相"的形式展示每个消费者，用户可通过主页查看、选择、操控消费者。

### 2.2 数据来源

- `shm_vector` 用户配置矢量（user_config域）
- `shm_vector` 消费者注册表矢量（registry域）

### 2.3 数据去向

- `stdout` 渲染主页布局
- `shm_vector` 主页状态矢量（homepage域）

### 2.4 来源锁定

- 用户ID锁定：必须从ctx获取user_id
- 消费者列表锁定：必须从registry域读取

## 3. 函数设计

### 3.1 主函数签名

```python
def user_homepage_render(
    ctx: dict,
    user_id: str,
    layout_config: dict = None
) -> dict:
    """
    [复杂度]: O(n), n为消费者数量
    [安全等级]: READ_ONLY_SHM
    """
```

### 3.2 参数规格表

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `ctx` | `dict` | 是 | 非空 | 上下文对象，含SHM访问器 |
| `user_id` | `str` | 是 | 长度 <= 64，非空 | 用户唯一标识 |
| `layout_config` | `dict` | 否 | - | 布局配置（列数、排序方式等） |

### 3.3 数据来源定义

| 矢量域 | 数据类型 | 用途 |
|--------|----------|------|
| `registry` | `json` | 消费者注册信息 |
| `user_config` | `json` | 用户偏好配置 |
| `homepage` | `json` | 主页状态缓存 |

### 3.4 返回值规格

```python
{
    "status": str,           # "ok" / "error"
    "displayed": str,        # 主页渲染输出
    "timestamp": float,      # 操作时间戳
    "source": str,           # "user_homepage_render"
    "metadata": {
        "user_id": str,      # 用户ID
        "consumer_count": int,  # 消费者数量
        "layout": str,       # 布局类型
        "consumers": [       # 消费者"相"列表
            {
                "consumer_id": str,
                "consumer_type": str,  # "waveform" / "table" / "form" 等
                "display_name": str,
                "status": str,         # "idle" / "running" / "error"
                "position": {"row": int, "col": int}
            }
        ]
    },
    "error": str             # 错误信息（仅status="error"时）
}
```

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | user_id 非空 | `ERR_EMPTY_USER_ID` | `error` |
| PRE-002 | user_id 长度 <= 64 | `ERR_USER_ID_TOO_LONG` | `error` |
| PRE-003 | ctx 包含shm_accessor | `ERR_MISSING_SHM_ACCESSOR` | `error` |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | metadata.consumers 包含该用户所有消费者 | 完整性保证 |
| POST-003 | 每个消费者"相"包含必需字段 | 结构完整 |

### 4.3 不变量

- 零全局状态：不读写任何全局变量
- 零堆分配：不使用 malloc/free/new/delete
- 零系统调用：不进行文件IO、网络IO、进程管理
- 确定性：相同输入必得相同输出
- 只读SHM：仅读取registry和user_config域，写入homepage域

### 4.4 消费者"相"类型映射

| 消费者类型 | 显示名称 | 图标标识 | 说明 |
|------------|----------|----------|------|
| `waveform` | 波形显示消费者 | `icon_waveform` | 实时波形渲染 |
| `table` | 表格显示消费者 | `icon_table` | 数据表格展示 |
| `form` | 控制表单消费者 | `icon_form` | 交互控制表单 |
| `console` | 控制台消费者 | `icon_console` | 文本输出 |
| `chart` | 图表消费者 | `icon_chart` | 统计图表 |

### 4.5 错误契约表

| 错误码 | 触发条件 | 返回status | 返回displayed | 返回error |
|--------|----------|------------|---------------|-----------|
| `ERR_EMPTY_USER_ID` | user_id为空 | `error` | `""` | "用户ID不能为空" |
| `ERR_USER_ID_TOO_LONG` | user_id超长 | `error` | `""` | "用户ID长度超过限制" |
| `ERR_MISSING_SHM_ACCESSOR` | ctx缺少SHM访问器 | `error` | `""` | "缺少SHM访问器" |
| `ERR_REGISTRY_READ_FAILED` | 注册表读取失败 | `error` | `""` | "无法读取消费者注册表" |
| `ERR_NO_CONSUMERS` | 用户无消费者 | `error` | `""` | "用户未拥有任何消费者" |

## 5. 测试场景

### 5.1 测试用例

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | user_id="user_001" | 默认布局 | status="ok", consumers包含3个消费者 |
| TC-002 | 正常 | user_id="user_001" | {"columns": 4} | status="ok", 布局为4列 |
| TC-003 | 正常 | user_id="user_002" | 默认布局 | status="ok", consumers包含1个消费者 |
| TC-004 | 边界 | user_id="" | 默认布局 | status="error", ERR_EMPTY_USER_ID |
| TC-005 | 边界 | user_id=64字符 | 默认布局 | status="ok" |
| TC-006 | 边界 | user_id=65字符 | 默认布局 | status="error", ERR_USER_ID_TOO_LONG |
| TC-007 | 异常 | user_id="nonexistent" | 默认布局 | status="error", ERR_NO_CONSUMERS |
| TC-008 | 异常 | ctx={} | 默认布局 | status="error", ERR_MISSING_SHM_ACCESSOR |
| TC-009 | 性能 | user_id="user_perf" | 默认布局, 100消费者 | 执行时间 <= 50ms |

### 5.2 测试数据准备

```python
# 用户1拥有3个消费者
user_001_consumers = [
    {"consumer_id": "cons_001", "type": "waveform", "display_name": "波形显示"},
    {"consumer_id": "cons_002", "type": "table", "display_name": "表格显示"},
    {"consumer_id": "cons_003", "type": "form", "display_name": "控制表单"}
]

# 用户2拥有1个消费者
user_002_consumers = [
    {"consumer_id": "cons_004", "type": "console", "display_name": "控制台"}
]
```

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `json` | JSON解析 | Python 3.8+ |
| `time` | 时间戳生成 | Python 3.8+ |

### 6.2 外部依赖声明

> **零外部依赖**：函数不得依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(n) | n为消费者数量 |
| 内存 | <= 64KB | 主页布局数据 |
| 栈帧 | <= 4KB | 递归深度限制 |
| 执行时限 | <= 100ms | 渲染时间 |

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 不进行任何文件读写 |
| 网络零IO | ✅ | 不进行任何网络通讯 |
| 零全局副作用 | ✅ | 不修改任何全局状态 |
| 输入消毒 | ✅ | 对user_id进行长度和字符校验 |
| 确定性输出 | ✅ | 相同输入产生相同输出 |
| SHM权限合规 | ✅ | 仅读取registry/user_config，写入homepage |
| 栈帧安全 | ✅ | 不存在栈溢出风险 |
| 零堆分配 | ✅ | 不使用动态内存分配 |

### 7.2 stdout特许声明

- **特许项目**: 主页布局渲染输出
- **输出格式**: ASCII艺术布局或JSON结构
- **频率限制**: 每次调用输出一次

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

## 附录：用户主页布局示例

```
┌─────────────────────────────────────────────────────────────┐
│                      用户主页 (user_001)                      │
├─────────────────────────┬───────────────────────────────────┤
│  ┌─────────────────┐   │  ┌─────────────────────────────┐   │
│  │  波形显示消费者   │   │  │      表格显示消费者          │   │
│  │  [icon_waveform] │   │  │      [icon_table]           │   │
│  │  状态: running   │   │  │      状态: idle             │   │
│  └─────────────────┘   │  └─────────────────────────────┘   │
├─────────────────────────┴───────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  控制表单消费者                        │   │
│  │                  [icon_form]                          │   │
│  │                  状态: idle                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```
