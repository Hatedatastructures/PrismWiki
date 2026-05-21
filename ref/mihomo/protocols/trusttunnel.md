---
title: TrustTunnel
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, trusttunnel]
---
# TrustTunnel 协议

TrustTunnel 是一种基于 TLS/QUIC 传输的隐蔽代理协议，通过自定义 TLS 扩展和认证机制实现安全的代理通信。它在 mihomo 中作为高级代理协议支持，提供了灵活的传输模式选择（TLS 或 QUIC）以及内置的用户认证。

## 协议概述

TrustTunnel 的核心设计理念是将代理协议完全嵌入到 TLS/QUIC 传输层中，使其在网络层面表现为标准的加密连接。其特性包括：

- **双传输模式**：支持 TLS over TCP 和 QUIC over UDP
- **用户认证**：内置用户名/密码认证机制
- **UDP 支持**：通过 TLS 隧道或 QUIC 原生支持 UDP 转发
- **健康检查**：内置连接健康检测机制
- **ECH 集成**：支持 Encrypted Client Hello 扩展
- **自定义 TLS 扩展**：通过自定义 TLS 扩展携带认证信息和协议标识
- **拥塞控制**：QUIC 模式下支持 BBR 等拥塞控制算法

TrustTunnel 与 AnyTLS 和 Reality 同属 TLS 隐蔽技术家族，但 TrustTunnel 的独特之处在于其自定义 TLS 扩展机制和灵活的传输模式切换能力。

## 自定义 TLS 扩展

### TLS 扩展机制

TrustTunnel 利用 TLS 扩展（TLS Extension，RFC 8446 Section 4.2）机制在 TLS Client Hello 中携带自定义数据。TLS 扩展是 TLS 协议的标准机制，允许客户端和服务器在握手阶段交换额外信息。

```
TLS Client Hello 结构:
+---------------------------+
| 版本号                     |
+---------------------------+
| 随机数 (32 字节)           |
+---------------------------+
| 会话 ID (变长)             |
+---------------------------+
| 密码套件列表 (变长)         |
+---------------------------+
| 压缩方法                   |
+---------------------------+
| 扩展列表 (变长)             |  ← TrustTunnel 自定义扩展在此
+---------------------------+
```

### TrustTunnel 自定义扩展格式

```
+------------------+------------------+------------------+
| Extension Type   | Extension Length | Extension Data   |
+------------------+------------------+------------------+
| 2 字节           | 2 字节           | 变长             |
+------------------+------------------+------------------+

Extension Data 结构:
+------------------+------------------+------------------+
| 认证令牌          | 协议版本          | 功能标志          |
+------------------+------------------+------------------+
| 变长             | 1 字节           | 2 字节           |
+------------------+------------------+------------------+
```

| 字段 | 说明 |
|------|------|
| `Extension Type` | 自定义扩展类型标识（在 IANA 注册或私有范围） |
| `Extension Length` | 扩展数据长度 |
| `认证令牌` | 经过加密的用户认证凭证 |
| `协议版本` | TrustTunnel 协议版本号 |
| `功能标志` | 位掩码，标识支持的 UDP、多路复用等功能 |

### 握手流程

```
Client                              Server
  |  TLS Client Hello                  |
  |  + TrustTunnel 自定义扩展           |
  |  (认证令牌 + 协议版本 + 功能标志)   |
  |---------------------------------->|
  |                                    | 解析自定义扩展
  |                                    | 验证认证令牌
  |                                    | 检查协议版本兼容
  |                                    |
  |  TLS Server Hello                  |
  |  + TrustTunnel 确认扩展             |
  |<----------------------------------|
  |                                    |
  |  TLS 握手完成                       |
  |  应用数据 (代理协议)                |
```

### 认证令牌

认证令牌通常经过以下处理：

```
认证令牌 = Encrypt(
    key = 服务器公钥,
    data = {
        username: "...",
        password_hash: "...",
        timestamp: 1234567890,
        nonce: random_bytes
    }
)
```

时间戳和 nonce 确保认证令牌不会被重放攻击利用。服务器在收到令牌后验证：
1. 时间戳在可接受的时钟偏差范围内
2. nonce 未被使用过（通过 nonce 缓存）
3. 用户名/密码哈希正确

## 认证机制

### 用户名/密码认证

TrustTunnel 使用用户名和密码进行认证：

```yaml
proxies:
  - name: "trusttunnel-tls"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
```

认证流程：
1. 客户端将用户名和密码组合编码为认证令牌
2. 令牌通过自定义 TLS 扩展在握手阶段发送给服务器
3. 服务器验证令牌有效性
4. 验证通过后完成 TLS 握手，进入代理数据传输阶段

### 认证失败处理

如果认证失败，TrustTunnel 服务器可以选择：
- 完成 TLS 握手但不处理代理数据（表现为正常的 HTTPS 连接）
- 直接终止 TLS 握手
- 返回特定的 TLS Alert 消息

## TLS 模式

### TLS over TCP

TLS 模式下，TrustTunnel 使用标准 TLS 1.2/1.3 进行加密传输：

```yaml
proxies:
  - name: "trusttunnel-tls"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    udp: true
    sni: server.example.com
    skip-cert-verify: false
    fingerprint: chrome
    client-fingerprint: chrome
    alpn:
      - h2
```

