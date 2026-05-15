# 沙箱商场方案与蓝图

> **文档性质**：微型商场契约宪法技术附录 —— 沙箱系统架构设计蓝图
> **适用范围**：商场UI系统调试期的沙箱环境构建、运行与销毁
> **前置法典**：
> - 商场UI部署域安全规范 v1.0（已生效）
> - 商场UI部署域安全规范补充修正案 v1.1（已生效）
> - 商场高速数据安全传输规范 v1.2 马赛克加密法（已生效）
> - 商场安全宪法生命周期管理修正案 v1.3 + 非密系统铁律（已生效）
> **版本**：v1.0 —— 与商场模式3.0、蓝图基因系统、三权分立体系、SHM内存宪法兼容

---

## 一、关键词定义

| 术语 | 定义 |
|------|------|
| **沙箱商场 (Sandbox Mall)** | 商场内核的完全隔离克隆，运行于独立虚拟化环境，仅含合成数据，属于**非密系统**，全透明 |
| **沙箱SHM (Sandbox SHM)** | 沙箱商场专用的共享内存矢量区，与生产SHM物理隔离，数据均为合成生成 |
| **合成数据引擎 (Synthetic Engine)** | 负责生成模拟频谱、波形、任务队列等调试数据的函数集合 |
| **调试Nginx (Debug Nginx)** | 沙箱商场的UI部署服务器，可读写挂载，明文HTTP，无seccomp |
| **沙箱Worker (Sandbox Worker)** | 沙箱商场专用的Worker探针，仅连接沙箱内核，禁止出站 |
| **透明铁律 (Transparency Doctrine)** | 沙箱商场作为非密系统的根本原则：全部状态明文可见，无任何保密措施 |
| **克隆协议 (Clone Protocol)** | 从生产商场内核生成沙箱商场克隆的标准化流程 |
| **沙箱销毁 (Sandbox Destruction)** | 调试完成后彻底抹除沙箱环境，确保无数据残留 |

---

## 二、哲学定位：沙箱是商场的「镜像分身」

> **沙箱商场不是商场的简化版，而是商场的「镜像分身」——它拥有与生产商场完全相同的代码逻辑、相同的SHM矢量结构、相同的任务调度机制，但它运行在平行宇宙中，只与合成数据互动。**

沙箱商场的存在意义：

1. **验证逻辑正确性**：在不影响生产系统的前提下，验证UI与商场内核的交互逻辑。
2. **暴露安全隐患**：全透明环境下，安全漏洞无处藏身——若沙箱中可见的漏洞在归档期被加密掩盖，则构成对消费者的欺诈。
3. **加速迭代**：开发者无需担心加密摩擦、签名验证、权限限制，专注于功能实现。
4. **教育训练**：新加入的AICoder或人类工程师，可在沙箱中完整观察商场运行机制。

---

