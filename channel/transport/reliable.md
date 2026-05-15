---
title: "reliable — TCP 可靠流式传输"
source: "include/prism/channel/transport/reliable.hpp"
module: "channel"
type: api
tags: [channel, transport, reliable, tcp, 连接池]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/transmission
  - channel/connection/pool
  - channel/health
  - ref/network/tcp
  - ref/programming/boost-asio
  - ref/network/connection-pool
---

# reliable.hpp

> 源码: `include/prism/channel/transport/reliable.hpp`
> 模块: [[channel|channel]]

## 概述

可靠的流式传输实现（[[ref/network/tcp|TCP]]）。封装 `boost::asio::ip::tcp::socket`，继承 [[channel/transport/transmission|transmission]] 接口。

核心特性：
- TCP 保证数据有序送达不丢失不重复
- 支持 [[channel/connection/pool|连接池]] 复用：通过 `pooled_connection` 构造时，`close()` 归还到连接池而非直接关闭
- scatter-gather 写入优化：帧头+载荷合并为单次系统调用

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[channel/connection/pool|pool]] | pooled_connection 连接池复用 |
| 依赖 | [[channel/health|health]] | recycle 时健康检测 |
| 被依赖 | 出站代理 | 所有基于 TCP 的连接都使用此类 |

## 命名空间

`psm::channel::transport`

## 类: reliable

> 源码: `include/prism/channel/transport/reliable.hpp:51`

### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `socket_type` | `net::ip::tcp::socket` | TCP socket 类型 |

### 构造函数

---

#### reliable(executor)

**功能说明**：使用执行器初始化未打开的 TCP socket，需后续调用 `open` 或 `accept`。

**签名**：
```cpp
explicit reliable(net::any_io_executor executor);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `executor` | `net::any_io_executor` | 执行器，用于初始化 socket |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_reliable(executor)` 工厂函数

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### reliable(socket)

**功能说明**：使用已构造并连接的 TCP socket 初始化传输层。

**签名**：
```cpp
explicit reliable(socket_type socket);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `socket_type` | 已连接的 TCP socket |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_reliable(socket)` 工厂函数

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### reliable(pooled)

**功能说明**：从 [[channel/connection/pool|连接池]] 获取的连接创建传输层。析构或 `close()` 时 socket 将被归还到连接池而非直接关闭，实现连接复用。

**签名**：
```cpp
explicit reliable(psm::channel::pooled_connection pooled);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `pooled` | `psm::channel::pooled_connection` | 来自连接池的 RAII 连接包装器 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_reliable(pooled_connection)` 工厂函数

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 成员函数

---

#### executor()

**功能说明**：返回底层 socket 关联的执行器，用于调度异步操作。

**签名**：
```cpp
executor_type executor() const override;
```

**参数**：无

**返回值**：`executor_type` — socket 执行器

**调用（向下）**：`native_socket().get_executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] 框架，[[channel/adapter/connector|connector]]

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read_some()

**功能说明**：调用底层 socket 的 `async_read_some` 实现 TCP 异步读取，返回实际读取的字节数。

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

**调用（向下）**：`native_socket().async_read_some()`, [[ref/programming/boost-asio|Boost.Asio]] `redirect_error`

**被调用（向上）**：[[channel/transport/transmission|transmission]] 基类的 `async_read()` 循环

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write_some()

**功能说明**：调用底层 socket 的 `async_write_some` 实现 TCP 异步写入。

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

**调用（向下）**：`native_socket().async_write_some()`

**被调用（向上）**：[[channel/transport/transmission|transmission]] 基类的 `async_write()` 循环

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write_scatter()

**功能说明**：Scatter-gather 写入优化。`count==2` 时通过 `net::async_write` 合并为单次 `WSASend`/`writev` 系统调用，避免帧头与载荷分两次写入。

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

**调用（向下）**：`net::async_write(native_socket(), bufs, token)` 或 `async_write()` 逐个写入

**被调用（向上）**：多路复用层帧头+载荷合并写入

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### close()

**功能说明**：关闭传输层。如果连接来自 [[channel/connection/pool|连接池]]（`pooled_` 非空），则取消挂起操作但不关闭 socket，由析构函数通过 `pooled_.reset()` 归还到连接池；否则直接关闭 socket。

**签名**：
```cpp
void close() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`socket->cancel()` 或 `socket->close()`

