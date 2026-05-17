---
title: "嗅探类型"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/component/sniffer"
tags: [mihomo, sniffing, http, tls, quic, dns, stun, 协议检测]
created: 2026-05-17
updated: 2026-05-17
related: [enable, ports]
---

# 嗅探类型

**类别**: Mihomo 嗅探配置

## 概述

Mihomo 支持多种协议的嗅探检测，每种协议使用不同的特征进行识别。本文档详细说明各嗅探类型的配置和原理。

## HTTP 嗅探

### 配置

```yaml
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| ports | []int/string | 嗅探端口范围 |
| override-destination | bool | 是否覆盖目标地址 |

### 检测原理

HTTP 嗅探通过解析 HTTP 请求头中的 Host 字段识别目标：

```http
GET / HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
```

检测流程：

1. 读取连接首包数据
2. 检查是否为 HTTP 方法（GET/POST/PUT/DELETE/HEAD/OPTIONS/CONNECT）
3. 解析 Host 头获取域名
4. 可选覆盖原始目标地址

### override-destination

```yaml
override-destination: true
```

| 值 | 说明 |
|-----|------|
| true | 使用 Host 头覆盖目标地址 |
| false | 仅用于路由，不覆盖目标 |

用途：HTTP 代理场景下修正实际目标。

## TLS 嗅探

### 配置

```yaml
sniffer:
  enable: true
  sniff:
    TLS:
      ports: [443, 8443]
```

### 检测原理

TLS 嗅探通过解析 TLS ClientHello 中的 SNI（Server Name Indication）扩展识别目标：

```
TLS ClientHello 结构：
┌──────────────────────────────────────┐
│ Handshake Type: ClientHello (0x01)   │
├──────────────────────────────────────┤
│ Version: TLS 1.2/1.3                 │
│ Random: 32 bytes                     │
│ Session ID: 0-32 bytes               │
│ Cipher Suites: 变长                  │
│ Compression Methods: 变长            │
│ Extensions:                          │
│   - Server Name (SNI): example.com   │
│   - ALPN: h2, http/1.1              │
│   - ...                              │
└──────────────────────────────────────┘
```

### SNI 提取

从 TLS ClientHello 中提取 SNI：

1. 检查是否为 TLS 握手（0x16 0x03 xx）
2. 解析 ClientHello 消息
3. 查找 Server Name 扩展（类型 0x0000）
4. 提取服务器名称

### TLS 版本支持

| 版本 | 检测支持 |
|------|----------|
| TLS 1.0 | 支持 |
| TLS 1.1 | 支持 |
| TLS 1.2 | 支持 |
| TLS 1.3 | 支持 |

## QUIC 嗅探

### 配置

```yaml
sniffer:
  enable: true
  sniff:
    QUIC:
      ports: [443, 8443]
```

### 检测原理

QUIC 嗅探通过解析 QUIC Initial 包中的加密帧提取 SNI：

```
QUIC Initial 包结构：
┌──────────────────────────────────────┐
│ Flags: 0xCO (Long Header)            │
│ Version: 0x00000001 (QUIC v1)        │
│ DCID Length: 变长                    │
│ DCID: Destination Connection ID      │
│ SCID Length: 变长                    │
│ SCID: Source Connection ID           │
│ Token Length: 变长                   │
│ Token: 可选                           │
│ Payload: 加密的 ClientHello          │
│   (解密后可提取 SNI)                  │
└──────────────────────────────────────┘
```

### QUIC 版本支持

| 版本 | 检测支持 |
|------|----------|
| QUIC v1 (RFC 9000) | 支持 |
| QUIC v2 | 支持 |
| draft-29 | 支持 |

## DNS 嗅探

### 配置

```yaml
sniffer:
  enable: true
  sniff:
    DNS:
      ports: [53]
```

### 检测原理

DNS 嗅探通过解析 DNS 查询包识别：

```
DNS 查询包结构：
┌──────────────────────────────────────┐
│ Transaction ID: 2 bytes              │
│ Flags: 2 bytes                       │
│ Questions: 2 bytes                   │
│ Answer RRs: 2 bytes                  │
│ Authority RRs: 2 bytes               │
│ Additional RRs: 2 bytes              │
│ Queries:                             │
│   - QNAME: example.com               │
│   - QTYPE: A/AAAA/...               │
│   - QCLASS: IN                       │
└──────────────────────────────────────┘
```

### 用途

识别 DNS 流量用于：

- DNS 劫持/转发
- DNS 规则匹配
- DNS over HTTPS/TLS 转换

## STUN 嗅探

### 配置

```yaml
sniffer:
  enable: true
  sniff:
    STUN:
      ports: [3478, 5349]
```

### 检测原理

STUN（Session Traversal Utilities for NAT）嗅探检测 NAT 穿透流量：

```
STUN 消息结构：
┌──────────────────────────────────────┐
│ Message Type: 2 bytes                │
│ Message Length: 2 bytes             │
│ Magic Cookie: 0x2112A444            │
│ Transaction ID: 12 bytes             │
│ Attributes: 变长                     │
└──────────────────────────────────────┘
```

检测 Magic Cookie `0x2112A444` 识别 STUN 流量。

### 用途

- 识别 WebRTC 流量
- NAT 穿透检测
- VoIP 流量识别

## 配置示例

### 常用配置

```yaml
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
```

### 全面嗅探

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 8080-8090, 8888]
      override-destination: true
    TLS:
      ports: [443, 8443, 9443]
    QUIC:
      ports: [443, 8443]
    STUN:
      ports: [3478, 5349]
    DNS:
      ports: [53]
```

### 仅 HTTPS

```yaml
sniffer:
  enable: true
  sniff:
    TLS:
      ports: [443]
    QUIC:
      ports: [443]
```

## 相关链接

- [[ref/mihomo/sniffing/overview|Sniffing 概览]] — 嗅探功能总览
- [[ref/mihomo/sniffing/enable|启用嗅探]] — 启用配置详解
- [[ref/mihomo/sniffing/ports|端口配置]] — 嗅探端口配置
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — TLS 协议分析