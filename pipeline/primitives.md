---
title: "primitives.hpp — 管道原语定义"
source: "include/prism/pipeline/primitives.hpp"
module: "pipeline"
type: api
tags: [pipeline, primitives, 原语, 隧道]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/protocols/http|http]]"
  - "[[pipeline/protocols/socks5|socks5]]"
  - "[[pipeline/protocols/trojan|trojan]]"
  - "[[pipeline/protocols/vless|vless]]"
  - "[[pipeline/protocols/shadowsocks|shadowsocks]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[resolve/router|router]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[channel/adapter/connector|connector]]"
---

# primitives.hpp

> 源码: `include/prism/pipeline/primitives.hpp`
> 实现: `src/prism/pipeline/primitives.cpp`
> 模块: [[pipeline|Pipeline]]

## 概述

定义协议管道共享的通用原语组件，包括连接关闭、预读回放、上游拨号、TLS 握手以及双向隧道转发等核心功能。这些原语为 HTTP、SOCKS5、Trojan、VLESS、SS2022 等具体协议处理提供底层支撑，确保协议处理逻辑的一致性和可复用性。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[resolve/router|router]] | 路由器 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 依赖 | [[protocol/analysis|analysis]] | 目标地址结构 |
| 依赖 | [[channel/transport/transmission|transmission]] | 传输层抽象 |
| 依赖 | [[channel/transport/reliable|reliable]] | 可靠传输 |
| 依赖 | [[channel/transport/snapshot|snapshot]] | 快照传输 |
| 依赖 | [[channel/adapter/connector|connector]] | SSL 连接器 |
| 依赖 | [[outbound/proxy|proxy]] | 出站代理 |
| 被依赖 | [[pipeline/protocols/http|http]] | HTTP 管道使用原语 |
| 被依赖 | [[pipeline/protocols/socks5|socks5]] | SOCKS5 管道使用原语 |
| 被依赖 | [[pipeline/protocols/trojan|trojan]] | Trojan 管道使用原语 |
| 被依赖 | [[pipeline/protocols/vless|vless]] | VLESS 管道使用原语 |
| 被依赖 | [[pipeline/protocols/shadowsocks|shadowsocks]] | SS2022 管道使用原语 |
| 被依赖 | [[agent/dispatch/table|table]] | Unknown 处理器使用 tunnel |

## 命名空间

`psm::pipeline::primitives`

---

## 函数: shut_close()（裸指针重载）

> 源码: `include/prism/pipeline/primitives.hpp:49`

### 功能

安全地关闭裸指针指向的传输对象，若指针为空则不做任何操作。

### 签名

```cpp
inline void shut_close(psm::channel::transport::transmission *trans) noexcept;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| trans | `psm::channel::transport::transmission *` | 传输对象的裸指针，可为空 |

### 返回值

无返回值。

### 调用（向下）

- `transmission::shutdown_write()` — 关闭写入方向
- `transmission::close()` — 关闭传输连接

### 被调用（向上）

- `tunnel()` — 隧道结束后关闭两端传输
- [[agent/dispatch/table|handle_unknown]] — Unknown 处理器清理资源

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 连接关闭语义（半关闭/全关闭）

---

## 函数: shut_close()（智能指针重载）

> 源码: `include/prism/pipeline/primitives.hpp:65`

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

### 返回值

无返回值。调用后 `trans` 被 `reset()` 为空。

### 调用（向下）

- `transmission::shutdown_write()` — 关闭写入方向
- `transmission::close()` — 关闭传输连接
- `shared_ptr::reset()` — 释放所有权

### 被调用（向上）

- `tunnel()` — 隧道结束后关闭并释放两端传输

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 连接关闭语义
- [[ref/memory/pmr|PMR]] — 智能指针与共享所有权

---

## 函数: dial()（路由器重载）

> 源码: `include/prism/pipeline/primitives.hpp:87`
> 实现: `src/prism/pipeline/primitives.cpp:74`

### 功能

通过路由器拨号连接上游服务器，根据目标地址的正向或反向标记选择路由策略，并将原始套接字包装为可靠传输对象返回。

### 签名

```cpp
auto dial(resolve::router &router, std::string_view label,
          const protocol::analysis::target &target, bool allow_reverse, bool require_open)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| router | `resolve::router &` | 路由器引用，用于选择上游路由 |
