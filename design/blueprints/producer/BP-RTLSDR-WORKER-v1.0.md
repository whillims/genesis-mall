# RTL-SDR Worker 蓝图

> **文档性质**: 微型商场契约宪法技术附录 —— 生产者Worker设计蓝图
> **蓝图ID**: BP-RTLSDR-WORKER-v1.0
> **适用范围**: RTL-SDR硬件采集Worker，实现频谱仪、示波器、功率计功能
> **前置法典**:
> - 商场安全宪法生命周期管理修正案 v1.3 + 非密系统铁律（已生效）
> - 蓝图基因系统规范 v1.0（已生效）
> - SHM内存宪法 v1.0（已生效）
> **版本**: v1.0

---

## 一、关键词定义

| 术语 | 定义 |
|------|------|
| **RTL-SDR** | Realtek RTL2832U DVB-T模块，可作为软件定义无线电接收器，频率范围24MHz-1.7GHz |
| **IQ数据** | 同相/正交(In-phase/Quadrature)复数采样数据，RTL-SDR原生输出格式 |
| **频谱仪模式** | 对IQ数据执行FFT，输出频率-功率谱密度矢量 |
| **示波器模式** | 提取IQ数据的时域波形，输出时间-幅度矢量 |
| **功率计模式** | 计算指定频段的总功率，输出功率值(dBm) |
| **采样率** | RTL-SDR支持采样率范围: 250ksps ~ 2.4Msps（推荐2.048Msps） |
| **增益模式** | RTL-SDR增益控制: 自动增益(AGC)或手动增益(0-49.6dB) |

---

## 二、硬件参数规格

### 2.1 RTL-SDR技术参数

| 参数 | 规格 | 说明 |
|------|------|------|
| **频率范围** | 24MHz ~ 1.7GHz | 覆盖HF/VHF/UHF频段 |
| **采样率** | 250ksps ~ 2.4Msps | 推荐使用2.048Msps稳定采样 |
| **采样精度** | 8-bit I + 8-bit Q | 复数采样，16-bit per sample |
| **ADC分辨率** | 7-bit有效 | 考虑噪声和量化误差 |
| **增益范围** | 0 ~ 49.6dB | 手动增益，步进0.1dB |
| **带宽** | ~2.4MHz | Nyquist带宽 |
| **接口** | USB 2.0 | 即插即用 |
| **驱动** | librtlsdr / pyrtlsdr | Python封装 |

### 2.2 依赖库

```
pyrtlsdr>=0.3.0
numpy>=1.20.0
scipy>=1.7.0
```

---

## 三、Worker架构设计

### 3.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTL-SDR Worker (生产者)                        │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  设备初始化   │    │  参数配置     │    │  生命周期管理  │       │
│  │ rtlsdr_init  │    │ rtlsdr_config │    │ rtlsdr_close │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│           │                  │                  │                │
│           └──────────────────┼──────────────────┘                │
│                              ▼                                   │
│                    ┌──────────────┐                              │
│                    │  IQ数据采集   │                              │
│                    │ rtlsdr_read  │                              │
│                    └──────────────┘                              │
│                              │                                   │
│              ┌───────────────┼───────────────┐                   │
│              ▼               ▼               ▼                   │
│     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐          │
│     │   频谱仪模式   │ │   示波器模式   │ │   功率计模式   │          │
│     │spectrum_analyze│ │oscilloscope  │ │power_meter   │          │
│     └──────────────┘ └──────────────┘ └──────────────┘          │
│              │               │               │                   │
│              └───────────────┼───────────────┘                   │
│                              ▼                                   │
│                    ┌──────────────┐                              │
│                    │  SHM矢量输出  │                              │
│                    │ shm_write    │                              │
│                    └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 函数签名

