---
title: dial
layer: core
source: I:/code/Prism/include/prism/connect/dial/dial.hpp
module: connect
tags:
  - dial
  - routing
  - dns
  - connection
  - happy-eyeballs
created: 2026-05-23
updated: 2026-05-23
---

# dial

连接拨号入口模块，提供 TCP 拨号、UDP 数据报和路由回调等自由函数，整合 DNS 解析、端点排序和竞速连接。

## 概述

`dial.hpp` 是 connect 层的拨号入口，将路由决策、DNS 解析、Happy Eyeballs 竞速连接统一封装为一组异步自由函数。上层协议处理器（SOCKS5、Trojan、VLESS 等）通过 `dial()` 或 `forward()` 函数建立到上游服务器的连接，无需关心底层路由选择逻辑。

核心职责：

- **路由选择**：根据 `protocol::target` 的 `positive` 标志选择反向路由或正向 DNS 解析路由
- **DNS 解析**：对非 IP 字面量的主机名调用 DNS 解析器获取端点列表
- **竞速连接**：通过 [[core/connect/dial/racer]] 实现 RFC 8305 Happy Eyeballs 并发连接
- **传输包装**：将 `pooled_connection` 包装为 `shared_transmission` 可靠传输对象
- **出站代理**：支持通过 `outbound::proxy` 进行间接连接

## 函数签名

### is_ipv6_literal

```cpp
inline auto is_ipv6_literal(const std::string_view host) noexcept -> bool;
```

检查目标地址是否为 IPv6 字面量。通过 `net::ip::make_address()` 尝试解析，成功且为 v6 则返回 `true`。

**参数：**
- `host`：目标主机名或 IP 地址字符串视图

**返回值：** IPv6 地址字面量返回 `true`，否则 `false`

**用途：** 在 IPv6 被禁用时提前拦截 IPv6 字面量目标，避免不必要的连接尝试。

### retry_connect

```cpp
auto retry_connect(router &rt, std::span<const tcp::endpoint> endpoints)
    -> net::awaitable<pooled_connection>;
```

从端点列表中尝试连接，委托给 [[core/connect/dial/racer|address_racer]] 进行竞速连接。

**参数：**
- `rt`：路由器引用，提供连接池访问
- `endpoints`：候选端点列表，通常由 DNS 解析返回

**返回值：** 成功连接的套接字（`valid() == true`），或无效 `pooled_connection`（全部失败时）

**流程：**

1. 空列表检查：`endpoints` 为空直接返回空连接
2. 创建 `address_racer` 实例
3. 调用 `racer.race(endpoints)` 进行竞速连接
4. 返回第一个成功的连接

### async_direct

```cpp
auto async_direct(router &rt, tcp::endpoint ep)
    -> net::awaitable<std::pair<fault::code, pooled_connection>>;
```

异步路由直连 TCP 端点。直接从连接池获取到指定端点的连接，不经过 DNS 解析。

**参数：**
- `rt`：路由器引用
- `ep`：目标 TCP 端点

**返回值：**
- `fault::code::success` + 有效连接：连接成功
- `fault::code::bad_gateway` + 空连接：连接失败

**流程：**

1. 调用 `rt.pool().async_acquire(ep)` 从连接池获取连接
2. 检查连接有效性
3. 返回结果码和连接

### async_forward

```cpp
auto async_forward(router &rt, std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, pooled_connection>>;
```

异步路由正向代理请求。这是正向连接的核心路径，整合 DNS 解析与竞速连接。

**参数：**
- `rt`：路由器引用，提供 DNS 解析器和连接池
- `host`：目标主机名
- `port`：目标服务端口（字符串形式）

**返回值：**
- `fault::code::success` + 有效连接：连接成功
- `fault::code::host_unreachable` + 空连接：DNS 解析失败或 IPv6 被禁用
- `fault::code::bad_gateway` + 空连接：连接失败

**流程：**

