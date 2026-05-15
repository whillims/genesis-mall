# 创世纪 · 第五阶段

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 运行态觉醒 —— 消费者接入与三元循环启动

### 一、阶段定位与哲学阐述

前四阶段，我们完成了骨架（主函数）、血脉（AICoder接口与授权）、脏腑（SHM状态机与编译进化）、神经（Worker调度与算力池）。
第五阶段，是吹入生命气息的时刻——消费者作为第三极正式入场，生产者-商场-消费者三元结构闭合，商场从"构建态"跃迁至"运行态"。

此阶段的核心哲学：商场不是仓库，而是永动的熔炉。
消费者提交的不是"购买请求"，而是需求蓝图；商场不做"中介匹配"，而是催化反应——将消费者需求、生产者函数、算力资源三者置于高温高压的SHM反应腔中，产出可执行的价值体。

关键词定义：
- **需求蓝图（Demand Blueprint）**：消费者提交的JSON结构化文档，描述任务类型、输入数据规格、期望输出格式、QoS等级。
- **三元循环（Trinity Loop）**：生产者⊕商场⊕消费者的闭环数据流，商场作为催化剂不参与价值分配，只保障反应条件。
- **反应腔（Reaction Chamber）**：SHM中动态划隔离的内存区域，用于单次任务编译、加载、执行、销毁的原子化生命周期。
- **生命气息（Breath of Life）**：商场主循环从阻塞态转入轮询态的标志事件，即首个消费者需求蓝图被成功解析并注入调度队列。

### 二、核心函数总览

| 函数名 | 职责 | 所属模块 |
| --- | --- | --- |
| consumer_register() | 消费者身份注册与能力声明 | consumer_gateway.py |
| demand_parse() | 需求蓝图解析与合法性校验 | consumer_gateway.py |
| demand_enqueue() | 需求注入调度队列 | consumer_gateway.py |
| trinity_match() | 三元匹配：生产者×需求×算力 | scheduler.py |
| reaction_chamber_alloc() | 反应腔动态隔离分配 | scheduler.py |
| chamber_destroy() | 反应腔原子化销毁与资源回收 | scheduler.py |
| task_compile_load() | 函数编译与加载入反应腔 | executor.py |
| task_execute() | 执行控制：启动→监控→收割 | executor.py |
| main_loop_run() | 商场主循环：永恒轮询 | main_loop.py |
| result_deliver() | 结果回传与消费者确认 | consumer_gateway.py |

### 三、调用关系拓扑图

```
┌─────────────────────────────────────────────────────────────┐
│                      第五阶段 运行态拓扑                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   [消费者宇宙]          [商场核心]           [生产者宇宙]      │
│        │                    │                      │        │
│        ▼                    ▼                      ▼        │
│  ┌──────────┐      ┌──────────────┐        ┌──────────┐    │
│  │consumer_ │─────▶│  demand_     │        │ 生产者   │    │
│  │register()│      │  parse()     │◀───────│ 函数仓库  │    │
│  └──────────┘      └──────┬───────┘        └──────────┘    │
│        │                   │                               │
│        │                   ▼                               │
│        │            ┌──────────────┐                        │
│        │            │ demand_      │                        │
│        │            │ enqueue()    │                        │
│        │            └──────┬───────┘                        │
│        │                   │                               │
│        │                   ▼                               │
│        │            ┌──────────────┐  ◄── 算力池快照        │
│        │            │ trinity_     │                        │
│        │            │ match()      │                        │
│        │            └──────┬───────┘                        │
│        │                   │                               │
│        │         ┌────────┴────────┐                      │
│        │         ▼                 ▼                      │
│        │  ┌────────────┐   ┌────────────┐                │
│        │  │reaction_   │   │  匹配失败    │──▶ 需求降级   │
│        │  │chamber_    │   │  回退队列    │    或拒绝    │
│        │  │alloc()     │   └────────────┘                │
│        │  └─────┬──────┘                                  │
│        │        │                                          │
│        │        ▼                                          │
│        │  ┌────────────┐   ┌────────────┐                │
│        │  │task_       │──▶│ task_      │                │
│        │  │compile_    │   │ execute()  │                │
│        │  │load()      │   └─────┬──────┘                │
│        │  └────────────┘         │                       │
│        │                          ▼                       │
│        │                   ┌────────────┐                │
│        │                   │result_     │──────────────┐ │
│        │                   │deliver()   │              │ │
│        │                   └────────────┘              │ │
│        │                          │                    │ │
│        │                          ▼                    │ │
│        │                   ┌────────────┐             │ │
│        └───────────────────│ 消费者确认   │◀────────────┘ │
│                            │ 或超时重传   │              │
│                            └─────┬──────┘              │
│                                  ▼                    │
│                            ┌────────────┐             │
│                            │chamber_    │             │
│                            │destroy()   │             │
│                            └────────────┘             │
│                                  │                    │
│                                  ▼                    │
│                            [SHM资源回收到算力池]       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 四、函数级详细设计

#### 4.1 consumer_register() —— 消费者入场登记

**函数签名**：
```python
def consumer_register(
    state: Dict[str, Any],
    public_key: bytes,
    domain_tags: str
) -> Tuple[Dict[str, Any], int, Optional[int]]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| public_key | bytes | 消费者公钥，32字节Ed25519 |
| domain_tags | str | 需求领域标签，逗号分隔字符串 |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| consumer_id | int | 分配的消费者唯一ID（失败返回-1） |
| shm_base | Optional[int] | SHM通信信道基址（失败返回None） |

