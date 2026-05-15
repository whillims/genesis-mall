# AICoder 调用协议 v1.0

> **协议定位**：商场主控程序与 AICoder 智能体之间的**函数级调用契约**。  
> **核心原则**：AICoder 是商场的"外脑"，不是独立主体；所有交互通过显式函数调用完成，禁止任何对象封装与隐式状态共享。

---

## 一、协议拓扑：三层函数调用链

```
┌─────────────────────────────────────────────────────────────┐
│                      商场主控层 (Mall Main)                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  mall_aicoder_dispatch(auth_doc, task_vector)        │   │
│  │   → 验签 → 解析授权 → 生成任务令牌 → 路由到算力池     │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  算力进程池适配层 (Pool Adapter)                      │   │
│  │  aicoder_task_inject(token, function_spec, shm_cfg)│   │
│  │   → 环境隔离 → SHM 挂载 → 启动 AICoder Worker        │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AICoder 执行层 (AICoder Worker)                      │   │
│  │  aicoder_execute_design(shm_input, shm_output)       │   │
│  │  aicoder_execute_debug(shm_input, shm_output)        │   │
│  │  aicoder_execute_evaluate(shm_input, shm_output)      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心接口函数定义

### 2.1 商场主控层：`mall_aicoder_dispatch`

```c
/* 商场主控层：AICoder 任务分发函数
 * 输入：授权文档字节流 + 任务类型枚举
 * 输出：任务令牌 + 错误码
 * 副作用：修改 SHM 任务队列状态矢量
 */
int mall_aicoder_dispatch(
    const uint8_t* auth_doc_blob,      /* 授权文档序列化字节流 */
    size_t auth_doc_len,               /* 文档长度 */
    enum AICoderTaskType task_type,    /* DESIGN | DEBUG | EVALUATE | FULL_PIPELINE */
    char* out_task_token,              /* 输出：任务令牌字符串 (64字节缓冲区) */
    size_t token_buf_size,             /* 缓冲区大小 */
    struct MallError* out_err          /* 输出：错误详情结构体 */
);
```

**执行流程**：
1. 调用 `mall_auth_verify(auth_doc_blob, auth_doc_len)` 验签
2. 调用 `mall_auth_parse_mandate(auth_doc_blob)` 提取权限矢量
3. 检查 `task_type` 是否在权限矢量允许范围内
4. 生成 UUID 作为 `task_token`
5. 将任务注入 `shm://mall/task_queue/aicoder_pending`
6. 返回 `MALL_OK` 或错误码（`MALL_ERR_AUTH_FAIL`, `MALL_ERR_PERM_DENIED`, `MALL_ERR_POOL_FULL`）

---

### 2.2 算力进程池适配层：`aicoder_task_inject`

```c
/* 算力池适配层：将任务注入隔离 Worker
 * 此函数由商场调度器调用，非 AICoder 直接调用
 */
int aicoder_task_inject(
    const char* task_token,
    const struct FunctionSpec* func_spec,   /* 解析后的函数规范 */
    const struct SandboxConfig* sandbox_cfg, /* 沙箱配置 */
    const struct ShmVectorConfig* shm_cfg,  /* SHM 读写键配置 */
    pid_t* out_worker_pid                    /* 输出：分配的 Worker 进程 ID */
);
```

**隔离机制**：
- 使用 `clone()` 或 `unshare()` 创建新的 PID/Network/Mount Namespace
- SHM 区域通过只读/只写映射精确控制（`mmap` 带 `PROT_READ` / `PROT_WRITE` 标志）
- 编译器路径硬编码为 `/opt/mall/compiler/aicoder_allowed`，AICoder 不可覆盖

---

### 2.3 AICoder 执行层：三阶段函数

#### 阶段 A：函数设计 `aicoder_execute_design`

```c
/* AICoder 设计阶段
 * 输入：需求描述 + 约束条件（通过 SHM 矢量传递）
 * 输出：函数实现代码 + 依赖声明 + 设计依据（写入 SHM 输出区）
 */
int aicoder_execute_design(
    const struct ShmVector* shm_in,   /* 输入矢量：function_spec + context_notes */
    struct ShmVector* shm_out,        /* 输出矢量：generated_code + design_rationale */
    struct AICoderMetrics* metrics    /* 输出：token消耗、耗时、置信度 */
);
```

**设计约束检查清单**（AICoder 内部必须执行）：
- [ ] 代码中不存在 `class`、`struct`（C 语言除外，但 C 的 struct 不得含函数指针模拟虚表）、`self`、`__init__` 等 OOP 关键字
- [ ] 所有变量在函数作用域内定义，无全局状态修改
- [ ] 输入输出严格符合 `FunctionSpec` 的 `input_schema` / `output_schema`
- [ ] 包含复杂度注释（时间/空间复杂度）

#### 阶段 B：代码调试 `aicoder_execute_debug`

