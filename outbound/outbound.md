---
title: Outbound 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [outbound, proxy, direct, network]
related: [[channel], [fault/overview], [resolve], [protocol/overview], [trace/overview], [pipeline/overview], [agent/overview]]
---

# Outbound 模块

## 1. 模块职责与边界

Outbound 模块定义 Prism 的**出站代理抽象接口**和直连实现。它提供：

- **`proxy` 抽象接口**：所有出站协议和代理组必须实现的核心接口
- **`direct` 直连实现**：最简单的出站代理，通过 DNS 解析 + Happy Eyeballs 建立连接

**边界**：Outbound 模块位于 pipeline 之下、channel 之上。`proxy` 接口支持组合模式（代理组和单个代理共享同一接口）。等价于 mihomo 的 `C.ProxyAdapter` 接口。

## 2. 关键类和接口

### 2.1 `proxy` — 出站代理抽象接口（`proxy.hpp`）

```cpp
namespace psm::outbound {
    using shared_transmission = channel::transport::shared_transmission;
    using datagram_router_fn = std::function<
        net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(
            std::string_view, std::string_view)>;

    class proxy {
    public:
        virtual ~proxy() = default;

        // 建立 TCP 连接
        virtual auto async_connect(const protocol::analysis::target &target,
                                   const net::any_io_executor &executor)
            -> net::awaitable<std::pair<fault::code, shared_transmission>> = 0;

        // 创建 UDP 数据报路由回调
        virtual auto make_datagram_router() -> datagram_router_fn = 0;

        // 代理名称
        [[nodiscard]] virtual auto name() const -> std::string_view = 0;

        // 是否支持 UDP（默认 true）
        [[nodiscard]] virtual auto supports_udp() const -> bool { return true; }
    };
}
```

**设计特性**：
- 纯协程：异步操作返回 `net::awaitable`
- 错误码：通过 `fault::code` 返回错误，无异常开销
- 组合模式：代理组实现同一接口，透明委托

### 2.2 `direct` — 直连出站代理（`direct.hpp`）

```cpp
class direct : public proxy {
public:
    explicit direct(resolve::router &router);

    auto async_connect(const protocol::analysis::target &target,
                       const net::any_io_executor &executor)
        -> net::awaitable<std::pair<fault::code, shared_transmission>> override;

    auto make_datagram_router() -> datagram_router_fn override;

    [[nodiscard]] auto name() const -> std::string_view override { return "DIRECT"; }

private:
    static auto is_ipv6_literal(std::string_view host) noexcept -> bool;
    resolve::router &router_;
};
```

**路由策略**：
- `target.positive == false`：反向路由（域名 → 预配置后端地址）
- `target.positive == true`：正向路由（DNS 解析 + TCP 连接）
- IPv6 禁用时拒绝 IPv6 地址字面量

**连接流程**：
1. IPv6 检查：`ipv6_disabled && is_ipv6_literal` → 拒绝
2. 路由：根据 `target.positive` 选择正向/反向路由
3. 错误检查：`failed(ec)` → 返回错误
4. 连接验证：`!conn.valid()` → 返回 `connection_refused`
5. 成功：返回 `{success, make_reliable(move(conn))}`

### 2.3 UDP 数据报路由

`make_datagram_router()` 返回一个协程回调函数，封装 DNS 解析：
```cpp
auto make_datagram_router() -> datagram_router_fn {
    const auto ptr = std::shared_ptr<resolve::router>(&router_, [](auto*){});
    return [ptr](host, port) -> net::awaitable<std::pair<fault::code, udp::endpoint>> {
        co_return co_await ptr->resolve_datagram_target(host, port);
    };
}
```

使用空删除器的 `shared_ptr` 避免生命周期问题（`router` 的生命周期由 worker 管理）。

## 3. 文件清单

### 头文件（`include/prism/outbound/`）

```
outbound.hpp             # 聚合头文件
├── proxy.hpp            # 出站代理抽象接口（proxy, datagram_router_fn）
└── direct.hpp           # 直连出站实现（direct）
```

**注意**：Outbound 模块是 header-only，无源文件。

## 4. 设计原理

### 4.1 组合模式
`proxy` 接口支持组合模式：代理组（url_test、fallback、load_balance、selector）也实现 `proxy` 接口，通过委托给子代理的 `async_connect` 实现。调用方透明地与单个代理或代理组交互。

### 4.2 等价 mihomo 设计
`proxy` 接口等价于 mihomo 的 `C.ProxyAdapter` 接口，`direct` 等价于 mihomo 的 `adapter/outbound/direct.go`。这保证了架构一致性。

### 4.3 非拥有指针
`direct` 持有 `router` 的引用（非拥有），生命周期由 worker 管理。`make_datagram_router` 使用空删除器的 `shared_ptr`，避免悬空引用。

### 4.4 协程 + 错误码
所有异步操作返回 `net::awaitable<std::pair<fault::code, T>>`，不抛异常。这与 Prism 的热路径无异常原则一致。

## 相关页面

- [[channel]] — Channel 模块（传输层）
- [[pipeline]] — Pipeline 模块（调用方）
- [[resolve]] — Resolve 模块（DNS 解析）
- [[memory]] — Memory 模块
- [[agent]] — Agent 模块
- [[architecture]] — 架构设计
