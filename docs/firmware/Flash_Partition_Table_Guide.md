# 16MB Flash 分区表指南 (Partition Table Guide)

针对 ESP32-S3-WROOM-1-N16R8 模组，必须定制分区表以支持超大音频模型和 OTA 升级。

---

## 1. 推荐的 partitions.csv 配置

请直接将以下内容存为项目根目录下的 `partitions_16mb.csv`：

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     ,        0x10000,
otadata,  data, ota,     ,        0x2000,
phy_init, data, phy,     ,        0x1000,
ota_0,    app,  ota_0,   ,        0x400000,
ota_1,    app,  ota_1,   ,        0x400000,
model,    data, spiffs,  ,        0x400000,
storage,  data, fat,     ,        0x3E0000,
```

### 字段解析：
- **ota_0 / ota_1 (4MB x 2)**：由于采用了全双工 WebRTC + AFE 算法，编译出的 bin 文件通常在 2MB+。分配 4MB 确保了长达 3-5 年的功能迭代空间。
- **model (4MB)**：专门存放 `WakeNet` 唤醒词模型和 `AFE` 权重。
- **storage (约 3.8MB)**：用于存放本地 MP3 提示音（如断网提醒、低电量提醒）。

---

## 2. 工程配置 (sdkconfig)

要让这套分区表生效，你必须在 `menuconfig` 或 `sdkconfig` 中修改：

1. **Flash Size 选择**：
   - `CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y`
   - `CONFIG_ESPTOOLPY_FLASHSIZE="16MB"`
2. **Partition Table 路径**：
   - `CONFIG_PARTITION_TABLE_CUSTOM=y`
   - `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_16mb.csv"`
   - `CONFIG_PARTITION_TABLE_OFFSET=0x8000`

---

## 3. 为什么不把模型塞进代码里？

- **解耦**：将 `Hi! Trump!` 模型放在独立的 `model` 分区，意味着你以后更换唤醒词时，只需要由于局部更新 Flash 扇区，而不需要重新全量编译和 OTA 整个 4MB 的应用固件。
- **提速**：OTA 升级时，如果不涉及模型更新，只需要下载 `ota_app` 分区，极大缩短升级时间。

---

## 4. 固件读取模型的方式

在代码里，使用乐鑫官方的 `esp_partition_find` 接口去挂载模型分区：

```c
const esp_partition_t *model_part = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_DATA_SPIFFS, "model");
// 随后交给 AFE/WakeNet 初始化函数使用
```