```
async_forward(rt, host, port)
    |
    v
[IP 字面量?]
    |-- Yes --> [IPv6 + disabled?]
    |               |-- Yes --> return host_unreachable
    |               |-- No  --> pool.async_acquire(endpoint)
    |                            |-- success --> return success + conn
    |                            |-- fail    --> return bad_gateway
    |
    |-- No --> rt.dns().resolve_tcp(host, port)
                |-- fail/empty --> return host_unreachable
                |-- success --> async_connect_with_retry(rt, endpoints)
                                |-- valid --> return success + conn
                                |-- invalid --> return bad_gateway
```

**关键细节：**
- IP 字面量跳过 DNS 解析，直接连接，减少一跳延迟
- IPv6 字面量在 IPv6 被禁用时返回 `host_unreachable`，避免无效连接
- DNS 解析后的多端点列表由 [[core/connect/dial/racer]] 竞速处理

### async_datagram

```cpp
auto async_datagram(router &rt, std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, net::ip::udp::socket>>;
```

异步路由数据报请求。解析目标地址并创建 UDP 套接字。

**参数：**
- `rt`：路由器引用
- `host`：目标主机名
- `port`：目标服务端口

**返回值：**
- `fault::code::success` + UDP 套接字：成功
- 错误码 + 空 UDP 套接字：失败

**流程：**

1. IP 字面量直接构造端点；否则 DNS 解析
2. 调用 `open_udp_socket()` 创建 UDP 套接字

### resolve_dgram

```cpp
auto resolve_dgram(router &rt, std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>;
```

仅解析数据报目标端点，不创建套接字。用于 UDP ASSOCIATE 场景，延迟到发送时才打开 UDP 套接字。

### open_udp

```cpp
inline auto open_udp(const net::any_io_executor &executor, const net::ip::udp::endpoint &target)
    -> std::pair<fault::code, net::ip::udp::socket>;
```

根据目标端点的地址类型自动选择 IPv4/IPv6 协议打开 UDP 套接字。

**参数：**
- `executor`：用于创建套接字的执行器
- `target`：目标 UDP 端点，用于确定协议版本

**返回值：**
- `fault::code::success` + UDP 套接字：打开成功
- `fault::code::io_error` + 空套接字：打开失败

### make_router

```cpp
inline auto make_router(router &rt)
    -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(std::string_view, std::string_view)>;
```

创建 UDP 数据报路由回调函数，避免每个协议处理器重复构造相同的回调。用于 SOCKS5 UDP ASSOCIATE 等场景。

> [!warning]
> 返回的回调持有 `rt` 的非拥有引用（空删除器 `shared_ptr`），调用方必须确保 `rt` 的生命周期长于回调的使用期。

### dial（路由器版本）

```cpp
auto dial(router &rt, dial_options opts)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

拨号连接上游服务器并包装为可靠传输。这是所有协议处理器的统一入口。

**参数：**
- `rt`：路由器引用
- `opts`：路由策略选项（标签、目标地址、路由标志）

**返回值：**
- `fault::code::success` + `shared_transmission`：成功
- 错误码 + `nullptr`：失败

**路由决策逻辑：**

```
dial(rt, opts)
    |
    v
[IPv6 disabled + IPv6 literal?]
    |-- Yes --> return ipv6_disabled
    |
    v
[allow_reverse && !target.positive?]
    |-- Yes --> rt.async_reverse(target.host)    // 反向路由表查找
    |-- No  --> async_forward(rt, target.host, target.port)  // DNS 解析 + 竞速
    |
    v
[连接成功?]
    |-- fail --> return error + nullptr
    |-- success + require_open && !conn.valid()
    |            --> return connection_refused + nullptr
    |-- success --> transport::make_reliable(std::move(conn))
                    return success + shared_transmission
```

**关键行为：**
- `allow_reverse` 且 `target.positive = false` 时优先查反向路由表
- 反向路由表查找失败时不回退到正向路由，直接返回错误
- 成功的连接通过 `transport::make_reliable()` 包装为 `shared_transmission`

### dial（出站代理版本）

```cpp
auto dial(outbound::proxy &outbound_proxy, const protocol::target &target,
          const net::any_io_executor &executor)
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

通过出站代理拨号连接上游。用于正向代理场景。

**参数：**
- `outbound_proxy`：出站代理引用
- `target`：目标地址信息
- `executor`：用于创建连接的执行器

**返回值：** 与路由器版本相同

## dial_options 结构体

