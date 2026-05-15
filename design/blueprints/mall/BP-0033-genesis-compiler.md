# 蓝图文档（定稿版）：BP-0033 -- 基因编译器

> **创世纪2.0-一阶段核心交付物 | 基因规则引擎生成器**
>
> *函数级设计 | 零OOP | SHM唯一血脉 | 人类审核至上*

---

## 1. 蓝图元数据

| 字段 | 值 |
|------|-----|
| 蓝图ID | `BP-0033-GENESIS-COMPILER` |
| 蓝图名称 | 基因编译器（Genesis Compiler） |
| 生产者 | `Genesis-Worker-001` |
| 提交时间 | `2026-05-11` |
| 版本 | `1.0.0` |
| 目标语言 | Python |
| 复杂度等级 | `Level-1`（核心基础设施级） |
| 设计范式 | 纯函数级，禁止OOP封装 |

**创世纪定位**：本蓝图是创世纪2.0-一阶段（GENESIS-COMPILER）的核心交付物。基因编译器将37个规则基因从自然语言宪章转化为可自动验证的规则引擎，是2.0七阶段路线图的起点。它是"基因即命运，编译即创世"理念的工程实现。

**关联蓝图**：本蓝图依赖以下1.0已激活蓝图提供的运行时服务：
- BP-0003（SHM矢量管理器）：编译器状态存储与五域交互
- BP-0004（异常处理体系）：编译器错误处理
- BP-0009（进程注册表）：编译器进程注册
- BP-0011（心跳生产者）：编译器心跳上报
- BP-0013（日志生产者）：编译器审计日志

---

## 2. 功能概述

### 2.1 功能描述

基因编译器是一个**规则引擎生成器**，不是传统意义上的代码编译器。它的输入是基因定义（自然语言宪章文件 + 结构化元数据），输出是可执行的铁律验证管线。

核心处理流程：解析宪章文件 → 提取结构化基因定义 → 分类约束强度 → 构建依赖DAG → 编码为可执行规则（断言/守卫/契约/审计） → 执行验证管线（静态分析/动态测试/突变测试） → 输出综合验证报告至SHM审计域。

### 2.2 数据来源

| 来源 | 类型 | 锁定状态 | 说明 |
|------|------|----------|------|
| 宪章文件（constitution-*.md） | 文件系统 | 来源锁定 | 20份宪章文件，基因的唯一来源 |
| 基因元数据（gene_metadata.json） | SHM bp域 | 来源锁定 | 基因ID、名称、约束强度的结构化描述 |
| SHM bus域事件 | SHM bus域 | 可配置 | 订阅"新宪章提交"事件触发增量编译 |

### 2.3 数据去向

| 去向 | 类型 | 说明 |
|------|------|------|
| SHM vec域 | 编译进度矢量、内存使用统计、管线队列深度 |
| SHM hb域 | 编译器心跳（5秒周期），报告当前阶段和健康状态 |
| SHM bp域 | 输出产物：GeneDef列表、DAG结构、编码产物、验证报告 |
| SHM auth域 | 编译器自身授权令牌验证（只读） |
| SHM bus域 | 发布编译阶段转换、基因编码完成、验证完成等事件 |

### 2.4 模块架构

基因编译器由5个核心模块、14个函数组成：

```
genesis_compiler/
├── module_gene_parser/          # 基因解析模块（3函数）
│   ├── parse_constitution       # 解析宪章文件，提取GeneDef
│   ├── extract_rule_statements  # 从GeneDef提取结构化规则陈述
│   └── validate_gene_structure  # 验证GeneDef结构完整性
├── module_constraint_classifier/ # 约束分类模块（2函数）
│   ├── classify_constraint_level # 四级约束强度分类
│   └── check_human_review_required # 判断是否需要人类审核
├── module_dependency_builder/    # 依赖构建模块（3函数）
│   ├── parse_dependency_refs     # 解析基因间依赖引用
│   ├── build_dependency_dag      # 构建有向无环依赖图
│   └── verify_dag_acyclic        # 验证DAG无循环依赖
├── module_rule_encoder/          # 规则编码模块（3函数）
│   ├── encode_to_assertion       # 绝对铁律 → 不可绕过断言
│   ├── encode_to_guard           # 强约束 → 运行时守卫函数
│   └── encode_to_contract        # 宪法级 → 接口契约文本
└── module_verification/          # 验证模块（3函数）
    ├── run_static_analysis       # 静态分析（蓝图代码合规扫描）
    ├── run_dynamic_tests         # 动态测试（运行时铁律触发验证）
    └── run_mutation_tests        # 突变测试（注入违规验证捕获能力）
```

