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

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Snell 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | v2 连接池 |

## 相关文档

- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
- [[../transport/snell]] - Snell 传输层