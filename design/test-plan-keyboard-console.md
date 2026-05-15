User:

键盘生产者 console消费者 测试计划


Kimi:

# 键盘生产者 × Console消费者 联合测试计划


## 一、测试纲领


## 二、测试分层架构

┌─────────────────────────────────────────┐
│  第一层：键盘生产者单元测试               │
│  （验证键盘扫描→数据封装→商场投递）       │
├─────────────────────────────────────────┤
│  第二层：Console消费者单元测试            │
│  （验证商场取货→格式化→屏幕输出）         │
├─────────────────────────────────────────┤
│  第三层：商场调度层测试                   │
│  （验证授权文件解析→函数加载→SHM通道建立） │
├─────────────────────────────────────────┤
│  第四层：端到端集成测试                   │
│  （键入字符 → 屏幕回显 全链路时序验证）    │
├─────────────────────────────────────────┤
│  第五层：压力与边界测试                   │
│  （高频输入、异常字符、缓冲区极限）        │
└─────────────────────────────────────────┘


## 三、测试环境配置

### 3.1 硬件环境

### 3.2 软件环境

### 3.3 测试桩与Mock

/* 键盘Mock层 —— 用于脱离物理键盘的自动化测试 */
void mock_keyboard_inject(const char* sequence);
void mock_keyboard_set_scan_error(int error_code);
void mock_keyboard_clear_buffer(void);

/* Console输出捕获 —— 用于验证消费者行为 */
char* console_capture_buffer(void);
void  console_capture_reset(void);
int   console_capture_line_count(void);


## 四、键盘生产者测试用例

### 4.1 功能测试

### 4.2 数据完整性测试

### 4.3 故障测试


## 五、Console消费者测试用例

### 5.1 功能测试

### 5.2 消费行为测试

### 5.3 故障测试


## 六、商场调度层测试用例

### 6.1 授权加载测试

### 6.2 数据路由测试


## 七、端到端集成测试用例

### 7.1 标准流程测试

### 7.2 异常场景测试


## 八、性能与压力测试

### 8.1 基准测试

### 8.2 压力测试脚本（伪代码）

void test_pressure_30s(void) {
    mock_keyboard_set_rate(1000);          /* 1000Hz输入 */
    sleep(30);
    
    assert(console_capture_line_count() > 0);
    assert(shm_overflow_count() == 0);     /* 零溢出 */
    assert(memory_leak_detected() == 0);   /* 零泄漏 */
    assert(cpu_usage_peak() < 10.0);       /* CPU峰值 < 10% */
}


## 九、测试执行流程

阶段0: 环境自检
    └─→ 运行 test_env_check() → 全部绿灯继续

阶段1: 单元测试并行
    ├─→ 键盘生产者测试桩启动
    └─→ Console消费者测试桩启动
        └─→ 各自输出 .testlog 文件

阶段2: 商场加载测试
    └─→ 运行 MK-001 ~ MK-103
        └─→ 生成 mall_load.report

阶段3: 集成联调
    └─→ 物理键盘 + 真实终端
        └─→ 人工执行 E2E-001 ~ E2E-005
        └─→ 脚本执行 E2E-101 ~ E2E-104

阶段4: 压力燃烧
    └─→ 24小时连续压力测试
        └─→ 每小时生成性能快照

阶段5: 报告汇总
    └─→ 生成最终测试报告
        └─→ 提交AICoder审核归档


## 十、验收标准


## 十一、测试交付物


## 十二、测试铁律（商场宪法补充条款）

测试即信任：未经测试覆盖的函数，商场视为不可信代码，拒绝加载。

Mock即真相：Mock层行为必须精确反映物理设备协议，Mock失真等于测试失效。

失败即停止：任何用例失败，立即冻结该生产者/消费者授权，禁止继续后续阶段。

日志即证据：所有断言结果必须持久化到只读存储，人工可审计，AI可复核。

覆盖即边界：测试覆盖不到的代码路径，视为设计盲区，必须在下一迭代补测。


测试计划版本: v1.0
编制: AICoder（外置审核节点）
审核状态: 待人类架构师终审
关联蓝图: keyboard_producer.blueprint | console_consumer.blueprint
关联授权: keyboard_producer.auth | console_consumer.auth



| 项目 | 内容 |
| --- | --- |
| 测试对象 | 键盘生产者（Keyboard Producer）+ Console消费者（Console Consumer） |
| 测试目的 | 验证生产者-商场-消费者全链路函数级行为正确性、数据完整性、时序稳定性 |
| 测试哲学 | 以测试为设计检验的唯一标准。任何无法被测试验证的设计，视为无效设计。 |
| 禁止项 | 禁止OOP封装、禁止隐藏状态、禁止未暴露的副作用 |


