---
title: "outbound — 出站代理模块"
layer: core
source: "I:/code/Prism/include/prism/outbound/"
module: "outbound"
type: overview
tags: [outbound, proxy, direct, adapter]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/router
  - core/channel/transport/transmission
  - core/protocol/analysis
---

# outbound — 出站代理模块

> 源码位置: `I:/code/Prism/include/prism/outbound/`
> 模块: [[outbound|outbound]]

## 模块概述

`outbound` 模块定义了出站代理的抽象接口及其实现，是 Prism 代理框架的核心组件之一。该模块采用组合模式设计，使代理组（url_test、fallback、load_balance、selector）和单个代理（direct、socks5、http、trojan、vless、ss 等）共享同一接口。

### 设计特性

- **纯协程设计**: 所有异步操作返回 `net::awaitable`，简化异步调用
- **错误码返回**: 通过 `fault::code` 返回错误，无异常开销
- **组合模式**: 代理组实现同一接口，透明委托给子代理
- **接口统一**: 调用方无需区分单个代理和代理组

## 模块组成

| 组件 | 文件 | 说明 |
|------|------|------|
| [[core/outbound/proxy\|proxy]] | `proxy.hpp` | 出站代理抽象接口 |
| [[core/outbound/direct\|direct]] | `direct.hpp` | 直连出站代理实现 |

## 核心接口

### proxy 抽象接口

`proxy` 是所有出站协议和代理组必须实现的抽象接口。定义了两个核心方法：

```cpp
class proxy
{
public:
    virtual ~proxy() = default;

    // TCP 连接建立
    virtual auto async_connect(const protocol::analysis::target &target,
                               const net::any_io_executor &executor)
        -> net::awaitable<std::pair<fault::code, shared_transmission>> = 0;

    // UDP 数据报路由
    virtual auto make_datagram_router()
        -> datagram_router_fn = 0;

    // 代理元信息
    [[nodiscard]] virtual auto name() const -> std::string_view = 0;
    [[nodiscard]] virtual auto supports_udp() const -> bool { return true; }
};
```

### direct 直连实现

`direct` 是最简单的 `proxy` 实现，将连接请求直接通过 DNS 解析和 Happy Eyeballs 建立连接，不经过任何上游代理。

```cpp
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
};
```

## 架构关系

```
┌─────────────────────────────────────────────────────────────┐
│                       调用方 (worker)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ proxy (接口)    │
                    └─────────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    ┌───────────┐     ┌───────────┐     ┌───────────┐
    │  direct   │     │  socks5   │     │  trojan   │
    │ (直连)    │     │  (协议)   │     │  (协议)   │
    └───────────┘     └───────────┘     └───────────┘
           │
           ▼
    ┌───────────┐
    │  router   │ ←── DNS 解析 + 连接池
    └───────────┘
```

## 等价参考

| Prism 组件 | mihomo 等价 |
|-----------|-------------|
| `outbound::proxy` | `constant/adapters.go` 中的 `ProxyAdapter` 接口 |
| `outbound::direct` | `adapter/outbound/direct.go` |

## 参见

- [[core/resolve/router|router]] — 分发层路由器
- [[core/channel/transport/transmission|transmission]] — 传输层抽象接口
- [[core/protocol/analysis|protocol::analysis]] — 协议分析模块
- [[core/fault/code|fault::code]] — 错误码定义