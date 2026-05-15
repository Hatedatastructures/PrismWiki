---
title: "stream — SOCKS5 协议中继器"
source: "include/prism/protocol/socks5/stream.hpp"
module: "protocol"
type: api
tags: [protocol, socks5, stream, relay, 中继, rfc1928, 协程]
related:
  - "[[protocol/socks5/constants|constants]]"
  - "[[protocol/socks5/wire|wire]]"
  - "[[protocol/socks5/config|config]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# stream.hpp

> 源码: `include/prism/protocol/socks5/stream.hpp`
> 模块: [[protocol|protocol]] > socks5

## 概述

SOCKS5 协议中继器 (RFC 1928)。实现完整的 SOCKS5 协议服务端中继器，提供协程友好的高级 API。该类继承自 `transport::transmission`，将底层传输层包装为 SOCKS5 协议中继，处理握手、认证、请求解析和响应生成。

核心特性：
- 协议完整性：支持 CONNECT、UDP_ASSOCIATE 和 BIND 命令
- 地址类型全面：支持 IPv4、IPv6 和域名
- 错误处理完善：使用 `fault::code` 错误码系统
- 能力控制：通过 `config` 结构控制命令启用状态
- 用户认证：支持 RFC 1929 用户名/密码认证

协议流程：方法协商 -> 请求处理 -> 命令检查 -> 响应发送 -> 数据转发

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[protocol/socks5/constants|constants]] | 协议常量 |
| 依赖 | [[protocol/socks5/message|message]] | 请求结构 |
| 依赖 | [[protocol/socks5/wire|wire]] | 线级解析 |
| 依赖 | [[protocol/socks5/config|config]] | 协议配置 |
| 依赖 | [[agent/account/directory|directory]] | 账户认证 |
| 依赖 | [[crypto/sha224|sha224]] | 密码哈希 |
| 依赖 | [[fault|fault]] | 错误码 |
| 依赖 | [[memory|memory]] | PMR 容器 |
| 被依赖 | [[pipeline/protocols/socks5|pipeline]] | 协议处理管道 |

## 命名空间

`psm::protocol::socks5`

---

## 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `route_callback` | `function<awaitable<pair<code, udp::endpoint>>(string_view, string_view)>` | UDP 路由回调，用于根据目标地址选择本地端点 |
| `shared_relay` | `std::shared_ptr<relay>` | 中继器共享智能指针 |

---

## 类: relay

> 源码: `include/prism/protocol/socks5/stream.hpp:58`

### 概述

SOCKS5 协议中继器。将底层传输层封装为完整的 SOCKS5 协议中继，提供协程友好的高层 API。该类实现了 SOCKS5 协议的服务端逻辑，包括方法协商、请求处理和响应生成。

### 类层次

```
channel::transport::transmission
  +-- socks5::relay (继承 transmission + enable_shared_from_this)
```

### 设计意图

- relay 对象持有 `next_layer_` 的独占所有权，生命周期与 relay 对象绑定
- 调用 `close()` 后 `next_layer_` 仍有效，可再次使用
- 析构时通过 `unique_ptr` 自动释放底层资源
- 可以通过 `release()` 提前转移所有权

### 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg = {},
               psm::agent::account::directory *account_dir = nullptr);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 已建立连接的底层传输层智能指针 |
| `cfg` | `const config &` | SOCKS5 协议配置 |
| `account_dir` | `psm::agent::account::directory *` | 账户目录指针，用于认证验证（可为空） |

---

### 函数: executor()

#### 功能说明

返回底层传输层的执行器，用于协程调度和异步操作。

#### 签名

```cpp
executor_type executor() const override;
```

#### 参数

无

#### 返回值

`executor_type` — 底层传输层的执行器。

#### 调用（向下）

`next_layer_->executor()`

#### 被调用（向上）

- Boost.Asio 协程调度

#### 知识域

Boost.Asio 执行器模型

---

### 函数: async_read_some()

#### 功能说明

握手成功后，从底层传输层读取数据。直接透传到底层传输层的 `async_read_some` 方法。

#### 签名

```cpp
auto async_read_some(const std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

#### 返回值

`net::awaitable<std::size_t>` — 异步操作，完成后返回读取的字节数。

#### 调用（向下）

`next_layer_->async_read_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 数据转发阶段

