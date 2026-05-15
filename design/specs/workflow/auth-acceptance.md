# 商场接受授权书流程规范

> **文档版本**: v2.0  
> **适用对象**: 微型商场主程序、外置AICoder、生产者Worker  
> **核心原则**: 函数级设计、物理隔离、安全编译、行为验证  
> **关联文档**: 《AICoder授权文件生成规则》《创世纪七阶段流程》《SHM矢量管理宪法》《函数授权文件生成.md》  
> **v2.0变更**: 增加AICoder七维审查流程、Python语言支持、input_builder机制、创世纪验证实例

---

## 一、流程总览

商场接受授权书是生产者进入生态的**唯一合法入口**。完整链路分为两大阶段：

1. **AICoder外置阶段**：生产者提交蓝图 → AICoder执行七维审查 → 生成授权文件
2. **商场内置阶段**：商场执行七步验证 → 注册函数表 → 载入算力池

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  生产者Worker  │────▶│   外置AICoder      │────▶│   生成授权文件    │
│  (提交蓝图)   │     │ (七维审查/测试/编译) │     │ (含代码+元数据)  │
└─────────────┘     └──────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│  载入算力池   │◀────│  注册函数表   │◀────│  商场七步验证     │
│  (获得执行权)  │     │ (分配FuncID) │     │ (接受授权书流程)  │
└─────────────┘     └─────────────┘     └─────────────────┘
```

---

## 二、AICoder七维审查流程（外置阶段）

外置AICoder收到蓝图后，必须执行以下七维审查，**缺一维则授权文件无效**。

### 2.1 七维审查清单

| 维度 | 名称 | 审查内容 | 判定标准 |
|------|------|----------|----------|
| D1 | 形式规范维 | 蓝图ID命名、函数签名完整性、返回值结构、错误码体系、章节完整性 | 全部符合《商场函数命名宪章》 |
| D2 | 安全铁律维 | 零全局状态、零堆分配、零系统调用、边界安全、输入消毒、stdout特许合规 | 三大零原则+边界安全全部通过 |
| D3 | 语义一致维 | 前置条件与错误码映射、后置条件与返回值一致、来源锁定语义、控制台输出格式 | 无歧义、无矛盾 |
| D4 | 性能资源维 | 内存约束、时间约束、栈深度、SIMD需求 | 在商场宿主承载范围内 |
| D5 | 测试覆盖维 | 正常/边界/异常/性能路径覆盖 | 至少覆盖底线场景 |
| D6 | 编译可行维 | 语言版本兼容、标准库依赖、语法合法性、无OOP构造 | 可无警告编译通过 |
| D7 | 生态兼容维 | SHM矢量兼容、线程池可入、函数目录可注册 | 输出可被SHM矢量管理器接管 |

### 2.2 AICoder代码精化铁律

精化蓝图为可编译代码时必须遵守：

1. **语义冻结**：不得改变算法本质、不得变更接口签名、不得放宽或收紧前置条件
2. **安全强化**：将生产者的"自证"转化为运行时断言或编译期强制
3. **防御编程**：所有指针解引用前校验，所有数学运算前保护
4. **零堆原则**：使用VLA或固定最大栈分配，禁止`malloc`/`free`/`new`/`delete`
5. **内联隔离**：所有内部辅助函数标记`static`，防止符号表污染
6. **错误码统一**：使用蓝图声明的错误码宏，禁止魔数
7. **头部封印**：代码头部必须嵌入授权编号、生产者身份、AICoder审查指纹

---

## 三、授权书格式规范

授权书为**单一结构化文件**，采用JSON格式，必须包含以下五大字段。任何缺失或格式错误，商场直接拒收。

### 3.1 授权书结构

```json
{
  "auth_header": {
    "blueprint_id": "BP-0001-ORIGIN",
    "producer_name": "Genesis-Worker-001",
    "producer_type": "IO_DEVICE",
    "submit_timestamp": "2026-05-10T08:00:00Z",
    "aicoder_signature": "SHA256-HASH",
    "auth_version": "1.0",
    "blueprint_hash_sha256": "蓝图文件SHA256"
  },
  "design_meta": {
    "function_name": "keyboard_echo",
    "language": "Python",
    "interface_type": "DIRECT_CALL",
    "input_schema": [
      {"name": "user_input", "type": "str", "constraint": "描述"}
    ],
    "output_schema": [
      {"name": "status", "type": "str", "constraint": "success|error"}
    ],
    "side_effect": ["stdout_write"],
    "dependency": ["time", "sys"],
    "stack_estimate_bytes": 2048,
    "max_execution_ms": 5,
    "error_codes": {
      "E001": "输入为空或仅含空白字符",
      "E009": "来源标识非法"
    }
  },
  "source_code": {
    "language": "Python",
    "code": "def keyboard_echo(user_input: str, config: dict | None = None) -> dict: ...",
    "hash_sha256": "CODE-HASH"
  },
  "compiled_binary": {
    "format": "CPython-314-PYC",
    "base64_blob": "base64编码的字节码...",
    "entry_symbol": "keyboard_echo",
    "relocations": [],
    "hash_sha256": "BINARY-HASH"
  },
  "verification_report": {
    "static_analysis_pass": true,
    "behavior_test_pass": true,
    "security_scan_pass": true,
    "performance_test_pass": true,
    "test_cases": [
      {
        "id": "TC-001",
        "name": "正常键盘输入回显",
        "input": {"user_input": "Hello, 商场!"},
        "expected_output": {"status": "success", "displayed": true},
        "actual_output": {"status": "success", "displayed": true},
        "pass": true
      }
    ],
    "aicoder_remarks": "七维审查全部通过。纯函数级设计..."
  }
}
```

### 3.2 关键字段约束

| 字段 | 约束 | 违规后果 |
|------|------|----------|
| `function_name` | 全局唯一，符合标识符规范 | 重名拒收 |
| `language` | 仅限 C / Python / Verilog / JavaScript | 不支持的编译目标拒收 |
| `code` | 禁止包含 `class`, `struct`, `global`, `static` (文件级) | 静态分析失败，直接拒收 |
| `side_effect` | 必须显式声明所有副作用 | 隐瞒副作用视为安全违规 |
| `compiled_binary` | 必须提供，商场不现场编译 | 缺失则拒收 |
| `verification_report` | 四项测试必须全为 `true` | 任一失败拒收 |
| `aicoder_signature` | 必须为有效的SHA256签名 | 签名不匹配拒收 |
| `blueprint_hash_sha256` | 必须与蓝图文件哈希一致 | 哈希不匹配拒收 |

### 3.3 编译产物格式规范

| 语言 | `compiled_binary.format` | 加载方式 | 入口符号 |
|------|--------------------------|----------|----------|
| C | `ELF-x86_64` | `dlsym`解析 | C函数名 |
| Python | `CPython-{version}-PYC` | `marshal.loads` + `exec` | Python函数名 |
| Verilog | `FPGA-BITSTREAM` | 烧录接口 | 模块名 |
| JavaScript | `V8-SNAPSHOT` | `v8.deserialize` | 导出函数名 |

### 3.4 测试用例input_builder机制

测试用例中，部分输入无法直接以JSON字面量表示（如超长字符串、包含空字节的字符串等），需使用`input_builder`字段声明构造方式：

```json
{
  "id": "TC-004",
  "name": "超长输入",
  "input": {
    "input_builder": {
      "type": "repeat",
      "char": "x",
      "count": 1025
    }
  },
  "expected_output": {"status": "error", "error.code": "E002"}
}
```

**input_builder类型定义**：

| type | 用途 | 参数 | 生成结果 |
|------|------|------|----------|
| `repeat` | 重复单字符 | `char`, `count` | `char * count` |
| `repeat_text` | 重复文本 | `text`, `count` | `text * count` |
| `whitespace` | 空白字符 | `value` | 原样使用value |
| `null_byte` | 包含空字节 | `base`, `position` | `base[:pos] + \x00 + base[pos:]` |

**优先级**：当`input_builder`存在时，忽略`user_input`字段，使用构造结果。

---

## 四、商场七步接受流程

商场主函数 `mall_accept_auth()` 为唯一入口，内部调用七个子流程。任何一步失败，立即返回错误码，授权书作废，不得重试同一份文件（需重新提交新蓝图）。

### Step 1: 格式完整性校验 `auth_validate_format()`

**职责**: 检查JSON结构完整性，字段是否存在，类型是否正确。

**判定标准**:
- JSON可解析
- 五大顶级字段齐全（`auth_header`, `design_meta`, `source_code`, `compiled_binary`, `verification_report`）
- 每个顶级字段内的必需子字段齐全
- 所有字符串字段非空
- `auth_version` 匹配商场当前版本（当前为`"1.0"`）

**输出**:
- 成功: `AUTH_OK`
- 失败: `AUTH_ERR_FORMAT_MISSING_FIELD`, `AUTH_ERR_VERSION_MISMATCH`

---

### Step 2: 数字签名与来源验证 `auth_verify_signature()`

**职责**: 验证授权书确实由受信任的外置AICoder签发，防止伪造。

**签名算法**:
1. 构造签名载荷：`SHA256(sort_json({blueprint_id, producer_name, function_name, source_hash, binary_hash}))`
2. 将载荷哈希与`auth_header.aicoder_signature`比对
3. 验证`submit_timestamp`在24小时有效期内

**判定标准**:
- 签名哈希完全匹配
- 提交时间戳在有效期内

**输出**:
- 成功: `AUTH_OK`
- 失败: `AUTH_ERR_SIG_INVALID`, `AUTH_ERR_SIG_EXPIRED`

---

### Step 3: 代码哈希一致性校验 `auth_verify_code_hash()`

**职责**: 确保授权书中的 `source_code.code` 与 `compiled_binary` 对应，且未被篡改。

**判定标准**:
- 重新计算 `source_code.code` 的SHA256，与 `source_code.hash_sha256` 比对
- Base64解码 `compiled_binary.base64_blob`，重新计算SHA256，与 `compiled_binary.hash_sha256` 比对
- 两项必须全部匹配

**输出**:
- 成功: `AUTH_OK`
- 失败: `AUTH_ERR_HASH_MISMATCH`

---

### Step 4: 静态规则审查 `auth_static_rule_check()`

**职责**: 商场独立执行二次静态审查，不盲信AICoder报告。这是**安全铁律**。

**审查规则（函数级设计铁律）**:

| 规则编号 | 规则描述 | 检查方式 |
|----------|----------|----------|
| R001 | 禁止 `class` / `struct` / `union` 定义 | 正则匹配 |
| R002 | 禁止文件级 `static` / `global` / `nonlocal` 变量 | 正则匹配 |
| R003 | 禁止 `malloc` / `free` / `new` / `delete` | 正则匹配 |
| R004 | 函数参数必须显式声明类型，禁止 `void*` 隐式传递 | 语法树扫描 |
| R005 | 禁止递归调用（防止栈溢出） | 调用图分析 |
| R006 | 禁止系统调用 `exec`, `fork`, `system`, `open`, `socket`, `subprocess` | 正则匹配 |
| R007 | 所有外部依赖必须在 `dependency` 中声明 | 符号表比对 |

**Python扩展规则**:

| 规则编号 | 规则描述 | 检查方式 |
|----------|----------|----------|
| R008 | 禁止 `import os` / `import subprocess` / `import socket` | 正则匹配 |
| R009 | 禁止 `os.system` / `os.open` / `urllib` / `requests` | 正则匹配 |
| R010 | 禁止 `open()` 文件IO | 正则匹配 |

**判定标准**:
- 零违规通过
- 任何违规记录到 `violations` 数组，流程终止

**输出**:
- 成功: `AUTH_OK`
- 失败: `AUTH_ERR_RULE_VIOLATION`

---

### Step 5: 沙箱加载与符号验证 `auth_sandbox_load()`

**职责**: 在隔离沙箱中加载编译后的二进制，验证其可执行性与入口符号。

**Python沙箱加载流程**:
1. Base64解码PYC字节码
2. 跳过PYC头部（16字节magic+flags+timestamp），使用`marshal.loads`解析代码对象
3. 在隔离的`module_dict`中`exec`代码对象
4. 使用`entry_symbol`从`module_dict`中解析入口函数
5. 验证解析结果为可调用对象
6. 若PYC加载失败，回退至`source_code.code`的`compile+exec`方式

**沙箱环境限制**:
- 无文件系统访问
- 无网络
- 栈大小限制为声明值的2倍

**输出**:
- 成功: `AUTH_OK` + 沙箱句柄
- 失败: `AUTH_ERR_LOAD_FAILED`, `AUTH_ERR_SYMBOL_MISSING`

---

### Step 6: 行为回归测试 `auth_behavior_regression()`

**职责**: 使用授权书中附带的测试用例，在沙箱中实际执行函数，验证行为一致性。

**执行动作**:
1. 逐条读取 `verification_report.test_cases`
2. 构造输入参数（优先使用`input_builder`，否则使用`user_input`/`config`）
3. 在沙箱中调用函数，捕获输出与副作用
4. 逐字段比对 `expected_output` 与实际返回值
5. 检查单条用例执行时间不超过 `max_execution_ms`

**输出字段比对规则**:
- `status`: 精确匹配
- `error.code`: 精确匹配（使用点号路径如`error.code`）
- `displayed`: 精确匹配
- `source`: 精确匹配
- `metadata.input_length`: 精确匹配

**判定标准**:
- 所有测试用例通过
- 无副作用越界
- 单条用例执行时间不超过 `max_execution_ms`

**输出**:
- 成功: `AUTH_OK`
- 失败: `AUTH_ERR_TEST_FAIL`, `AUTH_ERR_TIMEOUT`, `AUTH_ERR_SIDE_EFFECT`

---

### Step 7: 注册函数表与算力池分配 `auth_register_function()`

**职责**: 通过全部验证后，将函数正式注册到商场运行时，分配唯一FuncID。

**执行动作**:
1. 在商场全局函数表中分配 `func_id`（原子递增）
2. 生成全局唯一句柄 `FUNC-HANDLE-{序号:04d}`
3. 记录函数元数据: 名称、蓝图ID、生产者、类型、入口函数、栈预算、时限、副作用清单
4. 将沙箱句柄迁移至算力进程池的隔离执行单元
5. 更新SHM矢量: `func_registry[func_id] = {handle, meta, state: ACTIVE}`
6. 向生产者Worker发送 `AUTH_ACCEPTED` 回执，包含 `func_id`

**输出**:
- 成功: `AUTH_OK` + `assigned_id`
- 失败: `AUTH_ERR_REGISTRY_FULL`

---

## 五、流程状态机

```
[SUBMITTED]
    │
    ▼
