# AICoder 报告输出规范 v1.0

> **规范定位**：AICoder 完成设计-调试-评估流水线后，向商场与生产者交付的**正式报告标准**。  
> **设计原则**：报告不仅是结果的陈列，更是**商场编译进化权的决策依据**；报告必须结构化、可审计、可机器解析。

---

## 一、报告哲学

报告是 AICoder 工作的**最终物质化产物**。它承载着三层价值：
1. **信息层**：发生了什么、生成了什么、测出了什么
2. **证据层**：为什么可信、如何验证、边界在哪
3. **决策层**：商场是否应该接纳这份代码进入生产环境

报告必须同时服务三类读者：
- **人类架构师**：快速理解核心结论（执行摘要）
- **商场编译器**：提取结构化元数据（JSON 区块）
- **安全审计系统**：追溯完整操作链（审计追踪）

---

## 二、报告文件结构

### 2.1 物理存储

报告存储于商场的**报告持久化区**，路径规范：
```
shm://mall/reports/{task_token}/report.md       # 人类可读主报告
shm://mall/reports/{task_token}/metadata.json   # 机器解析元数据
shm://mall/reports/{task_token}/artifacts/      # 附件目录（代码、补丁、日志）
```

### 2.2 Markdown 主报告结构

```markdown
# AICoder 执行报告

## 1. 执行摘要 (Executive Summary)
## 2. 授权与约束 (Authorization & Constraints)
## 3. 设计产出 (Design Artifacts)
## 4. 调试产出 (Debug Artifacts)
## 5. 评估结果 (Evaluation Results)
## 6. 审计追踪 (Audit Trail)
## 7. 商场裁定建议 (Mall Verdict Recommendation)
## 8. 附录 (Appendices)
```

---

## 三、分章节规范

### 3.1 执行摘要

必须包含以下**关键指标卡片**（KPI Card）：

```markdown
| 指标 | 值 | 状态 |
|------|-----|------|
| 任务令牌 | `aic-7f3a-9e2b` | — |
| 目标函数 | `pulse_param_extract` | — |
| 总评分 | 87/100 | ✅ 通过 |
| OOP 合规 | 通过 | ✅ 通过 |
| 设计阶段 | 完成 | ✅ |
| 调试阶段 | 完成（发现1处潜在问题，已生成补丁） | ⚠️ |
| 评估阶段 | 完成 | ✅ |
| 商场建议 | **CONDITIONAL_ACCEPT** | 需人工审核调试补丁 |
| 执行耗时 | 4.2s | — |
| Token 消耗 | 15,234 | — |
```

### 3.2 授权与约束

复述授权文档的核心约束，证明 AICoder 在限定范围内工作：
- 权限矢量摘要
- 沙箱级别
- SHM 访问范围
- 函数签名哈希比对结果

### 3.3 设计产出

**代码展示规范**：
- 使用围栏代码块，标注语言
- 函数签名必须与 `FunctionSpec` 完全一致
- 代码后附**设计依据**（Design Rationale），解释关键决策

**示例**：
```markdown
### 3.3.1 函数实现

```python
def pulse_param_extract(iq_samples: np.ndarray, sample_rate: float) -> tuple[float, float, float]:
    # 脉冲参数提取函数。
    # 时间复杂度: O(N)
    # 空间复杂度: O(1) 除输入输出外
    # APFFT 算法实现...
    return pulse_width, pri, bandwidth
```

### 3.3.2 设计依据
- **算法选择**：采用 APFFT（All-Phase FFT）而非传统 FFT，因为在脉冲边缘检测场景下，APFFT 的频谱泄漏更低，符合微波TR测试的精度要求。
- **无动态分配**：预分配固定大小的缓冲区，避免实时系统中的内存碎片。
- **纯函数设计**：输入 `iq_samples` 为只读，返回值通过元组显式传递，无副作用。
```

### 3.4 调试产出

如果 DEBUG 阶段被触发且发现问题：
- **问题描述**：错误现象、触发条件
- **根因分析**：定位到具体代码行 + 逻辑缺陷解释
- **补丁差异**：使用 `diff` 格式
- **回归测试建议**：建议新增的测试用例

### 3.5 评估结果

**评分卡展示**：

```markdown
### 3.5.1 多维评分

| 维度 | 权重 | 原始得分 | 加权得分 | 详情 |
|------|------|----------|----------|------|
| 正确性 | 40% | 98% | 39.2 | 50/50 测试通过 |
| 性能 | 25% | 85% | 21.25 | 延迟 42ms，基准 50ms |
| 安全性 | 15% | 90% | 13.5 | 无高危警告 |
| 可维护性 | 10% | 80% | 8.0 | 圈复杂度 3 |
| OOP 合规 | 10% | 100% | 10.0 | 零 OOP 语义检测 |
| **总分** | **100%** | — | **91.95** | **通过** |

### 3.5.2 测试详情
- **单元测试**：50 组，全部通过
- **边界测试**：空输入、单样本、极大数组，全部通过
- **性能基准**：1000 次迭代，P99 延迟 45.2ms，满足 <50ms 约束

### 3.5.3 安全扫描
- 静态扫描工具：`mall_security_scanner_v2`
- 高危问题：0
- 中危问题：1（建议对 `sample_rate` 增加正值校验）
- 低危问题：0
```

