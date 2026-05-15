---
title: "UDP 协议"
category: "network"
type: ref
tags: [网络, udp, 传输]
created: 2026-05-15
updated: 2026-05-15
---

# UDP 协议

**类别**: 网络

## 概述

UDP（User Datagram Protocol）是无连接的、不可靠的传输层协议。Prism 使用 UDP 进行 DNS 查询和 UDP 代理。

## 原理

### 特性

- **无连接**: 不需要建立连接
- **不可靠**: 不保证送达、不保证顺序
- **低延迟**: 没有握手和确认开销
- **支持广播**: 支持一对多通信

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| parcel | UDP 包 | [[multiplex/parcel|parcel]] |
| udp_relay | UDP 中继 | [[protocol/common/udp_relay|udp_relay]] |
| upstream | DNS 查询 | [[resolve/dns/upstream|upstream]] |

## 参考资料

- [RFC 768 - User Datagram Protocol](https://tools.ietf.org/html/rfc768)

## 相关知识
- [[ref/network/tcp|TCP]] — TCP 协议