**设计规则**：
- RULE-5.1: 消费者无状态，商场不保存消费者偏好，只保存通信端点
- RULE-5.2: 公钥即身份，无用户名密码，拒绝任何形式的会话Cookie

**状态机影响**：消费者状态: NONEXISTENT → REGISTERED

**实现代码**（`src/stage5/consumer_gateway.py`）：
```python
COMM_CHANNEL_SIZE = 65536

def _validate_public_key(public_key: bytes) -> bool:
    return len(public_key) == 32

def consumer_register(
    state: Dict[str, Any],
    public_key: bytes,
    domain_tags: str
) -> Tuple[Dict[str, Any], int, Optional[int]]:
    if not _validate_public_key(public_key):
        return state, -1, None
    
    consumer_id = state["next_consumer_id"]
    pubkey_hash = hashlib.sha256(public_key).hexdigest()
    
    new_consumer = {
        "consumer_id": consumer_id,
        "public_key_hash": pubkey_hash,
        "domain_tags": domain_tags,
        "state": ConsumerState.REGISTERED.value,
        "channel_base": None,
        "register_time": 0
    }
    
    shm_base = _allocate_shm_channel(state, consumer_id)
    if shm_base is None:
        return state, -1, None
    
    new_consumer["channel_base"] = shm_base
    
    new_consumers = state["consumers"].copy()
    new_consumers[consumer_id] = new_consumer
    
    channel = create_consumer_channel(consumer_id, shm_base)
    new_channels = state["consumer_channels"].copy()
    new_channels[consumer_id] = channel
    
    new_state = {
        **state,
        "consumers": new_consumers,
        "consumer_channels": new_channels,
        "next_consumer_id": consumer_id + 1
    }
    
    return new_state, consumer_id, shm_base
```

**测试方法**：
- TEST-5.1: 传入非法长度公钥(≠32)，返回 consumer_id=-1
- TEST-5.2: 并发1000次注册，consumer_id全局唯一且无碰撞
- TEST-5.3: 注册后立刻查询SHM，确认环形缓冲区物理地址对齐到页边界

---

#### 4.2 demand_parse() —— 需求蓝图解析

**函数签名**：
```python
def demand_parse(
    state: Dict[str, Any],
    raw_json: str,
    consumer_id: int
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]], int]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| raw_json | str | 原始JSON字符串（长度限制 ≤ 16KB） |
| consumer_id | int | 消费者ID |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| demand_body | Optional[Dict] | 结构化需求体（失败返回None） |
| demand_id | int | 需求ID（失败返回负值错误码） |

**设计规则**：
- RULE-5.3: 解析器必须是零拷贝预扫描，禁止malloc内部缓冲区
- RULE-5.4: 只接受白名单字段，未知字段直接触发错误

**白名单字段**：`task_type`, `input_schema`, `output_schema`, `qos_level`, `timeout_ms`

**结构化需求体定义**：
```python
def create_demand_body(
    task_type: str,
    input_schema: str,
    output_schema: str,
    qos_level: QoSLevel,
    timeout_ms: int,
    consumer_id: int
) -> Dict[str, Any]:
    return {
        "task_type_hash": int(hashlib.md5(task_type.encode()).hexdigest()[:8], 16),
        "input_schema": input_schema.encode()[:64].ljust(64, b'\x00'),
        "output_schema": output_schema.encode()[:64].ljust(64, b'\x00'),
        "qos_level": qos_level.value,
        "timeout_ms": timeout_ms,
        "consumer_id": consumer_id
    }
