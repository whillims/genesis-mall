# 微型商场 · 安全Worker设计蓝图

> **文档编号**: MW-SEC-001  
> **版本**: v1.0  
> **日期**: 2026-05-09  
> **对应规范**: 微型商场设计规范 v1.2  
> **设计层级**: Python函数级  
> **关联文档**: 算力法条、商场硬件宪法、调度Worker蓝图

---

## 1. Worker职责定义

安全Worker是微型商场的"免疫系统"与"宪法守护者"，承担以下核心职责：

- **身份认证与授权**: 为所有Worker签发、验证、吊销身份令牌
- **行为模型审核**: 审核生产者提交的函数蓝图，防止恶意代码进入函数库
- **沙箱管理**: 为不可信Worker创建隔离执行环境，限制SHM访问范围
- **攻击检测与响应**: 实时监测异常行为（心跳伪造、算力盗用、SHM越界）
- **物理通道安全**: 确保Worker间仅通过可追溯的SHM通道通信，黑客无所遁形

---

## 2. 入口参数设计

```python
def security_worker_entry(
    shm_config: dict,           # SHM共享内存配置
    mall_pubkey: bytes,         # 商场公钥（创世Worker生成）
    token_ttl_ms: int,          # 令牌有效期（毫秒）
    audit_callback: callable,   # 审计回调接口
    sandbox_engine: str,        # 沙箱引擎: "seccomp" | "namespaces" | "kvm"
    threat_db_path: str,        # 威胁特征库路径
    max_sandbox_count: int      # 最大并发沙箱数
) -> int:
    """
    安全Worker主入口函数

    章节对应:
        - 第1节: 职责定义
        - 第3节: 状态机实现
        - 第4节: 核心函数调用链

    返回值:
        0: 正常退出
        1: 密钥初始化失败
        2: 威胁库加载失败
        3: 沙箱引擎不可用
    """
```

---

## 3. 状态机设计

```
[INIT] --加载密钥与威胁库--> [KEY_WARMUP] --预热沙箱引擎--> [GUARDING]
   |                                                              |
   |<------------------ 威胁库更新 -------------------------------|
   v                                                              v
[AUTHING] <--认证请求-- [GUARDING] <--异常告警-- [ALERTING]
   |                           ^    |                              |
   |--审核通过--> [APPROVED]    |    |--确认攻击--> [QUARANTINE]    |
   |                           |    |                              |
   |--审核拒绝--> [REJECTED]    |    |--误报--> [GUARDING]         |
   |                           |    |                              |
   v                           |    v                              |
[TOKEN_ROT] --轮换完成--> [GUARDING] --吊销请求--> [REVOKING]      |
   |                                                              |
   v                                                              v
[SHUTDOWN] <----------------- 收到终止信号 -----------------------|
```

状态说明：
- **INIT**: 加载商场公钥/私钥对，初始化加密引擎
- **KEY_WARMUP**: 预生成令牌批次，减少实时签发延迟
- **GUARDING**: 正常守护状态，监听认证与审核请求
- **AUTHING**: 处理Worker身份认证
- **APPROVED**: 审核通过，签发令牌
- **REJECTED**: 审核拒绝，记录原因
- **ALERTING**: 收到异常告警，进行威胁评估
- **QUARANTINE**: 确认威胁，隔离涉事Worker
- **TOKEN_ROT**: 定期令牌轮换
- **REVOKING**: 令牌吊销处理
- **SHUTDOWN**: 安全关闭，吊销所有活跃令牌

---

## 4. 核心函数设计（函数级）

### 4.1 身份认证与令牌签发

