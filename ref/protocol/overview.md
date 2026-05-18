---
title: "协议规范概览"
category: "protocol"
type: ref
layer: ref
module: ref
source: ""
tags: [协议, 规范, 概览, tls, socks5, http, tcp, udp, quic]
created: 2026-05-17
updated: 2026-05-17
---

# 协议规范概览

**类别**: 协议

## 概述

协议规范目录提供网络协议的规范解析，涵盖 TLS 握手、代理协议、传输协议等。每个文档解析协议帧格式、流程规范和关键实现要点。

### 协议分类

协议规范涵盖以下类别：

| 类别 | 内容 | 关键协议 |
|------|------|----------|
| 安全协议 | TLS 1.3 握手 | TLS 1.3 |
| 代理协议 | SOCKS5、HTTP CONNECT | SOCKS5、HTTP |
| 传输协议 | TCP、UDP、QUIC | TCP、UDP、QUIC |
| DNS 协议 | DNS 查询格式 | DNS-over-UDP/TCP/TLS/HTTPS |
| 多路复用 | smux、yamux | smux、yamux |

### 在 Prism 中的应用

Prism 项目中协议的应用：

| 协议 | 应用场景 | 实现模块 |
|------|----------|----------|
| TLS 1.3 | 安全传输、伪装 | [[core/channel/transport/encrypted]], [[core/stealth/overview|stealth]] |
| SOCKS5 | 入站代理协议 | [[core/protocol/socks5|socks5]] |
| HTTP CONNECT | 入站代理协议 | [[core/protocol/http|http]] |
| TCP | 底层传输 | [[core/channel/transport|transport]] |
| UDP | 底层传输 | [[core/channel/transport|transport]] |

## 核心协议

### TLS 1.3

TLS 1.3 是安全协议的核心：

```
TLS 1.3 握手流程（1-RTT）：

Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |          (key_share, supported_groups)   |
  |                                          |
  |<--------- ServerHello -------------------|
  |          (key_share, cipher_suite)       |
  |                                          | ← 计算握手密钥
  |<--------- EncryptedExtensions -----------|
  |<--------- Certificate -------------------|
  |<--------- CertificateVerify -------------|
  |<--------- Finished ----------------------|
  |                                          | ← 计算应用密钥
  |---------- Finished --------------------->|
  |                                          |
  |<========= Application Data ==============>|
```

详见 [[ref/protocol/tls-handshake|TLS 握手流程]]。

### SOCKS5

SOCKS5 是通用代理协议：

```
SOCKS5 协议流程：

1. 握手阶段
   客户端 → 服务端: VER(5) NMETHODS METHODS
   服务端 → 客户端: VER(5) METHOD

2. 认证阶段（如果需要）
   客户端 → 服务端: VER(1) ULEN UNAME PLEN PASSWD
   服务端 → 客户端: VER(1) STATUS

3. 连接请求
   客户端 → 服务端: VER(5) CMD ATYP DST.ADDR DST.PORT
   服务端 → 客户端: VER(5) REP ATYP BND.ADDR BND.PORT

4. 数据传输
   双向转发 TCP 数据
```

详见 [[ref/protocol/socks5-spec|SOCKS5 规范]]。

### HTTP CONNECT

HTTP CONNECT 用于建立隧道：

```
HTTP CONNECT 流程：

客户端 → 服务端:
CONNECT target.example.com:443 HTTP/1.1
Host: target.example.com
Proxy-Authorization: Basic <credentials>

服务端 → 客户端:
HTTP/1.1 200 Connection Established

后续:
客户端 ↔ 服务端: TLS 握手
客户端 ↔ 服务端: 加密数据传输
```

详见 [[ref/protocol/http-proxy-spec|HTTP 代理规范]]。

## 传输协议

### TCP

TCP 是可靠的传输协议：

