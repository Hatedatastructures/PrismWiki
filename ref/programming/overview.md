---
title: Ref Programming Overview
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, reference]
---

# 编程知识总览

> 本目录提供 Prism 项目相关的编程知识参考，包括 C++23 协程、Boost.Asio、PMR 内存管理等。
> 最后更新：2026-05-17

---

## 知识领域

| 领域 | 页面 | 内容 |
|------|------|------|
| 异步编程 | [[boost-asio|Boost.Asio 协程]] | Asio 异步模型、协程适配 |
| 语言特性 | [[cpp23-coroutine|C++23 协程]] | 协程原理、编译器支持 |
| 内存管理 | [[pmr-concepts|PMR 概念]] | Polymorphic Memory Resources |
| 编译优化 | [[constexpr|Constexpr 计算]] | 编译期计算、常量表达式 |
| 错误处理 | [[error-handling|错误处理策略]] | 异常 vs 错误码、Result 类型 |
| Go 参考 | [[go-concurrency|Go 并发模型]] | Goroutine、Channel（mihomo 参考） |

---

## Prism 编程特性

### C++23 协程

Prism 使用 C++23 协程实现异步处理：

```cpp
// 协程示例
asio::awaitable<void> handle_connection(tcp::socket socket) {
    auto data = co_await read_data(socket);
    co_await process_data(data);
    co_await write_response(socket);
}
```

详见 [[cpp23-coroutine|C++23 协程]]。

### PMR 内存池

热路径使用 PMR 避免动态分配：

```cpp
// PMR 使用
std::pmr::vector<uint8_t> buffer(pool_);
auto span = std::span(buffer.data(), buffer.size());
```

详见 [[pmr-concepts|PMR 概念]]。

### Boost.Asio

网络层基于 Boost.Asio：

```cpp
// Asio 协程
asio::co_spawn(ctx, handle_session(std::move(socket)), asio::detached);
```

详见 [[boost-asio|Boost.Asio 协程]]。

---

## 与其他层的关联

- 实现细节见 [[core/overview|Core 层]]
- 协程规范见 [[dev/coding/coroutine|协程约定]]
- PMR 使用见 [[dev/coding/pmr|PMR 使用规范]]

---

## 相关参考

- [[ref/crypto/overview|加密知识]] — AEAD、HKDF 等
- [[ref/protocol/overview|协议知识]] — TLS、SOCKS5 等
- [[ref/network/overview|网络知识]] — Happy Eyeballs、DNS 等