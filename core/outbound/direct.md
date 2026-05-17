---
title: "direct — 直连出站代理"
layer: core
source: "I:/code/Prism/include/prism/outbound/direct.hpp"
module: "outbound"
type: component
tags: [outbound, direct, adapter, router]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/outbound/proxy
  - core/resolve/router
  - core/channel/transport/reliable
  - core/channel/connection/pool
---

# direct — 直连出站代理

> 源码位置: `I:/code/Prism/include/prism/outbound/direct.hpp`
> 模块: [[core/outbound/overview|outbound]]

## 组件定位

`direct` 是最简单的 `outbound::proxy` 实现，将连接请求直接通过 DNS 解析和 Happy Eyeballs 建立连接，不经过任何上游代理。该类包装现有 `resolve::router` 和 `channel::connection_pool` 的直连行为。

**等价于 mihomo 的 `adapter/outbound/direct.go`。**

**特性**:
- Header-only 实现
- 每个 worker 持有一个实例
- 生命周期由 worker 管理
- router 引用必须比 direct 实例长寿

## 类定义

```cpp
namespace psm::outbound
{
    class direct : public proxy
    {
    public:
        explicit direct(resolve::router &router);

        auto async_connect(const protocol::analysis::target &target,
                           const net::any_io_executor &executor)
            -> net::awaitable<std::pair<fault::code, shared_transmission>> override;

        auto make_datagram_router()
            -> datagram_router_fn override;

        [[nodiscard]] auto name() const -> std::string_view override;

    private:
        static auto is_ipv6_literal(const std::string_view host) noexcept -> bool;

        resolve::router &router_;
    };
}
```

## 核心方法详解

### 构造函数

```cpp
explicit direct(resolve::router &router)
    : router_(router)
{
}
```

构造直连出站代理，绑定路由器引用。

**参数**:
- `router`: 路由器引用，用于 DNS 解析和连接建立

**注意**: router 引用必须比 direct 实例长寿。

---

### async_connect() 完整实现

```cpp
auto async_connect(const protocol::analysis::target &target,
                   const net::any_io_executor &executor)
    -> net::awaitable<std::pair<fault::code, shared_transmission>> override
{
    // 1. IPv6 地址字面量检查
    if (router_.ipv6_disabled() && is_ipv6_literal(target.host))
    {
        trace::debug("[Outbound.Direct] rejecting IPv6 literal: {}:{}", target.host, target.port);
        co_return std::pair{fault::code::ipv6_disabled, nullptr};
    }

    // 2. 路由到目标
    fault::code ec;
    channel::pooled_connection conn;

    if (!target.positive)
    {
        // 反向代理：域名 → 预配置的后端地址
        auto result = co_await router_.async_reverse(target.host);
        ec = result.first;
        conn = std::move(result.second);
    }
    else
    {
        // 正向代理：域名 → DNS 解析 → TCP 连接
        auto result = co_await router_.async_forward(target.host, target.port);
        ec = result.first;
        conn = std::move(result.second);
    }

    // 3. 错误处理
    if (fault::failed(ec))
    {
        trace::warn("[Outbound.Direct] route failed: {}, target: {}:{}", fault::describe(ec),
                    target.host, target.port);
        co_return std::pair{ec, nullptr};
    }

    // 4. 连接有效性验证
    if (!conn.valid())
    {
        trace::warn("[Outbound.Direct] socket not open, target: {}:{}", target.host, target.port);
        co_return std::pair{fault::code::connection_refused, nullptr};
    }

    // 5. 成功返回
    trace::info("[Outbound.Direct] success, target: {}:{}", target.host, target.port);
    co_return std::pair{fault::code::success,
                        channel::transport::make_reliable(std::move(conn))};
}
```

**实现流程**:

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | IPv6 检查 | 禁用 IPv6 时拒绝 IPv6 字面量 |
| 2 | 路由选择 | 根据 `target.positive` 选择正向/反向路由 |
| 3 | 错误处理 | 检查路由结果并记录日志 |
| 4 | 连接验证 | 确保连接有效 |
| 5 | 返回传输 | 封装为可靠传输对象 |

---

### make_datagram_router() 实现

```cpp
auto make_datagram_router()
    -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(
        std::string_view, std::string_view)> override
{
    // 创建非拥有 shared_ptr（空删除器）
    const auto ptr = std::shared_ptr<resolve::router>(&router_, []([[maybe_unused]] resolve::router *p)
                                                      {
                                                          // 非拥有指针，空删除器
                                                      });

    // 返回捕获 shared_ptr 的 lambda
    return [ptr](const std::string_view host, const std::string_view port)
               -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>
    {
        co_return co_await ptr->resolve_datagram_target(host, port);
    };
}
```