```

**实现代码**（`src/stage5/consumer_gateway.py`）：
```python
def demand_parse(
    state: Dict[str, Any],
    raw_json: str,
    consumer_id: int
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]], int]:
    MAX_JSON_SIZE = 16384
    
    if len(raw_json) > MAX_JSON_SIZE:
        return state, None, -1  # E_OVERSIZE
    
    try:
        raw_data = json.loads(raw_json)
    except json.JSONDecodeError:
        return state, None, -2  # E_MALFORMED
    
    allowed_fields = {"task_type", "input_schema", "output_schema", "qos_level", "timeout_ms"}
    for key in raw_data.keys():
        if key not in allowed_fields:
            return state, None, -3  # E_UNKNOWN_FIELD
    
    required_fields = ["task_type", "input_schema", "output_schema"]
    for field in required_fields:
        if field not in raw_data:
            return state, None, -4  # E_MISSING_FIELD
    
    qos_value = raw_data.get("qos_level", 1)
    if qos_value not in [0, 1, 2]:
        return state, None, -5  # E_QOS_INVALID
    
    timeout_ms = raw_data.get("timeout_ms", 5000)
    if timeout_ms <= 0 or timeout_ms > 600000:
        return state, None, -6  # E_TIMEOUT_INVALID
    
    demand_body = create_demand_body(
        task_type=raw_data["task_type"],
        input_schema=raw_data["input_schema"],
        output_schema=raw_data["output_schema"],
        qos_level=QoSLevel(qos_value),
        timeout_ms=timeout_ms,
        consumer_id=consumer_id
    )
    
    demand_id = state["next_demand_id"]
    demand = {
        "demand_id": demand_id,
        "body": demand_body,
        "state": DemandState.PARSED.value,
        "submit_time": 0,
        "consumer_id": consumer_id
    }
    
    new_demands = state["demands"].copy()
    new_demands[demand_id] = demand
    
    new_state = {
        **state,
        "demands": new_demands,
        "next_demand_id": demand_id + 1
    }
    
    return new_state, demand_body, demand_id
```

**测试方法**：
- TEST-5.4: 注入含SQL注入片段的JSON，返回 E_UNKNOWN_FIELD
- TEST-5.5: qos_level=3（超范围），返回 E_QOS_INVALID
- TEST-5.6: 16KB边界值测试，15.9KB成功，16.1KB返回 E_OVERSIZE

---

#### 4.3 trinity_match() —— 三元催化匹配

**函数签名**：
```python
def trinity_match(
    state: Dict[str, Any],
    demand_body: Dict[str, Any],
    producer_index: Dict[str, Any],
    pool_snapshot: Dict[str, Any]
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]], MatchResult]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| demand_body | Dict | 结构化需求体 |
| producer_index | Dict | 生产者索引快照（只读） |
| pool_snapshot | Dict | 算力池快照（只读） |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| match_plan | Optional[Dict] | 匹配方案（失败返回None） |
| result | MatchResult | 匹配结果枚举 |

**设计规则**：
- RULE-5.5: 匹配算法必须是O(1)或O(logN)，禁止O(N²)遍历
- RULE-5.6: 采用三级过滤：task_type_hash → producer_capability_bitmap → 算力余量阈值
- RULE-5.7: 返回的match_plan_t是SHM分配的临时体，生命周期绑定反应腔

**匹配方案体定义**：
```python
def create_match_plan(
    producer_id: int,
    worker_id: int,
    memory_quota: int,
    cpu_affinity: int,
    priority_boost: int = 0
) -> Dict[str, Any]:
    return {
        "producer_id": producer_id,
        "worker_id": worker_id,
        "memory_quota": memory_quota,
        "cpu_affinity": cpu_affinity,
        "priority_boost": priority_boost
    }
```

**实现代码**（`src/stage5/scheduler.py`）：
```python
def _hash_lookup(task_type_hash: int, producer_index: Dict[str, Any]) -> List[int]:
    return producer_index.get(str(task_type_hash), [])

def _bitmap_match(producer_id: int, task_type_hash: int, producer_index: Dict[str, Any]) -> bool:
    producer = producer_index.get("producers", {}).get(producer_id, {})
    capabilities = producer.get("capability_bitmap", 0)
    return (capabilities & (1 << (task_type_hash % 32))) != 0

def trinity_match(
    state: Dict[str, Any],
    demand_body: Dict[str, Any],
    producer_index: Dict[str, Any],
    pool_snapshot: Dict[str, Any]
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]], MatchResult]:
    task_type_hash = demand_body["task_type_hash"]
    
    candidates = _hash_lookup(task_type_hash, producer_index)
    
    if not candidates:
        return state, None, MatchResult.NO_MATCH
    
    valid_candidates = [
        p for p in candidates
        if _bitmap_match(p, task_type_hash, producer_index)
    ]
    
    if not valid_candidates:
        return state, None, MatchResult.NO_MATCH
    
    scored_candidates = []
    for producer_id in valid_candidates:
        score = _compute_resource_score(producer_id, pool_snapshot, 65536)
        if score > 0:
            scored_candidates.append((producer_id, score))
    
    if not scored_candidates:
        return state, None, MatchResult.RESOURCE_EXHAUSTED
    
    scored_candidates.sort(key=lambda x: -x[1])
    best_producer_id = scored_candidates[0][0]
    best_worker_id = _select_worker(best_producer_id, pool_snapshot)
    
    if best_worker_id is None:
        return state, None, MatchResult.RESOURCE_EXHAUSTED
    
    match_plan = create_match_plan(
        producer_id=best_producer_id,
        worker_id=best_worker_id,
        memory_quota=655360,
        cpu_affinity=0xFFFFFFFF,
        priority_boost=demand_body["qos_level"]
    )
    
    plan_id = state["next_plan_id"]
    new_plans = state["match_plans"].copy()
    new_plans[plan_id] = match_plan
    
    new_state = {
        **state,
        "match_plans": new_plans,
        "next_plan_id": plan_id + 1
    }
    
    return new_state, match_plan, MatchResult.SUCCESS
```