**被调用（向上）**：会话结束时调用

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/network/connection-pool|连接池]]

---

#### cancel()

**功能说明**：取消所有未完成的异步读写操作，被取消的操作返回 `operation_canceled`。

**签名**：
```cpp
void cancel() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`native_socket().cancel()`

**被调用（向上）**：超时处理或会话中断

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### shutdown_write()

**功能说明**：半关闭写端，调用 socket 的 `shutdown_send` 通知对端不再发送数据。

**签名**：
```cpp
void shutdown_write() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`native_socket().shutdown(shutdown_send)`

**被调用（向上）**：协议层在完成发送后调用

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### is_reliable()

**功能说明**：重写基类虚函数，返回 `true` 标识这是可靠传输。

**签名**：
```cpp
[[nodiscard]] bool is_reliable() const noexcept override;
```

**参数**：无

**返回值**：`bool` — 始终返回 `true`

**调用（向下）**：无

**被调用（向上）**：协议层根据可靠性选择策略

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### native_socket() (非 const)

**功能说明**：返回底层 TCP socket 的引用。如果连接来自 [[channel/connection/pool|连接池]]，则返回池中 socket。

**签名**：
```cpp
socket_type &native_socket() noexcept;
```

**参数**：无

**返回值**：`socket_type &` — socket 引用

**调用（向下）**：无

**被调用（向上）**：所有异步操作、设置 socket 选项（如 `TCP_NODELAY`）

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### native_socket() (const)

**功能说明**：返回底层 TCP socket 的常量引用，用于只读访问。

**签名**：
```cpp
const socket_type &native_socket() const noexcept;
```

**参数**：无

**返回值**：`const socket_type &` — socket 常量引用

**调用（向下）**：无

**被调用（向上）**：`executor()`, `is_reliable()` 等只读场景

**涉及的知识域**：[[ref/network/tcp|TCP]]

### 工厂函数

---

#### make_reliable(executor)

**功能说明**：使用执行器创建未打开的 TCP 传输层实例。

**签名**：
```cpp
inline shared_transmission make_reliable(net::any_io_executor executor);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `executor` | `net::any_io_executor` | 执行器 |

**返回值**：`shared_transmission` — reliable 实例

**调用（向下）**：`reliable(executor)` 构造函数

**被调用（向上）**：代理服务端接受连接时

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### make_reliable(socket)

**功能说明**：使用已连接的 TCP socket 创建传输层实例。

**签名**：
```cpp
inline shared_transmission make_reliable(net::ip::tcp::socket socket);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `net::ip::tcp::socket` | 已连接的 TCP socket |

**返回值**：`shared_transmission` — reliable 实例

**调用（向下）**：`reliable(socket)` 构造函数

**被调用（向上）**：listener 接受连接后创建传输层

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### make_reliable(pooled)

**功能说明**：使用来自 [[channel/connection/pool|连接池]] 的连接创建传输层实例。析构时自动归还到连接池。

**签名**：
```cpp
inline shared_transmission make_reliable(psm::channel::pooled_connection pooled);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `pooled` | `psm::channel::pooled_connection` | 来自连接池的连接 |

**返回值**：`shared_transmission` — reliable 实例

**调用（向下）**：`reliable(pooled)` 构造函数

**被调用（向上）**：[[channel/eyeball/racer|racer]] 竞速成功后、[[channel/connection/pool|pool]] `async_acquire` 命中缓存后

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 连接池复用流程

```
make_reliable(pooled) → reliable 构造
  → 使用连接 → close() 取消挂起操作但不关闭 socket
  → 析构 → pooled_.reset() → connection_pool::recycle()
    → healthy_fast() 健康检测 → 入栈或销毁
```
