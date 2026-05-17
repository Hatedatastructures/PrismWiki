---
title: DNS
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, dns]
---
# DNS 代理协议

DNS 代理用于处理 DNS 查询请求。

## 功能概述

DNS 代理特性：
- DNS 查询劫持
- 支持 TCP DNS
- 支持 UDP DNS
- DNS-over-HTTPS 支持

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/dns.go` | DNS 适配器 |

## YAML 配置示例

```yaml
proxies:
  - name: "DNS"
    type: dns
```

## 使用场景

| 场景 | 说明 |
|------|------|
| DNS 劫持 | 将 DNS 查询重定向 |
| DNS-over-HTTPS | 通过 HTTPS 进行 DNS |

## 工作机制

DNS 代理处理流程：
1. 接收 DNS 查询请求
2. 通过本地 DNS resolver 处理
3. 返回 DNS 响应

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | DNS 代理实现 |
| Resolver | 完全兼容 | DNS 解析集成 |
| Rules | 完全兼容 | DNS 规则匹配 |

## 相关文档

- [[direct]] - Direct 协议
- [[reject]] - Reject 协议
- [[../../client/mihomo-dns]] - DNS 配置
- [[../../resolve/dns]] - DNS 解析