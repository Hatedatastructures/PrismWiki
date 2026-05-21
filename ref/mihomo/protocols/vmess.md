---
title: VMess
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, vmess]
---
# VMess 协议

VMess 是 V2Ray 项目自主研发的核心代理协议，设计之初即强调安全性与反检测能力。该协议通过多层加密、动态头部和认证机制，使得中间人难以识别、篡改或重放通信内容。mihomo 在 `transport/vmess/` 下实现了完整的 VMess 客户端能力。

## 协议背景

VMess 协议诞生于 2016 年，旨在解决早期代理协议（如 SOCKS、Shadowsocks 原始版）存在的以下问题：

1. **特征指纹明显**：旧协议在 TCP 流中具有固定字节模式，容易被 DPI 深度包检测识别。
2. **缺乏重放保护**：相同明文产生相同密文，攻击者可截获并重放数据包。
3. **认证与传输耦合**：认证信息与业务数据未分层，导致灵活性不足。

VMess 通过以下机制应对这些问题：

- **Header 加密**：使用 AES-128-CFB 加密请求头部，使得流量分析无法识别目标地址和端口。
- **AEAD 模式**：v2 版本引入 AEAD（Authenticated Encryption with Associated Data），提供认证加密和防重放能力。
- **动态填充**：Global Padding 和 Authenticated Length 使数据包大小随机化，抵抗流量分类。
- **UUID + AlterID 认证**：服务端通过 UUID 识别客户端身份，AlterID（已弃用）用于多用户哈希派生。

### AEAD 模式与旧版的区别

| 特性 | 旧版（Legacy） | AEAD 模式 |
|------|---------------|-----------|
| 加密方式 | AES-CFB / ChaCha20 | AES-GCM / ChaCha20-Poly1305 |
| 认证 | HMAC-SHA256 校验和 | 内置 AEAD Tag（16 字节） |
| 重放保护 | 无 | 时间戳 + Nonce 去重 |
| 头部保护 | CFB 加密，无完整性校验 | AEAD 认证头部，防篡改 |
| 填充 | 固定长度填充 | 动态随机填充 |

在 AEAD 模式下，每个数据包使用独立派生的 Key 和 IV，确保即使明文相同，密文也完全不同。

## 帧格式

### VMess 请求帧（TCP / AEAD 模式）

完整的一次 VMess 请求按以下字节顺序构造：

```
+------------------+-------------------------------------------------+
| 字段             | 说明                                            |
+------------------+-------------------------------------------------+
| IV (16 bytes)    | 随机生成的初始化向量，用于 AES-CFB 加密头部     |
| Header (N bytes) | 经 AES-128-CFB 加密的 VMess 请求头（见下方）    |
| Header Hash      | Header 的 MD5 校验（旧版）或 AEAD Tag（AEAD 版）|
| Payload          | 加密的业务数据（按 AEAD Chunk 分包）            |
+------------------+-------------------------------------------------+
```

### 加密后的 Header 结构（解密后）

```
+--------+---------+----------+----------+---------+----------+-------+
| Cmd    | Option  | DataLen  | Request  | Address | Target   | PadLen|
| (1B)   | (1B)    | (2B/4B)  | ID(16B)  | Port(2B)| VarLen   | (1B)  |
+--------+---------+----------+----------+---------+----------+-------+
| Padding (VarLen) | FallbackIV (4B) |
+------------------+-----------------+
```

**字段说明**：

| 字段 | 长度 | 说明 |
|------|------|------|
| Cmd | 1 字节 | 命令类型：0x01 = TCP, 0x02 = UDP |
| Option | 1 字节 | 标志位：Bit0=全局填充, Bit1=连接复用, Bit2=流式传输 |
| DataLen | 2 字节（v1）/ 4 字节（v2） | 后续数据区域总长度 |
| Request ID | 16 字节 | 从 UUID 派生的请求标识，用于服务端匹配 |
| Address Port | 2 字节 | 目标端口（网络字节序） |
| Target Address | 变长 | 目标地址：1字节类型 + 地址数据（IPv4=4B, IPv6=16B, Domain=1B长度+字符串） |
| Padding | 变长 | 随机填充字节，使总长度为 16 的倍数 |
| FallbackIV | 4 字节 | 旧版兼容字段，AEAD 模式下忽略 |

### AEAD 数据 Chunk 帧

AEAD 模式将业务数据分块传输，每块格式如下：

```
+-------------------+------------------+------------------+
| AuthLen (2 bytes) | AuthData (N B)   | AEAD Tag (16 B)  |
+-------------------+------------------+------------------+
```

