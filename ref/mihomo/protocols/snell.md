---
title: Snell
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, snell]
---
# Snell 协议

Snell 是一种轻量级代理协议，专为高速代理设计。

## 协议概述

Snell 特性：
- 轻量级设计
- 支持 TCP 和 UDP（v3+）
- 支持混淆（TLS/HTTP）
- 版本兼容性：v1、v2、v3

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/snell.go` | Snell 适配器 |
| `transport/snell/snell.go` | Snell 协议实现 |
| `transport/snell/cipher.go` | Snell 加密 |

## YAML 配置示例

### Snell v3 配置

```yaml
proxies:
  - name: "snell-v3"
    type: snell
    server: server.example.com
    port: 443
    psk: your-psk
    version: 3
    udp: true
```

### Snell v2 配置

```yaml
proxies:
  - name: "snell-v2"
    type: snell
    server: server.example.com
    port: 443
    psk: your-psk
    version: 2
```

### TLS 混淆

```yaml
proxies:
  - name: "snell-obfs-tls"
    type: snell
    server: server.example.com
    port: 443
    psk: your-psk
    version: 3
    obfs-opts:
      mode: tls
      host: bing.com
```

### HTTP 混淆

```yaml
proxies:
  - name: "snell-obfs-http"
    type: snell
    server: server.example.com
    port: 443
    psk: your-psk
    version: 3
    obfs-opts:
      mode: http
      host: bing.com
```

## 版本差异

| Version | UDP | 描述 |
|---------|-----|------|
| v1 | 不支持 | 原始版本 |
| v2 | 不支持 | 连接池优化 |
| v3 | 支持 | UDP over TCP |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `snell` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `psk` | string | 是 | 预共享密钥 |
| `version` | int | 否 | Snell 版本（默认 3） |
| `udp` | bool | 否 | 启用 UDP（v3+） |
| `obfs-opts` | map | 否 | 混淆配置 |

## 协议背景

Snell 是由 `surge-team` 开发的轻量级代理协议，最初为 Surge 网络工具设计。它以简洁高效著称，协议开销极小，适合对性能要求高的场景。

### 协议设计理念

- **极简主义**：协议头部极小，减少额外开销
- **高速代理**：专注于 TCP 转发性能
- **版本迭代**：v1 → v2 → v3，逐步增加功能
- **混淆支持**：可选的 TLS/HTTP 混淆

### 版本演进

| 版本 | 年份 | 新增特性 |
|------|------|---------|
| v1 | 2018 | 基础 TCP 代理 |
| v2 | 2019 | 连接池、复用、命令协议 |
| v3 | 2020 | UDP 支持（UDP over TCP）、更安全的加密 |

## 帧格式详解

### Snell v1/v2 数据包格式

```
┌─────────────────────────────────────────────┐
│              Snell v1/v2 帧格式                │
├─────────────────────────────────────────────┤
│                                             │
│  +--------+--------+--------+------+------┐  │
│  | PSK    | CMD    | ATYP   | ADDR | PORT │  │
│  | (SHA256| (1B)   | (1B)   | (Var)| (2B) │  │
│  | 前16B)  |        |        |      |      │  │
│  +--------+--------+--------+------+------┘  │
│                                             │
│  PSK: 预共享密钥的 SHA256 前 16 字节           │
│  CMD: 0x01 = CONNECT, 0x02 = SHUTDOWN       │
│  ATYP: 0x00 = IPv4, 0x01 = 域名, 0x02 = IPv6│
│  ADDR: 目标地址（IPv4=4B, IPv6=16B, 域名=长度+内容）│
│  PORT: 目标端口（大端序）                      │
│                                             │
└─────────────────────────────────────────────┘
```

### Snell v3 数据包格式

v3 引入了更安全的加密方案：

```
┌─────────────────────────────────────────────┐
│              Snell v3 帧格式                   │
├─────────────────────────────────────────────┤
│                                             │
│  阶段一：握手                                 │
│  +--------+--------+                         │
│  | VERSION| NONCE  |                         │
│  | (1B)   | (15B)  |                         │
│  +--------+--------+                         │
│                                             │
│  阶段二：加密数据（使用 ChaCha20-Poly1305）     │
│  +--------+--------+--------+------+------┐  │
│  | CMD    | ATYP   | ADDR   | PORT | TAG  │  │
│  | (1B)   | (1B)   | (Var)  | (2B) | (16B)│  │
│  +--------+--------+--------+------+------┘  │
│                                             │
│  阶段三：UDP 数据包（可选）                     │
│  +--------+--------+------+                │
│  | PKT_ID | DATA   | TAG  |                │
│  | (2B)   | (Var)  | (16B)│                │
│  +--------+--------+------+                │
│                                             │
└─────────────────────────────────────────────┘
```

### v3 握手流程

```
┌──────────────────────────────────────────┐
│          Snell v3 握手流程                  │
├──────────────────────────────────────────┤
│                                          │
│  Client                    Server        │
│    |                         |            │
│    |  --- [VERSION][NONCE] ->|            │
│    |                         |            │
│    |  [双方计算共享密钥]       |            │
│    |  (PSK + NONCE → Key)    |            │
│    |                         |            │
│    |  <-- [Encrypted CMD] -- |            │
│    |      (CONNECT + ADDR)    |            │
│    |                         |            │
│    |  [数据隧道建立]           |            │
│    |  ←── 双向加密流 ──→      |            │
│    |                         |            │
│                                          │
└──────────────────────────────────────────┘
```

密钥派生：
```
Key = HKDF(PSK, NONCE)
```

## 认证机制

### PSK（Pre-Shared Key）认证

所有 Snell 版本都使用 PSK 进行认证。PSK 是一个客户端和服务器共享的密钥字符串。

```yaml
psk: your-psk
```

认证方式因版本而异：

| 版本 | 认证方式 |
|------|---------|
| v1 | PSK 的 SHA256 前 16 字节直接放在请求头部 |
| v2 | 同 v1，但增加了连接复用 |
| v3 | PSK + 随机 Nonce 派生会话密钥，全加密通信 |

### v3 安全增强

v3 相比 v1/v2 的安全性提升：

1. **全加密**：握手后所有数据都经过加密
2. **防重放**：Nonce 防止数据包重放
3. **AEAD**：使用 ChaCha20-Poly1305 提供完整性验证
4. **前向保密**：每次连接使用不同的 Nonce

## 混淆机制

### TLS 混淆

```yaml
obfs-opts:
  mode: tls
  host: bing.com
