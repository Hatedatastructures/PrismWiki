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
    struct string_hash { /* is_transparent */ };
    struct string_equal { /* is_transparent */ };

    template <typename Value>
    using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;

    using reverse_map = hash_map<tcp::endpoint>;

    explicit router(router_options opts);

    void set_endpoint(std::string_view host, std::uint16_t port);
    void add_route(std::string_view host, const tcp::endpoint &ep);

    [[nodiscard]] auto async_reverse(std::string_view host) const
        -> net::awaitable<std::pair<fault::code, pooled_connection>>;

    [[nodiscard]] auto ipv6_disabled() const noexcept -> bool;
    [[nodiscard]] auto pool() noexcept -> connection_pool &;
    [[nodiscard]] auto dns() noexcept -> resolve::dns::resolver &;
    [[nodiscard]] auto executor() const noexcept -> net::any_io_executor;
};
```

### router_options 构造选项

```cpp
struct router_options
{
    connection_pool &pool;
    net::io_context &ioc;
    resolve::dns::config dns_cfg;
    memory::resource_pointer mr = memory::current_resource();
};
```

| 字段 | 说明 |
|------|------|
| `pool` | 共享 TCP 连接池，由 worker 持有，router 仅持有引用 |
| `ioc` | 当前 worker 的 `io_context`，用于创建执行器 |
| `dns_cfg` | DNS 解析器配置（上游服务器、缓存策略） |
| `mr` | PMR 内存资源，用于反向路由表分配 |

## 透明哈希机制

`router` 使用 C++20 透明哈希（`is_transparent`）实现 `string_view` 和 `memory::string` 的混合查找，避免临时字符串构造。

### 为什么需要透明哈希？

每个请求的路由查找需要用 `string_view`（来自协议解析的零拷贝视图）查 `unordered_map<memory::string, endpoint>`。普通哈希表查找会构造临时 `memory::string`，触发 PMR 分配。透明查找让 `string_view` 直接查表，热路径零分配。

**后果**: `string_hash` 和 `string_equal` 必须保证跨类型哈希一致性（统一用 `std::hash<string_view>` 计算哈希值）。

### hash_map 模板别名

```cpp
template <typename Value>
using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;
```

效果：可以用 `std::string_view` 作为 key 直接查找 `unordered_map<memory::string, ...>`，无需构造临时 `memory::string`。

## 构造函数

```cpp
explicit router(router_options opts);
```

**初始化流程：**

1. 保存连接池引用 `pool_`（外部持有，非拥有）
2. 初始化内存资源 `mr_`，默认使用 `memory::current_resource()`
3. 通过 `resolve::dns::make_resolver()` 创建 DNS 解析器 `dns_`
4. 使用 `mr_` 分配反向路由表 `reverse_map_`
5. 从 `ioc` 获取执行器 `executor_`

## 核心方法

### set_endpoint

```cpp
void set_endpoint(std::string_view host, std::uint16_t port);
```

设置正向代理的默认上游端点。`host` 为空或 `port` 为 0 时清除配置。主机名从 `string_view` 复制到 `memory::string`（PMR 分配）。

### add_route

```cpp
void add_route(std::string_view host, const tcp::endpoint &ep);
```

添加反向代理路由规则。使用 `insert_or_assign`，重复主机名覆盖。运行期间可调用（配置热加载）。

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
| `executor()` | `net::any_io_executor` | 获取执行器 |

## 设计决策

### 为什么反向路由表使用透明哈希？

详见上方「透明哈希机制」段落。核心原因：协议解析产生 `string_view`，路由表 key 是 `memory::string`，透明查找避免热路径 PMR 分配。

### 为什么 router 不拥有连接池？

连接池（`connection_pool`）是 worker 级别共享资源，生命周期由 worker 管理。router 只是连接池的消费者之一。如果 router 拥有连接池，router 的析构时序会影响连接归还。

**后果**: `pool_` 是非拥有引用，worker 必须保证在 router 析构之前不销毁连接池。

### 为什么 DNS 解析器由 router 拥有？

DNS 解析器需要 `io_context` 执行异步解析，且 router 是唯一的调用方。通过 `unique_ptr` 拥有可以隐藏 DNS 实现细节（前向声明），减少头文件依赖传播。

**后果**: router 析构时 DNS 解析器的异步操作必须已完成，否则 `unique_ptr` 析构会终止进行中的 DNS 查询。

## 约束

### 线程安全

**类型**: 线程安全

**规则**: router 不是线程安全的，所有方法必须在同一个 strand/io_context 线程中调用。每个 worker 拥有独立的 router 实例。

**违反后果**: `reverse_map_` 的并发读写导致数据竞争，未定义行为。

**源码依据**: `router.hpp:49`

### 连接池生命周期

**类型**: 生命周期

**规则**: `pool_` 为非拥有引用，连接池的生命周期必须长于 router。

**违反后果**: router 通过悬挂引用访问已销毁的连接池，崩溃。

**源码依据**: `router.hpp:201`

### 反向路由表运行时修改

**类型**: 线程安全

**规则**: `add_route()` 可在运行期间调用（配置热加载），但必须在 router 所属的 io_context 线程上执行。

**违反后果**: 与 `async_reverse()` 并发访问 `reverse_map_`，数据竞争。

**源码依据**: `router.hpp:142`

## 故障场景

### 反向路由表未命中

**触发条件**: 请求的主机名不在反向路由表中（未配置或配置错误）。

**传播路径**: `async_reverse()` → `reverse_map_.find()` 未命中 → 返回 `bad_gateway` + 空连接 → `dial()` 返回错误 → 客户端连接失败。

**外部表现**: 特定域名始终连接失败。日志出现 `route failed: bad_gateway`。

**恢复机制**: 通过 `add_route()` 补充路由规则，或使用正向路由（`positive = true`）。

**日志关键字**: `bad_gateway` + `route failed`

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

1. **线程安全**：router 不是线程安全的，每个 worker 拥有独立实例
2. **生命周期**：`pool_` 为非拥有引用，连接池生命周期必须覆盖 router
3. **反向路由表修改**：`add_route()` 可运行期间调用，但需在正确的线程上执行
4. **透明哈希**：`find()` 接受 `string_view` 无需构造临时字符串
5. **DNS 解析器所有权**：`dns_` 为 `unique_ptr`，router 拥有完整生命周期
