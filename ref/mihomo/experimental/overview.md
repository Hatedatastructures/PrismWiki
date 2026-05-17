---
title: "Experimental 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/experimental"
tags: [mihomo, experimental, 实验性功能, quic-go, udp-over-tcp]
created: 2026-05-17
updated: 2026-05-17
related: [quic-go, udp-over-tcp]
---

# Experimental 概览

**类别**: Mihomo 实验性功能参考

## 概述

Experimental（实验性功能）配置块包含 Mihomo 的实验性或高级功能设置。这些功能可能仍在开发中或需要特殊配置。

### 实验性功能列表

| 功能 | 说明 |
|------|------|
| [[ref/mihomo/experimental/quic-go|quic-go]] | QUIC 实现配置 |
| [[ref/mihomo/experimental/udp-over-tcp|udp-over-tcp]] | UDP over TCP 转发 |

## 配置位置

```yaml
experimental:
  quic-go: true
  udp-over-tcp: false
```

## 功能详解

### quic-go

```yaml
quic-go: true
```

启用 quic-go 库的实验性功能。

用途：
- Hysteria 协议
- TUIC 协议
- MASQUE 协议

### udp-over-tcp

```yaml
udp-over-tcp: false
```

全局 UDP over TCP 模式。

用途：
- UDP 不稳定环境
- UDP 被阻断场景
- 提高转发稳定性

## 实验性功能特点

| 特点 | 说明 |
|------|------|
| 配置简单 | 通常为布尔开关 |
| 可能变更 | 功能可能调整 |
| 需测试 | 建议充分测试 |
| 可选启用 | 非必须功能 |

## 使用建议

### quic-go

启用场景：
- 使用 Hysteria/Hysteria2 协议
- 使用 TUIC 协议
- 需要 QUIC 相关功能

禁用场景：
- 不使用 QUIC 协议
- 追求稳定配置

### udp-over-tcp

启用场景：
- UDP 连接不稳定
- UDP 流量被阻断
- 游戏、VoIP 需稳定转发

禁用场景：
- UDP 正常可用
- 不需要特殊处理

## 配置示例

### 基础配置

```yaml
experimental:
  quic-go: true
```

### 完整配置

```yaml
experimental:
  quic-go: true
  udp-over-tcp: false
```

### QUIC 功能启用

```yaml
experimental:
  quic-go: true

proxies:
  - name: hysteria-node
    type: hysteria2
    server: hysteria.server.com
    port: 443
    password: "password"
```

### UDP over TCP 启用

```yaml
experimental:
  udp-over-tcp: true
```

## 相关链接

- [[ref/mihomo/experimental/quic-go|quic-go]] — QUIC 实现配置详解
- [[ref/mihomo/experimental/udp-over-tcp|udp-over-tcp]] — UDP over TCP 配置详解