---
title: "Prism 兼容性对照"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/compatibility
source: "mihomo-Meta/ vs Prism/"
tags: [mihomo, prism, compatibility, comparison, feature]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/compatibility/overview]]"
  - "[[ref/mihomo/compatibility/features]]"
  - "[[ref/mihomo/compatibility/protocol-matrix]]"
---

# Prism 兼容性对照

**类别**: Mihomo 兼容性 | **对照**: Prism vs mihomo

## 概述

Prism 是模块化设计的代理系统，与 mihomo 有不同的设计理念。本页面对照两者的功能差异和兼容情况。

## 设计理念对比

| 维度 | mihomo | Prism |
|------|--------|-------|
| 架构 | 单体核心 | 模块化系统 |
| 语言 | Go | C++20 |
| 配置 | YAML 单文件 | 模块化配置 |
| 扩展性 | 有限 | 高度可扩展 |
| 性能 | 中等 | 高性能目标 |

## 核心功能对比

### 代理功能

| 功能 | mihomo | Prism | 兼容性 |
|------|--------|-------|--------|
| TCP 代理 | 支持 | 支持 | 完全兼容 |
| UDP 代理 | 支持 | 支持 | 完全兼容 |
| 多路复用 | 支持 | 支持 | 协议兼容 |
| 连接池 | 支持 | 支持 | 完全兼容 |

### DNS 处理

| 功能 | mihomo | Prism | 兼容性 |
|------|--------|-------|--------|
| DNS 模块 | 内置 | 独立 resolve | 实现不同 |
| fake-ip | 原生支持 | 不支持 | 不兼容 |
| DoH/DoT | 支持 | 支持 | 完全兼容 |
| DNS 缓存 | 内置 | 七阶段管道 | 实现不同 |
| DNS 分流 | 支持 | 支持 | 完全兼容 |

### TUN 模式

| 功能 | mihomo | Prism | 兼容性 |
|------|--------|-------|--------|
| TUN 支持 | 内置 | 需外部模块 | 实现不同 |
| gVisor | 支持 | 不支持 | 不兼容 |
| System Stack | 支持 | 支持 | 兼容 |
| 自动路由 | 支持 | 需配置 | 实现不同 |

### 规则系统

| 功能 | mihomo | Prism | 兼容性 |
|------|--------|-------|--------|
| 域名规则 | 支持 | 支持 | 格式兼容 |
| IP 规则 | 支持 | 支持 | 格式兼容 |
| GeoIP | 支持 | 支持 | 数据兼容 |
| GeoSite | 支持 | 支持 | 数据兼容 |
| 进程规则 | 支持 | 支持 | 完全兼容 |
| 规则集 | 支持 | 支持 | 格式兼容 |

## 协议兼容性详细对照

### 完全兼容协议

| 协议 | mihomo 配置 | Prism 配置 | 说明 |
|------|-------------|------------|------|
| SOCKS5 | `type: socks5` | `socks5` | 完全兼容 |
| HTTP | `type: http` | `http` | 完全兼容 |
| Trojan | `type: trojan` | `trojan` | 完全兼容 |
| VLESS | `type: vless` | `vless` | 完全兼容 |
| Shadowsocks | `type: ss` | `ss` | AEAD 兼容 |

### 部分兼容协议

| 协议 | mihomo | Prism | 差异 |
|------|--------|-------|------|
| Hysteria2 | 原生 | 原生 | 配置格式差异 |
| WireGuard | 原生 | 原生 | 实现细节差异 |
| Reality | 原生 | 原生 | 参数差异 |

### 不兼容协议

| 协议 | mihomo | Prism | 原因 |
|------|--------|-------|------|
| VMess | 支持 | 不支持 | Prism 未实现 |
| TUIC | 支持 | 不支持 | Prism 未实现 |
| ShadowsocksR | 支持 | 不支持 | 已弃用 |
| Snell | 支持 | 不支持 | Prism 未实现 |

## 传输层兼容性

| 传输层 | mihomo | Prism | 兼容性 |
|--------|--------|-------|--------|
| WebSocket | 支持 | 支持 | 完全兼容 |
| gRPC/gUN | 支持 | 支持 | 完全兼容 |
| HTTP/2 | 支持 | 支持 | 完全兼容 |
| QUIC | 支持 | 支持 | 完全兼容 |
| Smux | 支持 | 支持 | 协议兼容 |
| Yamux | 支持 | 支持 | 协议兼容 |

## 安全特性对比

| 特性 | mihomo | Prism | 兼容性 |
|------|--------|-------|--------|
| TLS 1.3 | 支持 | 支持 | 完全兼容 |
| Reality | 支持 | 支持 | 完全兼容 |
| ECH | 支持 | 支持 | 完全兼容 |
| Client Hello | 自定义 | 自定义 | 完全兼容 |
| Session Ticket | 支持 | 支持 | 完全兼容 |

## 配置格式对比

### mihomo YAML 格式

```yaml
proxies:
  - name: "node"
    type: trojan
    server: server.com
    port: 443
    password: "pass"
    sni: server.com
```

### Prism 模块化配置

```yaml
# Prism 配置格式（概念）
outbounds:
  - id: node
    protocol: trojan
    endpoint:
      host: server.com
      port: 443
    auth:
      password: pass
    tls:
      sni: server.com
```

## 互操作建议

### 使用 mihomo 服务端 + Prism 客户端

推荐协议:
- Trojan + TLS (完全兼容)
- VLESS + Reality (完全兼容)
- Shadowsocks AEAD (完全兼容)
- Hysteria2 (需确认配置)

### 使用 Prism 服务端 + mihomo 客户端

推荐协议:
- 所有 Prism 支持的协议均可

### 需要避免的协议

- VMess (Prism 不支持)
- TUIC (Prism 不支持)
- ShadowsocksR (已弃用，不推荐)

## 相关链接

- [[overview]] — 兼容性总览
- [[features]] — 功能支持矩阵
- [[protocol-matrix]] — 协议支持矩阵
- [[ref/mihomo/protocols/overview]] — 协议总览