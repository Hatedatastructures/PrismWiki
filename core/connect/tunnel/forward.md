---
title: forward
layer: core
source: include/prism/connect/tunnel/forward.hpp
module: connect
tags:
  - forward
  - proxy
  - dial-and-tunnel
  - tcp-relay
  - session
created: 2026-05-23
updated: 2026-05-23
---

# forward

正向代理转发，组合 `dial()` + `tunnel()` 操作，提供完整的 TCP 隧道转发流程。所有协议的 TCP 隧道转发共用此函数。

## 概述

`forward()` 是协议处理器到数据转发之间的桥梁。它将两个核心操作串联起来：

1. **拨号（dial）**：通过路由器建立到上游服务器的连接
2. **隧道（tunnel）**：在客户端和上游之间建立双向透明数据转发

所有 TCP 协议处理器（SOCKS5、Trojan、VLESS、HTTP CONNECT）在完成协议协商后，都调用 `forward()` 进入数据转发阶段。

核心设计原则：

- **统一入口**：所有协议共用同一个转发函数，消除重复代码
- **出站代理感知**：自动检测是否配置了出站代理，选择直连或代理路径
- **错误统一处理**：拨号失败时根据错误类型输出不同级别的日志

## 函数签名

```cpp
auto forward(context::session &ctx, forward_options opts) -> net::awaitable<void>;
```

### forward_options

```cpp
struct forward_options
{
    std::string_view label;               // 协议标签，用于日志
    const protocol::target &target;       // 目标地址信息
    shared_transmission inbound;          // 入站传输对象
};
```

| 字段 | 说明 |
|------|------|
| `label` | 协议标签（如 "Trojan"、"SOCKS5"），用于日志 |
| `target` | 目标地址信息，路由决策的输入 |
| `inbound` | 入站传输对象（客户端连接） |

## 执行流程

```
forward(ctx, label, target, inbound)
    |
    v
[ctx.outbound_proxy 存在?]
    |
    +-- Yes --> [出站代理路径]
    |               |
    |               v
    |           dial(*ctx.outbound_proxy, target, executor)
    |               |
    |               +-- 成功 --> tunnel(inbound, outbound, ctx)
    |               +-- 失败 --> 日志 + return
    |
    +-- No --> [直连路径]
                    |
                    v
                dial(ctx.worker_ctx.router, label, target)
                    |
                    +-- 成功 --> tunnel(inbound, outbound, ctx)
                    +-- 失败 --> 日志 + return
```

### 出站代理路径

当会话配置了出站代理时（`ctx.outbound_proxy != nullptr`），使用 `outbound::proxy` 版本的 `dial()` 进行连接。出站代理可以是 SOCKS5、HTTP 等协议的上游代理。

**关键行为：**
- 使用 `ctx.worker_ctx.io_context.get_executor()` 作为执行器
- 出站代理路径不使用 [[core/connect/dial/router|router]] 的路由逻辑
- 出站代理由 `context::worker` 层面的 `outbound` 指针配置

### 直连路径

无出站代理时，使用 `router` 版本的 `dial()` 进行连接。路由器根据 `target` 的配置选择反向路由或正向 DNS 解析路由。

## 设计决策

### 为什么是自由函数而非成员方法？

`forward()` 不持有任何状态，只需参数传入的上下文。自由函数避免不必要的类层次和对象生命周期管理，也天然适合协程（不需要 `shared_from_this()`）。

### 为什么 ctx 是非 const 引用？

`tunnel()` 需要写统计信息（`flush_traffic()` 和 `accumulate_uplink/downlink()`），因此 `ctx` 为非 const 引用。

### 为什么出站代理路径先 co_return？

使用提前返回模式（early return），避免 `if-else` 嵌套。两个路径的 `tunnel()` 调用完全独立，不存在共享状态。

## 约束

### forward_options 引用生命周期

**类型**: 生命周期

**规则**: `forward_options` 持有 `target` 的 `const` 引用，`target` 必须在 `co_await forward()` 完成前有效。

**违反后果**: 悬挂引用，协程恢复后访问已释放内存。

**源码依据**: `forward.hpp:30`

### 出站代理生命周期

**类型**: 生命周期

**规则**: `ctx.outbound_proxy` 指针由 `context::worker` 持有，`forward()` 不拥有。worker 必须在会话期间保持 outbound_proxy 有效。

**违反后果**: 通过悬挂指针访问已销毁的出站代理，崩溃。

**源码依据**: `forward.hpp:41`

### inbound 所有权转移

**类型**: 资源管理

**规则**: `inbound` 通过值传递（`shared_transmission`），`tunnel()` 内部 `std::move()` 获取所有权。调用 `forward()` 后不应再使用 `inbound`。

**违反后果**: 使用已 move 的 `shared_transmission`，`get()` 返回 nullptr。

## 错误处理

### 错误码分类

```cpp
if (ec == fault::code::ipv6_disabled)
    trace::debug(...);  // IPv6 禁用：debug 级别
else
    trace::warn(...);   // 其他错误：warn 级别
```

| 错误类型 | 日志级别 | 说明 |
|----------|----------|------|
| `ipv6_disabled` | `debug` | IPv6 被禁用，目标是 IPv6 地址。属于正常拒绝，不是错误 |
| 其他错误 | `warn` | 连接失败、DNS 解析失败、超时等。需要关注 |

### 日志格式

```
[Connect.Forward] IPv6 disabled: <host>:<port>
[Connect.Forward] dial failed: <error_description>, target: <host>:<port>
```