---

## 3. 函数设计

### 3.1 主入口函数签名

```python
def genesis_compiler_init(ctx: dict, config: dict) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_WRITE_SHM
    初始化基因编译器，创建SHM运行态矢量，注册bus域事件订阅。
    """

def genesis_compiler_run(ctx: dict, state: dict) -> dict:
    """
    [复杂度]: O(G * R) 其中G=基因数量(37), R=平均规则数
    [安全等级]: READ_WRITE_SHM
    执行完整编译管线：解析→分类→DAG→编码→验证。
    """
```

### 3.2 模块一：gene_parser 函数签名

```python
def parse_constitution(file_path: str, ctx: dict) -> dict:
    """
    [复杂度]: O(n) n=文件行数
    [安全等级]: READ_ONLY_SHM
    解析单个宪章文件，提取GeneDef结构。
    """

def extract_rule_statements(gene_def: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(r) r=规则陈述数量
    [安全等级]: READ_ONLY_SHM
    从GeneDef的core_content中提取结构化规则陈述列表。
    """

def validate_gene_structure(gene_def: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_ONLY_SHM
    验证GeneDef结构的字段完整性和格式合规性。
    """
```

### 3.3 模块二：constraint_classifier 函数签名

```python
def classify_constraint_level(rule_stmt: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(k) k=关键词数量
    [安全等级]: READ_ONLY_SHM
    对单条规则陈述进行四级约束强度分类。
    """

def check_human_review_required(classified_rule: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(1)
    [安全等级]: READ_ONLY_SHM
    判断分类后的规则是否需要提交人类审核。
    """
```

### 3.4 模块三：dependency_builder 函数签名

```python
def parse_dependency_refs(gene_def_list: list, ctx: dict) -> dict:
    """
    [复杂度]: O(G * D) G=基因数, D=平均依赖数
    [安全等级]: READ_ONLY_SHM
    解析所有GeneDef中的依赖引用关系。
    """

def build_dependency_dag(gene_def_list: list, ctx: dict) -> dict:
    """
    [复杂度]: O(G + E) G=节点数, E=边数
    [安全等级]: READ_ONLY_SHM
    从GeneDef列表构建依赖有向无环图，执行Kahn拓扑排序。
    """

def verify_dag_acyclic(dag: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(G + E)
    [安全等级]: READ_ONLY_SHM
    验证DAG无循环依赖，返回验证结果。
    """
```

### 3.5 模块四：rule_encoder 函数签名

```python
def encode_to_assertion(rule_stmt: dict, target_lang: str, ctx: dict) -> dict:
    """
    [复杂度]: O(m) m=规则文本长度
    [安全等级]: READ_ONLY_SHM
    将绝对铁律编码为不可绕过断言（Python assert / C static_assert）。
    """

def encode_to_guard(rule_stmt: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(m)
    [安全等级]: READ_ONLY_SHM
    将强约束编码为运行时守卫函数。
    """

def encode_to_contract(rule_stmt: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(m)
    [安全等级]: READ_ONLY_SHM
    将宪法级规则编码为接口契约文本。
    """
```

### 3.6 模块五：verification 函数签名

```python
def run_static_analysis(blueprint_code: str, rule_engine: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(n * R) n=代码行数, R=规则数
    [安全等级]: READ_ONLY_SHM
    对蓝图代码执行静态合规扫描。
    """

def run_dynamic_tests(blueprint_code: str, rule_engine: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(T * R) T=测试用例数, R=规则数
    [安全等级]: READ_ONLY_SHM
    执行运行时铁律触发验证。
    """

def run_mutation_tests(blueprint_code: str, rule_engine: dict, ctx: dict) -> dict:
    """
    [复杂度]: O(M * R) M=突变体数(100), R=规则数
    [安全等级]: READ_ONLY_SHM
    执行突变测试，验证规则引擎捕获能力，目标kill_ratio > 90%。
    """
```