## 三、沙箱商场架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              沙箱商场 (Sandbox Mall)                          │
│                              ┌─────────────────┐                            │
│                              │   沙箱管理控制器   │                            │
│                              │  (Sandbox Ctrl)   │                            │
│                              │  ├─ 克隆协议执行   │                            │
│                              │  ├─ 生命周期管理   │                            │
│                              │  ├─ 合成数据注入   │                            │
│                              │  └─ 沙箱销毁调度   │                            │
│                              └─────────────────┘                            │
│                                        │                                    │
│                    ┌─────────────────────┼─────────────────────┐              │
│                    │                     │                     │              │
│                    ▼                     ▼                     ▼              │
│           ┌───────────────┐    ┌───────────────┐    ┌───────────────┐        │
│           │  沙箱内核克隆   │    │  沙箱SHM矢量区 │    │  沙箱Worker组  │        │
│           │ (Mall Core    │    │ (Synthetic    │    │ (Sandbox     │        │
│           │  Clone)       │    │  Vectors)     │    │  Workers)    │        │
│           │               │    │               │    │               │        │
│           │ ├─ 任务调度    │◄──►│ ├─ 合成频谱   │    │ ├─ 探针Worker  │        │
│           │ ├─ 算力评估    │    │ ├─ 合成波形   │    │ ├─ 日志Worker  │        │
│           │ ├─ 蓝图解析    │    │ ├─ 虚拟任务   │    │ └─ 监控Worker  │        │
│           │ └─ 安全策略    │    │ └─ 调试标记   │    │               │        │
│           │    (全透明)    │    │    (明文)     │    │    (仅内联)    │        │
│           └───────────────┘    └───────────────┘    └───────────────┘        │
│                    │                     │                     │              │
│                    └─────────────────────┼─────────────────────┘              │
│                                            │                                    │
│                                            ▼                                    │
│                              ┌─────────────────┐                            │
│                              │   调试Nginx服务器  │                            │
│                              │  (Debug Nginx)    │                            │
│                              │  ├─ 明文HTTP      │                            │
│                              │  ├─ 可读写挂载    │                            │
│                              │  ├─ 热重载支持    │                            │
│                              │  └─ 调试终端开放  │                            │
│                              └─────────────────┘                            │
│                                            │                                    │
│                                            ▼                                    │
│                              ┌─────────────────┐                            │
│                              │   开发者浏览器    │                            │
│                              │  (Dev Browser)    │                            │
│                              │  ├─ DevTools全开  │                            │
│                              │  ├─ 抓包代理      │                            │
│                              │  └─ 源码可见      │                            │
│                              └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘

                              🔒 物理隔离边界 🔒
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              生产商场 (Production Mall)                       │
│                              ┌─────────────────┐                            │
│                              │   生产管理控制器   │                            │
│                              │  (Production Ctrl)│                            │
│                              └─────────────────┘                            │
│                                        │                                    │
│                    ┌─────────────────────┼─────────────────────┐              │
│                    │                     │                     │              │
│                    ▼                     ▼                     ▼              │
│           ┌───────────────┐    ┌───────────────┐    ┌───────────────┐        │
│           │  生产内核      │    │  生产SHM矢量区 │    │  生产Worker组  │        │
│           │ (Mall Core)   │    │ (Production   │    │ (Production  │        │
│           │               │    │  Vectors)     │    │  Workers)    │        │
│           │ ├─ 任务调度    │◄──►│ ├─ 真实频谱   │    │ ├─ 探针Worker  │        │
│           │ ├─ 算力评估    │    │ ├─ 真实波形   │    │ ├─ 日志Worker  │        │
│           │ ├─ 蓝图解析    │    │ ├─ 真实任务   │    │ └─ 监控Worker  │        │
│           │ └─ 安全策略    │    │ └─ 消费者数据  │    │               │        │
│           │    (全封闭)    │    │    (加密)      │    │    (严格隔离)  │        │
│           └───────────────┘    └───────────────┘    └───────────────┘        │
│                    │                     │                     │              │
│                    └─────────────────────┼─────────────────────┘              │
│                                            │                                    │
│                                            ▼                                    │
│                              ┌─────────────────┐                            │
│                              │   生产Nginx服务器  │                            │
│                              │  (Prod Nginx)     │                            │
│                              │  ├─ HTTPS/TLS 1.3 │                            │
│                              │  ├─ 只读挂载      │                            │
│                              │  ├─ seccomp沙箱   │                            │
│                              │  └─ 签名验证      │                            │
│                              └─────────────────┘                            │
│                                            │                                    │
│                                            ▼                                    │
│                              ┌─────────────────┐                            │
│                              │   消费者浏览器    │                            │
│                              │  (Consumer      │                            │
│                              │   Browser)       │                            │
│                              │  ├─ 端到端解密   │                            │
│                              │  ├─ CAT确认      │                            │
│                              │  └─ 马赛克重组   │                            │
│                              └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、物理隔离边界

### 4.1 隔离原则

> **沙箱商场与生产商场之间，必须存在不可逾越的物理隔离边界。这不是网络策略的隔离，而是物理层面的隔离——不同的宿主机、不同的存储介质、不同的网络段。**

### 4.2 隔离层级

| 隔离层级 | 沙箱商场 | 生产商场 | 隔离手段 |
|---------|---------|---------|---------|
| **宿主机** | 独立物理机/独立虚拟机宿主机 | 生产集群 | 不同物理硬件或不同虚拟化集群 |
| **存储** | 独立磁盘/独立存储卷 | 生产存储阵列 | 不同LUN/不同磁盘分区 |
| **网络** | 调试专用VLAN（如 VLAN 100） | 生产VLAN（如 VLAN 10） | 三层交换机ACL，禁止跨VLAN路由 |
| **SHM** | 独立内存段（不同/dev/shm命名空间） | 生产/dev/shm | 命名空间隔离 + cgroup内存限制 |
| **仓库** | 调试仓库（debug-repo.mall.local） | 生产仓库（repo.mall.local） | DNS隔离 + 防火墙阻断 |
| **Worker** | 沙箱Worker仅绑定沙箱内核PID | 生产Worker绑定生产内核PID | PID命名空间隔离 |

