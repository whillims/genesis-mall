# 报告生成层接口深化 v1.0

> **文档定位**：商场-AICoder 体系的**最终物质化层**。  
> **核心原则**：报告不是"输出"，而是**商场编译进化权的决策证据**；报告必须同时服务人类、机器、审计系统三类读者。  
> **哲学隐喻**：报告是 AICoder 工作的"墓碑与勋章"——它既标记了一段智能体劳动的终结，也证明了这段劳动是否值得被商场接纳。

---

## 一、报告生成架构

```
mall_generate_aicoder_report()           [报告聚合主控]
├── mall_report_header()                  [报告头部]
├── mall_report_format_executive_summary() [执行摘要]
├── mall_report_format_design()           [设计章节]
│   ├── mall_report_format_code_block()    [代码块格式化]
│   └── mall_report_format_rationale()    [设计依据格式化]
├── mall_report_format_debug()            [调试章节]
│   ├── mall_report_format_patch()        [补丁格式化]
│   └── mall_report_format_root_cause()   [根因格式化]
├── mall_report_format_evaluate()         [评估章节]
│   ├── mall_report_format_score_card()    [评分卡格式化]
│   ├── mall_report_format_test_details()  [测试详情格式化]
│   └── mall_report_format_security_scan() [安全扫描格式化]
├── mall_report_format_audit()            [审计追踪格式化]
├── mall_report_verdict_recommend()       [裁定建议生成]
└── mall_report_serialize_metadata()      [元数据序列化]
```

---

## 二、报告聚合主控函数

### 2.1 `mall_generate_aicoder_report`

```c
/* 函数：AICoder 执行报告生成主控函数
 * 契约：
 *   前置条件：
 *     - task_token 对应任务已完成（PHASE_COMPLETED 或 PHASE_ABORTED）
 *     - design、debug、evaluate 至少有一个非空（根据 task_type）
 *     - audit 包含完整的审计追踪
 *   后置条件：
 *     - Markdown 报告已写入 shm://mall/reports/{token}/report.md
 *     - metadata.json 已写入 shm://mall/reports/{token}/metadata.json
 *     - 关键产物已复制到 artifacts/ 目录
 *     - out_report_path 包含报告 SHM 路径
 *   副作用：
 *     1. 创建/写入 shm://mall/reports/{token}/report.md
 *     2. 创建/写入 shm://mall/reports/{token}/metadata.json
 *     3. 调用 shm_task_namespace_preserve() 归档产物
 *     4. 写入系统日志：REPORT_GENERATED
 *   并发安全：由商场主控串行执行（每个任务一个报告）
 *   时间复杂度：O(total_artifact_size) —— 数据拷贝和格式化
 *   空间复杂度：O(1) —— 使用固定大小的格式化缓冲区
 */
int mall_generate_aicoder_report(
    const char* task_token,             /* [in] 任务令牌 */
    const struct PipelineArtifacts* artifacts, /* [in] 流水线产出物聚合 */
    const struct ScoreCard* scores,     /* [in] 评分卡 */
    const struct AuditTrail* audit,     /* [in] 审计追踪 */
    char* out_report_path,              /* [out] 报告 SHM 路径缓冲区 */
    u64 path_buf_size,                  /* [in] 缓冲区大小 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* PipelineArtifacts 定义 */
typedef struct {
    const DesignOutput* design;         /* 设计产出（可为 NULL） */
    const DebugOutput* debug;           /* 调试产出（可为 NULL） */
    const EvaluateOutput* evaluate;     /* 评估产出（可为 NULL） */
    AICoderTaskType task_type;          /* 任务类型（决定哪些字段有效） */
} PipelineArtifacts;

/* 执行流程：
 * Step 1: 验证输入
 *         - 检查 task_token 合法性
 *         - 检查 artifacts 与 task_type 一致性
 *         - 若 PHASE_ABORTED，则生成"失败报告"而非"成功报告"
 *
 * Step 2: 创建报告 SHM 区域
 *         shm_vector_create("shm://mall/reports/{token}/report.md", 32KB, &report_shm, &err)
 *         shm_vector_create("shm://mall/reports/{token}/metadata.json", 4KB, &meta_shm, &err)
 *
 * Step 3: 生成 Markdown 报告
 *         调用各格式化子函数，将结果追加到 Markdown 缓冲区
 *         缓冲区满（32KB）→ 截断并添加 "[报告截断，详见元数据]" 标记
 *
 * Step 4: 生成 metadata.json
 *         调用 mall_report_serialize_metadata()，将结构化数据写入 JSON 缓冲区
 *
 * Step 5: 归档产物
 *         shm_task_namespace_preserve(task_token, &final_report, &err)
 *
 * Step 6: 填充输出路径
 *         snprintf(out_report_path, path_buf_size, "shm://mall/reports/%s", task_token)
 *
 * Step 7: 日志记录
 *         mall_log(MALL_LOG_INFO, "REPORT_GENERATED task=%s score=%.2f", task_token, scores->total)
 */
```

