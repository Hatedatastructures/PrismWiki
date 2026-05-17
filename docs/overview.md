---
title: Prism 概述
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, overview]
---

# Prism 概述

Prism 是一款高性能代理引擎，支持多协议和多路复用，为现代网络而生。

## 功能简介

- **多协议支持**: SOCKS5、HTTP、Trojan、VLESS、Shadowsocks 2022
- **多路复用**: Smux、Yamux 单连接多流传输
- **伪装技术**: Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel
- **连接池**: 高效复用，降低延迟
- **DNS 缓存**: 快速解析，支持多种协议

## 如何启用

下载或编译 Prism 后，配置 `configuration.json` 即可启动服务。

```bash
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release
./build_release/src/Prism
```

## 常见问题

**Q: Prism 是什么？**
A: 一款支持多协议和多路复用的高性能代理引擎。

## 相关页面

- [[dev/configuration|配置详解]]
- [[dev/modules|模块架构]]
- [[performance/benchmark|性能基准]]