---
title: "vless.hpp — VLESS 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/vless.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# vless.hpp

> 源码: `include/prism/pipeline/protocols/vless.hpp`
> 实现: `src/prism/pipeline/protocols/vless.cpp`
> 模块: Pipeline / protocols

## 概述

VLESS 协议处理管道，处理 TLS 剥离后的内层流量。TLS 握手在 Session 层完成，本模块负责 VLESS 协议握手、UUID 验证、命令分发和多路复用引导。

## 命名空间

`psm::pipeline`

---

## 函数: vless()

> 源码位置: `vless.hpp:26`
> 实现位置: `vless.cpp`

### 功能

VLESS 协议处理函数，处理 TLS 剥离后的内层协议数据。

### 签名

```cpp
auto vless(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文 |
| data | `std::span<const std::byte>` | Session 层 TLS 剥离后的内层协议数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象

### 处理流程

```
+------------------+
| 包装入站传输      |
| wrap_with_preview() |
| use_global_mr=true |
+------------------+
         │
         ▼
+------------------+
| 创建 UUID 验证器  |
| account::directory |
| try_acquire()    |
+------------------+
         │
         ▼
+------------------+
| 创建 VLESS relay  |
| vless::make_relay() |
+------------------+
         │
         ▼
+------------------+
| 执行握手          |
| relay::handshake() |
| 解析 UUID + 目标  |
+------------------+
         │
         ▼
    +----+----+
    │         │
tcp/mux    udp
    │         │
    ▼         ▼
检测mux    创建
标记       路由回调
    │       async_associate()
    ▼
拨号
或 mux
    │
    ▼
+------------------+
| forward() 或     |
| bootstrap()      |
+------------------+
```

### 调用链（向下）

- [[core/pipeline/primitives#wrap_with_preview|wrap_with_preview]] — 包装入站传输（use_global_mr=true）
- `agent::account::directory::try_acquire` — UUID 验证
- `protocol::vless::make_relay` — 创建 VLESS 中继器
- `protocol::vless::relay::handshake` — 执行握手
- [[core/pipeline/primitives#forward|forward]] — TCP 转发
- [[core/pipeline/primitives#make_datagram_router|make_datagram_router]] — UDP 路由回调
- `protocol::vless::relay::async_associate` — UDP 中继
- [[core/pipeline/primitives#is_mux_target|is_mux_target]] — mux 标记检测
- `multiplex::bootstrap` — 多路复用引导

### 被调用（向上）

- [[core/agent/dispatch/table|dispatch]] — 协议分发表注册为 VLESS 处理器

---

## VLESS 协议特点

### 协议格式

VLESS 请求格式：
```
+--------+--------+--------+--------+--------+--------+--------+
|  VER   |  UUID  | ADDONS |  CMD   |  ADDR  |  PORT  |  DATA  |
+--------+--------+--------+--------+--------+--------+--------+
|   1    |   16   | Variable|   1    | Variable|   2    |Variable|
+--------+--------+--------+--------+--------+--------+--------+
```

### 命令类型

| 命令 | 值 | 说明 |
|------|-----|------|
| tcp | 0x01 | TCP 隧道 |
| udp | 0x02 | UDP 中继 |
| mux | 检测标记 | 多路复用 |

### UUID 验证

VLESS 使用 UUID 作为身份标识：
```cpp
// UUID 格式验证
if (!is_valid_uuid(request.uuid)) {
    return fault::code::auth_failed;
}

// 账户目录验证
auto validator = ctx.account_directory_ptr->try_acquire(uuid);
```

### mux 多路复用

VLESS 支持 smux/yamux 多路复用，检测方式：
- 目标地址以 `.mux.sing-box.arpa` 结尾

```cpp
if (primitives::is_mux_target(target.host, ctx.mux_enabled)) {
    co_await multiplex::bootstrap(ctx, inbound);
    co_return;
}
```

### ADDONS 扩展

VLESS 支持 ADDONS 扆展字段，用于传递额外信息（如 XTLS 流控）。

---

## 实现细节

### UUID 验证流程

```cpp
// 创建 UUID 验证器
auto validator = ctx.account_directory_ptr->try_acquire(uuid);

if (!validator) {
    trace::warn("{} auth failed: invalid uuid", VlessStr);
    co_return;
}
```

### TCP 命令处理

```cpp
// 检测 mux 标记
if (primitives::is_mux_target(target.host, ctx.mux_enabled)) {
    co_await multiplex::bootstrap(ctx, inbound);
    co_return;
}

// TCP 转发
co_await primitives::forward(ctx, VlessStr, target, inbound);
```

### UDP 命令处理

```cpp
// 创建路由回调
auto router_cb = primitives::make_datagram_router(ctx.worker.router);

// 启动 UDP 中继
co_await relay->async_associate(router_cb, ctx);
```

### 注意事项

- **TLS 层分离**: TLS 握手在 Session 层完成
- **全局内存池**: mux 场景使用 `use_global_mr=true`
- **UUID 格式**: 标准 UUID 格式（16 字节）

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/pipeline/primitives|管道原语]]
- [[core/protocol/vless|VLESS 协议格式]]
- [[core/multiplex/bootstrap|多路复用引导]]
- [[core/agent/account/directory|账户目录]]