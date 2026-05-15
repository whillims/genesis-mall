# 设计-调试-评估流水线 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **流水线定位**：AICoder 在商场算力池内的**三阶段生产流水线**。  
> **核心思想**：将 AICoder 的"设计→调试→评估"转化为可观测、可回滚、可审计的状态机管道。  
> **哲学隐喻**：这不是传统的 CI/CD，而是**智能体的认知流水线**——设计是"构想"，调试是"自省"，评估是"审判"。

---

## 一、流水线总体架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   DESIGN    │────→│   DEBUG     │────→│  EVALUATE   │────→│   REPORT    │
│   构想阶段   │     │   自省阶段   │     │   审判阶段   │     │   归档阶段   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │                   │
      ▼                   ▼                   ▼                   ▼
  SHM:design          SHM:patch           SHM:score          SHM:report
  SHM:rationale       SHM:root_cause      SHM:detailed       SHM:audit
```

每个阶段的输出是下一阶段的输入（可选，由 `Mandate` 中的 `perm_*` 决定）。  
阶段间通过**商场调度函数** `mall_pipeline_stage_gate()` 进行质量门禁控制。

---

## 二、DESIGN 阶段：函数构想

### 2.1 输入
- `function_spec`：目标函数签名、输入输出模式、约束条件
- `context_notes`：业务上下文、算法偏好、历史参考代码
- `existing_artifacts`：上一版本代码（如果是迭代设计）

### 2.2 AICoder 内部设计流程

```
需求解析 → 算法选择 → 伪代码生成 → 代码生成 → OOP扫描 → 约束检查 → 输出
```

**子函数分解**（AICoder 内部函数级实现，非 OOP 类方法）：

```c
/* AICoder 内部：设计阶段主控函数 */
int aicoder_design_pipeline(
    const struct DesignInput* input,
    struct DesignOutput* output,
    struct DesignAudit* audit
) {
    int err;
    struct AlgorithmDraft draft;
    struct GeneratedCode code;

    err = aicoder_parse_requirements(input, &draft);      /* 需求解析 */
    if (err) return DESIGN_ERR_REQUIREMENT;

    err = aicoder_select_algorithm(&draft, input->constraints); /* 算法选择 */
    if (err) return DESIGN_ERR_NO_ALGORITHM;

    err = aicoder_generate_pseudocode(&draft, &code);     /* 伪代码 */
    if (err) return DESIGN_ERR_PSEUDOCODE;

    err = aicoder_generate_implementation(&code, input->language); /* 代码生成 */
    if (err) return DESIGN_ERR_GENERATION;

    err = aicoder_oop_scan(&code);                        /* OOP 扫描 */
    if (err) {
        audit->oop_violation_count++;
        return DESIGN_ERR_OOP_VIOLATION;                  /* 致命错误 */
    }

    err = aicoder_constraint_check(&code, input->constraints); /* 约束检查 */
    if (err) return DESIGN_ERR_CONSTRAINT;

    output->code = code;
    return DESIGN_OK;
}
```

### 2.3 输出产物

| 产物 | SHM 键 | 格式 | 说明 |
|------|--------|------|------|
| 函数实现 | `shm://aicoder/{token}/design/code` | 纯文本 | 目标语言的函数代码 |
| 设计依据 | `shm://aicoder/{token}/design/rationale` | Markdown | 算法选择理由、复杂度分析 |
| 依赖清单 | `shm://aicoder/{token}/design/deps` | JSON | 外部依赖及风险等级 |
| 设计审计 | `shm://aicoder/{token}/design/audit` | JSON | 各子步骤耗时、置信度 |

---

## 三、DEBUG 阶段：代码自省

### 3.1 触发条件
DEBUG 阶段有两种触发模式：
1. **流水线模式**：DESIGN 完成后自动进入，AICoder 对自己生成的代码进行自检
2. **独立模式**：生产者提交已有代码 + 错误信息，请求 AICoder 调试

### 3.2 调试流程

```
错误解析 → 代码静态分析 → 根因定位 → 补丁生成 → 补丁验证 → 输出
```

**关键函数**：

```c
/* AICoder 调试阶段主控函数 */
int aicoder_debug_pipeline(
    const struct DebugInput* input,      /* buggy_code + error_trace + test_cases */
    struct DebugOutput* output,
    struct DebugAudit* audit
) {
    struct ErrorProfile profile;
    struct RootCause cause;
    struct Patch patch;

    aicoder_parse_error_trace(input->error_trace, &profile);
    aicoder_static_analyze(input->buggy_code, &profile);
    aicoder_locate_root_cause(input->buggy_code, &profile, &cause);
    aicoder_generate_patch(input->buggy_code, &cause, &patch);
    aicoder_verify_patch(&patch, input->test_cases, &output->patch_valid);

    output->patch = patch;
    output->root_cause = cause;
    return DEBUG_OK;
}
```

### 3.3 调试输出

| 产物 | SHM 键 | 说明 |
|------|--------|------|
| 补丁差异 | `.../debug/patch_diff` | unified diff 格式 |
| 根因分析 | `.../debug/root_cause` | 自然语言 + 代码位置标注 |
| 回归计划 | `.../debug/regression_plan` | 建议补充的测试用例 |
| 调试审计 | `.../debug/audit` | 分析路径、尝试次数 |

**重要约束**：补丁必须通过 `mall_patch_apply()` 由商场审核后应用，AICoder 无直接修改权。

---

