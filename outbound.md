---
title: "Outbound 模块"
module: "outbound"
type: module
tags: [outbound, proxy, direct, 出站, 代理, 协程]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/router
  - channel/transport/transmission
  - channel/transport/reliable
  - channel/connection/pool
  - protocol/analysis
  - fault/code
---

# Outbound 模块

> 模块: [[outbound|outbound]]

## 概述

Outbound 模块定义出站代理接口和实现。所有出站协议（direct, socks5, http, trojan, vless, ss）以及代理组（url_test, fallback, load_balance, selector）必须实现 `proxy` 接口。采用纯协程设计，使用 `net::awaitable` 作为异步操作返回类型。通过组合模式使代理组和单个代理共享同一接口。

---

## proxy.hpp — 出站代理抽象接口

> 源码: `include/prism/outbound/proxy.hpp`
> 模块: [[outbound|outbound]]

### 概述

出站代理抽象接口。定义所有出站协议和代理组必须实现的核心接口。核心方法包括 TCP 连接建立和 UDP 路由回调。等价于 mihomo 的 `C.ProxyAdapter` 接口。所有异步操作通过 `std::error_code&` 参数返回错误，避免异常开销。

### 命名空间

`psm::outbound`

### 类型别名: datagram_router_fn

- **功能说明**: UDP 数据报路由回调类型。接受 (host, port) 返回 (错误码, udp_endpoint)。
- **签名**: `using datagram_router_fn = std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(std::string_view, std::string_view)>`
- **参数**: host — 目标主机名；port — 目标端口
- **返回值**: `awaitable<pair<fault::code, udp::endpoint>>` — 协程对象，完成后返回错误码和 UDP 端点
- **调用（向下）**: 无（类型别名）
- **被调用（向上）**: `proxy::make_datagram_router()` 返回值、UDP 代理路径
- **知识域**: UDP 路由回调抽象

### 类: proxy

- **功能说明**: 出站代理抽象接口。所有出站协议和代理组必须实现此接口。设计特性：纯协程、错误码返回、组合模式。
- **签名**: `class proxy`

#### proxy::~proxy() [virtual]

- **功能说明**: 虚析构函数，确保子类正确析构。
- **签名**: `virtual ~proxy() = default`
- **参数**: 无
- **返回值**: 不适用
- **调用（向下）**: 无
- **被调用（向上）**: 子类析构链
- **知识域**: 多态析构

#### proxy::async_connect() [pure virtual]

- **功能说明**: 建立 TCP 连接到目标。实现类内部决定路由策略：直连走 DNS 解析 + 连接池，代理走上游协议握手，代理组走子代理选择。
- **签名**: `virtual auto async_connect(const protocol::analysis::target &target, const net::any_io_executor &executor) -> net::awaitable<std::pair<fault::code, shared_transmission>> = 0`
- **参数**: `target` — 目标地址信息（host + port + positive 标记）；`executor` — 用于创建连接的执行器
- **返回值**: `awaitable<pair<fault::code, shared_transmission>>` — 协程对象，完成后返回 (错误码, 传输对象) 对
- **调用（向下）**: 子类实现决定
- **被调用（向上）**: `pipeline::primitives::dial()`、协议处理器的出站连接
- **知识域**: 出站 TCP 连接抽象

#### proxy::make_datagram_router() [pure virtual]

- **功能说明**: 创建 UDP 数据报路由回调。替代直接传递 router 引用，将 DNS 解析封装在实现内部。
- **签名**: `virtual auto make_datagram_router() -> datagram_router_fn = 0`
- **参数**: 无
- **返回值**: `datagram_router_fn` — 回调函数
- **调用（向下）**: 子类实现决定
- **被调用（向上）**: UDP 代理路径、`pipeline::primitives` UDP 路由
- **知识域**: UDP 路由回调工厂

#### proxy::name() [pure virtual]

- **功能说明**: 获取代理名称。
- **签名**: `[[nodiscard]] virtual auto name() const -> std::string_view = 0`
- **参数**: 无
- **返回值**: `std::string_view` — 代理名称的字符串视图
- **调用（向下）**: 无
- **被调用（向上）**: 日志记录、代理选择器显示
- **知识域**: 代理标识

#### proxy::supports_udp()

- **功能说明**: 是否支持 UDP。默认返回 true，不支持 UDP 的代理应重写此方法。
- **签名**: `[[nodiscard]] virtual auto supports_udp() const -> bool`
- **参数**: 无
- **返回值**: `bool` — 默认 true
- **调用（向下）**: 无
- **被调用（向上）**: UDP 代理路径、代理组 UDP 支持检查
- **知识域**: UDP 能力查询

---