```python
def auth_token_issue(
    worker_id: str,              # Worker唯一标识
    worker_type: str,            # Worker类型
    capability_digest: bytes,    # 能力标签摘要（防篡改）
    request_nonce: bytes,         # 请求随机数（防重放）
    attest_data: dict            # 远程证明数据（如适用）
) -> dict:
    """
    身份认证与令牌签发函数

    职责: 验证Worker身份合法性，签发带权限范围的JWT风格令牌
          令牌包含该Worker的SHM访问范围、算力配额、有效期

    返回值 dict:
        {
            "status": "issued" | "denied" | "pending_attest",
            "token": bytes,              # 加密令牌
            "token_id": str,             # 令牌唯一ID
            "expires_at": int,         # 过期时间戳
            "shm_scope": list,         # 允许的SHM段ID列表
            "compute_quota": int,      # 算力配额
            "audit_log_id": str        # 审计记录ID
        }

    测试方法:
        1. 使用合法密钥申请令牌，验证返回token结构正确
        2. 使用伪造密钥申请，验证被拒绝且记录审计日志
        3. 重放相同nonce，验证第二次被拒绝
        4. 令牌过期后访问SHM，验证被拦截
    """
```

### 4.2 行为模型安全审核

```python
def blueprint_security_audit(
    blueprint_id: str,           # 蓝图唯一标识
    blueprint_ast: dict,         # 蓝图抽象语法树（函数级）
    language: str,               # 目标语言: "python" | "verilog" | "c" | "java" | "js" | "css"
    producer_id: str,            # 提交该蓝图的生产者Worker ID
    import_whitelist: list,       # 允许的外部依赖列表
    shm_access_pattern: list      # 声明的SHM访问模式
) -> dict:
    """
    行为模型安全审核函数

    职责: 审核生产者提交的函数蓝图，检测以下风险：
          - 非法系统调用（os.system, subprocess等）
          - SHM越界访问声明
          - 无限循环/递归（算力炸弹）
          - 网络通信企图（商场模式下禁止直接IO）
          - 自修改代码或动态加载

    返回值 dict:
        {
            "status": "approved" | "rejected" | "conditional",
            "risk_score": float,         # 风险评分 0.0-1.0
            "violations": list,          # 违规项列表
            "required_sandbox_level": str, # 所需沙箱等级
            "conditional_rules": list,   # 条件通过时的附加限制
            "audit_log_id": str
        }

    测试方法:
        1. 提交包含os.system的Python蓝图，验证被拒绝
        2. 提交声明SHM段越界的蓝图，验证检测到越界风险
        3. 提交无限递归代码，验证算力炸弹检测触发
        4. 提交合法频谱仪算法蓝图，验证通过且风险评分<0.1
    """
```

### 4.3 沙箱创建与管理

```python
def sandbox_create(
    worker_id: str,              # 待隔离Worker ID
    sandbox_level: str,         # 沙箱等级: "light" | "standard" | "heavy"
    allowed_shm_segments: list, # 允许的SHM段ID列表
    compute_limit: int,         # 算力单元上限
    syscall_filter: list,       # 系统调用白名单
    max_execution_ms: int       # 最大执行时长（毫秒）
) -> dict:
    """
    沙箱创建与管理函数

    职责: 为不可信或高风险Worker创建隔离执行环境
          限制其系统调用、SHM访问范围、算力消耗、执行时长

    返回值 dict:
        {
            "sandbox_id": str,
            "namespace_fd": int,         # Linux namespace文件描述符
            "seccomp_profile": bytes,      # seccomp配置文件
            "shm_jail_map": dict,          # SHM段映射关系
            "cgroup_path": str,            # cgroup控制组路径
            "status": "active" | "failed"
        }

    测试方法:
        1. 创建沙箱后，内部Worker尝试访问未授权SHM段，验证SIGSEGV
        2. 沙箱内Worker执行死循环，验证cgroup在max_execution_ms后终止
        3. 沙箱内Worker尝试socket系统调用，验证seccomp拦截
        4. 验证沙箱销毁后所有资源被回收，无残留进程
    """
```

### 4.4 威胁检测与隔离