[FORMAT_CHECK] ──失败──▶ [REJECTED]
    │
    ▼
[SIG_VERIFY] ──失败──▶ [REJECTED]
    │
    ▼
[HASH_CHECK] ──失败──▶ [REJECTED]
    │
    ▼
[STATIC_RULE] ──失败──▶ [REJECTED]
    │
    ▼
[LOAD_SANDBOX] ──失败──▶ [REJECTED]
    │
    ▼
[REGRESSION] ──失败──▶ [REJECTED]
    │
    ▼
[REGISTERED] ──成功──▶ [ACTIVE]
```

**状态定义**:
- `REJECTED`: 授权书作废，记录审计日志，通知生产者失败原因
- `ACTIVE`: 函数可被调度器调用，进入正常生命周期

---

## 六、商场主函数接口

### 6.1 Python实现接口

```python
def mall_accept_auth(auth_json_blob: bytes) -> Tuple[int, str, Optional[int]]:
    """
    商场接受授权书的唯一入口。

    Args:
        auth_json_blob: 授权文件JSON的字节流

    Returns:
        (error_code, message, func_id)
        error_code=0表示成功，func_id为分配的函数ID
    """

def call_registered_func(func_id: int, *args, **kwargs) -> Any:
    """
    调用已注册的函数。

    Args:
        func_id: 函数注册ID

    Returns:
        函数执行结果
    """