### 3.7 参数规格表

**parse_constitution 参数**：

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `file_path` | `str` | 是 | 长度 <= 512，必须以`.md`结尾 | 宪章文件绝对路径 |
| `ctx` | `dict` | 是 | 必须包含shm_handle、audit_domain、parser_config | 编译上下文 |

**classify_constraint_level 参数**：

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `rule_stmt` | `dict` | 是 | 必须包含rule_text、keywords、scope字段 | 规则陈述结构 |
| `ctx` | `dict` | 是 | 非空 | 编译上下文 |

**build_dependency_dag 参数**：

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `gene_def_list` | `list` | 是 | 非空，元素必须为dict | GeneDef列表 |
| `ctx` | `dict` | 是 | 非空 | 编译上下文 |

**run_mutation_tests 参数**：

| 参数名 | 类型 | 必填 | 约束条件 | 说明 |
|--------|------|------|----------|------|
| `blueprint_code` | `str` | 是 | 非空 | 被验证的蓝图代码 |
| `rule_engine` | `dict` | 是 | 必须包含encoded_rules列表 | 编码后的规则引擎 |
| `ctx` | `dict` | 是 | 非空 | 编译上下文 |

### 3.8 返回值规格

所有函数返回标准dict结构：

```
{
    "status": "ok" | "error",
    "displayed": str | bool,
    "timestamp": str,           # ISO 8601 UTC
    "source": str,              # 函数名（固定值）
    "metadata": dict,           # 函数特定元数据
    "error": "" | {"code": str, "message": str}
}
```

**parse_constitution metadata 扩展字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `gene_id` | `str` | 提取的基因ID，格式GENE-XXXX |
| `gene_name` | `str` | 基因名称 |
| `source_charter` | `str` | 来源宪章文件名 |
| `constraint_level` | `str` | 约束强度分类 |
| `dependencies` | `list` | 依赖的基因ID列表 |
| `extracted_rules` | `list` | 提取的规则陈述列表 |
| `line_count` | `int` | 宪章文件行数 |
| `rule_count` | `int` | 提取的规则数量 |

**build_dependency_dag metadata 扩展字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `dag_nodes` | `list` | DAG节点列表 |
| `dag_edges` | `list` | DAG边列表 |
| `topological_order` | `list` | 拓扑排序后的基因ID列表 |
| `levels` | `dict` | 按层级分组的基因ID |
| `cycles_detected` | `list` | 若存在循环，列出涉及的基因ID |

**run_mutation_tests metadata 扩展字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `mutants_created` | `int` | 突变体总数（目标100） |
| `mutants_killed` | `int` | 被规则引擎捕获的突变体数 |
| `kill_ratio` | `float` | 捕获率（目标 > 0.90） |
| `survivors` | `list` | 未被捕获的突变体详情 |

### 3.9 数据来源定义（SHM交互）

| 矢量域 | 数据类型 | 访问模式 | 说明 |
|--------|----------|----------|------|
| vec | dict | 读写 | 编译进度矢量、内存使用统计、管线队列深度 |
| hb | dict | 只写 | 编译器心跳（5秒周期） |
| bp | list/dict | 只写 | 输出产物：GeneDef、DAG、编码产物、验证报告 |
| auth | str | 只读 | 编译器自身授权令牌 |
| bus | dict | 读写 | 订阅/发布事件 |

---

## 4. 行为契约

### 4.1 前置条件

