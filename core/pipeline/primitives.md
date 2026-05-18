---
title: "primitives.hpp — 管道原语定义"
layer: core
source: "I:/code/Prism/include/prism/pipeline/primitives.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# primitives.hpp

> 源码: `include/prism/pipeline/primitives.hpp`
> 实现: `src/prism/pipeline/primitives.cpp`
> 模块: Pipeline / primitives

## 概述

定义协议管道共享的通用原语组件，包括连接关闭、预读回放、上游拨号、TLS 握手以及双向隧道转发等核心功能。这些原语为 HTTP、SOCKS5、Trojan、VLESS、SS2022 等具体协议处理提供底层支撑。

## 命名空间

`psm::pipeline::primitives`

## 核心类型定义

```cpp
using ssl_connector = channel::connector;
using ssl_stream = ssl::stream<ssl_connector>;
using shared_ssl_stream = std::shared_ptr<ssl_stream>;
using shared_transmission = channel::transport::shared_transmission;
```

---

## 函数: shut_close() (裸指针重载)

> 源码位置: `primitives.hpp:49`

### 功能

安全地关闭裸指针指向的传输对象，若指针为空则不做任何操作。

### 签名

```cpp
inline void shut_close(psm::channel::transport::transmission *trans) noexcept;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| trans | `transmission *` | 传输对象的裸指针，可为空 |

### 实现逻辑

```cpp
inline void shut_close(psm::channel::transport::transmission *trans) noexcept
{
    if (trans)
    {
        trans->shutdown_write();  // 半关闭：关闭写入方向
        trans->close();           // 完全关闭连接
    }
}
```

### 调用链

- 调用: `transmission::shutdown_write()` → `transmission::close()`
- 被调用: `tunnel()` → [[core/pipeline/primitives#tunnel|tunnel]]

---

## 函数: shut_close() (智能指针重载)

> 源码位置: `primitives.hpp:65`

### 功能

关闭并释放智能指针持有的传输对象，先关闭连接再释放所有权。

### 签名

```cpp
inline void shut_close(shared_transmission &trans) noexcept;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| trans | `shared_transmission &` | 持有传输对象的智能指针引用 |

### 实现逻辑

```cpp
inline void shut_close(shared_transmission &trans) noexcept
{
    if (trans)
    {
        trans->shutdown_write();
        trans->close();
        trans.reset();  // 释放所有权
    }
}
```

---

## 函数: dial() (路由器重载)

> 源码位置: `primitives.hpp:87`
> 实现位置: `primitives.cpp:74`

### 功能

通过路由器拨号连接上游服务器，根据目标地址的正向或反向标记选择路由策略。

### 签名

```cpp
auto dial(resolve::router &router, std::string_view label,
          const protocol::analysis::target &target, bool allow_reverse, bool require_open)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| router | `resolve::router &` | 路由器引用 |
| label | `std::string_view` | 协议标签（日志用） |
| target | `const protocol::analysis::target &` | 目标地址 |
| allow_reverse | `bool` | 是否允许反向路由 |
| require_open | `bool` | 是否要求套接字已打开 |

### 返回值

`std::pair<fault::code, shared_transmission>` — 错误码和传输对象

### 调用链

- 调用: [[core/resolve/router|router::async_forward]] — 正向路由
- 调用: [[core/resolve/router|router::async_reverse]] — 反向路由
- 被调用: `forward()` → [[core/pipeline/primitives#forward|forward]]

---

## 函数: dial() (出站代理重载)

> 源码位置: `primitives.hpp:101`
> 实现位置: `primitives.cpp:305`

### 功能

通过出站代理拨号连接上游，委托给出站代理的 `async_connect` 方法。

### 签名

```cpp
auto dial(outbound::proxy &outbound_proxy, const protocol::analysis::target &target,
          const net::any_io_executor &executor)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

### 调用链