**返回值**：
- `MALL_OK` — 报告生成成功
- `MALL_ERR_SHM_ALLOC_FAIL` — SHM 分配失败
- `MALL_ERR_SHM_SERIALIZE_FAIL` — 序列化失败
- `MALL_ERR_UNKNOWN` — 其他错误

---

## 三、报告格式化子函数

### 3.1 `mall_report_header`

```c
/* 函数：生成报告头部
 * 契约：
 *   前置条件：task_token 有效
 *   后置条件：返回 Markdown 格式的报告头部字符串
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1) —— 输出到固定大小缓冲区
 */
struct MarkdownBlock mall_report_header(
    const char* task_token,             /* [in] 任务令牌 */
    u64 timestamp_ns                    /* [in] 报告生成时间戳 */
);

/* 输出格式示例：
 * # AICoder 执行报告
 * 
 * > **任务令牌**: `aic-7f3a-9e2b`  
 * > **生成时间**: 2026-05-09 21:20:13 UTC  
 * > **报告版本**: v1.0  
 * > **商场节点**: mall_node_alpha
 */
```

---

### 3.2 `mall_report_format_executive_summary`

```c
/* 函数：格式化执行摘要
 * 契约：
 *   前置条件：artifacts 和 scores 有效
 *   后置条件：返回 Markdown 格式的执行摘要
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
struct MarkdownBlock mall_report_format_executive_summary(
    const struct PipelineArtifacts* artifacts, /* [in] 流水线产出物 */
    const struct ScoreCard* scores,     /* [in] 评分卡 */
    const struct TaskState* state       /* [in] 任务最终状态 */
);

/* 输出格式示例：
 * ## 1. 执行摘要 (Executive Summary)
 * 
 * | 指标 | 值 | 状态 |
 * |------|-----|------|
 * | 任务令牌 | `aic-7f3a-9e2b` | — |
 * | 目标函数 | `pulse_param_extract` | — |
 * | 总评分 | 92.7/100 | ✅ 通过 |
 * | OOP 合规 | 通过 | ✅ 通过 |
 * | 设计阶段 | 完成 | ✅ |
 * | 调试阶段 | 完成（发现1处问题，已生成补丁） | ⚠️ |
 * | 评估阶段 | 完成 | ✅ |
 * | 商场建议 | **CONDITIONAL_ACCEPT** | 需人工审核 |
 * | 执行耗时 | 4.2s | — |
 * | Token 消耗 | 15,234 | — |
 * 
 * ### 1.1 评分雷达
 * ```
 * 正确性    [████████████████████░░░░] 39.2/40
 * 性能      [███████████████████░░░░░] 23.0/25
 * 安全性    [█████████████████░░░░░░░] 13.5/15
 * 可维护性  [████████████░░░░░░░░░░░░] 7.5/10
 * OOP合规   [████████████████████] 10.0/10
 * ```
 */
```

---

### 3.3 `mall_report_format_design`

```c
/* 函数：格式化设计章节
 * 契约：
 *   前置条件：design 非 NULL；design->generated_code 非空
 *   后置条件：返回 Markdown 格式的设计章节
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(code_length)
 *   空间复杂度：O(1) —— 输出受缓冲区限制，代码截断到 4KB 显示
 */
struct MarkdownBlock mall_report_format_design(
    const struct DesignOutput* design,  /* [in] 设计产出 */
    const struct FunctionSpec* spec      /* [in] 函数规范（用于验证签名一致性） */
);

/* 输出格式示例：
 * ## 3. 设计产出 (Design Artifacts)
 * 
 * ### 3.1 函数实现
 * 
 * ```python
 * def pulse_param_extract(iq_samples: np.ndarray, sample_rate: float) -> tuple[float, float, float]:
 *     # 基于APFFT的脉冲参数提取。
 *     # 时间复杂度: O(N log N)
 *     # 空间复杂度: O(N)
 *     N = len(iq_samples)
 *     if N == 0 or sample_rate <= 0:
 *         return 0.0, 0.0, 0.0
 *     # ... (截断，完整代码见 artifacts/code.py)
 * ```
 * 
 * ### 3.2 设计依据
 * - **算法选择**: APFFT（All-Phase FFT）
 * - **复杂度**: 时间 O(N log N)，空间 O(N)
 * - **约束满足**: 延迟 38ms < 50ms ✅ | 内存 64MB < 128MB ✅
 * 
 * ### 3.3 依赖清单
 * | 依赖 | 版本 | 风险等级 |
 * |------|------|----------|
 * | numpy | 1.24.0 | 低风险 ✅ |
 * 
 * ### 3.4 置信度
 * AICoder 设计置信度: **0.94**
 */
```

