---
title: "UDP over TCP"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/experimental"
tags: [mihomo, experimental, udp-over-tcp, udp, tcp, 转发]
created: 2026-05-17
updated: 2026-05-17
related: [overview, quic-go]
---

# UDP over TCP

**类别**: Mihomo 实验性功能

## 概述

UDP over TCP 配置允许将 UDP 流量封装在 TCP 连接中转发。当 UDP 连接不稳定或被阻断时，UDP over TCP 提供更可靠的转发方式。

## 基础配置

### 全局配置

```yaml
experimental:
  udp-over-tcp: true
```

### 节点级配置

```yaml
proxies:
  - name: ss-node
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    udp-over-tcp: true
    udp-over-tcp-version: 2
```

### Provider 级配置

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      udp-over-tcp: true
      udp-over-tcp-version: 2
```

## 配置参数

### udp-over-tcp

```yaml
udp-over-tcp: true
```

| 值 | 说明 |
|-----|------|
| true | 启用 UDP over TCP |
| false | 直接转发 UDP（默认） |

### udp-over-tcp-version

```yaml
udp-over-tcp-version: 2
```

版本选择：

| 值 | 说明 | 推荐 |
|-----|------|------|
| 1 | 版本 1 | 较旧实现 |
| 2 | 版本 2 | 推荐 |

## 工作原理

```
UDP over TCP 转发流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端 UDP 流量                             │
│      │                                      │
│      │ UDP 数据包                            │
│      ▼                                      │
│  Mihomo                                     │
│      │                                      │
│      │ 封装为 TCP 消息                       │
│      │                                      │
│      │ TCP 连接                              │
│      ▼                                      │
│  代理服务器                                  │
│      │                                      │
│      │ 解封装 TCP 消息                       │
│      │                                      │
│      │ 发送 UDP 数据包                       │
│      ▼                                      │
│  目标服务器                                  │
│                                             │
└─────────────────────────────────────────────┘
```

## TCP 封装格式

### 版本 1

```
UDP over TCP v1 封装：
┌────────────────────────────────────────────┐
│ Length: 2 bytes                            │
│ UDP Payload: Length bytes                  │
└────────────────────────────────────────────┘
```

### 版本 2

```
UDP over TCP v2 封装：
┌────────────────────────────────────────────┐
│ Atyp: 1 byte                               │
│ Address: 变长                              │
│ Port: 2 bytes                              │
│ Length: 2 bytes                            │
│ UDP Payload: Length bytes                  │
└────────────────────────────────────────────┘
```

版本 2 包含目标地址，更适合某些场景。

## 适用场景

### UDP 不稳定

```yaml
experimental:
  udp-over-tcp: true
```

用途：
- UDP 连接频繁断开
- UDP 丢包严重
- 需要稳定 UDP 转发

### UDP 被阻断

```yaml
udp-over-tcp: true
```

用途：
- UDP 端口被封锁
- UDP 协议被识别阻断
- 仅 TCP 可用

### 游戏/VoIP

UDP over TCP 适用于：

| 应用 | UDP 需求 | 适用性 |
|------|----------|--------|
| 游戏 | 高频 UDP | 可用 |
| VoIP | 实时语音 | 可用 |
| DNS | 低频 UDP | 一般 |
| QUIC | 内置 UDP | 不推荐 |

## 配置示例

### 全局启用

```yaml
experimental:
  udp-over-tcp: true
  quic-go: false
```

### 节点级启用

```yaml
proxies:
  - name: ss-node-1
    type: ss
    server: server1.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    udp-over-tcp: true
    udp-over-tcp-version: 2

  - name: ss-node-2
    type: ss
    server: server2.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    # 不启用 UDP over TCP
```

### Provider 启用

```yaml
proxy-providers:
  udp-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      udp-over-tcp: true
      udp-over-tcp-version: 2
```

### Shadowsocks UDP over TCP

```yaml
proxies:
  - name: ss-udp-tcp
    type: ss
    server: ss.server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    udp-over-tcp: true
    udp-over-tcp-version: 2
```

## 性能考量

UDP over TCP 的性能影响：

| 因素 | 影响 |
|------|------|
| TCP 开销 | 增加帧头 |
| 可靠性 | TCP 重传 |
| 延迟 | 可能增加 |
| 稳定性 | 提高 |

### 延迟对比

| 方式 | 预期延迟 |
|------|----------|
| 直接 UDP | 低 |
| UDP over TCP | 略高 |
| TCP | 相当 |

### 建议配置

根据场景选择：

| 场景 | 建议 |
|------|------|
| UDP 正常 | 不启用 |
| UDP 不稳定 | 启用 v2 |
| UDP 阻断 | 启用 v2 |
| 游戏 | 测试后决定 |

## 与 QUIC 的关系

UDP over TCP 与 QUIC：

- QUIC 本身基于 UDP
- UDP over TCP 可能影响 QUIC 协议
- 不推荐同时用于 QUIC 流量

建议：
- 使用 Hysteria/TUIC 时不启用 UDP over TCP
- 仅对非 QUIC UDP 流量启用

## 排错指南

### UDP 仍不稳定

检查项：

1. udp-over-tcp 是否正确启用
2. 服务端是否支持 UDP over TCP
3. 版本是否匹配

### 连接失败

可能原因：

- 服务端不支持该功能
- 版本不匹配
- TCP 端口未开放

### 性能下降

调优建议：

- 使用版本 2
- 检查 TCP buffer 配置
- 确认服务端带宽

## 相关链接

- [[ref/mihomo/experimental/overview|Experimental 概览]] — 实验性功能总览
- [[ref/mihomo/experimental/quic-go|quic-go]] — QUIC 实现配置
- [[ref/mihomo/provider/override|Override]] — Provider 覆盖配置