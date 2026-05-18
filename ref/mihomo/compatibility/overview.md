---
title: "mihomo 兼容性参考"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/compatibility
source: "mihomo-Meta/"
tags: [mihomo, compatibility, prism, features, protocol, transport]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/compatibility/prism]]"
  - "[[ref/mihomo/compatibility/features]]"
  - "[[ref/mihomo/compatibility/protocol-matrix]]"
  - "[[ref/mihomo/compatibility/transport-matrix]]"
---

# mihomo 兼容性参考

**类别**: Mihomo 兼容性 | **模块**: 功能与协议支持矩阵

## 概述

mihomo（原 Clash.Meta）是功能最丰富的代理核心之一，支持广泛的代理协议、传输层和安全特性。本章节提供 mihomo 与 Prism 及其他实现的兼容性对照。

### 核心特性

| 特性 | mihomo | 说明 |
|------|--------|------|
| 协议支持 | 丰富 | 支持 20+ 代理协议 |
| 传输层 | 多样 | WebSocket/gRPC/QUIC 等 |
| 安全特性 | 完备 | TLS/Reality/ECH 等 |
| TUN 模式 | 支持 | gVisor/System/Mixed |
| DNS 处理 | fake-ip | 防泄露效果好 |

## 子页面导航

| 页面 | 描述 |
|------|------|
| [[prism]] | Prism 兼容性对照 |
| [[features]] | 功能支持矩阵 |
| [[protocol-matrix]] | 协议支持矩阵 |
| [[transport-matrix]] | 传输层支持矩阵 |

## 快速对比

### mihomo vs Prism

| 维度 | mihomo | Prism |
|------|--------|-------|
| 设计理念 | 单体核心 | 模块化设计 |
| 协议支持 | 20+ 协议 | 精选协议 |
| fake-ip | 原生支持 | 不支持 |
| TUN 模式 | 原生支持 | 独立模块 |
| DNS 处理 | 内置 | 七阶段管道 |
| 配置格式 | YAML | 模块化配置 |

### 协议支持对比

| 协议 | mihomo | Prism |
|------|--------|-------|
| SOCKS5 | 支持 | 支持 |
| HTTP | 支持 | 支持 |
| Trojan | 支持 | 支持 |
| VLESS | 支持 | 支持 |
| VMess | 支持 | 不支持 |
| Shadowsocks | 支持 | 支持 |
| Hysteria2 | 支持 | 支持 |
| TUIC | 支持 | 不支持 |
| WireGuard | 支持 | 支持 |

### 传输层对比

| 传输层 | mihomo | Prism |
|--------|--------|-------|
| WebSocket | 支持 | 支持 |
| gRPC/gUN | 支持 | 支持 |
| HTTP/2 | 支持 | 支持 |
| QUIC | 支持 | 支持 |
| Smux | 支持 | 支持 |
| Yamux | 支持 | 支持 |

## 兼容性说明

### 协议兼容

mihomo 支持的协议大多数与 Prism 兼容:
- 完全兼容: SOCKS5, HTTP, Trojan, VLESS, Shadowsocks
- 部分兼容: Hysteria2 (实现差异)
- 不兼容: VMess, TUIC (Prism 不支持)

### 配置兼容

mihomo 配置格式与 Prism 不同:
- mihomo: YAML 单文件配置
- Prism: 模块化配置，支持多文件

### 功能兼容

核心代理功能兼容，但实现方式不同:
- TUN: mihomo 内置，Prism 需外部模块
- DNS: mihomo fake-ip，Prism 独立 resolve
- 规则: 两者都支持，格式略有差异

## 源码位置

- 协议实现: `adapter/outbound/*.go`
- 传输层: `transport/*/`
- 配置解析: `config/*.go`

## 相关链接

- [[ref/mihomo/protocols/overview]] — 协议总览
- [[ref/mihomo/transport/overview]] — 传输层总览
- [[ref/mihomo/config/overview]] — 配置总览
- [[ref/mihomo/index|mihomo]] — mihomo 概述