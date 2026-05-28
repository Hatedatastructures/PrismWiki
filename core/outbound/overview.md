---
title: "outbound — 出站代理模块"
layer: core
module: outbound
source: I:/code/Prism/include/prism/outbound/
tags: [outbound, proxy, direct, adapter]
created: 2026-05-17
updated: 2026-05-28
---

# outbound — 出站代理模块

出站代理的抽象接口及实现。采用组合模式，代理组（url_test、fallback、load_balance）和单个代理（direct、socks5、trojan 等）共享 `proxy` 接口。

## 设计决策

### 为什么 proxy 接口只有 TCP connect 和 UDP datagram 两个方法？

**问题**: 出站代理的职责是"建立到目标的连接"，接口需要覆盖所有代理场景同时保持最小化。

**选择**: TCP 连接通过 `async_connect()` 返回 `shared_transmission`，UDP 通过 `make_datagram_router()` 返回路由函数。两个方法覆盖所有代理场景。

**后果**: 新增传输类型（如 WebSocket）需要在 `shared_transmission` 层而非 `proxy` 层扩展。

### 为什么 proxy 是虚基类而非 type-erased 的 function？

**问题**: 需要多态接口，但有多种实现方式。

**选择**: 代理组的子代理数量在配置加载时确定，运行时不变。虚函数调用的间接跳转开销（~1ns）相比网络 I/O 延迟（~ms 级）可忽略。虚基类比 `std::function` 更清晰表达"多态接口"语义。

**后果**: 所有代理类型必须继承 `proxy` 并实现虚函数。

## 模块组成

| 组件 | 文件 | 说明 |
|------|------|------|
| [[core/outbound/proxy\|proxy]] | `proxy.hpp` | 出站代理抽象接口 |
| [[core/outbound/direct\|direct]] | `direct.hpp` | 直连出站代理实现 |

## 核心接口

### proxy 抽象接口

`proxy` 是所有出站协议和代理组必须实现的抽象接口。

| 方法 | 签名 | 说明 |
|------|------|------|
| async_connect | `virtual auto async_connect(const target&, const any_io_executor&) -> awaitable<pair<fault::code, shared_transmission>> = 0` | TCP 连接建立 |
| make_datagram_router | `virtual auto make_datagram_router() -> datagram_router_fn = 0` | UDP 数据报路由 |
| name | `virtual auto name() const -> string_view = 0` | 代理元信息 |
| supports_udp | `virtual auto supports_udp() const -> bool` | 是否支持 UDP（默认 true） |

### direct 直连实现

`direct` 是最简单的 `proxy` 实现，将连接请求直接通过 DNS 解析和 Happy Eyeballs 建立连接，不经过任何上游代理。

| 方法 | 说明 |
|------|------|
| 构造 | `explicit direct(resolve::router &router)` |
| async_connect | 通过 router DNS 解析 + Happy Eyeballs 建立直连 |
| make_datagram_router | 返回 UDP 路由函数 |
| name | 返回 "direct" |

## 架构关系

调用方 (worker) 持有 `proxy` 接口指针，具体实现包括 direct、socks5、trojan 等。direct 通过 router 进行 DNS 解析和连接池管理。

## 等价参考

| Prism 组件 | mihomo 等价 |
|-----------|-------------|
| `outbound::proxy` | `constant/adapters.go` 中的 `ProxyAdapter` 接口 |
| `outbound::direct` | `adapter/outbound/direct.go` |

## 参见

- [[core/resolve/router|router]] — 分发层路由器
- [[core/transport/transmission|transmission]] — 传输层抽象接口
- [[core/protocol/analysis|protocol::analysis]] — 协议分析模块
- [[core/fault/code|fault::code]] — 错误码定义