| 编号 | 前置条件 | 违反时错误码 | 返回status |
|------|----------|-------------|------------|
| PRE-001 | file_path 存在且为.md文件 | `FILE_NOT_FOUND` | error |
| PRE-002 | 宪章文件包含"宪章名称"文件头 | `INVALID_CHARTER_FORMAT` | error |
| PRE-003 | 宪章文件中可识别到至少一个基因标识 | `NO_GENE_DETECTED` | error |
| PRE-004 | ctx 包含 shm_handle、audit_domain、parser_config | `INVALID_CONTEXT` | error |
| PRE-005 | gene_def_list 非空 | `EMPTY_GENE_LIST` | error |
| PRE-006 | rule_stmt 包含 rule_text、keywords、scope 字段 | `INVALID_RULE_STMT` | error |
| PRE-007 | blueprint_code 非空 | `EMPTY_BLUEPRINT_CODE` | error |
| PRE-008 | rule_engine 包含 encoded_rules 列表 | `INVALID_RULE_ENGINE` | error |
| PRE-009 | 编译器授权令牌有效（auth域验证通过） | `AUTH_TOKEN_INVALID` | error |
| PRE-010 | DAG无循环依赖 | `CYCLE_DETECTED` | error |

### 4.2 后置条件

| 编号 | 后置条件 | 说明 |
|------|----------|------|
| POST-001 | status 必须为 "ok" 或 "error" | 不允许其他值 |
| POST-002 | 成功时 metadata 包含 gene_id 或 dag_nodes 或 mutants_killed | 根据函数类型返回对应数据 |
| POST-003 | 失败时 error 包含 code 和 message | 错误信息必须结构化 |
| POST-004 | 所有SHM写入操作仅限于授权的五域 | 不得越域写入 |
| POST-005 | 审计追踪记录已追加到 audit_trail | 每次操作可追溯 |
| POST-006 | 编码产物 bypass_impossible 字段对绝对铁律为 true | 绝对铁律不可绕过 |

### 4.3 不变量

- 零全局状态：编译器内部不读写任何全局变量，所有状态通过ctx参数传递
- 零堆分配：编译过程中的临时缓冲区使用预分配SHM池
- 零系统调用：除SHM读写外不进行文件IO、网络IO、进程管理
- 零副作用：不修改输入参数，不产生外部可观察效果（除返回值和SHM写入）
- 确定性：相同输入必得相同输出
- 拓扑顺序不变：给定相同的GeneDef列表，拓扑排序结果确定
- 审计不可篡改：audit_trail 只追加不修改

### 4.4 约束分类规则

classify_constraint_level 基于关键词匹配和置信度评分：

| 关键词 | 候选级别 | 权重 |
|--------|----------|------|
| "绝对铁律"、"禁止"、"永不"、"不可" | absolute | 1.0 |
| "宪法级"、"根本"、"基石"、"主权" | constitutional | 0.9 |
| "强约束"、"必须"、"强制"、"严格" | strong | 0.7 |
| "中等约束"、"建议"、"推荐"、"应当" | moderate | 0.4 |

分类逻辑：取最高得分级别为候选；若最高得分 < 0.5 标记为 "uncertain" 强制人类审核；absolute 和 constitutional 级别 human_review_required 强制为 true。

### 4.5 突变算子定义

| 算子ID | 算子名称 | 操作 | 目标基因 |
|--------|----------|------|----------|
| M-01 | OOP注入 | 在代码中插入class定义 | 函数原子基因 |
| M-02 | 全局变量注入 | 插入 global x 语句 | 三大零原则 |
| M-03 | Socket使用 | 插入 import socket | SHM唯一血脉 |
| M-04 | 堆分配注入 | 插入 malloc() 调用 | 三大零原则 |
| M-05 | 返回值篡改 | 删除 status 字段 | 蓝图标准结构 |
| M-06 | 依赖伪造 | 修改依赖DAG中的边 | 基因依赖链 |
| M-07 | 约束降级 | 将 assert 改为 warning | 绝对铁律 |
| M-08 | 时间戳伪造 | 返回非法时间格式 | 时间契约基因 |
| M-09 | 双向流动 | 消费者向生产者写入 | 单向流动基因 |
| M-10 | SHM越界 | 写入超出vec域边界 | SHM污染熔断 |

每种算子生成10个突变体，共100个。目标 kill_ratio > 90%。

### 4.6 错误契约表

