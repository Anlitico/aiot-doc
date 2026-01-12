# 系统稳定性与性能调优 (Stability & Performance)

为了确保量产质量，我们应尽量复用乐鑫官方（Espressif）的标准方案。

---

## 1. 内存分配：复用官方 Capability Allocator

不再手动管理每一块 Buffer，而是通过优化 **SDKConfig** 引导系统行为。

### 1.1 核心宏定义 (sdkconfig)
- `CONFIG_SPIRAM_MALLOC_ALWAYS_INTERNAL=4096`：小于 4KB 的申请用 SRAM（快），大于 4KB 的自动去 PSRAM（大）。
- `CONFIG_BT_ALLOCATION_FROM_SPIRAM_FIRST=y`：强制蓝牙协议栈优先使用 PSRAM，为音频算法腾出昂贵的 SRAM。
- `CONFIG_WIFI_LWIP_ALLOCATION_FROM_SPIRAM_FIRST=y`：Wi-Fi 堆栈也移向 PSRAM。

### 1.2 显式代码约束
对于 LiveKit 的大容量 Ringbuffer，推荐使用官方宏：
```c
// 申请 PSRAM 内存的官方标准写法
uint8_t *buf = (uint8_t *)heap_caps_malloc(8192, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
```

---

## 2. 功耗管理：复用官方电源管理 (PM) 框架

不需要手动开关 Modem-sleep，应使用官方的 **自动频率缩放 (DFS)**。

### 2.1 官方 PM 初始化
```c
esp_pm_config_t pm_config = {
    .max_freq_mhz = 240, // 满负载频率
    .min_freq_mhz = 80,  // 空闲时降频
    .light_sleep_enable = true // 允许自动进入 Light-sleep
};
ESP_ERROR_CHECK(esp_pm_configure(&pm_config));
```

### 2.2 效果
- **自动降温**：当海绵宝宝处于 Idle 状态等待唤醒时，系统会自动降频并关闭不必要的时钟，大幅降低芯片表面温度。

---

## 3. 生产测试：复用乐鑫生产工具链 (FCT)

500 台量产建议直接采用乐鑫官方的生产线测试方案。

- **工具链**：复用 **EspRFTestTool** 及其配套的生产下载工具。
- **固件端配合**：
    - 固件支持 **Factory 分区**，内置一段简单的硬件自检逻辑。
    - 产线工人通过串口发送特定的 `AT` 指令或按下组合键触发。
    - **自检内容**：I2S 读取（测麦克风）、I2S 输出（播放 1kHz 测试音测喇叭）、Wi-Fi 信号质量 (RSSI) 抽检。

---

## 4. 故障自我修复：官方看门狗 (WDT)

- **复用官方任务看门狗 (TWDT)**：在主业务逻辑（如 LiveKit 循环）中手动喂狗。
- **系统看门狗 (IWDT)**：应对整个系统的“假死”或硬件锁死，确保设备在无响应 10 秒后自动硬重启。

---

## 5. 本地 UI 音效反馈 (Local Audio Assets)

当 Wi-Fi 未连接或 LiveKit 正在握手时，设备需要本地音频反馈来安抚用户。

### 5.1 资源存储
- **存储位置**：Flash 中的 `storage` 分区 (FATFS/SPIFFS)。
- **推荐格式**：16kHz, Mono, MP3 (由于 CPU 有加速，MP3 比 WAV 更省空间)。

### 1.2 播放逻辑
- **抢占机制**：本地音效播放器应具有高优先级。当触发唤醒但网络未通时，立即播放 `wakeup.mp3`。
- **典型素材包**：
    - `boot.mp3`：开机音乐。
    - `pairing.mp3`：进入配网模式。
    - `net_error.mp3`：断网提示语。

---

## 6. 远程监测与诊断 (ESP-Insights)

500 台外测最担心的就是“死机原因不明”。

- **方案**：接入乐鑫官方 **ESP-Insights**。
- **功能**：
    - **崩溃分析**：设备重启后自动上传 Core Dump，你可以在后台直接看报错在哪一行代码。
    - **指标监控**：监控 PSRAM 剩余量、Wi-Fi 信号强度 (RSSI)、VAD 触发频率。
    - **自定义事件**：记录每台设备每天被用户唤醒了多少次，用于产品迭代决策。

---

## 7. 音频流畅度保障：DMA 与 SRAM

在高并发（Wi-Fi + WebRTC）场景下，I2S 的稳定性至关重要。

- **原则**：**I2S 硬件 DMA 缓冲区严禁放在 PSRAM**。
- **固件对策**：
    - 在初始化 I2S 时，确保分配标志不含 `MALLOC_CAP_SPIRAM`。
    - 确保 `sdkconfig` 里的 `CONFIG_I2S_ISR_IRAM_SAFE` 开启，这样即使正在 Flash 擦写（如 OTA），音频也不会中断。

---

## 8. 存储寿命优化：防止 Flash 磨损

海绵宝宝会频繁保存 Wi-Fi 连接状态和版本号，不合理的写入会缩短设备寿命。

- **官方方案**：坚持使用 `nvs_flash` 库而不是 SPIFFS 来保存配置。
- **优化点**：
    - **合并写入**：不要每变动一个参数就写一次 Flash。建议在内存中维护状态映射，每分钟或在进入休眠前执行一次 `nvs_commit()`。
    - **磨损均衡**：NVS 默认已带磨损均衡，但确保 NVS 分区分配了至少 24KB 空间，以供逻辑层进行物理平衡。

---

## 9. 出口合规固件锁 (FCC Compliance Firmware - Design for US Market)

针对出口美国的设备，固件层面必须施加以下**不可绕过的软件锁**，否则 FCC ID 申请将被驳回。

### 9.1 地区码锁定 (Country Code Lock)
- **要求**：在 `esp_wifi_init()` 之后，必须立即调用 `esp_wifi_set_country()`，并将 Country Code 硬编码为 `US` (United States)。
- **目的**：严禁设备启用 Channel 12, 13, 14，因为这些频段在美国被军方占用，一旦触发会被强制下架。

### 9.2 发射功率限制 (TX Power Limit)
- **要求**：最大发射功率建议限制在 **78 (19.5 dBm)**。不要设为最大值，以减少带外杂散 (Spurious Emission)。
- **代码**：`esp_wifi_set_max_tx_power(78);`