```cpp
struct dial_options
{
    std::string_view label;              // 协议标签，用于日志记录
    const protocol::target &target;      // 解析后的上游目标地址

    enum class flag : std::uint8_t
    {
        normal,       // 允许反向路由，要求套接字已打开
        no_reverse,   // 禁止反向路由
        no_open,      // 不要求套接字已打开
        neither       // 禁止反向路由 + 不要求套接字已打开
    };

    flag routing{flag::normal};          // 路由策略标志
};
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `label` | — | 协议标签（如 "Trojan"、"SOCKS5"），用于日志 |
| `target` | — | 解析后的上游目标地址 |
| `routing` | `normal` | 路由策略标志，控制反向路由和套接字打开要求 |

**使用场景：**
- `normal`：标准反向/正向代理连接
- `no_reverse`：强制正向连接，忽略反向路由表
- `no_open`：允许返回未打开的套接字（测试场景）
- `neither`：完全绕过路由和状态检查

## 设计决策

### 为什么 dial 函数使用 dial_options 结构体收敛参数？

`dial()` 原本有 4 个参数（router、label、target、flag），超过项目规范上限 3 个。将 label、target、flag 合并到 `dial_options`，函数签名降为 2 个参数。这也使得未来新增路由策略只需扩展 `flag` 枚举，不影响调用方。

**后果**: `dial_options` 持有 `target` 的引用，调用方必须保证 `target` 在 `co_await dial()` 完成前有效。

### 为什么反向路由失败不回退到正向路由？

反向路由表是配置层面的显式映射（dest → endpoint），如果配置了反向路由但查找失败，说明配置错误或目标不在映射中。静默回退到 DNS 解析会掩盖配置问题，让运维难以察觉路由错误。

**后果**: 反向路由表缺失的域名需要显式配置为正向路由（`positive = true`），不会被自动降级。

### 为什么 retry_connect 委托给 racer 而非自己重试？

手动重试（逐端点串行尝试）在 DNS 返回多个地址时延迟叠加。racer 实现 RFC 8305 Happy Eyeballs v2 算法，IPv6 优先 50ms 后并行发起 IPv4，显著降低多地址场景的首连延迟。

**后果**: 所有端点都失败时，延迟约等于单端点超时 + IPv4 竞速窗口，而非所有超时的总和。

## 约束

### dial_options 引用生命周期

**类型**: 生命周期

**规则**: `dial_options` 持有 `protocol::target` 的 `const` 引用，调用方必须保证 `target` 在 `co_await dial()` 完成前不被销毁。

**违反后果**: 悬挂引用，协程恢复后访问已释放内存，未定义行为。

**源码依据**: `dial.hpp:157`

### make_router 回调持有非拥有引用

**类型**: 生命周期

**规则**: `make_router()` 返回的 `std::function` 通过空删除器 `shared_ptr` 持有 `router&`，不会延长 router 生命周期。

**违反后果**: router 析构后调用回调 → 访问已释放内存 → 崩溃。

**源码依据**: `dial.hpp:141`

### 协程纯度

**类型**: 线程安全

**规则**: 所有函数返回 `net::awaitable`，必须在 io_context 线程上 `co_await`。禁止在协程中使用阻塞 socket 操作或 `std::mutex`。

**违反后果**: 阻塞 io_context 线程，所有同线程的连接停滞。

**源码依据**: CLAUDE.md 协程纯度规则

## 调用链

```
协议处理器 (SOCKS5/Trojan/VLESS/HTTP)
    |
    v
dial(rt, label, target, opts)
    |
    +-- [反向路由] --> rt.async_reverse(host)
    |                      |
    |                      v
    |                  [[core/connect/dial/router|router.async_reverse]]
    |                      |
    |                      v
    |                  pool.async_acquire(endpoint)
    |
    +-- [正向路由] --> async_forward(rt, host, port)
                           |
                           +-- [IP字面量] --> pool.async_acquire(endpoint)
                           |
                           +-- [域名] --> rt.dns().resolve_tcp(host, port)
                                              |
                                              v
                                          async_connect_with_retry(rt, endpoints)
                                              |
                                              v
                                          [[core/connect/dial/racer|address_racer.race]]
                                              |
                                              v
                                          pool.async_acquire(endpoint)