| label | `std::string_view` | 协议标签，用于日志记录 |
| target | `const protocol::analysis::target &` | 解析后的上游目标地址 |
| allow_reverse | `bool` | 是否允许使用反向路由（按域名查找预配置后端） |
| require_open | `bool` | 是否要求返回的套接字已打开 |

### 返回值

`std::pair<fault::code, shared_transmission>` — 错误码和传输对象的配对。连接失败时错误码非 `success`，传输对象为 `nullptr`。

### 调用（向下）

- [[resolve/router|router::async_forward]] — 正向路由（DNS 解析 + TCP 连接）
- [[resolve/router|router::async_reverse]] — 反向路由（按域名查找预配置后端）
- `channel::transport::make_reliable()` — 将连接池连接包装为可靠传输层

### 被调用（向上）

- `forward()` — 组合 dial + tunnel 时调用
- [[pipeline/protocols/http|http]] — HTTP CONNECT/普通请求拨号
- [[pipeline/protocols/socks5|socks5]] — SOCKS5 CONNECT 命令拨号

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 连接建立
- [[ref/network/connection-pool|连接池]] — 连接池化与复用
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — IPv4/IPv6 并行连接

### 流程

1. 检查 IPv6 禁用标志，拒绝 IPv6 字面量地址
2. 根据 `allow_reverse` 和 `target.positive` 选择正向或反向路由
3. 路由失败时返回错误码
4. `require_open` 为 true 时验证连接有效性
5. 将连接池连接包装为可靠传输层返回

---

## 函数: dial()（出站代理重载）

> 源码: `include/prism/pipeline/primitives.hpp:101`
> 实现: `src/prism/pipeline/primitives.cpp:305`

### 功能

通过出站代理拨号连接上游，委托给出站代理的 `async_connect` 方法建立连接。这是新的路由路径，替代直接调用 router 的方式。

### 签名

```cpp
auto dial(outbound::proxy &outbound_proxy, const protocol::analysis::target &target,
          const net::any_io_executor &executor)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| outbound_proxy | `outbound::proxy &` | 出站代理引用 |
| target | `const protocol::analysis::target &` | 目标地址信息 |
| executor | `const net::any_io_executor &` | 用于创建连接的执行器 |

### 返回值

`std::pair<fault::code, shared_transmission>` — 错误码和传输对象的配对。

### 调用（向下）

- [[outbound/proxy|proxy::async_connect]] — 委托出站代理建立连接

### 被调用（向上）

- `forward()` — 通过出站代理转发时调用
- [[pipeline/protocols/http|http]] — HTTP 通过出站代理拨号
- [[pipeline/protocols/socks5|socks5]] — SOCKS5 通过出站代理拨号

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 连接建立
- [[ref/network/connection-pool|连接池]] — 出站代理连接管理

---

## 函数: ssl_handshake()

> 源码: `include/prism/pipeline/primitives.hpp:115`
> 实现: `src/prism/pipeline/primitives.cpp:30`

### 功能

执行 TLS 服务端握手，将入站传输层包装为 connector 适配器再包装为 ssl_stream，完成标准 TLS 握手。握手失败时从 connector 恢复传输层所有权，避免 transport 丢失。

### 签名

```cpp
auto ssl_handshake(shared_transmission inbound, ssl::context &ssl_ctx)
    -> net::awaitable<std::tuple<fault::code, shared_ssl_stream, shared_transmission>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| inbound | `shared_transmission` | 入站传输层（所有权被转移） |
| ssl_ctx | `ssl::context &` | SSL 上下文，包含证书和私钥配置 |

### 返回值

`std::tuple<fault::code, shared_ssl_stream, shared_transmission>` — 三元组：错误码、TLS 流（成功时非空）、失败时从 connector 恢复的传输层（成功时为 `nullptr`）。

### 调用（向下）

- [[channel/adapter/connector|connector]] — 将 transmission 适配为 Boost.Asio 流式接口
- `ssl::stream::async_handshake()` — 执行 TLS 服务端握手
- `connector::release()` — 握手失败时恢复传输层所有权

### 被调用（向上）

- `stealth::native` — TLS 伪装方案中执行服务端握手

### 涉及的知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[ref/crypto/x25519|X25519]] — TLS 密钥交换中使用的 X25519

### 流程

