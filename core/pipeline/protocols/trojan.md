---
title: "trojan.hpp — Trojan 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/trojan.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# trojan.hpp

> 源码: `include/prism/pipeline/protocols/trojan.hpp`
> 实现: `src/prism/pipeline/protocols/trojan.cpp`
> 模块: Pipeline / protocols

## 概述

Trojan 协议处理管道，处理 TLS 剥离后的内层流量。TLS 握手在 Session 层完成，本模块负责 Trojan 协议握手、凭据验证、命令分发和多路复用引导。

## 命名空间

`psm::pipeline`

---

## 函数: trojan()

> 源码位置: `trojan.hpp:31`
> 实现位置: `trojan.cpp`

### 功能

Trojan 协议处理函数，处理 TLS 剥离后的内层协议数据。

### 签名

```cpp
auto trojan(session_context &ctx, std::span<const std::byte> data)
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
| 创建凭证验证器    |
| account::directory |
| try_acquire()    |
+------------------+
         │
         ▼
+------------------+
| 创建 Trojan relay |
| trojan::make_relay() |
+------------------+
         │
         ▼
+------------------+
| 执行握手          |
| relay::handshake() |
| 解析凭据 + 目标   |
+------------------+
         │
         ▼
+--+--+--+--+--+
   |  |  |  |
CONNECT UDP mux
   |  |  |
   ▼  ▼  ▼
检测mux 创建 mux检测
标记    路由 bootstrap()
   |    回调    |
   ▼    |      ▼
拨号   async   多路复用
或     associate()
mux    |
   ▼
+------------------+
| forward() 或     |
| async_associate()|
+------------------+
```

### 调用链（向下）

- [[core/pipeline/primitives#wrap_with_preview|wrap_with_preview]] — 包装入站传输（use_global_mr=true）
- `agent::account::directory::try_acquire` — 凭证验证
- `protocol::trojan::make_relay` — 创建 Trojan 中继器
- `protocol::trojan::relay::handshake` — 执行握手
- [[core/pipeline/primitives#forward|forward]] — TCP 转发
- [[core/pipeline/primitives#make_datagram_router|make_datagram_router]] — UDP 路由回调
- `protocol::trojan::relay::async_associate` — UDP 中继
- [[core/pipeline/primitives#is_mux_target|is_mux_target]] — mux 标记检测
- `multiplex::bootstrap` — 多路复用引导

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表注册为 Trojan 处理器

---

## Trojan 协议特点

### 协议格式

Trojan 协议请求格式：
```
+----------------------+----------------------+----------+-----+
|      SHA224(password)|      地址           | CRLF     | CMD |
+----------------------+----------------------+----------+-----+
|        56 bytes      |    Variable         |   2      |  1  |
+----------------------+----------------------+----------+-----+
```

### 命令类型

| 命令 | 值 | 说明 |
|------|-----|------|
| CONNECT | 0x01 | TCP 隧道 |
| UDP_ASSOCIATE | 0x03 | UDP 中继 |
| mux 标记 | 0x7F | 直接 mux 模式 |

### 凭证验证

Trojan 使用 SHA224 哈希密码：
```cpp
// 密码哈希计算
auto hash = sha224(password);
// 请求中的凭据与哈希比对
if (request.hash != hash) {
    return fault::code::auth_failed;
}
```

### mux 多路复用

Trojan 支持 smux/yamux 多路复用，检测方式：
1. 命令字节为 0x7F（直接 mux 模式）
2. 目标地址以 `.mux.sing-box.arpa` 结尾

```cpp
if (cmd == 0x7F || primitives::is_mux_target(host, ctx.mux_enabled)) {
    co_await multiplex::bootstrap(ctx, inbound);
    co_return;
}
```

---

## 实现细节

### 凭证验证流程

```cpp
// 创建凭证验证器
auto validator = ctx.account_directory_ptr->try_acquire(hash);

if (!validator) {
    trace::warn("{} auth failed: invalid credential", TrojanStr);
    co_return;
}
```

### CONNECT 命令处理

```cpp
// 检测 mux 标记
if (primitives::is_mux_target(target.host, ctx.mux_enabled)) {
    co_await multiplex::bootstrap(ctx, inbound);
    co_return;
}

// TCP 转发
co_await primitives::forward(ctx, TrojanStr, target, inbound);
```

### UDP_ASSOCIATE 命令处理

```cpp
// 创建路由回调
auto router_cb = primitives::make_datagram_router(ctx.worker.router);

// 启动 UDP 中继
co_await relay->async_associate(router_cb, ctx);
```

### 注意事项

- **TLS 层分离**: TLS 握手在 Session 层完成，本函数处理内层协议
- **全局内存池**: mux 场景使用 `use_global_mr=true`
- **账户验证**: 通过 account::directory 统一管理

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/pipeline/primitives|管道原语]]
- [[protocol/trojan|Trojan 协议格式]]
- [[multiplex/bootstrap|多路复用引导]]
- [[agent/account/directory|账户目录]]