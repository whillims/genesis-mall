# SHM 矢量管理接口深化 v1.0

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **文档定位**：商场-AICoder 体系的**内存宪法执行层**。  
> **核心原则**：SHM 是商场唯一的进程间通信机制；所有 SHM 操作必须原子化、可审计、可回滚；禁止指针嵌套，禁止动态扩容。  
> **哲学隐喻**：SHM 矢量不是"共享内存"，而是**命名状态管道**——它有方向（读/写）、有边界（容量）、有版本（乐观锁）、有主人（PID）。

---

## 一、SHM 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    SHM 全局命名空间                           │
├─────────────────────────────────────────────────────────────┤
│  shm://mall/global_state           → 商场全局状态              │
│  shm://mall/task_queue/pending     → 待处理任务队列            │
│  shm://mall/task_queue/assigned    → 已分配任务队列           │
│  shm://mall/auth_state/{doc_id}    → 授权文档生命周期状态      │
│  shm://mall/reports/{token}        → 最终报告持久化区          │
│  shm://mall/producer/{id}/mailbox  → 生产者通知信箱            │
├─────────────────────────────────────────────────────────────┤
│  shm://aicoder/{token}/input       → AICoder 任务输入          │
│  shm://aicoder/{token}/output      → AICoder 任务输出          │
│  shm://aicoder/{token}/state       → AICoder 任务状态          │
│  shm://aicoder/{token}/design/*      → 设计阶段产物              │
│  shm://aicoder/{token}/debug/*       → 调试阶段产物              │
│  shm://aicoder/{token}/evaluate/*    → 评估阶段产物              │
│  shm://aicoder/{token}/tmp/*         → 临时计算区（执行后销毁）   │
├─────────────────────────────────────────────────────────────┤
│  shm://producer/{id}/data/*          → 生产者数据区（只读给AICoder）│
│  shm://repo/{func_name}              → 代码仓库引用               │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、SHM 矢量生命周期函数

### 2.1 `shm_vector_create`

```c
/* 函数：创建命名 SHM 矢量
 * 契约：
 *   前置条件：key 为合法路径字符串（以 "shm://" 开头）；capacity > 0；
 *             当前进程拥有该命名空间的创建权限
 *   后置条件：SHM 矢量已创建，元数据区已初始化，数据区已清零
 *   副作用：
 *     1. 调用 shm_open(key, O_CREAT | O_RDWR, 0600) 创建 POSIX 共享内存对象
 *     2. ftruncate() 分配 capacity + sizeof(ShmVectorMeta) 字节
 *     3. mmap() 映射到进程地址空间
 *     4. 写入 ShmVectorMeta 初始值（version=0, used=0, owner_pid=0）
 *   并发安全：创建操作由商场主控进程串行执行，无并发竞争
 *   时间复杂度：O(1) —— 系统调用开销
 *   空间复杂度：O(capacity) —— 内核分配物理页
 */
int shm_vector_create(
    const char* key,                    /* [in] SHM 矢量键名 */
    u64 capacity,                      /* [in] 数据区容量（字节） */
    ShmVector* out_vector,             /* [out] 矢量句柄 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 键名合法性检查规则：
 * - 必须以 "shm://" 开头
 * - 总长度 <= MALL_MAX_PATH_LEN (256)
 * - 只允许字符：[a-zA-Z0-9_/.-]
 * - 禁止 ".." 序列（防止路径遍历）
 * - 禁止空段（如 "shm://a//b"）
 * - 必须以字母或数字结尾
 */
```

---

### 2.2 `shm_vector_open`

