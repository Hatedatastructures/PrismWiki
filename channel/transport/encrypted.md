---
title: "encrypted — TLS 加密传输层"
source: "include/prism/channel/transport/encrypted.hpp"
module: "channel"
type: api
tags: [channel, transport, encrypted, tls, 加密]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/transmission
  - channel/adapter/connector
  - channel/transport/reliable
  - ref/protocol/tls-1.3
  - ref/network/tcp
  - ref/programming/boost-asio
---

# encrypted.hpp

> 源码: `include/prism/channel/transport/encrypted.hpp`
> 模块: [[channel|channel]]

## 概述

TLS 加密传输层。将 `ssl::stream<[[channel/adapter/connector|connector]]>` 适配为 [[channel/transport/transmission|transmission]] 接口，使协议装饰器（如 trojan::stream）能够装饰 TLS 加密流。

核心职责：
- 继承 transmission 接口实现 TLS 传输层
- 所有异步操作返回 `net::awaitable` 简化协程调用
- 自动映射 [[ref/programming/boost-asio|Boost.System]] 错误码到项目错误码

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[channel/adapter/connector|connector]] | 作为 ssl::stream 模板参数 |
| 被依赖 | TLS 相关协议 | 所有基于 [[ref/protocol/tls-1.3|TLS]] 的协议都使用此类 |

## 命名空间

`psm::channel::transport`

## 类: encrypted

> 源码: `include/prism/channel/transport/encrypted.hpp:45`

### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `connector_type` | `psm::channel::connector` | 适配器类型 |
| `stream_type` | `ssl::stream<connector_type>` | TLS 流类型 |
| `shared_stream` | `shared_ptr<stream_type>` | TLS 流共享指针 |

### 构造函数

---

#### encrypted(ssl_stream)

**功能说明**：使用已建立的 TLS 流创建加密传输层。TLS 流必须已完成握手。

**签名**：
```cpp
explicit encrypted(shared_stream ssl_stream);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `ssl_stream` | `shared_stream` | TLS 流的共享指针，已完成握手 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_encrypted()` 工厂函数

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

### 成员函数

---

#### is_reliable()

**功能说明**：TLS 基于 [[ref/network/tcp|TCP]]，始终返回 `true`。

**签名**：
```cpp
[[nodiscard]] bool is_reliable() const noexcept override;
```

**参数**：无

**返回值**：`bool` — 始终 `true`

**调用（向下）**：无

**被调用（向上）**：协议层根据可靠性选择策略

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### executor()

**功能说明**：返回底层 TLS 流关联的执行器。

**签名**：
```cpp
[[nodiscard]] executor_type executor() const override;
```

**参数**：无

**返回值**：`executor_type` — TLS 流执行器

**调用（向下）**：`ssl_stream_->get_executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] 框架，[[channel/adapter/connector|connector]]

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read_some()

**功能说明**：调用底层 TLS 流的 `async_read_some` 实现异步读取。

**签名**：
```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 读取的字节数

**调用（向下）**：`ssl_stream_->async_read_some()`

**被调用（向上）**：[[channel/transport/transmission|transmission]] 基类的 `async_read()` 循环

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]], [[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write_some()

**功能说明**：调用底层 TLS 流的 `async_write_some` 实现异步写入。

**签名**：
```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 写入的字节数

**调用（向下）**：`ssl_stream_->async_write_some()`

**被调用（向上）**：[[channel/transport/transmission|transmission]] 基类的 `async_write()` 循环

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]], [[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write_scatter()

**功能说明**：Scatter-gather 写入优化。`count==2` 时合并为单次 `SSL_write`，将帧头和载荷合并为一条 TLS 记录，避免两次加密操作和额外的 TLS 帧头开销。

**签名**：
```cpp
auto async_write_scatter(const std::span<const std::byte> *buffers, std::size_t count, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffers` | `const std::span<const std::byte> *` | 缓冲区数组 |
| `count` | `std::size_t` | 缓冲区数量 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 实际写入的总字节数

**调用（向下）**：`net::async_write(*ssl_stream_, bufs, token)` 或 `async_write()` 逐个写入

**被调用（向上）**：多路复用层帧头+载荷合并写入

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

---

#### close()

**功能说明**：先发送 TLS `close_notify` 通知对端，然后关闭底层传输层。`SSL_shutdown` 在非阻塞模式下立即返回。

**签名**：
```cpp
void close() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`::SSL_shutdown()`, `ssl_stream_->lowest_layer().transmission().close()`

**被调用（向上）**：会话结束时调用

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

---

#### cancel()

**功能说明**：取消底层传输层所有挂起的异步操作。

**签名**：
```cpp
void cancel() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`ssl_stream_->lowest_layer().transmission().cancel()`

**被调用（向上）**：超时处理或会话中断

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### stream() (非 const)

**功能说明**：返回内部 TLS 流的引用，用于直接操作 TLS 层。

**签名**：
```cpp
[[nodiscard]] stream_type &stream() noexcept;
```

**参数**：无

**返回值**：`stream_type &` — TLS 流引用

**调用（向下）**：无

**被调用（向上）**：TLS 握手操作、获取 TLS 会话信息

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

---

#### stream() (const)

**功能说明**：返回内部 TLS 流的常量引用，用于只读访问。

**签名**：
```cpp
[[nodiscard]] const stream_type &stream() const noexcept;
```

**参数**：无

**返回值**：`const stream_type &` — TLS 流常量引用

**调用（向下）**：无

**被调用（向上）**：只读场景

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

---

#### release()

**功能说明**：释放 TLS 流所有权，调用后对象不再持有流。

**签名**：
```cpp
shared_stream release();
```

**参数**：无

**返回值**：`shared_stream` — TLS 流共享指针

**调用（向下）**：`std::move(ssl_stream_)`

**被调用（向上）**：需要转移 TLS 流所有权的场景

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

### 工厂函数

---

#### make_encrypted()

**功能说明**：使用已建立的 TLS 流创建加密传输层实例。

**签名**：
```cpp
inline shared_transmission make_encrypted(encrypted::shared_stream ssl_stream);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `ssl_stream` | `encrypted::shared_stream` | TLS 流的共享指针 |

**返回值**：`shared_transmission` — 加密传输层指针

**调用（向下）**：`encrypted(ssl_stream)` 构造函数

**被调用（向上）**：TLS 握手完成后创建加密传输层

**涉及的知识域**：[[ref/protocol/tls-1.3|TLS 1.3]]

### close() 流程

```
encrypted::close()
  → ::SSL_shutdown(native_handle)     // 发送 close_notify
  → ssl_stream_->lowest_layer()       // 获取 connector
    → .transmission()                  // 获取内层 reliable
    → .close()                         // 关闭 TCP socket
```
