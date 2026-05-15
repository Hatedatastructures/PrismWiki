---
title: "trojan.hpp — Trojan 协议处理管道"
source: "include/prism/pipeline/protocols/trojan.hpp"
module: "pipeline"
type: api
tags: [pipeline, protocols, trojan, Trojan]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/primitives|primitives]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[agent/account/directory|account directory]]"
  - "[[multiplex/bootstrap|mux bootstrap]]"
  - "[[ref/protocol/tls-1.3|TLS 1.3]]"
---

# trojan.hpp

> 源码: `include/prism/pipeline/protocols/trojan.hpp`
> 实现: `src/prism/pipeline/protocols/trojan.cpp`
> 模块: [[pipeline|Pipeline]] / protocols

## 概述

声明 Trojan 代理协议的会话处理函数。TLS 握手在 Session 层完成，本模块负责 Trojan 协议握手、凭据验证、命令分发和多路复用引导。支持 CONNECT（TCP 隧道）、UDP_ASSOCIATE（UDP over TLS）和 mux 多路复用（cmd=0x7F 或 CONNECT + 虚假 mux 地址）。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[pipeline/primitives|primitives]] | 隧道原语（forward, wrap_with_preview, is_mux_target, make_datagram_router） |
| 依赖 | [[agent/account/directory|directory]] | 账户目录，统一认证和连接数限制 |
| 依赖 | [[multiplex/bootstrap|bootstrap]] | 多路复用引导 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[agent/dispatch/table|table]] | handler_table 注册 trojan |

## 命名空间

`psm::pipeline`

---

## 函数: trojan()

> 源码: `include/prism/pipeline/protocols/trojan.hpp:31`
> 实现: `src/prism/pipeline/protocols/trojan.cpp:15`

### 功能

Trojan 协议处理函数，处理 TLS 剥离后的 Trojan 协议流量，包括握手解析凭据和目标地址、账户验证、命令分发（CONNECT/UDP_ASSOCIATE/mux）。

### 签名

```cpp
auto trojan(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，包含入站传输和配置信息 |
| data | `std::span<const std::byte>` | 预读数据，Session 层 TLS 剥离后的内层协议数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象，处理完成后返回。

### 调用（向下）

- [[pipeline/primitives|wrap_with_preview]] — 包装入站传输（use_global_mr=true，mux 模式下避免 UAF）
- [[agent/account/directory|try_acquire]] — 验证凭据并获取账户租约
- `protocol::trojan::make_relay` — 创建 Trojan 中继器
- `protocol::trojan::relay::handshake` — Trojan 握手解析凭据和目标地址
- `protocol::trojan::to_string` — 将目标地址转换为字符串
- [[pipeline/primitives|is_mux_target]] — 检测 mux 标记地址
- [[multiplex/bootstrap|bootstrap]] — 创建多路复用会话
- [[pipeline/primitives|forward]] — CONNECT 命令拨号 + 隧道转发
- [[pipeline/primitives|make_datagram_router]] — UDP_ASSOCIATE 创建路由回调
- `protocol::trojan::relay::async_associate` — UDP_ASSOCIATE 启动 UDP 中继

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表中注册为 Trojan 处理器

### 涉及的知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 握手在 Session 层完成
- [[ref/protocol/smux-yamux|Smux/Yamux]] — 多路复用协议
- [[ref/network/udp|UDP]] — UDP over TLS 中继

### 流程

1. 包装入站传输（use_global_mr=true，mux 模式下 transport 生命周期超出 session）
2. 创建凭证验证器（通过 account::directory 统一认证和连接数限制）
3. 创建 Trojan 中继器并执行握手
4. 根据命令类型分发：
   - **CONNECT**：解析目标地址 → 检测 mux 标记 → 拨号 + 隧道转发 或 进入多路复用
   - **UDP_ASSOCIATE**：创建 datagram_router → 启动 UDP 中继
   - **mux (0x7F)**：直接进入多路复用模式
   - **未知命令**：记录日志

### 错误处理

- 握手失败时记录日志并静默关闭连接
- UDP_ASSOCIATE 失败时记录日志
- 凭证验证失败时记录日志并静默关闭连接

### 注意事项

- TLS 握手在 Session 层完成，本函数处理的是 TLS 剥离后的内层协议数据
- mux 模式下 inbound 会被移交给 smux_craft 并脱离 session 生命周期，因此使用全局内存池
- Mihomo smux 兼容：客户端用 CONNECT + 虚假地址（`.mux.sing-box.arpa`）标记 mux 连接
- Trojan mux (cmd=0x7F) 直接进入多路复用模式，无需解析目标地址