```c
/* 函数：打开已存在的 SHM 矢量
 * 契约：
 *   前置条件：key 指向已创建的 SHM 矢量；mode 为合法访问模式
 *   后置条件：矢量已映射到进程地址空间；out_vector 包含有效句柄
 *   副作用：
 *     1. shm_open(key, flags, 0600) —— flags 由 mode 决定
 *     2. mmap() 映射
 *     3. 若 mode == SHM_MODE_WRITE 或 SHM_MODE_READWRITE，更新 meta.owner_pid
 *   并发安全：映射操作本身线程安全；owner_pid 更新使用原子操作
 *   时间复杂度：O(1)
 *   空间复杂度：O(1) —— 仅创建进程页表项
 */
int shm_vector_open(
    const char* key,                    /* [in] SHM 矢量键名 */
    ShmAccessMode mode,                /* [in] 访问模式 */
    ShmVector* out_vector,             /* [out] 矢量句柄 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 访问模式映射到 open flags：
 * SHM_MODE_READ       → O_RDONLY
 * SHM_MODE_WRITE      → O_WRONLY
 * SHM_MODE_READWRITE  → O_RDWR
 *
 * 权限检查：
 * - 打开前检查当前进程的 WorkerShmMap 是否包含该 key
 * - 若 mode 为 WRITE 但 WorkerShmMap.write_keys 不包含 → MALL_ERR_SHM_ACCESS_DENIED
 * - 若 mode 为 READ 但 WorkerShmMap.read_keys 不包含 → MALL_ERR_SHM_ACCESS_DENIED
 */
```

---

### 2.3 `shm_vector_close`

```c
/* 函数：关闭 SHM 矢量
 * 契约：
 *   前置条件：vector 为有效打开的句柄
 *   后置条件：映射已解除；若当前进程为 owner，owner_pid 清零
 *   副作用：
 *     1. munmap() 解除映射
 *     2. 若 vector->meta.owner_pid == getpid()，原子清零 owner_pid
 *     3. close() 文件描述符
 *   并发安全：owner_pid 清零使用原子操作
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int shm_vector_close(
    ShmVector* vector,                 /* [in,out] 矢量句柄（关闭后置空） */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

### 2.4 `shm_vector_destroy`

```c
/* 函数：销毁 SHM 矢量
 * 契约：
 *   前置条件：key 指向已创建的 SHM 矢量；当前进程为创建者或商场主控进程
 *   后置条件：SHM 对象已从系统中删除，所有映射失效
 *   副作用：
 *     1. shm_unlink(key) 删除 POSIX 共享内存对象
 *     2. 向所有持有该矢量的进程发送 SIGUSR1（通知映射失效）
 *   并发安全：销毁操作由商场主控串行执行；持有进程收到 SIGUSR1 后必须调用 shm_vector_close
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int shm_vector_destroy(
    const char* key,                    /* [in] SHM 矢量键名 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 销毁策略（按场景）：
 * 场景1: 任务完成 → 销毁 shm://aicoder/{token}/tmp/* 临时区
 * 场景2: 任务回滚 → 销毁 shm://aicoder/{token}/* 所有临时区
 * 场景3: 报告归档 → 保留 shm://mall/reports/{token}，销毁其他区域
 * 场景4: TTL 过期 → 商场定时任务自动销毁所有相关 SHM
 */
```

---

## 三、SHM 矢量读写函数

### 3.1 `shm_vector_read`

```c
/* 函数：从 SHM 矢量读取数据
 * 契约：
 *   前置条件：vector 已以 READ 或 READWRITE 模式打开；offset < vector->meta.capacity
 *   后置条件：buffer 包含从 SHM 读取的数据；out_bytes_read 包含实际读取字节数
 *   副作用：无 SHM 写入；仅读取
 *   并发安全：乐观锁机制——读取前检查 version，读取后校验 version 是否变化；
 *             若变化则重试（最多 3 次），超过则返回 MALL_ERR_SHM_CORRUPTION
 *   时间复杂度：O(bytes) —— 内存拷贝
 *   空间复杂度：O(1)
 */
int shm_vector_read(
    const ShmVector* vector,           /* [in] 矢量句柄 */
    u64 offset,                        /* [in] 数据区偏移 */
    u64 bytes,                         /* [in] 请求读取字节数 */
    void* buffer,                      /* [out] 用户缓冲区 */
    u64* out_bytes_read,              /* [out] 实际读取字节数 */
    struct MallError* out_err          /* [out] 错误详情 */
);