```python
# 设备初始化
def rtlsdr_init(device_index: int = 0) -> dict:
    """
    初始化RTL-SDR设备
    输入: device_index - 设备索引（多设备时使用）
    输出: {"status": "ok", "device_handle": handle, "serial": "..."} 或错误
    """

# 参数配置
def rtlsdr_config(
    device_handle: object,
    center_freq_hz: int,
    sample_rate_hz: int = 2048000,
    gain_db: float = None,  # None表示自动增益
    ppm_error: int = 0
) -> dict:
    """
    配置RTL-SDR参数
    输入: 设备句柄、中心频率、采样率、增益、PPM校准
    输出: {"status": "ok", "actual_freq": ..., "actual_rate": ...}
    """

# IQ数据采集
def rtlsdr_read(
    device_handle: object,
    num_samples: int = 1024 * 1024
) -> dict:
    """
    读取IQ采样数据
    输入: 设备句柄、采样点数
    输出: {"status": "ok", "i_data": [...], "q_data": [...], "timestamp_ns": ...}
    """

# 频谱仪模式
def spectrum_analyze(
    i_data: list,
    q_data: list,
    sample_rate_hz: int,
    center_freq_hz: int,
    fft_size: int = 1024,
    window: str = "hamming"
) -> dict:
    """
    频谱分析（FFT）
    输入: IQ数据、采样率、中心频率、FFT大小、窗函数
    输出: {"freq_axis": [...], "power_dbm": [...], "peak_freq": ..., "peak_power": ...}
    """

# 示波器模式
def oscilloscope(
    i_data: list,
    q_data: list,
    sample_rate_hz: int,
    mode: str = "iq",  # "i", "q", "iq", "magnitude", "phase"
    time_span_ms: float = None
) -> dict:
    """
    时域波形显示
    输入: IQ数据、采样率、显示模式、时间跨度
    输出: {"time_axis": [...], "amplitude": [...], "mode": ...}
    """

# 功率计模式
def power_meter(
    i_data: list,
    q_data: list,
    center_freq_hz: int,
    bandwidth_hz: int = None,
    impedance_ohm: float = 50.0
) -> dict:
    """
    功率测量
    输入: IQ数据、中心频率、带宽、阻抗
    输出: {"power_dbm": ..., "power_watts": ..., "peak_power_dbm": ...}
    """

# 设备关闭
def rtlsdr_close(device_handle: object) -> dict:
    """
    关闭RTL-SDR设备
    输入: 设备句柄
    输出: {"status": "ok"}
    """
```

---

## 四、详细函数设计

### 4.1 设备初始化函数

```python
def rtlsdr_init(device_index: int = 0) -> dict:
    """
    初始化RTL-SDR设备
    
    参数:
        device_index: 设备索引，默认0（第一个设备）
    
    返回:
        成功: {"status": "ok", "device_handle": RtlSdr对象, "serial": "设备序列号"}
        失败: {"status": "error", "code": "ERR_NO_DEVICE", "message": "..."}
    
    异常处理:
        - 设备未连接: 返回ERR_NO_DEVICE
        - 驱动未安装: 返回ERR_DRIVER_MISSING
        - 设备被占用: 返回ERR_DEVICE_BUSY
    """
    try:
        from rtlsdr import RtlSdr
        
        # 获取设备数量
        device_count = RtlSdr.get_device_count()
        if device_count == 0:
            return {"status": "error", "code": "ERR_NO_DEVICE", "message": "未检测到RTL-SDR设备"}
        
        if device_index >= device_count:
            return {"status": "error", "code": "ERR_INVALID_INDEX", "message": f"设备索引超出范围，共{device_count}个设备"}
        
        # 打开设备
        sdr = RtlSdr(device_index)
        serial = sdr.serial_number if hasattr(sdr, 'serial_number') else "unknown"
        
        return {
            "status": "ok",
            "device_handle": sdr,
            "serial": serial,
            "device_index": device_index
        }
    except ImportError:
        return {"status": "error", "code": "ERR_DRIVER_MISSING", "message": "pyrtlsdr库未安装"}
    except Exception as e:
        return {"status": "error", "code": "ERR_DEVICE_BUSY", "message": str(e)}
```

### 4.2 参数配置函数

