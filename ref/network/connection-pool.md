---
title: "连接池"
category: "network"
type: ref
tags: [网络, 连接池, 性能]
created: 2026-05-15
updated: 2026-05-15
---

# 连接池

**类别**: 网络

## 概述

连接池是一种缓存和复用网络连接的技术，避免频繁建立和关闭连接的开销。Prism 使用连接池管理上游服务器连接。

## 原理

### 工作流程

```
1. 请求连接时，检查池中是否有空闲连接
2. 如果有，复用空闲连接
3. 如果没有，建立新连接
4. 连接使用完毕后，归还到池中
5. 空闲连接超时后自动关闭
```

### 配置参数

- **max_cache_per_endpoint**: 每个端点的最大缓存连接数
- **max_idle_seconds**: 空闲连接超时时间
- **connect_timeout_ms**: 连接超时时间

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| pool | 连接池 | [[channel/connection/pool|pool]] |

## 参考资料

- [Connection Pool](https://en.wikipedia.org/wiki/Connection_pool)

## 相关知识
- [[ref/network/tcp|TCP]] — TCP 协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞争