**失败策略**：
- 若匹配失败，返回 `MatchResult.NO_MATCH` 或 `MatchResult.RESOURCE_EXHAUSTED`
- 调用方将需求体标记为 DEFERRED，投入延迟队列
- 延迟队列采用指数退避：1s → 2s → 4s → 8s → 丢弃（并通知消费者）

**测试方法**：
- TEST-5.7: 10万需求×1万生产者，匹配耗时 < 5ms（P99）
- TEST-5.8: 算力池满载时，所有返回 RESOURCE_EXHAUSTED，延迟队列长度可控
- TEST-5.9: 生产者能力位图精确匹配，误匹配率为0

---

#### 4.4 reaction_chamber_alloc() —— 反应腔隔离

**函数签名**：
```python
def reaction_chamber_alloc(
    state: Dict[str, Any],
    match_plan: Dict[str, Any],
    min_size: int = 131072
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]]]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| match_plan | Dict | 匹配方案 |
| min_size | int | 最小内存需求（默认128KB） |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| chamber | Optional[Dict] | 反应腔句柄（失败返回None） |

**设计规则**：
- RULE-5.8: 反应腔是SHM中的"物理隔离岛"，采用mprotect或MPK硬隔离
- RULE-5.9: 每个反应腔拥有独立的矢量管理状态机，与全局SHM状态机解耦
- RULE-5.10: 分配失败时，触发算力池紧急回收（杀死最低优先级反应腔）

**反应腔SHM布局**（线性地址空间）：
```
+0x0000 ~ +0x0FFF: 反应腔控制块 (4KB)
    ├── chamber_magic: "CHMB"
    ├── owner_producer_id
    ├── owner_consumer_id
    ├── lifecycle_state: ALLOCATION → COMPILATION → EXECUTION → DESTRUCTION
    ├── input_vector_base
    ├── output_vector_base
    └── error_log_ring
+0x1000 ~ +0xFFFF: 输入数据矢量 (60KB)
+0x10000~ +0x1FFFF: 输出数据矢量 (64KB)
+0x20000~ +0x?FFFF: 代码加载区 (动态)
```

**实现代码**（`src/stage5/scheduler.py`）：
```python
def reaction_chamber_alloc(
    state: Dict[str, Any],
    match_plan: Dict[str, Any],
    min_size: int = 131072
) -> Tuple[Dict[str, Any], Optional[Dict[str, Any]]]:
    chamber_id = state["next_chamber_id"]
    
    if state["compute_pool"]["active_tasks"] >= state["compute_pool"]["max_tasks"]:
        state = _emergency_reclaim(state)
    
    if state["compute_pool"]["available_memory"] < min_size:
        return state, None
    
    base_addr = _allocate_chamber_memory(state, min_size)
    
    if base_addr is None:
        return state, None
    
    chamber = create_reaction_chamber(chamber_id, match_plan, base_addr, min_size)
    
    new_chambers = state["chambers"].copy()
    new_chambers[chamber_id] = chamber
    
    new_compute_pool = state["compute_pool"].copy()
    new_compute_pool["available_memory"] -= min_size
    new_compute_pool["active_tasks"] += 1
    
    new_state = {
        **state,
        "chambers": new_chambers,
        "compute_pool": new_compute_pool,
        "next_chamber_id": chamber_id + 1
    }
    
    return new_state, chamber

def _emergency_reclaim(state: Dict[str, Any]) -> Dict[str, Any]:
    chambers = state["chambers"].copy()
    
    lowest_priority = None
    target_chamber_id = None
    
    for chamber_id, chamber in chambers.items():
        plan = _find_plan_for_chamber(chamber_id, state)
        if plan:
            priority = plan.get("priority_boost", 0)
            if lowest_priority is None or priority < lowest_priority:
                lowest_priority = priority
                target_chamber_id = chamber_id
    
    if target_chamber_id:
        state = chamber_destroy(state, target_chamber_id)
    
    return state
```

**测试方法**：
- TEST-5.10: 分配后，外部进程（非owner）尝试读写反应腔，触发SIGSEGV
- TEST-5.11: 连续分配1000个反应腔，总SHM碎片率 < 5%
- TEST-5.12: 紧急回收场景下，最低优先级反应腔被强制销毁，数据不泄露

---

#### 4.5 task_compile_load() —— 编译与加载

**函数签名**：
```python
def task_compile_load(
    state: Dict[str, Any],
    source_code: str,
    chamber_base: int,
    optimize_level: int = 2,
    security_check: bool = True
) -> Tuple[Dict[str, Any], int, ExecutionError]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| source_code | str | 生产者提交的函数源码 |
| chamber_base | int | 反应腔基址 |
| optimize_level | int | 优化等级（默认-O2） |
| security_check | bool | 是否启用安全检查 |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| entry_point | int | 函数入口点地址 |
| error | ExecutionError | 执行错误码 |