```python
def rtlsdr_config(
    device_handle: object,
    center_freq_hz: int,
    sample_rate_hz: int = 2048000,
    gain_db: float = None,
    ppm_error: int = 0
) -> dict:
    """
    配置RTL-SDR参数
    
    参数:
        device_handle: RtlSdr设备句柄
        center_freq_hz: 中心频率(Hz)，范围24e6 ~ 1.7e9
        sample_rate_hz: 采样率(Hz)，推荐2048000
        gain_db: 增益(dB)，None表示自动增益，范围0~49.6
        ppm_error: PPM频率校准值
    
    返回:
        成功: {"status": "ok", "actual_freq": ..., "actual_rate": ..., "gain_mode": ...}
        失败: {"status": "error", "code": "...", "message": "..."}
    """
    try:
        sdr = device_handle
        
        # 验证频率范围
        if center_freq_hz < 24e6 or center_freq_hz > 1.7e9:
            return {"status": "error", "code": "ERR_FREQ_RANGE", "message": "频率超出范围(24MHz-1.7GHz)"}
        
        # 验证采样率
        if sample_rate_hz < 250000 or sample_rate_hz > 2400000:
            return {"status": "error", "code": "ERR_SAMPLE_RATE", "message": "采样率超出范围(250ksps-2.4Msps)"}
        
        # 配置参数
        sdr.center_freq = center_freq_hz
        sdr.sample_rate = sample_rate_hz
        
        # 配置增益
        if gain_db is None:
            sdr.gain = 'auto'
            gain_mode = "auto"
        else:
            sdr.gain = gain_db
            gain_mode = f"manual_{gain_db}dB"
        
        # 配置PPM校准
        if ppm_error != 0:
            sdr.freq_correction = ppm_error
        
        return {
            "status": "ok",
            "actual_freq": sdr.center_freq,
            "actual_rate": sdr.sample_rate,
            "gain_mode": gain_mode,
            "ppm_correction": ppm_error
        }
    except Exception as e:
        return {"status": "error", "code": "ERR_CONFIG_FAIL", "message": str(e)}
```

### 4.3 IQ数据采集函数

```python
def rtlsdr_read(
    device_handle: object,
    num_samples: int = 1024 * 1024
) -> dict:
    """
    读取IQ采样数据
    
    参数:
        device_handle: RtlSdr设备句柄
        num_samples: 采样点数，默认1M点
    
    返回:
        成功: {"status": "ok", "iq_complex": np.array, "timestamp_ns": ..., "sample_count": ...}
        失败: {"status": "error", "code": "...", "message": "..."}
    """
    import numpy as np
    import time
    
    try:
        sdr = device_handle
        
        # 读取IQ数据（复数格式）
        start_time = time.time_ns()
        iq_samples = sdr.read_samples(num_samples)
        end_time = time.time_ns()
        
        # 分离I和Q分量
        i_data = iq_samples.real.tolist()
        q_data = iq_samples.imag.tolist()
        
        return {
            "status": "ok",
            "i_data": i_data,
            "q_data": q_data,
            "iq_complex": iq_samples.tolist(),
            "timestamp_ns": start_time,
            "acquire_time_ns": end_time - start_time,
            "sample_count": len(iq_samples)
        }
    except Exception as e:
        return {"status": "error", "code": "ERR_READ_FAIL", "message": str(e)}
```

### 4.4 频谱仪函数

