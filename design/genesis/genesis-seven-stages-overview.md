# 创世纪工程实施七阶段

> **声明**: 本文档内容通过引导AI自动生成，仅供设计参考与学术交流。

> **架构基准**：AICoder外置 | 授权文件含代码 | 人为提交商场 | SHM唯一通道 | 消费者多订阅  
> **进化原则**：初期人工驱动 → 后期AI自治  
> **版本**：Genesis-1.0  
> **日期**：2026-05-10  
> **审核状态**：待审核

---

## 总纲

创世纪工程的目标：从零开始构建一个可自治运行的商场（Mall）系统。商场是生产者与消费者的运行容器，通过SHM（共享内存）作为唯一数据通道，实现进程级隔离与高效数据交换。

**核心铁律**：
1. SHM是商场内所有数据流动的唯一通道
2. AICoder外置，生成含代码的授权文件
3. 授权文件由人类操作者提交给商场
4. 商场拥有加载、调度、监控、进化的完整主权
5. 人类永远保留战略审核权与最高控制权

---

## 阶段一：商场本体创世（The Void）

### 1.1 阶段目标
商场进程从操作系统中诞生，建立自身的物理存在。此阶段商场不依赖任何外部输入，独立完成基础设施初始化。

### 1.2 商场能力增量
- 进程守护框架初始化（信号捕获、崩溃保护）
- SHM主段创建（固定容量，默认 1MB，可配置）
- 商场状态机启动并进入 `STANDBY` 状态
- 基础日志系统启动（输出到文件 + 可选Console）
- 授权文件监听接口就绪（文件系统轮询机制）

### 1.3 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 启动商场进程 | 人类操作者 | 系统部署时 |
| 配置SHM容量 | 人类操作者 | 启动参数 `--shm-size` |
| 指定授权文件收件箱路径 | 人类操作者 | 启动参数 `--inbox-path` |

### 1.4 启动流程
```
操作系统
    ↓
python mall_main.py --shm-size=1048576 --inbox-path=./mall_inbox/
    ↓
[1] 信号捕获注册 (SIGINT, SIGTERM, SIGCHLD)
[2] 日志系统初始化 → ./mall_logs/mall_{timestamp}.log
[3] SHM主段创建 → /dev/shm/mall_core_seg (Linux) 或命名共享内存段
[4] SHM状态机初始化 → 空闲段表、已分配段表
[5] 授权文件监听器启动 → 轮询 inbox 目录（默认 500ms 间隔）
[6] 状态机切换：EMPTY → STANDBY
[7] 进入主循环（空转等待）
```

### 1.5 验收标准（量化）
1. 商场进程 PID 存活，脱离终端后持续运行（`nohup` 或守护进程模式）
2. SHM 主段创建成功，容量与启动参数一致，可通过系统工具验证（`ls -l /dev/shm/`）
3. 状态机日志输出：`[MALL] STATE: STANDBY — awaiting authorization files`
4. 空转 300 秒无内存泄漏，CPU 占用 < 1%（通过 `top`/`htop` 验证）
5. 进程收到 SIGTERM 时，优雅关闭：释放 SHM、刷新日志、退出码 0

### 1.6 阶段出口条件
- 商场进程稳定空转
- 授权文件收件箱目录已创建并监听
- 日志系统正常输出

---

## 阶段二：授权文件接入协议（The Word）

### 2.1 阶段目标
定义并固化"授权文件"的标准格式，建立商场与外部世界（AICoder/人类）的唯一契约接口。此阶段商场学会"识字"，但不执行任何代码。

### 2.2 商场能力增量
- 授权文件解析器（JSON Schema 严格校验）
- 字段完整性检查：`blueprint_id`, `blueprint_type`, `producer_info`, `code_payload`, `shm_contract`, `mall_signature`
- 人为提交通道：指定目录 `./mall_inbox/` 文件系统轮询
- 提交确认回执机制：生成 `ACK-{blueprint_id}-{timestamp}.json` 或 `REJECT-{blueprint_id}-{timestamp}.json`
- 授权文件版本兼容性检查