### 4.3 隔离验证测试

沙箱商场启动前，必须执行以下隔离验证：

```c
// 隔离验证函数（C语言，函数级设计，禁止OOP）
int sandbox_isolation_verify(void) {
    // 1. 验证宿主机隔离
    if (is_same_physical_host(PROD_HOST_ID)) {
        log_fatal("SANDBOX_VIOLATION: 沙箱与生产共享宿主机");
        return -1;
    }

    // 2. 验证存储隔离
    if (shm_is_same_device(PROD_SHM_DEVICE)) {
        log_fatal("SANDBOX_VIOLATION: 沙箱与生产共享存储设备");
        return -1;
    }

    // 3. 验证网络隔离
    if (network_can_reach(PROD_VLAN_SUBNET)) {
        log_fatal("SANDBOX_VIOLATION: 沙箱网络可访问生产子网");
        return -1;
    }

    // 4. 验证仓库隔离
    if (repo_can_access(PROD_REPO_URL)) {
        log_fatal("SANDBOX_VIOLATION: 沙箱可访问生产仓库");
        return -1;
    }

    // 5. 验证SHM隔离
    if (shm_has_prod_vectors()) {
        log_fatal("SANDBOX_VIOLATION: 沙箱SHM含生产矢量标记");
        return -1;
    }

    log_info("SANDBOX_ISOLATION_OK: 全部隔离验证通过");
    return 0;
}
```

---

## 五、合成数据引擎设计

### 5.1 合成数据原则

> **合成数据不是随机噪声，而是具有真实统计特征的模拟数据。它的目的是让调试者相信自己在与真实系统互动，同时确保即使数据泄露，也不含任何真实消费者信息。**

### 5.2 合成数据类型

| 数据类型 | 合成方法 | 真实感保障 | 可识别标记 |
|---------|---------|-----------|-----------|
| **合成频谱** | 基于真实频谱的统计模型（高斯混合）生成 | 频谱形态、峰值分布、噪声基底与真实数据一致 | 全部频率点标记 `SYNTH_` 前缀 |
| **合成波形** | 基于真实波形的参数化模型（正弦+噪声+脉冲）生成 | 时域特征、脉冲参数、上升沿/下降沿与真实一致 | 时间戳含 `DEBUG_` 前缀 |
| **虚拟任务** | 从预定义任务模板库随机组合生成 | 任务结构、参数类型、依赖关系与真实任务一致 | 任务ID以 `TASK-DEBUG-` 开头 |
| **虚拟消费者** | 从虚拟身份池随机分配 | 消费者档案结构完整，但姓名/ID为虚构 | 消费者ID以 `CONSUMER-TEST-` 开头 |
| **合成日志** | 基于真实日志模式的模板生成 | 日志格式、字段、级别与真实一致 | 日志源标记 `SANDBOX` |

### 5.3 合成数据引擎函数设计

