---
title: 配置说明
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 配置说明

Prism 配置文件结构简览，快速了解各模块功能。

## 功能简介

配置文件 `configuration.json` 分为 8 大模块：agent、pool、buffer、protocol、multiplex、stealth、dns、trace。

## 如何启用

程序启动时自动加载 `configuration.json`，或通过 `--config <path>` 指定路径。

## 配置模块速览

| 模块 | 功能 |
|------|------|
| `agent` | 监听地址、端口、证书、认证 |
| `pool` | 连接池缓存、超时、缓冲区 |
| `protocol` | SOCKS5/Trojan/VLESS/Shadowsocks 协议开关 |
| `multiplex` | Smux/Yamux 多路复用参数 |
| `stealth` | Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel 伪装 |
| `dns` | DNS 服务器、缓存、解析策略 |
| `trace` | 日志级别、路径、轮转 |

## 常见问题

**Q: 如何修改监听端口？**
A: 修改 `agent.addressable.port`。

**Q: 如何启用调试日志？**
A: 设置 `trace.log_level = "debug"`。

## 相关页面

- [[dev/configuration|配置详解]]
- [[agent/worker/worker|Worker 配置]]
- [[dev/testing|测试配置]]