**设计要点**:
- 使用 `shared_ptr` + 空删除器延长 router 生命周期
- lambda 捕获 `shared_ptr`，确保回调执行时 router 有效
- 委托给 `router::resolve_datagram_target` 进行 DNS 解析

---

### name() 实现

```cpp
[[nodiscard]] auto name() const -> std::string_view override
{
    return "DIRECT";
}
```

返回固定名称 `"DIRECT"`，用于日志和调试。

---

### is_ipv6_literal() 辅助函数

```cpp
static auto is_ipv6_literal(const std::string_view host) noexcept -> bool
{
    boost::system::error_code ec;
    const auto addr = net::ip::make_address(host, ec);
    return !ec && addr.is_v6();
}
```

检查目标地址是否为 IPv6 字面量。

**参数**:
- `host`: 目标主机名或 IP 地址

**返回值**: 如果是有效的 IPv6 地址字面量返回 `true`

## 路由策略

### 正向代理 vs 反向代理

| 模式 | `target.positive` | 行为 |
|------|-------------------|------|
| 正向代理 | `true` | 域名 → DNS 解析 → TCP 连接 |
| 反向代理 | `false` | 域名 → 预配置后端地址 |

### 正向代理流程

```
target (host:port, positive=true)
    │
    ▼
┌─────────────────────────────────────────────────┐
│                 async_forward                     │
├─────────────────────────────────────────────────┤
│  1. IP 字面量检测                                 │
│     ├─ IPv4 字面量 → 直接连接                     │
│     └─ IPv6 字面量 → 检查是否禁用 IPv6            │
│                                                   │
│  2. 域名解析                                      │
│     └─ dns_->resolve_tcp(host, port)             │
│                                                   │
│  3. Happy Eyeballs 连接                          │
│     └─ address_racer::race(endpoints)            │
└─────────────────────────────────────────────────┘
    │
    ▼
pooled_connection → make_reliable() → shared_transmission
```

### 反向代理流程

```
target (host, positive=false)
    │
    ▼
┌─────────────────────────────────────────────────┐
│                 async_reverse                     │
├─────────────────────────────────────────────────┤
│  查表: reverse_map_.find(host)                   │
│     ├─ 命中 → 获取预配置的 endpoint              │
│     └─ 未命中 → 返回错误                          │
│                                                   │
│  连接: pool_.async_acquire(endpoint)             │
└─────────────────────────────────────────────────┘
    │
    ▼
pooled_connection → make_reliable() → shared_transmission
```

## 成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `router_` | `resolve::router &` | 路由器引用，用于 DNS 解析和连接建立 |

## 调用链

```
direct::async_connect
    │
    ├─→ IPv6 字面量检查
    │       → is_ipv6_literal()
    │
    ├─→ 正向代理分支 (target.positive == true)
    │       → router_.async_forward(host, port)
    │             ├─→ dns_->resolve_tcp()
    │             └─→ address_racer::race()
    │
    ├─→ 反向代理分支 (target.positive == false)
    │       → router_.async_reverse(host)
    │             └─→ reverse_map_.find()
    │
    └─→ 返回传输对象
            → channel::transport::make_reliable(conn)
```

## 设计决策

### 非拥有 shared_ptr 模式

`make_datagram_router()` 使用空删除器的 `shared_ptr` 延长 router 生命周期：

```cpp
// 非拥有指针，空删除器
const auto ptr = std::shared_ptr<resolve::router>(&router_, [](resolve::router *) {});
```

**优点**:
- 避免悬空引用
- lambda 可安全捕获
- 无需额外的生命周期管理

### IPv6 禁用检查

在 IPv6 禁用场景下，提前拒绝 IPv6 字面量地址：

```cpp
if (router_.ipv6_disabled() && is_ipv6_literal(target.host))
{
    co_return std::pair{fault::code::ipv6_disabled, nullptr};
}
```

### 连接有效性验证

返回前验证连接有效性：

```cpp
if (!conn.valid())
{
    co_return std::pair{fault::code::connection_refused, nullptr};
}
```

## 参见

- [[core/outbound/proxy|proxy]] — 出站代理抽象接口
- [[core/resolve/router|router]] — 分发层路由器
- [[core/channel/transport/reliable|reliable]] — TCP 可靠传输实现
- [[core/channel/connection/pool|connection_pool]] — TCP 连接池
- [[core/fault/code|fault::code]] — 错误码定义