```c
/* AICoder 调试阶段
 * 输入：待调试代码 + 错误日志/测试失败信息
 * 输出：修复补丁 + 根因分析 + 回归测试建议
 */
int aicoder_execute_debug(
    const struct ShmVector* shm_in,   /* 输入：buggy_code + error_trace + test_cases */
    struct ShmVector* shm_out,        /* 输出：patch_diff + root_cause + regression_plan */
    struct AICoderMetrics* metrics
);
```

**调试铁律**：AICoder **不得直接修改生产代码**。它只能生成 `patch_diff`，由商场的 `mall_patch_apply()` 函数在审核后应用。

#### 阶段 C：质量评估 `aicoder_execute_evaluate`

```c
/* AICoder 评估阶段
 * 输入：函数代码 + 测试用例集 + 评估维度配置
 * 输出：多维评分 + 详细报告 + 改进建议
 */
int aicoder_execute_evaluate(
    const struct ShmVector* shm_in,   /* 输入：code_under_test + test_suite + eval_config */
    struct ShmVector* shm_out,        /* 输出：score_card + detailed_report + improvement_suggestions */
    struct AICoderMetrics* metrics
);
```

**评估维度**（可配置）：
| 维度 | 权重 | 评估方法 |
|------|------|----------|
| 正确性 | 40% | 单元测试通过率 |
| 性能 | 25% | 基准测试（latency, throughput） |
| 安全性 | 15% | 静态扫描（禁止函数、缓冲区溢出风险） |
| 可维护性 | 10% | 圈复杂度、函数长度、注释覆盖率 |
| OOP 合规 | 10% | 语法树扫描，确保零 OOP 语义 |

---

## 三、SHM 矢量通信规范

AICoder 与商场之间的一切数据交换通过**命名 SHM 矢量**完成，禁止管道、Socket、文件等 side-channel。

### 3.1 输入矢量格式 `shm://aicoder/{task_token}/input`

```json
{
  "task_type": "DESIGN",
  "function_spec": { /* 同授权文档 FunctionSpec */ },
  "context": {
    "existing_code_refs": ["shm://repo/func_a", "shm://repo/func_b"],
    "test_cases": [/* 输入/输出对 */],
    "constraints": { /* 性能/安全约束 */ }
  },
  "previous_artifacts": null /* 或上一阶段的输出引用 */
}
```

### 3.2 输出矢量格式 `shm://aicoder/{task_token}/output`

```json
{
  "status": "COMPLETED | FAILED | PARTIAL",
  "artifacts": {
    "generated_code": "...",
    "design_rationale": "...",
    "patch_diff": "...",
    "score_card": { /* 评估分数 */ }
  },
  "metrics": {
    "tokens_consumed": 15234,
    "execution_time_ms": 4500,
    "confidence_score": 0.92
  },
  "audit_trail": [
    {"action": "DESIGN_START", "timestamp": "..."},
    {"action": "OOP_SCAN_PASSED", "timestamp": "..."},
    {"action": "CODE_GENERATED", "timestamp": "..."}
  ]
}
```

### 3.3 状态矢量 `shm://aicoder/{task_token}/state`

用于商场轮询或事件通知：
```json
{
  "current_phase": "DESIGN | DEBUG | EVALUATE | REPORTING",
  "phase_progress": 0.75,
  "last_heartbeat": "2026-05-09T21:20:00Z",
  "worker_pid": 12345
}
```

---

## 四、错误码体系

| 错误码 | 常量 | 含义 | 处理建议 |
|--------|------|------|----------|
| 0 | `MALL_OK` | 成功 | — |
| 101 | `MALL_ERR_AUTH_FAIL` | 授权文档验签失败 | 拒绝服务，记录安全事件 |
| 102 | `MALL_ERR_PERM_DENIED` | 权限不足 | 返回生产者，提示升级授权 |
| 103 | `MALL_ERR_POOL_FULL` | 算力池满载 | 进入队列等待或拒绝 |
| 201 | `AICODER_ERR_DESIGN_FAIL` | 设计阶段失败 | 返回部分结果 + 失败原因 |
| 202 | `AICODER_ERR_DEBUG_FAIL` | 调试阶段失败 | 可能无法生成有效补丁 |
| 203 | `AICODER_ERR_EVAL_FAIL` | 评估阶段失败 | 通常因测试环境异常 |
| 301 | `AICODER_ERR_OOP_DETECTED` | 生成代码含 OOP 语义 | **强制回滚**，记录违规 |
| 302 | `AICODER_ERR_SHM_VIOLATION` | SHM 访问越界 | 立即终止 Worker，隔离审查 |

**错误码 301 的特殊处理**：一旦 AICoder 生成包含 `class`、`self`、继承等 OOP 结构的代码，商场的 `mall_oop_scanner()` 函数立即触发**强制回滚**，清空该任务的所有 SHM 输出区，并向生产者发送**违规通知**。累计 3 次违规的生产者，永久吊销 AICoder 接入资格。

---

## 五、审核宣言

> 本协议确保 AICoder 始终作为商场的**外部认知扩展**，而非独立行为体。  
> 所有接口均为纯函数，所有状态通过 SHM 显式传递，所有权限通过授权文档预先限定。  
> **审核状态**：待人类架构师最终裁定。
