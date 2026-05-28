---
tags: [transport, adapter, connector]
layer: core
source: I:/code/Prism/include/prism/transport/adapter/connector.hpp
title: connector
---

# connector

Socket 异步 IO 适配器，将 `transmission` 接口适配为 Boost.Asio 的 `AsyncReadStream`/`AsyncWriteStream` 概念。

## 概述

`connector` 解决的核心问题：BoringSSL 的 `ssl::stream<Stream>` 模板要求底层 Stream 满足 AsyncStream 概念（`async_read_some`/`async_write_some` 模板方法 + `get_executor`），而 `transmission` 是纯虚基类，不满足这些模板要求。

关键能力：

- **接口适配** — 将 transmission 适配为 Asio 流概念，使 `ssl::stream<connector>` 可以包装任意 transmission 实现
- **预读数据注入** — 构造时注入协议检测阶段已读取的数据，首次 `async_read_some` 时优先返回，避免数据丢失
- **所有权管理** — 使用 `shared_ptr` 持有 transmission，确保异步操作期间对象存活

## 接口

| 方法 | 签名 | 说明 |
|------|------|------|
| 构造 | `connector(transmission_ptr, span<const byte> preread = {})` | 接受传输层 + 可选预读数据 |
| 移动 | `connector(connector&&)` / `operator=(connector&&)` | 仅移动，禁止拷贝 |
| get_executor | `-> executor_type` | 委托给底层 transmission |
| async_read_some | `(MutableBufferSequence, CompletionToken)` | 预读缓冲区优先，耗尽后委托 transmission |
| async_write_some | `(ConstBufferSequence, CompletionToken)` | 直接委托 transmission，零额外开销 |
| async_write | `(span<const byte>, error_code&) -> awaitable<size_t>` | 完整写入，委托 transmission 虚函数 |
| async_read | `(span<byte>, error_code&) -> awaitable<size_t>` | 完整读取，委托 transmission 虚函数 |
| lowest_layer | `-> connector&` | 返回自身，满足 Asio lowest_layer 要求 |
| transmission | `-> transmission&` | 访问底层传输层 |
| release | `-> transmission_ptr` | 释放所有权，之后对象不可用 |

内部状态：`transmission_ptr trans_` + `memory::vector<byte> preread_buffer_` + `size_t preread_offset_`

### async_read_some 预读逻辑

预读缓冲区有未消费数据时，直接从缓冲区 memcpy 到用户 buffer 并同步返回（不发起异步操作）。缓冲区耗尽后，委托给 transmission 的 completion-handler 路径。

```
async_read_some 调用
        |
        v
预读缓冲区有数据？
     |         |
    Yes       No
     |         |
从缓冲区    委托 transmission
memcpy       async_read_some
同步返回     异步返回
```

### async_write_some

直接通过 `async_initiate` 将 Asio buffer 适配为 `std::span<const std::byte>` 委托给 transmission，写入路径零额外开销。

## 调用链

```
protocol -> connector -> transmission -> socket
```

| Asio 概念 | connector 实现 |
|-----------|----------------|
| `AsyncReadStream` | `async_read_some()` |
| `AsyncWriteStream` | `async_write_some()` |
| `get_executor()` | 委托给 transmission |
| `lowest_layer_type` | 自身 |

## 设计决策

### 为什么需要 connector 适配器？

**问题**: BoringSSL 的 `ssl::stream<Stream>` 模板要求 Stream 满足 Boost.Asio 的 AsyncStream 概念。transmission 是纯虚基类，不满足模板概念。

**选择**: connector 作为适配器，将 transmission 接口适配为 AsyncStream 概念。内部持有 `shared_ptr<transmission>`，提供模板化的 async_read_some/async_write_some 和 get_executor。

**后果**: `ssl::stream<connector>` 可以包装任意 transmission 实现，使 TLS 加密层与传输层完全解耦。

**替代方案**: 让 transmission 继承 AsyncStream 概念，但这需要 transmission 成为模板类，破坏运行时多态。

**源码依据**: `transport/adapter/connector.hpp:39-266`

### 为什么内嵌预读数据而非使用 preview 装饰器？

**问题**: ssl::stream 在构造时就开始使用底层 stream 的 async_read_some。如果底层是 preview 装饰器，ssl::stream 的首次读取会经过 preview 的预读缓冲区再经过 co_spawn 桥接，产生额外协程帧。

**选择**: connector 内嵌独立的 `preread_buffer_`，在首次 async_read_some 时同步返回预读数据，避免额外的协程帧和装饰器层级。

**后果**: 预读数据在 connector 和 preview 两层都存在可能。实际使用中，ssl::stream 构造时使用 connector 的预读，非 TLS 场景使用 preview 的预读，不会重复。

**源码依据**: `transport/adapter/connector.hpp:133-164`

## 约束

### connector 禁止拷贝

**类型**: 所有权
**规则**: 禁止拷贝构造和拷贝赋值，只允许移动。`shared_ptr<transmission>` 的所有权语义与 `ssl::stream` 的生命周期绑定。
**违反后果**: 编译期错误

### preread_buffer_ 只在 async_read_some 中消费

**类型**: 状态一致性
**规则**: 预读缓冲区只在 async_read_some 中消费，async_write_some 不操作预读缓冲区。
**违反后果**: 无实际违反途径（private 成员）

### release() 后对象不可用

**类型**: 生命周期
**规则**: release() 将内部 trans_ 移动返回后，connector 不再持有传输层。
**违反后果**: 空指针解引用

## 跨模块契约

| 上游模块 | 对 connector 的调用 | 契约 |
|----------|---------------------|------|
| encrypted | `ssl::stream<connector>(connector(transmission), ssl_ctx)` | connector 满足 AsyncStream 概念 |
| encrypted::ssl_handshake | `connector(inbound, preread)` + `ssl_stream->handshake()` | 预读数据在 TLS 握手前注入 |
| stealth/reality | `connector(transmission)` 直接构造 | 无预读数据，纯适配 |

## 注意事项

1. **预读时机** — 预读数据注入必须在协议握手之前完成
2. **数据丢失** — 注入时机不当可能导致协议解析失败
3. **线程安全** — 遵循 transmission 的线程安全保证

## 相关类型

- [[core/transport/transmission]] - 底层传输抽象
- [[core/transport/overview]] - 通道层
