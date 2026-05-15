# 商场异常处理设计蓝图

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

## 一、蓝图概述

本蓝图实现《商场异常处理总则》的函数级落地。作为商场核心子系统，异常处理模块以**纯函数集合**形式存在，通过SHM状态区与商场主进程（main()）交互，不持有私有状态，不创建独立进程。

**蓝图编号**：`BLUEPRINT-EXCEPTION-001`  
**版本**：`v1.0`  
**依赖**：`商场SHM设计蓝图 v1.0`、`商场main()设计蓝图 v1.0`  
**AICoder审查状态**：待外置AICoder评估

---

## 二、架构关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        商场主进程 main()                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ 调度循环    │  │ SHM管理器   │  │ 异常处理核心模块    │  │
│  │ scheduler   │◄─┤  shm_core   │◄─┤ anomaly_core        │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                    │             │
│         └────────────────┴────────────────────┘             │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              SHM 共享内存空间（商场管理）                │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │  │
│  │  │ 正常数据区 │ │ 异常事件区 │ │ 隔离取证区 │ │ 状态灯塔区 │ │  │
│  │  │ DataZone │ │ AER Zone │ │ QZone    │ │ Beacon   │ │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   外置 AICoder（天眼）                         │
│              接收异常归档 / 重新评估熔断实体                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、SHM内存布局扩展

基于《商场SHM设计蓝图》，新增以下区域：

| 区域名称 | 偏移基址 | 大小 | 用途 |
|----------|----------|------|------|
| `AER_RING_BUFFER` | `SHM_BASE + 2MB` | 512KB | 环形异常事件记录缓冲区，覆盖式写入 |
| `QUARANTINE_ZONE` | `SHM_BASE + 2.5MB` | 256KB | SHM污染取证隔离区 |
| `BEACON_STATUS` | `SHM_BASE + 2.75MB` | 64KB | 实体健康状态灯塔（心跳标记位图） |
| `ANOMALY_REGISTRY` | `SHM_BASE + 2.81MB` | 128KB | 当前活跃异常登记表 |

---

## 四、核心函数设计

### 4.1 异常检测层（Detection Layer）

```c
/* anomaly_detect.h - 函数级接口，禁止OOP */

/**
 * @brief 扫描SHM写操作边界，检测越界污染
 * @param entity_id 实体唯一标识
 * @param write_offset 实体声明的写入偏移
 * @param write_size 实体声明的写入大小
 * @param actual_offset 实际写入偏移
 * @param actual_size 实际写入大小
 * @return 异常码，0表示无异常
 */
int anomaly_detect_shm_pollution(
    int entity_id,
    size_t write_offset,
    size_t write_size,
    size_t actual_offset,
    size_t actual_size
);

/**
 * @brief 检测实体心跳超时（幽灵进程识别）
 * @param entity_id 实体唯一标识
 * @param last_beat_ns 上次心跳时间戳（纳秒）
 * @param timeout_ns 蓝图约定超时阈值
 * @return 1=存活, 0=超时, -1=实体未注册
 */
int anomaly_detect_heartbeat_loss(
    int entity_id,
    uint64_t last_beat_ns,
    uint64_t timeout_ns
);

/**
 * @brief 监控Queue水位，预防堆积溢出
 * @param queue_depth 当前队列深度
 * @param threshold_warning 预警阈值
 * @param threshold_critical 临界阈值
 * @return 0=正常, 1=预警, 2=临界
 */
int anomaly_detect_queue_pressure(
    size_t queue_depth,
    size_t threshold_warning,
    size_t threshold_critical
);
```

### 4.2 异常分级与记录层（Classification & Logging Layer）

```c
/* anomaly_classify.h */

/**
 * @brief 根据异常性质与影响范围判定等级
 * @param anomaly_code 原始异常码
 * @param affected_entities 受影响实体数量
 * @param shm_corruption_flag SHM是否受损
 * @return 分级结果: 0=Info, 1=Warning, 2=Critical, 3=Fatal
 */
int anomaly_classify_level(
    const char* anomaly_code,
    int affected_entities,
    int shm_corruption_flag
);

/**
 * @brief 向AER环形缓冲区写入异常事件记录
 * @param record 异常事件结构体指针（纯数据，无方法）
 * @param shm_aer_base AER区基址
 * @return 写入偏移，-1表示缓冲区满（触发强制归档）
 */
int anomaly_aer_write(
    const struct AnomalyEventRecord* record,
    void* shm_aer_base
);

/**
 * @brief 从AER缓冲区读取最近N条记录用于诊断
 * @param shm_aer_base AER区基址
 * @param count 请求记录数
 * @param output_buffer 输出缓冲区（由调用者分配）
 * @return 实际读取记录数
 */
int anomaly_aer_read_recent(
    void* shm_aer_base,
    int count,
    struct AnomalyEventRecord* output_buffer
);
```

