---
title: "connector — Socket 异步 IO 适配器"
source: "include/prism/channel/adapter/connector.hpp"
module: "channel"
type: api
tags: [channel, adapter, connector, asio, 适配器]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/transmission
  - channel/transport/encrypted
  - memory/container
  - ref/programming/boost-asio
  - ref/network/tcp
  - ref/protocol/tls-1.3
---

# connector.hpp

> 源码: `include/prism/channel/adapter/connector.hpp`
> 模块: [[channel|channel]]

## 概述

Socket 异步 IO 适配器。统一 [[ref/network/tcp|TCP]] 和 [[ref/network/udp|UDP]] 的异步读写接口，将 [[channel/transport/transmission|transmission]] 适配为 [[ref/programming/boost-asio|Boost.Asio]] 的 `AsyncReadStream`/`AsyncWriteStream` 概念。

核心特性：
- 将 transmission 接口适配为 Boost.Asio 标准流概念
- 支持注入预读数据，避免协议检测阶段已读取数据丢失
- 内部使用 `shared_ptr` 持有 transmission，确保异步操作期间传输对象不被提前释放

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/transport/transmission|transmission]] | 持有 shared_transmission |
| 依赖 | [[memory/container|memory]] | PMR 向量用于预读缓冲区 |
| 被依赖 | BoringSSL `ssl::stream` | 作为 `ssl::stream` 的模板参数 |
| 被依赖 | [[channel/transport/encrypted|encrypted]] | 通过 `ssl::stream<connector>` 间接使用 |

## 命名空间

`psm::channel`

## 类: connector

> 源码: `include/prism/channel/adapter/connector.hpp:42`

### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `executor_type` | `any_io_executor` | 执行器类型 |
| `transmission_ptr` | `shared_transmission` | 传输层指针类型 |
| `lowest_layer_type` | `connector` | 底层类型（自身） |

### 构造函数

---

#### connector(trans, preread)

**功能说明**：使用传输层指针和可选的预读数据构造适配器。预读数据将在首次 `async_read_some` 调用时优先返回。

**签名**：
```cpp
explicit connector(transmission_ptr trans, std::span<const std::byte> preread = {});
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `trans` | `transmission_ptr` | 传输层对象指针（所有权转移） |
| `preread` | `std::span<const std::byte>` | 预读数据切片，默认为空 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：协议识别完成后构造适配器，传入预读的协议探测数据

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### connector(connector &&)

**功能说明**：移动构造函数，转移传输层指针、预读缓冲区和偏移量的所有权。移动后源对象偏移量重置为零。

**签名**：
```cpp
connector(connector &&other) noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `other` | `connector &&` | 要移动的适配器对象 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：作为 `ssl::stream` 模板参数时移动构造

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### operator=(connector &&)

**功能说明**：移动赋值运算符，转移所有权并防止自赋值。

**签名**：
```cpp
connector &operator=(connector &&other) noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `other` | `connector &&` | 要移动的适配器对象 |

**返回值**：`connector &` — 当前对象的引用

**调用（向下）**：无

**被调用（向上）**：移动赋值场景

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

### 成员函数

---

#### get_executor()

**功能说明**：返回底层传输层关联的执行器，满足 Boost.Asio AsyncStream 概念要求。

**签名**：
```cpp
executor_type get_executor();
```

**参数**：无

**返回值**：`executor_type` — 执行器对象

**调用（向下）**：`trans_->executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] `ssl::stream` 内部调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### executor()

**功能说明**：委托给 `get_executor()`，提供便捷的执行器访问。

**签名**：
```cpp
executor_type executor();
```

**参数**：无

**返回值**：`executor_type` — 执行器对象

**调用（向下）**：`get_executor()`

**被调用（向上）**：直接调用场景

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read_some(buffers, token)

**功能说明**：适配 Boost.Asio 的 `async_read_some`。如果存在未消费的预读数据，优先从预读缓冲区拷贝到用户缓冲区（同步），预读消费完毕后委托给传输层（异步）。

**签名**：
```cpp
template <typename MutableBufferSequence, typename CompletionToken>
auto async_read_some(const MutableBufferSequence &buffers, CompletionToken &&token);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffers` | `const MutableBufferSequence &` | 可变缓冲区序列 |
| `token` | `CompletionToken &&` | 完成令牌（协程、回调等） |