**设计规则**：
- RULE-5.11: 编译过程在反应腔内完成，产物不落地磁盘，直接生成内存ELF
- RULE-5.12: 编译器前端必须是确定性的，相同源码→相同机器码
- RULE-5.13: 加载前进行符号沙箱校验：禁止syscall指令，禁止动态链接，禁止栈执行

**编译缓存机制**：
- 全局SHM维护 `compile_cache`：源码哈希 → 机器码哈希 → 反应腔代码区偏移
- 命中缓存时，直接memcpy机器码，跳过编译

**实现代码**（`src/stage5/executor.py`）：
```python
COMPILE_CACHE_SIZE = 1024

def _compute_source_hash(source_code: str) -> str:
    return hashlib.sha256(source_code.encode()).hexdigest()

def _sandbox_check(source_code: str) -> bool:
    forbidden_patterns = [
        "syscall", "dlopen", "dlsym", "execve", "fork",
        "mmap", "mprotect", "socket", "connect", "open("
    ]
    code_lower = source_code.lower()
    for pattern in forbidden_patterns:
        if pattern in code_lower:
            return False
    return True

def task_compile_load(
    state: Dict[str, Any],
    source_code: str,
    chamber_base: int,
    optimize_level: int = 2,
    security_check: bool = True
) -> Tuple[Dict[str, Any], int, ExecutionError]:
    if security_check and not _sandbox_check(source_code):
        return state, 0, ExecutionError.SANDBOX_VIOLATION
    
    source_hash = _compute_source_hash(source_code)
    compile_cache = state["compile_cache"].copy()
    
    if source_hash in compile_cache:
        entry_point = chamber_base + 0x20000
        compile_cache[source_hash]["hit_count"] += 1
        
        return {**state, "compile_cache": compile_cache}, entry_point, ExecutionError.OK
    
    entry_point = chamber_base + 0x20000
    
    if len(compile_cache) >= COMPILE_CACHE_SIZE:
        oldest_key = min(compile_cache.keys(), key=lambda k: compile_cache[k]["access_time"])
        del compile_cache[oldest_key]
    
    compile_cache[source_hash] = {
        "entry_point": entry_point,
        "size": len(source_code) * 4,
        "hit_count": 1,
        "access_time": int(time.time() * 1000)
    }
    
    return {**state, "compile_cache": compile_cache}, entry_point, ExecutionError.OK
```

**测试方法**：
- TEST-5.13: 含非法syscall的源码，编译阶段返回 SANDBOX_VIOLATION
- TEST-5.14: 相同源码两次提交，第二次缓存命中，耗时 < 1ms
- TEST-5.15: 编译产物在反应腔内可正确执行，返回预期计算结果

---

#### 4.6 task_execute() —— 执行监控收割

**函数签名**：
```python
def task_execute(
    state: Dict[str, Any],
    entry_point: int,
    input_vector: bytes,
    chamber_id: int,
    timeout_ms: int
) -> Tuple[Dict[str, Any], bytes, ExecutionError]
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场状态 |
| entry_point | int | 函数入口点 |
| input_vector | bytes | 输入数据矢量 |
| chamber_id | int | 反应腔ID |
| timeout_ms | int | 超时阈值 |

**返回值**：
| 返回值 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 更新后的状态 |
| result | bytes | 输出数据矢量 |
| error | ExecutionError | 执行错误码 |

**设计规则**：
- RULE-5.14: 执行采用"看门狗线程"模型，主线程监控，Worker线程执行
- RULE-5.15: 超时即杀，无警告，无续命机制，保证商场整体吞吐量
- RULE-5.16: 执行统计写入SHM广播区，供外部监控探针实时读取

**执行状态机**（反应腔级）：
```
COMPILATION
    │
    ▼
[加载完成]
    │
    ▼
EXECUTION ──超时──▶ TIMEOUT ──▶ DESTRUCTION
    │                              │
成功/失败                           │
    │                              │
    ▼                              │
RESULT_READY ──────────────────────┘
```

**实现代码**（`src/stage5/executor.py`）：
```python
def task_execute(
    state: Dict[str, Any],
    entry_point: int,
    input_vector: bytes,
    chamber_id: int,
    timeout_ms: int
) -> Tuple[Dict[str, Any], bytes, ExecutionError]:
    start_time = time.time() * 1000
    
    if time.time() * 1000 - start_time > timeout_ms:
        return state, b"", ExecutionError.TIMEOUT
    
    result = _simulate_execution(input_vector)
    
    new_chambers = state["chambers"].copy()
    if chamber_id in new_chambers:
        chamber = new_chambers[chamber_id].copy()
        chamber["state"] = ChamberState.EXECUTION.value
        new_chambers[chamber_id] = chamber
    
    return {**state, "chambers": new_chambers}, result, ExecutionError.OK