### 2.3 授权文件标准格式（Schema）
```json
{
  "blueprint_id": "BP-001",
  "blueprint_type": "producer",
  "blueprint_name": "KeyboardProducer",
  "version": "1.0",
  "generated_by": "AICoder-v2.1",
  "generated_at": "2026-05-10T08:00:00Z",
  "human_approved": true,
  "human_approver": "operator_name",

  "producer_info": {
    "name": "KeyboardProducer",
    "type": "producer",
    "language": "Python",
    "entry_function": "keyboard_main"
  },

  "shm_contract": {
    "required_size": 256,
    "vector_name": "keyboard_raw",
    "format": "utf-8-string",
    "max_length": 256,
    "lifecycle": "persistent"
  },

  "code_payload": {
    "language": "Python",
    "source": "def keyboard_main(shm_vector):\n    ...",
    "hash_sha256": "a1b2c3..."
  },

  "mall_signature": {
    "required_api_version": "1.0",
    "compatible_mall_versions": ["1.0", "1.1"]
  }
}
```

### 2.4 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 将授权文件放入收件箱 | 人类操作者 | AICoder生成授权文件后 |
| 审核ACK/REJECT回执 | 人类操作者 | 商场处理完成后 |
| 修正被拒授权文件重新提交 | 人类操作者 | REJECT后 |

### 2.5 处理流程
```
人类操作者将 auth_bp001.json 放入 ./mall_inbox/
    ↓
商场轮询检测到新文件（通过文件mtime或inotify）
    ↓
[1] 文件读取 → 加载到内存
[2] JSON Schema 校验 → 字段完整性检查
[3] code_payload.hash_sha256 校验（可选，确保传输无损）
[4] mall_signature.api_version 兼容性检查
[5] 决策：
    ├─ 合法 → 生成 ACK 文件 → 移入 ./mall_authorized/
    └─ 非法 → 生成 REJECT 文件（附原因） → 移入 ./mall_rejected/
```

### 2.6 验收标准（量化）
1. 合法授权文件放入 inbox 后，商场在 < 500ms 内完成解析并生成 ACK
2. 非法授权文件（缺字段、code_payload为空、JSON语法错误）被 REJECT，错误原因精确到字段级
3. ACK 文件包含：`blueprint_id`, `received_at`, `validated_at`, `status: AUTHORIZED`
4. REJECT 文件包含：`blueprint_id`, `rejected_at`, `reason: "missing_field: shm_contract"`
5. 商场日志记录：`[AUTH] Received BP-001 from human operator, format valid, status: AUTHORIZED`
6. **此阶段不执行 code_payload 中的任何代码**，仅完成格式与契约审查

### 2.7 阶段出口条件
- 至少一张授权文件被成功解析并生成 ACK
- 至少一张非法授权文件被成功拒绝并生成 REJECT（边界测试）
- 授权文件目录结构（inbox/authorized/rejected）正常运转

---

## 阶段三：第一张蓝图落地 — 键盘生产者（The First Producer）

### 3.1 阶段目标
商场接收并执行第一张含代码的授权文件，键盘生产者成为第一个在商场SHM中留下数据的生命体。此阶段标志着商场从"空转"进入"有生命体运行"。

### 3.2 商场能力增量
- 算力进程池初始化（第一个 Worker 进程诞生）
- 代码加载器：将 `code_payload.source` 注入隔离进程
- SHM 矢量分配引擎：根据 `shm_contract` 分配物理地址
- 生产者生命周期管理：启动、心跳检测、状态监控
- 生产者-SHM 绑定注册表

### 3.3 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 提交键盘生产者授权文件 | 人类操作者 | 阶段二完成后 |
| 通过键盘输入测试数据 | 人类操作者 | 生产者加载成功后 |
| 观察商场日志确认生产者存活 | 人类操作者 | 启动后 |

### 3.4 加载与执行流程
```
商场从 ./mall_authorized/ 读取 auth_bp001.json
    ↓
[1] 二次校验（重复加载检查：blueprint_id 是否已存在）
[2] SHM 矢量分配：
    - 查询空闲段表
    - 分配 shm_seg_001:0x0100，长度 256 字节
    - 更新已分配段表
[3] 进程池创建 Worker 进程（fork 或 spawn）
[4] 代码注入：将 code_payload.source 写入临时模块文件
[5] Worker 进程导入模块，定位 entry_function
[6] 启动 entry_function(shm_vector_handle)
[7] 商场主进程启动心跳监控线程（每 1000ms 检测一次）
[8] 状态机：STANDBY → ACTIVE
```