#### 知识域

协程异步 I/O

---

### 函数: async_write_some()

#### 功能说明

握手成功后，向底层传输层写入数据。直接透传到底层传输层的 `async_write_some` 方法。

#### 签名

```cpp
auto async_write_some(const std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

#### 返回值

`net::awaitable<std::size_t>` — 异步操作，完成后返回写入的字节数。

#### 调用（向下）

`next_layer_->async_write_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 数据转发阶段

#### 知识域

协程异步 I/O

---

### 函数: close()

#### 功能说明

关闭底层传输层连接，释放网络资源。调用后 `next_layer_` 指针仍然有效，但连接已断开。

#### 签名

```cpp
void close() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

`next_layer_->close()`

#### 被调用（向上）

- session 析构

#### 知识域

资源生命周期管理

---

### 函数: cancel()

#### 功能说明

取消底层传输层上所有待处理的异步操作，触发操作以 `operation_aborted` 错误码完成。

#### 签名

```cpp
void cancel() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

`next_layer_->cancel()`

#### 被调用（向上）

- session 超时/错误处理

#### 知识域

异步操作取消机制

---

### 函数: handshake()

#### 功能说明

执行完整的 SOCKS5 协议握手流程，包括方法协商、请求解析和命令检查。握手过程首先进行方法协商：读取客户端支持的方法列表，并选择认证方式（无认证或密码认证）；协商成功后进入请求解析阶段，读取命令、地址类型和目标地址，并根据配置检查命令是否允许。

命令处理规则：
- `connect` 命令：要求 `enable_tcp` 为 `true`，`form` 设为 `stream`
- `udp_associate` 命令：要求 `enable_udp` 为 `true`，`form` 设为 `datagram`
- `bind` 命令：要求 `enable_bind` 为 `true`，`form` 设为 `stream`

失败行为分类：协议错误会发送 SOCKS5 错误响应并返回错误码；网络错误直接返回错误码，不发送响应。

#### 签名

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, request>>` — 异步操作，返回错误码和请求信息。

#### 调用（向下）

- `negotiated_authentication()` — 方法协商和认证
- `read_request_header()` — 读取请求头部
- [[protocol/socks5/wire|wire]] `parse_header()` — 解析头部
- `read_address<4/16>()` / `read_domain_address()` — 读取地址
- `async_write_error()` — 发送错误响应

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 协议处理管道

#### 知识域

RFC 1928 SOCKS5 协议握手流程

---

### 函数: async_write_success()

#### 功能说明

构建并发送 SOCKS5 成功响应，包含绑定地址和端口信息。响应格式遵循 RFC 1928 规范。

#### 签名

```cpp
auto async_write_success(const request &info) const -> net::awaitable<fault::code>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `info` | `const request &` | 请求信息，用于回显绑定地址和端口 |

#### 返回值

`net::awaitable<fault::code>` — 异步操作，完成后返回错误码。

#### 调用（向下）

- `build_success_response()` — 构建响应报文
- `async_write_impl()` — 写入底层

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 连接成功后发送

#### 知识域

RFC 1928 响应格式

---

### 函数: async_write_error()

#### 功能说明

构建并发送 SOCKS5 错误响应，使用固定格式的错误报文。响应中地址字段填充为零。

#### 签名

