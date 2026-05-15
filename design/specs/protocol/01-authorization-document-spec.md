# 商场生产者授权文档规范 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档定位**：生产者向商场申请 AICoder 智能体服务的唯一合法凭证。  
> **设计原则**：物理隔离、编译权分离、最小权限原则、函数级原子授权。  
> **哲学基石**：商场拥有函数的**编译进化权**，AICoder 仅拥有**设计生成权**，生产者拥有**需求定义权**——三权分立，防止智能体自利。

---

## 一、授权文档拓扑结构

授权文档不是简单的"申请表"，而是生产者与商场之间的**契约式状态机**。它定义了 AICoder 可以触碰的边界，以及商场必须守护的禁区。

```
┌─────────────────────────────────────────────┐
│          Authorization Document             │
│              (契约状态机载体)                  │
├─────────────────────────────────────────────┤
│  1. Header       → 元数据与身份锚定           │
│  2. Mandate      → 授权范围与权限矢量         │
│  3. FunctionSpec → 目标函数原子定义            │
│  4. Sandbox      → 安全沙箱与SHM边界           │
│  5. Signature    → 生产者数字签名与商场验签     │
└─────────────────────────────────────────────┘
```

---

## 二、Header：元数据与身份锚定

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `doc_id` | UUID | 是 | 全局唯一授权文档标识 |
| `producer_id` | String | 是 | 生产者唯一身份标识 |
| `mall_node_id` | String | 是 | 目标商场节点标识 |
| `timestamp_utc` | ISO-8601 | 是 | 文档生成时间戳 |
| `ttl_seconds` | Integer | 是 | 授权有效期，默认 3600s |
| `version` | String | 是 | 规范版本，当前 `1.0` |

**设计铁律**：`producer_id` 必须与商场注册表中的 SHM 身份令牌绑定，防止伪造生产者身份发起 AICoder 调用。

---

## 三、Mandate：授权范围与权限矢量

授权采用**权限矢量**模型，而非布尔开关。每个维度代表 AICoder 的一种能力，值为 0~3 的整数：

- `0` — 禁止（Forbidden）
- `1` — 只读/建议（Advisory）
- `2` — 生成/设计（Generative）
- `3` — 生成+评估+调试（Full）

| 权限维度 | 字段名 | 含义 |
|----------|--------|------|
| 函数设计 | `perm_design` | 生成函数体、接口定义、依赖声明 |
| 代码调试 | `perm_debug` | 分析运行时错误、生成修复补丁 |
| 质量评估 | `perm_evaluate` | 执行静态分析、性能测试、安全扫描 |
| 文档生成 | `perm_document` | 生成设计文档、API 文档、注释 |
| 依赖引入 | `perm_dependency` | 允许引入外部库/模块的最大风险等级 |

**示例权限矢量**：
```json
{
  "perm_design": 3,
  "perm_debug": 2,
  "perm_evaluate": 3,
  "perm_document": 1,
  "perm_dependency": 1
}
```
> 解读：允许 AICoder 全权设计与评估，允许调试但不可自动应用补丁（需商场审核），文档仅建议，依赖引入仅限低风险标准库。

---

## 四、FunctionSpec：目标函数原子定义

**禁止 OOP**。FunctionSpec 必须以纯函数为核心原子，所有交互通过显式参数与返回值完成。

```json
{
  "function_spec": {
    "function_name": "pulse_param_extract",
    "language": "python",
    "signature_hash": "sha256:abc123...",
    "input_schema": [
      {"name": "iq_samples", "type": "ndarray[complex128]", "shape_hint": "(N,)", "shm_key": "shm://producer_7/iq_vec"},
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
    "context_notes": "用于微波TR测试系统的脉冲参数提取，要求实时性，禁止动态内存分配"
  }
}
```

**关键约束说明**：
- `oop_forbidden: true` — 商场编译器必须拒绝任何包含类定义、`self` 引用、继承结构的生成代码。
- `side_effect_free` — 函数不得修改全局状态、SHM 区域（除显式返回值映射外）、文件系统。
- `signature_hash` — 生产者预先计算的函数签名哈希，防止 AICoder 篡改函数契约。

---

## 五、Sandbox：安全沙箱与 SHM 边界

AICoder 运行于商场的**隔离算力进程池**中，通过 SHM 矢量与外部通信。

| 字段 | 说明 |
|------|------|
| `sandbox_level` | `light`（仅语法检查）/ `standard`（容器隔离）/ `heavy`（硬件级虚拟化） |
| `shm_read_keys` | AICoder 可读取的 SHM 矢量键列表 |
| `shm_write_keys` | AICoder 可写入的 SHM 矢量键列表（通常仅限报告区） |
| `network_access` | `false` 默认，AICoder 不得访问外部网络 |
| `compiler_path` | 商场指定的受控编译器路径，AICoder 无权指定 |

**SHM 宪法原则**：AICoder 的写入键必须指向商场的**临时评估区**，而非生产者的核心数据区。评估通过后，由商场的**编译进化权**决定是否合并到主分支。

---

## 六、Signature：数字签名与验签流程

```
生产者私钥签名( Header + Mandate + FunctionSpec + Sandbox ) → 签名区块
商场公钥验签 → 解析权限矢量 → 生成 AICoder 任务令牌 → 注入算力进程池
```

签名算法：Ed25519（轻量、抗量子预备）。  
验签失败 → 文档直接销毁，记录安全审计日志，生产者身份冻结 300 秒。

---

## 七、授权文档生命周期状态机

```
SUBMITTED → VERIFIED → TOKEN_ISSUED → AICODER_EXECUTING → 
    ├─→ COMPLETED → REPORT_AUDIT → ARCHIVED
    ├─→ REVOKED (生产者主动撤销)
    └─→ EXPIRED (TTL 耗尽)
```

每个状态转换由商场的**主控状态机函数**原子性完成，状态存储于 SHM 状态矢量 `shm://mall/auth_state/{doc_id}`。

---

## 八、审核宣言

> 本规范由创世 Worker 起草，经人类架构师审核。  
> **审核状态**：待最终裁定。  
> 任何引入 OOP 语义、类继承、隐式状态修改的授权文档，商场有权永久拒绝该生产者的 AICoder 接入资格。
