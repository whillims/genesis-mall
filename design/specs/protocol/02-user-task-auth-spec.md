# 用户任务授权文件规范 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档定位**: 用户向商场提交任务执行的委托凭证，引用已授权Worker，不携带代码
> **设计原则**: 三界隔离、代码归属AICoder、用户只委托执行
> **核心铁律**: 用户任务授权文件**禁止**包含 `source_code`、`compiled_binary`、`verification_report`

---

## 一、与生产者授权文件的本质区别

| 维度 | 生产者授权文件 (01规范) | 用户任务授权文件 (02规范) |
|------|------------------------|--------------------------|
| 提交者 | 生产者（通过AICoder） | 用户（直接提交） |
| 携带代码 | 是（5字段含代码+编译产物） | **否**（只有Worker引用+参数） |
| 验证流程 | 七步验证（含沙箱加载/回归测试） | 三步验证（格式/签名/Worker存在性） |
| 目的 | 将新Worker载入商场算力池 | 委托已注册Worker执行任务 |
| 代码归属 | AICoder域产出 | 不涉及代码 |
| 蓝图归属 | 生产者域提交 | 不涉及蓝图 |

**三界隔离约束**:
- 生产者域: 只提交蓝图，不提交代码
- AICoder域: 审查蓝图，生成代码和授权文件
- 商场域: 只认授权文件，不接收蓝图
- **用户域**: 只提交任务委托，引用已有Worker，不触碰代码

---

## 二、文件结构

用户任务授权文件为JSON格式，包含**两个顶级字段**:

```json
{
  "auth_header": {
    "auth_type": "user_task",
    "task_id": "USER-TASK-001",
    "consumer_id": "CONSUMER-TEST-001",
    "submit_timestamp": "2026-05-13T00:00:00Z",
    "user_signature": "SHA256-HASH",
    "auth_version": "1.0",
    "task_label": "task_colle/task_a/task_sa"
  },
  "task_config": {
    "worker_handle": "FUNC-HANDLE-XXXX",
    "worker_name": "report_writer",
    "params": {
      "message": "hello,我正在跑任务"
    }
  }
}
```

### 2.1 auth_header 字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `auth_type` | String | 是 | 固定值 `"user_task"`，用于区分生产者授权文件 |
| `task_id` | String | 是 | 用户任务唯一标识 |
| `consumer_id` | String | 是 | 提交任务的消费者ID |
| `submit_timestamp` | ISO-8601 | 是 | 提交时间戳 |
| `user_signature` | String | 是 | 用户签名（SHA256，基于task_config内容） |
| `auth_version` | String | 是 | 规范版本，当前 `"1.0"` |
| `task_label` | String | 否 | 任务标签，格式 `task_colle/task_a/task_sa` |

### 2.2 task_config 字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `worker_handle` | String | 是 | 目标Worker的函数注册表Handle（如 `FUNC-HANDLE-0002`） |
| `worker_name` | String | 是 | 目标Worker函数名（如 `report_writer`），用于冗余校验 |
| `params` | Dict | 是 | 传递给Worker的ctx参数 |

---

## 三、商场三步验证流程

用户任务授权文件进入商场后，执行**三步验证**（区别于生产者授权的七步验证）：

### Step 1: 格式校验

检查项:
- JSON可解析
- 两个顶级字段齐全: `auth_header`, `task_config`
- `auth_type` == `"user_task"`
- `auth_header` 必需字段齐全
- `task_config` 必需字段齐全
- `auth_version` == `"1.0"`
- `worker_handle` 格式合法（`FUNC-HANDLE-XXXX`）
- `params` 为dict

### Step 2: 签名验证

签名载荷:
```python
sig_payload = json.dumps({
    "task_id": header["task_id"],
    "consumer_id": header["consumer_id"],
    "worker_handle": config["worker_handle"],
    "worker_name": config["worker_name"],
    "params_hash": sha256(json.dumps(config["params"], sort_keys=True))
}, sort_keys=True, separators=(",", ":"))
```

验证: `SHA256(sig_payload) == user_signature`

### Step 3: Worker存在性验证

检查项:
- `worker_handle` 在函数注册表中存在
- 对应Worker状态为 ACTIVE
- `worker_name` 与注册表中的函数名一致

验证通过后，商场从函数注册表中取出已授权的Worker执行，传入 `task_config.params` 作为ctx。

---

## 四、禁止项

| 禁止 | 原因 |
|------|------|
| 包含 `source_code` 字段 | 代码属于AICoder域，用户任务不触碰代码 |
| 包含 `compiled_binary` 字段 | 编译产物属于AICoder域 |
| 包含 `verification_report` 字段 | 验证报告属于AICoder域 |
| 包含 `design_meta` 字段 | 函数设计元数据属于AICoder域 |
| `auth_type` 不为 `"user_task"` | 防止与生产者授权文件混淆 |

---

## 五、执行流程

```
用户提交任务授权文件
       │
       ▼
  Step1: 格式校验 ──失败──▶ REJECT
       │
       ▼
  Step2: 签名验证 ──失败──▶ REJECT
       │
       ▼
  Step3: Worker存在性 ──失败──▶ REJECT (Worker未注册/非ACTIVE)
       │
       ▼
  从函数注册表取出Worker
       │
       ▼
  call_registered_func(handle, params)
       │
       ▼
  返回执行结果给用户
```

---

## 六、与现有系统的关系

```
生产者蓝图 ──▶ AICoder七维审查 ──▶ 生产者授权文件(5字段) ──▶ 商场七步验证 ──▶ 函数注册表
                                                                                    │
用户任务授权文件(2字段) ──────────────────────▶ 商场三步验证 ──────────────────────▶ 查表执行
```

用户任务授权文件是**消费侧**凭证，它依赖函数注册表中已有的Worker，不创建新Worker。
