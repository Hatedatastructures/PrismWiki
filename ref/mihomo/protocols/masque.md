---
title: MASQUE
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, masque]
---
# MASQUE 协议

MASQUE（Multiplexed Application Substrate over QUIC Encryption）是一组 IETF 标准化协议，基于 HTTP/3（QUIC）实现 IP-in-IP 隧道和 UDP 代理功能。它允许客户端通过 HTTP/3 连接建立 IP 层隧道，将任意 IP 数据包封装在 HTTP/3 流中传输，从而实现网络层的代理。

## 协议概述

MASQUE 的核心特性：

- **IP-in-IP over HTTP/3**：将 IP 数据包封装在 HTTP/3/QUIC 连接中
- **Connect-IP 扩展**：使用 HTTP CONNECT-IP 方法建立 IP 隧道
- **UDP 代理机制**：通过 HTTP Datagram 扩展代理 UDP 流量
- **远程 DNS 解析**：支持通过隧道进行 DNS 查询
- **ECDSA 认证**：使用 ECDSA 密钥对进行客户端-服务器认证
- **拥塞控制**：继承 QUIC 的拥塞控制机制（BBR、Cubic 等）
- **标准化协议**：IETF 标准（RFC 9298、RFC 9221），非私有协议

MASQUE 是 IETF 标准化的代理协议，与 WireGuard 类似提供 IP 层隧道能力，但传输层使用 HTTP/3 而非 UDP 原生协议。这使得 MASQUE 流量在外观上完全等同于正常的 HTTPS 流量。

## IP-in-IP over HTTP/3

### 原理

MASQUE 将 IP 数据包封装在 HTTP/3 的 QUIC 连接中，形成 IP-in-QUIC-in-UDP 的封装结构：

```
原始 IP 数据包
    ↓
封装为 IP-in-IP（隧道层）
    ↓
封装为 HTTP Datagram（HTTP/3 层）
    ↓
封装为 QUIC Stream/Datagram（QUIC 层）
    ↓
封装为 UDP 数据包（传输层）
```

封装格式：

```
+---------------------------------------------+
| UDP Header                                   |
+---------------------------------------------+
| QUIC Header                                  |
+---------------------------------------------+
| HTTP/3 Frame (Datagram or Stream)            |
+---------------------------------------------+
| MASQUE Context                                 |
+---------------------------------------------+
| 内层 IP Header                               |
+---------------------------------------------+
| 内层 IP Payload (TCP/UDP/ICMP)                |
+---------------------------------------------+
```

### Connect-IP 扩展

MASQUE 使用 HTTP 的 CONNECT 方法扩展来建立 IP 隧道：

```
HTTP 请求:
CONNECT ip/2001:db8::1/48 HTTP/3
Host: masque.example.com
Protocol: connect-ip
Capsule-Protocol: 1

HTTP 响应:
HTTP/3 200 OK
Capsule-Protocol: 1
```

`connect-ip` 协议标识符告诉服务器这是一个 MASQUE IP 隧道请求，而非普通的 HTTP CONNECT 代理请求。

### 隧道建立流程

```
Client                              MASQUE Server
  |  QUIC 连接建立 (0-RTT/1-RTT)     |
  |-------------------------------->|
  |                                  |
  |  HTTP CONNECT-IP 请求             |
  |  (指定 IP 前缀/路由)              |
  |-------------------------------->|
  |                                  | 验证 ECDSA 签名
  |                                  | 分配隧道端点
  |  HTTP 200 OK                     |
  |  (确认隧道建立)                   |
  |<--------------------------------|
  |                                  |
  |  IP 数据包 (通过 QUIC Datagram)   |
  |-------------------------------->|
  |                                  | 解封装 IP 包
  |                                  | 路由到目标
  |  IP 响应数据包                    |
  |<--------------------------------|
  |  解封装                          |
  |  交付给本地协议栈                 |
```

### 地址分配

MASQUE 隧道中，客户端需要配置本地 IP 地址：

```yaml
proxies:
  - name: "masque-proxy"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2          # 客户端隧道 IP
```

| 字段 | 说明 |
|------|------|
| `ip` | 客户端在隧道中的 IPv4 地址（通常是 /32 或 /24 子网内） |
| `ipv6` | 客户端在隧道中的 IPv6 地址（可选） |

服务端根据客户端的公钥和配置的 IP 地址池为客户端分配隧道 IP。

## UDP 代理机制

### HTTP Datagram 扩展

MASQUE 通过 HTTP Datagram 扩展（RFC 9221）实现 UDP 代理。HTTP Datagram 允许在 HTTP/3 连接中传输无格式的二进制数据：

```
QUIC Datagram Frame:
+------------------+------------------+------------------+
| Stream ID (QPACK)| Data Length      | Datagram Data    |
+------------------+------------------+------------------+
| 变长             | 变长             | 变长             |
+------------------+------------------+------------------+
```

UDP 代理流程：

```
本地应用 UDP 请求
    ↓
封装为 MASQUE UDP Datagram
    ↓
通过 QUIC Datagram 发送
    ↓
MASQUE 服务器解封装
    ↓
发送 UDP 到目标
    ↓
收到 UDP 响应
    ↓
封装为 MASQUE UDP Datagram
    ↓
通过 QUIC Datagram 返回
    ↓
客户端解封装
    ↓
返回给本地应用
```

