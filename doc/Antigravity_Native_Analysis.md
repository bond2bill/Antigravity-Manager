# Antigravity 原生请求与设备指纹技术分析报告

本报告基于对 Antigravity 原生应用（`extension.js` 和 `workbench.desktop.main.js`）的逆向工程分析，旨在为反代服务器（Proxy Server）提供精准的请求模拟与优化指南。

---

## 1. Connect-RPC 传输层实现

Antigravity 深度集成了 `@connectrpc/connect` 框架，采用 Connect 协议而非纯 gRPC。

### 核心特征：
- **协议类型**：HTTP/1.1 或 HTTP/2 承载的 Connect Protocol。
- **Content-Type**：
    - 文本类（Chat/Edit）：`application/connect+json`
    - 二进制类：`application/connect+proto`
- **流式响应**：AI 响应使用分块传输（Chunked Encoding）。反代服务器必须配置 `proxy_buffering off` 并在 Header 中添加 `X-Accel-Buffering: no` 以保证低延迟。
- **Trailers 处理**：Connect 协议通过 HTTP Trailers 传递 `grpc-status`。反代服务器需确保不丢失响应末尾的 Trailers 头部。

---

## 2. Checksum（校验和）生成算法

这是绕过服务端风控的最关键逻辑。原生应用对每一个涉及 AI 逻辑的请求体进行完整性校验。

### 生成逻辑：
1. **算法**：标准 **SHA-256**。
2. **输入**：序列化后的 JSON 字符串（Request Body）。
3. **输出格式**：Hex 或 Base64（取决于具体的 Header 要求）。
4. **注入字段**：通常为 `x-cursor-checksum` 或 `x-antigravity-checksum`。

### 实现代码参考（Node.js）：
```javascript
const crypto = require('crypto');

/**
 * 生成请求体校验和
 * @param {Object|string} body 请求体内容
 * @returns {string} SHA-256 哈希值 (Hex)
 */
function generateChecksum(body) {
    const content = typeof body === 'string' ? body : JSON.stringify(body);
    return crypto.createHash('sha256').update(content).digest('hex');
}
```

---

## 3. 设备指纹 (machineId & deviceId) 获取

Antigravity 使用多级 ID 交叉识别，防止单设备多账号滥用。

### ID 构成：
1. **`machineId` (静态 UUID)**：
    - 路径：`~/Library/Application Support/Antigravity/storage.json`
    - 特点：首次运行生成，持久化存在，是最核心的身份标识。
2. **`macMachineId` (硬件哈希)**：
    - 逻辑：获取物理网卡 MAC 地址 -> `sha256(mac + salt)`。
3. **`devDeviceId` (系统 UUID)**：
    - macOS: `ioreg -rd1 -c IOPlatformExpertDevice` 中的 `IOPlatformUUID`。
    - Linux: `/etc/machine-id`。

### 生成算法：
```javascript
// 模拟 macMachineId 生成
function getMacMachineId(macAddress) {
    const salt = "antigravity-salt-2024"; // 示例盐值
    return crypto.createHash('sha256').update(macAddress + salt).digest('hex');
}
```

---

## 4. 元数据 (Metadata) 与 Headers 构造

所有的 RPC 请求都通过 `buildHeaders` 函数统一构造。

### 核心 Headers 列表：
| Header | 说明 | 是否必选 |
| :--- | :--- | :--- |
| `User-Agent` | 通常为 `Antigravity/0.x.x (darwin; x64)` | 是 |
| `X-Cursor-Client-Version` | 匹配原生应用版本号 | 是 |
| `X-Cursor-Id` | 填入 `machineId` | 是 |
| `X-Machine-Id` | 填入硬件哈希指纹 | 是 |
| `X-Ghost-Token` | 用户鉴权 Token | 是 |
| `Connect-Protocol-Version` | 固定为 `1` | 是 |

---

## 5. 反代服务器优化建议

为了使本项目的反代请求完美绕过检测并提升性能，请按以下步骤优化：

### 5.1 指纹持久化绑定
不要为每个请求生成随机指纹。
- **策略**：建立 `Token -> DeviceConfig` 的映射表。
- **效果**：确保同一账号在后端看到的设备 ID 始终如一，避免触发“异地多设备”风险提示。

### 5.2 动态 Checksum 重写
如果反代层修改了 Body（如注入自定义 Prompt）：
- **必须**在发送请求前重新计算 SHA-256。
- **更新** `x-cursor-checksum` 头部，否则后端会报 `400 Bad Request` 或 `Checksum Failed`。

### 5.3 连接复用 (Keep-Alive)
- **优化**：开启上游连接池。
- **原因**：原生 Connect-RPC 频繁发送短小的请求，开启 Keep-Alive 可减少 100ms+ 的握手延迟。

### 5.4 UA 严格匹配
- **检查**：User-Agent 中的系统版本（如 `darwin; x64`）必须与指纹中提取的硬件架构一致。

---
*本报告由 Antigravity 自动化分析工具生成。*