TLS 模式的优势：
- 兼容性最好，所有网络环境都支持 TLS
- 可以复用现有的 TLS 中间件和基础设施
- 与 CDN 和反向代理兼容

### TLS 配置选项

| 字段 | 说明 |
|------|------|
| `sni` | TLS 服务器名称指示，用于 TLS 握手 |
| `skip-cert-verify` | 跳过服务器证书验证（不推荐生产使用） |
| `fingerprint` | TLS 证书指纹验证 |
| `client-fingerprint` | TLS Client Hello 指纹（JA3/JA4） |
| `alpn` | 应用层协议协商列表，如 `h2`、`http/1.1` |

## QUIC 模式

### QUIC over UDP

QUIC 模式下，TrustTunnel 使用 QUIC 协议（基于 UDP）进行传输：

```yaml
proxies:
  - name: "trusttunnel-quic"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    quic: true
    congestion-controller: bbr
    cwnd: 32
```

QUIC 的优势：
- **低延迟**：0-RTT 或 1-RTT 连接建立
- **内置多路复用**：无队头阻塞（Head-of-Line Blocking）
- **连接迁移**：IP 切换时连接保持
- **更好的拥塞控制**：用户空间实现，支持 BBR 等现代算法

### QUIC 拥塞控制

| 控制器 | 描述 | 适用场景 |
|--------|------|----------|
| `bbr` | Bottleneck Bandwidth and RTT | 高带宽、长延迟连接 |
| `cubic` | 基于窗口的拥塞控制 | 通用场景 |
| `new_reno` | 经典 TCP 拥塞控制 | 兼容性场景 |
| `pcc` | Performance-oriented Congestion Control | 性能敏感场景 |

### QUIC 连接参数

| 字段 | 类型 | 说明 |
|------|------|------|
| `congestion-controller` | string | 拥塞控制算法 |
| `cwnd` | int | 初始拥塞窗口大小（数据包数） |
| `max_idle_timeout` | duration | 最大空闲超时 |
| `keepalive_period` | duration | Keep-Alive 间隔 |

## 健康检查

TrustTunnel 内置健康检查机制，确保代理连接的可用性：

```yaml
proxies:
  - name: "trusttunnel-health"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    health-check: true
```

健康检查方式：
1. **TLS 握手探测**：定期尝试 TLS 握手，验证服务器可达
2. **HTTP 探测**：通过代理发送 HTTP HEAD 请求到已知可达的域名
3. **延迟测量**：记录健康检查的 RTT，用于延迟排序和故障检测

## UDP 支持

TrustTunnel 支持 UDP 转发，在两种模式下实现方式不同：

### TLS 模式下的 UDP

TLS 本身是面向流的协议，UDP 需要通过隧道封装：

```
UDP Datagram
    ↓
添加序列号和长度前缀
    ↓
通过 TLS 加密通道传输
    ↓
服务端解封装，还原 UDP Datagram
    ↓
转发到目标
```

### QUIC 模式下的 UDP

QUIC 原生支持不可靠数据报（Datagram），UDP 转发更加高效：

```
UDP Datagram
    ↓
封装为 QUIC Datagram
    ↓
通过 QUIC 连接发送
    ↓
服务端提取 QUIC Datagram
    ↓
还原并转发 UDP Datagram
```

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/trusttunnel.go` | TrustTunnel 适配器，配置解析和连接管理 |
| `transport/trusttunnel/trusttunnel.go` | TrustTunnel 协议实现，TLS/QUIC 传输和认证 |
| `transport/trusttunnel/` | 相关辅助模块 |

## 完整配置示例

### TLS 模式

```yaml
proxies:
  - name: "trusttunnel-tls"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    udp: true
```

### QUIC 模式

```yaml
proxies:
  - name: "trusttunnel-quic"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    quic: true
    congestion-controller: bbr
    cwnd: 32
```

### 健康检查

```yaml
proxies:
  - name: "trusttunnel-health"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    health-check: true
```

### 完整配置

```yaml
proxies:
  - name: "trusttunnel-full"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    sni: server.example.com
    skip-cert-verify: false
    fingerprint: chrome
    client-fingerprint: chrome
    alpn:
      - h2
    udp: true
    quic: false
    health-check: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `trusttunnel` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `username` | string | 否 | 用户名 |
| `password` | string | 否 | 密码 |
| `udp` | bool | 否 | 启用 UDP 转发 |
| `quic` | bool | 否 | 使用 QUIC 传输 |
| `health-check` | bool | 否 | 启用健康检查 |
| `sni` | string | 否 | TLS SNI |
| `alpn` | []string | 否 | ALPN 协议列表 |
| `congestion-controller` | string | 否 | QUIC 拥塞控制 |
| `cwnd` | int | 否 | QUIC 初始拥塞窗口 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | TrustTunnel 实现 |
| Pipeline | 完全兼容 | TLS/QUIC 流 |
| Stealth | 完全兼容 | TLS 隐蔽 |
| Multiplex | 完全兼容 | QUIC 内置多路复用 |

## 相关文档

- [[anytls]] - AnyTLS 协议
- [[ech]] - ECH 加密
- [[core/stealth/trusttunnel|trusttunnel]] - TrustTunnel 详细文档
- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
