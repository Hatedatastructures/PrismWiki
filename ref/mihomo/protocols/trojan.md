---
title: Trojan
created: 2026-05-17
updated: 2026-05-18
layer: ref
tags: [mihomo, protocol, trojan]
---
# Trojan 协议

Trojan 是一种轻量级 TLS 代理协议，通过 TLS 隐藏代理流量。

## 协议概述

Trojan 特性：
- 基于 TLS 的轻量级代理
- 使用密码认证（SHA224）
- 支持 WebSocket 和 gRPC 传输
- 支持 UDP over TCP
- 支持 Shadowsocks 加密层（Trojan-Go）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/trojan.go` | Trojan 适配器 |
| `transport/trojan/trojan.go` | Trojan 协议实现 |
| `transport/gun/gun.go` | gRPC 传输 |

## 协议背景

Trojan 协议由 GreaterFire 于 2018 年提出，设计目标是创建一种在流量特征上与正常 HTTPS 无法区分的代理协议。

### 设计动机

传统代理协议（如 SOCKS5、HTTP Proxy）具有明显的流量特征，容易被 GFW 等深度包检测系统识别。Trojan 的核心设计理念：

1. **TLS 伪装**：Trojan 的流量与标准 TLS 1.2/1.3 握手完全一致
2. **密码认证**：使用密码而非 UUID 进行认证，简化部署
3. **回退机制**：非 Trojan 流量可回退到正常 Web 服务器，进一步增强隐蔽性

### 与 Shadowsocks 的对比

| 特性 | Shadowsocks | Trojan |
|------|-------------|--------|
| 流量特征 | 加密流量（熵值高，易被识别） | 标准 TLS 流量（与 HTTPS 无异） |
| 认证方式 | 密码（派生密钥） | 密码（SHA224 哈希） |
| 加密 | AEAD（aes-256-gcm 等） | 依赖 TLS 层加密 |
| 隐蔽性 | 中等（加密流量可被特征识别） | 高（与标准 TLS 无异） |
| 回退 | 不支持 | 支持（Fallback 到 Web 服务器） |

### 与 VLESS 的对比

| 特性 | Trojan | VLESS |
|------|--------|-------|
| 认证 | 密码（SHA224） | UUID（16 字节） |
| 地址编码 | SOCKS5 格式 | 自定义二进制格式 |
| XTLS 支持 | 不支持 | 支持 |
| UDP 支持 | UDP over TCP | 原生 UDP + XUDP |
| 多路复用 | gRPC | gRPC + XUDP |

## 帧格式

### 请求帧

Trojan 请求帧采用 SOCKS5 兼容的地址编码格式：

```
[Trojan Request Header]
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Hex(SHA224(password)) (56 bytes)           |
|                                                               |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  CRLF (2B)    |               Command (1B)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   ATYP        |           Address (variable)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Port (2B, Big Endian) |  CRLF (2B)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### 字段详解

| 字段 | 长度 | 格式 | 说明 |
|------|------|------|------|
| `Hex(SHA224(password))` | 56 bytes | ASCII Hex 字符串 | 密码的 SHA224 哈希值，以十六进制字符串形式传输。例如密码为 `password123`，则传输 `SHA224("password123")` 的 56 字符十六进制表示 |
| `CRLF` | 2 bytes | `\r\n` (0x0D 0x0A) | 分隔符，固定值 |
| `Command` | 1 byte | `0x01`=CONNECT<br>`0x02`=UDP ASSOCIATE | 命令类型。CONNECT 用于 TCP 代理，UDP ASSOCIATE 用于 UDP 代理 |
| `ATYP` | 1 byte | `0x01`=IPv4<br>`0x02`=Domain<br>`0x03`=IPv6 | 目标地址类型，与 SOCKS5 协议一致 |
| `Address` | 可变 | - | 目标地址（编码方式同 SOCKS5） |
| `Port` | 2 bytes | Big Endian | 目标端口，网络字节序 |
| `CRLF` | 2 bytes | `\r\n` (0x0D 0x0A) | 结束分隔符 |

#### 地址编码规则

| ATYP | Address 格式 | 示例 |
|------|-------------|------|
| `0x01` (IPv4) | 4 bytes | `1.2.3.4` -> `01 02 03 04` |
| `0x02` (Domain) | 1 byte 长度 (max 255) + N bytes | `google.com` -> `0A 67 6F 6F 67 6C 65 2E 63 6F 6D` |
| `0x03` (IPv6) | 16 bytes | `2001:db8::1` -> `20 01 0D B8 00...00 01` |

#### 密码哈希计算

```
hex(SHA224(password))
```

- 对明文密码计算 SHA224 哈希（28 字节）
- 将哈希值转为 56 字符的十六进制字符串
- 以 ASCII 形式传输（56 字节）