```python
def spectrum_analyze(
    i_data: list,
    q_data: list,
    sample_rate_hz: int,
    center_freq_hz: int,
    fft_size: int = 1024,
    window: str = "hamming"
) -> dict:
    """
    频谱分析（FFT）
    
    参数:
        i_data: I分量数据
        q_data: Q分量数据
        sample_rate_hz: 采样率
        center_freq_hz: 中心频率
        fft_size: FFT大小，必须是2的幂
        window: 窗函数类型 (hamming/hanning/blackman/rectangular)
    
    返回:
        {"freq_axis": [...], "power_dbm": [...], "peak_freq": ..., "peak_power_dbm": ...}
    """
    import numpy as np
    from scipy import signal
    from scipy.fft import fft, fftfreq
    
    # 重建复数信号
    iq = np.array(i_data) + 1j * np.array(q_data)
    
    # 分段FFT平均（提高频谱稳定性）
    num_segments = len(iq) // fft_size
    if num_segments == 0:
        num_segments = 1
        fft_size = len(iq)
    
    # 选择窗函数
    window_funcs = {
        "hamming": np.hamming,
        "hanning": np.hanning,
        "blackman": np.blackman,
        "rectangular": lambda n: np.ones(n)
    }
    win = window_funcs.get(window, np.hamming)(fft_size)
    
    # 计算平均频谱
    psd_sum = np.zeros(fft_size)
    for i in range(num_segments):
        segment = iq[i * fft_size:(i + 1) * fft_size] * win
        spectrum = fft(segment)
        psd = np.abs(spectrum) ** 2 / (fft_size * np.sum(win ** 2))
        psd_sum += psd
    
    psd_avg = psd_sum / num_segments
    
    # 转换为dBm（假设50欧姆阻抗）
    # dBm = 10*log10(mW) = 10*log10(V^2/R*1000) = 10*log10(V^2) + 10*log10(1000/50)
    # 对于归一化ADC，需要校准因子
    power_dbfs = 10 * np.log10(psd_avg + 1e-12)  # dBFS
    power_dbm = power_dbfs - 30  # 假设校准因子
    
    # 频率轴
    freq_offset = fftfreq(fft_size, 1 / sample_rate_hz)
    freq_axis = center_freq_hz + freq_offset
    
    # FFTshift（将0Hz移到中心）
    freq_axis = np.fft.fftshift(freq_axis)
    power_dbm = np.fft.fftshift(power_dbm)
    
    # 找峰值
    peak_idx = np.argmax(power_dbm)
    peak_freq = freq_axis[peak_idx]
    peak_power = power_dbm[peak_idx]
    
    return {
        "freq_axis": freq_axis.tolist(),
        "power_dbm": power_dbm.tolist(),
        "peak_freq_hz": float(peak_freq),
        "peak_power_dbm": float(peak_power),
        "fft_size": fft_size,
        "num_averages": num_segments,
        "window": window,
        "rbw_hz": sample_rate_hz / fft_size  # 分辨率带宽
    }
```

### 4.5 示波器函数

```python
def oscilloscope(
    i_data: list,
    q_data: list,
    sample_rate_hz: int,
    mode: str = "iq",
    time_span_ms: float = None
) -> dict:
    """
    时域波形显示
    
    参数:
        i_data: I分量数据
        q_data: Q分量数据
        sample_rate_hz: 采样率
        mode: 显示模式
            - "i": 仅显示I分量
            - "q": 仅显示Q分量
            - "iq": 同时显示I和Q
            - "magnitude": 显示幅度 |IQ|
            - "phase": 显示相位 angle(IQ)
        time_span_ms: 时间跨度(ms)，None表示全部数据
    
    返回:
        {"time_axis": [...], "amplitude": [...], "mode": ...}
    """
    import numpy as np
    
    i_arr = np.array(i_data)
    q_arr = np.array(q_data)
    iq = i_arr + 1j * q_arr
    
    # 计算时间轴
    total_samples = len(i_arr)
    total_time_s = total_samples / sample_rate_hz
    
    if time_span_ms is not None:
        span_samples = int(time_span_ms * 1e-3 * sample_rate_hz)
        span_samples = min(span_samples, total_samples)
    else:
        span_samples = total_samples
    
    time_axis = np.arange(span_samples) / sample_rate_hz  # 秒
    
    # 根据模式计算幅度
    if mode == "i":
        amplitude = i_arr[:span_samples]
    elif mode == "q":
        amplitude = q_arr[:span_samples]
    elif mode == "iq":
        amplitude = {
            "i": i_arr[:span_samples].tolist(),
            "q": q_arr[:span_samples].tolist()
        }
    elif mode == "magnitude":
        amplitude = np.abs(iq[:span_samples])
    elif mode == "phase":
        amplitude = np.angle(iq[:span_samples])
    else:
        amplitude = i_arr[:span_samples]
    
    return {
        "time_axis_s": time_axis.tolist(),
        "amplitude": amplitude if isinstance(amplitude, dict) else amplitude.tolist(),
        "mode": mode,
        "sample_count": span_samples,
        "time_span_s": span_samples / sample_rate_hz,
        "sample_rate_hz": sample_rate_hz
    }
```

### 4.6 功率计函数

