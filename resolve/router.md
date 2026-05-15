---
title: "router.hpp — 分发层路由器"
source: "include/prism/resolve/router.hpp"
module: "resolve"
type: api
tags: [resolve, router, 路由, 连接池, DNS]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - channel/connection/pool
  - pipeline/primitives
  - fault/code
---

# router.hpp

> 源码: `include/prism/resolve/router.hpp` + `src/prism/resolve/router.cpp`
> 模块: [[resolve|Resolve]]

## 概述

分发层路由器，整合 [[resolve/dns/dns|DNS 解析器]]、反向路由表和 [[channel/connection/pool|连接池]] 等子组件，为上层提供统一的路由接口。路由器支持多种路由模式：反向代理路由、直连端点路由、正向代理路由和数据报路由。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[resolve/dns/dns|dns]] | DNS 解析器接口 |
| 依赖 | [[channel/connection/pool|pool]] | 连接池 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[agent/context|context]] | worker_context 持有 router |
| 被依赖 | [[pipeline/primitives|primitives]] | dial/forward 使用 router |

## 命名空间

`psm::resolve`

---

## 类: router

### 概述

分发层路由器，负责将各类路由请求分发到正确的处理路径。

### 设计意图

内部管理 [[resolve/dns/dns|DNS 解析器]]、反向路由表和 [[channel/connection/pool|连接池]]。根据请求类型选择处理路径：
- 反向代理请求通过查找预配置的路由表
- 直连请求直接通过传输源建立连接
- 正向代理请求通过 [[resolve/dns/dns|DNS 解析器]] 进行域名解析并建立连接
- 数据报请求通过 DNS 解析来获取目标地址

### 内部辅助结构体

#### `string_hash`

透明字符串哈希函数对象，支持 `std::string_view` 和 `memory::string` 的哈希计算，无需类型转换。通过 `is_transparent` 类型别名启用透明查找特性。

#### `string_equal`

透明字符串相等比较函数对象，支持 `std::string_view` 和 `memory::string` 的混合比较。提供四种 `operator()` 重载覆盖所有类型组合。

### 类型别名

```cpp
template <typename Value>
using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;
using reverse_map = hash_map<tcp::endpoint>;
```

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `connection_pool &` | `pool_` | 共享 TCP 传输源 |
| `memory::resource_pointer` | `mr_` | 内存资源 |
| `std::unique_ptr<dns::resolver>` | `dns_` | DNS 解析器 |
| `reverse_map` | `reverse_map_` | 反向路由表 |
| `net::any_io_executor` | `executor_` | 执行器（用于创建 UDP socket） |
| `std::optional<memory::string>` | `positive_host_` | 正向代理主机名 |
| `std::uint16_t` | `positive_port_` | 正向代理端口 |

---

### 函数: router()

- **功能说明**: 构造分发路由器，初始化 [[resolve/dns/dns|DNS 解析器]]（通过 `dns::make_resolver` 工厂）、反向路由表和 [[channel/connection/pool|连接池]] 引用。
- **签名**:
  ```cpp
  explicit router(connection_pool &pool, net::io_context &ioc, dns::config dns_cfg,
                  memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `pool` | `connection_pool &` | 共享 TCP 连接池，用于获取连接 |
  | `ioc` | `net::io_context &` | IO 上下文，用于创建执行器和定时器 |
  | `dns_cfg` | `dns::config` | DNS 解析器配置 |
  | `mr` | `memory::resource_pointer` | 内存资源，用于内部存储分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: `dns::make_resolver()` → 创建完整 DNS 解析管道
- **被调用（向上）**: [[agent/context|worker_context]] 在 worker 初始化时构造 router
- **知识域**: [[ref/network/connection-pool|连接池]]、[[resolve/dns/dns|DNS 解析]]

---

### 函数: set_positive_endpoint()

- **功能说明**: 设置正向代理的默认上游端点。配置后，正向代理请求将转发到该端点而非直接连接目标。传入空 host 或 port=0 时清除已有配置。
- **签名**:
  ```cpp
  void set_positive_endpoint(std::string_view host, std::uint16_t port);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 上游服务器主机名 |
  | `port` | `std::uint16_t` | 上游服务器端口 |

- **返回值**: 无
- **调用（向下）**: 无（仅设置内部成员变量 `positive_host_` 和 `positive_port_`）
- **被调用（向上）**: [[agent/context|worker_context]] 在配置加载时调用
- **知识域**: [[resolve/router|路由策略]]

---

