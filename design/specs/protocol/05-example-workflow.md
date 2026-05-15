# 完整示例：生产者 → 授权文档 → AICoder → 报告

> **示例定位**：一个贯穿全链路的演示，展示微波TR测试系统中的生产者如何调用 AICoder 完成函数开发。  
> **目标函数**：`pulse_param_extract` —— 脉冲参数提取（Python）。

---

## 一、场景设定

- **生产者**：`producer_tr_01`（微波TR测试系统的信号处理模块生产者）
- **商场节点**：`mall_node_alpha`
- **目标**：为脉冲参数提取功能生成一个高性能、无副作用、OOP-free 的 Python 函数
- **授权范围**：设计(3) + 调试(2) + 评估(3) + 文档(1) + 低风险依赖(1)

---

## 二、Step 1：生产者生成授权文档

```json
{
  "header": {
    "doc_id": "auth-2026-0509-tr01-001",
    "producer_id": "producer_tr_01",
    "mall_node_id": "mall_node_alpha",
    "timestamp_utc": "2026-05-09T21:20:00Z",
    "ttl_seconds": 3600,
    "version": "1.0"
  },
  "mandate": {
    "perm_design": 3,
    "perm_debug": 2,
    "perm_evaluate": 3,
    "perm_document": 1,
    "perm_dependency": 1
  },
  "function_spec": {
    "function_name": "pulse_param_extract",
    "language": "python",
    "signature_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "input_schema": [
      {"name": "iq_samples", "type": "ndarray[complex128]", "shape_hint": "(N,)", "shm_key": "shm://producer_tr_01/iq_vec"},
      {"name": "sample_rate", "type": "float64", "unit": "Hz"}
    ],
    "output_schema": [
      {"name": "pulse_width", "type": "float64", "unit": "s"},
      {"name": "pri", "type": "float64", "unit": "s"},
      {"name": "bandwidth", "type": "float64", "unit": "Hz"}
    ],
    "constraints": {
      "max_latency_ms": 50,
      "max_memory_mb": 128,
      "deterministic": true,
      "side_effect_free": true,
      "oop_forbidden": true
    },
    "context_notes": "微波TR测试系统脉冲参数提取。要求实时处理，禁止动态内存分配。优先使用APFFT算法。"
  },
  "sandbox": {
    "sandbox_level": "standard",
    "shm_read_keys": ["shm://producer_tr_01/iq_vec", "shm://repo/ref_apfft"],
    "shm_write_keys": ["shm://aicoder/{token}/output"],
    "network_access": false,
    "compiler_path": "/opt/mall/compiler/python3_aicoder"
  },
  "signature": "ed25519:sig_content_here..."
}
```

---

## 三、Step 2：商场接收与验签

```c
/* 商场主函数入口 */
int mall_main() {
    /* ... 初始化 SHM、算力池、状态机 ... */

    /* 监听生产者提交通道 */
    struct AuthDocument auth_doc;
    while (mall_poll_producer_channel(&auth_doc) == MALL_OK) {

        char task_token[64];
        struct MallError err;

        /* 调用分发函数 */
        int rc = mall_aicoder_dispatch(
            auth_doc.blob, 
            auth_doc.len,
            TASK_FULL_PIPELINE,  /* DESIGN + DEBUG + EVALUATE */
            task_token,
            sizeof(task_token),
            &err
        );

        if (rc == MALL_OK) {
            mall_log("任务 %s 已注入算力池", task_token);
        } else {
            mall_log_error("任务拒绝: %s", err.message);
            mall_notify_producer(auth_doc.producer_id, err);
        }
    }
}
```

**验签结果**：`AUTH_VERIFIED` → 权限矢量解析通过 → 生成令牌 `aic-7f3a-9e2b`

---

## 四、Step 3：AICoder 执行流水线

### 4.1 DESIGN 阶段

AICoder Worker 读取 SHM 输入矢量，执行 `aicoder_execute_design()`：

```python
# AICoder 生成的函数（示例）
import numpy as np
from typing import Tuple

def pulse_param_extract(iq_samples: np.ndarray, sample_rate: float) -> Tuple[float, float, float]:
    # 基于APFFT的脉冲参数提取。
    # 时间复杂度: O(N log N) —— FFT核心
    # 空间复杂度: O(N) —— 输入规模，无额外动态分配
    N = len(iq_samples)
    if N == 0 or sample_rate <= 0:
        return 0.0, 0.0, 0.0

    # APFFT 预处理：全相位数据重组（无类封装，纯数组操作）
    ap_window = np.ones(N // 2 + 1)  # 矩形全相位窗
    # ... APFFT 核心算法 ...

    # 脉冲检测与参数提取
    pulse_width = 1.2e-6   # 示例值
    pri = 1.0e-3           # 示例值
    bandwidth = 50.0e6     # 示例值

    return pulse_width, pri, bandwidth
```

