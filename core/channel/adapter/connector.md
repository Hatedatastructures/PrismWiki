---
tags: [channel, adapter, connector]
layer: core
module: channel
source: I:/code/Prism/include/prism/transport/adapter/connector.hpp
title: connector
---

# connector

Socket 异步 IO 适配器，将 `transmission` 适配为 Boost.Asio 的 `AsyncReadStream`/`AsyncWriteStream` 概念。支持预读数据注入，避免协议检测阶段数据丢失。

## 核心接口

| 方法 | 说明 |
|------|------|
| `connector(trans, preread)` | 构造，可选注入预读数据 |
| `async_read_some(buffers, token)` | 读取——优先返回预读数据，消费完后委托给 transmission |
| `async_write_some(buffers, token)` | 写入——直接委托给 transmission |
| `async_write(buffer, ec)` | 完整写入 |
| `async_read(buffer, ec)` | 完整读取 |
| `get_executor()` / `executor()` | 返回 io_context executor |
| `release()` | 释放 transmission 所有权 |

## 设计决策

### 为什么需要预读数据注入？

协议检测阶段（probe）已从 socket 读取了前 24 字节。这些数据被 `probe_result` 保存，但 socket 缓冲区中已不存在。后续协议握手需要这些数据（如 HTTP 请求行、SOCKS5 握手帧）。`connector` 在首次 `async_read_some` 时先返回预读数据，然后再委托给实际 socket。

**后果**: 预读数据必须在构造时注入，不能延迟。构造后无法追加。

### 为什么用 shared_ptr 持有 transmission？

`connector` 可能被 `co_spawn` 的独立协程持有。如果用裸指针，transmission 可能在协程挂起期间被析构。`shared_ptr` 保证异步操作期间传输对象存活。

**后果**: `release()` 将 shared_ptr 移出，之后 connector 不可再使用。

### 为什么 lowest_layer_type 是自身？

BoringSSL 的 `ssl::stream` 模板要求底层流有 `lowest_layer_type` 定义。connector 作为最底层（直接包装 transmission），lowest_layer 就是自己。

**后果**: ssl::stream\<connector\> 的 `lowest_layer()` 返回 connector 引用。

## 约束

### 预读数据注入时机

**类型**: 调用顺序

**规则**: 预读数据必须在协议握手前注入（构造时传入）。握手开始后无法追加。

**违反后果**: 协议解析读到不完整数据，握手失败。

**源码依据**: `connector.hpp` 构造函数

### async_read_some 的预读数据一次性

**类型**: 状态前置

**规则**: 预读数据消费完后（`preread_offset_ >= preread_buffer_.size()`），后续调用直接委托给 transmission。预读数据不会重复返回。

**源码依据**: `connector.hpp` async_read_some 实现

## 使用场景

```cpp
// 协议检测阶段已读取数据
auto result = co_await probe(transport, 24);

// 注入预读数据，创建 connector
connector conn(transport, result.preload_bytes());

// 协议握手——首次读取自动返回预读数据
co_await ssl_handshake(conn, ...);
```

## 引用关系

### 依赖

- [[core/channel/transport/transmission|transmission]]：底层传输抽象

### 被引用

- [[core/instance/worker/tls|worker/tls]]：创建 ssl::stream\<connector\>
- [[core/stealth/scheme|stealth::scheme]]：伪装方案握手使用
