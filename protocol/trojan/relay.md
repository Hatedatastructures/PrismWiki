---
title: "relay — Trojan 协议中继器"
source: "include/prism/protocol/trojan/relay.hpp"
module: "protocol"
type: api
tags: [protocol, trojan, relay, 中继, 装饰器, TLS, UDP]
related:
  - "[[protocol/trojan/format|format]]"
  - "[[protocol/trojan/constants|constants]]"
  - "[[protocol/trojan/config|config]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# relay.hpp

> 源码: `include/prism/protocol/trojan/relay.hpp`
> 实现: `src/prism/protocol/trojan/relay.cpp`
> 模块: [[protocol|protocol]] > trojan

## 概述

Trojan 协议中继器。实现完整的 Trojan 协议中继器，包装底层传输层并添加协议握手和数据处理功能。采用装饰器设计模式，透明地增强底层传输层的功能，支持链式组合。所有操作基于 `boost::asio::awaitable`，支持异步无阻塞处理。

协议流程：凭据读取 -> 凭据验证 -> CRLF 验证 -> 命令解析 -> 地址读取 -> 命令检查 -> 数据转发

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[protocol/trojan/format|format]] | 格式编解码 |
| 依赖 | [[protocol/trojan/constants|constants]] | 协议常量 |
| 依赖 | [[protocol/trojan/message|message]] | 请求结构 |
| 依赖 | [[protocol/trojan/config|config]] | 协议配置 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[pipeline/protocols/trojan|pipeline]] | 协议处理管道 |

## 命名空间

`psm::protocol::trojan`

---

## 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `route_callback` | `function<awaitable<pair<code, udp::endpoint>>(string_view, string_view)>` | UDP 路由回调 |
| `shared_relay` | `std::shared_ptr<relay>` | 中继器共享智能指针 |

---

## 类: relay

> 源码: `include/prism/protocol/trojan/relay.hpp:45`

### 概述

Trojan 协议中继器。采用装饰器设计模式，继承自 `transport::transmission` 提供统一的传输层接口。支持 CONNECT（需 `enable_tcp=true`）和 UDP_ASSOCIATE（需 `enable_udp=true`）命令。

### 类层次

```
channel::transport::transmission
  +-- trojan::relay (继承 transmission + enable_shared_from_this)
```

### 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg = {},
               std::function<bool(std::string_view)> credential_verifier = nullptr);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 已建立连接的底层传输层 |
| `cfg` | `const config &` | 协议配置 |
| `credential_verifier` | `function<bool(string_view)>` | 用户凭据验证回调（可选） |

---

### 函数: executor()

#### 功能说明

获取底层传输层的执行器。

#### 签名

```cpp
executor_type executor() const override;
```

#### 参数

无

#### 返回值

`executor_type` — 执行器。

#### 调用（向下）

`next_layer_->executor()`

#### 被调用（向上）

- Boost.Asio 协程调度

#### 知识域

Boost.Asio 执行器

---

### 函数: async_read_some()

#### 功能说明

透传到底层传输层的异步读取。

#### 签名

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<std::size_t>` — 读取字节数。

#### 调用（向下）

`next_layer_->async_read_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] 数据转发

#### 知识域

协程异步 I/O

---

### 函数: async_write_some()

#### 功能说明

透传到底层传输层的异步写入。

#### 签名

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<std::size_t>` — 写入字节数。

#### 调用（向下）

`next_layer_->async_write_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] 数据转发

#### 知识域

协程异步 I/O

---

### 函数: close()

#### 功能说明

关闭底层传输层。

#### 签名

```cpp
void close() override;
```

#### 调用（向下）

`next_layer_->close()`

#### 被调用（向上）

- session 析构

---

### 函数: cancel()

#### 功能说明

取消所有未完成的异步操作。

#### 签名

```cpp
void cancel() override;
```

#### 调用（向下）

`next_layer_->cancel()`

#### 被调用（向上）

- session 超时处理

---

### 函数: handshake()

#### 功能说明

执行完整的 Trojan 协议握手流程。状态机：读取 56 字节凭据 -> 调用验证器 -> 读取 CRLF -> 解析命令和地址类型 -> 读取目标地址和端口 -> 检查命令是否允许。

#### 签名