---

### 3.4 `mall_report_format_debug`

```c
/* 函数：格式化调试章节
 * 契约：
 *   前置条件：debug 非 NULL
 *   后置条件：返回 Markdown 格式的调试章节
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(patch_length)
 *   空间复杂度：O(1)
 */
struct MarkdownBlock mall_report_format_debug(
    const struct DebugOutput* debug,    /* [in] 调试产出 */
    const struct AICoderMetrics* metrics /* [in] 调试阶段资源统计 */
);

/* 输出格式示例：
 * ## 4. 调试产出 (Debug Artifacts)
 * 
 * ### 4.1 问题描述
 * **类型**: 边界条件缺陷  
 * **位置**: 第 12 行  
 * **现象**: 当 `iq_samples` 长度 < 16 时，APFFT 窗口退化，频谱分辨率不足
 * 
 * ### 4.2 根因分析
 * APFFT 算法要求最小样本数 N >= 16 以保证全相位窗的有效覆盖。原实现未处理小样本边界，
 * 导致 `np.ones(N // 2 + 1)` 在 N=1 时生成长度为 1 的窗口，失去全相位特性。
 * 
 * ### 4.3 补丁差异
 * ```diff
 * --- a/pulse_param_extract.py
 * +++ b/pulse_param_extract.py
 * @@ -8,6 +8,9 @@ def pulse_param_extract(...)
 *      N = len(iq_samples)
 *      if N == 0 or sample_rate <= 0:
 *          return 0.0, 0.0, 0.0
 * +    if N < 16:
 * +        # 样本过少，APFFT 无意义，回退到直接测量
 * +        return _fallback_direct_measure(iq_samples, sample_rate)
 * ```
 * 
 * ### 4.4 补丁验证
 * - **验证状态**: ✅ 通过
 * - **回归测试**: 50/50 通过（含新增边界测试）
 * 
 * ### 4.5 回归测试建议
 * - 增加测试用例: `iq_samples` 长度为 1, 2, 15, 16 的边界测试
 * - 增加测试用例: `sample_rate` 为 0 和负值的鲁棒性测试
 */
```

---

### 3.5 `mall_report_format_evaluate`

```c
/* 函数：格式化评估章节
 * 契约：
 *   前置条件：evaluate 非 NULL；scores 有效
 *   后置条件：返回 Markdown 格式的评估章节
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
struct MarkdownBlock mall_report_format_evaluate(
    const struct EvaluateOutput* evaluate, /* [in] 评估产出 */
    const struct ScoreCard* scores        /* [in] 评分卡 */
);

/* 输出格式示例：
 * ## 5. 评估结果 (Evaluation Results)
 * 
 * ### 5.1 多维评分卡
 * 
 * | 维度 | 权重 | 原始得分 | 加权得分 | 详情 |
 * |------|------|----------|----------|------|
 * | 正确性 | 40% | 98% | 39.2 | 50/50 测试通过 |
 * | 性能 | 25% | 92% | 23.0 | P99 38ms < 50ms |
 * | 安全性 | 15% | 90% | 13.5 | 0 高危，1 中危 |
 * | 可维护性 | 10% | 75% | 7.5 | 圈复杂度 4 |
 * | OOP 合规 | 10% | 100% | 10.0 | 零 OOP 语义 |
 * | **总分** | **100%** | — | **92.7** | **✅ 通过** |
 * 
 * ### 5.2 测试详情
 * - **单元测试**: 50 组，全部通过
 * - **边界测试**: 8 组，全部通过
 * - **性能基准**: 1000 次迭代，P99=38ms，P50=35ms，均值=36.2ms
 * 
 * ### 5.3 安全扫描
 * - **扫描工具**: mall_security_scanner_v2
 * - **高危**: 0
 * - **中危**: 1 —— 建议增加 `sample_rate > 0` 校验（已在补丁中修复）
 * - **低危**: 0
 * 
 * ### 5.4 改进建议
 * 1. **[低优先级]** 考虑使用 `np.float64` 显式类型转换，避免平台差异
 * 2. **[低优先级]** 增加函数文档字符串（docstring）以提升可维护性
 */