```python
def power_meter(
    i_data: list,
    q_data: list,
    center_freq_hz: int,
    bandwidth_hz: int = None,
    impedance_ohm: float = 50.0
) -> dict:
    """
    功率测量
    
    参数:
        i_data: I分量数据
        q_data: Q分量数据
        center_freq_hz: 中心频率（用于显示）
        bandwidth_hz: 测量带宽，None表示全带宽
        impedance_ohm: 阻抗(欧姆)，默认50
    
    返回:
        {"power_dbm": ..., "power_watts": ..., "peak_power_dbm": ...}
    """
    import numpy as np
    
    i_arr = np.array(i_data)
    q_arr = np.array(q_data)
    iq = i_arr + 1j * q_arr
    
    # 计算瞬时功率
    # P = |IQ|^2 / R (对于归一化ADC，需要校准因子)
    magnitude_sq = np.abs(iq) ** 2
    
    # 平均功率
    avg_magnitude_sq = np.mean(magnitude_sq)
    avg_power_watts = avg_magnitude_sq / impedance_ohm
    avg_power_dbm = 10 * np.log10(avg_power_watts * 1000 + 1e-15)
    
    # 峰值功率
    peak_magnitude_sq = np.max(magnitude_sq)
    peak_power_watts = peak_magnitude_sq / impedance_ohm
    peak_power_dbm = 10 * np.log10(peak_power_watts * 1000 + 1e-15)
    
    # 峰均比(PAPR)
    papr_db = peak_power_dbm - avg_power_dbm
    
    return {
        "center_freq_hz": center_freq_hz,
        "bandwidth_hz": bandwidth_hz,
        "avg_power_dbm": float(avg_power_dbm),
        "avg_power_watts": float(avg_power_watts),
        "peak_power_dbm": float(peak_power_dbm),
        "peak_power_watts": float(peak_power_watts),
        "papr_db": float(papr_db),
        "impedance_ohm": impedance_ohm,
        "sample_count": len(iq)
    }
```

### 4.7 设备关闭函数

```python
def rtlsdr_close(device_handle: object) -> dict:
    """
    关闭RTL-SDR设备
    
    参数:
        device_handle: RtlSdr设备句柄
    
    返回:
        {"status": "ok"}
    """
    try:
        if device_handle is not None:
            device_handle.close()
        return {"status": "ok"}
    except Exception as e:
        return {"status": "error", "code": "ERR_CLOSE_FAIL", "message": str(e)}
```

---

## 五、SHM矢量输出格式

### 5.1 频谱仪矢量

```json
{
    "vector_type": "spectrum",
    "domain": "rtlsdr_worker",
    "owner": "rtlsdr_producer",
    "timestamp_ns": 1234567890123456789,
    "payload": {
        "freq_axis_hz": [24000000, 24001000, ...],
        "power_dbm": [-60.5, -58.3, ...],
        "peak_freq_hz": 100500000,
        "peak_power_dbm": -42.3,
        "center_freq_hz": 100000000,
        "span_hz": 2048000,
        "rbw_hz": 2000
    }
}
```

### 5.2 示波器矢量

```json
{
    "vector_type": "waveform",
    "domain": "rtlsdr_worker",
    "owner": "rtlsdr_producer",
    "timestamp_ns": 1234567890123456789,
    "payload": {
        "time_axis_s": [0.0, 4.88e-7, 9.77e-7, ...],
        "amplitude": {"i": [0.1, -0.2, ...], "q": [0.05, 0.1, ...]},
        "mode": "iq",
        "sample_rate_hz": 2048000,
        "time_span_s": 0.001
    }
}
```

### 5.3 功率计矢量

```json
{
    "vector_type": "power",
    "domain": "rtlsdr_worker",
    "owner": "rtlsdr_producer",
    "timestamp_ns": 1234567890123456789,
    "payload": {
        "avg_power_dbm": -45.2,
        "peak_power_dbm": -32.1,
        "papr_db": 13.1,
        "center_freq_hz": 100000000,
        "bandwidth_hz": 2048000
    }
}
```

---

## 六、Worker生命周期

### 6.1 状态机

