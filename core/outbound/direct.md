---
title: "direct — 直连出站代理"
layer: core
source: "I:/code/Prism/include/prism/outbound/direct.hpp"
module: outbound
tags: [outbound, direct, adapter, router]
updated: 2026-05-27
---

# direct — 直连出站代理

> 源码位置: `include/prism/outbound/direct.hpp`

## 概述

`direct` 是最简单的 [[core/outbound/proxy|proxy]] 实现，将连接请求直接通过 DNS 解析和 Happy Eyeballs 建立连接，不经过任何上游代理。Header-only，每个 worker 持有一个实例。

## 核心接口

| 方法 | 说明 |
|------|------|
| `direct(router)` | 构造，绑定 `connect::router` 引用 |
| `async_connect(target, executor)` | 根据 `target.positive` 选择正向/反向路由 |
| `make_router()` | 返回 UDP 路由回调，非拥有 `shared_ptr` + 空删除器 |
| `name()` | 返回 `"DIRECT"` |

## 设计决策

### 为什么 make_router() 用非拥有 shared_ptr？

`make_router()` 返回的 lambda 可能在 router 析构后仍被调用（UDP 协议的异步特性）。用空删除器的 `shared_ptr` 捕获 router 引用，避免悬挂引用的同时不转移所有权（router 由 worker 管理）。

**后果**: 如果 router 先于 lambda 析构，空删除器的 `shared_ptr` 不会阻止释放，回调访问悬挂指针。实际由 worker 生命周期保证安全。

### 为什么 IPv6 检查在路由之前？

`router_.ipv6_disabled()` 时，提前拒绝 IPv6 字面量避免不必要的 DNS 解析和连接尝试。对域名不做检查（DNS 可能返回 IPv4 结果），只拦截确定是 IPv6 的情况。

**后果**: 仅拦截字面量（`::1`, `2001:db8::1`），域名中的 IPv6 地址由 racer 竞速阶段自然过滤。

## 约束

### router 引用必须比 direct 实例长寿

**类型**: 生命周期

**规则**: `direct` 持有 `connect::router` 的裸引用，router 必须在 direct 使用期间存活。

**违反后果**: 悬挂引用，未定义行为。

**源码依据**: `direct.hpp:133` `connect::router &router_`

## 路由策略

| target.positive | 调用 | 行为 |
|-----------------|------|------|
| `true` | `connect::async_forward(router, host, port)` | DNS 解析 → racer 竞速 → 连接池 |
| `false` | `router_.async_reverse(host)` | 反向路由表查找 → 预配置后端 |

## 引用关系

### 依赖

- [[core/outbound/proxy|proxy]]：出站代理接口
- [[core/connect/dial/router|connect::router]]：路由与连接池
- [[core/transport/reliable|reliable]]：TCP 传输封装

### 被引用

- [[core/instance/worker/worker|worker]]：每个 worker 持有 direct 实例
