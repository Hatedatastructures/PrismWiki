---
title: "Pipeline 协议处理器详解"
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [pipeline, processors, dispatch, http, socks5, trojan, vless, shadowsocks]
related: ["[[pipeline]]", "[[protocol]]", "[[recognition]]", "[[agent]]", "[[channel]]", "[[multiplex]]"]
---

# Pipeline 协议处理器详解

> 源码：`include/prism/pipeline/protocols.hpp` · `include/prism/pipeline/protocols/*.hpp` · `src/prism/pipeline/protocols/*.cpp`
> 相关：[[pipeline]] | [[protocol]] | [[recognition]] | [[agent]] | [[multiplex]]

## 1. 概述

Pipeline 协议处理器是 Prism 中各协议的会话级处理函数，负责从协议握手到隧道转发的完整流程。每个处理器统一签名为：

```cpp
auto handler(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
```

- `ctx` — 会话上下文，包含入站传输、配置、路由器、帧内存池等
- `data` — 协议探测阶段的预读数据，通过 [[pipeline]] 的 `preview` 装饰器回放

## 2. Dispatch 分发表

协议处理器通过编译期常量数组注册到 `agent::dispatch::handler_table`（`include/prism/agent/dispatch/table.hpp`），数组索引对应 `protocol::protocol_type` 枚举值。零虚函数、零动态分配：

```cpp
inline constexpr std::array<handler_func *, ...> handler_table{
    /* unknown     */ handle_unknown,       // 原始 TCP 透传
    /* http        */ pipeline::http,
    /* socks5      */ pipeline::socks5,
    /* trojan      */ pipeline::trojan,
    /* vless       */ pipeline::vless,
    /* shadowsocks */ pipeline::shadowsocks,
    /* tls         */ handle_unknown,       // TLS 由 stage chain 处理，不直接到达
};
```

分发逻辑：`dispatch()` 将 `protocol_type` 转为数组索引，越界时 fallback 到 `handle_unknown`。

## 3. 共享原语操作

所有处理器共享 [[pipeline]] 模块的 `primitives` 命名空间提供的通用操作：

| 原语 | 功能 | 调用者 |
|------|------|--------|
| `wrap_with_preview()` | 将预读数据包装为 `preview` 传输装饰器 | 所有处理器 |
| `dial()` | 通过路由器或出站代理拨号上游 | HTTP、SOCKS5 |
| `forward()` | 组合 `dial + tunnel` 的便捷函数 | Trojan、VLESS、SS2022 |
| `tunnel()` | 双向全双工隧道转发 | 所有处理器 |
| `ssl_handshake()` | TLS 服务端握手 | [[stealth]] Native |
| `is_mux_target()` | 检测 mux 多路复用标记地址 | Trojan、VLESS |
| `make_datagram_router()` | 创建 UDP 数据报路由回调 | SOCKS5、Trojan、VLESS |

## 4. 各处理器详解

### 4.1 HTTP（`pipeline::http`）

处理 HTTP 代理协议（CONNECT 和普通请求）。

**流程：**
1. `wrap_with_preview()` 回放预读数据
2. 创建 `protocol::http::relay` 并执行握手（读取请求头 + 解析 + 代理认证）
3. `protocol::analysis::resolve()` 解析目标地址
4. `dial()` 连接上游（支持出站代理和路由器两种路径）
5. 分发：
   - **CONNECT** — 发送 `200 Connection Established`，进入 `tunnel()` 双向转发
   - **普通请求** — 重写绝对 URI 为相对路径后转发原始数据，再进入隧道

**与 [[protocol]] 的关系：** 调用 `protocol::http::make_relay()` 创建中继对象，协议编解码由 `protocol::http::relay` 封装。

### 4.2 SOCKS5（`pipeline::socks5`）

处理 SOCKS5 代理协议，支持 CONNECT 和 UDP_ASSOCIATE 命令。