def get_func_registry() -> Dict[int, Dict[str, Any]]:
    """获取函数注册表副本"""

def get_audit_log() -> List[Dict[str, Any]]:
    """获取审计日志副本"""
```

### 6.2 错误码定义

| 错误码 | 值 | 含义 |
|--------|-----|------|
| `AUTH_OK` | 0 | 验证通过 |
| `AUTH_ERR_FORMAT_MISSING_FIELD` | 1001 | 格式校验失败: 字段缺失 |
| `AUTH_ERR_VERSION_MISMATCH` | 1002 | 版本不匹配 |
| `AUTH_ERR_SIG_INVALID` | 2001 | 签名验证失败 |
| `AUTH_ERR_SIG_EXPIRED` | 2002 | 签名已过期 |
| `AUTH_ERR_HASH_MISMATCH` | 3001 | 哈希校验失败 |
| `AUTH_ERR_RULE_VIOLATION` | 4001 | 静态规则违规 |
| `AUTH_ERR_LOAD_FAILED` | 5001 | 沙箱加载失败 |
| `AUTH_ERR_SYMBOL_MISSING` | 5002 | 入口符号缺失 |
| `AUTH_ERR_TEST_FAIL` | 6001 | 行为回归测试失败 |
| `AUTH_ERR_TIMEOUT` | 6002 | 执行超时 |
| `AUTH_ERR_SIDE_EFFECT` | 6003 | 副作用越界 |
| `AUTH_ERR_REGISTRY_FULL` | 7001 | 函数注册表已满 |

---

## 七、创世纪验证实例：键盘生产者 BP-0001-ORIGIN

### 7.1 生产者信息

| 属性 | 值 |
|------|-----|
| 蓝图ID | `BP-0001-ORIGIN` |
| 生产者名称 | `Genesis-Worker-001` |
| 生产者类型 | `IO_DEVICE` |
| 功能描述 | 键盘输入回显，接收用户输入字符串，执行校验，输出至控制台 |
| 数据来源 | 键盘输入（固定为keyboard） |
| 数据去向 | 控制台stdout |
| 语言 | Python (>=3.10) |

### 7.2 AICoder七维审查结果

| 维度 | 结果 | 关键检查项 |
|------|------|-----------|
| D1 形式规范维 | PASS | 蓝图ID命名合规、函数签名完整、5个错误码覆盖全部异常 |
| D2 安全铁律维 | PASS | 零全局/零堆/零系统调用、stdout特许已声明、三重输入校验 |
| D3 语义一致维 | PASS | 5条前置条件→5个错误码一一映射、来源锁定keyboard |
| D4 性能资源维 | PASS | ≤256KB、processing_ms<5、栈深O(1) |
| D5 测试覆盖维 | PASS | 正常/边界/异常/性能4类10项用例 |
| D6 编译可行维 | PASS | Python≥3.10、仅time/sys依赖、无OOP |
| D7 生态兼容维 | PASS | 返回dict可JSON序列化、无状态可并发 |

### 7.3 商场七步验证结果

| 步骤 | 函数 | 结果 | 详情 |
|------|------|------|------|
| Step 1 | `auth_validate_format()` | PASS | 5大字段齐全，版本1.0匹配 |
| Step 2 | `auth_verify_signature()` | PASS | AICoder签名SHA256校验通过 |
| Step 3 | `auth_verify_code_hash()` | PASS | 源代码+PYC双哈希匹配 |
| Step 4 | `auth_static_rule_check()` | PASS | 23项OOP/安全规则零违规 |
| Step 5 | `auth_sandbox_load()` | PASS | 入口符号keyboard_echo解析成功 |
| Step 6 | `auth_behavior_regression()` | PASS | 10项行为回归测试全部通过 |
| Step 7 | `auth_register_function()` | PASS | FuncID=1, Handle=FUNC-HANDLE-0001 |

### 7.4 授权文件关键指纹

| 项目 | 值 |
|------|-----|
| 授权ID | `MALL-AUTH-20260510-0001-CODE` |
| 源代码SHA-256 | `5e68b5f54ec70f0be6b73374a037e566bb445755f0cdf5e16127bbe52dcf3acc` |
| 编译产物SHA-256 | `734997084d88e8c4be2bc73359ffda6ffacf4922b1a077a5b2939c7ea28425bc` |
| 蓝图SHA-256 | `48366011194ac9ddf20bb3ee3438fd0bd799fdff795009cccb46defe1045e8fb` |
| AICoder签名 | `f743d6c52fa7e4ccb7bb50078b311772bbfcab263fa2548edcd2695e1c893513` |

### 7.5 函数注册表记录

```
FuncID=1
  Handle: FUNC-HANDLE-0001
  Blueprint: BP-0001-ORIGIN
  Producer: Genesis-Worker-001
  Function: keyboard_echo
  Type: PROTOCOL
  State: ACTIVE
  Stack: 2048 bytes
  MaxExec: 5 ms
  SideEffect: ['stdout_write']
  Dependency: ['time', 'sys']