```

TLS 混淆模拟 TLS 握手的 Client Hello 消息，使流量看起来像正常的 HTTPS 连接：

```
[伪造的 TLS Client Hello]
  ├─ TLS Record Layer (Content Type = 0x16)
  ├─ TLS Handshake (Type = 0x01, Client Hello)
  ├─ TLS Version (0x0301 = TLS 1.0)
  ├─ Random
  ├─ Session ID
  ├─ Cipher Suites
  └─ Extensions (SNI, ALPN, etc.)
[Snell 加密数据]
```

### HTTP 混淆

```yaml
obfs-opts:
  mode: http
  host: bing.com
```

HTTP 混淆模拟 HTTP GET 请求：

```
GET / HTTP/1.1\r\n
Host: bing.com\r\n
User-Agent: Mozilla/5.0\r\n
\r\n
[Snell 加密数据]
```

## 连接池（v2+）

Snell v2 引入了连接池机制：

```
┌────────────────────────────────────────┐
│           Snell v2 连接池                │
├────────────────────────────────────────┤
│                                        │
│  [已建立的 TCP 连接]                     │
│  ├─ 连接 1 (空闲)                       │
│  ├─ 连接 2 (正在传输)                    │
│  └─ 连接 3 (空闲)                       │
│                                        │
│  新请求 → 选择空闲连接 → 复用            │
│                                        │
│  优势：                                  │
│  - 减少 TCP 握手延迟                    │
│  - 提高高并发场景性能                     │
│  - 降低服务器连接数                      │
│                                        │
└────────────────────────────────────────┘
```

## UDP 支持（v3）

Snell v3 通过 UDP over TCP 实现 UDP 代理：

```
UDP 数据包封装：
+--------+--------+--------+------+======+
| PKT_ID | Length | ATYP   | ADDR | DATA |
| (2B)   | (2B)   | (1B)   | (Var)|      |
+--------+--------+--------+------+======+
```

- **PKT_ID**：唯一标识每个 UDP 数据包
- 在同一个 TCP 连接上复用，通过 PKT_ID 区分不同 UDP 会话

## 错误处理

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| PSK 不匹配 | 密钥错误 | 服务器断开连接 |
| 版本不兼容 | 客户端/服务器版本不匹配 | 连接失败 |
| 混淆失败 | obfs 配置错误 | 回退到明文 |
| UDP 包丢失 | TCP 连接断开 | 上层重传 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Snell 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | v2 连接池 |

## 相关文档

- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
- [[../transport/snell]] - Snell 传输层