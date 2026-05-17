---
title: "router — 分发层路由器"
layer: core
source: "I:/code/Prism/include/prism/resolve/router.hpp"
module: "resolve"
type: component
tags: [router, resolve, dispatch, routing]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/dns
  - core/channel/connection/pool
  - core/channel/eyeball/racer
---

# router — 分发层路由器

> 源码位置: `I:/code/Prism/include/prism/resolve/router.hpp`
> 实现文件: `I:/code/Prism/src/prism/resolve/router.cpp`
> 模块: [[resolve|resolve]]

## 组件定位

`router` 是分发层的顶层门面，整合了 DNS 解析器、反向路由表和连接池等子组件，为上层提供统一的路由接口。路由器支持四种路由模式：反向代理路由、直连端点路由、正向代理路由和数据报路由。

## 类型定义

### 透明哈希映射

```cpp
// 透明字符串哈希函数对象（支持 string_view 和 memory::string 混合查找）
struct string_hash {
    using is_transparent = void;
    auto operator()(std::string_view value) const noexcept -> std::size_t;
    auto operator()(const memory::string &value) const noexcept -> std::size_t;
};

// 透明字符串相等比较函数对象
struct string_equal {
    using is_transparent = void;
    auto operator()(std::string_view left, std::string_view right) const noexcept -> bool;
    auto operator()(const memory::string &left, std::string_view right) const noexcept -> bool;
    // ... 其他重载
};

// 支持异构查找的哈希表
template <typename Value>
using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;

using reverse_map = hash_map<tcp::endpoint>;  // 反向路由表类型
```

**设计决策**: 使用透明哈希（transparent hash）避免 `string_view → string` 的临时对象构造开销，在路由查找热路径实现零堆分配。

## 核心方法

### 构造函数

```cpp
explicit router(connection_pool &pool, net::io_context &ioc, dns::config dns_cfg,
                memory::resource_pointer mr = memory::current_resource());
```

初始化流程：
1. 绑定共享 TCP 连接池 `pool_`
2. 创建 DNS 解析器 `dns_ = dns::make_resolver(ioc, dns_cfg, mr)`
3. 初始化反向路由表 `reverse_map_`
4. 获取执行器用于 UDP socket 创建

### 四种路由模式

| 方法 | 用途 | 流程 |
|------|------|------|
| `async_reverse(host)` | 反向代理路由 | 查表 → 获取端点 → 连接 |
| `async_direct(ep)` | 直连端点 | 直接连接 |
| `async_forward(host, port)` | 正向代理 | DNS 解析 → 端点竞速 → 连接 |
| `async_datagram(host, port)` | 数据报 | DNS 解析 → 创建 UDP socket |

### async_forward 实现详解

```cpp
auto router::async_forward(std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, pooled_connection>>
{
    // 第一步：检测 IP 字面量（IPv4 或 IPv6）
    {
        boost::system::error_code ec;
        const auto addr = net::ip::make_address(host, ec);
        if (!ec) {
            // IPv6 地址检查：如果禁用 IPv6 则拒绝
            if (addr.is_v6() && ipv6_disabled()) {
                co_return std::make_pair(fault::code::host_unreachable, pooled_connection{});
            }
            // IP 字面量：直接构造 endpoint 连接，无需 DNS 解析
            const auto port_num = parse_port(port).value_or(0);
            const tcp::endpoint ep(addr, port_num);
            auto [code, conn] = co_await pool_.async_acquire(ep);
            // ...
        }
    }

    // 第二步：非字面量，需要进行 DNS 解析
    auto [resolve_ec, endpoints] = co_await dns_->resolve_tcp(host, port);
    if (fault::failed(resolve_ec) || endpoints.empty()) {
        co_return std::make_pair(fault::code::host_unreachable, pooled_connection{});
    }

    // 第三步：Happy Eyeballs 端点竞速连接
    auto conn = co_await connect_with_retry(endpoints);
    // ...
}
```

