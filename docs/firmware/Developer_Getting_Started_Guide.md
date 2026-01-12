# 开发者快速上手指南 (Developer Getting Started Guide)

本指南专门为从未接触过 IoT 开发的工程师设计，旨在帮助你快速在你的电脑上搭建起 ESP32-S3 的开发环境，并在 **ESP32-Korvo-1** 开发板上运行起第一个音频程序。

---

## 1. 准备工作 (Prerequisites)

在开始安装之前，请确保你的电脑上已经安装了以下基础工具：
- **Python 3.x**：[官方下载](https://www.python.org/downloads/) (注意勾选 "Add Python to PATH")。
- **Git**：[官方下载](https://git-scm.com/downloads/)。
- **VS Code**：[官方下载](https://code.visualstudio.com/)。

---

## 2. 搭建 ESP-IDF 开发环境 (核心)

ESP-IDF 是乐鑫官方的物联网开发框架，它是所有代码运行的基础。

### 2.1 安装 VS Code 插件
1. 打开 VS Code，点击左侧的“插件”图标。
2. 搜索并安装 **"Espressif IDF"**。
3. 安装完成后，按下 `F1` 键，输入 `ESP-IDF: Configure ESP-IDF Extension`。
4. 选择 **"Express"** 模式：
   - **ESP-IDF Version**：选择 `v5.1.2` (建议使用稳定版本 5.1 以上)。
   - **ESP-IDF Tools Path**：建议放在一个没有中文和空格的路径下（例如 `C:\esp_tools`）。
5. 点击 **"Install"**，然后静静等待安装完成。

### 2.2 验证安装
按下 ``Ctrl + ` `` 打开终端，输入：
```bash
idf.py --version
```
如果显示 `ESP-IDF v5.1.2`，恭喜你，底座搭好了！

---

## 3. 集成音频框架 ESP-ADF

ESP-ADF 是专门处理音乐、录音、音频流水线的扩展库。

1. **克隆源码**：
   找一个合适的目录，在终端运行：
   ```bash
   git clone --recursive https://github.com/espressif/esp-adf.git
   ```
2. **设置环境变量**：
   - 在 VS Code 设置中搜索 "ESP-ADF Path"，并将刚才克隆的 `esp-adf` 文件夹路径填进去。
   - 或者在系统环境变量中添加 `ADF_PATH`，路径为你克隆的地址。

---

## 4. 硬件连接 (ESP32-Korvo-1)

1. **物理连接**：
   - 使用两根 Micro-USB 线连接开发板：
     - **UART 口**：用于烧录程序和查看调试信息（Log）。
     - **USB 口**：S3 原生 USB 调试，可选。
2. **驱动安装**：
   - 如果电脑没反应，通常需要安装 **CP210x USB to UART Bridge** 驱动。
3. **识别串口**：
   - 在 VS Code 底部工具栏点击“串口”图标，选择对应的 `COM` 口（Windows）或 `/dev/cu.usbserial`（Mac）。

---

## 5. 运行第一个“响声”程序 (Hello World)

我们要让板子播放一段 MP3。

1. **导入项目**：
   - 在 VS Code 中按下 `F1`，选择 `ESP-IDF: Show Examples Projects`。
   - 找到 `ESP-ADF` 的示例：`get-started/play_mp3`。
2. **配置与编译**：
   - 点击底部工具栏的“垃圾桶”图标（Full Clean）。
   - 点击“齿轮”图标（Build）。第一次编译较慢，需要 3-5 分钟。
3. **烧录与观察**：
   - 点击“闪电”图标（Flash）。
   - 烧录完成后，点击“小显示器”图标（Monitor）。
   - **成功标志**：你会看到终端滚过大量的调试信息，同时板子的喇叭里传出了音乐！

---

## 6. 新手核心心法 (Core Concepts)

- **Flash vs RAM**：
  - **Flash** 就像硬盘，存放你的代码。
  - **RAM (PSRAM)** 就像内存，LiveKit 推流时的数据都在这里缓存。
- **Monitor 是你的眼睛**：
  - 如果板子没反应，一定要看 Monitor 里的输出。
  - 红色字体代表 Error，这是解决问题的唯一依据。
- **文档是你的导师**：
  - 遇到坑先查 [乐鑫官方在线文档](https://docs.espressif.com/)。