1. 检查入站传输是否有效
2. 将传输层包装为 ssl_connector 适配器
3. 创建 ssl_stream（ssl_connector + ssl_ctx）
4. 执行 TLS 服务端握手
5. 握手成功返回 TLS 流；失败时从 connector 恢复传输层

---

## 类: preview

> 源码: `include/prism/pipeline/primitives.hpp:128`
> 实现: `src/prism/pipeline/primitives.cpp:121`

### 概述

预读数据回放包装器。在协议嗅探阶段，部分数据可能已被从入站传输中读取。该包装器将这些预读数据保存在内部缓冲区，在后续读取时优先返回预读数据，待预读数据耗尽后再委托给内部传输对象。继承自 `transmission` 抽象基类，可透明地替换原始传输。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `shared_transmission` | `inner_` | 内部传输对象 |
| `memory::vector<std::byte>` | `preread_buffer_` | 预读数据缓冲区（拥有所有权） |
| `std::size_t` | `offset_` | 当前预读偏移量 |

### 设计意图

协议嗅探时读取的数据不能丢弃，需要在后续协议处理中"重放"。preview 类通过装饰器模式包装原始传输层，确保数据不丢失且对上层透明。

---

### 成员函数: preview()

#### 功能

构造预读回放包装器，将预读数据复制到内部缓冲区。

#### 签名

```cpp
explicit preview(shared_transmission inner, std::span<const std::byte> preread,
                 memory::resource_pointer mr = memory::current_resource());
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| inner | `shared_transmission` | 被包装的内部传输对象 |
| preread | `std::span<const std::byte>` | 协议嗅探期间捕获的预读数据 |
| mr | `memory::resource_pointer` | 内存资源，用于预读缓冲区分配，默认使用当前帧竞技场 |

#### 返回值

构造函数，无返回值。

#### 调用（向下）

- `memory::vector` 构造 — 使用 PMR 分配器复制预读数据

#### 被调用（向上）

- `wrap_with_preview()` — 包装入站传输时创建 preview 实例

#### 涉及的知识域

- [[ref/memory/pmr|PMR]] — 多态内存资源分配
- [[ref/memory/arena|竞技场]] — 帧竞技场分配器

---

### 成员函数: is_reliable()

#### 功能

报告内部传输是否可靠（TCP 为可靠，UDP 为不可靠）。

#### 签名

```cpp
[[nodiscard]] bool is_reliable() const noexcept override;
```

#### 参数

无参数。

#### 返回值

`bool` — 若内部传输可靠返回 `true`，否则返回 `false`。

#### 调用（向下）

- `transmission::is_reliable()` — 委托给内部传输对象

#### 被调用（向上）

- 传输层可靠性检查 — 各协议处理器判断传输类型

#### 涉及的知识域

- [[ref/network/tcp|TCP]] — 可靠传输
- [[ref/network/udp|UDP]] — 不可靠传输

---

### 成员函数: executor()

#### 功能

获取内部传输的执行器，委托给内部传输对象。

#### 签名

```cpp
[[nodiscard]] executor_type executor() const override;
```

#### 参数

无参数。

#### 返回值

`executor_type` — 绑定到内部传输的执行器。

#### 调用（向下）

- `transmission::executor()` — 委托给内部传输对象

#### 被调用（向上）

- Boost.Asio 异步操作 — 获取执行器以调度协程

#### 涉及的知识域

- [[ref/programming/boost-asio|Boost.Asio]] — 执行器模型

---

### 成员函数: async_read_some()

#### 功能

从预读缓冲区或内部流读取数据，优先返回预读数据，耗尽后委托给内部传输对象。

#### 签名

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffer | `std::span<std::byte>` | 目标缓冲区 |
| ec | `std::error_code &` | 输出错误码 |

#### 返回值

`net::awaitable<std::size_t>` — 读取的字节数。

#### 调用（向下）

- `std::memcpy` — 从预读缓冲区拷贝数据
- `transmission::async_read_some()` — 预读耗尽后委托给内部传输

#### 被调用（向上）

- `tunnel()` — 隧道转发中读取数据
- 各协议 relay — 握手阶段读取协议数据

#### 涉及的知识域

- [[ref/programming/c++23-coroutines|C++23 协程]] — 协程异步读取
- [[ref/programming/boost-asio|Boost.Asio]] — AsyncReadStream 概念

---

### 成员函数: async_write_some()

#### 功能

将数据写入内部流，委托给内部传输对象的 `async_write_some` 方法。

#### 签名

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffer | `std::span<const std::byte>` | 源数据缓冲区 |
| ec | `std::error_code &` | 输出错误码 |

#### 返回值

`net::awaitable<std::size_t>` — 写入的字节数。

#### 调用（向下）

- `transmission::async_write_some()` — 委托给内部传输

#### 被调用（向上）

- `tunnel()` — 隧道转发中写入数据

#### 涉及的知识域

- [[ref/programming/boost-asio|Boost.Asio]] — AsyncWriteStream 概念

---

### 成员函数: async_write()

#### 功能

完整写入操作，委托给内部传输的 `async_write`，让子类（如 UDP）的特化生效。

#### 签名

```cpp
auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffer | `std::span<const std::byte>` | 源数据缓冲区 |
| ec | `std::error_code &` | 输出错误码 |

