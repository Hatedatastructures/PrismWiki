---
title: "transmission — 传输层抽象接口"
source: "include/prism/channel/transport/transmission.hpp"
module: "channel"
type: api
tags: [channel, transport, transmission, 接口, 协程]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/reliable
  - channel/transport/encrypted
  - channel/transport/unreliable
  - channel/transport/snapshot
  - channel/adapter/connector
  - ref/network/tcp
  - ref/network/udp
  - ref/programming/boost-asio
---

# transmission.hpp

> 源码: `include/prism/channel/transport/transmission.hpp`
> 模块: [[channel|channel]]

## 概述

传输层抽象接口。定义所有传输层的统一接口，采用纯协程设计，使用 `net::awaitable` 作为异步操作返回类型。所有具体的传输实现（TCP、UDP）和协议装饰器都应继承此接口。

设计特性：
- **分层架构** — 支持 TCP、UDP 和协议装饰器的分层设计
- **协程优先** — 异步操作返回 `net::awaitable`，避免回调地狱
- **错误码返回** — 通过 `std::error_code&` 参数返回错误，避免异常开销
- **概念兼容** — 提供 `get_executor` 方法，兼容 [[ref/programming/boost-asio|Boost.Asio]] 执行器概念

## 命名空间

`psm::channel::transport`

## 类: transmission

传输层抽象基类，所有传输实现必须继承此接口。

### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `executor_type` | `net::any_io_executor` | 执行器类型 |

### 成员函数

---

#### ~transmission()

**功能说明**：虚析构函数，确保子类析构函数被正确调用。

**签名**：
```cpp
virtual ~transmission() = default;
```

**参数**：无

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：所有派生类析构时隐式调用

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

---

#### is_reliable()

**功能说明**：检查传输是否可靠（如 [[ref/network/tcp|TCP]]），用于优化路径选择，避免 `dynamic_cast`。

**签名**：
```cpp
[[nodiscard]] virtual bool is_reliable() const noexcept;
```

**参数**：无

**返回值**：`bool` — `true` 表示可靠传输（TCP），`false` 表示不可靠（[[ref/network/udp|UDP]]）

**调用（向下）**：无

**被调用（向上）**：
- [[channel/transport/reliable|reliable]] 重写返回 `true`
- [[channel/transport/encrypted|encrypted]] 重写返回 `true`
- [[channel/transport/unreliable|unreliable]] 继承默认返回 `false`
- [[channel/transport/snapshot|snapshot]] 委托给内层传输
- 协议层根据可靠性选择转发策略

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/network/udp|UDP]]

---

#### executor()

**功能说明**：获取关联的执行器，用于调度异步操作。纯虚函数，必须由子类实现。

**签名**：
```cpp
[[nodiscard]] virtual executor_type executor() const = 0;
```

**参数**：无

**返回值**：`executor_type` — 关联的 Boost.Asio 执行器

**调用（向下）**：无

**被调用（向上）**：
- [[channel/adapter/connector|connector]] 的 `get_executor()` 委托给此方法
- [[ref/programming/boost-asio|Boost.Asio]] 异步框架通过此方法获取执行器

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### get_executor()

**功能说明**：获取关联的执行器，兼容 [[ref/programming/boost-asio|Boost.Asio]] AsyncStream 概念。委托给 `executor()`。

**签名**：
```cpp
[[nodiscard]] executor_type get_executor() const;
```

**参数**：无

**返回值**：`executor_type` — 执行器

**调用（向下）**：`executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] 框架内部调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read_some()

**功能说明**：异步读取部分数据到缓冲区，可能返回比请求更少的字节。纯虚函数。

**签名**：
```cpp
virtual auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> = 0;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际读取的字节数

**调用（向下）**：由派生类实现，调用底层 socket API

**被调用（向上）**：
- `async_read()` 循环调用此方法
- [[channel/transport/snapshot|snapshot]] 捕获读取内容后委托给内层
- [[channel/adapter/connector|connector]] 预读数据消费完毕后委托
- 通过文件作用域 `async_read_some` 包装函数适配 Asio 概念

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/programming/c++23-coroutines|C++23 协程]]

---

#### async_write_some()

**功能说明**：异步写入部分数据，可能写入比请求更少的字节。纯虚函数。

