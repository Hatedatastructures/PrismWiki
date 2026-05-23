---
title: "socks5.hpp — SOCKS5 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/socks5.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# socks5.hpp

> 源码: `include/prism/pipeline/protocols/socks5.hpp`
> 实现: `src/prism/pipeline/protocols/socks5.cpp`
> 模块: Pipeline / protocols

## 概述

SOCKS5 协议处理管道，处理 SOCKS5 代理请求。支持 CONNECT（TCP 隧道）和 UDP_ASSOCIATE（UDP 中继）命令。

## 命名空间

`psm::pipeline`

---

## 函数: socks5()

> 源码位置: `socks5.hpp:30`
> 实现位置: `socks5.cpp`

### 功能

SOCKS5 代理协议处理函数，执行握手协商、请求解析、命令分发和隧道转发。

### 签名

```cpp
auto socks5(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文 |
| data | `std::span<const std::byte>` | 预读数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象

### 处理流程

```
+------------------+
| 包装入站传输      |
| wrap_with_preview() |
+------------------+
         │
         ▼
+------------------+
| 创建 SOCKS5 relay |
| socks5::make_relay() |
+------------------+
         │
         ▼
+------------------+
| 执行握手          |
| relay::handshake() |
| 认证方法协商      |
| 请求解析          |
+------------------+
         │
         ▼
    +----+----+
    │         │
CONNECT  UDP_ASSOCIATE
    │         │
    ▼         ▼
解析目标   创建路由回调
拨号上游   make_datagram_router()
发送成功   async_associate()
响应       │
    │      UDP 中继
    ▼
+------------------+
| 建立双向隧道      |
| tunnel()         |
+------------------+
```

### 调用链（向下）

- [[core/pipeline/primitives#wrap_with_preview|wrap_with_preview]] — 包装入站传输
- `protocol::socks5::make_relay` — 创建 SOCKS5 中继器
- `protocol::socks5::relay::handshake` — 执行握手
- [[core/pipeline/primitives#dial|dial]] — 拨号上游
- `protocol::socks5::relay::write_success` — 发送成功响应
- `protocol::socks5::relay::write_error` — 发送错误响应
- [[core/pipeline/primitives#tunnel|tunnel]] — 建立隧道
- [[core/pipeline/primitives#make_datagram_router|make_datagram_router]] — UDP 路由回调
- `protocol::socks5::relay::async_associate` — UDP 中继

### 被调用（向上）

- [[core/instance/dispatch/table|dispatch]] — 协议分发表注册为 SOCKS5 处理器

---

## SOCKS5 协议特点

### 命令类型

| 命令 | 值 | 说明 |
|------|-----|------|
| CONNECT | 0x01 | TCP 隧道 |
| BIND | 0x02 | 不支持 |
| UDP_ASSOCIATE | 0x03 | UDP 中继 |

### 认证方法

| 方法 | 值 | 说明 |
|------|-----|------|
| No Auth | 0x00 | 无认证 |
| GSSAPI | 0x01 | 不支持 |
| Username/Password | 0x02 | 用户名密码认证 |
| No Acceptable | 0xFF | 无可用方法 |

### 地址类型

| 类型 | 值 | 说明 |
|------|-----|------|
| IPv4 | 0x01 | 4 字节 IPv4 地址 |
| Domain | 0x03 | 域名 + 长度 |
| IPv6 | 0x04 | 16 字节 IPv6 地址 |

### UDP_ASSOCIATE 流程

```
客户端 -> UDP_ASSOCIATE 请求 -> 代理
客户端 <- 成功响应 (UDP 端口) <- 代理

客户端 -> UDP 数据报 -> 代理 (UDP 端口) -> 目标
客户端 <- UDP 数据报 <- 代理 <- 目标
```

UDP 数据报格式：
```
+----+------+------+----------+----------+----------+
|RSV | FRAG | ATYP | DST.ADDR | DST.PORT |   DATA   |
+----+------+------+----------+----------+----------+
| 2  |  1   |  1   | Variable |    2     | Variable |
+----+------+------+----------+----------+----------+
```

---

## 实现细节

### CONNECT 命令处理

```cpp
// 解析目标地址
auto target = analysis::resolve(request);

// 拨号上游
auto [ec, outbound] = co_await primitives::dial(...);

// 发送响应
if (fault::failed(ec)) {
    co_await relay->write_error(socks5::reply::host_unreachable);
    co_return;
}

co_await relay->write_success();

// 建立隧道
co_await primitives::tunnel(relay->release(), outbound, ctx);
```

### UDP_ASSOCIATE 命令处理

```cpp
// 创建路由回调
auto router_cb = primitives::make_datagram_router(ctx.worker.router);

// 启动 UDP 中继
co_await relay->async_associate(router_cb, ctx);
```

### 注意事项

- SOCKS5 要求预读数据为空（握手从初始协商开始）
- IPv6 禁用时返回 `network_unreachable`
- UDP 中继需要额外的 UDP socket

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/pipeline/primitives|管道原语]]
- [[core/protocol/socks5/stream|SOCKS5 协议流]]
- [[core/protocol/analysis|目标地址解析]]