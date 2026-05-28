---
title: Connect 模块
created: 2026-05-27
updated: 2026-05-27
layer: core
module: connect
source:
  - I:/code/Prism/include/prism/connect.hpp
tags: [connect, overview, outbound]
---

# Connect 模块

Connect 模块管理出站连接的生命周期，从路由选择到连接建立再到双向数据转发。

## 架构

```
协议处理器 (SOCKS5/Trojan/VLESS/HTTP)
    |
    v
forward(ctx, opts)                            ← 入口：拨号+隧道
    |
    +-- dial(router, opts)                     ← 拨号：路由+DNS+竞速
    |       |
    |       +-- router.async_reverse(host)     ← 反向路由表查找
    |       +-- router.dns().resolve_tcp()     ← DNS 解析
    |       +-- address_racer.race(endpoints)  ← Happy Eyeballs
    |       +-- pool.async_acquire(endpoint)   ← 连接池复用/新建
    |
    +-- tunnel(inbound, outbound, ctx)         ← 双向数据转发
            |
            +-- operator|| (upload || download) ← 全双工并发
```

## 子模块

| 子模块 | 功能 | 源码 |
|--------|------|------|
| [[core/connect/dial/router\|Router]] | 反向路由表 + DNS + 连接池协调 | `connect/dial/router.hpp` |
| [[core/connect/dial/racer\|Racer]] | Happy Eyeballs 竞速连接 (RFC 8305) | `connect/dial/racer.hpp` |
| [[core/connect/dial/dial\|Dial]] | 拨号器，整合路由+竞速+重试 | `connect/dial/dial.hpp` |
| [[core/connect/pool/pool\|Pool]] | 连接池，LIFO 复用空闲连接 | `connect/pool/pool.hpp` |
| [[core/connect/pool/health\|Health]] | 连接健康检查 (SO_ERROR + FIN 检测) | `connect/pool/health.hpp` |
| [[core/connect/pool/config\|Pool Config]] | 连接池配置参数 | `connect/pool/config.hpp` |
| [[core/connect/tunnel/tunnel\|Tunnel]] | 双向数据转发 (operator||) | `connect/tunnel/tunnel.hpp` |
| [[core/connect/tunnel/forward\|Forward]] | 协议级转发 (dial + tunnel) | `connect/tunnel/forward.hpp` |

## 设计决策

### 为什么 dial/tunnel/forward 都是自由函数？

这些函数不持有状态，只依赖参数传入的上下文（router、session）。自由函数避免不必要的类层次和对象生命周期管理，也天然适合协程——不需要 `shared_from_this()`。

### 为什么连接池使用 LIFO 而非 FIFO？

LIFO 保证最近归还的连接最先被复用，最近使用的连接最可能仍处于健康状态。长时间空闲的连接自然沉底，在 cleanup 时优先被淘汰。

## 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| forward | dial | `forward()` 检查 `ctx.outbound_proxy` 决定使用哪个 `dial()` 重载 |
| dial | router | `dial()` 根据 `target.positive` 决定反向路由或正向 DNS 路由 |
| racer | pool | `racer` 通过 `pool.async_acquire()` 获取连接，不直接创建 socket |
| pool | health | `pool.recycle()` 调用 `healthy_fast()` 检测连接健康，不健康连接直接关闭 |
| tunnel | context::session | `tunnel()` 通过 `ctx.worker_ctx.traffic` 刷写流量统计，`ctx.account_lease` 累加账户用量 |
| tunnel | transport | `tunnel()` 使用 `shut_close()` 优雅关闭传输层 |
| router | resolve::dns | router 拥有 DNS 解析器实例，提供 `dns()` 访问器 |

## 相关文档

- [[core/transport/overview|Transport]] — 传输层
- [[core/pipeline/overview|Pipeline]] — 协议处理管道
- [[core/resolve/overview|Resolve]] — DNS 解析
- [[core/context/context|Context]] — 会话上下文