### 函数: add_reverse_route()

- **功能说明**: 添加反向代理路由规则，将指定主机名的请求路由到给定的目标 TCP 端点。使用 `insert_or_assign` 语义，重复 host 会覆盖旧路由。
- **签名**:
  ```cpp
  void add_reverse_route(std::string_view host, const tcp::endpoint &ep);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 匹配的主机名 |
  | `ep` | `const tcp::endpoint &` | 目标 TCP 端点 |

- **返回值**: 无
- **调用（向下）**: 无（仅操作 `reverse_map_` 哈希表）
- **被调用（向上）**: [[agent/context|worker_context]] 在加载反向路由配置时调用
- **知识域**: [[resolve/router|路由策略]]

---

### 函数: async_reverse()

- **功能说明**: 异步路由反向代理请求。通过反向路由表查找目标端点，然后从 [[channel/connection/pool|连接池]] 获取连接。未找到路由时返回 `bad_gateway`。
- **签名**:
  ```cpp
  [[nodiscard]] auto async_reverse(std::string_view host) const
      -> net::awaitable<std::pair<fault::code, pooled_connection>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 目标主机名 |

- **返回值**: `net::awaitable<std::pair<fault::code, pooled_connection>>` — 错误码与连接的配对
- **调用（向下）**: `reverse_map_.find()` → `pool_.async_acquire(endpoint)`
- **被调用（向上）**: [[pipeline/primitives|primitives]] 中 `dial()` 在反向路由场景下调用
- **知识域**: [[resolve/router|路由策略]]、[[ref/network/connection-pool|连接池]]

---

### 函数: ipv6_disabled()

- **功能说明**: 查询是否禁用了 IPv6。委托给内部 [[resolve/dns/dns|DNS 解析器]] 的配置查询。
- **签名**:
  ```cpp
  [[nodiscard]] auto ipv6_disabled() const noexcept -> bool;
  ```
- **参数**: 无
- **返回值**: `bool` — 禁用 IPv6 返回 `true`，否则返回 `false`
- **调用（向下）**: `dns_->ipv6_disabled()`
- **被调用（向上）**: `async_forward()`、`async_datagram()`、`resolve_datagram_target()` 内部使用
- **知识域**: [[resolve/dns/config|DNS 配置]]

---

### 函数: async_direct()