| 错误码 | 触发条件 | 返回status | 返回displayed | 返回error |
|--------|----------|------------|---------------|-----------|
| `FILE_NOT_FOUND` | 宪章文件不存在 | error | False | 文件路径不存在 |
| `INVALID_CHARTER_FORMAT` | 文件头格式非法 | error | False | 宪章文件缺少标准文件头 |
| `NO_GENE_DETECTED` | 未识别到基因标识 | error | False | 宪章文件中未检测到基因定义 |
| `INVALID_CONTEXT` | ctx缺少必要字段 | error | False | 编译上下文缺少必要字段 |
| `EMPTY_GENE_LIST` | GeneDef列表为空 | error | False | 基因定义列表不能为空 |
| `INVALID_RULE_STMT` | 规则陈述结构不完整 | error | False | 规则陈述缺少必填字段 |
| `EMPTY_BLUEPRINT_CODE` | 蓝图代码为空 | error | False | 蓝图代码不能为空 |
| `INVALID_RULE_ENGINE` | 规则引擎结构不完整 | error | False | 规则引擎缺少编码规则 |
| `AUTH_TOKEN_INVALID` | 授权令牌无效 | error | False | 编译器授权令牌验证失败 |
| `CYCLE_DETECTED` | DAG存在循环依赖 | error | False | 检测到循环依赖，涉及基因ID列表 |
| `ENCODING_FAILED` | 规则编码失败 | error | False | 规则编码过程中发生错误 |
| `VERIFICATION_FAILED` | 验证管线失败 | error | False | 验证管线报告失败 |
| `MUTATION_RATIO_LOW` | 突变捕获率低于阈值 | error | False | kill_ratio 低于90%阈值 |

---

## 5. 测试场景（验收标准）

### 5.1 测试分层

| 层级 | 名称 | 范围 | 数量 |
|------|------|------|------|
| Layer 1 | 单函数隔离测试 | 每个函数独立测试 | 14 x 2 = 28 |
| Layer 2 | 模块内集成测试 | 模块内函数组合 | 5 x 3 = 15 |
| Layer 3 | 模块间交互测试 | 跨模块数据流 | 10 |
| Layer 4 | 端到端编译管线测试 | 完整编译流程 | 8 |
| Layer 5 | 压力与边界测试 | 极端输入、并发 | 10 |
| 专项 | 突变测试 | 规则引擎捕获能力 | 100 |
| **合计** | | | **~200** |

### 5.2 关键测试用例

| 编号 | 类别 | 输入 | 配置 | 预期输出 |
|------|------|------|------|----------|
| TC-001 | 正常 | constitution-function-paradigm.md | 默认 | gene_id="GENE-0001", constraint_level="absolute", extracted_rules 非空 |
| TC-002 | 正常 | 37个宪章文件批量解析 | 默认 | 37个GeneDef全部提取成功，0个错误 |
| TC-003 | 边界 | 规则文本"禁止OOP" | 默认 | constraint_level="absolute", confidence > 0.95 |
| TC-004 | 边界 | 规则文本"建议函数返回值包含metadata字段" | 默认 | constraint_level="moderate", human_review_required=False |
| TC-005 | 异常 | 格式错误的宪章文件（缺少基因标识） | 默认 | status="error", code="NO_GENE_DETECTED" |
| TC-006 | 异常 | 不存在的文件路径 | 默认 | status="error", code="FILE_NOT_FOUND" |
| TC-007 | 异常 | GeneA→GeneB→GeneC→GeneA 循环依赖 | 默认 | status="error", cycles_detected 非空 |
| TC-008 | 正常 | 绝对铁律规则 + target_lang="python" | 默认 | generated_code 包含 assert，bypass_impossible=True |
| TC-009 | 正常 | 强约束规则 | 默认 | generated_code 为守卫函数，包含 status/error 字段 |
| TC-010 | 异常 | 缺少 status 字段的函数返回值 + 守卫函数 | 默认 | 守卫函数返回 error，code="MISSING_STATUS_FIELD" |
| TC-011 | 正常 | 包含 class MyClass 的蓝图代码 + 静态分析 | 默认 | violations_found >= 1, overall_status="failed" |
| TC-012 | 正常 | M-01算子注入10个突变体 + 突变测试 | 默认 | mutants_killed >= 9, kill_ratio >= 0.9 |
| TC-013 | 正常 | 全部10种算子100个突变体 | 默认 | kill_ratio > 0.90, survivors 列表非空则需人工分析 |
| TC-014 | 正常 | 完整编译管线（37基因→编码→验证） | 默认 | overall_status="passed", 200项测试全部通过 |
| TC-015 | 性能 | 37个宪章文件批量编译 | 默认 | 总耗时 < 5000ms |
| TC-016 | 边界 | 空文件（0字节） | 默认 | status="error", code="INVALID_CHARTER_FORMAT" |
| TC-017 | 边界 | 超长规则文本（10000字符） | 默认 | 正常处理，不截断不溢出 |
| TC-018 | 异常 | ctx 缺少 shm_handle 字段 | 默认 | status="error", code="INVALID_CONTEXT" |
| TC-019 | 正常 | 人类审核标记验证（absolute级规则） | 默认 | human_review_required=True |
| TC-020 | 正常 | SHM五域写入验证 | 默认 | vec/hb/bp/bus 域均有写入，auth域仅读取 |