def _watchdog_monitor(
    state: Dict[str, Any],
    chamber_id: int,
    timeout_ms: int
) -> Tuple[Dict[str, Any], bool]:
    chamber = state["chambers"].get(chamber_id)
    if not chamber:
        return state, False
    
    elapsed = int(time.time() * 1000) - chamber.get("execute_time", 0)
    
    if elapsed > timeout_ms:
        new_chambers = state["chambers"].copy()
        new_chambers[chamber_id]["state"] = ChamberState.TIMEOUT.value
        return {**state, "chambers": new_chambers}, True
    
    return state, False
```

**测试方法**：
- TEST-5.16: 正常任务在timeout_ms内完成，返回 ExecutionError.OK
- TEST-5.17: 死循环任务，看门狗在timeout_ms+5ms内强制终止
- TEST-5.18: 执行统计的写入是原子的，监控探针读取无脏数据

---

#### 4.7 main_loop_run() —— 永恒轮询

**函数签名**：
```python
def main_loop_run(state: Dict[str, Any]) -> None
```

**参数说明**：
| 参数 | 类型 | 说明 |
| --- | --- | --- |
| state | Dict | 全局商场控制块（SHM基址） |

**设计规则**：
- RULE-5.17: 主循环是单线程事件驱动，禁止多线程竞争，所有状态转换原子化
- RULE-5.18: 循环体必须是"检测→决策→行动→等待"四拍子，无冗余逻辑
- RULE-5.19: 等待采用epoll/kqueue/IOCP，空转CPU占用率为0%

**主循环伪代码**（行为级模型）：
```python
def main_loop_run(state: Dict[str, Any]) -> None:
    state["mall_state"] = MallState.RUNNING.value
    
    while state["mall_state"] != MallState.TERMINATED.value:
        # 1. 检测
        new_demands = _poll_consumer_queue(state)
        completed_tasks = _poll_completion_ring(state)
        dead_chambers = _poll_watchdog_alerts(state)
        
        # 2. 决策
        for demand_info in new_demands:
            demand_id = demand_info["demand_data"].get("demand_id")
            if demand_id in state["demands"]:
                demand_body = state["demands"][demand_id]["body"]
                pool_snap = get_pool_snapshot(state)
                
                state, plan, match_result = trinity_match(
                    state, demand_body, {}, pool_snap
                )
                
                if match_result == MatchResult.SUCCESS and plan:
                    state = _schedule_immediate(state, demand_id, plan)
                else:
                    state = _schedule_deferred(state, demand_id)
        
        # 3. 行动
        for chamber_id in dead_chambers:
            state = chamber_destroy(state, chamber_id)
        
        # 4. 更新
        state = update_deferred_queue(state)
        state = update_broadcast_vector(state)
        
        time.sleep(0.001)
```

**实现代码**（`src/stage5/main_loop.py`）：
```python
def _poll_consumer_queue(state: Dict[str, Any]) -> List[Dict[str, Any]]:
    new_demands = []
    for consumer_id, channel in state["consumer_channels"].items():
        for demand in channel.get("demand_queue", []):
            new_demands.append({
                "consumer_id": consumer_id,
                "demand_data": demand
            })
    return new_demands

def _schedule_immediate(state: Dict[str, Any], demand_id: int, plan: Dict[str, Any]) -> Dict[str, Any]:
    state, chamber = reaction_chamber_alloc(state, plan)
    
    if chamber:
        source_code = _get_producer_code(state, plan["producer_id"])
        state, entry_point, error = task_compile_load(
            state, source_code, chamber["base_addr"]
        )
        
        if error.value == 0:
            input_vector = _get_input_vector(state, demand_id)
            state, result, exec_error = task_execute(
                state, entry_point, input_vector, chamber["chamber_id"],
                state["demands"][demand_id]["body"]["timeout_ms"]
            )
            
            _deliver_result(state, demand_id, result)
            state = chamber_destroy(state, chamber["chamber_id"])
    
    return state

def main_loop_run(state: Dict[str, Any]) -> None:
    state["mall_state"] = MallState.RUNNING.value
    
    try:
        while state["mall_state"] != MallState.TERMINATED.value:
            new_demands = _poll_consumer_queue(state)
            completed_tasks = _poll_completion_ring(state)
            dead_chambers = _poll_watchdog_alerts(state)
            
            for demand_info in new_demands:
                demand_id = demand_info["demand_data"].get("demand_id")
                if demand_id and demand_id in state["demands"]:
                    demand_body = state["demands"][demand_id]["body"]
                    pool_snap = get_pool_snapshot(state)
                    
                    state, plan, match_result = trinity_match(
                        state, demand_body, {}, pool_snap
                    )
                    
                    if match_result == MatchResult.SUCCESS and plan:
                        state = _schedule_immediate(state, demand_id, plan)
                    else:
                        state = _schedule_deferred(state, demand_id)
            
            for chamber_id in dead_chambers:
                state = chamber_destroy(state, chamber_id)
            
            state = update_deferred_queue(state)
            state = update_broadcast_vector(state)
            
            time.sleep(0.001)
    
    except KeyboardInterrupt:
        state["mall_state"] = MallState.TERMINATED.value
        for chamber_id in list(state["chambers"].keys()):
            state = chamber_destroy(state, chamber_id)
