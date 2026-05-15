---
title: "TCP 协议"
category: "network"
type: ref
tags: [网络, tcp, 传输]
created: 2026-05-15
updated: 2026-05-15
---

# TCP 协议

**类别**: 网络

## 概述

TCP（Transmission Control Protocol）是面向连接的、可靠的传输层协议。Prism 主要使用 TCP 进行代理通信。

## 原理

### 特性

- **面向连接**: 三次握手建立连接
- **可靠传输**: 确认、重传、排序
- **流控**: 滑动窗口机制
- **拥塞控制**: 慢启动、拥塞避免

### 连接建立

```
Client                    Server
  |                          |
  |--- SYN ----------------->|
  |<-- SYN+ACK --------------|
  |--- ACK ----------------->|
  |                          |
  |<========= 数据 =========>|
```

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| listener | TCP 监听 | [[agent/front/listener|listener]] |
| connection_pool | TCP 连接池 | [[channel/connection/pool|pool]] |
| duct | TCP 流 | [[multiplex/duct|duct]] |

## 参考资料

- [RFC 793 - Transmission Control Protocol](https://tools.ietf.org/html/rfc793)

## 相关知识
- [[ref/network/udp|UDP]] — UDP 协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞争