#### 返回值

`net::awaitable<std::size_t>` — 写入的字节数。

#### 调用（向下）

- `transmission::async_write()` — 委托给内部传输的完整写入

#### 被调用（向上）

- `tunnel()` — `complete_write=true` 时使用完整写入语义

#### 涉及的知识域

- [[ref/programming/boost-asio|Boost.Asio]] — AsyncWriteStream 概念

---

### 成员函数: close()

#### 功能

关闭内部传输流，清空预读缓冲区后关闭内部传输连接。

#### 签名

```cpp
void close() override;
```

#### 参数

无参数。

#### 返回值

无返回值。

#### 调用（向下）

- `transmission::close()` — 关闭内部传输连接

#### 被调用（向上）

- `shut_close()` — 通过基类指针调用 close

#### 涉及的知识域

- [[ref/network/tcp|TCP]] — 连接关闭

---

### 成员函数: cancel()

#### 功能

取消内部传输的待处理操作。

#### 签名

```cpp
void cancel() override;
```

#### 参数

无参数。

#### 返回值

无返回值。

#### 调用（向下）

- `transmission::cancel()` — 取消内部传输上的挂起异步操作

#### 被调用（向上）

- 协程取消传播 — 会话超时或连接断开时取消操作

#### 涉及的知识域

- [[ref/programming/boost-asio|Boost.Asio]] — 异步操作取消

---

### 成员函数: inner()

#### 功能

获取内部传输对象的智能指针。

#### 签名

```cpp
[[nodiscard]] auto inner() const noexcept -> shared_transmission;
```

#### 参数

无参数。

#### 返回值

`shared_transmission` — 内部传输的 shared_ptr。

#### 调用（向下）

无。

#### 被调用（向上）

- `find_reliable()` — 穿透 preview 层查找底层 reliable 传输

#### 涉及的知识域

- [[ref/memory/pmr|PMR]] — 智能指针与共享所有权

---

## 函数: wrap_with_preview()

> 源码: `include/prism/pipeline/primitives.hpp:224`

### 功能

将入站传输包装为带预读数据的传输，若 `data` 不为空则创建 preview 包装器；mux 模式下使用全局内存池避免 smux_craft 析构时的 UAF 风险。

### 签名

```cpp
inline auto wrap_with_preview(session_context &ctx, std::span<const std::byte> data,
                              bool use_global_mr = false) -> shared_transmission;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，包含入站传输和帧内存资源 |
| data | `std::span<const std::byte>` | 协议嗅探期间捕获的预读数据 |
| use_global_mr | `bool` | 是否使用全局内存池而非帧竞技场，默认 false |

### 返回值

`shared_transmission` — 包装后的传输对象；若 `data` 为空则直接返回原始入站传输。

### 调用（向下）

- `preview` 构造函数 — 创建预读回放包装器

### 被调用（向上）

- [[pipeline/protocols/http|http]] — HTTP 处理器包装入站传输
- [[pipeline/protocols/socks5|socks5]] — SOCKS5 处理器包装入站传输
- [[pipeline/protocols/trojan|trojan]] — Trojan 处理器包装入站传输（use_global_mr=true）
- [[pipeline/protocols/vless|vless]] — VLESS 处理器包装入站传输（use_global_mr=true）
- [[pipeline/protocols/shadowsocks|shadowsocks]] — SS2022 处理器包装入站传输（use_global_mr=true）

### 涉及的知识域

- [[ref/memory/pmr|PMR]] — 多态内存资源分配
- [[ref/memory/arena|竞技场]] — 帧竞技场 vs 全局内存池

### 注意事项

- 调用后 `ctx.inbound` 被置空，所有权转移至返回值
- mux 模式下 `use_global_mr=true`，因为 transport 生命周期超出 session 帧竞技场

---

## 函数: tunnel()

> 源码: `include/prism/pipeline/primitives.hpp:251`
> 实现: `src/prism/pipeline/primitives.cpp:196`

### 功能

在两个流之间运行全双工隧道，建立双向数据转发，任一方向断开即终止整个隧道，隧道结束后自动关闭两端连接。

### 签名

```cpp
auto tunnel(shared_transmission inbound, shared_transmission outbound,
            const session_context &ctx, bool complete_write = true)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| inbound | `shared_transmission` | 入站流对象（客户端侧） |