### 3.5 SHM 矢量分配规则
```
SHM 主段结构（1MB）：
├─ 0x0000 ~ 0x00FF: 商场元数据区（状态、注册表、锁）
├─ 0x0100 ~ 0x01FF: [已分配] keyboard_raw (BP-001, 256字节)
├─ 0x0200 ~ 0x02FF: [空闲]
├─ ...
└─ 0xFF00 ~ 0xFFFF: 预留区
```

### 3.6 验收标准（量化）
1. 键盘生产者进程在商场进程池内启动，PID 与商场主进程分离（`ps aux | grep mall` 可见）
2. SHM 矢量 `keyboard_raw@shm_seg_001:0x0100` 被分配并锁定，注册表正确记录
3. 操作者敲击键盘，SHM 对应地址在 < 10ms 内出现数据变化（可通过调试工具读取 SHM 验证）
4. 商场心跳监控：每 1000ms 收到生产者心跳，超时 3000ms 标记 `SUSPECTED_DEAD`
5. 生产者崩溃（如被 kill -9）时，商场在 < 200ms 内检测到并标记 `DEAD`
6. 商场日志：`[PROD] BP-001 (KeyboardProducer) alive, PID=12345, SHM vector mapped at 0x0100`

### 3.7 阶段出口条件
- 键盘生产者进程稳定运行
- SHM 中有可观测的数据写入
- 商场能正确监控生产者生命周期

---

## 阶段四：第二张蓝图落地 — Console 消费者（The First Consumer）

### 4.1 阶段目标
商场加载 Console 消费者，建立"生产者→SHM→消费者"的完整数据链路。消费者天生具备多订阅能力，通过 `input_subscriptions` 声明读取目标。

### 4.2 商场能力增量
- 消费者加载器（与生产者隔离的独立进程空间）
- `input_subscriptions` 解析与 SHM 地址匹配引擎
- 订阅关系注册表：维护 "消费者 → 生产者矢量" 的映射
- 消费者调度器：轮询模式 / 事件驱动模式（初版仅轮询）
- 消费者生命周期管理（与生产者对称）

### 4.3 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 提交 Console 消费者授权文件 | 人类操作者 | 阶段三完成后 |
| 观察 Console 显示输出 | 人类操作者 | 消费者加载后 |
| 验证多订阅声明正确性 | 人类操作者 | 阶段六扩展时 |

### 4.4 消费者蓝图关键字段
```json
{
  "blueprint_id": "BP-002",
  "blueprint_type": "consumer",
  "blueprint_name": "ConsoleDisplay",

  "consumer_info": {
    "name": "ConsoleDisplay",
    "type": "consumer",
    "language": "Python",
    "entry_function": "console_main"
  },

  "input_subscriptions": [
    {
      "vector_name": "keyboard_raw",
      "shm_segment": "shm_seg_001",
      "offset": "0x0100",
      "format": "utf-8-string",
      "max_length": 256,
      "blocking": false,
      "poll_interval_ms": 10
    }
  ],

  "consumer_behavior": {
    "mode": "poll",
    "output_target": "stdout",
    "transform": "none",
    "prefix": "[CHAT] "
  },

  "shm_contract": {
    "required_size": 0,
    "lifecycle": "persistent"
  }
}
```

### 4.5 加载与订阅流程
```
商场从 ./mall_authorized/ 读取 auth_bp002.json
    ↓
[1] 解析 input_subscriptions[0]
[2] 查询已分配段表：keyboard_raw → shm_seg_001:0x0100
[3] 地址匹配验证：offset 0x0100 确实属于 keyboard_raw 矢量范围
[4] 分配消费者进程（独立 Worker）
[5] 注入代码，启动 entry_function(shm_vector_handle_list)
[6] 注册订阅关系：ConsoleDisplay → [keyboard_raw@0x0100]
[7] 启动消费者轮询调度（按 poll_interval_ms）
```

### 4.6 验收标准（量化）
1. Console 消费者进程启动，PID 与键盘生产者、商场主进程三者物理隔离
2. 订阅注册表正确记录：`ConsoleDisplay` 订阅 `keyboard_raw@0x0100`
3. 操作者敲击键盘，Console 在 < 50ms 内显示字符（含 SHM 读取 + 显示延迟）
4. 消费者进程崩溃时，商场在 < 200ms 内检测到，标记 `DEAD`，不影响生产者运行
5. `input_subscriptions` 支持多条目解析（初版仅激活第一条，字段解析不报错）
6. 商场日志：`[CONS] BP-002 (ConsoleDisplay) subscribed to 1 vector(s), polling every 10ms`

