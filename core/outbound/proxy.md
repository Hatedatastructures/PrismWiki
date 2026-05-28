---
title: "proxy — 出站代理抽象接口"
layer: core
source: "I:/code/Prism/include/prism/outbound/proxy.hpp"
module: outbound
tags: [outbound, proxy, interface, adapter]
updated: 2026-05-27
---

# proxy — 出站代理抽象接口

> 源码位置: `include/prism/outbound/proxy.hpp`

## 概述

`proxy` 是所有出站协议和代理组必须实现的抽象接口。纯协程设计，通过组合模式使代理组和单个代理共享同一接口，调用方无需区分。

## 核心接口

| 方法 | 说明 |
|------|------|
| `async_connect(target, executor)` | 纯虚，建立 TCP 连接，返回 `(fault::code, shared_transmission)` |
| `make_router()` | 纯虚，创建 UDP 数据报路由回调 |
| `name()` | 纯虚，代理名称（日志/调试） |
| `supports_udp()` | 虚，默认 `true`，不支持 UDP 的代理重写 |

## 类型别名

| 类型 | 定义 | 说明 |
|------|------|------|
| `shared_transmission` | `transport::shared_transmission` | 传输层智能指针 |
| `router_fn` | `std::function<awaitable<pair<fault::code, udp_endpoint>>(string_view, string_view)>` | UDP 路由回调 |

## 设计决策

### 为什么接口只有 TCP connect 和 UDP datagram 两个方法？

出站代理的职责是"建立到目标的连接"。无论底层协议如何，上层只需要一个可用的传输对象。两个方法覆盖所有代理场景，接口最小化。

**后果**: 新增传输类型（如 WebSocket）需要在 `shared_transmission` 层而非 `proxy` 层扩展。

### 为什么用虚基类而非 type-erased 的 function？

代理组的子代理数量在配置加载时确定，运行时不变。虚函数调用的间接跳转开销（~1ns）相比网络 I/O 延迟（~ms 级）可忽略。虚基类比 `std::function` 更清晰表达"多态接口"语义。

**后果**: 所有代理类型必须继承 `proxy` 并实现虚函数。

### 为什么 make_router() 返回 function 而非传递 router 引用？

调用方（UDP 协议处理器）不需要知道 DNS 解析细节。`router_fn` 将 DNS 解析封装在实现内部，代理组可以委托给子代理的路由器。

**后果**: `std::function` 有堆分配开销，但 UDP 路由回调在会话生命周期内只创建一次。

## 实现策略

| 实现类 | async_connect 行为 |
|--------|-------------------|
| `direct` | DNS 解析 → racer 竞速 → 直连 |
| 协议代理 (socks5/trojan/vless/ss) | 上游代理 async_connect → 协议握手 |
| 代理组 (url_test/fallback/load_balance) | 选择子代理 → 委托 |

## 引用关系

### 依赖

- [[core/transport/transmission|transmission]]：传输层抽象
- [[core/fault/code|fault::code]]：错误码
- [[core/protocol/common/target|protocol::target]]：目标地址结构

### 被引用

- [[core/outbound/direct|direct]]：直连实现
- [[core/connect/dial/dial|dial]]：通过 direct 调用路由