```
TCP 连接生命周期：

1. 三次握手建立
   客户端 → 服务端: SYN
   服务端 → 客户端: SYN-ACK
   客户端 → 服务端: ACK

2. 数据传输
   可靠传输、流量控制、拥塞控制

3. 四次挥手断开
   客户端 → 服务端: FIN
   服务端 → 客户端: ACK
   服务端 → 客户端: FIN
   客户端 → 服务端: ACK
```

详见 [[ref/protocol/tcp-basics|TCP 基础]]。

### UDP

UDP 是无连接的数据报协议：

```
UDP 特性：

- 无连接：无需建立连接
- 不可靠：可能丢失、乱序
- 数据报：保留消息边界
- 快速：无握手开销

应用场景：
- DNS 查询
- QUIC 底层
- UDP 代理转发
```

详见 [[ref/protocol/udp-basics|UDP 基础]]。

### QUIC

QUIC 是基于 UDP 的可靠传输：

```
QUIC 特性：

- 基于 UDP：无需 TCP 握手
- 内置 TLS 1.3：加密内置
- 连接 ID：支持连接迁移
- 0-RTT：快速恢复连接
- 多路复用：内置流支持

在 Prism 中的潜在应用：
- HTTP/3 支持
- 快速连接恢复
```

详见 [[ref/protocol/quic-basics|QUIC 基础]]。

## DNS 协议

### DNS 查询格式

DNS 查询的标准格式：

```
DNS 查询报文结构：

Header (12 字节):
  ID: 16 位查询 ID
  Flags: QR, OPCODE, AA, TC, RD, RA, Z, RCODE
  QDCOUNT: 问题数
  ANCOUNT: 回答数
  NSCOUNT: 授权数
  ARCOUNT: 附加数

Question Section:
  QNAME: 域名（标签序列）
  QTYPE: 查询类型（A=1, AAAA=28）
  QCLASS: 类（IN=1）

Answer Section: 响应中包含
```

### DNS-over-X

DNS 的多种传输方式：

| 方式 | 传输 | 端口 | 特点 |
|------|------|------|------|
| DNS-over-UDP | UDP | 53 | 传统方式，快速 |
| DNS-over-TCP | TCP | 53 | 大响应，可靠 |
| DNS-over-TLS | TLS | 853 | 加密，隐私 |
| DNS-over-HTTPS | HTTPS | 443 | 加密，穿透 |

详见 [[ref/protocol/dns-over-udp]], [[ref/protocol/dns-over-tls]], [[ref/protocol/dns-over-https]]。

## 子目录索引

| 文件 | 内容 | 链接 |
|------|------|------|
| TLS 握手流程 | TLS 1.3 握手规范 | [[ref/protocol/tls-handshake]] |
| SOCKS5 规范 | RFC 1928 解析 | [[ref/protocol/socks5-spec]] |
| HTTP 代理规范 | HTTP CONNECT 规范 | [[ref/protocol/http-proxy-spec]] |
| TCP 基础 | TCP 协议原理 | [[ref/protocol/tcp-basics]] |
| UDP 基础 | UDP 协议原理 | [[ref/protocol/udp-basics]] |
| QUIC 基础 | QUIC 协议原理 | [[ref/protocol/quic-basics]] |
| TLS 1.3 协议 | TLS 1.3 完整规范 | [[ref/protocol/tls-1.3]] |
| TLS ClientHello | ClientHello 解析 | [[ref/protocol/tls-clienthello]] |
| TLS ServerHello | ServerHello 解析 | [[ref/protocol/tls-serverhello]] |
| TLS 扩展 | TLS 扩展详解 | [[ref/protocol/tls-extensions]] |
| smux/yamux | 多路复用协议 | [[ref/protocol/smux-yamux]] |

## 参见

- [[ref/crypto/overview|密码学原理概览]] — 加密原理
- [[ref/network/overview|网络基础概览]] — 网络原理
- [[core/protocol/analysis|协议分析]] — 协议实现层
- [[ref/overview|参考资料概览]] — 参考资料层索引