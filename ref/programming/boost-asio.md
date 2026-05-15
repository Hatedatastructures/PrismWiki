---
title: "Boost.Asio"
category: "programming"
type: ref
tags: [编程, boost, asio, 异步]
created: 2026-05-15
updated: 2026-05-15
---

# Boost.Asio

**类别**: 编程

## 概述

Boost.Asio 是跨平台的异步 I/O 框架，提供事件循环、异步操作、协程支持。Prism 使用 Boost.Asio 作为核心 I/O 框架。

## 原理

### 核心概念

- **io_context**: 事件循环
- **executor**: 执行器
- **strand**: 串行执行
- **socket**: 网络套接字

### 协程支持

Boost.Asio 原生支持 C++20 协程：
```cpp
net::awaitable<void> async_operation() {
    co_await socket.async_read(buffer);
}
```

## 在 Prism 中的应用

Prism 使用 `namespace net = boost::asio;`。

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| worker | 每线程一个 io_context | [[agent/worker/worker|worker]] |
| listener | 异步监听 | [[agent/front/listener|listener]] |
| pool | 异步连接池 | [[channel/connection/pool|pool]] |

## 参考资料

- [Boost.Asio](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)

## 相关知识
- [[ref/programming/c++23-coroutines|C++23 协程]] — 协程支持