**流程：**
1. `wrap_with_preview()` 回放预读数据
2. 创建 `protocol::socks5::relay` 并执行握手（方法协商 + 请求解析）
3. 分发命令：
   - **CONNECT** — 解析目标地址，`dial()` 拨号上游，发送 SOCKS5 成功响应，`tunnel()` 转发
   - **UDP_ASSOCIATE** — 创建 `datagram_router`，`async_associate()` 启动 UDP 中继
   - **BIND** — 不支持，返回 `command_not_supported` 错误

**错误处理：** 拨号失败时按 RFC 1928 语义返回 SOCKS5 错误码（`network_unreachable` / `host_unreachable`）。

### 4.3 Trojan（`pipeline::trojan`）

处理 Trojan 代理协议。TLS 已在 [[stealth]] 层剥离，此处理器处理明文 Trojan 流量。

**流程：**
1. `wrap_with_preview(ctx, data, true)` — mux 模式使用全局内存池避免 UAF
2. 创建凭证验证回调（通过 `account::try_acquire` 验证 + 连接数限制）
3. 创建 `protocol::trojan::relay` 并执行握手（凭据 + 目标地址解析）
4. 分发命令：
   - **CONNECT** — 检测 mux 标记地址，若匹配则 `multiplex::bootstrap()` 启动多路复用；否则 `forward()` 隧道转发
   - **UDP_ASSOCIATE** — `async_associate()` 启动 UDP 中继
   - **MUX (0x7F)** — 直接进入多路复用模式

**与 [[multiplex]] 的关系：** mux 模式下 transport 生命周期由 multiplexer 接管，清除 session 的 close/cancel 回调。

### 4.4 VLESS（`pipeline::vless`）

处理 VLESS 代理协议，结构与 Trojan 高度相似。

**流程：**
1. `wrap_with_preview(ctx, data, true)` — 全局内存池
2. UUID 验证回调（同样通过 `account::try_acquire`）
3. 创建 `protocol::vless::relay` 并执行握手
4. 分发命令：
   - **TCP / MUX** — mux 标记检测 → 多路复用或 `forward()` 隧道
   - **UDP** — `async_associate()` UDP 中继

**与 Trojan 的区别：** VLESS 使用 UUID 验证而非 SHA-224 密码哈希；协议报文格式不同但处理流程几乎一致。

### 4.5 Shadowsocks 2022（`pipeline::shadowsocks`）

处理 SS2022 协议，采用无正特征设计（排除法检测）。

**流程：**
1. `wrap_with_preview(ctx, data, true)` — 全局内存池
2. 获取 worker 线程独占的 `salt_pool`（`thread_local`，无需锁）
3. 创建 `protocol::shadowsocks::relay` 并执行握手（AEAD 解密 + 时间戳验证 + 地址解析）
4. **乐观响应** — 先发送 `acknowledge()` 再拨号（与 mihomo 客户端行为一致）
5. relay 本身作为 `transmission` 传入 `forward()`，AEAD 加解密持续进行

**特殊设计：** SS2022 的 relay 同时实现 `transmission` 接口，隧道转发期间每次读写都经过 AEAD 加解密，无需额外包装层。

## 5. 处理器与 protocol 模块的关系

Pipeline 处理器（`pipeline::protocols/*`）是**编排层**，负责会话级流程控制。协议编解码（`protocol::*`）是**实现层**，负责具体的报文解析和序列化：

```
pipeline::trojan()        ──编排──▶  protocol::trojan::relay::handshake()
                                    protocol::trojan::relay::async_associate()

pipeline::http()          ──编排──▶  protocol::http::relay::handshake()
                                    protocol::http::relay::forward()

pipeline::shadowsocks()   ──编排──▶  protocol::shadowsocks::relay::handshake()
                                    protocol::shadowsocks::relay::acknowledge()
```

每个协议的 `make_relay()` 工厂函数创建对应的 relay 对象，pipeline 处理器通过 relay 的公共方法驱动协议流程。这种分离使得协议编解码可以独立测试，pipeline 专注于流程编排。