| outbound | `shared_transmission` | 出站流对象（上游侧） |
| ctx | `const session_context &` | 会话上下文，提供内存资源和缓冲区配置 |
| complete_write | `bool` | 是否使用完整写入语义，默认 true |

### 返回值

`net::awaitable<void>` — 隧道结束后完成。

### 调用（向下）

- `transmission::async_read_some()` — 从一端读取数据
- `transmission::async_write()` — 完整写入到对端（complete_write=true）
- `transmission::async_write_some()` — 单次写入到对端（complete_write=false）
- `shut_close()` — 隧道结束后关闭两端传输
- `boost::asio::experimental::awaitable_operators::operator||` — 并行双向转发

### 被调用（向上）

- `forward()` — 拨号成功后建立隧道
- [[pipeline/protocols/http|http]] — HTTP CONNECT/普通请求建立隧道
- [[pipeline/protocols/socks5|socks5]] — SOCKS5 CONNECT 命令建立隧道
- [[agent/dispatch/table|handle_unknown]] — Unknown 协议原始透传

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 双向数据转发
- [[ref/programming/c++23-coroutines|C++23 协程]] — 协程并行组合（awaitable_operators）
- [[ref/programming/boost-asio|Boost.Asio]] — 异步读写操作

### 流程

1. 从帧竞技场分配缓冲区，切成两半（上行/下行各一块）
2. 启动两个并行协程：入站→出站、出站→入站
3. 每个协程循环：读取 → 写入，直到任一端失败或 EOF
4. 任一方向结束时自动取消另一个方向
5. 输出传输统计（上行/下行字节数、持续时间）
6. 关闭两端传输连接

### 注意事项

- 缓冲区大小由 `ctx.buffer_size` 配置，至少 2 字节
- `complete_write=true` 确保所有读取的数据都被完整写入对端
- `complete_write=false` 适用于对吞吐量优先于可靠性的场景

---

## 函数: forward()

> 源码: `include/prism/pipeline/primitives.hpp:266`
> 实现: `src/prism/pipeline/primitives.cpp:269`

### 功能

拨号连接上游并建立双向隧道，组合 `dial` + `tunnel` 操作，支持通过出站代理或路由器两种路径。所有协议的 TCP 隧道转发共用此函数。

### 签名

```cpp
auto forward(session_context &ctx, std::string_view label,
             const protocol::analysis::target &target,
             shared_transmission inbound) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，提供路由器和出站代理 |
| label | `std::string_view` | 协议标签，用于日志记录 |
| target | `const protocol::analysis::target &` | 目标地址信息 |
| inbound | `shared_transmission` | 入站传输对象 |

### 返回值

`net::awaitable<void>` — 隧道结束后完成。

### 调用（向下）

- `dial()`（出站代理重载） — 通过出站代理拨号
- `dial()`（路由器重载） — 通过路由器拨号
- `tunnel()` — 建立双向隧道

### 被调用（向上）

- [[pipeline/protocols/trojan|trojan]] — Trojan CONNECT 命令转发
- [[pipeline/protocols/vless|vless]] — VLESS TCP 命令转发
- [[pipeline/protocols/shadowsocks|shadowsocks]] — SS2022 数据转发

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP 连接建立与数据转发
- [[ref/network/connection-pool|连接池]] — 连接池化与复用

### 流程

1. 检查是否配置了出站代理
2. 若有出站代理：通过 `dial(outbound::proxy)` 拨号
3. 若无出站代理：通过 `dial(router)` 拨号（allow_reverse=true）
4. 拨号失败时记录日志并返回
5. 拨号成功后调用 `tunnel()` 建立双向隧道

---

## 函数: is_mux_target()

> 源码: `include/prism/pipeline/primitives.hpp:278`
> 实现: `src/prism/pipeline/primitives.cpp:319`

### 功能

检测目标地址是否为 Mihomo/sing-box 兼容的 mux 多路复用标记地址（以 `.mux.sing-box.arpa` 结尾）。

### 签名

```cpp
[[nodiscard]] auto is_mux_target(std::string_view host, bool mux_enabled) noexcept -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| host | `std::string_view` | 目标主机名 |
| mux_enabled | `bool` | 是否启用多路复用 |