- `AuthLen`：大端序 uint16，表示 `AuthData` 的长度（最高位为 1 表示最后一个 Chunk）。
- `AuthData`：加密的业务数据。
- `AEAD Tag`：Poly1305 或 GCM 认证标签。

每个 Chunk 的 Key 和 IV 由主 Key 和当前 Nonce 计数器派生：

```
ChunkKey  = HKDF-SHA256(MainKey, "vmess-aead-chunk-key" + Counter)
ChunkIV   = HKDF-SHA256(MainKey, "vmess-aead-chunk-iv" + Counter)
Counter  += 1
```

## 协议版本

### v1（Legacy）

- DataLen 为 2 字节（最大 64 KB 头部）。
- 使用 MD5 校验头部完整性。
- 使用 AES-128-CFB 加密数据载荷。
- 无内置重放保护。
- AlterID 用于多用户派生（从 UUID + AlterID 派生多个 UUID）。

### v2（AEAD）

- DataLen 扩展为 4 字节。
- 使用 AEAD 认证加密（AES-128-GCM 或 ChaCha20-Poly1305）。
- 引入时间戳字段（8 字节），服务端进行时间窗口校验（默认 +/- 120 秒），拒绝超期请求。
- Nonce 计数器用于去重，防止重放攻击。
- 弃用 AlterID（设为 0）。

## Key/IV 派生流程

VMess AEAD 模式的密钥派生流程如下：

```
UUID + AlterID (Legacy)
       │
       ▼
  ┌─────────────┐
  │  MD5 派生    │  ← v1: MD5(UUID + AlterID × N) 产生 N 个派生 UUID
  └─────────────┘
       │
       ▼ (v2 直接使用 UUID)
  ┌─────────────┐
  │  AES-128    │  ← 从 UUID 字节直接截取为 Key (16B)
  │  Key/IV     │  ← IV 同样从 UUID 截取（不同偏移）
  └─────────────┘
       │
       ▼
  ┌─────────────┐
  │  Chunk Key  │  ← 每块数据使用 HKDF 派生独立 Key/IV
  │  / IV       │
  └─────────────┘
```

具体步骤：

1. **主 Key 派生**：将 UUID（16 字节）直接用作 AES-128 主 Key。
2. **Header IV**：从 UUID 的特定偏移提取 16 字节作为 Header 加密 IV。
3. **AEAD 会话 Key**：对 Header 中的时间戳做 HMAC 运算，生成会话 Key。
4. **Chunk 派生**：每发送/接收一个 Chunk，计数器递增，HKDF 派生新的 Key/IV。

## 通信流程

### 客户端到服务端的完整握手时序

```
客户端                                                    服务端
   │                                                         │
   │  1. 生成随机 IV (16B)                                   │
   │  2. 构造 VMess Header                                   │
   │     - Cmd (TCP/UDP)                                     │
   │     - Option 标志                                       │
   │     - 时间戳 (v2)                                       │
   │     - 目标地址 + 端口                                    │
   │     - 随机填充                                          │
   │  3. AES-128-CFB 加密 Header                             │
   │  4. 发送: IV + 加密Header + AuthTag                     │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  5. 接收 IV，用相同 Key 解密 Header
   │                                                         │  6. 校验时间戳窗口（v2: +/- 120s）
   │                                                         │  7. 校验 Request ID / UUID
   │                                                         │  8. 解析目标地址
   │                                                         │
   │                                                         │  9. 建立到目标的 TCP/UDP 连接
   │                                                         │
   │  10. 业务数据分 Chunk 加密传输                           │
   │ ◄────────────────────────────────────────────────────── │  11. 转发目标服务器响应（同样分 Chunk）
   │                                                         │
   │  12. 最后一个 Chunk (AuthLen 最高位=1)                   │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │
```

### WebSocket / gRPC 传输封装

当 `network` 设置为 `ws`、`grpc`、`h2` 或 `http` 时，VMess 帧被封装在对应协议载荷内：

- **WebSocket**：VMess 帧作为 WebSocket Binary Frame 的 payload 传输。
- **gRPC**：通过 gRPC `Tun` 服务的双向流传输，VMess 帧作为 Protobuf `byte` 字段。
- **HTTP/2**：通过 HTTP/2 数据流传输，VMess 帧作为 HTTP Body。
- **HTTP**：伪装为普通 HTTP 请求/响应，VMess 帧嵌入 Body。

## 错误处理

### 认证失败

当服务端检测到以下情况时，连接被立即关闭：