示例：密码 `hello` 的 SHA224 为 `3d2cf44e1f1927297f6009821a9b0a28534d83a227161f360895c6cb`

### 响应帧

Trojan 没有独立的响应帧。服务端在验证密码并连接目标后，直接进入双向数据转发阶段。

### UDP 数据帧

在 UDP ASSOCIATE 模式（Command = `0x02`）下：

```
[Trojan UDP Packet]
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   ATYP        |           Address (variable)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Port (2B, Big Endian) |         Payload (variable)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

每个 UDP 数据包都携带完整的地址信息，使服务端知道将数据转发到哪个目标。

## 通信流程

### TCP CONNECT 模式（Command = 0x01）

```
Client                              Server
  |                                  |
  |-- [TCP Connect] ---------------->|  (端口 443 或自定义)
  |                                  |
  |-- [TLS Handshake] =============>|  标准 TLS 1.2/1.3 握手
  |<-- [TLS Handshake] =============|  与正常 HTTPS 无异
  |                                  |
  |-- [Trojan Request Header] ----->|  发送请求帧
  |   SHA224(password), CRLF,       |
  |   Command, ATYP, Addr, Port     |
  |                                  |
  |                                  |  [验证密码哈希]
  |                                  |  哈希匹配失败 -> [Fallback]
  |                                  |  将连接转交给 Web 服务器
  |                                  |
  |                                  |  哈希匹配成功 ->
  |                                  |  [Connect to Target]
  |                                  |
  |<======== Data Stream (Bidirectional) =========>|
  |                                  |
```

**步骤说明**：

1. **TCP 连接**：客户端连接到 Trojan 服务器（通常 443 端口）
2. **TLS 握手**：标准 TLS 握手，与访问正常 HTTPS 网站完全一致
3. **发送请求帧**：TLS 通道建立后，客户端发送 Trojan 请求帧
4. **密码验证**：服务端计算已知密码的 SHA224 哈希，与接收值比对
5. **Fallback 机制**：若密码不匹配，服务端将连接回退到正常 Web 服务器（如 Nginx）
6. **目标连接**：密码验证通过后，服务端连接到目标地址
7. **双向转发**：进入透明双向数据转发

### UDP ASSOCIATE 模式（Command = 0x02）

```
Client                              Server
  |                                  |
  |-- [TLS + Trojan Request] ------>|  Command = 0x02
  |   (ATYP + Address + Port 为      |  注册 UDP 会话
  |    任意值，仅用于建立会话)        |
  |                                  |
  |-- [UDP Packet 1] -------------->|  ATYP + Addr + Port + Payload
  |                                  |
  |-- [UDP Packet 2] -------------->|  发送到不同目标
  |                                  |
  |<-- [UDP Response 1] ------------|  目标响应
  |                                  |
```

UDP 模式下，所有 UDP 数据包都通过已建立的 TLS 通道传输，每个数据包自包含目标地址。

## 错误处理

### 认证失败（密码不匹配）

Trojan 的独特处理方式是 **Fallback（回退）**：

1. **行为**：服务端不关闭连接，而是将其转交给配置的 Web 服务器
2. **客户端表现**：客户端会收到 Web 服务器的正常 HTTP 响应
3. **隐蔽性**：这使得端口扫描或探测工具看到的是正常的 Web 服务，而非代理服务器

```
[Fallback 示例]

客户端发送错误密码 -> 服务端回退到 Nginx
客户端收到 HTTP 响应:
  HTTP/1.1 200 OK
  Content-Type: text/html
  ...
  <html>Normal Website Content</html>
```

### 目标连接失败

当服务端无法连接目标地址时：

1. **行为**：服务端关闭 TLS 连接
2. **客户端表现**：连接被对端关闭（EOF 或 RST）

### 协议格式错误

当接收到的数据不符合 Trojan 请求帧格式时：

1. 服务端可能将其视为非 Trojan 流量
2. 触发 Fallback 机制（如配置了 Web 回退）
3. 或直接关闭连接（如未配置 Fallback）

## Trojan-Go 扩展

Trojan-Go 是 Trojan 的增强实现，引入了额外特性：

### SS 加密层（Shadowsocks Encryption）

```
[带 SS 加密的 Trojan 数据流]

Payload
  |
  v
+------------------+
|  SS Encrypt      |  <-- aes-128-gcm / chacha20-ietf-poly1305
+------------------+
  |
  v
+------------------+
|  Trojan Header   |
+------------------+
  |
  v
