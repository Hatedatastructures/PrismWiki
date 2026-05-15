---
title: "unreliable — UDP 数据报传输"
source: "include/prism/channel/transport/unreliable.hpp"
module: "channel"
type: api
tags: [channel, transport, unreliable, udp, 数据报]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/transmission
  - ref/network/udp
  - ref/programming/boost-asio
  - ref/network/tcp
---

# unreliable.hpp

> 源码: `include/prism/channel/transport/unreliable.hpp`
> 模块: [[channel|channel]]

## 概述

不可靠的数据报传输实现（[[ref/network/udp|UDP]]）。封装 `boost::asio::ip::udp::socket`，继承 [[channel/transport/transmission|transmission]] 接口。模拟流式语义，内部记录远程端点以实现连接式操作。

设计特性：
- **数据报语义** — UDP 不保证数据送达、顺序或去重
- **连接模拟** — 通过记录远程端点实现类似 [[ref/network/tcp|TCP]] 的连接式操作
- **来源过滤** — 接收时自动过滤非远程端点的数据报

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 被依赖 | UDP 出站代理 | 所有基于 UDP 的连接使用此类 |

## 命名空间

`psm::channel::transport`

## 类: unreliable

> 源码: `include/prism/channel/transport/unreliable.hpp:53`

### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `socket_type` | `net::ip::udp::socket` | UDP socket 类型 |
| `endpoint_type` | `net::ip::udp::endpoint` | UDP 端点类型 |

### 构造函数

---

#### unreliable(executor, remote_endpoint)

**功能说明**：使用执行器初始化 UDP socket。Socket 在构造时不打开，远程端点可选，未设置时首次接收的数据报来源将自动设为远程端点。

**签名**：
```cpp
explicit unreliable(net::any_io_executor executor, std::optional<endpoint_type> remote_endpoint = std::nullopt);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `executor` | `net::any_io_executor` | 执行器，用于初始化 socket |
| `remote_endpoint` | `std::optional<endpoint_type>` | 远程端点（可选） |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_unreliable(executor)` 工厂函数

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### unreliable(socket, remote_endpoint)

**功能说明**：使用已构造的 UDP socket 初始化传输层。Socket 必须已打开。

**签名**：
```cpp
explicit unreliable(socket_type socket, std::optional<endpoint_type> remote_endpoint = std::nullopt);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `socket_type` | 已构造的 UDP socket |
| `remote_endpoint` | `std::optional<endpoint_type>` | 远程端点（可选） |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_unreliable(socket)` 工厂函数

**涉及的知识域**：[[ref/network/udp|UDP]]

### 成员函数

---

#### executor()

**功能说明**：返回底层 socket 关联的执行器。

**签名**：
```cpp
executor_type executor() const override;
```

**参数**：无

**返回值**：`executor_type` — socket 执行器

**调用（向下）**：`socket_.get_executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] 框架

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### set_remote_endpoint()

**功能说明**：设置发送操作的目标端点。设置后所有发送指向该端点，接收操作验证来源是否匹配，不匹配则丢弃。

**签名**：
```cpp
void set_remote_endpoint(const endpoint_type &endpoint);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `endpoint` | `const endpoint_type &` | 远程端点 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：协议层建立 UDP 连接后设置目标地址

**涉及的知识域**：[[ref/network/udp|UDP]]

---

#### remote_endpoint()

**功能说明**：返回当前设置的远程端点。如果未设置则返回空。

**签名**：
```cpp
std::optional<endpoint_type> remote_endpoint() const noexcept;
```

**参数**：无

**返回值**：`std::optional<endpoint_type>` — 远程端点（如果已设置）

**调用（向下）**：无

**被调用（向上）**：协议层查询当前目标地址

**涉及的知识域**：[[ref/network/udp|UDP]]

---

#### async_read_some()

**功能说明**：调用 `async_receive_from` 接收数据报，自动过滤非远程端点的数据报。如果尚未设置远程端点，首次接收的来源自动设为远程端点。

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

**调用（向下）**：`socket_.async_receive_from()`

