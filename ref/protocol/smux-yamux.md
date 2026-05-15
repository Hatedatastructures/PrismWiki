---
title: "SMUX 与 Yamux 多路复用协议"
category: "protocol"
type: ref
tags: [协议, smux, yamux, 多路复用]
created: 2026-05-15
updated: 2026-05-15
---

# SMUX 与 Yamux 多路复用协议

**类别**: 协议

## 概述

SMUX 和 Yamux 都是多路复用协议，允许在单个 TCP 连接上运行多个独立的流。Prism 同时支持这两种协议。

## 原理

### SMUX

SMUX 是轻量级多路复用协议：
- 帧头：12 字节
- 支持流控
- 支持 keepalive

### Yamux

Yamux 是 HashiCorp 开发的多路复用协议：
- 帧头：12 字节
- 支持流控（窗口机制）
- 支持 ping/pong

### 对比

| 特性 | SMUX | Yamux |
|------|------|-------|
| 帧头大小 | 12 字节 | 12 字节 |
| 流控 | 简单 | 窗口机制 |
| keepalive | 支持 | ping/pong |
| 初始窗口 | 固定 | 可配置 |

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| smux::craft | SMUX 帧构造 | [[multiplex/smux/craft|craft]] |
| smux::frame | SMUX 帧处理 | [[multiplex/smux/frame|frame]] |
| yamux::craft | Yamux 帧构造 | [[multiplex/yamux/craft|craft]] |
| yamux::frame | Yamux 帧处理 | [[multiplex/yamux/frame|frame]] |

## 参考资料

- [smux - Simple Multiplexer](https://github.com/xtaci/smux)
- [yamux - Yet Another Multiplexer](https://github.com/hashicorp/yamux)

## 相关知识

- [[ref/network/tcp|TCP]] — 底层传输协议
