---
title: "NAT 穿透"
category: "network"
type: ref
tags: [网络, nat, 穿透]
created: 2026-05-15
updated: 2026-05-15
---

# NAT 穿透

**类别**: 网络

## 概述

NAT 穿透是使位于 NAT 后面的设备能够直接通信的技术。在代理场景中，NAT 穿透用于建立 P2P 连接。

## 原理

### NAT 类型

| 类型 | 说明 |
|------|------|
| 完全锥形 | 任何外部主机都可以连接 |
| 受限锥形 | 只有曾经通信过的主机可以连接 |
| 端口受限锥形 | 只有相同端口的主机可以连接 |
| 对称 | 每个目的地址使用不同的映射 |

### 穿透技术

- **STUN**: 获取 NAT 映射后的地址
- **TURN**: 通过中继服务器转发
- **ICE**: 综合使用多种穿透技术

## 在 Prism 中的应用

Prism 目前不直接使用 NAT 穿透技术，但理解 NAT 穿透有助于设计代理协议。

| 概念 | 说明 | 链接 |
|------|------|------|
| UDP 中继 | Prism 支持 UDP 代理，可能涉及 NAT 穿透 | [[protocol/common/udp_relay|udp_relay]] |

## 参考资料

- [RFC 8489 - STUN](https://tools.ietf.org/html/rfc8489)
- [RFC 8656 - TURN](https://tools.ietf.org/html/rfc8656)

## 相关知识
- [[ref/network/udp|UDP]] — UDP 协议