| 组件 | 规格 |
| --- | --- |
| 输入设备 | 标准PS/2或USB HID键盘 |
| 显示终端 | ANSI兼容TTY / 虚拟终端 / SSH会话 |
| 测试主机 | x86_64 Linux / WSL2 / 裸机嵌入式板 |


| 组件 | 版本/要求 |
| --- | --- |
| 商场主程序 | 最新编译版本，启用调试符号 |
| 键盘生产者授权文件 | keyboard_producer.auth（含AICoder审核签名） |
| Console消费者授权文件 | console_consumer.auth（含AICoder审核签名） |
| 测试框架 | 自定义C测试桩（函数级断言宏） |
| 监控工具 | strace（系统调用追踪）、shmdump（共享内存查看器） |


| 用例ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- | --- |
| KP-001 | 单字符采集 | 商场运行，生产者已授权加载 | 注入字符 'A' | 生产者输出数据包：{type: KEY_EVENT, value: 'A', timestamp: valid} | verify_packet_format() |
| KP-002 | 数字键采集 | 同上 | 注入字符串 "123" | 生成3个独立数据包，顺序一致 | verify_packet_sequence() |
| KP-003 | 回车键编码 | 同上 | 注入 '\n' | 数据包value字段为 0x0A，标记 FLAG_LINE_END | verify_special_key() |
| KP-004 | 退格键处理 | 同上 | 注入 "Hel\x08lo" | 生成退格事件包 FLAG_BACKSPACE，商场不丢弃 | verify_backspace_event() |
| KP-005 | 组合键Ctrl+C | 同上 | 注入Ctrl+C扫描码 | 生成控制字符包 value: 0x03, FLAG_CTRL | verify_control_key() |
| KP-006 | 空输入防抖 | 同上 | 持续10ms无按键 | 无数据包生成，CPU占用率 < 1% | verify_idle_behavior() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| KP-101 | 数据包长度恒定 | 注入1000次随机字符 | 所有数据包长度严格为 sizeof(key_packet_t) | verify_packet_size_constancy() |
| KP-102 | 时间戳单调递增 | 注入连续字符流 | 相邻包时间戳差值 > 0 | verify_timestamp_monotonic() |
| KP-103 | SHM写入原子性 | 高并发注入（模拟） | shm_write() 返回成功，无半写包 | verify_shm_atomic_write() |
| KP-104 | 生产者标识符 | 任意输入 | 包头 producer_id 恒为 AUTH_KEYBOARD_PRODUCER_ID | verify_producer_id() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| KP-201 | 键盘断线 | 模拟 ENODEV | 生产者进入 STATE_WAIT_RECONNECT，商场收到 EVENT_PRODUCER_DOWN | verify_disconnect_handling() |
| KP-202 | 缓冲区溢出 | 以1MHz频率注入（超设计极限） | 生产者丢弃溢出帧，计数器 overflow_cnt++，商场日志告警 | verify_overflow_discard() |
| KP-203 | 无效扫描码 | 注入未定义扫描码 0xFF | 生成 EVENT_UNKNOWN_SCANCODE，不崩溃 | verify_unknown_scancode() |