### 4.3 处置执行层（Execution Layer）

```c
/* anomaly_execute.h */

/**
 * @brief 对指定实体执行熔断（切断SHM访问与调度）
 * @param entity_id 目标实体
 * @param fuse_reason 熔断原因码
 * @param shm_beacon_base 灯塔区基址（更新状态为FUSED）
 * @return 0=成功, -1=实体不存在, -2=已是熔断态
 */
int anomaly_execute_fuse(
    int entity_id,
    const char* fuse_reason,
    void* shm_beacon_base
);

/**
 * @brief 启动SHM隔离区取证
 * @param polluted_offset 污染数据起始偏移
 * @param polluted_size 污染数据大小
 * @param shm_quarantine_base 隔离区基址
 * @param out_qzone_offset 输出隔离区新偏移
 * @return 0=成功, -1=隔离区满
 */
int anomaly_execute_quarantine(
    size_t polluted_offset,
    size_t polluted_size,
    void* shm_quarantine_base,
    size_t* out_qzone_offset
);

/**
 * @brief 清除幽灵进程并回收资源
 * @param pid 进程ID
 * @param entity_id 实体ID（用于SHM令牌回收）
 * @param shm_registry_base 异常登记表基址
 * @return 0=成功清除, 1=进程已不存在, -1=权限不足
 */
int anomaly_execute_ghost_purge(
    int pid,
    int entity_id,
    void* shm_registry_base
);

/**
 * @brief 商场全局降级模式切换
 * @param degrade_level 降级等级: 1=轻降(暂停准入), 2=中降(驱逐消费者), 3=重降(保留核心)
 * @param shm_beacon_base 灯塔区基址（广播降级信号）
 * @return 前一等级（用于恢复时回退）
 */
int anomaly_execute_degrade(
    int degrade_level,
    void* shm_beacon_base
);
```

### 4.4 自愈与恢复层（Recovery Layer）

```c
/* anomaly_recovery.h */

/**
 * @brief 尝试对一般级异常实体进行自愈重启
 * @param entity_id 目标实体
 * @param blueprint_hash 实体蓝图哈希（用于重新加载）
 * @param retry_count 当前重试次数
 * @param max_retry 最大重试阈值
 * @return 0=自愈成功, 1=重试次数耗尽, 2=蓝图损坏需重新授权
 */
int anomaly_recovery_self_heal(
    int entity_id,
    const char* blueprint_hash,
    int retry_count,
    int max_retry
);

/**
 * @brief 从降级模式恢复至正常模式
 * @param target_level 目标恢复等级（通常为0）
 * @param shm_beacon_base 灯塔区基址
 * @return 0=成功, -1=条件不满足（如AICoder仍失联）
 */
int anomaly_recovery_degrade_restore(
    int target_level,
    void* shm_beacon_base
);

/**
 * @brief 异常事件归档至外置存储（供AICoder分析）
 * @param aer_buffer AER记录数组
 * @param record_count 记录数量
 * @param archive_path 归档路径（商场配置项）
 * @return 成功归档数量
 */
int anomaly_recovery_archive(
    const struct AnomalyEventRecord* aer_buffer,
    int record_count,
    const char* archive_path
);
```

---

## 五、状态灯塔（Beacon）设计

灯塔区使用位图+状态字描述每个实体健康状态：

```
每个实体占用 8 字节：
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ 状态字  │  心跳计数 │  异常码  │  异常次数 │  熔断标记 │  降级标记 │  保留   │  保留   │
│ 1 byte │ 1 byte │ 2 bytes│ 1 byte │ 1 byte │ 1 byte │ 1 byte │ 1 byte │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘

状态字枚举（函数常量）：
0x00 = NORMAL    (正常)
0x01 = WARNING   (预警)
0x02 = FUSED     (熔断)
0x03 = GHOST     (幽灵)
0x04 = DEGRADED  (降级中)
0x05 = HEALING   (自愈中)
0xFF = INVALID   (未注册)
```

---

## 六、与商场main()的集成接口

异常处理模块作为**函数库**被main()调用，不独立成进程：

