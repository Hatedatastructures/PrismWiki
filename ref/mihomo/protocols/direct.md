---
title: Direct
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, direct]
---
# Direct 直连协议

Direct 提供直接网络连接，不经过代理。

## 功能概述

Direct 特性：
- 直接网络连接
- 支持回环检测
- 支持 TCP 和 UDP
- 支持 DNS 解析

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/direct.go` | Direct 适配器 |

## YAML 配置示例

```yaml
proxies:
  - name: "DIRECT"
    type: direct

  - name: "COMPATIBLE"
    type: direct
```

## 使用场景

| 场景 | 说明 |
|------|------|
| 直连规则 | 匹配直连流量 |
| COMPATIBLE | 兼容模式，兜底规则 |

## 回环检测

Direct 内置回环检测机制，防止流量循环：
- 检查 TCP 连接回环
- 检查 UDP 包回环

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Direct 实现 |
| Pipeline | 完全兼容 | L3 协议 |
| Resolver | 完全兼容 | DNS 解析 |

## 相关文档

- [[reject]] - Reject 协议
- [[ref/mihomo/protocols/dns|dns]] - DNS 代理
- [[ref/mihomo/proxy-groups|proxy-groups]] - 代理组配置