### 5.3 input_builder 机制

| 构造器 | 用途 | 示例 |
|--------|------|------|
| `repeat` | 重复单字符生成超长规则文本 | `{"type":"repeat","char":"禁","count":5000}` |
| `repeat_text` | 重复文本生成大规模宪章 | `{"type":"repeat_text","text":"禁止OOP。","count":1000}` |
| `whitespace` | 空白字符注入测试 | `{"type":"whitespace","value":"   \n\t  "}` |
| `null_byte` | 空字节注入测试 | `{"type":"null_byte","base":"禁止OOP","position":2}` |

---

## 6. 依赖与资源声明

### 6.1 标准库依赖

| 模块 | 用途 | 版本要求 |
|------|------|----------|
| `re` | 正则表达式匹配（基因标识/关键词提取） | Python 3.8+ |
| `hashlib` | SHA-256校验（审计哈希、报告防篡改） | Python 3.8+ |
| `json` | JSON序列化/反序列化（元数据/报告） | Python 3.8+ |
| `time` | 时间戳生成（ISO 8601） | Python 3.8+ |
| `collections` | deque（拓扑排序Kahn算法） | Python 3.8+ |
| `copy` | 深拷贝（DAG构建防引用污染） | Python 3.8+ |

### 6.2 外部依赖声明

**零外部依赖**：基因编译器不依赖任何第三方库。

### 6.3 系统资源约束

| 资源 | 约束 | 说明 |
|------|------|------|
| CPU | O(G * R) 单次编译 | G=37基因, R=平均规则数 |
| 内存 | <= 512KB | 编译器运行态（含ctx、DAG、编码产物缓冲） |
| 栈帧 | <= 16KB | 最大递归深度受拓扑排序迭代限制 |
| 执行时限 | <= 5000ms | 37基因完整编译管线 |
| SHM占用 | <= 2MB | 五域运行态矢量 + 输出产物 |

---

## 7. 安全与隔离铁律

### 7.1 安全约束检查清单

| 约束项 | 声明 | 说明 |
|--------|------|------|
| 文件系统零IO | ✅ | 编译器仅在初始化时读取宪章文件（由商场授权），运行时零文件IO |
| 网络零IO | ✅ | 不进行任何网络通讯，IPC全部通过SHM五域 |
| 零全局副作用 | ✅ | 所有状态通过ctx参数传递，无全局变量 |
| 输入消毒 | ✅ | 所有输入进行类型/长度/格式边界校验 |
| 确定性输出 | ✅ | 相同输入产生相同输出（除时间戳字段外） |
| SHM权限合规 | ✅ | 仅访问授权的五域矢量，不越域读写 |
| 栈帧安全 | ✅ | 无递归，最大迭代深度受基因数量限制 |
| 零堆分配 | ✅ | 临时缓冲区使用预分配SHM池 |

### 7.2 特殊安全声明

**文件读取特许**：parse_constitution 函数需要在初始化阶段读取宪章文件（.md），这是基因编译器的核心输入源。此文件读取操作在商场授权范围内执行，读取路径由auth域令牌白名单限定。

**人类审核至上铁律**：
- absolute 和 constitutional 级别规则的编码产物必须标记 human_review_required=True
- 以下决策永远不可由编译器自动完成：基因约束强度降级、三界隔离原则放宽、SHM唯一血脉替代方案、黑名单移除、新基因加入
- 审核日志写入SHM审计域（bus域特殊分区），采用只追加写入，任何修改尝试触发SHM污染熔断

