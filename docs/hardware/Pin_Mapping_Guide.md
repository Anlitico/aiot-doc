# 软硬接口定义表 (Pin Mapping Guide)

本协议是硬件 PCB 设计与固件代码编写的共同依据，任何引脚变更均需双方同步。

## 1. 核心管脚分配表

针对 **ESP32-S3-WROOM-1-N16R8** 模组优化的引脚分配：

| 类别 | 信号名称 (Signal) | GPIO 编号 | 功能说明 | 固件配置常量 |
| :--- | :--- | :--- | :--- | :--- |
| **音频输入 (Mic)** | I2S_M_WS | **GPIO 4** | 麦克风左右声道切换 | `MIC_I2S_WS` |
| | I2S_M_SCK | **GPIO 5** | 麦克风时钟信号 | `MIC_I2S_BCK` |
| | I2S_M_DATA | **GPIO 6** | 麦克风数据输入 (DIN) | `MIC_I2S_DIN` |
| **音频输出 (Amp)** | I2S_A_WS | **GPIO 15** | 功放采样切换 (LRCK) | `SPK_I2S_WS` |
| | I2S_A_SCK | **GPIO 16** | 功放串行时钟 (BCLK) | `SPK_I2S_BCK` |
| | I2S_A_DATA | **GPIO 17** | 功放数据输出 (DOUT) | `SPK_I2S_DOUT` |
| **交互反馈** | RGB_DATA | **GPIO 18** | 驱动 WS2812B 全彩灯珠 | `GPIO_RGB_LED` |
| | TOUCH_PAD | **GPIO 2** | 捏一捏唤醒 (Touch Sensor) | `TOUCH_WAKEUP` |
| **电源监测** | BAT_ADC | **GPIO 1** | 锂电池电压监测 (分压采穴) | `ADC_BAT_SENSE` |
| **系统/调试** | USB_D+ | **GPIO 20** | 原生 USB 烧写与 Log | - |
| | USB_D- | **GPIO 19** | 原生 USB 烧写与 Log | - |
| | BOOT_KEY | **GPIO 0** | 物理下载模式按键 | - |

## 2. 严禁占用的引脚 (Forbidden Pins)

以下引脚已被模组内部闪存 (Flash) 和内存 (PSRAM) 占用，**严禁在 PCB 上拉线或悬空接任何零件**：
- **GPIO 26 ~ 32**
- **GPIO 33 ~ 37** (针对 N16R8 的 Octal SPI 专用)

## 3. 设计注意事项 (For Hardware Engineer)

1. **AEC 回声消除参考信号**：
   - **重要**：本项目采用乐鑫官方推荐的“软件内部环回”方案。固件会自动从播放任务中复制数据给 AFE。
   - **结论**：**硬件上不需要**从功放输出端引回任何信号线。这节省了引脚并简化了电路。
2. **I2S 走线**：音频信号线长度应尽量相等，且包地处理，远离 Wi-Fi 天线。
3. **阻抗匹配**：GPIO 19/20 的 USB 差分线建议走 90 欧姆阻抗控制。
4. **ESD 防护 (玩具安全)**：
   - USB 接口路径上必须增加集成的 ESD 保护二极管（如 SRV05-4）。
   - 麦克风拾音孔和外露的 LED 灯珠建议在 PCB 边缘增加静电泄放环 (ESD Ring)。
5. **散热设计 (Thermal)**：
   - ESP32-S3 底部焊盘 (GND Pad) 必须与 PCB 主地层通过 9-16 个过孔连接。
   - MAX98357A 功放底部同样需要过孔散热，防止大音量对话时热保护触发断音。
6. **上拉/下拉**：
   - GPIO 0 在 PCB 上需预留一个 10k 欧姆上拉电阻到 3.3V。
   - GPIO 45/46 启动时需保持默认电平（建议悬空），不要加外部强拉负载。

## 4. 固件宏定义定义建议 (For Firmware Engineer)

```c
// 在项目的 board_def.h 中定义
#define MIC_I2S_PORT   I2S_NUM_0
#define SPK_I2S_PORT   I2S_NUM_1

#define BOARD_MIC_I2S_WS      (4)
#define BOARD_MIC_I2S_BCK     (5)
#define BOARD_MIC_I2S_DIN     (6)

#define BOARD_SPK_I2S_WS      (15)
#define BOARD_SPK_I2S_BCK     (16)
#define BOARD_SPK_I2S_DOUT    (17)

#define BOARD_RGB_LED_PIN     (18)
#define BOARD_BAT_ADC_CHAN    ADC_CHANNEL_0 // GPIO 1
```