### 4.7 阶段出口条件
- Console 消费者进程稳定运行
- 键盘输入能在 Console 中实时显示
- 生产者与消费者崩溃互不影响

---

## 阶段五：端到端闭环验证（The First Breath）

### 5.1 阶段目标
键盘输入 → SHM 写入 → Console 读取 → 屏幕显示，完整生命循环首次跑通。执行系统化测试计划，验证数据链路的正确性、稳定性与性能边界。

### 5.2 商场能力增量
- 端到端链路监控器：追踪一次按键的完整生命周期（时间戳链）
- 测试模式开关：`--test-mode` 自动注入确定性测试数据流
- 性能指标采集器：延迟、吞吐量、SHM 竞争率、CPU/内存占用
- 测试报告生成器：自动输出 Markdown 格式报告

### 5.3 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 执行功能测试（正常输入） | 人类操作者 | 阶段四完成后 |
| 执行边界测试（超长输入、空输入） | 人类操作者 | 功能测试通过后 |
| 执行异常测试（杀进程、断SHM） | 人类操作者 | 边界测试通过后 |
| 审核测试报告 | 人类操作者 | 所有测试完成后 |

### 5.4 测试计划

#### 5.4.1 功能测试
```
输入序列：abcdefghijklmnopqrstuvwxyz0123456789!@#$%
预期输出：Console 逐字符显示，顺序一致，无丢字
通过标准：100 次随机输入，正确率 100%
```

#### 5.4.2 边界测试
```
测试项1：空击（无输入）
预期：Console 无输出，无异常

测试项2：超长字符串（300字节，超过 max_length=256）
预期：截断至 256 字节，剩余 44 字节丢弃，商场日志告警

测试项3：快速连击（每秒 50 次按键）
预期：无丢字，SHM 无覆盖冲突
```

#### 5.4.3 异常测试
```
测试项1：强制杀死键盘生产者（kill -9 PID）
预期：商场 200ms 内标记 DEAD，Console 进入等待状态（轮询空转），不崩溃

测试项2：强制杀死 Console 消费者
预期：商场 200ms 内标记 DEAD，键盘生产者继续写入 SHM，数据不丢失

测试项3：重启键盘生产者（重新提交授权文件）
预期：新进程接管原 SHM 地址，Console 自动恢复订阅，无需重新加载消费者
```

### 5.5 性能指标
| 指标 | 目标值 | 测量方法 |
|------|--------|----------|
| 端到端延迟（按键→显示） | < 50ms（P95） | 时间戳差值 |
| SHM 写入延迟 | < 5ms | 生产者内部计时 |
| SHM 读取延迟 | < 5ms | 消费者内部计时 |
| 进程崩溃检测时间 | < 200ms | 心跳超时计时 |
| 300秒稳定运行内存泄漏 | 0 | 前后 RSS 对比 |

### 5.6 验收标准（量化）
1. 功能测试通过率 100%
2. 边界测试全部通过，截断机制生效，无缓冲区溢出
3. 异常测试全部通过，单点故障不影响整体系统
4. 性能指标全部达标
5. 测试报告自动生成：`./mall_reports/genesis_test_report_{timestamp}.md`

### 5.7 阶段出口条件
- 测试报告通过人类审核
- 端到端链路被证明稳定可靠
- 系统具备单点故障容忍能力

---

## 阶段六：多实例与调度进化（The Multitude）

### 6.1 阶段目标
商场从"单生产者-单消费者"进化为"多生产者并发、消费者多订阅"的复杂生态。验证商场的调度能力、SHM 动态分配能力与扩展性。

### 6.2 商场能力增量
- 多生产者调度器：时间片轮转 + 事件驱动混合调度
- SHM 矢量动态分配：新生产者接入时自动寻址，不冲突
- 消费者多路复用：同时订阅多个生产者，按优先级/轮询读取
- 算力进程池动态扩容：根据负载自动增减 Worker 槽位
- 生产者类型注册表：支持异构生产者并存（键盘、网络、文件、传感器等）