```c
// 合成频谱生成函数
// 输入：频谱模板ID、采样点数、中心频率、带宽
// 输出：合成频谱矢量（写入沙箱SHM）
int synth_spectrum_generate(
    const char* template_id,
    size_t sample_count,
    double center_freq_hz,
    double bandwidth_hz,
    shm_vector_t* output_vector
) {
    // 1. 加载频谱统计模型
    spectrum_model_t* model = spectrum_model_load(template_id);
    if (!model) {
        log_error("SYNTH_FAIL: 模板 %s 不存在", template_id);
        return -1;
    }

    // 2. 生成基础频谱形态
    double* spectrum = malloc(sample_count * sizeof(double));
    spectrum_model_render(model, center_freq_hz, bandwidth_hz, spectrum, sample_count);

    // 3. 添加真实感噪声
    noise_profile_t* noise = noise_profile_create(model->noise_params);
    noise_profile_apply(noise, spectrum, sample_count);
    noise_profile_destroy(noise);

    // 4. 添加调试标记（不可移除）
    for (size_t i = 0; i < sample_count; i++) {
        spectrum[i] = synth_mark(spectrum[i], SYNTH_MARK_SPECTRUM);
    }

    // 5. 写入沙箱SHM矢量区
    shm_vector_write(output_vector, spectrum, sample_count * sizeof(double));

    // 6. 记录合成日志
    log_info("SYNTH_OK: 频谱 %s 已生成 %zu 点", template_id, sample_count);

    free(spectrum);
    spectrum_model_destroy(model);
    return 0;
}

// 合成波形生成函数
int synth_waveform_generate(
    const char* template_id,
    size_t sample_count,
    double sample_rate_hz,
    shm_vector_t* output_vector
) {
    waveform_model_t* model = waveform_model_load(template_id);
    if (!model) return -1;

    double* waveform = malloc(sample_count * sizeof(double));
    waveform_model_render(model, sample_rate_hz, waveform, sample_count);

    // 添加脉冲特征
    pulse_params_t pulse = pulse_params_from_model(model);
    pulse_inject(waveform, sample_count, &pulse);

    // 添加调试标记
    for (size_t i = 0; i < sample_count; i++) {
        waveform[i] = synth_mark(waveform[i], SYNTH_MARK_WAVEFORM);
    }

    shm_vector_write(output_vector, waveform, sample_count * sizeof(double));
    log_info("SYNTH_OK: 波形 %s 已生成 %zu 点", template_id, sample_count);

    free(waveform);
    waveform_model_destroy(model);
    return 0;
}
```

### 5.4 合成数据不可污染铁律

合成数据引擎必须遵守：

- ❌ **禁止读取生产SHM**：合成引擎的数据源只能是预加载的统计模型，不能实时读取生产数据。
- ❌ **禁止反向标注**：不得将 `SYNTH_` 标记移除或伪装为真实数据。
- ✅ **强制标记注入**：每一字节合成数据必须包含不可移除的调试标记。
- ✅ **模型隔离存储**：统计模型文件存储于沙箱专用路径，与生产模型物理隔离。

---

## 六、调试Nginx服务器设计

### 6.1 调试Nginx与生产Nginx的对比

| 属性 | 调试Nginx（沙箱） | 生产Nginx（归档） |
|------|-----------------|-----------------|
| **部署域性质** | 非密系统 | 密系统 |
| **传输协议** | HTTP（明文） | HTTPS（TLS 1.3） |
| **代码挂载** | 可读写（RW） | 只读（RO） |
| **seccomp** | 禁用 | 强制启用 |
| **签名验证** | 禁用 | 强制启用 |
| **CAT登录** | 禁用（免登录） | 强制启用 |
| **马赛克加密** | 禁用 | 强制启用（归档期） |
| **终端访问** | 开放SSH/Shell | 禁止直接登录 |
| **热重载** | 支持（修改即生效） | 禁止（需重新归档） |
| **DevTools** | 完全开放 | 消费者浏览器端开放 |
| **日志** | 明文全量 | 加密审计链 |

### 6.2 调试Nginx配置

```nginx
# /etc/nginx/nginx.conf (Debug Mode)
user dev-user;
worker_processes auto;
pid /var/run/nginx.pid;

daemon off;
error_log /var/log/nginx/debug.log debug;  # debug级别，全量记录

events {
    worker_connections 1024;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 明文HTTP（调试专用端口8080）
    server {
        listen 8080;
        server_name sandbox-ui.mall.local;

        root /var/ui-debug/;  # 可读写挂载点
        index index.html;

        # 允许跨域调试请求
        add_header Access-Control-Allow-Origin "*";
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "*";

        # 禁用缓存（确保修改即时生效）
        add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";

        # 调试API代理至沙箱商场内核
        location /api/ {
            proxy_pass http://sandbox-core:9000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Sandbox-Mode "true";  # 标记沙箱请求
        }

        # 调试信息端点（仅沙箱可用）
        location /debug/info {
            allow 192.168.100.0/24;  # 调试网段
            deny all;
            proxy_pass http://sandbox-core:9000/debug/info;
        }
    }
}
```

### 6.3 沙箱请求标记

所有从调试Nginx发往沙箱商场的请求，必须携带 `X-Sandbox-Mode: true` 头部：

