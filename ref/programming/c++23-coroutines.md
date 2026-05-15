---
title: "C++23 协程"
category: "programming"
type: ref
tags: [编程, c++, 协程, 异步]
created: 2026-05-15
updated: 2026-05-15
---

# C++23 协程

**类别**: 编程

## 概述

C++23 协程是 C++20 引入的协程特性，允许编写异步代码而无需回调。Prism 使用协程实现纯异步架构。

## 原理

### 核心概念

- **co_await**: 挂起协程，等待异步操作完成
- **co_return**: 从协程返回值
- **co_yield**: 生成值（用于生成器）

### 协程类型

- **awaitable**: 可等待的对象
- **promise_type**: 协程的承诺类型
- **coroutine_handle**: 协程句柄

## 在 Prism 中的应用

Prism 使用 `boost::asio::awaitable<T>` 作为协程类型。

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| handshake | 协程握手 | [[stealth/scheme|scheme]] |
| session | 协程会话 | [[agent/session/session|session]] |

## 参考资料

- [C++20 Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)

## 相关知识
- [[ref/programming/boost-asio|Boost.Asio]] — 异步 I/O 框架
