---
title: "Happy Eyeballs 算法"
category: "network"
type: ref
tags: [网络, happy-eyeballs, 连接, ipv6]
created: 2026-05-15
updated: 2026-05-15
---

# Happy Eyeballs 算法

**类别**: 网络

## 概述

Happy Eyeballs 是一种连接建立算法，同时尝试 IPv6 和 IPv6 连接，选择先成功的那个。Prism 使用此算法优化连接建立速度。

## 原理

### 算法流程

```
1. 解析域名，获取 IPv6 和 IPv4 地址
2. 启动 IPv6 连接尝试
3. 等待 250ms
4. 如果 IPv6 未成功，启动 IPv4 连接尝试
5. 使用先成功的连接
```

### 优势

- 自动选择最优路径
- 兼容 IPv4 和 IPv6
- 减少连接延迟

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| racer | 连接竞争 | [[channel/eyeball/racer|racer]] |

## 参考资料

- [RFC 8305 - Happy Eyeballs Version 2](https://tools.ietf.org/html/rfc8305)

## 相关知识
- [[ref/network/tcp|TCP]] — TCP 协议
- [[ref/network/connection-pool|连接池]] — 连接池
