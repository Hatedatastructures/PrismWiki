---
title: Pipeline 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [pipeline, protocol, overview, design]
related: [[protocol], [recognition], [agent], [channel/overview], [resolve/overview], [memory/overview], [fault/overview]]
---

# Pipeline 模块

Pipeline 模块是 Prism 的**协议处理管道层**，位于识别层和路由层之间。它提供通用原语（预读回放、上游拨号、双向隧道）和各协议的会话处理函数，将协议编解码（protocol）与连接建立（channel/resolve）编排为完整的请求处理流程。

## 概述

### 核心职责

- **管道原语**：提供预读回放、上游拨号、TLS 握手、双向隧道等通用操作
- **协议会话处理**：HTTP/SOCKS5/Trojan/VLESS/Shadowsocks 的完整握手和转发编排
- **协议编排**：协调 识别结果 → 协议握手 → 路由拨号 → 隧道转发 的完整流程

### 模块结构

```
include/prism/pipeline/
├── primitives.hpp            # 管道原语（核心）
├── protocols.hpp             # 聚合头文件
└── protocols/
    ├── http.hpp              # HTTP 代理协议处理
    ├── socks5.hpp            # SOCKS5 代理协议处理
    ├── trojan.hpp            # Trojan 协议处理
    ├── vless.hpp             # VLESS 协议处理
    └── shadowsocks.hpp       # SS2022 协议处理
```

### 关键组件

#### primitives（管道原语）

所有协议处理共享的底层操作，位于 `primitives` 命名空间：

| 原语 | 功能 |
|------|------|
| `shut_close()` | 安全关闭传输层（裸指针或智能指针） |
| `dial()` | 拨号上游连接（通过 router 或出站代理） |
| `ssl_handshake()` | TLS 服务端握手，失败时恢复传输层所有权 |
| `preview` | 预读数据回放包装器，优先返回嗅探阶段的数据 |
| `wrap_with_preview()` | 将入站传输包装为带预读数据的传输 |
| `tunnel()` | 双向隧道转发，两个半缓冲区处理双向数据 |
| `forward()` | 组合 `dial + tunnel`，所有协议 TCP 转发共用 |
| `is_mux_target()` | 检测 mux 多路复用标记地址 |
| `make_datagram_router()` | 创建 UDP 数据报路由回调 |
| `find_reliable()` | 从传输层包装链中解包找到底层 TCP socket |

#### 协议处理函数

每个协议提供统一签名的处理函数：

```cpp
namespace psm::pipeline {
    auto http(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
    auto socks5(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
    auto trojan(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
    auto vless(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
    auto shadowsocks(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
}
```

这些函数是编排层，调用 protocol 模块的 relay 进行协议级逻辑，再调用 primitives 建立连接和隧道。

### 数据流

```
session.diversion()
    │ recognition 识别结果
    ▼
dispatch::table[type](ctx, data)    ← 编译期函数表分发
    │
    ▼
pipeline handler (如 socks5)
    │ 解析协议头 → 提取目标地址
    ├── TCP → primitives::forward() → dial() + tunnel()
    │         ├── 无 mux → 直接连接目标
    │         └── mux → 启动多路复用器
    └── UDP → make_datagram_router() → UDP 中继
```

## 详细设计

### 模块职责与边界

#### 职责

- **协议会话处理**：HTTP/SOCKS5/Trojan/VLESS/Shadowsocks 的完整握手和转发编排
- **管道原语**：预读回放、上游拨号、TLS 握手、双向隧道
- **协议编排**：协调识别结果 → 协议处理 → 路由 → 隧道的完整流程

#### 边界

- **不负责**：协议编解码细节（由 protocol 模块的 relay/format 处理）
- **不负责**：协议识别（由 recognition 模块处理）
- **不负责**：DNS 解析和路由决策（由 resolve 模块处理）
- **不负责**：连接池管理（由 channel 模块处理）

### 关键类和接口

#### `primitives::preview` — 预读数据回放包装器

**文件**: `primitives.hpp`

```cpp
class preview final : public channel::transport::transmission {
public:
    explicit preview(shared_transmission inner,
                     std::span<const std::byte> preread,
                     memory::resource_pointer mr = memory::current_resource());

    auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;     // 优先返回预读数据
    auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;
    auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;     // 委托给内部传输的完整写入

    void close() override;
    void cancel() override;
    [[nodiscard]] auto inner() const noexcept -> shared_transmission;

private:
    shared_transmission inner_;
    memory::vector<std::byte> preread_buffer_;  // PMR 分配
    std::size_t offset_{0};
};
```

继承 `transmission` 抽象基类，可透明替换原始传输。`async_read_some` 优先从预读缓冲区返回数据，耗尽后委托给内部传输。`async_write` 直接委托给内部传输，让子类（如 UDP）的特化生效。

#### 管道原语函数

**文件**: `primitives.hpp`