**被调用（向上）**：[[channel/transport/transmission|transmission]] 基类的 `async_read()` 循环

**涉及的知识域**：[[ref/network/udp|UDP]], [[ref/programming/boost-asio|Boost.Asio]]

**async_read_some 过滤逻辑**：
1. 调用 `async_receive_from` 接收数据报
2. 如果未设置远程端点 → 首次接收的来源自动设为远程端点
3. 如果来源匹配远程端点 → 返回数据
4. 来源不匹配 → 丢弃并继续等待（回到步骤 1）

---

#### async_write_some()

**功能说明**：调用 `async_send_to` 向远程端点发送数据报。如果未设置远程端点则返回 `io_error` 错误。

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

**调用（向下）**：`socket_.async_send_to()`

**被调用（向上）**：协议层发送 UDP 数据

**涉及的知识域**：[[ref/network/udp|UDP]], [[ref/programming/boost-asio|Boost.Asio]]

---

#### async_write()

**功能说明**：UDP 数据报一次发送完成，无需循环。直接委托给 `async_write_some`。

**签名**：
```cpp
auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 写入的字节数

**调用（向下）**：`async_write_some()`

**被调用（向上）**：[[channel/transport/transmission|transmission]] `async_write()` 被重写

**涉及的知识域**：[[ref/network/udp|UDP]]

---

#### close()

**功能说明**：关闭底层 UDP socket，所有未完成的异步操作将被取消。

**签名**：
```cpp
void close() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`socket_.close()`

**被调用（向上）**：会话结束时调用

**涉及的知识域**：[[ref/network/udp|UDP]]

---

#### cancel()

**功能说明**：取消所有未完成的异步操作，被取消的操作返回 `operation_canceled`。

**签名**：
```cpp
void cancel() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`socket_.cancel()`

**被调用（向上）**：超时处理或会话中断

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### native_socket() (非 const)

**功能说明**：返回底层 UDP socket 的引用。

**签名**：
```cpp
socket_type &native_socket() noexcept;
```

**参数**：无

**返回值**：`socket_type &` — socket 引用

**调用（向下）**：无

**被调用（向上）**：设置 socket 选项、绑定端口

**涉及的知识域**：[[ref/network/udp|UDP]]

---

#### native_socket() (const)

**功能说明**：返回底层 UDP socket 的常量引用。

**签名**：
```cpp
const socket_type &native_socket() const noexcept;
```

**参数**：无

**返回值**：`const socket_type &` — socket 常量引用

**调用（向下）**：无

**被调用（向上）**：只读场景

**涉及的知识域**：[[ref/network/udp|UDP]]

### 工厂函数

---

#### make_unreliable(executor, remote_endpoint)

**功能说明**：使用执行器创建 UDP 传输层实例。

**签名**：
```cpp
inline auto make_unreliable(net::any_io_executor executor, std::optional<net::ip::udp::endpoint> remote_endpoint = std::nullopt)
    -> shared_transmission;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `executor` | `net::any_io_executor` | 执行器 |
| `remote_endpoint` | `std::optional<net::ip::udp::endpoint>` | 远程端点（可选） |

**返回值**：`shared_transmission` — unreliable 实例

**调用（向下）**：`unreliable(executor, remote_endpoint)` 构造函数

**被调用（向上）**：UDP 代理初始化时

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### make_unreliable(socket, remote_endpoint)

**功能说明**：使用已构造的 UDP socket 创建传输层实例。

**签名**：
```cpp
inline auto make_unreliable(net::ip::udp::socket socket, std::optional<net::ip::udp::endpoint> remote_endpoint = std::nullopt)
    -> shared_transmission;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `net::ip::udp::socket` | 已构造的 UDP socket |
| `remote_endpoint` | `std::optional<net::ip::udp::endpoint>` | 远程端点（可选） |

**返回值**：`shared_transmission` — unreliable 实例

**调用（向下）**：`unreliable(socket, remote_endpoint)` 构造函数

**被调用（向上）**：UDP 代理从已有 socket 创建传输层

**涉及的知识域**：[[ref/network/udp|UDP]]