```c
// 沙箱商场内核请求处理函数
int mall_request_handle(http_request_t* req) {
    const char* sandbox_mode = http_header_get(req, "X-Sandbox-Mode");

    if (sandbox_mode && strcmp(sandbox_mode, "true") == 0) {
        // 沙箱模式：允许明文响应，禁用加密，返回调试信息
        req->mode = REQ_MODE_SANDBOX;
        req->encryption = ENC_NONE;
        req->debug_info = DEBUG_FULL;
    } else {
        // 非沙箱请求进入生产模式处理（需CAT、签名、加密）
        req->mode = REQ_MODE_PRODUCTION;
        return mall_production_handle(req);
    }

    return mall_sandbox_handle(req);
}
```

---

## 七、沙箱Worker组设计

### 7.1 沙箱Worker与生产Worker的对比

| 属性 | 沙箱Worker | 生产Worker |
|------|-----------|-----------|
| **连接目标** | 仅沙箱内核PID | 仅生产内核PID |
| **出站权限** | **绝对禁止** | 经专门Worker审批后有限出站 |
| **数据访问** | 仅沙箱SHM | 生产SHM（经矢量关卡） |
| **日志上报** | 明文日志至调试控制台 | 加密审计日志至商场内核 |
| **蓝图执行** | 调试蓝图（无签名验证） | 归档蓝图（强制签名验证） |
| **生命周期** | 随沙箱商场同生同灭 | 长期运行，热更新 |

### 7.2 沙箱Worker函数设计

```c
// 沙箱Worker主循环
void sandbox_worker_main_loop(worker_ctx_t* ctx) {
    // 1. 验证自身绑定的是沙箱内核
    if (!is_sandbox_kernel(ctx->kernel_pid)) {
        log_fatal("WORKER_VIOLATION: Worker绑定非沙箱内核，强制退出");
        worker_exit(ctx, EXIT_CODE_VIOLATION);
        return;
    }

    // 2. 验证无出站能力
    if (network_has_outbound_capability()) {
        log_fatal("WORKER_VIOLATION: 沙箱Worker拥有出站能力，强制退出");
        worker_exit(ctx, EXIT_CODE_VIOLATION);
        return;
    }

    // 3. 主循环
    while (ctx->running) {
        task_t* task = shm_task_queue_pop(ctx->sandbox_shm_queue);
        if (!task) {
            usleep(1000);  // 1ms空闲轮询
            continue;
        }

        // 4. 验证任务为合成任务
        if (!task_is_synthetic(task)) {
            log_error("WORKER_REJECT: 非合成任务 %s", task->id);
            task_mark_rejected(task, REASON_NON_SYNTHETIC);
            continue;
        }

        // 5. 执行任务（调试模式：允许异常抛出，全量日志）
        task_result_t* result = task_execute_debug(task, ctx);

        // 6. 结果写入沙箱SHM（明文）
        shm_result_write(ctx->sandbox_shm_results, result);

        // 7. 明文调试日志
        log_debug("WORKER_DONE: 任务 %s 完成，结果哈希 %s",
                  task->id, result->hash);

        task_result_destroy(result);
    }
}
```

---

## 八、沙箱商场生命周期

### 8.1 生命周期状态机

```
┌──────────┐    克隆协议    ┌──────────┐    合成数据注入    ┌──────────┐
│   空闲    │ ─────────────► │  初始化中  │ ────────────────► │  运行中   │
│  (Idle)   │                │ (Init)    │                   │ (Running) │
└──────────┘                └──────────┘                   └──────────┘
                                                               │
                                                               │ 调试完成
                                                               ▼
                                                          ┌──────────┐
                                                          │  待销毁   │
                                                          │ (Pending  │
                                                          │  Destroy) │
                                                          └──────────┘
                                                               │
                                                               │ 销毁协议
                                                               ▼
                                                          ┌──────────┐
                                                          │   已销毁  │
                                                          │ (Destroyed│
                                                          └──────────┘
```

### 8.2 克隆协议

