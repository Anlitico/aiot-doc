# LiveKit 连接与信令指南 (LiveKit Transport & Signaling)

本指南详述了 ESP32 如何安全地认证并连接到 LiveKit 服务器。

## 1. 安全认证流程 (Authentication Flow)

为了保护你的 API Key 不被泄露，严禁在固件中硬编码 Secret。

### 交互时序图
1. **设备端**：发起 HTTPS 请求到 `App Server`，提供自己的 DeviceID。
2. **App Server**：验证设备合法性，调用 LiveKit SDK 生成一个 **JWT Access Token**。
3. **App Server**：将 Token 和服务器 URL 返回给设备。
4. **设备端**：使用该 Token 通过 `livekit_room_connect()` 函数直接与 LiveKit 服务器握手。

---

## 2. 接口定义 (API Interface)

### 2.1 获取 Join Token (HTTP GET)
**Endpoint**: `https://api.yourdomain.com/v1/auth/livekit`

**Query Params**:
- `device_id`: 设备的唯一 MAC 地址或序列号。
- `room_name`: (可选) 默认为海绵宝宝的私有房间。

**Success Response (JSON)**:
```json
{
  "status": "success",
  "data": {
    "url": "wss://toy-xxx.livekit.cloud",
    "token": "TOKEN_STRING_HERE",
    "participant_identity": "spongebob_001"
  }
}
```

---

## 3. ESP32 侧状态机 (Connectivity State Machine)

固件需要处理复杂的网络环境，必须实现以下状态转换：

| 状态 (State) | 触发动作 | 异常处理 |
| :--- | :--- | :--- |
| **DISCONNECTED** | 检查 Wi-Fi 是否连接 | 等待 Wi-Fi 自动重连 |
| **GET_TOKEN** | 发起 HTTPS 请求获取门票 | 若 HTTP 失败，指数级退避重试 |
| **CONNECTING** | 调用 `livekit_room_connect()` | 5秒超时则返回 GET_TOKEN 阶段 |
| **CONNECTED** | 订阅远端音频轨，发布本地麦克风轨 | 实时监测连接心跳 |
| **ERROR** | 记录错误码，亮红灯提示 | 3次失败后进入深睡眠或提醒配网 |

---

## 4. 连接时序策略：按需 (On-Demand) vs 预连接 (Pre-Connected)

为了解决首句对话的“愣神”问题，建议采用预连接策略：

### 4.1 推荐流程：长连接预热 (Warm-up)
1. **系统初始化**：Wi-Fi 连通后，立即在后台发起 Token 请求并调用 `livekit_room_connect()`。
2. **待机状态 (Idle)**：保持连接不掉线，但不开启麦克风采集。
3. **唤醒词触发 (Active)**：**瞬间**开启音频 Track 发布，延迟控制在 <200ms。
4. **自动休眠**：若连续 300 秒无语音交互，则断开连接进入低功耗监听模式。

---

## 5. WebRTC 性能参数协商

为了在 ESP32 上获得最佳音质与延迟平衡，建议配置以下参数：

- **音频编码**：Opus (LiveKit 默认支持)。
- **采样率**：16kHz (语音场景的最佳平衡点，兼顾带宽与清晰度)。
- **位率 (Bitrate)**：24kbps (足以承载自然的名人语音)。
- **帧长 (Frame Size)**：20ms (降低端到端延迟)。

---

## 5. 本地连接代码实现思路

```c
// 初始化 LiveKit 配置
lk_room_config_t config = {
    .auto_subscribe = true, // 自动订阅 AI 的声音
    .adaptive_stream = false,
};

// 执行连接
esp_err_t err = lk_room_connect(room_handle, server_url, token, &config);
if (err == ESP_OK) {
    ESP_LOGI(TAG, "成功进入房间，海绵宝宝上线！");
}
```