```cpp
auto handshake() const -> net::awaitable<std::pair<fault::code, request>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, request>>` — 错误码和请求信息。

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取数据
- [[protocol/trojan/format|format]] `parse_credential()` — 解析凭据
- [[protocol/trojan/format|format]] `parse_crlf()` — 验证分隔符
- [[protocol/trojan/format|format]] `parse_cmd_atyp()` — 解析命令
- [[protocol/trojan/format|format]] `parse_ipv4()` / `parse_ipv6()` / `parse_domain()` — 解析地址
- [[protocol/trojan/format|format]] `parse_port()` — 解析端口
- `verifier_()` — 凭据验证回调

#### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] 协议处理管道

#### 知识域

Trojan 协议握手、SHA224 凭据

---

### 函数: next_layer()（非常量/常量）

#### 功能说明

获取底层传输层引用。

#### 签名

```cpp
psm::channel::transport::transmission &next_layer() noexcept;
const psm::channel::transport::transmission &next_layer() const noexcept;
```

#### 调用（向下）

无

#### 被调用（向上）

- pipeline 层操作底层传输

---

### 函数: release()

#### 功能说明

释放底层传输层所有权。释放后 relay 不再持有传输层，不应再调用其方法。

#### 签名

```cpp
shared_transmission release();
```

#### 返回值

`shared_transmission` — 底层传输层指针。

#### 调用（向下）

`std::move(next_layer_)`

#### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] 握手完成后转移所有权

#### 知识域

所有权转移

---

### 函数: async_associate()（公有）

#### 功能说明

处理 UDP_ASSOCIATE 命令。进入 UDP over TLS 模式，从 TLS 流读取封装的 UDP 数据包，解析目标地址后通过 UDP socket 转发，并将响应封装回 TLS 流。支持空闲超时机制。

#### 签名

```cpp
auto async_associate(route_callback route_cb) const -> net::awaitable<fault::code>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `route_cb` | `route_callback` | 路由回调函数 |

#### 返回值

`net::awaitable<fault::code>` — 异步操作结果。

#### 调用（向下）

- `udp_frame_loop()` — UDP 帧处理循环
- [[protocol/common/udp_relay|udp_relay]] `relay_udp_packet()` — 转发数据包

#### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] UDP 命令处理

#### 知识域

UDP over TLS、Trojan UDP 帧格式

---

### 函数: udp_frame_loop()（私有）

#### 功能说明

从 TLS 流读取 UDP 数据包，解析并转发到目标，然后将响应封装回 TLS 流。支持空闲超时。

#### 签名

```cpp
auto udp_frame_loop(route_callback &route_cb, net::steady_timer &idle_timer) const
    -> net::awaitable<void>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `route_cb` | `route_callback &` | 路由回调 |
| `idle_timer` | `net::steady_timer &` | 空闲超时计时器 |

#### 返回值

`net::awaitable<void>`

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取 UDP 帧
- [[protocol/trojan/format|format]] `parse_udp_packet()` — 解析 UDP 包
- `route_cb()` — 路由查询
- [[protocol/common/udp_relay|udp_relay]] `relay_udp_packet()` — 转发
- [[protocol/trojan/format|format]] `build_udp_packet()` — 构建响应

#### 被调用（向上）

- `async_associate()` UDP 关联

#### 知识域

UDP over TLS、协程并发

---

## 工厂函数: make_relay()

### 功能说明

工厂函数，创建 Trojan 协议中继器实例。封装 `std::make_shared` 调用，返回 `shared_relay`。

### 签名

```cpp
inline shared_relay make_relay(shared_transmission next_layer, const config &cfg = {},
                               std::function<bool(std::string_view)> credential_verifier = nullptr);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层 |
| `cfg` | `const config &` | 协议配置 |
| `credential_verifier` | `function<bool(string_view)>` | 凭据验证回调 |

### 返回值

`shared_relay` — 中继器共享指针。

### 调用（向下）

- `std::make_shared<relay>(next_layer, cfg, credential_verifier)`

### 被调用（向上）

- [[pipeline/protocols/trojan|pipeline]] 创建中继器

### 知识域

- [[protocol/trojan/relay|relay]] Trojan 中继器
- [[pipeline/protocols/trojan|pipeline]] 协议处理管道

## 线程安全

单个实例非线程安全，应在同一协程或 strand 内使用。

## 相关页面

- [[protocol/trojan/format|format]] — 格式编解码
- [[protocol/trojan/constants|constants]] — 协议常量
- [[protocol/trojan/config|config]] — 协议配置
- [[protocol/trojan/message|message]] — 消息结构