**签名**：
```cpp
virtual auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> = 0;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际写入的字节数

**调用（向下）**：由派生类实现

**被调用（向上）**：
- `async_write()` 循环调用此方法
- [[channel/adapter/connector|connector]] 直接委托
- 通过文件作用域 `async_write_some` 包装函数适配 Asio 概念

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/programming/c++23-coroutines|C++23 协程]]

---

#### shutdown_write()

**功能说明**：半关闭写端，通知对端不再发送数据。默认空实现，[[channel/transport/reliable|reliable]] 重写以调用 socket shutdown。

**签名**：
```cpp
virtual void shutdown_write();
```

**参数**：无

**返回值**：无

**调用（向下）**：派生类中调用 `socket.shutdown(shutdown_send)`

**被调用（向上）**：协议层在完成发送后调用

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### close()

**功能说明**：关闭传输层，释放底层资源。关闭后所有未完成的异步操作将被取消。纯虚函数。

**签名**：
```cpp
virtual void close() = 0;
```

**参数**：无

**返回值**：无

**调用（向下）**：由派生类实现

**被调用（向上）**：
- [[channel/transport/encrypted|encrypted]] 先发 `SSL_shutdown` 再关闭底层
- [[channel/transport/snapshot|snapshot]] 委托给内层传输
- 会话生命周期结束时调用

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### cancel()

**功能说明**：取消所有未完成的异步操作，被取消的操作返回 `operation_canceled` 错误。纯虚函数。

**签名**：
```cpp
virtual void cancel() = 0;
```

**参数**：无

**返回值**：无

**调用（向下）**：由派生类实现

**被调用（向上）**：
- [[channel/transport/snapshot|snapshot]] 委托给内层传输
- 超时处理或连接中断时调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write()

**功能说明**：完整写入操作，循环调用 `async_write_some` 直到所有数据发送完毕。子类（如 UDP）可重写。

**签名**：
```cpp
virtual auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 要写入的数据缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际写入的总字节数

**调用（向下）**：`async_write_some()`

**被调用（向上）**：
- `async_write_scatter()` 逐缓冲区调用
- [[channel/adapter/connector|connector]] 的 `async_write()` 委托
- [[ref/network/udp|UDP]] 的 `unreliable` 重写为直接调用 `async_write_some`

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/programming/c++23-coroutines|C++23 协程]]

---

#### async_write_scatter()

**功能说明**：Scatter-gather 写入操作，将多个缓冲区按顺序写入，减少系统调用次数。默认实现逐个调用 `async_write`。

**签名**：
```cpp
virtual auto async_write_scatter(const std::span<const std::byte> *buffers, std::size_t count, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffers` | `const std::span<const std::byte> *` | 缓冲区数组 |
| `count` | `std::size_t` | 缓冲区数量 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际写入的总字节数

**调用（向下）**：`async_write()`

**被调用（向上）**：
- [[channel/transport/reliable|reliable]] 和 [[channel/transport/encrypted|encrypted]] 重写为原生 scatter-gather 优化
- 多路复用帧头+载荷合并写入

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### async_read()

**功能说明**：完整读取操作，循环调用 `async_read_some` 直到缓冲区填满。

**签名**：
```cpp
virtual auto async_read(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收数据的缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际读取的总字节数

**调用（向下）**：`async_read_some()`

**被调用（向上）**：
- [[channel/adapter/connector|connector]] 的 `async_read()` 委托
- 协议解析需要完整数据时调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/programming/c++23-coroutines|C++23 协程]]

### 文件作用域模板函数

---

#### async_read_some (包装)

**功能说明**：将 [[ref/programming/boost-asio|Boost.Asio]] 的 `async_read_some` 调用适配到 `transmission` 接口，支持任意完成令牌。通过 `async_initiate` 桥接协程和回调。

**签名**：
```cpp
template <typename MutableBufferSequence, typename CompletionToken>
auto async_read_some(std::shared_ptr<transmission> trans, const MutableBufferSequence &buffers, CompletionToken &&token);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `trans` | `std::shared_ptr<transmission>` | 传输对象共享指针 |
| `buffers` | `const MutableBufferSequence &` | 可变缓冲区序列 |
| `token` | `CompletionToken &&` | 完成令牌（协程、回调等） |

**返回值**：取决于 `CompletionToken`，`use_awaitable` 时为 `net::awaitable<std::pair<std::size_t, std::error_code>>`

**调用（向下）**：`trans->async_read_some()`

**被调用（向上）**：
- [[channel/adapter/connector|connector]] 的 `async_read_some()` 预读消费完毕后调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write_some (包装)

**功能说明**：将 [[ref/programming/boost-asio|Boost.Asio]] 的 `async_write_some` 调用适配到 `transmission` 接口，支持任意完成令牌。

**签名**：
```cpp
template <typename ConstBufferSequence, typename CompletionToken>
auto async_write_some(std::shared_ptr<transmission> trans, const ConstBufferSequence &buffers, CompletionToken &&token);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `trans` | `std::shared_ptr<transmission>` | 传输对象共享指针 |
| `buffers` | `const ConstBufferSequence &` | 常量缓冲区序列 |
| `token` | `CompletionToken &&` | 完成令牌 |

**返回值**：取决于 `CompletionToken`

**调用（向下）**：`trans->async_write_some()`

**被调用（向上）**：
- [[channel/adapter/connector|connector]] 的 `async_write_some()` 委托

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

### 类型别名

---

#### shared_transmission

**功能说明**：传输层智能指针类型，用于管理传输层对象的生命周期。

**签名**：
```cpp
using shared_transmission = std::shared_ptr<transmission>;
```

**调用（向下）**：无

**被调用（向上）**：
- 所有工厂函数返回此类型
- [[channel/adapter/connector|connector]] 持有此类型
- [[channel/transport/snapshot|snapshot]] 持有内层传输

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

### 派生层次

```
transmission
  ├── reliable    [[channel/transport/reliable|reliable]]
  ├── encrypted   [[channel/transport/encrypted|encrypted]]
  ├── unreliable  [[channel/transport/unreliable|unreliable]]
  └── snapshot    [[channel/transport/snapshot|snapshot]]
```