**OOP 扫描结果**：`PASS`（无 class/self/继承）

### 4.2 DEBUG 阶段

AICoder 自检发现边界问题：当 `iq_samples` 长度为 1 时，`N // 2 + 1` 导致窗口大小为 1，APFFT 退化。

**生成补丁**：
```diff
--- a/pulse_param_extract.py
+++ b/pulse_param_extract.py
@@ -8,6 +8,9 @@ def pulse_param_extract(iq_samples: np.ndarray, sample_rate: float) -> Tuple[
     N = len(iq_samples)
     if N == 0 or sample_rate <= 0:
         return 0.0, 0.0, 0.0
+    if N < 16:
+        # 样本过少，APFFT 无意义，回退到直接测量
+        return _fallback_direct_measure(iq_samples, sample_rate)

     # APFFT 预处理...
```

**根因分析**：APFFT 算法要求最小样本数以保证频谱分辨率，原实现未处理小样本边界。

### 4.3 EVALUATE 阶段

| 维度 | 结果 | 得分 |
|------|------|------|
| 正确性 | 50/50 测试通过（含边界测试） | 39.2/40 |
| 性能 | P99 延迟 38ms（约束 <50ms） | 23.0/25 |
| 安全性 | 0 高危，1 中危（样本率校验建议） | 13.5/15 |
| 可维护性 | 圈复杂度 4 | 7.5/10 |
| OOP 合规 | 零 OOP 语义 | 10.0/10 |
| **总分** | | **92.7/100** |

---

## 五、Step 4：报告生成与交付

商场调用 `mall_generate_aicoder_report()`，生成报告并写入：
- `shm://mall/reports/aic-7f3a-9e2b/report.md`
- `shm://mall/reports/aic-7f3a-9e2b/metadata.json`

**商场裁定**：`CONDITIONAL_ACCEPT` —— 接受设计产出，但要求生产者审核调试补丁后再合并到主分支。

---

## 六、Step 5：生产者接收报告

生产者通过 SHM 读取报告：

```c
/* 生产者侧：报告接收函数 */
int producer_fetch_report(const char* task_token, struct AICoderReport* report) {
    char path[256];
    snprintf(path, sizeof(path), "shm://mall/reports/%s/metadata.json", task_token);

    struct ShmVector* shm_meta = shm_read(path);
    report->score = json_extract_score(shm_meta);
    report->verdict = json_extract_verdict(shm_meta);
    report->artifact_paths = json_extract_artifacts(shm_meta);

    /* 如果 CONDITIONAL_ACCEPT，人工审核补丁 */
    if (report->verdict == CONDITIONAL_ACCEPT) {
        producer_review_patch(report->artifact_paths.patch_diff);
        producer_decide_apply(report);
    }

    return PRODUCER_OK;
}
```

---

## 七、全链路时序图

```
producer_tr_01          mall_node_alpha          AICoder Worker          SHM Report Area
      │                        │                         │                    │
      │  1. submit auth doc    │                         │                    │
      │──────────────────────>│                         │                    │
      │                        │  2. verify & issue token│                    │
      │                        │────────────────────────>│                    │
      │                        │  3. inject task         │                    │
      │                        │────────────────────────>│                    │
      │                        │                         │  4. DESIGN         │
      │                        │                         │────┐               │
      │                        │                         │    │ write code    │
      │                        │                         │<───┘               │
      │                        │                         │  5. DEBUG          │
      │                        │                         │────┐               │
      │                        │                         │    │ find bug      │
      │                        │                         │    │ generate patch│
      │                        │                         │<───┘               │
      │                        │                         │  6. EVALUATE       │
      │                        │                         │────┐               │
      │                        │                         │    │ run tests     │
      │                        │                         │    │ score         │
      │                        │                         │<───┘               │
      │                        │  7. return artifacts    │                    │
      │                        │<────────────────────────│                    │
      │                        │  8. generate report     │                    │
      │                        │─────────────────────────────────────────────>│
      │  9. notify complete    │                         │                    │
      │<──────────────────────│                         │                    │
      │  10. fetch report      │                         │                    │
      │──────────────────────>│                         │                    │
      │                        │  11. read SHM report    │                    │
      │                        │─────────────────────────────────────────────>│
      │  12. return report     │                         │                    │
      │<──────────────────────│                         │                    │
      │                        │                         │                    │
```

---

## 八、审核宣言

> 此示例展示了从"人类意图"到"机器生成"再到"人类审核"的完整闭环。  
> AICoder 不是替代生产者，而是将生产者的意图放大、结构化、可验证。  
> 商场的编译进化权始终在人类架构师手中，AICoder 只是认知的延伸。  
> **审核状态**：待人类架构师最终裁定。