**不可让渡清单**（写入编译器铁律，任何AI越权尝试自动熔断）：
1. 任何基因的约束强度降级
2. 三界隔离原则的任何形式的放宽
3. SHM唯一血脉原则的替代方案批准
4. 黑名单的移除
5. 新基因的加入（必须通过人类七维审查）

### 7.3 编译器自身铁律

基因编译器自身必须接受与1.0相同的铁律审查：
- 编译器全部使用纯函数编写，禁止OOP
- 编译器接受函数原子基因断言扫描
- 编译器接受三大零原则验证
- 编译器自身的授权文件必须通过商场七步验证

---

## 8. 授权文件生成要求（致外置AICoder）

### 8.1 契约审查指令

依据本蓝图第4节行为契约进行逐条验证：
- 验证全部13项前置条件及其错误码映射
- 验证全部6项后置条件的返回值结构
- 验证7项不变量在所有函数中的遵守
- 验证约束分类规则的四级关键词匹配逻辑
- 验证10种突变算子的注入和捕获逻辑

### 8.2 代码生成指令

将蓝图意图转化为纯函数级Python实现：
- 5个模块14个函数，全部纯函数，零OOP
- 所有状态通过ctx参数传递
- 标准返回值结构（status/displayed/timestamp/source/metadata/error）
- _make_ctx() 工厂函数创建初始上下文
- _ok_result() / _error_result() 标准返回构造器
- Kahn算法实现拓扑排序
- 正则表达式实现基因标识/关键词提取

### 8.3 测试覆盖指令

基于第5节测试场景生成验收测试用例：
- 200项测试，覆盖5层 + 专项突变测试
- 20个关键测试用例（TC-001~TC-020）必须全部实现
- 10种突变算子每种10个突变体，共100个
- input_builder 机制支持4种构造器
- 测试文件命名：test_genesis_compiler.py

### 8.4 安全扫描指令

验证三大零原则、SHM权限、输入消毒：
- 扫描全部14个函数，确认零全局变量、零class定义、零import os/subprocess/socket
- 验证SHM五域访问权限合规
- 验证文件读取仅在parse_constitution中且路径受auth域白名单限定
- 验证不可让渡清单已硬编码为编译器铁律

### 8.5 授权文件输出

按5字段JSON结构生成授权文件：
- auth_header：蓝图ID=BP-0033-GENESIS-COMPILER，生产者=Genesis-Worker-001
- design_meta：14个函数签名、输入输出schema、37个基因编码映射、13个错误码
- source_code：纯函数级Python实现
- compiled_binary：PYC编译产物（base64编码）
- verification_report：200项测试报告 + 100项突变测试报告

### 8.6 基因-宪章-编码映射表