**实现要点**:
1. IP 字面量检测避免不必要的 DNS 查询
2. IPv6 禁用时直接拒绝 IPv6 字面量
3. 使用 Happy Eyeballs 算法连接多端点

### connect_with_retry 实现

```cpp
auto router::connect_with_retry(std::span<const tcp::endpoint> endpoints)
    -> net::awaitable<pooled_connection>
{
    // RFC 8305 Happy Eyeballs：staggered delay 竞速
    channel::eyeball::address_racer racer(pool_);
    co_return co_await racer.race(endpoints);
}
```

委托给 [[core/channel/eyeball/racer|address_racer]] 实现并发竞速连接。

### async_datagram 实现

```cpp
auto router::async_datagram(std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, net::ip::udp::socket>>
{
    // 检测 IP 字面量，直接构造 endpoint
    net::ip::udp::endpoint target;
    {
        boost::system::error_code ec;
        const auto addr = net::ip::make_address(host, ec);
        if (!ec) {
            // IPv6 禁用检查
            if (addr.is_v6() && ipv6_disabled()) {
                co_return std::pair{fault::code::host_unreachable, net::ip::udp::socket{executor_}};
            }
            target = net::ip::udp::endpoint(addr, parse_port(port).value_or(0));
        } else {
            // 需要解析
            const auto [resolve_ec, resolved] = co_await dns_->resolve_udp(host, port);
            // ...
        }
    }
    co_return open_udp_socket(executor_, target);
}
```

## 辅助函数

### open_udp_socket

```cpp
inline auto open_udp_socket(const net::any_io_executor &executor,
                            const net::ip::udp::endpoint &target)
    -> std::pair<fault::code, net::ip::udp::socket>
{
    const auto protocol = target.address().is_v6() 
        ? net::ip::udp::v6() 
        : net::ip::udp::v4();
    net::ip::udp::socket socket(executor);
    socket.open(protocol, ec);
    // ...
}
```

根据目标地址类型自动选择 IPv4 或 IPv6 协议。

## 成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `pool_` | `connection_pool &` | 共享 TCP 连接池引用 |
| `mr_` | `memory::resource_pointer` | 内存资源 |
| `dns_` | `std::unique_ptr<dns::resolver>` | DNS 解析器实例 |
| `reverse_map_` | `reverse_map` | 反向路由表 |
| `executor_` | `net::any_io_executor` | 执行器（用于 UDP socket） |
| `positive_host_` | `std::optional<memory::string>` | 正向代理主机名 |
| `positive_port_` | `std::uint16_t` | 正向代理端口 |

## 调用链

```
router::async_forward
  │
  ├─→ IP 字面量分支
  │     → pool_.async_acquire(endpoint)
  │     → pooled_connection
  │
  └─→ 域名解析分支
        → dns_->resolve_tcp(host, port)
        │     ├─→ rules_engine::match()
        │     ├─→ cache::get()
        │     ├─→ upstream::resolve()
        │     └─→ cache::put()
        │
        → connect_with_retry(endpoints)
              → address_racer::race()
                    → pool_.async_acquire() × N (竞速)
                    → 第一个成功 wins
```

## 设计决策

### 透明哈希优化

反向路由表使用透明哈希，允许 `string_view` 直接查找，避免构造临时 `memory::string`：

```cpp
// 传统方式（需要临时对象构造）
reverse_map_.find(memory::string(host));  // 有堆分配

// 透明哈希（零分配）
reverse_map_.find(host);  // 直接使用 string_view
```

### IP 字面量短路

在 `async_forward` 和 `async_datagram` 中，优先检测 IP 字面量：

- 若 host 是有效 IP 地址，直接构造 endpoint
- 无需发起 DNS 查询，节省时间和资源
- 立即通过连接池建立连接

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/channel/connection/pool|connection_pool]] — TCP 连接池
- [[core/channel/eyeball/racer|address_racer]] — Happy Eyeballs 竞速器
- [[core/resolve/dns/detail/utility|parse_port]] — 零分配端口解析