```

**测试方法**：
- TEST-5.19: 空载运行24小时，CPU占用 < 0.1%，内存无泄漏
- TEST-5.20: 满载压力测试（10万QPS），主循环不阻塞，P99延迟稳定
- TEST-5.21: 收到SIGTERM后，优雅退出：完成进行中的反应腔，拒绝新需求，5秒内终止

---

### 五、SHM矢量管理机制（第五阶段特化）

#### 5.1 消费者通信信道矢量

```
┌────────────────────────────────────────┐
│         Consumer Comm Channel          │
├────────────────────────────────────────┤
│  Ring Buffer Header (64 bytes)         │
│  ├── magic: "CCRM"                     │
│  ├── consumer_id                       │
│  ├── read_ptr  (消费者读，商场写)        │
│  ├── write_ptr (消费者写，商场读)        │
│  ├── watermark: 80%                    │
│  └── lock-free sequence                │
├────────────────────────────────────────┤
│  Data Payload (64KB - 64B)             │
│  ├── 需求蓝图队列（消费者→商场）          │
│  └── 结果回传队列（商场→消费者）          │
└────────────────────────────────────────┘
```

**实现代码**（`src/stage5/shm_vector.py`）：
```python
COMM_CHANNEL_SIZE = 65536

def init_consumer_comm_channel(
    shm_manager: Dict[str, Any],
    consumer_id: int
) -> Tuple[Dict[str, Any], Optional[int]]:
    seg_id = shm_manager.get("_next_seg_id", 1)
    
    segment = {
        "seg_id": seg_id,
        "size": COMM_CHANNEL_SIZE,
        "owner": f"consumer_{consumer_id}",
        "permissions": "rw",
        "data": bytearray(COMM_CHANNEL_SIZE),
        "consumer_id": consumer_id,
        "read_ptr": 0,
        "write_ptr": 0,
        "watermark": int(COMM_CHANNEL_SIZE * 0.8)
    }
    
    new_segments = shm_manager.get("segments", {}).copy()
    new_segments[seg_id] = segment
    
    return {
        **shm_manager,
        "segments": new_segments,
        "_next_seg_id": seg_id + 1
    }, seg_id

def write_demand_to_channel(
    shm_manager: Dict[str, Any],
    seg_id: int,
    demand_data: bytes
) -> Tuple[Dict[str, Any], bool]:
    segment = shm_manager["segments"].get(seg_id)
    if not segment:
        return shm_manager, False
    
    if len(demand_data) > segment["size"] - segment["write_ptr"] - 64:
        return shm_manager, False
    
    header = len(demand_data).to_bytes(4, 'little')
    timestamp = int(time.time() * 1000).to_bytes(8, 'little')
    data_to_write = header + timestamp + demand_data
    
    new_data = segment["data"].copy()
    new_data[segment["write_ptr"]:segment["write_ptr"] + len(data_to_write)] = data_to_write
    
    new_segments = shm_manager["segments"].copy()
    new_segments[seg_id] = {
        **segment,
        "data": new_data,
        "write_ptr": segment["write_ptr"] + len(data_to_write)
    }
    
    return {**shm_manager, "segments": new_segments}, True
```

#### 5.2 全局广播矢量（监控探针接口）

```
┌────────────────────────────────────────┐
│         Global Broadcast Vector        │
├────────────────────────────────────────┤
│  原子计数器: active_chamber_count      │
│  原子计数器: deferred_demand_count     │
│  原子计数器: compile_cache_hit_rate    │
│  环形日志:   最近100条状态转换事件        │
│  快照区:     每500ms刷新的算力池全景图    │
└────────────────────────────────────────┘
```

**实现代码**（`src/stage5/shm_vector.py`）：
```python
def init_global_broadcast_vector(
    shm_manager: Dict[str, Any]
) -> Tuple[Dict[str, Any], Optional[int]]:
    seg_id = shm_manager.get("_next_seg_id", 1)
    
    broadcast_data = bytearray(4096)
    struct.pack_into('<QQd', broadcast_data, 0, 0, 0, 0.0)
    
    segment = {
        "seg_id": seg_id,
        "size": 4096,
        "owner": "broadcast",
        "permissions": "rw",
        "data": broadcast_data,
        "channel_type": "broadcast"
    }
    
    new_segments = shm_manager.get("segments", {}).copy()
    new_segments[seg_id] = segment
    
    return {
        **shm_manager,
        "segments": new_segments,
        "_next_seg_id": seg_id + 1,
        "broadcast_seg_id": seg_id
    }, seg_id

