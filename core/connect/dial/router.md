---
title: router
layer: core
source: I:/code/Prism/include/prism/connect/dial/router.hpp
module: connect
tags:
  - router
  - routing
  - reverse-proxy
  - dns
  - connection-pool
created: 2026-05-23
updated: 2026-05-23
---

# router

连接路由器，管理反向路由映射、正向代理目标和 DNS 解析。作为 connect 层的顶层协调器，整合 DNS 解析器、反向路由表和连接池，为上层提供统一的路由接口。

## 概述

`router` 是分发层到连接层的桥梁。它持有三大核心子组件：

- **反向路由表**（`reverse_map`）：主机名到 TCP 端点的静态映射，用于反向代理场景
- **DNS 解析器**（`dns_`）：域名解析，用于正向代理和数据报路由
- **连接池**（`pool_`）：TCP 连接复用，减少握手开销

路由器支持四种路由模式：

| 模式 | 方法 | 场景 |
|------|------|------|
| 反向代理路由 | `async_reverse()` | 根据主机名查表获取目标端点 |
| 直连端点路由 | `pool().async_acquire(ep)` | 已知 IP 端点，直接从池获取连接 |
| 正向代理路由 | `set_positive_endpoint()` | 配置默认上游代理端点 |
| 数据报路由 | 委托给 DNS 解析 | UDP 场景的域名解析 |

## 类定义

```cpp
class router
{
public:
    // ── 透明哈希支持 ──
    struct string_hash { /* is_transparent, 支持 string_view 和 memory::string */ };
    struct string_equal { /* is_transparent, 混合比较 */ };

    template <typename Value>
    using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;

    using reverse_map = hash_map<tcp::endpoint>;

    // ── 构造 ──
    explicit router(connection_pool &pool, net::io_context &ioc,
                    resolve::dns::config dns_cfg,
                    memory::resource_pointer mr = memory::current_resource());

    // ── 正向代理配置 ──
    void set_positive_endpoint(std::string_view host, std::uint16_t port);

    // ── 反向路由表操作 ──
    void add_reverse_route(std::string_view host, const tcp::endpoint &ep);

    // ── 异步路由 ──
    [[nodiscard]] auto async_reverse(std::string_view host) const
        -> net::awaitable<std::pair<fault::code, pooled_connection>>;

    // ── 访问器 ──
    [[nodiscard]] auto ipv6_disabled() const noexcept -> bool;
    [[nodiscard]] auto pool() noexcept -> connection_pool &;
    [[nodiscard]] auto dns() noexcept -> resolve::dns::resolver &;
    [[nodiscard]] auto executor() const noexcept -> net::any_io_executor;
    [[nodiscard]] auto positive_host() const noexcept -> const std::optional<memory::string> &;
    [[nodiscard]] auto positive_port() const noexcept -> std::uint16_t;

private:
    connection_pool &pool_;
    memory::resource_pointer mr_;
    std::unique_ptr<resolve::dns::resolver> dns_;
    reverse_map reverse_map_;
    net::any_io_executor executor_;
    std::optional<memory::string> positive_host_;
    std::uint16_t positive_port_{0};
};
```

## 透明哈希机制

`router` 使用 C++20 透明哈希（`is_transparent`）实现 `string_view` 和 `memory::string` 的混合查找，避免临时字符串构造。

### string_hash

```cpp
struct string_hash
{
    using is_transparent = void;

    [[nodiscard]] auto operator()(const std::string_view value) const noexcept -> std::size_t;
    [[nodiscard]] auto operator()(const memory::string &value) const noexcept -> std::size_t;
};
```

对 `std::string_view` 和 `memory::string` 统一使用 `std::hash<std::string_view>` 计算哈希，保证跨类型哈希一致性。

### string_equal

```cpp
struct string_equal
{
    using is_transparent = void;

    [[nodiscard]] auto operator()(const std::string_view left, const std::string_view right) const noexcept -> bool;
    [[nodiscard]] auto operator()(const memory::string &left, const std::string_view right) const noexcept -> bool;
    [[nodiscard]] auto operator()(const std::string_view left, const memory::string &right) const noexcept -> bool;
    [[nodiscard]] auto operator()(const memory::string &left, const memory::string &right) const noexcept -> bool;
};
```

四种组合覆盖所有混合比较场景，所有比较最终委托给 `string_view` 的 `operator==`。

### hash_map 模板别名

```cpp
template <typename Value>
using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;
```