```
┌──────────┐    rtlsdr_init    ┌──────────┐    rtlsdr_config    ┌──────────┐
│   空闲    │ ────────────────► │  已初始化  │ ──────────────────► │  已配置   │
│  (Idle)   │                   │ (Init)    │                     │ (Config) │
└──────────┘                   └──────────┘                     └──────────┘
                                                                      │
                                                                      │ rtlsdr_read
                                                                      ▼
┌──────────┐    rtlsdr_close    ┌──────────┐    数据处理完成    ┌──────────┐
│   关闭    │ ◄──────────────── │  数据采集   │ ◄───────────────── │  处理中   │
│ (Closed)  │                   │ (Sampling) │                     │(Process) │
└──────────┘                   └──────────┘                     └──────────┘
```

### 6.2 错误码定义

| 错误码 | 含义 | 处理建议 |
|--------|------|----------|
| ERR_NO_DEVICE | 未检测到RTL-SDR设备 | 检查USB连接 |
| ERR_DRIVER_MISSING | pyrtlsdr库未安装 | pip install pyrtlsdr |
| ERR_DEVICE_BUSY | 设备被占用 | 关闭其他使用该设备的程序 |
| ERR_INVALID_INDEX | 设备索引无效 | 检查设备数量 |
| ERR_FREQ_RANGE | 频率超出范围 | 使用24MHz-1.7GHz范围 |
| ERR_SAMPLE_RATE | 采样率无效 | 使用250ksps-2.4Msps范围 |
| ERR_CONFIG_FAIL | 配置失败 | 检查参数有效性 |
| ERR_READ_FAIL | 数据读取失败 | 检查设备状态 |
| ERR_CLOSE_FAIL | 设备关闭失败 | 强制释放资源 |

---

## 七、测试场景

### 7.1 设备初始化测试

| 测试ID | 测试内容 | 预期结果 |
|--------|----------|----------|
| RTL-INIT-001 | 无设备时初始化 | 返回ERR_NO_DEVICE |
| RTL-INIT-002 | 正常初始化 | 返回device_handle |
| RTL-INIT-003 | 重复初始化 | 返回ERR_DEVICE_BUSY |

### 7.2 频谱仪测试

| 测试ID | 测试内容 | 预期结果 |
|--------|----------|----------|
| RTL-SPEC-001 | 已知信号频率检测 | peak_freq与信号频率一致 |
| RTL-SPEC-002 | FFT大小变化 | RBW相应变化 |
| RTL-SPEC-003 | 窗函数切换 | 频谱形状正确变化 |

### 7.3 示波器测试

| 测试ID | 测试内容 | 预期结果 |
|--------|----------|----------|
| RTL-OSC-001 | I/Q模式输出 | 同时输出I和Q |
| RTL-OSC-002 | 幅度模式 | 输出|IQ| |
| RTL-OSC-003 | 相位模式 | 输出angle(IQ) |

### 7.4 功率计测试

| 测试ID | 测试内容 | 预期结果 |
|--------|----------|----------|
| RTL-PWR-001 | 已知功率信号测量 | 功率值在校准误差内 |
| RTL-PWR-002 | 峰均比计算 | PAPR正确 |
| RTL-PWR-003 | 阻抗切换 | 功率值按阻抗变化 |

---

## 八、法典衔接

本蓝图作为《微型商场契约宪法》的生产者Worker设计文档，与以下法典衔接：

- **v1.3 非密系统铁律**: 调试期全透明，数据明文输出至SHM
- **蓝图基因系统**: 作为生产者Worker蓝图纳入基因库
- **SHM内存宪法**: 输出矢量写入SHM指定区域
- **三权分立体系**: AICoder拥有设计权，商场拥有部署权

---

## 九、依赖与部署

### 9.1 系统依赖

```bash
# Python依赖
pip install pyrtlsdr numpy scipy

# Windows驱动
# 下载: https://ftp.osmocom.org/binaries/windows/rtl-sdr/
# 安装: 将rtlsdr.dll放入Python目录或系统PATH
```

### 9.2 硬件要求

- RTL-SDR设备（RTL2832U芯片）
- USB 2.0接口
- 天线（根据频段选择）

---

> **文档状态**: 待审核
> **审核通过后生效**，作为RTL-SDR Worker实现的强制性设计蓝图。
