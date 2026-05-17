---
title: Mieru
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, mieru]
---
# Mieru 协议

Mieru 是一种新兴的代理协议，支持 TCP 和 UDP。

## 协议概述

Mieru 特性：
- 支持 TCP 和 UDP 传输
- 支持端口范围
- 支持多路复用
- 支持 Traffic Pattern

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/mieru.go` | Mieru 适配器 |

## YAML 配置示例

### TCP 模式

```yaml
proxies:
  - name: "mieru-tcp"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
```

### UDP 模式

```yaml
proxies:
  - name: "mieru-udp"
    type: mieru
    server: server.example.com
    port-range: 10000-20000
    transport: UDP
    username: user
    password: pass
```

### 多路复用

```yaml
proxies:
  - name: "mieru-mux"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    multiplexing: HIGH
```

### Traffic Pattern

```yaml
proxies:
  - name: "mieru-pattern"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    traffic-pattern: "pattern-string"
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `mieru` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口 |
| `port-range` | string | 否 | 端口范围 |
| `transport` | string | 是 | TCP 或 UDP |
| `username` | string | 是 | 用户名 |
| `password` | string | 是 | 密码 |
| `multiplexing` | string | 否 | 多路复用级别 |
| `handshake-mode` | string | 否 | 握手模式 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Mieru 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 内置多路复用 |

## 相关文档

- [[../../protocol/proxy-protocols]] - 代理协议概述