效果：可以用 `std::string_view` 作为 key 直接查找 `unordered_map<memory::string, ...>`，无需构造临时 `memory::string`。

```cpp
// 查找示例（无需构造 memory::string）
std::string_view host = "example.com";
auto it = reverse_map_.find(host);  // 直接用 string_view 查找
```

## 构造函数

```cpp
explicit router(connection_pool &pool, net::io_context &ioc,
                resolve::dns::config dns_cfg,
                memory::resource_pointer mr = memory::current_resource());
```

**初始化流程：**

1. 保存连接池引用 `pool_`（外部持有，非拥有）
2. 初始化内存资源 `mr_`，默认使用 `memory::current_resource()`
3. 通过 `resolve::dns::make_resolver()` 创建 DNS 解析器 `dns_`
4. 使用 `mr_` 分配反向路由表 `reverse_map_`
5. 从 `ioc` 获取执行器 `executor_`

**依赖关系：**
- `pool` 由 [[core/connect/pool/pool|connection_pool]] 提供，通常是 worker 级别共享
- `ioc` 为当前 worker 的 `io_context`
- `dns_cfg` 为 DNS 配置，包含上游服务器列表、缓存策略等

## 核心方法

### set_positive_endpoint

```cpp
void set_positive_endpoint(std::string_view host, std::uint16_t port);
```

设置正向代理的默认上游端点。配置后，正向代理请求将转发到该端点而非直接连接目标。

**行为：**
- `host` 为空或 `port` 为 0 时清除正向代理配置
- 主机名从 `string_view` 复制到 `memory::string`（使用 PMR 分配）

### add_reverse_route

```cpp
void add_reverse_route(std::string_view host, const tcp::endpoint &ep);
```

添加反向代理路由规则。将指定主机名的请求路由到给定的目标端点。

**行为：**
- 主机名从 `string_view` 复制到 `memory::string`
- 使用 `insert_or_assign`，重复主机名会覆盖之前的端点
- 运行期间可被修改（用于配置热加载）

### async_reverse

```cpp
[[nodiscard]] auto async_reverse(std::string_view host) const
    -> net::awaitable<std::pair<fault::code, pooled_connection>>;
```

异步路由反向代理请求。通过反向路由表查找目标端点并建立连接。

**流程：**

```
async_reverse(host)
    |
    v
[reverse_map_.find(host)]
    |
    +-- 未找到 --> return bad_gateway + 空 conn
    |
    +-- 找到 --> pool_.async_acquire(endpoint)
                    |
                    +-- 成功 --> return success + conn
                    +-- 失败 --> return bad_gateway + 空 conn
```

**关键细节：**
- 使用透明哈希查找，`host` 为 `string_view` 无需构造 `memory::string`
- 找到路由后委托给 [[core/connect/pool/pool|connection_pool.async_acquire]]
- 路由表未命中直接返回 `bad_gateway`，不回退到 DNS 解析
- 方法为 `const`，表明不修改路由表状态

**返回值：**
- `fault::code::success` + 有效连接：成功
- `fault::code::bad_gateway` + 空连接：路由表未命中或连接失败

### 访问器方法

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `ipv6_disabled()` | `bool` | 委托给 DNS 解析器的 IPv6 配置查询 |
| `pool()` | `connection_pool &` | 获取连接池引用 |
| `dns()` | `resolve::dns::resolver &` | 获取 DNS 解析器引用 |
| `executor()` | `net::any_io_executor` | 获取执行器（用于创建 UDP socket） |
| `positive_host()` | `const optional<memory::string> &` | 获取正向代理主机名 |
| `positive_port()` | `uint16_t` | 获取正向代理端口 |

## 内部结构

### 数据成员

```cpp
connection_pool &pool_;                             // 连接池引用（非拥有）
memory::resource_pointer mr_;                       // PMR 内存资源
std::unique_ptr<resolve::dns::resolver> dns_;       // DNS 解析器（拥有）
reverse_map reverse_map_;                           // 反向路由表（透明哈希）
net::any_io_executor executor_;                     // 执行器
std::optional<memory::string> positive_host_;       // 正向代理主机名
std::uint16_t positive_port_{0};                    // 正向代理端口
```

### 反向路由表类型

```cpp
using reverse_map = hash_map<tcp::endpoint>;
// 展开后:
// unordered_map<
//     memory::string,
//     tcp::endpoint,
//     string_hash,      // 透明哈希
//     string_equal       // 透明比较
// >
```

## 四种路由模式详解