/* 乐观锁读取算法：
 * int retries = 3;
 * while (retries-- > 0) {
 *     u64 version_before = atomic_load(&vector->meta.version);
 *     u64 used = atomic_load(&vector->meta.used);
 *     if (offset + bytes > used) bytes = used - offset; // 截断到有效数据区
 *     memcpy(buffer, vector->data + offset, bytes);
 *     u64 version_after = atomic_load(&vector->meta.version);
 *     if (version_before == version_after) {
 *         *out_bytes_read = bytes;
 *         return MALL_OK; // 读取成功
 *     }
 * }
 * return MALL_ERR_SHM_CORRUPTION; // 重试耗尽
 */
```

---

### 3.2 `shm_vector_write`

```c
/* 函数：向 SHM 矢量写入数据
 * 契约：
 *   前置条件：vector 已以 WRITE 或 READWRITE 模式打开；
 *             offset + bytes <= vector->meta.capacity；
 *             当前进程为 owner_pid 或 owner_pid == 0（无主）
 *   后置条件：数据已写入 SHM；meta.used 已更新；meta.version 已递增
 *   副作用：
 *     1. 原子递增 version（作为写入开始标记）
 *     2. memcpy() 写入数据
 *     3. 原子更新 used = max(used, offset + bytes)
 *     4. 原子递增 version（作为写入完成标记）
 *   并发安全：version 双递增机制确保读者能检测到写入中状态；
 *             owner_pid 检查防止未授权进程写入
 *   时间复杂度：O(bytes)
 *   空间复杂度：O(1)
 */
