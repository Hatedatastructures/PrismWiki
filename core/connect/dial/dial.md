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

### async_connect_with_retry

```cpp
auto async_connect_with_retry(router &rt, std::span<const tcp::endpoint> endpoints)
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

### resolve_datagram_target

```cpp
auto resolve_datagram_target(router &rt, std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>;
```

仅解析数据报目标端点，不创建套接字。用于 UDP ASSOCIATE 场景，延迟到发送时才打开 UDP 套接字。

### open_udp_socket

```cpp
inline auto open_udp_socket(const net::any_io_executor &executor, const net::ip::udp::endpoint &target)
    -> std::pair<fault::code, net::ip::udp::socket>;
```

根据目标端点的地址类型自动选择 IPv4/IPv6 协议打开 UDP 套接字。

**参数：**
- `executor`：用于创建套接字的执行器
- `target`：目标 UDP 端点，用于确定协议版本

**返回值：**
- `fault::code::success` + UDP 套接字：打开成功
- `fault::code::io_error` + 空套接字：打开失败

### make_datagram_router

```cpp
inline auto make_datagram_router(router &rt)
    -> std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(std::string_view, std::string_view)>;
```

创建 UDP 数据报路由回调函数，避免每个协议处理器重复构造相同的回调。用于 SOCKS5 UDP ASSOCIATE 等场景。

> [!warning]
> 返回的回调持有 `rt` 的非拥有引用（空删除器 `shared_ptr`），调用方必须确保 `rt` 的生命周期长于回调的使用期。

### dial（路由器版本）

```cpp
auto dial(router &rt, std::string_view label, const protocol::target &target,
          dial_options opts = {})
    -> net::awaitable<std::pair<fault::code, shared_transmission>>;
```

拨号连接上游服务器并包装为可靠传输。这是所有协议处理器的统一入口。

**参数：**
- `rt`：路由器引用
- `label`：协议标签（如 "Trojan"、"SOCKS5"），用于日志
- `target`：解析后的上游目标地址（`protocol::target`）
- `opts`：路由策略选项

**返回值：**
- `fault::code::success` + `shared_transmission`：成功
- 错误码 + `nullptr`：失败

**路由决策逻辑：**

```
dial(rt, label, target, opts)
    |
    v
[IPv6 disabled + IPv6 literal?]
    |-- Yes --> return ipv6_disabled
    |
    v
[opts.allow_reverse && !target.positive?]
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
- `allow_reverse = true` 且 `target.positive = false` 时优先查反向路由表
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
    bool allow_reverse{true};  // 是否允许使用反向路由
    bool require_open{true};   // 是否要求返回的套接字已打开
};
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `allow_reverse` | `true` | 允许通过反向路由表连接，适用于反向代理场景 |
| `require_open` | `true` | 要求返回的套接字已打开，关闭后返回 `connection_refused` |

**使用场景：**
- 默认选项（全部 `true`）：标准反向/正向代理连接
- `allow_reverse = false`：强制正向连接，忽略反向路由表
- `require_open = false`：允许返回未打开的套接字（用于测试或特殊场景）

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
- [[core/transport]]：传输层抽象，`make_reliable()` 将连接包装为传输对象
- [[core/context/context]]：会话上下文，提供路由器和缓冲区配置
- [[core/protocol/common/target]]：目标地址结构体，是路由决策的输入
- [[core/outbound/proxy]]：出站代理，支持正向代理间接连接

## 注意事项

1. **协程纯度**：所有函数返回 `net::awaitable`，无阻塞操作
2. **IPv6 过滤**：IPv6 禁用时（`rt.ipv6_disabled()`），IPv6 字面量直接返回 `host_unreachable`，不会触发连接
3. **反向路由优先**：`allow_reverse = true` 且非正向代理请求时，优先查反向路由表；反向路由查找失败不回退
4. **生命周期**：`make_datagram_router()` 返回的回调持有路由器的非拥有引用，调用方需确保路由器生命周期
5. **日志标签**：使用 `[Connect.Dial]` 前缀，便于日志过滤和诊断