### 6.3 人为操作点
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 提交第三张授权文件（新生产者类型） | 人类操作者 | 阶段五完成后 |
| 提交第四张授权文件（扩展消费者订阅） | 人类操作者 | 新生产者加载后 |
| 观察多路并发输出 | 人类操作者 | 多实例运行时 |
| 审核调度公平性 | 人类操作者 | 压力测试后 |

### 6.4 扩展场景

#### 场景A：网络聊天生产者加入
```
新生产者：NetworkChatProducer (BP-003)
输出矢量：network_chat@shm_seg_001:0x0200 (256字节)

Console 消费者升级订阅：
  input_subscriptions[0]: keyboard_raw@0x0100
  input_subscriptions[1]: network_chat@0x0200

显示规则：
  [KEYBOARD] 用户键盘输入
  [NETWORK] 来自网络的消息
```

#### 场景B：文件日志生产者加入
```
新生产者：FileLoggerProducer (BP-004)
输出矢量：file_log@shm_seg_001:0x0300

消费者：ConsoleDisplay 可选择性订阅（人类通过新授权文件控制）
```

### 6.5 SHM 动态分配规则
```
新生产者接入时：
  [1] 查询已分配段表，找到最高地址
  [2] 新分配地址 = 最高地址 + 对齐偏移（默认 256 字节对齐）
  [3] 检查是否超出 SHM 总容量
  [4] 更新空闲段表、已分配段表
  [5] 返回矢量句柄给生产者进程
```

### 6.6 调度策略
```
生产者调度（初版）：
  - 每个生产者独立进程，操作系统调度
  - 商场不干预 CPU 分配，仅监控心跳

消费者调度（初版）：
  - 轮询模式：按 poll_interval_ms 依次读取各订阅矢量
  - 非阻塞：某矢量无数据时立即切换到下一个
  - 优先级：input_subscriptions 数组顺序即优先级顺序
```

### 6.7 验收标准（量化）
1. 3 个及以上生产者同时运行，SHM 矢量无地址冲突，注册表正确
2. Console 消费者同时订阅 2 个及以上生产者，显示输出包含来源标识
3. 各生产者获得相近的 CPU 时间片（差异 < 20%，通过 `pidstat` 验证）
4. 商场进程池 Worker 数随负载动态调整：空闲时收缩至最小（2个），繁忙时扩张
5. SHM 读写竞争率 < 5%，无死锁、无数据覆盖
6. 新增生产者时，现有生产者与消费者无中断（热插拔能力）

### 6.8 阶段出口条件
- 多生产者并发稳定运行
- 消费者多订阅能力验证通过
- 商场调度器被证明可扩展

---

## 阶段七：商场自治铁律生效（The Sabbath）

### 7.1 阶段目标
商场进入自治状态：自我监控、自我修复、自我进化。人类操作者从"日常操作"退居"战略审核"。此阶段标志着创世纪完成，商场成为独立运行的智能体容器。

### 7.2 商场能力增量
- 自治状态机：`AUTONOMOUS`
- 异常自愈引擎：生产者崩溃 → 自动重启（最多 3 次指数退避）
- SHM 宪法强制执行：非法内存访问的进程被立即隔离（SIGKILL + 审计日志）
- 授权文件热更新：新旧版本平滑切换（双缓冲机制）
- 商场健康仪表盘：导出核心指标到 `./mall_status.json`
- 负载动态评估：根据生产者数量、SHM 使用率、CPU 负载自动调整进程池
- 告警系统：异常事件通过日志 + 状态文件通知人类

### 7.3 人为操作点（战略级）
| 操作 | 执行者 | 触发条件 |
|------|--------|----------|
| 提交全新类型蓝图（战略创新） | 人类操作者 | 需要引入从未有过的生产者类型 |
| 审核商场异常审计日志 | 人类操作者 | 告警系统触发 |
| 手动触发商场全局重启 | 人类操作者 | 灾难恢复 |
| 修订 SHM 宪法 | 人类操作者 | 架构升级 |
| 接入/断开 AICoder 外置服务 | 人类操作者 | AICoder 版本升级或故障 |

### 7.4 自治规则