- 调用: [[outbound/proxy|proxy::async_connect]]
- 被调用: `forward()` → [[core/pipeline/primitives#forward|forward]]

---

## 函数: ssl_handshake()

> 源码位置: `primitives.hpp:115`
> 实现位置: `primitives.cpp:30`

### 功能

执行 TLS 服务端握手，将入站传输层包装为 connector 适配器再包装为 ssl_stream。

### 签名

```cpp
auto ssl_handshake(shared_transmission inbound, ssl::context &ssl_ctx)
    -> net::awaitable<std::tuple<fault::code, shared_ssl_stream, shared_transmission>>;
```

### 返回值

三元组：
- 错误码
- TLS 流（成功时非空）
- 失败时从 connector 恢复的传输层

### 调用链

- 调用: [[core/channel/adapter/connector|connector]] — 适配 Boost.Asio 流接口
- 调用: `ssl::stream::async_handshake()` — TLS 服务端握手
- 被调用: `stealth::native` — TLS 伪装方案

---

## 类: preview

> 源码位置: `primitives.hpp:128`
> 实现位置: `primitives.cpp:121`

### 功能

预读数据回放包装器。在协议嗅探阶段已读取的数据保存在内部缓冲区，后续读取时优先返回预读数据，耗尽后委托给内部传输对象。

### 继承关系

继承自 `channel::transport::transmission` 抽象基类。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `shared_transmission` | `inner_` | 内部传输对象 |
| `memory::vector<std::byte>` | `preread_buffer_` | 预读数据缓冲区 |
| `std::size_t` | `offset_` | 当前预读偏移量 |

### 主要方法

#### preview() 构造函数

```cpp
explicit preview(shared_transmission inner, std::span<const std::byte> preread,
                 memory::resource_pointer mr = memory::current_resource());
```

将预读数据复制到内部缓冲区。

#### async_read_some()

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

优先从预读缓冲区返回数据，耗尽后委托给内部传输。

#### async_write_some()

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

委托给内部传输对象。

#### inner()

```cpp
[[nodiscard]] auto inner() const noexcept -> shared_transmission;
```

返回内部传输对象的智能指针。

---

## 函数: wrap_with_preview()

> 源码位置: `primitives.hpp:224`

### 功能

将入站传输包装为带预读数据的传输。

### 签名

```cpp
inline auto wrap_with_preview(session_context &ctx, std::span<const std::byte> data,
                              bool use_global_mr = false) -> shared_transmission;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文 |
| data | `std::span<const std::byte>` | 预读数据 |
| use_global_mr | `bool` | 是否使用全局内存池（mux 场景） |

### 实现逻辑

```cpp
inline auto wrap_with_preview(session_context &ctx, std::span<const std::byte> data,
                              bool use_global_mr = false) -> shared_transmission
{
    auto inbound = std::move(ctx.inbound);
    if (!data.empty())
    {
        auto *mr = use_global_mr ? nullptr : ctx.frame_arena.get();
        inbound = std::make_shared<preview>(std::move(inbound), data, mr);
    }
    return inbound;
}
```

### 调用链

- 被调用: [[core/pipeline/protocols/http|http]], [[core/pipeline/protocols/socks5|socks5]], [[core/pipeline/protocols/trojan|trojan]], [[core/pipeline/protocols/vless|vless]], [[core/pipeline/protocols/shadowsocks|shadowsocks]]

---

## 函数: tunnel()

> 源码位置: `primitives.hpp:251`
> 实现位置: `primitives.cpp:196`

### 功能

在两个流之间运行全双工隧道，建立双向数据转发。

### 签名

```cpp
auto tunnel(shared_transmission inbound, shared_transmission outbound,
            const session_context &ctx, bool complete_write = true)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| inbound | `shared_transmission` | 入站流（客户端侧） |
| outbound | `shared_transmission` | 出站流（上游侧） |
| ctx | `const session_context &` | 会话上下文 |
| complete_write | `bool` | 是否使用完整写入语义 |

### 流程

1. 从帧竞技场分配缓冲区，切成两半（上行/下行）
2. 启动两个并行协程：入站→出站、出站→入站
3. 使用 `awaitable_operators::operator||` 并行执行
4. 任一方向断开即终止整个隧道
5. 输出传输统计
6. 关闭两端传输连接

---

## 函数: forward()

> 源码位置: `primitives.hpp:266`
> 实现位置: `primitives.cpp:269`

### 功能

拨号连接上游并建立双向隧道，组合 `dial` + `tunnel` 操作。

### 签名

```cpp
auto forward(session_context &ctx, std::string_view label,
             const protocol::analysis::target &target,
             shared_transmission inbound) -> net::awaitable<void>;
```

### 流程

1. 检查是否配置了出站代理
2. 有出站代理: `dial(outbound_proxy)`
3. 无出站代理: `dial(router, allow_reverse=true)`
4. 拨号成功后调用 `tunnel()`

### 调用链

- 调用: [[core/pipeline/primitives#dial|dial]]
- 调用: [[core/pipeline/primitives#tunnel|tunnel]]
- 被调用: [[core/pipeline/protocols/trojan|trojan]], [[core/pipeline/protocols/vless|vless]], [[core/pipeline/protocols/shadowsocks|shadowsocks]]

---

## 函数: is_mux_target()

> 源码位置: `primitives.hpp:278`
> 实现位置: `primitives.cpp:319`

### 功能

检测目标地址是否为 mux 多路复用标记地址（以 `.mux.sing-box.arpa` 结尾）。

### 签名

```cpp
[[nodiscard]] auto is_mux_target(std::string_view host, bool mux_enabled) noexcept -> bool;
```

---

## 函数: make_datagram_router()

> 源码位置: `primitives.hpp:290`
> 实现位置: `primitives.cpp:325`

### 功能

创建 UDP 数据报路由回调函数，用于 UDP ASSOCIATE。

### 签名

```cpp
auto make_datagram_router(resolve::router &router)
    -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(
        std::string_view, std::string_view)>;
```

### 调用链

- 被调用: [[core/pipeline/protocols/socks5|socks5]], [[core/pipeline/protocols/trojan|trojan]], [[core/pipeline/protocols/vless|vless]]

---

## 函数: find_reliable()

> 源码位置: `primitives.hpp:301`

### 功能

从传输层包装链中解包找到底层 reliable 传输，穿透 snapshot/preview 等装饰层。

### 签名

```cpp
inline auto find_reliable(shared_transmission &trans) noexcept
    -> psm::channel::transport::reliable *;
```

### 实现逻辑

```cpp
inline auto find_reliable(shared_transmission &trans) noexcept
    -> psm::channel::transport::reliable *
{
    psm::channel::transport::transmission *raw = trans.get();
    while (raw)
    {
        if (auto *p = dynamic_cast<preview *>(raw))
        {
            raw = p->inner().get();
            continue;
        }
        if (auto *s = dynamic_cast<channel::transport::snapshot *>(raw))
        {
            raw = s->inner().get();
            continue;
        }
        break;
    }
    return dynamic_cast<channel::transport::reliable *>(raw);
}
```

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/channel/transport/transmission|传输层抽象]]
- [[core/resolve/router|路由器]]
- [[outbound/proxy|出站代理]]