```c
// 沙箱商场克隆函数
// 从生产商场内核生成完全隔离的沙箱克隆
int sandbox_clone_execute(const char* prod_core_image, sandbox_t** out_sandbox) {
    // 1. 分配沙箱资源
    sandbox_t* sb = sandbox_alloc();
    if (!sb) return -1;

    // 2. 加载生产内核代码镜像（只读，不执行）
    sb->core_code = core_image_load(prod_core_image);
    if (!sb->core_code) {
        log_error("CLONE_FAIL: 无法加载生产内核镜像 %s", prod_core_image);
        sandbox_free(sb);
        return -1;
    }

    // 3. 创建独立虚拟化环境
    sb->vm = vm_create(SANDBOX_VM_CONFIG);
    if (!sb->vm) {
        log_error("CLONE_FAIL: 虚拟机创建失败");
        sandbox_free(sb);
        return -1;
    }

    // 4. 部署沙箱内核代码（代码级克隆，非数据克隆）
    vm_deploy_code(sb->vm, sb->core_code);

    // 5. 创建独立SHM矢量区
    sb->shm = shm_create_sandbox_region(SANDBOX_SHM_SIZE);
    if (!sb->shm) {
        log_error("CLONE_FAIL: 沙箱SHM创建失败");
        sandbox_destroy(sb);
        return -1;
    }

    // 6. 执行隔离验证
    if (sandbox_isolation_verify() != 0) {
        log_error("CLONE_FAIL: 隔离验证未通过");
        sandbox_destroy(sb);
        return -1;
    }

    // 7. 启动合成数据引擎
    sb->synth_engine = synth_engine_create(sb->shm);
    synth_engine_warmup(sb->synth_engine);

    // 8. 启动沙箱Worker组
    sb->workers = worker_group_create_sandbox(sb->vm, sb->shm, MAX_SANDBOX_WORKERS);
    worker_group_start(sb->workers);

    // 9. 启动调试Nginx
    sb->nginx = debug_nginx_create(sb);
    debug_nginx_start(sb->nginx);

    // 10. 标记状态
    sb->state = SANDBOX_STATE_RUNNING;

    log_info("CLONE_OK: 沙箱商场已启动，ID=%s", sb->id);
    *out_sandbox = sb;
    return 0;
}
```

### 8.3 销毁协议

```c
// 沙箱商场销毁函数
// 彻底抹除沙箱环境，确保无数据残留
int sandbox_destroy_execute(sandbox_t* sb) {
    if (!sb) return -1;

    log_info("DESTROY_START: 开始销毁沙箱 %s", sb->id);

    // 1. 停止调试Nginx
    if (sb->nginx) {
        debug_nginx_stop(sb->nginx);
        debug_nginx_destroy(sb->nginx);
        sb->nginx = NULL;
    }

    // 2. 停止沙箱Worker组
    if (sb->workers) {
        worker_group_stop(sb->workers);
        worker_group_destroy(sb->workers);
        sb->workers = NULL;
    }

    // 3. 销毁合成数据引擎
    if (sb->synth_engine) {
        synth_engine_destroy(sb->synth_engine);
        sb->synth_engine = NULL;
    }

    // 4. 安全擦除SHM矢量区（覆写后释放）
    if (sb->shm) {
        shm_secure_wipe(sb->shm);  // 多次覆写（如DoD 5220.22-M标准）
        shm_destroy(sb->shm);
        sb->shm = NULL;
    }

    // 5. 销毁虚拟机环境
    if (sb->vm) {
        vm_destroy(sb->vm);
        sb->vm = NULL;
    }

    // 6. 释放代码镜像
    if (sb->core_code) {
        core_image_destroy(sb->core_code);
        sb->core_code = NULL;
    }

    // 7. 验证无残留
    if (sandbox_residue_check(sb->id) != 0) {
        log_fatal("DESTROY_FAIL: 沙箱 %s 存在数据残留", sb->id);
        return -1;
    }

    // 8. 标记已销毁
    sb->state = SANDBOX_STATE_DESTROYED;

    log_info("DESTROY_OK: 沙箱 %s 已彻底销毁", sb->id);

    // 9. 释放沙箱结构体
    free(sb);
    return 0;
}
```

---

## 九、蓝图基因层面的沙箱定义

### 9.1 沙箱作为蓝图基因的一种形态

在蓝图基因系统中，沙箱商场不是独立实体，而是**商场内核蓝图的「调试表达形态」**：

```
商场内核蓝图（抽象设计文档）
    │
    ├─► 生产表达形态 ──► 生产商场内核（归档态，密系统）
    │
    └─► 调试表达形态 ──► 沙箱商场内核（调试态，非密系统）
            │
            ├─ 代码基因：与生产形态完全相同
            ├─ 数据基因：合成数据引擎生成的虚拟数据
            ├─ 环境基因：透明、开放、无加密
            └─ 生命周期基因：临时、可销毁、不留痕
```

