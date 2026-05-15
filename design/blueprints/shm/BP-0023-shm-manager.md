# BP-0004-shm-manager.md

> **蓝图编号**：BP-0004  
> **产品名称**：shm-manager（共享内存管理器）  
> **产品类型**：基础设施产品  
> **优先级**：P0  
> **依赖**：无（商场最底层设施）  
> **版本**：v1.0  
> **状态**：待实现

---

## 1. 产品概述

`shm-manager` 是商场的**物理层基础设施**，负责 `multiprocessing.shared_memory` 的全生命周期管理。它制定并执行**SHM内存宪法**——包括段创建/销毁、矢量分配与回收、访问权限控制、内存状态机形式化验证。

所有上层产品（生产者、消费者、通道、路由器）均通过 `shm-manager` 提供的函数接口操作SHM，**禁止任何产品直接调用操作系统底层内存API**。

---

## 2. 设计约束

| 约束项 | 规则 |
|--------|------|
| 编程范式 | 纯函数集群，禁止类、继承、封装、多态 |
| 内存模型 | 仅使用 `multiprocessing.shared_memory.SharedMemory`，禁止 `mmap` 直接调用 |
| 权限模型 | 只读段（RO）与读写段（RW）严格分离，段头描述权限，运行时校验 |
| 并发安全 | 所有指针更新必须使用原子操作或商场全局锁（SHM锁段） |
| 故障策略 | 段损坏时进入 `CORRUPTED` 状态，触发错误守卫（BP-0007）隔离 |
| 碎片管理 | 采用固定块分配策略，禁止动态堆分配导致碎片 |

---

## 3. 关键词定义

| 术语 | 定义 |
|------|------|
| **SHM根段** | 商场主进程创建的第一个共享内存段，包含全局目录表与配置区 |
| **矢量（Vector）** | SHM中的逻辑数据单元，由 `[基地址, 长度, 类型, 所有者ID]` 四元组描述 |
| **段目录表** | 位于SHM根段起始处的数组，记录所有子段的名称、大小、创建者、状态 |
| **内存宪法** | 不可变更的元数据规则：段名格式、权限掩码、对齐要求、生命周期钩子 |
| **固定块池** | 预分配的等大小内存块（如4KB/块），用于小对象分配，避免碎片 |
| **SHM锁段** | 专用段，存储自旋锁或信号量，用于保护目录表与关键元数据 |

---

## 4. 接口定义

### 4.1 段生命周期管理

```c
// 创建商场SHM根段，初始化目录表与宪法区
int shm_create_root(
    const char* mall_name,      // 商场唯一标识名（如 "mall-genesis-01"）
    uint64_t root_size,         // 根段总字节数（最小 64KB）
    uint32_t max_segments,      // 最大子段数量
    SharedMemory** out_root     // 出参：根段句柄
);

// 创建子段（由商场或授权产品调用）
int shm_create_segment(
    SharedMemory* root,         // SHM根段句柄
    const char* seg_name,       // 子段名（格式：{product_id}-{purpose}）
    uint64_t seg_size,          // 子段大小
    uint8_t permission,         // 权限掩码：bit0=可读, bit1=可写, bit2=可执行（禁止）
    uint16_t owner_id,          // 创建者产品ID
    SharedMemory** out_seg      // 出参：子段句柄
);

// 销毁子段（仅段所有者或商场主进程可调用）
int shm_destroy_segment(
    SharedMemory* root,
    const char* seg_name,
    uint16_t caller_id
);

// 销毁根段（商场关闭时调用，级联销毁所有子段）
int shm_destroy_root(SharedMemory* root);
```

### 4.2 矢量管理

```c
// 在指定段内分配一个矢量
int shm_vector_alloc(
    SharedMemory* seg,          // 目标段
    uint32_t vec_size,          // 矢量字节数
    uint8_t vec_type,           // 类型码：0=原始数据, 1=控制块, 2=日志, 3=配置
    uint16_t owner_id,
    uint64_t* out_offset        // 出参：矢量在段内的偏移地址
);

// 回收矢量（标记为FREE，不立即清零，由垃圾回收策略处理）
int shm_vector_free(
    SharedMemory* seg,
    uint64_t offset,
    uint16_t caller_id
);

// 查询矢量信息
int shm_vector_query(
    SharedMemory* seg,
    uint64_t offset,
    uint32_t* out_size,
    uint8_t* out_type,
    uint16_t* out_owner,
    uint8_t* out_status        // 0=FREE, 1=ALLOCATED, 2=LOCKED
);
```