### 返回值

`bool` — 若目标地址为 mux 标记地址且 mux 已启用返回 `true`。

### 调用（向下）

无。

### 被调用（向上）

- [[pipeline/protocols/trojan|trojan]] — Trojan 检测 mux 标记地址
- [[pipeline/protocols/vless|vless]] — VLESS 检测 mux 标记地址

### 涉及的知识域

- [[ref/protocol/smux-yamux|Smux/Yamux]] — 多路复用协议
- [[multiplex/bootstrap|bootstrap]] — 多路复用引导

---

## 函数: make_datagram_router()

> 源码: `include/prism/pipeline/primitives.hpp:290`
> 实现: `src/prism/pipeline/primitives.cpp:325`

### 功能

创建 UDP 数据报路由回调函数，用于 UDP ASSOCIATE 的路由。返回的回调持有 router 的非拥有引用（空删除器 shared_ptr），避免每个协议重复构造。

### 签名

```cpp
auto make_datagram_router(resolve::router &router)
    -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(
        std::string_view, std::string_view)>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| router | `resolve::router &` | 路由器引用 |

### 返回值

`std::function<...>` — UDP 路由回调函数，接受主机名和端口，返回错误码和 UDP 端点的配对。

### 调用（向下）

- [[resolve/router|router::resolve_datagram_target]] — 解析 UDP 目标地址

### 被调用（向上）

- [[pipeline/protocols/socks5|socks5]] — SOCKS5 UDP_ASSOCIATE 创建路由回调
- [[pipeline/protocols/trojan|trojan]] — Trojan UDP_ASSOCIATE 创建路由回调
- [[pipeline/protocols/vless|vless]] — VLESS UDP 命令创建路由回调

### 涉及的知识域

- [[ref/network/udp|UDP]] — UDP 数据报传输
- [[ref/network/nat-traversal|NAT 穿透]] — UDP NAT 穿透

### 注意事项

- 返回的回调持有 router 的非拥有引用（空删除器 shared_ptr）
- 调用方必须确保 router 的生命周期长于回调的使用期

---

## 函数: find_reliable()

> 源码: `include/prism/pipeline/primitives.hpp:301`

### 功能

从传输层包装链中解包找到底层 reliable 传输，穿透 snapshot/preview 等装饰层，找到底层的 TCP socket 传输。用于 ShadowTLS/Restls 等需要直接操作 raw socket 的 scheme。

### 签名

```cpp
inline auto find_reliable(shared_transmission &trans) noexcept
    -> psm::channel::transport::reliable *;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| trans | `shared_transmission &` | 传输层智能指针（可能被 snapshot/preview 包装） |

### 返回值

`psm::channel::transport::reliable *` — reliable 传输的裸指针，找不到返回 `nullptr`。

### 调用（向下）

- `dynamic_cast<preview *>` — 穿透 preview 层
- `dynamic_cast<snapshot *>` — 穿透 snapshot 层
- `dynamic_cast<reliable *>` — 最终类型转换

### 被调用（向上）

- `stealth::shadowtls::scheme` — ShadowTLS 方案中查找底层 TCP socket
- `stealth::restls::scheme` — Restls 方案中查找底层 TCP socket

### 涉及的知识域

- [[ref/network/tcp|TCP]] — TCP socket 直接操作
- [[channel/transport/reliable|reliable]] — 可靠传输层
- [[channel/transport/snapshot|snapshot]] — 快照传输装饰器

### 注意事项

- 使用 `dynamic_cast` 进行运行时类型识别，穿透多层装饰器
- 返回裸指针，调用方不拥有所有权