### 9.2 沙箱蓝图基因条目

```json
{
  "gene_id": "sandbox-mall-core-v1",
  "parent_gene": "mall-core-v1",
  "expression_mode": "SANDBOX",
  "security_class": "NON_SECRET",
  "transparency_level": "FULL",
  "encryption_policy": "NONE",
  "data_policy": "SYNTHETIC_ONLY",
  "lifecycle": "EPHEMERAL",
  "isolation_verification": "MANDATORY",
  "destruction_protocol": "SECURE_WIPE",
  "approved_by": ["AICoder", "Mall", "Consumer-Rep"],
  "archived": false
}
```

---

## 十、测试场景：沙箱商场设计检验

### 10.1 隔离测试

| 测试ID | 测试内容 | 预期结果 | 通过标准 |
|--------|---------|---------|---------|
| SB-ISO-001 | 沙箱网络ping生产网关 | 100%丢包 | 网络隔离有效 |
| SB-ISO-002 | 沙箱进程访问生产SHM路径 | 权限拒绝 | SHM隔离有效 |
| SB-ISO-003 | 沙箱Worker尝试出站TCP连接 | 连接拒绝 | Worker出站隔离有效 |
| SB-ISO-004 | 沙箱仓库拉取生产仓库代码 | 404/拒绝 | 仓库隔离有效 |

### 10.2 透明测试

| 测试ID | 测试内容 | 预期结果 | 通过标准 |
|--------|---------|---------|---------|
| SB-TRA-001 | 开发者直接读取沙箱SHM内存 | 明文可读 | 数据透明有效 |
| SB-TRA-002 | 开发者使用gdb附加沙箱内核 | 可单步调试 | 代码透明有效 |
| SB-TRA-003 | 开发者Wireshark抓取沙箱流量 | 明文HTTP可见 | 网络透明有效 |
| SB-TRA-004 | 开发者修改UI代码并热重载 | 修改即时生效 | 可读写挂载有效 |

### 10.3 合成数据测试

| 测试ID | 测试内容 | 预期结果 | 通过标准 |
|--------|---------|---------|---------|
| SB-SYN-001 | 合成频谱含 `SYNTH_` 标记 | 全部数据点含标记 | 标记注入有效 |
| SB-SYN-002 | 合成任务ID以 `TASK-DEBUG-` 开头 | 格式正确 | 虚拟身份有效 |
| SB-SYN-003 | 合成数据引擎尝试读取生产SHM | 权限拒绝 | 数据源隔离有效 |
| SB-SYN-004 | 沙箱运行24小时后数据总量 | 不超过预设上限 | 数据量控制有效 |

### 10.4 销毁测试

| 测试ID | 测试内容 | 预期结果 | 通过标准 |
|--------|---------|---------|---------|
| SB-DES-001 | 销毁后检查SHM残留 | 0字节残留 | 安全擦除有效 |
| SB-DES-002 | 销毁后检查进程残留 | 无相关进程 | 进程清理有效 |
| SB-DES-003 | 销毁后检查网络端口占用 | 端口释放 | 网络清理有效 |
| SB-DES-004 | 销毁后检查日志完整性 | 含完整销毁记录 | 审计留痕有效 |

---

## 十一、法典衔接声明

本蓝图作为《微型商场契约宪法》的沙箱系统架构设计文档，与以下法典直接衔接：

- **v1.3 生命周期管理**：沙箱商场是调试期的唯一合法载体，调试完成后必须销毁，不得直接进入试运行期。
- **v1.3 非密系统铁律**：沙箱商场严格属于非密系统，任何保密措施的混入均视为违规。
- **蓝图基因系统**：沙箱商场作为商场内核蓝图的「调试表达形态」，纳入基因库管理。
- **三权分立体系**：AICoder拥有沙箱蓝图设计权，商场拥有沙箱资源分配与销毁权，消费者在调试期不介入（无真实数据）。
- **SHM内存宪法**：沙箱SHM与生产SHM严格隔离，合成数据引擎不得触碰生产矢量区。

---

> **文档状态**：待审核
> **审核通过后生效**，作为商场UI系统调试期沙箱环境构建的强制性架构蓝图。