```

---

### 3.6 `mall_report_format_audit`

```c
/* 函数：格式化审计追踪章节
 * 契约：
 *   前置条件：audit 非 NULL；audit->count > 0
 *   后置条件：返回 Markdown 格式的审计追踪
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(audit->count)
 *   空间复杂度：O(1) —— 最多显示 64 条事件，超出截断
 */
struct MarkdownBlock mall_report_format_audit(
    const struct AuditTrail* audit      /* [in] 审计追踪 */
);

/* 输出格式示例：
 * ## 6. 审计追踪 (Audit Trail)
 * 
 * | 时间戳 (UTC) | 事件 | 阶段 | 详情 |
 * |--------------|------|------|------|
 * | 21:20:01.123 | TASK_RECEIVED | — | 任务进入商场队列 |
 * | 21:20:01.456 | AUTH_VERIFIED | — | 授权文档验签通过，权限矢量 [3,2,3,1,1] |
 * | 21:20:02.001 | DESIGN_START | DESIGN | 开始函数设计，Worker PID=12345 |
 * | 21:20:04.567 | OOP_SCAN_PASSED | DESIGN | 内部 OOP 扫描通过，0 违规 |
 * | 21:20:05.234 | DESIGN_COMPLETE | DESIGN | 代码生成完成，置信度 0.94 |
 * | 21:20:05.345 | DEBUG_START | DEBUG | 自检调试启动 |
 * | 21:20:06.789 | BUG_DETECTED | DEBUG | 发现边界条件缺陷，行号 12 |
 * | 21:20:07.012 | PATCH_GENERATED | DEBUG | 补丁生成完成，验证通过 |
 * | 21:20:07.345 | DEBUG_COMPLETE | DEBUG | 调试阶段完成 |
 * | 21:20:07.456 | EVAL_START | EVALUATE | 质量评估启动 |
 * | 21:20:09.012 | TESTS_PASSED | EVALUATE | 全部 50 组测试通过 |
 * | 21:20:09.234 | EVAL_COMPLETE | EVALUATE | 评估完成，总分 92.7 |
 * | 21:20:09.345 | REPORT_GENERATED | REPORT | 报告生成完成 |
 * 
 * ### 6.1 资源消耗
 * - **总耗时**: 8.222s
 * - **Token 消耗**: 15,234
 * - **内存峰值**: 128MB
 * - **Worker PID**: 12345
 */
```

---

### 3.7 `mall_report_verdict_recommend`

```c
/* 函数：生成商场裁定建议
 * 契约：
 *   前置条件：scores 有效；audit 完整
 *   后置条件：返回 MallVerdict 枚举值及理由说明
 *   副作用：无
 *   并发安全：纯函数
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
struct VerdictRecommendation mall_report_verdict_recommend(
    const struct ScoreCard* scores,     /* [in] 评分卡 */
    const struct DebugOutput* debug,    /* [in] 调试产出（可为 NULL） */
    const struct AuditTrail* audit      /* [in] 审计追踪 */
);

/* VerdictRecommendation 定义 */
typedef struct {
    MallVerdict verdict;                /* RECOMMEND_REJECT / CONDITIONAL / ACCEPT */
    char rationale[1024];               /* 建议理由 */
    char risk_level[16];                /* "LOW" / "MEDIUM" / "HIGH" */
    char suggested_action[512];         /* 建议操作 */
} VerdictRecommendation;

/* 裁定算法（纯函数）：
 * 
 * IF scores->oop_compliance < 10.0
 *     → VERDICT_REJECT, "OOP 合规一票否决", "HIGH", "重新设计，确保零 OOP 语义"
 * 
 * ELSE IF scores->total < 60.0
 *     → VERDICT_REJECT, "总分低于通过线", "HIGH", "重新设计或降低约束"
 * 
 * ELSE IF scores->total >= 80.0 AND debug == NULL
 *     → VERDICT_ACCEPT, "全部通过，无调试问题", "LOW", "直接合并到主分支"
 * 
 * ELSE IF scores->total >= 80.0 AND debug != NULL AND debug->patch_valid == MALL_TRUE
 *     → VERDICT_CONDITIONAL, "高分通过但含调试补丁", "LOW", "人工审核补丁后合并"
 * 
 * ELSE IF scores->total >= 60.0 AND scores->total < 80.0
 *     → VERDICT_CONDITIONAL, "通过但需优化", "MEDIUM", "按改进建议优化后重新评估"
 * 
 * ELSE
 *     → VERDICT_REJECT, "未满足基本质量要求", "HIGH", "重新设计"
 */