| 用例ID | 用例名称 | 前置条件 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- | --- |
| CC-001 | 单字符回显 | 商场SHM中有单字符包 | 消费者调度执行 | 终端显示 'A'，光标右移1列 | verify_console_output() |
| CC-002 | 字符串拼接 | SHM中有 "Hello" 序列包 | 消费者顺序消费 | 终端显示 "Hello"，无空格插入 | verify_string_concat() |
| CC-003 | 换行处理 | 收到 FLAG_LINE_END 包 | 消费者处理 | 光标移至下一行首列 | verify_newline_behavior() |
| CC-004 | 退格可视化 | 收到 FLAG_BACKSPACE 包 | 消费者处理 | 光标左移1列，覆盖空格，再左移 | verify_backspace_render() |
| CC-005 | 清屏指令 | 收到 FLAG_CLEAR 包 | 消费者处理 | 终端清空，光标归位 (0,0) | verify_clear_screen() |
| CC-006 | 颜色控制 | 收到带 ATTR_COLOR_RED 包 | 消费者处理 | 输出 \033[31m 前缀，后续字符红色 | verify_ansi_color() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| CC-101 | 消费确认机制 | 消费者处理完包 | SHM中该包标记 CONSUMED | verify_consume_ack() |
| CC-102 | 消费速率匹配 | 生产者100包/秒，消费者50包/秒 | 商场缓冲区增长至200包后触发反压 | verify_backpressure() |
| CC-103 | 空轮询抑制 | SHM无数据 | 消费者进入 STATE_IDLE，CPU占用 < 1% | verify_idle_poll() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| CC-201 | 终端断开 | 模拟 SIGPIPE / TTY丢失 | 消费者进入 STATE_HANGUP，商场收到 EVENT_CONSUMER_DOWN | verify_tty_disconnect() |
| CC-202 | 非法控制序列 | 收到 value: 0x1B 无后续转义 | 消费者原样输出 \033，不阻塞 | verify_malformed_escape() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| MK-001 | 双授权并行加载 | 同时加载键盘、Console授权文件 | 两个函数指针注册成功，SHM通道独立 | verify_dual_auth_load() |
| MK-002 | 授权签名验证 | 篡改授权文件中1字节代码 | load_auth() 返回 ERR_SIGNATURE_INVALID，拒绝加载 | verify_auth_integrity() |
| MK-003 | 函数符号解析 | 加载有效授权 | dlsym 等价操作找到 keyboard_produce() 与 console_consume() | verify_symbol_resolution() |


| 用例ID | 用例名称 | 操作步骤 | 预期结果 | 验证函数 |
| --- | --- | --- | --- | --- |
| MK-101 | 生产者→消费者直连 | 键盘输入字符 'X' | 数据包经SHM直达消费者，无中间拷贝 | verify_zero_copy_route() |
| MK-102 | 多生产者隔离 | 尝试加载第二个键盘生产者 | 商场拒绝：ERR_PRODUCER_COLLISION（同类生产者唯一性铁律） | verify_producer_uniqueness() |
| MK-103 | 消费者缺失容错 | 键盘生产者活跃，Console消费者未加载 | 商场缓冲区满后暂停调度生产者，日志告警 | verify_orphan_producer_handling() |


| 用例ID | 场景描述 | 输入动作 | 预期终端输出 | 时序要求 |
| --- | --- | --- | --- | --- |
| E2E-001 | 基础回显 | 键入 Hi + Enter | Hi 换行 | 键入到显示 < 50ms |
| E2E-002 | 行编辑 | 键入 Helo + Backspace + l + Enter | Hello 换行 | 退格响应 < 30ms |
| E2E-003 | 数字输入 | 键入 2024 | 2024 | 无丢包 |
| E2E-004 | 符号混合 | 键入 a+b=c | a+b=c | 符号解析正确 |
| E2E-005 | 长文本输入 | 持续键入1024字符无回车 | 终端正确显示1024字符，光标位置正确 | 无截断、无乱序 |


| 用例ID | 场景描述 | 操作步骤 | 预期系统行为 |
| --- | --- | --- | --- |
| E2E-101 | 热拔插键盘 | 运行中拔除USB键盘，5秒后插回 | 生产者检测到断开→暂停→重连恢复，已输入数据不丢失 |
| E2E-102 | 终端切换 | 运行中切换TTY（Ctrl+Alt+F2） | Console消费者检测到TTY变更→暂停→新TTY恢复 |
| E2E-103 | 商场崩溃恢复 | kill -9 商场主进程后重启 | 授权文件重新加载，生产者和消费者状态重置，无残留SHM |
| E2E-104 | 高频暴力输入 | 以200Hz频率持续输入60秒 | 无丢包、无乱序、无内存泄漏，商场缓冲区峰值 < 80% |


| 指标 | 测试方法 | 合格线 | 优秀线 |
| --- | --- | --- | --- |
| 端到端延迟 | 示波器/逻辑分析仪测量键入到显示 | < 50ms | < 10ms |
| 最大吞吐 | 机械臂/脚本模拟极限键入 | > 100包/秒 | > 500包/秒 |
| CPU占用 | top 采样，空闲态 | < 1% | < 0.1% |
| 内存占用 | pmap 采样，稳定态 | < 1MB | < 512KB |


| 等级 | 条件 | 处置 |
| --- | --- | --- |
| 通过 | 全部用例通过，性能达标，24小时燃烧无异常 | 授权文件生效，允许进入商场生产环境 |
| 有条件通过 | 功能用例100%通过，性能1项未达优秀线 | 授权文件附性能备注，限期优化 |
| 不通过 | 任一功能用例失败，或性能未达合格线 | 退回AICoder重新设计调试，商场拒绝加载 |


| 文件名 | 说明 |
| --- | --- |
| keyboard_test_report.md | 键盘生产者独立测试报告 |
| console_test_report.md | Console消费者独立测试报告 |
| mall_integration_report.md | 商场调度层测试报告 |
| e2e_test_report.md | 端到端集成测试报告 |
| stress_24h.log | 24小时压力测试原始日志 |
| test_coverage.map | 函数覆盖映射（每行代码对应测试用例ID） |