```

### 7.6 实际调用验证

```
调用: call_registered_func(1, "商场已激活!")
输出: [KEYBOARD] 商场已激活! | 2026-05-10T...Z
状态: success, displayed=True, source=keyboard, version=2.0.0
```

---

## 八、安全与隔离铁律

### 8.1 商场绝不妥协的红线

1. **不盲信**: AICoder的测试报告仅供参考，商场必须独立执行Step 4-6
2. **不持久**: 沙箱内禁止文件系统写操作，所有数据通过SHM矢量传递
3. **不超预算**: 函数执行严格受 `stack_estimate` 和 `max_execution_ms` 约束，越界立即终止
4. **不重试**: 同一份授权书失败后禁止重试，必须重新提交新蓝图（防止哈希碰撞攻击）
5. **不跳步**: 七步验证必须严格顺序执行，不得跳过任何步骤

### 8.2 Python沙箱安全补充

对于Python语言，由于PYC字节码可被反序列化执行，商场需额外注意：
- PYC加载失败时，允许回退至`source_code.code`的`compile+exec`方式，但必须经过Step 4的静态规则审查
- 沙箱`module_dict`仅注入`__builtins__`，不注入`os`、`subprocess`等危险模块
- 入口函数必须为可调用对象，不得为类实例

### 8.3 审计日志

每一步验证结果必须写入只读审计日志：

```python
{
    "timestamp_ns": int,       # 纳秒时间戳
    "blueprint_id": str,       # 蓝图ID
    "step_name": str,          # 步骤名称
    "result_code": int,        # 结果码
    "detail": str              # 详细信息
}
```

---

## 九、与AICoder外置架构的衔接

由于AICoder外置，商场与AICoder的边界如下：

| 职责 | AICoder（外置） | 商场（内置） |
|------|----------------|-------------|
| 蓝图审查 | ✅ 七维深度审查 | ❌ 不参与 |
| 代码生成 | ✅ 生成源代码 | ❌ 不参与 |
| 编译链接 | ✅ 生成PYC/ELF/SO | ❌ 不参与 |
| 行为测试 | ✅ 生成测试报告 | ❌ 仅供参考 |
| 格式校验 | ❌ 不执行 | ✅ Step 1 |
| 签名验证 | ❌ 不执行 | ✅ Step 2 |
| 哈希校验 | ❌ 不执行 | ✅ Step 3 |
| 静态规则 | ❌ 不执行 | ✅ Step 4（独立二次审查） |
| 沙箱加载 | ❌ 不执行 | ✅ Step 5 |
| 回归测试 | ❌ 不执行 | ✅ Step 6（独立二次验证） |
| 函数注册 | ❌ 不执行 | ✅ Step 7 |

**设计意图**: AICoder负责"生产质量"，商场负责"准入安全"。两者物理隔离，形成制衡。

---

## 十、附录：调用时序图

```
Producer          AICoder           Mall (mall_accept_auth)     Sandbox       Registry
   │                 │                       │                      │              │
   │──蓝图BP-XXXX───▶│                       │                      │              │
   │                 │──七维审查(D1~D7)──────┤                      │              │
   │                 │──代码生成──────────────┤                      │              │
   │                 │──编译PYC/ELF──────────┤                      │              │
   │                 │──行为测试(10+项)───────┤                      │              │
   │                 │──生成授权文件──────────┤                      │              │
   │◀──授权文件──────│                       │                      │              │
   │──auth_json─────────────────────────────▶│                      │              │
   │                                         │──Step1:format()──────┤              │
   │                                         │◀──────OK─────────────│              │
   │                                         │──Step2:sig_verify()──┤              │
   │                                         │◀──────OK─────────────│              │
   │                                         │──Step3:hash_check()──┤              │
   │                                         │◀──────OK─────────────│              │
   │                                         │──Step4:static_rule()─┤              │
   │                                         │◀──────OK─────────────│              │
   │                                         │──Step5:sandbox_load()▶│             │
   │                                         │◀──────handle─────────│              │
   │                                         │──Step6:regression()──▶│             │
   │                                         │◀──────OK─────────────│              │
   │                                         │──Step7:register()────┤─────────────▶│
   │                                         │◀──────func_id────────┤◀─────────────│
   │◀──AUTH_ACCEPTED────────────────────────│                      │              │
```

---

## 十一、产出物清单

本次创世纪验证（BP-0001-ORIGIN）的完整产出物：

| 文件 | 路径 | 说明 |
|------|------|------|
| 蓝图文档 | `键盘生产者蓝图.md` | 生产者提交的契约蓝图 |
| 授权源代码 | `src/workers/keyboard_echo.py` | AICoder精化后的纯函数实现 |
| 授权文件 | `AUTH-BP-0001-ORIGIN.json` | 5字段结构完整授权书 |
| 七步验证流程 | `src/stage5/mall_auth.py` | `mall_accept_auth()` 实现 |
| 单元测试 | `tests/test_keyboard_echo.py` | 18项验收测试 |
| 流程测试 | `tests/test_mall_auth.py` | 七步验证集成测试 |

---

*本文档由AICoder生成，经人类架构师审核通过。所有接口为函数级设计，禁止以任何方式引入OOP结构。v2.0基于创世纪第一张蓝图BP-0001-ORIGIN的实际验证经验修订。*