### 4.3 权限与校验

```c
// 校验调用者是否有权限访问指定段/矢量
int shm_check_permission(
    SharedMemory* root,
    const char* seg_name,
    uint64_t offset,
    uint16_t caller_id,
    uint8_t required_perm       // 所需权限：0=读, 1=写
);

// 获取SHM健康状态
int shm_health_check(
    SharedMemory* root,
    uint32_t* out_total_segments,
    uint32_t* out_used_segments,
    uint32_t* out_corrupted_segments,
    uint8_t* out_root_status   // 0=HEALTHY, 1=DEGRADED, 2=CORRUPTED
);
```

---

## 5. SHM数据结构设计

### 5.1 根段布局（前4KB为宪法区）

```c
// 根段基地址 + 0x0000
struct shm_constitution {
    uint64_t magic;              // "MALLSHM" = 0x4D414C4C53484D00
    uint32_t version;            // 0x00010000
    uint32_t constitution_size;  // 本结构大小（固定4096字节）
    uint64_t root_size;          // 根段总大小
    uint32_t max_segments;       // 最大子段数
    uint32_t segment_count;      // 当前子段数
    uint8_t  root_status;        // 0=HEALTHY, 1=DEGRADED, 2=CORRUPTED
    uint8_t  reserved[7];
    // ... 填充至 256 字节
};

// 根段基地址 + 0x0100：段目录表（每项32字节）
struct segment_directory_entry {
    char     seg_name[16];       // 子段名（null-terminated）
    uint64_t seg_size;           // 子段大小
    uint16_t owner_id;           // 创建者ID
    uint8_t  permission;         // 权限掩码
    uint8_t  status;             // 0=FREE, 1=ACTIVE, 2=MARKED_FOR_DELETE
    uint8_t  reserved[4];
};
// 最多支持 (4096-256)/32 = 120 个子段（当 max_segments=120 时）

// 根段基地址 + 0x1000：SHM锁区
struct shm_lock_region {
    uint32_t directory_lock;     // 目录表自旋锁（0=空闲, 1=占用）
    uint32_t allocator_lock;     // 分配器自旋锁
    uint32_t health_lock;        // 健康检查锁
    uint8_t  reserved[4];
};
```

### 5.2 子段布局

```c
// 子段基地址 + 0x0000：段头（64字节）
struct segment_header {
    uint64_t magic;              // "SEGMENT" = 0x5345474D454E5400
    char     seg_name[16];       // 段名副本
    uint64_t seg_size;           // 本段大小
    uint16_t owner_id;
    uint8_t  permission;
    uint8_t  status;
    uint32_t vector_count;       // 当前矢量数
    uint32_t free_block_count;   // 空闲块数
    uint8_t  reserved[12];
};

// 子段基地址 + 0x0040：矢量分配表（每项32字节）
struct vector_entry {
    uint64_t offset;             // 矢量在段内偏移
    uint32_t size;               // 矢量大小
    uint8_t  type;               // 类型码
    uint8_t  status;             // 0=FREE, 1=ALLOCATED, 2=LOCKED
    uint16_t owner_id;
    uint8_t  reserved[8];
};
```

---

## 6. 函数详细设计

### 6.1 `shm_create_root`

**逻辑流程**：
1. 校验 `mall_name` 非空，`root_size >= 65536`，`max_segments >= 1`
2. 调用 `multiprocessing.shared_memory.SharedMemory(name=mall_name, create=True, size=root_size)`
3. 清零整个段
4. 写入 `shm_constitution`：magic、version、constitution_size、root_size、max_segments
5. 初始化 `segment_count = 0`，`root_status = HEALTHY`
6. 初始化段目录表为全FREE
7. 初始化SHM锁区（所有锁置0）
8. 返回根段句柄

**返回值**：
- `0`：成功
- `-EINVAL`：参数非法
- `-EEXIST`：同名SHM已存在
- `-ENOMEM`：系统内存不足

### 6.2 `shm_create_segment`

**逻辑流程**：
1. 校验 `root` 非空，`seg_name` 符合格式 `[A-Za-z0-9_-]{1,15}`
2. 获取 `directory_lock`，遍历目录表检查重名
3. 查找第一个 `status == FREE` 的目录项
4. 调用底层API创建子段 `SharedMemory(name=seg_name, create=True, size=seg_size)`
5. 写入目录项：`seg_name`、`seg_size`、`owner_id`、`permission`、`status = ACTIVE`
6. 初始化子段头（magic、seg_name副本等）
7. 递增 `segment_count`
8. 释放 `directory_lock`
9. 返回子段句柄