日志包含：
- 模块标签：`[Connect.Forward]`
- 错误描述：`fault::describe(ec)` 将错误码转为可读字符串
- 目标地址：`host:port` 格式

## 时序图

### 直连路径

```
Handler         forward()          dial()          router          tunnel()
  |                 |                 |               |               |
  |--forward()----->|                 |               |               |
  |                 |                 |               |               |
  |                 |--dial(router)-->|               |               |
  |                 |                 |--route------->|               |
  |                 |                 |               |               |
  |                 |                 |<--conn--------|               |
  |                 |<--trans---------|               |               |
  |                 |                 |               |               |
  |                 |--tunnel()-------------------------------------->|
  |                 |                                 |               |
  |                 |                      [双向数据转发]              |
  |                 |                                 |               |
  |                 |<-----------------------------------------------|
  |                 |                                 |               |
  |<--done---------|                                 |               |
```

### 出站代理路径

```
Handler         forward()          dial(proxy)     outbound_proxy   tunnel()
  |                 |                 |               |               |
  |--forward()----->|                 |               |               |
  |                 |                 |               |               |
  |                 |--dial(proxy)--> |               |               |
  |                 |                 |--connect()--->|               |
  |                 |                 |               |               |
  |                 |                 |<--trans-------|               |
  |                 |<--trans---------|               |               |
  |                 |                 |               |               |
  |                 |--tunnel()-------------------------------------->|
  |                 |                                 |               |
  |                 |                      [双向数据转发]              |
  |                 |                                 |               |
  |                 |<-----------------------------------------------|
  |                 |                                 |               |
  |<--done---------|                                 |               |
```

## 调用方

以下协议处理器在完成协议协商后调用 `forward()`：

| 协议 | Handler 位置 | label 参数 |
|------|-------------|------------|
| SOCKS5 TCP | `protocol/socks5/conn.hpp` | `"SOCKS5"` |
| Trojan TCP | `protocol/trojan/conn.hpp` | `"Trojan"` |
| VLESS TCP | `protocol/vless/conn.hpp` | `"VLESS"` |
| HTTP CONNECT | `protocol/http/conn.hpp` | `"HTTP"` |
| Shadowsocks TCP | `protocol/shadowsocks/conn.hpp` | `"SS2022"` |

## 调用链详解

```
协议处理器 (SOCKS5/Trojan/VLESS/HTTP/SS2022)
    |
    v
forward(ctx, label, target, inbound)
    |
    +-- [outbound_proxy 路径]
    |       |
    |       v
    |   dial(outbound::proxy, target, executor)   ---> [[core/outbound/proxy]]
    |       |
    |       v
    |   [[core/connect/tunnel/tunnel|tunnel(inbound, outbound, ctx)]]
    |
    +-- [直连路径]
            |
            v
        dial(router, label, target)                ---> [[core/connect/dial/dial]]
            |
            v
        [[core/connect/dial/router|router]] --路由选择--> 连接
            |
            v
        [[core/connect/tunnel/tunnel|tunnel(inbound, outbound, ctx)]]
```

## 与其他模块的关系

- [[core/connect/dial/dial]]：拨号函数，`forward()` 是 `dial()` 的主要调用方
- [[core/connect/tunnel/tunnel]]：隧道函数，`forward()` 调用 `tunnel()` 进行数据转发
- [[core/connect/dial/router]]：路由器，`dial()` 内部使用路由器进行路由选择
- [[core/connect/util]]：`tunnel()` 内部使用 `shut_close()` 关闭连接
- [[core/connect/pool/pool]]：连接池，`dial()` 通过路由器获取连接
- [[core/context/context]]：会话上下文，提供路由器、出站代理和统计信息
- [[core/protocol/common/target]]：目标地址结构体，路由决策的输入
- [[core/outbound/proxy]]：出站代理，正向代理场景使用

## 设计要点

### 为什么是自由函数而非成员方法？

`forward()` 设计为自由函数而非类成员方法，原因：

1. **无状态**：`forward()` 不持有任何状态，只需要参数传入的上下文
2. **灵活性**：可以被任何协议处理器调用，不需要继承关系
3. **简洁性**：避免了不必要的类层次和对象生命周期管理
4. **协程友好**：自由函数天然适合协程，不需要 `shared_from_this()`

### 为什么 ctx 是非 const 引用？

`tunnel()` 需要写统计信息（流量统计、账户用量累加），因此 `ctx` 为非 const 引用：

```cpp
auto forward(context::session &ctx, ...);
//                                  ^ 非 const
```

`tunnel()` 内部操作：
- `ctx.worker_ctx.traffic->flush_traffic(...)`：刷写流量统计
- `account::accumulate_uplink/downlink(ctx.account_lease.get(), ...)`：累加账户用量

### 为什么出站代理路径先 co_return？

```cpp
if (ctx.outbound_proxy)
{
    // ... 出站代理路径 ...
    co_await tunnel(...);
    co_return;  // 提前返回
}

// 直连路径
```

使用提前返回模式（early return），避免 `if-else` 嵌套，提高代码可读性。两个路径的 `tunnel()` 调用完全独立，不存在共享状态。

## 注意事项

1. **所有权转移**：`inbound` 通过值传递，`tunnel()` 内部 `std::move()` 获取所有权
2. **日志标签**：使用 `[Connect.Forward]` 前缀
3. **错误静默**：拨号失败时只输出日志，不抛异常
4. **会话结束**：`forward()` 返回后，会话生命周期由上层管理
