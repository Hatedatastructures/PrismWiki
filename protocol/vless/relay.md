---
title: "relay — VLESS 协议中继器"
source: "include/prism/protocol/vless/relay.hpp"
module: "protocol"
type: api
tags: [protocol, vless, relay, 中继, 装饰器, UUID, TLS]
related:
  - "[[protocol/vless/format|format]]"
  - "[[protocol/vless/constants|constants]]"
  - "[[protocol/vless/config|config]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# relay.hpp

> 源码: `include/prism/protocol/vless/relay.hpp`
> 实现: `src/prism/protocol/vless/relay.cpp`
> 模块: [[protocol|protocol]] > vless

## 概述

VLESS 协议中继器。实现 VLESS 协议的中继器，包装底层传输层并添加协议握手和 UUID 验证功能。VLESS 协议运行在 TLS 内层，通过 UUID 进行用户认证。认证逻辑通过 `verifier` 回调委托给 pipeline 层，与 `account::directory` 对接。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[protocol/vless/format|format]] | 格式编解码 |
| 依赖 | [[protocol/vless/constants|constants]] | 协议常量 |
| 依赖 | [[protocol/vless/message|message]] | 请求结构 |
| 依赖 | [[protocol/vless/config|config]] | 协议配置 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[pipeline/protocols/vless|pipeline]] | 协议处理管道 |

## 命名空间

`psm::protocol::vless`

---

## 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `route_callback` | `function<awaitable<pair<code, udp::endpoint>>(string_view, string_view)>` | UDP 路由回调 |
| `shared_relay` | `std::shared_ptr<relay>` | 中继器共享智能指针 |

---

## 类: relay

> 源码: `include/prism/protocol/vless/relay.hpp:37`

### 概述

VLESS 协议中继器。采用装饰器设计模式，继承自 `transport::transmission`。认证通过 `verifier` 回调实现，pipeline 层传入的回调负责查询 `account::directory` 并获取连接租约。

### 类层次

```
channel::transport::transmission
  +-- vless::relay (继承 transmission + enable_shared_from_this)
```

### 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg = {},
               std::function<bool(std::string_view)> verifier = nullptr);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层智能指针 |
| `cfg` | `const config &` | VLESS 协议配置 |
| `verifier` | `function<bool(string_view)>` | UUID 验证回调，为 nullptr 时跳过认证 |

---

### 函数: handshake()

#### 功能说明

执行 VLESS 协议握手。从传输层读取并解析 VLESS 请求头，通过 `verifier` 回调验证 UUID，发送响应。数据通过 preview 回放，读取即消费，不会残留。

流程：版本读取(0x00) -> UUID 读取(16B) -> UUID 验证 -> 附加信息跳过 -> 命令解析 -> 地址读取

#### 签名

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, request>>` — 错误码和请求信息。

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取数据
- `verifier_()` — UUID 验证回调
- [[protocol/vless/format|format]] `parse_request()` — 解析请求
- [[protocol/vless/format|format]] `make_response()` — 构建响应

#### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 协议处理管道

#### 知识域

VLESS 协议握手、UUID 认证

---

### 函数: async_associate()

#### 功能说明

处理 UDP 命令。进入 UDP over TLS 模式，从 TLS 流读取封装的 UDP 数据包，解析目标地址后通过 UDP socket 转发，并将响应封装回 TLS 流。支持空闲超时机制。

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

- [[pipeline/protocols/vless|pipeline]] UDP 命令处理

#### 知识域

UDP over TLS、VLESS UDP 帧格式

---

### 函数: executor()

#### 功能说明

透传至底层传输层，获取执行器。

#### 签名

```cpp
executor_type executor() const override;
```

#### 参数

无

#### 返回值

`executor_type` — 执行器。

#### 调用（向下）

- `next_layer_->executor()`

#### 被调用（向上）

- Boost.Asio 协程调度

#### 知识域

- [[channel/transport/transmission|transmission]] Boost.Asio 执行器

---

### 函数: async_read_some()

#### 功能说明

透传至底层传输层的异步读取。

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

- `next_layer_->async_read_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 数据转发