```c
/* main.c 中的异常处理集成点 */

int main(void) {
    /* 1. SHM初始化（含异常处理区域） */
    void* shm_base = shm_core_init(SHM_TOTAL_SIZE);
    void* aer_zone = shm_base + AER_RING_OFFSET;
    void* beacon_zone = shm_base + BEACON_STATUS_OFFSET;

    /* 2. 注册异常处理信号捕获 */
    signal_register_anomaly_handlers();

    /* 3. 主调度循环 */
    while (1) {
        /* 3.1 调度前：扫描灯塔区，识别幽灵与超时 */
        anomaly_scan_beacon(beacon_zone, MAX_ENTITY_COUNT);

        /* 3.2 调度中：检测SHM写边界（通过钩子） */
        /* 注：实际通过shm_core_write_wrapper统一拦截 */

        /* 3.3 调度后：检查Queue压力 */
        size_t q_depth = queue_get_depth();
        int pressure = anomaly_detect_queue_pressure(
            q_depth, 
            QUEUE_WARN_THRESHOLD, 
            QUEUE_CRIT_THRESHOLD
        );
        if (pressure >= 2) {
            anomaly_execute_degrade(1, beacon_zone);
        }

        /* 3.4 周期归档AER */
        if (tick % AER_ARCHIVE_INTERVAL == 0) {
            struct AnomalyEventRecord batch[64];
            int n = anomaly_aer_read_recent(aer_zone, 64, batch);
            anomaly_recovery_archive(batch, n, AER_ARCHIVE_PATH);
        }
    }
}
```

---

## 七、AICoder外置交互流程

```
[商场异常处理模块]
        │
        ├─► 严重/致命异常 ──► 生成AER ──► [归档文件]
        │                                    │
        │                                    ▼
        │                           [外置 AICoder]
        │                           1. 分析异常模式
        │                           2. 更新本地规则缓存
        │                           3. 重新评估熔断实体蓝图
        │                                    │
        │                                    ▼
        └────────────────◄── [新规则/评估报告] ◄──┘
              (天眼恢复后或本地缓存更新)
```

**关键约束**：
- AICoder失联时，`anomaly_execute_degrade()` 自动切换至本地规则缓存模式
- 本地缓存为只读快照，由AICoder上次同步时生成，存储于商场配置区
- 禁止商场在AICoder失联期间准入新蓝图（创世纪铁律）

---

## 八、函数依赖关系

```
anomaly_core_init()
    ├── anomaly_detect_shm_pollution()
    ├── anomaly_detect_heartbeat_loss()
    └── anomaly_detect_queue_pressure()
            │
            ▼
    anomaly_classify_level()
            │
            ├──► Info/Warning ──► anomaly_recovery_self_heal()
            │
            ├──► Critical ──► anomaly_execute_fuse()
            │                 ├── anomaly_execute_quarantine()
            │                 └── anomaly_execute_ghost_purge()
            │
            └──► Fatal ──► anomaly_execute_degrade()
                              └── anomaly_recovery_archive()
```

---

## 九、测试验证点

| 测试项 | 验证内容 | 通过标准 |
|--------|----------|----------|
| T-001 | 模拟生产者SHM越界写 | 触发`C-100-002`，实体熔断，隔离区有数据 |
| T-002 | 模拟消费者心跳丢失 | 触发`C-200-002`，标记GHOST，执行purge |
| T-003 | 模拟Queue堆积 | 触发`I-400-001`后升级至`F-000-002`，启动降级 |
| T-004 | AICoder失联场景 | 进入离线模式，新蓝图拒绝，现有实体正常运行 |
| T-005 | 自愈重试耗尽 | 一般级异常重试3次后自动升级为Critical并熔断 |
| T-006 | AER环形缓冲区覆盖 | 满环后新记录覆盖最旧记录，无内存泄漏 |

---

## 十、审核声明

本蓝图已遵循：
- ✅ **函数级设计**：所有接口为纯C函数，无结构体方法、无类继承
- ✅ **SHM宪法**：异常数据仅存于SHM指定区域，不额外分配堆内存
- ✅ **AICoder外置**：异常归档外送分析，本地仅执行规则缓存
- ✅ **主进程非阻塞**：异常检测嵌入调度循环，无独立守护进程
- ✅ **与创世纪兼容**：适配七阶段演进，第一阶段即可植入检测钩子

**待审核项**：请确认异常状态灯塔的8字节实体描述是否满足未来扩展需求，或需预留至16字节。

---

*蓝图编号：BLUEPRINT-EXCEPTION-001 | 版本：v1.0 | 日期：2026-05-10*