```

## 时序图

```
Client          Handler        dial()          router         DNS           racer        pool
  |               |              |               |             |              |            |
  |---CONNECT---->|              |               |             |              |            |
  |               |--dial()----->|               |             |              |            |
  |               |              |               |             |              |            |
  |               |              |--resolve_tcp()|------------>|              |            |
  |               |              |               |             |              |            |
  |               |              |<--endpoints---|<------------|              |            |
  |               |              |               |             |              |            |
  |               |              |--race()-------|-------------|------------->|            |
  |               |              |               |             |              |            |
  |               |              |               |             |              |--acquire()->|
  |               |              |               |             |              |            |
  |               |              |               |             |              |<--conn-----|
  |               |              |               |             |              |            |
  |               |              |<--conn--------|-------------|-------------|            |
  |               |              |               |             |              |            |
  |               |              |--make_reliable()                                                            |
  |               |<--trans------+               |             |              |            |
  |               |              |               |             |              |            |
```

## 与其他模块的关系

- [[core/connect/dial/router]]：路由器，提供反向路由表查找和连接池访问
- [[core/connect/dial/racer]]：Happy Eyeballs 竞速连接器
- [[core/connect/pool/pool]]：TCP 连接池，提供连接复用
- [[core/connect/tunnel/forward]]：组合 `dial()` + `tunnel()` 的正向代理转发
- [[core/connect/tunnel/tunnel]]：双向隧道转发
- [[core/resolve/dns/dns]]：DNS 解析器，提供域名解析
- [[core/transport/overview]]：传输层抽象，`make_reliable()` 将连接包装为传输对象
- [[core/context/context]]：会话上下文，提供路由器和缓冲区配置
- [[core/protocol/common/target]]：目标地址结构体，是路由决策的输入
- [[core/outbound/proxy]]：出站代理，支持正向代理间接连接

## 故障场景

### DNS 解析失败导致连接失败

**触发条件**: 目标域名无法解析（DNS 服务器无响应、域名不存在、网络中断）。

**传播路径**: `rt.dns().resolve_tcp()` 返回错误 → `async_forward()` 返回 `host_noreply` → `dial()` 返回错误 + nullptr → 协议处理器返回连接失败 → 客户端收到连接重置。

**外部表现**: 客户端连接超时。日志出现 `[Connect.Dial] DNS resolve xxx:yyy failed`。

**恢复机制**: 自动。下次请求重新触发 DNS 解析。如果是临时故障，新请求可能成功。

**日志关键字**: `DNS resolve` + `failed`

### 连接池耗尽

**触发条件**: 上游服务器响应慢，所有连接被占用且未归还。

**传播路径**: `pool.async_acquire()` 等待超时 → `async_forward()` 返回 `bad_gateway` → `dial()` 返回错误 → 客户端连接失败。

**外部表现**: 批量客户端连接超时。日志出现 `route failed: bad_gateway`。

**恢复机制**: 自动。活跃连接关闭后归还池中，新请求可复用。可调低连接池 `max_pool_size` 加速回收。

**日志关键字**: `bad_gateway` + `route failed`

### IPv6 被禁用但目标为 IPv6

**触发条件**: 配置禁用 IPv6，但目标地址是 IPv6 字面量或 DNS 解析仅返回 IPv6 地址。

**传播路径**: `is_ipv6()` 返回 true → 直接返回 `ipv6_disabled` 或 `host_noreply`，不发起任何连接。

**外部表现**: 特定目标始终连接失败。日志出现 `rejecting IPv6 literal`。

**恢复机制**: 修改配置启用 IPv6 或更换目标地址。

**日志关键字**: `IPv6 disabled` 或 `IPv6 literal`

## 注意事项

1. **IPv6 过滤**：IPv6 禁用时（`rt.ipv6_disabled()`），IPv6 字面量直接返回 `host_unreachable`，不会触发连接
2. **反向路由优先**：路由策略允许反向路由且非正向代理请求时，优先查反向路由表；反向路由查找失败不回退
3. **日志标签**：使用 `[Connect.Dial]` 前缀，便于日志过滤和诊断