### 与 IP-in-IP 的区别

| 特性 | IP-in-IP 隧道 | UDP 代理 |
|------|--------------|----------|
| 传输内容 | 任意 IP 数据包 | 仅 UDP 数据报 |
| 封装方式 | IP-in-IP + HTTP Datagram | MASQUE UDP Datagram |
| 适用场景 | 全局代理（所有协议） | 仅 UDP 流量 |
| 性能 | 有 IP 封装开销 | 更轻量 |
| 配置 | 需要配置隧道 IP | 无需额外 IP |

mihomo 的 MASQUE 实现同时支持两种模式。

## 远程 DNS 解析

MASQUE 支持通过隧道进行远程 DNS 解析：

```yaml
proxies:
  - name: "masque-dns"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    remote-dns-resolve: true
    dns:
      - 1.1.1.1
```

远程 DNS 解析的优势：
1. **防止 DNS 污染**：DNS 查询通过加密隧道传输，中间人无法篡改
2. **地理 DNS**：使用代理服务器所在地区的 DNS，获取正确的 CDN 解析
3. **统一解析**：所有 DNS 查询经过同一隧道，便于管理

DNS 查询通过 IP 隧道发送到配置的 DNS 服务器：

```
本地 DNS 查询 (example.com)
    ↓
封装到 MASQUE 隧道
    ↓
通过 HTTP/3 发送到 MASQUE 服务器
    ↓
服务器在远端向 1.1.1.1 发起 DNS 查询
    ↓
获取 DNS 响应
    ↓
通过隧道返回给客户端
```

## ECDSA 认证

MASQUE 使用 ECDSA（椭圆曲线数字签名算法）进行客户端认证：

```
认证流程:
1. 客户端使用私钥对连接请求签名
2. 服务器使用客户端公钥验证签名
3. 验证通过后建立隧道
```

```yaml
proxies:
  - name: "masque-proxy"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key   # 客户端 ECDSA 私钥 (Base64)
    public-key: server-public-key   # 服务器 ECDSA 公钥 (Base64)
    ip: 10.0.0.2
```

| 字段 | 说明 |
|------|------|
| `private-key` | 客户端 ECDSA P-256 私钥，Base64 编码 |
| `public-key` | 服务器 ECDSA P-256 公钥，Base64 编码 |

ECDSA P-256 的优势：
- 密钥短（256 位 = 32 字节私钥，64 字节公钥）
- 签名小（64 字节）
- 高性能（比 RSA 快得多）
- 标准化（NIST P-256 曲线）

## 拥塞控制

MASQUE 继承 QUIC 的拥塞控制机制，支持多种算法：

```yaml
proxies:
  - name: "masque-cc"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    congestion-controller: bbr
    cwnd: 32
```

| 控制器 | 描述 | 适用场景 |
|--------|------|----------|
| `bbr` | Bottleneck Bandwidth and RTT | 高带宽、长延迟（推荐） |
| `cubic` | 基于窗口的拥塞控制 | 通用场景 |
| `new_reno` | 经典 TCP 风格 | 兼容性场景 |
| `pcc` | 性能导向拥塞控制 | 延迟敏感场景 |

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/masque.go` | MASQUE 适配器，配置解析和出站集成 |
| `transport/masque/masque.go` | MASQUE 协议实现，Connect-IP 隧道和 Datagram |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "masque-proxy"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
```

### IPv6 配置

```yaml
proxies:
  - name: "masque-ipv6"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    ipv6: fd00::2
```

### 远程 DNS

```yaml
proxies:
  - name: "masque-dns"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    remote-dns-resolve: true
    dns:
      - 1.1.1.1
```

### 拥塞控制

```yaml
proxies:
  - name: "masque-cc"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    congestion-controller: bbr
    cwnd: 32
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `masque` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `private-key` | string | 是 | ECDSA 私钥（Base64） |
| `public-key` | string | 是 | 服务器公钥（Base64） |
| `ip` | string | 是 | 本地 IPv4 地址 |
| `ipv6` | string | 否 | 本地 IPv6 地址 |
| `mtu` | int | 否 | MTU（默认 1280） |
| `remote-dns-resolve` | bool | 否 | 启用远程 DNS |
| `dns` | []string | 否 | 远程 DNS 服务器列表 |
| `congestion-controller` | string | 否 | 拥塞控制算法 |
| `cwnd` | int | 否 | 初始拥塞窗口 |

## 与 WireGuard 的对比

| 特性 | MASQUE | WireGuard |
|------|--------|-----------|
| 传输层 | HTTP/3 (QUIC over UDP) | 原生 UDP |
| 外观 | 完全等同 HTTPS | 独特的 WireGuard 握手 |
| 标准 | IETF 标准 | 独立协议 |
| NAT 穿越 | 优秀（QUIC 内置） | 优秀 |
| 防火墙穿透 | 极佳（HTTPS 外观） | 良好（可能被 DPI 识别） |
| 端口 | 443（标准 HTTPS） | 自定义 UDP 端口 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | MASQUE 实现 |
| Pipeline | 完全兼容 | L3 协议 |
| Multiplex | 完全兼容 | HTTP/3 多路复用 |
| Resolver | 完全兼容 | 远程 DNS 集成 |

## 相关文档

- [[wireguard]] - WireGuard 协议
- [[dev/debugging/tls|tls]] - TLS/QUIC
- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