## direct.hpp — 直连出站代理

> 源码: `include/prism/outbound/direct.hpp`
> 模块: [[outbound|outbound]]

### 概述

直连出站代理。将 `proxy` 接口映射到 `resolve::router` 路由，是最简单的 proxy 实现。所有连接请求直接通过 DNS 解析和 Happy Eyeballs 建立连接，不经过任何上游代理。等价于 mihomo 的 `adapter/outbound/direct.go`。每个 worker 持有一个实例。

### 类: direct

- **功能说明**: 直连出站代理。路由策略：`target.positive == false` 时尝试反向路由（域名到预配置后端），否则走正向路由（DNS 解析 + TCP 连接）。完整复刻了原 `primitives::dial()` 的行为逻辑，包括 IPv6 禁用检查和连接有效性验证。
- **签名**: `class direct : public proxy`

#### direct::direct(router&)

- **功能说明**: 构造直连出站代理。
- **签名**: `explicit direct(resolve::router &router)`
- **参数**: `router` — 路由器引用，用于 DNS 解析和连接建立
- **返回值**: 不适用（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `worker::worker` 构造中创建 `outbound_direct_`
- **知识域**: 直连代理构造

#### direct::async_connect()

- **功能说明**: 建立 TCP 连接到目标。流程：检查 IPv6 禁用 -> 判断正向/反向路由 -> 调用 router 的 async_forward/async_reverse -> 验证连接有效性 -> 包装为 reliable 传输。
- **签名**: `auto async_connect(const protocol::analysis::target &target, const net::any_io_executor &executor) -> net::awaitable<std::pair<fault::code, shared_transmission>> override`
- **参数**: `target` — 目标地址信息；`executor` — 用于创建连接的执行器
- **返回值**: `awaitable<pair<fault::code, shared_transmission>>` — 协程对象，完成后返回 (错误码, 传输对象) 对
- **调用（向下）**: `router_.ipv6_disabled()`、`is_ipv6_literal()`、`router_.async_reverse()`、`router_.async_forward()`、`fault::failed()`、`channel::transport::make_reliable()`、`trace::debug/info/warn()`
- **被调用（向上）**: `pipeline::primitives::dial()` 通过 `proxy` 接口调用
- **知识域**: 直连 TCP 连接建立

#### direct::make_datagram_router()

- **功能说明**: 创建 UDP 数据报路由回调。返回一个 lambda，内部委托给 `router_.resolve_datagram_target()`。使用非拥有指针（空删除器）包装 router 引用。
- **签名**: `auto make_datagram_router() -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(std::string_view, std::string_view)> override`
- **参数**: 无
- **返回值**: `datagram_router_fn` — UDP 路由回调函数
- **调用（向下）**: `router_.resolve_datagram_target()`
- **被调用（向上）**: UDP 代理路径
- **知识域**: 直连 UDP 路由

#### direct::name()

- **功能说明**: 获取代理名称。返回固定值 `"DIRECT"`。
- **签名**: `[[nodiscard]] auto name() const -> std::string_view override`
- **参数**: 无
- **返回值**: `std::string_view` — 固定为 `"DIRECT"`
- **调用（向下）**: 无
- **被调用（向上）**: 日志记录、代理标识
- **知识域**: 直连代理标识

#### direct::is_ipv6_literal() [private, static]

- **功能说明**: 检查目标地址是否为 IPv6 字面量。使用 `boost::asio::ip::make_address()` 解析并判断。
- **签名**: `static auto is_ipv6_literal(std::string_view host) noexcept -> bool`
- **参数**: `host` — 目标主机名或 IP 地址
- **返回值**: `bool` — 如果是 IPv6 地址字面量返回 true
- **调用（向下）**: `net::ip::make_address()`
- **被调用（向上）**: `direct::async_connect()`
- **知识域**: IPv6 地址检测

---

## outbound 概述

Outbound 模块的详细设计文档。`proxy` 抽象接口定义了出站代理的核心契约（async_connect + make_datagram_router + name + supports_udp），`direct` 是最简单的实现，将请求直接路由到目标。代理组（url_test, fallback 等）也实现 `proxy` 接口，通过组合模式委托给子代理。

## 相关页面

- [[resolve/router|router]] — DNS 路由，direct 内部使用
- [[channel/transport/transmission|transmission]] — 传输抽象，async_connect 返回的传输对象
- [[channel/transport/reliable|reliable]] — 可靠传输包装，direct 使用 make_reliable() 包装连接
- [[channel/connection/pool|pool]] — 连接池，router 内部管理
- [[protocol/analysis|analysis]] — 协议分析，target 结构定义
- [[fault/code|fault::code]] — 错误码，async_connect 返回的错误类型