#### 知识域

- [[channel/transport/transmission|transmission]] 协程异步 I/O

---

### 函数: async_write_some()

#### 功能说明

透传至底层传输层的异步写入。

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

- `next_layer_->async_write_some(buffer, ec)`

#### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 数据转发

#### 知识域

- [[channel/transport/transmission|transmission]] 协程异步 I/O

---

### 函数: close()

#### 功能说明

透传至底层传输层，关闭连接。

#### 签名

```cpp
void close() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

- `next_layer_->close()`

#### 被调用（向上）

- session 析构

#### 知识域

- [[channel/transport/transmission|transmission]] 连接生命周期

---

### 函数: cancel()

#### 功能说明

透传至底层传输层，取消所有未完成的异步操作。

#### 签名

```cpp
void cancel() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

- `next_layer_->cancel()`

#### 被调用（向上）

- session 超时处理

#### 知识域

- [[channel/transport/transmission|transmission]] 异步操作取消

---

### 函数: next_layer()

#### 功能说明

获取底层传输层引用。

#### 签名

```cpp
psm::channel::transport::transmission &next_layer() noexcept;
const psm::channel::transport::transmission &next_layer() const noexcept;
```

#### 参数

无

#### 返回值

`transmission &` / `const transmission &` — 底层传输层引用。

#### 调用（向下）

无

#### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 操作底层传输

#### 知识域

- [[channel/transport/transmission|transmission]] 装饰器访问

---

### 函数: release()

#### 功能说明

释放底层传输层所有权。释放后 relay 不再持有传输层，不应再调用其方法。

#### 签名

```cpp
shared_transmission release();
```

#### 参数

无

#### 返回值

`shared_transmission` — 底层传输层指针。

#### 调用（向下）

- `std::move(next_layer_)`

#### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 握手完成后转移所有权

#### 知识域

- [[channel/transport/transmission|transmission]] 所有权转移

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
- [[protocol/vless/format|format]] `parse_udp_packet()` — 解析 UDP 包
- `route_cb()` — 路由查询
- [[protocol/common/udp_relay|udp_relay]] `relay_udp_packet()` — 转发
- [[protocol/vless/format|format]] `build_udp_packet()` — 构建响应

#### 被调用（向上）

- `async_associate()` UDP 关联

#### 知识域

- [[protocol/common/udp_relay|udp_relay]] UDP over TLS
- [[protocol/vless/format|format]] VLESS UDP 帧格式

---

## 工厂函数: make_relay()

### 功能说明

工厂函数，创建 VLESS 协议中继器实例。封装 `std::make_shared` 调用，返回 `shared_relay`。

### 签名

```cpp
inline shared_relay make_relay(shared_transmission next_layer, const config &cfg = {},
                               std::function<bool(std::string_view)> verifier = nullptr);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层智能指针 |
| `cfg` | `const config &` | VLESS 协议配置 |
| `verifier` | `function<bool(string_view)>` | UUID 验证回调，为 nullptr 时跳过认证 |

### 返回值

`shared_relay` — VLESS 中继器共享指针。

### 调用（向下）

- `std::make_shared<relay>(next_layer, cfg, verifier)`

### 被调用（向上）

- [[pipeline/protocols/vless|pipeline]] 创建中继器

### 知识域

- [[protocol/vless/relay|relay]] VLESS 中继器
- [[pipeline/protocols/vless|pipeline]] 协议处理管道

## 相关页面

- [[protocol/vless/format|format]] — 格式编解码
- [[protocol/vless/constants|constants]] — 协议常量
- [[protocol/vless/config|config]] — 协议配置
- [[protocol/vless/message|message]] — 消息结构