```

---

## 四、元数据序列化函数

### 4.1 `mall_report_serialize_metadata`

```c
/* 函数：将报告结构化为 JSON 元数据
 * 契约：
 *   前置条件：report 为有效填充的 FinalReport；buffer 容量 >= 4KB
 *   后置条件：buffer 包含有效的 JSON 字符串；out_len 包含实际长度
 *   副作用：无外部写入（纯格式化）
 *   并发安全：纯函数
 *   时间复杂度：O(1) —— 固定大小结构体
 *   空间复杂度：O(1)
 */
int mall_report_serialize_metadata(
    const struct FinalReport* report,   /* [in] 最终报告 */
    char* buffer,                      /* [out] JSON 输出缓冲区 */
    u64 buffer_size,                   /* [in] 缓冲区大小 */
    u64* out_len,                      /* [out] 实际 JSON 长度 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* JSON 输出格式（严格模式，无换行缩进以节省空间）：
 * {
 *   "report_version": "1.0",
 *   "task_token": "aic-7f3a-9e2b",
 *   "producer_id": "producer_tr_01",
 *   "function_name": "pulse_param_extract",
 *   "language": "python",
 *   "status": "COMPLETED",
 *   "score_card": {
 *     "correctness": 39.2,
 *     "performance": 23.0,
 *     "security": 13.5,
 *     "maintainability": 7.5,
 *     "oop_compliance": 10.0,
 *     "total": 92.7
 *   },
 *   "phases": {
 *     "design": {"status": "COMPLETED", "duration_ms": 3000, "confidence": 0.94},
 *     "debug": {"status": "COMPLETED", "issues_found": 1, "duration_ms": 2000, "patch_valid": true},
 *     "evaluate": {"status": "COMPLETED", "duration_ms": 1500, "tests_passed": 50, "tests_total": 50}
 *   },
 *   "oop_clean": true,
 *   "verdict_recommendation": "CONDITIONAL_ACCEPT",
 *   "risk_level": "LOW",
 *   "total_duration_ms": 8222,
 *   "tokens_consumed": 15234,
 *   "artifact_paths": {
 *     "report_md": "shm://mall/reports/aic-7f3a-9e2b/report.md",
 *     "metadata_json": "shm://mall/reports/aic-7f3a-9e2b/metadata.json",
 *     "generated_code": "shm://mall/reports/aic-7f3a-9e2b/artifacts/code.py",
 *     "patch_diff": "shm://mall/reports/aic-7f3a-9e2b/artifacts/patch.diff",
 *     "test_results": "shm://mall/reports/aic-7f3a-9e2b/artifacts/test_results.json"
 *   }
 * }
 */
```

---

## 五、MarkdownBlock 类型定义

```c
/* Markdown 文本块 —— 报告格式化函数的返回类型 */
typedef struct {
    char content[16384];                /* Markdown 内容（16KB 上限） */
    u64 length;                         /* 实际内容长度 */
    mall_bool truncated;                /* 是否被截断 */
} MarkdownBlock;

/* MarkdownBlock 操作函数 */
MarkdownBlock markdown_block_init(void);                                    /* 初始化空块 */
int markdown_block_append(MarkdownBlock* block, const char* text);        /* 追加文本 */
int markdown_block_appendf(MarkdownBlock* block, const char* fmt, ...);     /* 格式化追加 */
int markdown_block_append_block(MarkdownBlock* dest, const MarkdownBlock* src); /* 追加另一个块 */
```

---

## 六、审核宣言

> 报告生成层是 AICoder 劳动的**最终审判庭**。  
> 它不做创造，只做聚合与格式化——将分散在 SHM 各区域的产出物汇聚成一份可审计、可决策、可归档的证据。  
> 报告的价值不在于它的长度，而在于它的**结构化程度**——人类能在 30 秒内读懂执行摘要，机器能在 10 毫秒内解析 metadata.json，审计系统能在 5 分钟内追溯完整操作链。  
> 没有报告的 AICoder 调用是**非法劳动**——商场有权拒绝为任何未生成报告的任务支付算力配额。  
> **审核状态**：待人类架构师最终裁定。