### 3.6 审计追踪

以时间线形式记录所有关键事件：

```markdown
| 时间戳 (UTC) | 事件 | 阶段 | 详情 |
|--------------|------|------|------|
| 21:20:01 | TASK_RECEIVED | — | 任务进入商场队列 |
| 21:20:02 | AUTH_VERIFIED | — | 授权文档验签通过 |
| 21:20:03 | DESIGN_START | DESIGN | 开始函数设计 |
| 21:20:05 | OOP_SCAN_PASSED | DESIGN | OOP 扫描通过 |
| 21:20:06 | DESIGN_COMPLETE | DESIGN | 代码生成完成 |
| 21:20:07 | DEBUG_START | DEBUG | 自检调试启动 |
| 21:20:08 | BUG_DETECTED | DEBUG | 发现边界条件缺陷 |
| 21:20:09 | PATCH_GENERATED | DEBUG | 补丁生成完成 |
| 21:20:10 | EVAL_START | EVALUATE | 质量评估启动 |
| 21:20:12 | TESTS_PASSED | EVALUATE | 全部测试通过 |
| 21:20:13 | REPORT_GENERATED | REPORT | 报告生成完成 |
```

### 3.7 商场裁定建议

AICoder 不拥有裁定权，但可以基于评估结果给出**建议**：

```markdown
> **AICoder 建议**：`CONDITIONAL_ACCEPT`
> 
> **理由**：总评分 91.95 超过通过线（80），OOP 合规满分。但调试阶段发现一处边界条件缺陷，虽已生成补丁，建议商场编译进化权持有者审核该补丁后再合并。
> 
> **风险等级**：低
> **建议操作**：人工审核 `patch_diff` 后，调用 `mall_patch_apply()`
```

---

## 四、metadata.json 机器解析规范

与 Markdown 报告并行生成，供商场自动化系统消费：

```json
{
  "report_version": "1.0",
  "task_token": "aic-7f3a-9e2b",
  "producer_id": "producer_7",
  "function_name": "pulse_param_extract",
  "language": "python",
  "status": "COMPLETED",
  "score_card": {
    "correctness": 39.2,
    "performance": 21.25,
    "security": 13.5,
    "maintainability": 8.0,
    "oop_compliance": 10.0,
    "total": 91.95
  },
  "phases": {
    "design": {"status": "COMPLETED", "duration_ms": 3000},
    "debug": {"status": "COMPLETED", "issues_found": 1, "duration_ms": 2000},
    "evaluate": {"status": "COMPLETED", "duration_ms": 1500}
  },
  "oop_clean": true,
  "verdict_recommendation": "CONDITIONAL_ACCEPT",
  "risk_level": "LOW",
  "artifact_paths": {
    "generated_code": "shm://mall/reports/aic-7f3a-9e2b/artifacts/code.py",
    "patch_diff": "shm://mall/reports/aic-7f3a-9e2b/artifacts/patch.diff",
    "test_results": "shm://mall/reports/aic-7f3a-9e2b/artifacts/test_results.json"
  }
}
```

---

## 五、报告生成函数签名

```c
/* 商场层：报告生成主控函数 */
int mall_report_generate(
    const char* task_token,
    const struct PipelineArtifacts* artifacts,
    const struct ScoreCard* scores,
    const struct AuditTrail* audit,
    char* out_report_path,              /* 输出：报告 SHM 路径 */
    size_t path_buf_size
);

/* 商场层：报告格式化子函数 */
struct MarkdownBlock mall_report_format_executive_summary(const struct PipelineArtifacts* a);
struct MarkdownBlock mall_report_format_design(const struct DesignOutput* d);
struct MarkdownBlock mall_report_format_debug(const struct DebugOutput* d);
struct MarkdownBlock mall_report_format_evaluate(const struct EvaluateOutput* e);
struct MarkdownBlock mall_report_format_audit(const struct AuditTrail* a);
```

---

## 六、审核宣言

> 报告是 AICoder 工作的**墓碑与勋章**——它既标记了一段智能体劳动的终结，也证明了这段劳动是否值得被商场接纳。  
> 没有报告的设计是盲动，没有评分的报告是空谈，没有审计的评分是欺骗。  
> **审核状态**：待人类架构师最终裁定。