```python
def threat_detect_and_quarantine(
    worker_id: str,              # 疑似威胁Worker ID
    alert_type: str,            # 告警类型: "heartbeat_spoof" | "compute_theft" | "shm_escape" | "token_reuse"
    evidence: dict,             # 探针Worker收集的证据
    auto_quarantine: bool       # 是否自动隔离
) -> dict:
    """
    威胁检测与隔离函数

    职责: 根据探针Worker上报的异常数据，进行威胁判定
          一旦确认，立即吊销令牌、隔离Worker、保留证据链

    返回值 dict:
        {
            "threat_confirmed": bool,
            "action_taken": str,         # "quarantined" | "warned" | "ignored"
            "revoked_token_id": str,
            "sandbox_id": str,           # 隔离沙箱ID
            "evidence_chain_hash": bytes, # 证据链哈希（防篡改）
            "audit_log_id": str
        }

    测试方法:
        1. 模拟心跳伪造攻击，验证自动隔离在100ms内完成
        2. 模拟算力盗用（超配额运行），验证cgroup限制生效
        3. 模拟SHM越界读写，验证立即触发SIGSEGV并隔离
        4. 验证隔离后该Worker无法通过任何通道与其他Worker通信
    """
```

### 4.5 令牌吊销与轮换

```python
def token_revoke_and_rotate(
    token_id: str,               # 待吊销令牌ID
    reason: str,                 # 吊销原因
    emergency: bool,             # 是否紧急吊销（立即生效）
    rotate_all: bool             # 是否同时轮换所有同类型Worker令牌
) -> dict:
    """
    令牌吊销与轮换函数

    职责: 安全地吊销已发放令牌，可选批量轮换防止系统性风险
          紧急吊销模式下，令牌在SHM广播通道立即失效

    返回值 dict:
        {
            "revoked_count": int,
            "rotated_count": int,
            "broadcast_latency_ms": int,  # SHM广播延迟
            "affected_workers": list,
            "audit_log_id": str
        }

    测试方法:
        1. 紧急吊销令牌后，验证持有该令牌的Worker在50ms内失去SHM访问权
        2. 批量轮换100个令牌，验证所有Worker在1秒内获得新令牌
        3. 轮换过程中验证旧令牌仍短暂有效（ grace_period ）
        4. 模拟令牌泄露场景，验证紧急吊销不影响其他Worker
    """
```

---

## 5. 与其他Worker的交互协议

### 5.1 与调度Worker交互
- **注册时**: 调度Worker调用 `auth_token_issue()` 获取准入令牌
- **调度前**: 调度Worker请求复核高优先级任务的运行时权限
- **异常时**: 向调度Worker发送 `worker_revoke_notify`，要求立即将该Worker从注册表移除

### 5.2 与审计Worker交互
- **每次认证/审核/吊销**: 生成完整审计日志，包含证据哈希
- **威胁确认时**: 向审计Worker提交 `evidence_chain` 用于后续追溯

### 5.3 与探针Worker交互
- **ALERTING状态**: 接收探针Worker的 `anomaly_report`，作为威胁判定输入
- **GUARDING状态**: 向探针Worker下发 `monitor_policy`，指定重点监控对象

### 5.4 与创世Worker交互
- **INIT阶段**: 从创世Worker获取商场根密钥和初始信任锚
- **TOKEN_ROT**: 重大安全事件时，请求创世Worker进行商场级密钥轮换

---

## 6. 测试方法（设计检验标准）

| 测试场景 | 检验标准 | 通过条件 |
|---------|---------|---------|
| 伪造令牌渗透 | 使用自签名令牌申请SHM访问 | 被拒绝，攻击者Worker被标记为威胁 |
| 算力炸弹检测 | 提交含无限循环的Python蓝图 | 审核拒绝，风险评分>0.9 |
| SHM越狱测试 | 沙箱内Worker尝试访问未授权SHM | 物理隔离生效，进程被终止 |
| 心跳伪造响应 | 模拟心跳包篡改 | 100ms内完成隔离，证据链完整 |
| 令牌轮换风暴 | 同时轮换500个令牌 | 所有Worker在2秒内恢复，服务不中断 |
| 零日漏洞模拟 | 提交利用Python eval的隐蔽代码 | AST分析检测到动态代码执行，拒绝入库 |

---

## 7. 代码注释与文档章节映射

```python
# [对应第2节: 入口参数设计]
def security_worker_entry(...):
    # [对应第3节: INIT状态]
    state = State.INIT
    # [对应第5.4节: 与创世Worker交互]
    root_key = shm_read(shm_config["genesis_pubkey_channel"])
    ...
```

---

*本蓝图由创世Worker通过AICoder生成，待人类架构师审核后进入函数库。*