**返回值**：
- `0`：成功
- `-EINVAL`：参数非法或格式错误
- `-EEXIST`：段名已存在
- `-ENOSPC`：目录表已满

### 6.3 `shm_vector_alloc`

**逻辑流程**：
1. 校验 `seg` 非空，`vec_size > 0`
2. 读取子段头，校验 `magic`
3. 获取 `allocator_lock`
4. 遍历矢量分配表，查找 `status == FREE` 且 `size >= vec_size` 的项（最佳匹配）
5. 若无合适项，在段尾追加新矢量（检查不越界）
6. 设置矢量项：`offset`、`size`、`type`、`status = ALLOCATED`、`owner_id`
7. 递增 `vector_count`
8. 释放 `allocator_lock`
9. 返回偏移地址

**返回值**：
- `0`：成功
- `-ENOSPC`：段空间不足
- `-EBADSEG`：段头损坏

### 6.4 `shm_check_permission`

**逻辑流程**：
1. 在目录表中查找 `seg_name`
2. 校验目录项 `status == ACTIVE`
3. 比较 `caller_id` 与 `owner_id`：
   - 若相等，授予全部权限
   - 若不相等，检查 `permission` 掩码与 `required_perm`
4. 若 `required_perm == 写` 且段为RO，返回 `-EACCES`
5. 返回0（允许）

---

## 7. 测试场景

### 7.1 单元测试：根段创建与销毁

```
测试名：test_shm_root_lifecycle
步骤：
  1. 调用 shm_create_root("test-mall", 65536, 16, &root)
  2. 校验 root 非空
  3. 校验宪法区 magic = 0x4D414C4C53484D00
  4. 调用 shm_destroy_root(root)
  5. 尝试再次访问（应失败）
断言：
  - 创建返回0
  - 销毁返回0
  - 重复访问返回 -ENOENT
```

### 7.2 压力测试：批量段创建

```
测试名：test_shm_mass_segments
步骤：
  1. 创建根段（max_segments=120）
  2. 循环创建 120 个子段，每段 4KB
  3. 尝试创建第121个段
断言：
  - 前120次均返回0
  - 第121次返回 -ENOSPC
  - segment_count == 120
```

### 7.3 并发测试：矢量分配竞争

```
测试名：test_shm_vector_race
步骤：
  1. 创建根段与子段（size=1MB）
  2. 启动4个Worker进程，每个并发分配 100 个矢量
  3. 所有Worker完成后检查
断言：
  - 总分配矢量数 == 400
  - 无重叠偏移
  - 无段头损坏
```

### 7.4 异常测试：权限越界

```
测试名：test_shm_permission_violation
步骤：
  1. 创建根段
  2. 以 owner_id=0x0001 创建RO段 "config"
  3. 以 caller_id=0x0002 请求写入 "config"
断言：
  - shm_check_permission 返回 -EACCES
  - 错误日志记录越界事件（含 caller_id 与 seg_name）
```

---

## 8. 依赖与授权

| 项 | 说明 |
|----|------|
| 前置蓝图 | 无（最底层设施） |
| 后置蓝图 | 所有其他BP均依赖本设施 |
| AICoder授权 | 需外置AICoder生成授权文件 `AUTH-BP-0004.json`，含内存宪法不可变更条款 |
| 商场加载 | 由商场 `main()` 在创世第1阶段最先加载，作为基础设施初始化 |

---

## 9. 附录：SHM内存宪法（不可变更条款）

```
宪法第1条：所有SHM段名必须以商场名称为前缀，格式 {mall_name}-{product_id}-{purpose}
宪法第2条：禁止创建可执行权限（bit2）段，违者商场立即终止该Worker
宪法第3条：根段宪法区（前4KB）为只读，任何写入请求返回 -EACCES
宪法第4条：段目录表修改必须持有 directory_lock，超时（1秒）未释放视为死锁，触发BP-0007
宪法第5条：子段销毁后，目录项状态置为 MARKED_FOR_DELETE，延迟10秒物理释放（防止悬停指针）
宪法第6条：矢量分配必须8字节对齐，未对齐请求自动向上取整
```

---

*蓝图审核状态：待审核*  
*设计者：创世Worker*  
*AICoder评估：待外置AICoder审查*