int shm_vector_write(
    ShmVector* vector,                 /* [in] 矢量句柄 */
    u64 offset,                        /* [in] 数据区偏移 */
    const void* data,                  /* [in] 数据源 */
    u64 bytes,                         /* [in] 写入字节数 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 写入算法：
 * // 1. 权限检查
 * if (vector->meta.owner_pid != 0 && vector->meta.owner_pid != getpid())
 *     return MALL_ERR_SHM_ACCESS_DENIED;
 * 
 * // 2. 容量检查
 * if (offset + bytes > vector->meta.capacity)
 *     return MALL_ERR_SHM_ALLOC_FAIL;
 * 
 * // 3. 原子标记写入开始
 * atomic_fetch_add(&vector->meta.version, 1);
 * 
 * // 4. 写入数据
 * memcpy(vector->data + offset, data, bytes);
 * 
 * // 5. 更新 used（原子取最大值）
 * u64 new_used = offset + bytes;
 * u64 old_used = atomic_load(&vector->meta.used);
 * while (new_used > old_used) {
 *     if (atomic_compare_exchange_weak(&vector->meta.used, &old_used, new_used))
 *         break;
 * }
 * 
 * // 6. 原子标记写入完成
 * atomic_fetch_add(&vector->meta.version, 1);
 * 
 * return MALL_OK;
 */
```

---

### 3.3 `shm_vector_clear`

```c
/* 函数：清空 SHM 矢量数据区
 * 契约：
 *   前置条件：vector 已以 WRITE 或 READWRITE 模式打开；当前进程为 owner
 *   后置条件：数据区已清零（memset 0）；meta.used = 0；meta.version += 2
 *   副作用：原子清零操作
 *   并发安全：同 shm_vector_write
 *   时间复杂度：O(capacity) —— 全量清零
 *   空间复杂度：O(1)
 */
int shm_vector_clear(
    ShmVector* vector,                 /* [in] 矢量句柄 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

## 四、SHM 结构体序列化函数

### 4.1 `shm_serialize_auth_document`

```c
/* 函数：授权文档序列化到 SHM
 * 契约：
 *   前置条件：auth_doc 为有效填充的结构体；vector 已打开且容量 >= sizeof(AuthDocument)
 *   后置条件：AuthDocument 已扁平化为字节流写入 SHM
 *   副作用：调用 shm_vector_write()
 *   并发安全：由写入端保证
 *   时间复杂度：O(sizeof(AuthDocument))
 *   空间复杂度：O(1)
 */
int shm_serialize_auth_document(
    const AuthDocument* auth_doc,       /* [in] 授权文档 */
    ShmVector* vector,                 /* [in,out] 目标 SHM 矢量 */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 序列化布局（小端序，无填充）：
 * Offset 0:   AuthHeader (固定大小)
 * Offset 256: PermissionVector (固定大小)
 * Offset 272: FunctionSpec (固定大小)
 * Offset ...: SandboxConfig (固定大小)
 * Offset ...: AuthSignature (固定大小)
 * Offset ...: checksum (u32)
 * 
 * 总大小: 约 8KB，精确计算为 sizeof(AuthDocument)
 */
```

---

### 4.2 `shm_deserialize_auth_document`

```c
/* 函数：从 SHM 反序列化授权文档
 * 契约：
 *   前置条件：vector 包含有效的序列化授权文档
 *   后置条件：auth_doc 已填充
 *   副作用：调用 shm_vector_read()
 *   并发安全：乐观锁读取
 *   时间复杂度：O(sizeof(AuthDocument))
 *   空间复杂度：O(1)
 */
int shm_deserialize_auth_document(
    const ShmVector* vector,           /* [in] 源 SHM 矢量 */
    AuthDocument* auth_doc,            /* [out] 授权文档 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

### 4.3 `shm_serialize_final_report`

```c
/* 函数：最终报告序列化到 SHM
 * 契约：
 *   前置条件：report 为有效填充的结构体；vector 容量 >= sizeof(FinalReport)
 *   后置条件：FinalReport 已写入 SHM
 *   副作用：调用 shm_vector_write()
 *   并发安全：由写入端保证
 *   时间复杂度：O(sizeof(FinalReport))
 *   空间复杂度：O(1)
 */
int shm_serialize_final_report(
    const FinalReport* report,          /* [in] 最终报告 */
    ShmVector* vector,                 /* [in,out] 目标 SHM 矢量 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

## 五、SHM 批量管理函数

### 5.1 `shm_task_namespace_create`

```c
/* 函数：创建任务命名空间（批量创建任务相关 SHM 区域）
 * 契约：
 *   前置条件：task_token 为有效 UUID；auth_doc 已验签
 *   后置条件：任务所需的所有 SHM 区域已创建
 *   副作用：
 *     1. 创建 shm://aicoder/{token}/input
 *     2. 创建 shm://aicoder/{token}/output
 *     3. 创建 shm://aicoder/{token}/state
 *     4. 创建 shm://aicoder/{token}/design/code (8KB)
 *     5. 创建 shm://aicoder/{token}/design/rationale (2KB)
 *     6. 创建 shm://aicoder/{token}/debug/patch_diff (4KB)
 *     7. 创建 shm://aicoder/{token}/evaluate/score_card (1KB)
 *     8. 创建 shm://aicoder/{token}/tmp/ (预留 16KB 临时区)
 *   并发安全：由 mall_aicoder_dispatch() 串行调用
 *   时间复杂度：O(1) —— 固定 8 个创建操作
 *   空间复杂度：O(1) —— 固定预分配
 */
int shm_task_namespace_create(
    const char* task_token,             /* [in] 任务令牌 */
    const AuthDocument* auth_doc,       /* [in] 授权文档（用于计算容量需求） */
    struct MallError* out_err           /* [out] 错误详情 */
);

/* 容量计算规则：
 * input 区:  max(4KB, sizeof(FunctionSpec) + sizeof(TestSuite))
 * output 区: max(8KB, sizeof(DesignOutput) + sizeof(DebugOutput) + sizeof(EvaluateOutput))
 * state 区: sizeof(TaskState)
 * design/code: 8KB（固定）
 * design/rationale: 2KB（固定）
 * debug/patch_diff: 4KB（固定）
 * evaluate/score_card: 1KB（固定）
 * tmp/: 16KB（固定，动态计算时溢出 → 截断）
 */
```

---

### 5.2 `shm_task_namespace_destroy`

```c
/* 函数：销毁任务命名空间（批量删除任务相关 SHM 区域）
 * 契约：
 *   前置条件：task_token 对应任务已完成或已回滚
 *   后置条件：所有临时 SHM 区域已销毁；报告区保留（如果存在）
 *   副作用：
 *     1. 销毁 input、output、state、design/*、debug/*、evaluate/*、tmp/*
 *     2. 保留 mall/reports/{token}（报告持久化区）
 *   并发安全：由 mall_pipeline_rollback() 或 mall_task_cleanup() 串行调用
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int shm_task_namespace_destroy(
    const char* task_token,             /* [in] 任务令牌 */
    mall_bool preserve_report,         /* [in] 是否保留报告区 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

### 5.3 `shm_task_namespace_preserve`

```c
/* 函数：归档任务命名空间（将关键产物复制到持久化区）
 * 契约：
 *   前置条件：task_token 对应任务已完成；report 已生成
 *   后置条件：关键产物已复制到 shm://mall/reports/{token}/
 *   副作用：
 *     1. 创建 shm://mall/reports/{token}/report.md
 *     2. 创建 shm://mall/reports/{token}/metadata.json
 *     3. 复制 design/code 到 shm://mall/reports/{token}/artifacts/code.py
 *     4. 复制 debug/patch_diff 到 shm://mall/reports/{token}/artifacts/patch.diff
 *     5. 复制 evaluate/score_card 到 shm://mall/reports/{token}/artifacts/score.json
 *   并发安全：由 mall_generate_aicoder_report() 串行调用
 *   时间复杂度：O(total_artifact_size)
 *   空间复杂度：O(1)
 */
int shm_task_namespace_preserve(
    const char* task_token,             /* [in] 任务令牌 */
    const FinalReport* report,          /* [in] 最终报告 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

## 六、SHM 监控与诊断函数

### 6.1 `shm_vector_inspect`

```c
/* 函数：SHM 矢量诊断检查
 * 契约：
 *   前置条件：key 指向已创建的 SHM 矢量
 *   后置条件：返回矢量元数据和健康状况
 *   副作用：无写入操作，仅读取元数据
 *   并发安全：纯读取，线程安全
 *   时间复杂度：O(1)
 *   空间复杂度：O(1)
 */
int shm_vector_inspect(
    const char* key,                    /* [in] SHM 矢量键名 */
    struct ShmVectorMeta* out_meta,     /* [out] 元数据副本 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

### 6.2 `shm_vector_checksum_verify`

```c
/* 函数：SHM 数据区校验和验证
 * 契约：
 *   前置条件：vector 已打开
 *   后置条件：返回校验和是否匹配
 *   副作用：计算 CRC32（纯 CPU 操作）
 *   并发安全：计算期间数据可能被修改（乐观锁检测）
 *   时间复杂度：O(used)
 *   空间复杂度：O(1)
 */
int shm_vector_checksum_verify(
    const ShmVector* vector,           /* [in] 矢量句柄 */
    mall_bool* out_valid,              /* [out] 校验和是否有效 */
    struct MallError* out_err           /* [out] 错误详情 */
);
```

---

## 七、审核宣言

> SHM 矢量管理是商场模式的**物理层**。  
> 没有严格的 SHM 管理，函数级隔离就是空谈；没有原子化读写，状态机就是幻觉；没有命名空间管控，进程间通信就会退化为混乱的管道 spaghetti。  
> 每个 SHM 矢量都是**有主权的领土**——它有边界、有主人、有方向、有版本。AICoder Worker 只能在商场划定的领土上活动，任何越界访问都是**主权侵犯**，必须立即驱逐。  
> **审核状态**：待人类架构师最终裁定。