**返回值**：取决于 `CompletionToken`，`use_awaitable` 时为 `net::awaitable<std::pair<std::size_t, std::error_code>>`

**调用（向下）**：
- 预读阶段：`std::memcpy` 从 `preread_buffer_` 拷贝
- 消费完毕：`transport::async_read_some(trans_, buffers, token)`

**被调用（向上）**：BoringSSL `ssl::stream` 内部调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/protocol/tls-1.3|TLS]]

---

#### async_write_some(buffers, token)

**功能说明**：适配 Boost.Asio 的 `async_write_some`，直接委托给传输层。

**签名**：
```cpp
template <typename ConstBufferSequence, typename CompletionToken>
auto async_write_some(const ConstBufferSequence &buffers, CompletionToken &&token);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffers` | `const ConstBufferSequence &` | 常量缓冲区序列 |
| `token` | `CompletionToken &&` | 完成令牌 |

**返回值**：取决于 `CompletionToken`

**调用（向下）**：`transport::async_write_some(trans_, buffers, token)`

**被调用（向上）**：BoringSSL `ssl::stream` 内部调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/protocol/tls-1.3|TLS]]

---

#### async_write(buffer, ec)

**功能说明**：完整写入操作，委托给 `transmission` 的 `async_write` 虚函数，允许子类（如 [[ref/network/udp|UDP]]）自定义行为。

**签名**：
```cpp
auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 要写入的数据缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际写入的总字节数

**调用（向下）**：`trans_->async_write(buffer, ec)`

**被调用（向上）**：协议层完整写入数据

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read(buffer, ec)

**功能说明**：完整读取操作，委托给 `transmission` 的 `async_read` 虚函数。

**签名**：
```cpp
auto async_read(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收数据的缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际读取的总字节数

**调用（向下）**：`trans_->async_read(buffer, ec)`

**被调用（向上）**：协议层完整读取数据

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### lowest_layer() (非 const)

**功能说明**：返回 `connector` 自身的引用，满足 Boost.Asio `lowest_layer` 访问要求。

**签名**：
```cpp
lowest_layer_type &lowest_layer();
```

**参数**：无

**返回值**：`lowest_layer_type &` — 当前对象的引用

**调用（向下）**：无

**被调用（向上）**：`ssl::stream` 访问最底层流

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### lowest_layer() (const)

**功能说明**：返回 `connector` 自身的常量引用。

**签名**：
```cpp
[[nodiscard]] const lowest_layer_type &lowest_layer() const;
```

**参数**：无

**返回值**：`const lowest_layer_type &` — 当前对象的常量引用

**调用（向下）**：无

**被调用（向上）**：只读场景

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### transmission()

**功能说明**：返回内部持有的传输层对象的引用，用于直接操作传输层。

**签名**：
```cpp
auto &transmission() const;
```

**参数**：无

**返回值**：`transport::transmission &` — 传输层对象的引用

**调用（向下）**：无

**被调用（向上）**：[[channel/transport/encrypted|encrypted]] 的 `close()` 和 `cancel()` 通过 `lowest_layer().transmission()` 访问内层传输

**涉及的知识域**：[[channel/transport/transmission|transmission]]

---

#### release()

**功能说明**：释放传输层所有权，调用后对象不再持有传输层。

**签名**：
```cpp
transmission_ptr release();
```

**参数**：无

**返回值**：`transmission_ptr` — 传输层对象指针

**调用（向下）**：`std::move(trans_)`

**被调用（向上）**：需要转移传输层所有权的场景

**涉及的知识域**：[[channel/transport/transmission|transmission]]

### 设计约束

- 预读数据注入必须在协议握手之前完成
- 满足 Boost.Asio `AsyncReadStream`/`AsyncWriteStream` 概念，可直接用于 `ssl::stream` 模板参数
- 内部持有 `shared_ptr<transmission>`，确保异步操作期间传输对象不被提前释放

### 在 TLS 栈中的位置

```
ssl::stream<connector>
  └── connector（适配层，持有 shared_transmission）
        └── reliable / unreliable（实际传输）
```