| 基因ID | 基因名称 | 来源宪章 | 约束级别 | 编码格式 | 注入点 |
|--------|----------|----------|----------|----------|--------|
| GENE-0001 | 函数原子基因 | function-paradigm.md | absolute | assertion | module_init |
| GENE-0002 | SHM唯一血脉 | shm-vector-space.md | absolute | assertion | module_init |
| GENE-0003 | 三界隔离 | authorization-generation.md | absolute | assertion | module_init |
| GENE-0004 | 三大零原则 | authorization-generation.md | absolute | guard | runtime_check |
| GENE-0005 | 人类审核至上 | function-paradigm.md | absolute | assertion | function_entry |
| GENE-0006 | 主权分离 | mall-constitution-comprehensive.md, law-mall-identity.md | constitutional | guard | runtime_check |
| GENE-0007 | 四角色权力 | mall-intent-transparency.md | constitutional | contract | interface_def |
| GENE-0008 | 三方信任 | consumer-law-trilateral.md | constitutional | contract | interface_def |
| GENE-0009 | 蓝图标准结构 | blueprint-standard.md | strong | guard | function_entry |
| GENE-0010 | SHM六域宪法 | shm-vector-space.md, law-mall-identity.md | strong | guard | runtime_check |
| GENE-0011 | 消费者五维画像 | consumer-profile-general.md | strong | contract | interface_def |
| GENE-0012 | 任务生命周期 | law-task-charter.md | strong | guard | runtime_check |
| GENE-0013 | 时间契约 | law-time-contract.md, law-mall-identity.md | strong | guard | runtime_check |
| GENE-0014 | 单向流动与路径分级 | law-trade-mask.md, law-mall-identity.md | strong | assertion | function_entry |
| GENE-0015 | 意图匹配 | consumer-profile-intent.md | strong | contract | interface_def |
| GENE-0016 | 双向契约批准 | task-flow-manifesto.md | strong | guard | runtime_check |
| GENE-0017 | 熔断自愈 | consumer-law-circuit-breaker.md | strong | guard | runtime_check |
| GENE-0018 | 编译进化 | function-paradigm.md | strong | assertion | module_init |
| GENE-0019 | 信用体系 | consumer-law-time.md | moderate | contract | interface_def |
| GENE-0020 | 私利检测 | mall-intent-transparency.md | absolute | guard | runtime_check |
| GENE-0021 | SHM与产品污染熔断 | shm-vector-space.md, law-product.md | absolute | guard | runtime_check |
| GENE-0022 | 意图透明 | mall-intent-transparency.md | strong | contract | interface_def |
| GENE-0023 | 静默拒绝 | authorization-generation.md | strong | guard | runtime_check |
| GENE-0024 | 黑名单 | mall-intent-transparency.md | absolute | assertion | module_init |
| GENE-0025 | 单向漏斗 | mall-constitution-comprehensive.md | absolute | assertion | function_entry |
| GENE-0026 | 安全三重防线 | mall-constitution-comprehensive.md | strong | guard | runtime_check |
| GENE-0027 | 控制型安全 | consumer-profile-taxonomy.md | absolute | guard | runtime_check |
| GENE-0028 | 约束服务双柱 | law-mall-identity.md | constitutional | contract | interface_def |
| GENE-0029 | 交易路径分级 | law-mall-identity.md, law-trade-mask.md | strong | guard | runtime_check |
| GENE-0030 | 交易指标体系 | law-mall-identity.md | strong | guard | runtime_check |
| GENE-0031 | 主动建议能力 | law-mall-identity.md | strong | guard | runtime_check |
| GENE-0032 | 产出速度调节 | law-mall-identity.md | strong | guard | runtime_check |
| GENE-0033 | 产品本体定义 | law-product.md | constitutional | contract | interface_def |
| GENE-0034 | 产品生命周期 | law-product.md | absolute | assertion | function_entry |
| GENE-0035 | 产品类型签名 | law-product.md | strong | guard | runtime_check |
| GENE-0036 | 产品所有权 | law-product.md | constitutional | contract | interface_def |
| GENE-0037 | 产品相UI解耦 | law-product.md | constitutional | contract | interface_def |

---

## 附录A：编译器SHM状态机

```
INIT → PARSING → CLASSIFYING → BUILDING_DAG → ENCODING → VERIFYING → COMPLETED
 ↑                                                                  ↓
 └────────────────── ERROR（任何阶段出错回退到INIT）──────────────────┘
```

## 附录B：事件定义

**订阅事件**：
- `event.charter.submitted`：新宪章文件提交，触发增量编译
- `event.gene.mutation_request`：人类审核员请求基因变异，触发重新编译

**发布事件**：
- `event.compiler.phase_changed`：编译阶段转换通知
- `event.compiler.gene_encoded`：单个基因编码完成通知
- `event.compiler.verification_complete`：验证管线完成通知
- `event.compiler.error`：编译错误通知

## 附录C：文档版本记录

| 版本 | 日期 | 说明 | 审核状态 |
|------|------|------|----------|
| 1.0.0 | 2026-05-11 | 初稿，整合基因编译器详细设计文档与2.0远景行动纲领 | 待审核 |

---

*本蓝图遵循创世纪项目蓝图标准格式总则v1.0，八节结构齐全。*
*所有设计均为函数级，零OOP，符合函数原子基因绝对铁律。*
*基因编译器是创世纪2.0的起点，37个规则基因的可执行化是文明级跃迁的第一步。*