- **功能说明**: 异步路由直连 TCP 端点。直接通过 [[channel/connection/pool|连接池]] 获取到指定端点的连接，无需任何 DNS 查找。
- **签名**:
  ```cpp
  [[nodiscard]] auto async_direct(tcp::endpoint ep) const
      -> net::awaitable<std::pair<fault::code, pooled_connection>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `ep` | `tcp::endpoint` | 目标 TCP 端点 |

- **返回值**: `net::awaitable<std::pair<fault::code, pooled_connection>>` — 错误码与连接的配对
- **调用（向下）**: `pool_.async_acquire(ep)`
- **被调用（向上）**: [[pipeline/primitives|primitives]] 中 `dial()` 在直连场景下调用
- **知识域**: [[ref/network/connection-pool|连接池]]

---

### 函数: async_forward()

- **功能说明**: 异步路由正向代理请求。首先检测 host 是否为 IP 字面量（IPv4/IPv6），若是则直接构造端点连接；否则通过 [[resolve/dns/dns|DNS 解析器]] 进行域名解析，再使用 Happy Eyeballs 竞速连接。IPv6 字面量在禁用 IPv6 时被拒绝。
- **签名**:
  ```cpp
  [[nodiscard]] auto async_forward(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, pooled_connection>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 目标主机名或 IP 字面量 |
  | `port` | `std::string_view` | 目标服务端口字符串 |

- **返回值**: `net::awaitable<std::pair<fault::code, pooled_connection>>` — 错误码与连接的配对
- **调用（向下）**: `dns_->resolve_tcp(host, port)` → `connect_with_retry(endpoints)` → `address_racer::race()`
- **被调用（向上）**: [[pipeline/primitives|primitives]] 中 `dial()` 在正向代理场景下调用
- **知识域**: [[resolve/dns/dns|DNS 解析]]、[[ref/network/connection-pool|连接池]]

---

### 函数: async_datagram()

- **功能说明**: 异步路由数据报请求。检测 host 是否为 IP 字面量，若是则直接构造 UDP 端点；否则通过 [[resolve/dns/dns|DNS 解析器]] 解析。最终通过 `open_udp_socket()` 创建并打开 UDP 套接字。
- **签名**:
  ```cpp
  [[nodiscard]] auto async_datagram(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, net::ip::udp::socket>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 目标主机名或 IP 字面量 |
  | `port` | `std::string_view` | 目标服务端口字符串 |

- **返回值**: `net::awaitable<std::pair<fault::code, net::ip::udp::socket>>` — 错误码与 UDP 套接字的配对
- **调用（向下）**: `dns_->resolve_udp(host, port)` → `open_udp_socket(executor_, target)`
- **被调用（向上）**: [[pipeline/primitives|primitives]] 中 UDP 转发场景调用
- **知识域**: [[resolve/dns/dns|DNS 解析]]

---

### 函数: resolve_datagram_target()

- **功能说明**: 解析数据报目标端点。仅返回解析后的 UDP 端点信息，不创建套接字。适用于需要延迟创建套接字的场景。
- **签名**:
  ```cpp
  [[nodiscard]] auto resolve_datagram_target(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 目标主机名或 IP 字面量 |
  | `port` | `std::string_view` | 目标服务端口字符串 |

- **返回值**: `net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>` — 错误码与 UDP 端点的配对
- **调用（向下）**: `dns_->resolve_udp(host, port)`
- **被调用（向上）**: [[pipeline/primitives|primitives]] 中 `make_datagram_router()` 调用
- **知识域**: [[resolve/dns/dns|DNS 解析]]

---

### 函数: async_positive() [private]

- **功能说明**: 路由到正向代理端点的内部方法。当前为 TODO 桩实现，直接返回 `not_supported`。
- **签名**:
  ```cpp
  [[nodiscard]] auto async_positive(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, pooled_connection>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 目标主机名 |
  | `port` | `std::string_view` | 目标服务端口 |

- **返回值**: `net::awaitable<std::pair<fault::code, pooled_connection>>` — 当前固定返回 `not_supported`
- **调用（向下）**: 无（TODO 桩）
- **被调用（向上）**: 未被调用（预留接口）
- **知识域**: [[resolve/router|路由策略]]

---

### 函数: connect_with_retry() [private]

- **功能说明**: 从端点列表中尝试连接。使用 RFC 8305 Happy Eyeballs 竞速策略（`address_racer`），并发尝试多个候选端点，返回首个成功连接。
- **签名**:
  ```cpp
  [[nodiscard]] auto connect_with_retry(std::span<const tcp::endpoint> endpoints)
      -> net::awaitable<pooled_connection>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `endpoints` | `std::span<const tcp::endpoint>` | 候选端点列表 |

- **返回值**: `net::awaitable<pooled_connection>` — 成功连接的包装连接，失败时返回无效连接
- **调用（向下）**: `address_racer::race(endpoints)`
- **被调用（向上）**: `async_forward()` 在 DNS 解析成功后调用
- **知识域**: [[ref/network/connection-pool|连接池]]

---

## 自由函数: open_udp_socket()

- **功能说明**: 打开 UDP 套接字。根据目标端点的地址类型自动选择 IPv4 或 IPv6 协议，创建并打开对应的 UDP 套接字。
- **签名**:
  ```cpp
  inline auto open_udp_socket(const net::any_io_executor &executor, const net::ip::udp::endpoint &target)
      -> std::pair<fault::code, net::ip::udp::socket>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `executor` | `const net::any_io_executor &` | 用于创建套接字的执行器 |
  | `target` | `const net::ip::udp::endpoint &` | 目标 UDP 端点，用于确定协议版本 |

- **返回值**: `std::pair<fault::code, net::ip::udp::socket>` — 结果码与 UDP 套接字的配对
- **调用（向下）**: `socket.open(protocol, ec)`
- **被调用（向上）**: `router::async_datagram()` 在获取目标端点后调用
- **知识域**: [[resolve/router|路由策略]]

---

## 调用链总览

```
[[pipeline/primitives|dial]]        → router::async_forward() / async_reverse() / async_direct()
[[pipeline/primitives|make_datagram]] → router::async_datagram() / resolve_datagram_target()

router::async_forward()   → dns_->resolve_tcp() → connect_with_retry() → address_racer::race()
router::async_datagram()  → dns_->resolve_udp() → open_udp_socket()
router::async_reverse()   → reverse_map_.find() → pool_.async_acquire()
router::async_direct()    → pool_.async_acquire()
```

---

## 知识域

- [[resolve/router|路由策略]]
- [[ref/network/connection-pool|连接池]]
- [[resolve/dns/dns|DNS 解析]]