| 错误类型 | 触发条件 | 服务端行为 |
|---------|---------|-----------|
| UUID 不匹配 | Request ID 无法匹配任何已注册用户 | 直接关闭连接，不返回错误信息 |
| 时间戳超期 | 请求时间戳与服务端时间差 > 120 秒 | 拒绝请求，记录警告日志 |
| Header 校验失败 | MD5 校验和（v1）或 AEAD Tag（v2）不匹配 | 直接关闭连接 |
| 协议版本不支持 | Cmd 字段不在已知范围内 | 丢弃连接 |

### 重放检测

AEAD 模式下，服务端维护一个滑动窗口记录已使用的 Nonce/时间戳组合：

1. 收到请求后，提取时间戳和 Nonce。
2. 检查是否在该时间窗口内已出现过相同的 Nonce。
3. 如果已出现，判定为重放攻击，静默丢弃。
4. 窗口大小由服务端配置，通常为 120 秒。

### 解密失败

Chunk 级别的 AEAD Tag 校验失败：

- 当前 Chunk 被丢弃。
- 连接标记为不可信，后续 Chunk 不再处理。
- 连接最终超时关闭。

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | VMess 协议实现，支持 v1/v2/AEAD |
| Pipeline | 完全兼容 | 支持 StreamConnContext 封装 |
| Crypto | 完全兼容 | 使用 crypto/aead、crypto/block 模块 |
| Multiplex | 支持 | XUDP 模式通过内部多路复用 |
| Stealth | 部分兼容 | Reality/TLS 伪装需额外配置 |

Prism 的 VMess 实现注意点：

- `crypto/aead` 模块提供 Chunk 级 AEAD 加解密。
- `crypto/block` 模块提供 Header 级 AES-CFB 加密。
- Header 中的地址编码（IPv4/IPv6/Domain）与 Prism 的地址抽象兼容。
- 时间戳校验依赖系统时钟，建议配置 NTP 同步。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/vmess.go` | VMess 适配器 |
| `transport/vmess/vmess.go` | VMess 传输层实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "vmess-proxy"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    udp: true
```

### WebSocket 配置

```yaml
proxies:
  - name: "vmess-ws"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: ws
    tls: true
    ws-opts:
      path: /vmess
      headers:
        Host: server.example.com
```

### gRPC 配置

```yaml
proxies:
  - name: "vmess-grpc"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: grpc
    tls: true
    grpc-opts:
      grpc-service-name: vmess
```

### HTTP/2 配置

```yaml
proxies:
  - name: "vmess-h2"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: h2
    tls: true
    h2-opts:
      host:
        - server.example.com
      path: /vmess
```

### HTTP 配置

```yaml
proxies:
  - name: "vmess-http"
    type: vmess
    server: server.example.com
    port: 80
    uuid: your-uuid
    network: http
    http-opts:
      method: GET
      path:
        - /vmess
      headers:
        Host:
          - server.example.com
```

### Reality 的配置

```yaml
proxies:
  - name: "vmess-reality"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    reality-opts:
      public-key: xxx
      short-id: xxx
    client-fingerprint: chrome
```

### XUDP 配置

```yaml
proxies:
  - name: "vmess-xudp"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    xudp: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `vmess` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `uuid` | string | 是 | 用户 UUID |
| `alterId` | int | 否 | AlterID（已弃用） |
| `cipher` | string | 否 | 加密算法 |
| `tls` | bool | 否 | 启用 TLS |
| `network` | string | 否 | 传输方式 |
| `udp` | bool | 否 | 启用 UDP |
| `xudp` | bool | 否 | 启用 XUDP |
| `servername` | string | 否 | TLS 服务器名称（覆盖 SNI） |
| `packet-addr` | bool | 否 | 启用 PacketAddr 模式（与 xudp 互斥） |
| `packet-encoding` | string | 否 | UDP 包编码：packetaddr/xudp |
| `global-padding` | bool | 否 | 启用全局填充（增强隐蔽性） |
| `authenticated-length` | bool | 否 | 启用认证长度字段 |
| `client-fingerprint` | string | 否 | TLS 客户端指纹（如 chrome） |

## 加密选项

| Cipher | 描述 |
|--------|------|
| `auto` | 自动选择（默认） |
| `aes-128-gcm` | AES-128-GCM |
| `chacha20-poly1305` | ChaCha20-Poly1305 |
| `none` | 无加密（不推荐） |

## 相关文档

- [[vless]] - VLESS 协议
- [[ref/protocol/vmess-spec|vmess-spec]] - VMess 协议规范
- [[core/crypto/aead|aead]] - AEAD 加密