+------------------+
|  TLS             |
+------------------+
```

- 在 Trojan 协议层与 TLS 层之间增加 SS AEAD 加密
- 提供额外的安全性（即使 TLS 被中间人截获，数据仍加密）
- `ss-opts` 配置项控制 SS 加密

### WebSocket 传输

- Trojan 可通过 WebSocket 传输，进一步伪装为 WebSocket 流量
- WS 路径和 Host 可自定义
- 在受限网络环境中可绕过 WebSocket 白名单

### Mux 多路复用

- Trojan-Go 支持类似 SMUX 的连接多路复用
- 在单个 TLS 连接上复用多个 TCP 流
- mihomo 中通过 gRPC 实现类似功能

## 安全考量

### 密码安全

- Trojan 密码仅用于认证，不派生加密密钥
- 密码以 SHA224 哈希形式传输（在 TLS 通道内）
- 建议使用强密码（16+ 字符，高熵值）

### Fallback 配置

- 正确配置 Fallback Web 服务器对隐蔽性至关重要
- Fallback 服务器应返回正常网站内容
- 避免使用空响应或错误页面作为 Fallback

### TLS 指纹

- 客户端 TLS 指纹影响隐蔽性
- 建议使用 `client-fingerprint: chrome` 或类似浏览器指纹
- mihomo 支持多种 TLS 指纹模拟

### Reality 集成

- Trojan 可与 Reality 协议结合
- Reality 提供无证书的 TLS 伪装
- 消除证书验证的可疑性

### 基本配置

```yaml
proxies:
  - name: "trojan-proxy"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    udp: true
```

### WebSocket 配置

```yaml
proxies:
  - name: "trojan-ws"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    network: ws
    ws-opts:
      path: /trojan
      headers:
        Host: server.example.com
```

### gRPC 配置

```yaml
proxies:
  - name: "trojan-grpc"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    network: grpc
    grpc-opts:
      grpc-service-name: trojan
```

### Reality 配置

```yaml
proxies:
  - name: "trojan-reality"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    reality-opts:
      public-key: xxx
      short-id: xxx
```

### Trojan-Go SS 加密

```yaml
proxies:
  - name: "trojan-ss"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    ss-opts:
      enabled: true
      method: aes-128-gcm
      password: ss-password
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `trojan` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `password` | string | 是 | Trojan 密码 |
| `udp` | bool | 否 | 启用 UDP |
| `network` | string | 否 | 传输方式：tcp/ws/grpc |
| `sni` | string | 否 | TLS SNI |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `fingerprint` | string | 否 | TLS 指纹 |
| `ech-opts` | object | 否 | ECH 配置选项 |
| `reality-opts` | object | 否 | Reality 配置选项（含 public-key、short-id） |
| `client-fingerprint` | string | 否 | TLS 客户端指纹（如 chrome） |

## 与 Prism 的兼容性

### 六阶段流水线映射

Prism 使用六阶段流水线架构处理 Trojan 协议：

| 阶段 | 职责 | Trojan 处理内容 |
|------|------|-----------------|
| `recognition` | 协议识别 | TLS 握手完成后识别 Trojan 请求帧（56 字节 SHA224 + CRLF 模式） |
| `pipeline` | 协议解析 | 解析 Trojan 帧，提取 Command、ATYP、Address、Port |
| `channel` | 连接建立 | 根据解析结果建立到目标的 TCP/UDP 连接 |
| `multiplex` | 多路复用 | 通过 gRPC 实现连接级多路复用 |
| `transport` | 传输层 | TLS/Reality 加密处理，Fallback 机制 |
| `forward` | 数据转发 | 双向透明数据转发 |

### 模块兼容性详情

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Trojan 协议标准实现，支持所有传输模式 |
| Pipeline | 完全兼容 | 支持 StreamConnContext，Trojan 帧解析与转发 |
| Multiplex | 支持 | 通过 gRPC 实现多路复用 |
| Crypto | 间接依赖 | Trojan 自身无加密（SS 扩展除外），依赖 TLS 模块 |
| TLS | 完全兼容 | 支持 TLS 1.2/1.3、Reality、ECH、Fallback |

### Prism 特有优化

- **Fallback 处理**：Prism 支持 Trojan Fallback 机制，认证失败时将连接转交给内置/外部 Web 服务器
- **内存池分配**：Trojan 请求帧使用 PMR 内存池分配
- **协程异步**：C++23 协程实现异步 Trojan 帧处理和 Fallback 调度

## Prism 实现边界

- 密码以 SHA224 hex（56 字节）明文在 TLS 内层传输
- 缓冲区 320 字节固定，域名长度 255 时零余量
- 日志不区分"格式错误"和"密码错误"

详见 [[core/protocol/trojan/format#实现边界|Trojan 实现边界]]

## 相关文档

- [[ref/protocol/trojan-spec|trojan-spec]] - Trojan 协议规范
- [[reality]] - Reality TLS 隐蔽
- [[../transport/gun]] - gRPC 传输