#### 自愈规则
```
生产者进程标记 DEAD：
  [1] 商场尝试重启（第1次，延迟 0ms）
  [2] 若再次 DEAD，延迟 1000ms 后第2次重启
  [3] 若再次 DEAD，延迟 3000ms 后第3次重启
  [4] 若第3次仍 DEAD，标记 PERMANENT_DEAD，告警人类
  [5] 消费者进程同理
```

#### SHM 宪法执行
```
任何进程尝试访问未分配的 SHM 地址：
  [1] 商场监控线程捕获异常信号（SIGSEGV 或自定义检查）
  [2] 立即发送 SIGKILL 给违规进程
  [3] 记录审计日志：`[SECURITY] PID=12345 attempted illegal SHM access at 0x0FFF, terminated`
  [4] 将该 blueprint_id 加入黑名单，禁止再次加载（除非人类手动解除）
```

#### 热更新规则
```
新授权文件替换旧版本（同 blueprint_id）：
  [1] 商场加载新版本到影子进程池
  [2] 旧版本继续运行，新版本预热
  [3] 切换 SHM 矢量绑定（原子操作）
  [4] 杀死旧进程，新版本接管
  [5] 服务中断时间 = 0
```

### 7.5 验收标准（量化）
1. 商场 7×24 小时运行，可用性 > 99.9%（允许计划内维护窗口）
2. 生产者随机崩溃 100 次，商场自愈成功率 100%，平均恢复时间 < 500ms
3. SHM 宪法违规（越界读写）被 100% 捕获并隔离，无逃逸案例
4. 热更新：新版本替换旧版本，服务中断时间 = 0，数据不丢失
5. 负载动态评估：生产者从 1 个增至 10 个，进程池从 2 个 Worker 自动扩展至 12 个
6. 商场日志最后一条：`[MALL] STATE: AUTONOMOUS — Genesis complete. Awaiting strategic input.`

### 7.6 阶段出口条件
- 商场连续 72 小时无人类干预稳定运行
- 自愈、宪法执行、热更新三大铁律验证通过
- 人类操作者确认：日常无需操作，仅保留战略审核权

---

## 附录A：七阶段与商场状态机对照表

| 阶段 | 商场状态 | 状态说明 | main() 核心行为 |
|------|----------|----------|-----------------|
| 一 | `EMPTY → STANDBY` | 从虚无到待命 | 初始化 SHM，空转等待 |
| 二 | `STANDBY` | 待命，学习识字 | 监听 inbox，解析授权文件 |
| 三 | `STANDBY → ACTIVE` | 激活，第一生命体 | 加载第一个生产者，进程池启动 |
| 四 | `ACTIVE` | 运行中，建立链路 | 加载消费者，建立订阅关系 |
| 五 | `ACTIVE` | 验证生命循环 | 端到端测试，性能采集 |
| 六 | `ACTIVE → SCALING` | 扩展，族群繁衍 | 多实例调度，动态扩容 |
| 七 | `SCALING → AUTONOMOUS` | 自治，安息日 | 自愈监控，热更新，负载评估 |

---

## 附录B：人为操作在七阶段中的分布

```
阶段一: 人启动商场进程
阶段二: 人提交授权文件 → 商场解析（人审核ACK/REJECT）
阶段三: 人提交生产者授权文件 → 人测试键盘输入
阶段四: 人提交消费者授权文件 → 人观察Console输出
阶段五: 人执行完整测试计划 → 人审核测试报告
阶段六: 人提交新生产者/消费者 → 人观察多路并发
阶段七: 人退居战略审核 → 仅介入异常、创新、灾难恢复
```

**铁律**：阶段七之前，人为操作是**必要输入**；阶段七之后，人为操作是**战略干预**。

---

## 附录C：阶段间衔接关系

```
阶段一 (STANDBY)
    ↓ 人类启动完成
阶段二 (STANDBY)
    ↓ 至少一张授权文件被ACK
阶段三 (ACTIVE)
    ↓ 生产者进程稳定运行 + SHM有数据
阶段四 (ACTIVE)
    ↓ 消费者加载成功 + 端到端显示正常
阶段五 (ACTIVE)
    ↓ 测试报告通过审核
阶段六 (SCALING)
    ↓ 多实例并发验证通过
阶段七 (AUTONOMOUS)
    ↓ 72小时无干预稳定运行
创世纪完成
```

---

*本文档基于 AICoder 外置架构、授权文件人为提交、SHM 唯一通道、初期人工后期AI自治四大基准编写。*