### 1. 反向代理路由

通过 `async_reverse()` 实现。查找预配置的主机名到端点映射表。

**使用场景：** 透明代理、伪装域名路由

**调用方：** [[core/connect/dial/dial|dial()]] 函数在 `allow_reverse = true` 且 `!target.positive` 时调用

### 2. 直连端点路由

直接调用 `pool().async_acquire(ep)` 获取连接，不经过 DNS 解析。

**使用场景：** IP 字面量目标、反向路由表命中后连接

**调用方：** [[core/connect/dial/dial|async_direct()]]、`async_reverse()` 内部

### 3. 正向代理路由

通过 `set_positive_endpoint()` 配置默认上游代理。正向代理模式下所有请求转发到配置的上游端点。

**使用场景：** 代理链、上游转发

> [!note]
> 当前 `set_positive_endpoint()` 已实现，但正向代理路由的完整逻辑（`async_forward` 中判断 `positive_host_`）尚未完全集成。

### 4. 数据报路由

通过 `dns()` 访问 DNS 解析器，解析 UDP 目标端点。

**使用场景：** SOCKS5 UDP ASSOCIATE、DNS 查询转发

**调用方：** [[core/connect/dial/dial|async_datagram()]]、`resolve_datagram_target()`

## 调用链

```
上层（dial / forward）
    |
    v
router
    |
    +-- async_reverse(host)
    |       |
    |       v
    |   reverse_map_.find(host)
    |       |
    |       v
    |   pool_.async_acquire(endpoint)  ---> [[core/connect/pool/pool]]
    |
    +-- dns().resolve_tcp(host, port)  ---> [[core/resolve/dns/dns]]
    |
    +-- dns().resolve_udp(host, port)  ---> [[core/resolve/dns/dns]]
    |
    +-- pool().async_acquire(endpoint) ---> [[core/connect/pool/pool]]
```

## 时序图：反向路由

```
dial()              router              reverse_map       pool
  |                   |                     |               |
  |--async_reverse()-->|                     |               |
  |                   |--find(host)-------->|               |
  |                   |                     |               |
  |                   |<--endpoint----------|               |
  |                   |                     |               |
  |                   |--async_acquire(ep)->|               |
  |                   |                     |--acquire()---->|
  |                   |                     |               |
  |                   |                     |<--conn--------|
  |                   |                     |               |
  |<--success+conn----|                     |               |
  |                   |                     |               |
```

## 时序图：正向路由（DNS 解析 + 竞速连接）

```
dial()              router              DNS              racer            pool
  |                   |                  |                 |                |
  |--dns()----------->|                  |                 |                |
  |                   |--resolve_tcp()-->|                 |                |
  |                   |                  |                 |                |
  |                   |<--endpoints------|                 |                |
  |                   |                  |                 |                |
  |<--endpoints-------|                  |                 |                |
  |                   |                  |                 |                |
  |--async_connect_with_retry()          |                 |                |
  |                   |                  |                 |--race()------->|
  |                   |                  |                 |                |
  |                   |                  |                 |<--conn---------|
  |                   |                  |                 |                |
  |<--conn------------------------------------------------|                |
  |                   |                  |                 |                |
```

## 与其他模块的关系

- [[core/connect/dial/dial]]：拨号入口，是路由器的主要调用方
- [[core/connect/dial/racer]]：竞速连接器，通过连接池间接使用
- [[core/connect/pool/pool]]：TCP 连接池，router 持有其引用
- [[core/connect/pool/config]]：连接池配置
- [[core/connect/pool/health]]：连接池健康检测
- [[core/resolve/dns/dns]]：DNS 解析器，router 拥有其实例
- [[core/context/context]]：会话上下文，worker 层持有 router 实例
- [[core/protocol/common/target]]：目标地址结构体，路由决策的输入

## 注意事项

1. **线程安全**：router 不是线程安全的，应在单个 strand 上下文中使用。每个 worker 拥有独立的 router 实例。
2. **生命周期**：`pool_` 为非拥有引用，调用方需确保连接池的生命周期覆盖 router。
3. **反向路由表修改**：`add_reverse_route()` 可在运行期间调用（配置热加载），但需确保在正确的线程/strand 上下文中执行。
4. **透明哈希**：反向路由表使用透明哈希，`find()` 接受 `string_view` 无需构造临时字符串，降低热路径分配。
5. **DNS 解析器所有权**：`dns_` 为 `unique_ptr`，router 拥有 DNS 解析器的完整生命周期。