```cpp
auto async_write_error(reply_code code) const -> net::awaitable<fault::code>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `code` | `reply_code` | SOCKS5 错误码 |

#### 返回值

`net::awaitable<fault::code>` — 异步操作，完成后返回错误码。

#### 调用（向下）

`async_write_impl()` — 写入底层

#### 被调用（向上）

- `handshake()` 握手失败时
- `async_associate()` UDP 关联失败时

#### 知识域

RFC 1928 错误响应格式

---

### 函数: next_layer()（非常量）

#### 功能说明

返回底层传输层的可变引用，用于直接操作底层连接。

#### 签名

```cpp
psm::channel::transport::transmission &next_layer() noexcept;
```

#### 参数

无

#### 返回值

`transmission &` — 底层传输层引用。

#### 调用（向下）

无

#### 被调用（向上）

- pipeline 层直接操作底层传输

#### 知识域

传输层抽象

---

### 函数: next_layer()（常量）

#### 功能说明

返回底层传输层的只读引用，用于查询底层连接状态。

#### 签名

```cpp
const psm::channel::transport::transmission &next_layer() const noexcept;
```

#### 参数

无

#### 返回值

`const transmission &` — 底层传输层常量引用。

#### 调用（向下）

无

#### 被调用（向上）

- 状态查询

#### 知识域

传输层抽象

---

### 函数: is_valid()

#### 功能说明

检查 `next_layer_` 指针是否有效，用于判断是否可以安全调用读写方法。

#### 签名

```cpp
[[nodiscard]] bool is_valid() const noexcept;
```

#### 参数

无

#### 返回值

`bool` — `true` 表示有效，`false` 表示已被 `release()` 转移。

#### 调用（向下）

无

#### 被调用（向上）

- pipeline 层检查传输层状态

#### 知识域

生命周期管理

---

### 函数: release()

#### 功能说明

释放底层传输层的所有权并返回指针。释放后 `is_valid()` 返回 `false`，不应再调用读写方法。用于将底层连接转移给其他组件管理。

#### 签名

```cpp
shared_transmission release();
```

#### 参数

无

#### 返回值

`shared_transmission` — 底层传输层指针。

#### 调用（向下）

`std::move(next_layer_)`

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 握手完成后转移所有权给 tunnel

#### 知识域

所有权转移（move semantics）

---

### 函数: async_associate()（公有）

#### 功能说明

处理客户端发起的 UDP 关联请求，绑定本地端口并返回关联地址。成功后进入 UDP 数据报转发循环，直到控制连接关闭或空闲超时。

#### 签名

```cpp
auto async_associate(const request &request_info, route_callback route_callback) const
    -> net::awaitable<fault::code>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `request_info` | `const request &` | 包含请求信息的 SOCKS5 请求结构体 |
| `route_callback` | `route_callback` | 路由回调函数 |

#### 返回值

`net::awaitable<fault::code>` — 异步操作，完成后返回错误码。

#### 调用（向下）

- `bind_datagram_port()` — 绑定 UDP 端口
- `async_write_associate_success()` — 发送成功响应
- `associate_loop()` — UDP 数据报转发主循环
- `wait_control_close()` — 监听控制面关闭

#### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] UDP 命令处理

#### 知识域

SOCKS5 UDP_ASSOCIATE、RFC 1928 Section 6

---

### 函数: negotiated_authentication()（私有）

#### 功能说明

读取客户端发送的方法协商请求，验证协议版本，根据配置和客户端支持的方法选择认证方式。当 `enable_auth` 为 `true` 且账户目录可用时，优先选择用户名/密码认证 (0x02)；否则选择无认证 (0x00)。

#### 签名

```cpp
auto negotiated_authentication() -> net::awaitable<std::pair<fault::code, auth_method>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, auth_method>>` — 协商结果错误码与选定的认证方法。

#### 调用（向下）

- `async_read_impl()` — 读取协商数据
- `async_write_impl()` — 发送协商响应
- `perform_password_auth()` — 执行密码认证

#### 被调用（向上）

- `handshake()` 握手流程

#### 知识域

RFC 1928 方法协商、RFC 1929 认证

---

### 函数: perform_password_auth()（私有）

#### 功能说明

读取客户端发送的用户名/密码认证请求，解析后使用 SHA224 对密码进行哈希，通过 `account::directory` 验证凭证并获取连接租约。认证失败时不暴露具体原因。

#### 签名

```cpp
auto perform_password_auth() -> net::awaitable<std::pair<fault::code, bool>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, bool>>` — 错误码与认证结果。

#### 调用（向下）

- `async_read_impl()` — 读取认证数据
- [[protocol/socks5/wire|wire]] `parse_password_auth()` — 解析认证请求
- [[protocol/socks5/wire|wire]] `build_password_auth_response()` — 构建响应
- [[crypto/sha224|sha224]] — 密码哈希
- [[agent/account/directory|directory]] `try_acquire()` — 验证凭证

#### 被调用（向上）

- `negotiated_authentication()` 认证流程

#### 知识域

RFC 1929 用户名/密码认证、SHA224 哈希

---

### 函数: associate_loop()（私有）

#### 功能说明

UDP_ASSOCIATE 主循环。持续读取客户端发往 ingress 的 UDP 数据报，并逐包转发。当 socket 被取消时（控制面关闭触发）或空闲超时时，协程退出。

