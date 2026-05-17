---
title: SOCKS4
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, socks4]
---
# SOCKS4 协议

SOCKS4 是 SOCKS 协议的早期版本，仅支持 TCP 连接，不支持认证。

## 协议概述

SOCKS4 特性：
- 仅支持 TCP 代理
- 支持 SOCKS4a 域名传递
- 支持用户 ID（非密码认证）
- 不支持 IPv6

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/base.go` | SOCKS4 通过基础适配器处理 |
| `transport/socks4/socks4.go` | SOCKS4 协议实现 |

## YAML 配置示例

```yaml
proxies:
  - name: "socks4-proxy"
    type: socks4
    server: server.example.com
    port: 1080
```

## SOCKS4a 扩展

SOCKS4a 允许传递域名而非 IP：
- 目标 IP 设为 `0.0.0.x`（x > 0）
- 域名追加在请求末尾

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | 基础 TCP 代理 |
| Pipeline | 完全兼容 | 仅 TCP 流 |
| UDP 支持 | 不兼容 | SOCKS4 不支持 UDP |

## 相关文档

- [[socks5]] - SOCKS5 协议
- [[../transport/socks4]] - SOCKS4 传输层
- [[../../protocol/socks5]] - SOCKS 协议家族