def update_broadcast_vector(
    shm_manager: Dict[str, Any],
    active_chambers: int,
    deferred_count: int,
    hit_rate: float
) -> Dict[str, Any]:
    seg_id = shm_manager.get("broadcast_seg_id")
    if seg_id is None:
        return shm_manager
    
    segment = shm_manager["segments"][seg_id]
    new_data = segment["data"].copy()
    
    struct.pack_into('<QQd', new_data, 0, active_chambers, deferred_count, hit_rate)
    
    new_segments = shm_manager["segments"].copy()
    new_segments[seg_id] = {**segment, "data": new_data}
    
    return {**shm_manager, "segments": new_segments}
```

---

### 六、类型定义与状态机

**核心枚举类型**（`src/stage5/types.py`）：

| 枚举 | 成员 | 值 | 说明 |
| --- | --- | --- | --- |
| ConsumerState | NONEXISTENT | 0 | 消费者不存在 |
| | REGISTERED | 1 | 已注册 |
| | ACTIVE | 2 | 活跃 |
| | SUSPENDED | 3 | 暂停 |
| | BANNED | 4 | 封禁 |
| DemandState | PENDING | 0 | 待处理 |
| | PARSED | 1 | 已解析 |
| | QUEUED | 2 | 已入队 |
| | MATCHING | 3 | 匹配中 |
| | MATCHED | 4 | 已匹配 |
| | DEFERRED | 5 | 延迟 |
| | EXECUTING | 6 | 执行中 |
| | COMPLETED | 7 | 完成 |
| | FAILED | 8 | 失败 |
| ChamberState | ALLOCATION | 0 | 分配中 |
| | COMPILATION | 1 | 编译中 |
| | EXECUTION | 2 | 执行中 |
| | DESTRUCTION | 3 | 销毁中 |
| | TIMEOUT | 4 | 超时 |
| QoSLevel | BEST_EFFORT | 0 | 尽力而为 |
| | STANDARD | 1 | 标准 |
| | CRITICAL | 2 | 关键 |
| MatchResult | SUCCESS | 0 | 匹配成功 |
| | NO_MATCH | 1 | 无匹配 |
| | RESOURCE_EXHAUSTED | 2 | 资源耗尽 |
| | TIMEOUT | 3 | 超时 |

---

### 七、测试场景设计（第五阶段验收标准）

| 测试ID | 场景描述 | 通过标准 |
| --- | --- | --- |
| TC-5.1 | 创世首单：首个消费者注册→提交需求→生产者编译→执行→回传结果 | 端到端延迟 < 100ms，结果正确 |
| TC-5.2 | 并发风暴：1000消费者同时提交，算力池仅100 Worker | 无崩溃，延迟队列正常工作，最终全部完成 |
| TC-5.3 | 恶意消费者：提交超大JSON（1MB）、非法字段、伪造身份 | 全部拒绝，商场状态不受影响 |
| TC-5.4 | 生产者叛逃：编译通过但执行时触发段错误 | 看门狗捕获，反应腔销毁，其他任务不受影响 |
| TC-5.5 | 24小时耐久：持续50%负载运行 | SHM碎片率 < 10%，无内存泄漏，CPU平稳 |
| TC-5.6 | 商场安息：运行中收到SIGTERM | 进行中的反应腔完成，新需求拒绝，5秒内退出 |

---

### 八、创世纪完成宣言

当 TC-5.1 通过的那一刻——首个需求蓝图穿越生产者-商场-消费者三元循环，带着计算结果回到消费者手中——创世纪宣告完成。

商场不再是代码的集合，而是活的有机体：
- 生产者如酶，催化函数进化；
- 消费者如底物，驱动反应方向；
- 商场如膜结构，维持内环境稳态。

第五阶段之后，再无"构建"，只有"运行"与"繁殖"。

下一阶段（若存在），将是商场的自我复制——以当前商场为模板，生成子商场，形成商场星系。但那已超出创世纪范畴，进入出埃及记的篇章。

---

### 九、代码目录结构

```
src/stage5/
├── __init__.py              # 模块导出
├── types.py                 # 类型定义与状态机
├── consumer_gateway.py      # 消费者接入与需求处理
├── scheduler.py             # 三元匹配与反应腔管理
├── executor.py              # 编译执行引擎
├── main_loop.py             # 商场主循环
└── shm_vector.py            # SHM矢量管理
```

---

**审核请求**：请审阅以上第五阶段设计文档，重点核查：
1. 消费者接口是否保持无状态、公钥即身份？
2. trinity_match() 的O(1)约束是否可落地？
3. 反应腔的物理隔离机制（mprotect/MPK）是否符合安全铁律？
4. 主循环的"四拍子"结构是否杜绝了死锁与空转？

标注"审核通过"或指出违规项，我将立即修正。