#### 签名

```cpp
auto associate_loop(net::ip::udp::socket &ingress_socket, route_callback &route_callback,
                    net::steady_timer &idle_timer) const -> net::awaitable<void>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `ingress_socket` | `net::ip::udp::socket &` | 入站 UDP socket |
| `route_callback` | `route_callback &` | 路由回调函数 |
| `idle_timer` | `net::steady_timer &` | 空闲超时计时器 |

#### 返回值

`net::awaitable<void>` — 异步操作。

#### 调用（向下）

- `ingress_socket.async_receive_from()` — 接收 UDP 数据报
- `relay_single_datagram()` — 转发单个数据报

#### 被调用（向上）

- `async_associate()` UDP 关联主循环

#### 知识域

UDP 中继、协程并发

---

### 函数: relay_single_datagram()（私有）

#### 功能说明

处理单个 SOCKS5 UDP 数据报的转发。流程包括解码 SOCKS5 UDP 报头、调用路由回调解析目标端点、发送 payload 到目标并等待回包、将回包重新封装为 SOCKS5 UDP 数据报回写客户端。

#### 签名

```cpp
auto relay_single_datagram(net::ip::udp::socket &ingress_socket,
                           std::span<const std::byte> ingress_packet,
                           const net::ip::udp::endpoint &client_endpoint,
                           route_callback &route_callback,
                           memory::vector<std::byte> &target_buffer) const
    -> net::awaitable<void>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `ingress_socket` | `net::ip::udp::socket &` | 入站 UDP socket |
| `ingress_packet` | `std::span<const std::byte>` | 入站数据包 |
| `client_endpoint` | `const net::ip::udp::endpoint &` | 客户端端点 |
| `route_callback` | `route_callback &` | 路由回调函数 |
| `target_buffer` | `memory::vector<std::byte> &` | 目标缓冲区 |

#### 返回值

`net::awaitable<void>` — 异步操作。

#### 调用（向下）

- [[protocol/socks5/wire|wire]] `decode_udp_header()` — 解码 UDP 报头
- `route_callback()` — 路由查询
- `egress_socket.async_send_to()` — 发送到目标
- `egress_socket.async_receive_from()` — 接收响应
- [[protocol/socks5/wire|wire]] `encode_udp_datagram()` — 编码响应数据报

#### 被调用（向上）

- `associate_loop()` 主循环

#### 知识域

SOCKS5 UDP 数据报转发、双 socket 模型

---

### 函数: build_success_response()（私有静态）

#### 功能说明

构建符合 RFC 1928 规范的成功响应报文。格式为 VER(1) + REP(1) + RSV(1) + ATYP(1) + BND.ADDR(变长) + BND.PORT(2)。

#### 签名

```cpp
static auto build_success_response(const request &req, std::span<std::uint8_t> buffer) -> std::size_t;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `req` | `const request &` | 请求信息 |
| `buffer` | `std::span<std::uint8_t>` | 输出缓冲区（至少 262 字节） |

#### 返回值

`std::size_t` — 实际写入的字节数。

#### 调用（向下）

无（纯数据构造）

#### 被调用（向上）

- `async_write_success()` 构建响应

#### 知识域

RFC 1928 响应格式

---

## 工厂函数: make_relay()

### 功能说明

工厂函数，封装 `std::make_shared` 调用，简化对象创建。

### 签名

```cpp
inline shared_relay make_relay(shared_transmission next_layer, const config &cfg = {},
                               psm::agent::account::directory *account_dir = nullptr);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层指针 |
| `cfg` | `const config &` | SOCKS5 协议配置 |
| `account_dir` | `psm::agent::account::directory *` | 账户目录指针 |

### 返回值

`shared_relay` — 中继器对象共享指针。

### 调用（向下）

`std::make_shared<relay>(...)`

### 被调用（向上）

- [[pipeline/protocols/socks5|pipeline]] 创建中继器

### 知识域

工厂模式、智能指针

## 线程安全

实例非线程安全，应在同一协程或 strand 内使用。

## 相关页面

- [[protocol/socks5/constants|constants]] — 协议常量
- [[protocol/socks5/wire|wire]] — 线级解析
- [[protocol/socks5/config|config]] — 协议配置
- [[protocol/socks5/message|message]] — 消息结构