```cpp
namespace psm::pipeline::primitives {
    void shut_close(transmission *trans) noexcept;
    void shut_close(shared_transmission &trans) noexcept;

    auto dial(resolve::router &router, std::string_view label,
              const protocol::analysis::target &target, bool allow_reverse, bool require_open)
        -> net::awaitable<std::pair<fault::code, shared_transmission>>;

    auto dial(outbound::proxy &proxy, const protocol::analysis::target &target,
              const net::any_io_executor &executor)
        -> net::awaitable<std::pair<fault::code, shared_transmission>>;

    auto ssl_handshake(shared_transmission inbound, ssl::context &ssl_ctx)
        -> net::awaitable<std::tuple<fault::code, shared_ssl_stream, shared_transmission>>;

    inline auto wrap_with_preview(session_context &ctx, std::span<const std::byte> data,
                                  bool use_global_mr = false) -> shared_transmission;

    auto tunnel(shared_transmission inbound, shared_transmission outbound,
                const session_context &ctx, bool complete_write = true)
        -> net::awaitable<void>;

    auto forward(session_context &ctx, std::string_view label,
                 const protocol::analysis::target &target, shared_transmission inbound)
        -> net::awaitable<void>;

    [[nodiscard]] auto is_mux_target(std::string_view host, bool mux_enabled) noexcept -> bool;
    auto make_datagram_router(resolve::router &router) -> std::function<...>;
    inline auto find_reliable(shared_transmission &trans) noexcept -> reliable *;
}
```

#### 协议处理函数

**文件**: `protocols/*.hpp`

每个协议处理器的签名统一为 `(session_context &, std::span<const std::byte>) -> awaitable<void>`。处理器内部调用 protocol 模块的 relay 完成协议级逻辑，再调用 primitives 建立连接。

| 函数 | 文件 | 调用的 protocol 组件 | 支持的命令 |
|------|------|---------------------|-----------|
| `http()` | `protocols/http.hpp` | `http::relay` | CONNECT（隧道）、普通请求（转发） |
| `socks5()` | `protocols/socks5.hpp` | `socks5::stream` | CONNECT、UDP_ASSOCIATE |
| `trojan()` | `protocols/trojan.hpp` | `trojan::relay` | CONNECT、UDP_ASSOCIATE、MUX |
| `vless()` | `protocols/vless.hpp` | `vless::relay` | TCP、UDP、MUX |
| `shadowsocks()` | `protocols/shadowsocks.hpp` | `shadowsocks::relay` | TCP、UDP |

### 文件结构

#### 源文件 (`src/prism/pipeline/`)

```
pipeline/
├── primitives.cpp            # 管道原语实现
└── protocols/
    ├── http.cpp              # HTTP 协议处理实现
    ├── socks5.cpp            # SOCKS5 协议处理实现
    ├── trojan.cpp            # Trojan 协议处理实现
    ├── vless.cpp             # VLESS 协议处理实现
    └── shadowsocks.cpp       # SS2022 协议处理实现
```

### 文件依赖关系

```
primitives.hpp
├── boost/asio.hpp, boost/asio/ssl.hpp
├── agent/context.hpp
├── resolve/router.hpp
├── fault/code.hpp
├── protocol/analysis.hpp
├── channel/transport/{transmission,reliable,snapshot}.hpp
├── channel/adapter/connector.hpp
└── outbound/proxy.hpp

protocols/http.hpp, socks5.hpp, trojan.hpp, vless.hpp, shadowsocks.hpp
├── agent/context.hpp
└── pipeline/primitives.hpp
```

## 模块依赖

### 依赖的模块

| 模块 | 用途 |
|------|------|
| `protocol` | `analysis::target`、各协议的 `relay` 和 `format` |
| `resolve` | `router` 路由器 |
| `channel/transport` | `transmission`、`reliable`、`snapshot` 传输层 |
| `channel/adapter` | `connector` 连接器（TLS 握手用） |
| `agent/context` | `session_context` 会话上下文 |
| `memory` | PMR 容器和内存池 |
| `fault` | 错误码定义 |
| `outbound` | 出站代理抽象 |

### 被依赖的模块

| 模块 | 用途 |
|------|------|
| `agent/dispatch` | handler_table 引用各协议处理函数 |
| `agent/session` | session::diversion() 通过 dispatch 间接调用 |

## 设计原理

- **原语复用模式**：每个协议都需要拨号、隧道、预读等功能。解决方案：提取通用原语到 `primitives` 命名空间，`forward()` 组合 `dial + tunnel` 作为所有协议 TCP 转发的共用入口
- **协议与编排分离**：两层架构——`protocol::*::relay` 处理协议级逻辑（握手、编解码），`pipeline::protocols::*` 处理编排逻辑（调用 relay + 原语）
- **预读数据管理**：协议嗅探消耗了部分数据，`preview` 包装器将预读数据保存在内部缓冲区，`async_read_some` 优先返回预读数据，对上层透明。`wrap_with_preview` 在 mux 模式下使用全局内存池避免 smux_craft 析构时的 UAF 风险
- **错误处理策略**：握手失败时静默关闭连接，不发送错误响应；路由失败时记录日志，关闭连接。不抛异常，使用 `fault::code` 返回错误
- **完整写入语义**：`tunnel()` 的 `complete_write` 参数控制写入行为——true 时确保所有读取的数据都被完整写入对端，false 时使用单次写入适用于吞吐量优先场景

## 相关链接

- [[protocol]] — Protocol 模块
- [[agent]] — Agent 模块
- [[channel]] — Channel 模块
- [[memory]] — Memory 模块