## 四、EVALUATE 阶段：质量审判

### 4.1 评估维度与评分算法

评估采用**加权多维评分模型**，总分 100：

```c
struct ScoreCard {
    double correctness;      /* 正确性：测试通过率 × 40 */
    double performance;      /* 性能：归一化后 × 25 */
    double security;         /* 安全性：静态扫描得分 × 15 */
    double maintainability;  /* 可维护性：圈复杂度倒数 × 10 */
    double oop_compliance;   /* OOP合规：布尔值 × 10 */
    double total;            /* 加权总分 */
};

/* 评分计算函数（纯函数，无副作用） */
struct ScoreCard evaluate_score(
    const struct TestResult* tests,
    const struct PerfResult* perf,
    const struct SecurityScan* sec,
    const struct ComplexityMetrics* comp,
    bool oop_clean
) {
    struct ScoreCard s;
    s.correctness = tests->pass_rate * 40.0;
    s.performance = normalize(perf->latency_ms, perf->baseline_ms) * 25.0;
    s.security = sec->score * 15.0;
    s.maintainability = (1.0 / (1.0 + comp->cyclomatic)) * 10.0;
    s.oop_compliance = oop_clean ? 10.0 : 0.0;
    s.total = s.correctness + s.performance + s.security + s.maintainability + s.oop_compliance;
    return s;
}
```

### 4.2 评估流程

```
代码加载 → 语法编译 → 单元测试 → 性能基准 → 安全扫描 → 
    复杂度分析 → OOP扫描 → 评分汇总 → 报告生成
```

**门禁规则**：
- `total < 60`：拒绝，返回生产者重新设计
- `total 60~80`：警告，允许进入但标记为"需优化"
- `total > 80`：通过，可进入 REPORT 阶段
- `oop_compliance == 0`：**一票否决**，无论其他维度得分多高

### 4.3 评估输出

| 产物 | SHM 键 | 说明 |
|------|--------|------|
| 评分卡 | `.../evaluate/score_card` | 结构化 JSON |
| 详细报告 | `.../evaluate/detailed_report` | Markdown，含测试详情、火焰图引用 |
| 改进建议 | `.../evaluate/improvements` | 优先级排序的优化建议 |
| 评估审计 | `.../evaluate/audit` | 各子步骤耗时、环境信息 |

---

## 五、REPORT 阶段：归档与交付

### 5.1 报告聚合函数

```c
/* 商场层：报告聚合函数 */
int mall_generate_aicoder_report(
    const char* task_token,
    const struct DesignOutput* design,
    const struct DebugOutput* debug,
    const struct EvaluateOutput* eval,
    struct FinalReport* report
) {
    report->header = mall_report_header(task_token);
    report->design_section = mall_format_design(design);
    report->debug_section = mall_format_debug(debug);
    report->evaluate_section = mall_format_evaluate(eval);
    report->executive_summary = mall_generate_summary(design, debug, eval);
    report->audit_trail = mall_compile_audit_trail(task_token);

    /* 写入 SHM 报告区 */
    shm_write(report, "shm://mall/reports/{task_token}");
    return REPORT_OK;
}
```

### 5.2 报告结构

```
1. 执行摘要 (Executive Summary)
   - 任务令牌、生产者、目标函数、总评分、通过状态
2. 设计章节 (Design)
   - 函数实现代码、设计依据、依赖清单
3. 调试章节 (Debug)
   - 补丁差异（如有）、根因分析、回归计划
4. 评估章节 (Evaluate)
   - 评分卡、测试详情、性能基准、安全扫描结果
5. 审计追踪 (Audit Trail)
   - 完整时间线、各阶段耗时、AICoder 资源消耗
6. 商场裁定 (Mall Verdict)
   - 商场编译进化权的最终裁定：ACCEPT / REJECT / CONDITIONAL
```

---

## 六、流水线状态机

```
                    ┌─────────────────────────────────────────────┐
                    │                                             │
    ┌───────────┐   ▼   ┌───────────┐   ▼   ┌───────────┐   ▼   ┌───────────┐
    │  IDLE     │──────→│  DESIGN   │──────→│  DEBUG    │──────→│ EVALUATE  │
    └───────────┘       └───────────┘       └───────────┘       └───────────┘
                            │                   │                   │
                            │ FAIL              │ FAIL              │ FAIL
                            ▼                   ▼                   ▼
                        ┌───────────┐       ┌───────────┐       ┌───────────┐
                        │  ABORT    │       │  ABORT    │       │  ABORT    │
                        │ (回滚)    │       │ (回滚)    │       │ (回滚)    │
                        └───────────┘       └───────────┘       └───────────┘
                                                                │
                                                                │ PASS
                                                                ▼
                                                            ┌───────────┐
                                                            │  REPORT   │
                                                            └───────────┘
                                                                │
                                                                ▼
                                                            ┌───────────┐
                                                            │ ARCHIVED  │
                                                            └───────────┘
```

**回滚机制**：任何阶段失败，商场调用 `mall_pipeline_rollback(task_token)`，清空该令牌下的所有 SHM 临时区，释放算力池 Worker，向生产者发送失败通知。

---

## 七、审核宣言

> 此流水线是 AICoder 的**认知工厂**，将不可控的生成式 AI 转化为可观测的工业流程。  
> 设计是创造，调试是批判，评估是审判——三者缺一不可，构成完整的智能体生产闭环。  
> **审核状态**：待人类架构师最终裁定。
