---
title: "vless.hpp — VLESS 协议处理管道"
source: "include/prism/pipeline/protocols/vless.hpp"
module: "pipeline"
type: api
tags: [pipeline, protocols, vless, VLESS]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/primitives|primitives]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[agent/account/directory|account directory]]"
  - "[[multiplex/bootstrap|mux bootstrap]]"
  - "[[ref/protocol/tls-1.3|TLS 1.3]]"
---

# vless.hpp

> 源码: `include/prism/pipeline/protocols/vless.hpp`
> 实现: `src/prism/pipeline/protocols/vless.cpp`
> 模块: [[pipeline|Pipeline]] / protocols

## 概述

声明 VLESS 代理协议的会话处理函数。TLS 握手在 Session 层完成，本模块负责 VLESS 协议握手、UUID 验证、命令分发和多路复用引导。支持 TCP（含 mux 标记检测）、UDP 和 mux 命令。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[pipeline/primitives|primitives]] | 隧道原语（forward, wrap_with_preview, is_mux_target, make_datagram_router） |
| 依赖 | [[agent/account/directory|directory]] | 账户目录，统一认证和连接数限制 |
| 依赖 | [[multiplex/bootstrap|bootstrap]] | 多路复用引导 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[agent/dispatch/table|table]] | handler_table 注册 vless |

## 命名空间

`psm::pipeline`

---

## 函数: vless()

> 源码: `include/prism/pipeline/protocols/vless.hpp:26`
> 实现: `src/prism/pipeline/protocols/vless.cpp:16`

### 功能

VLESS 协议处理函数，处理 TLS 剥离后的 VLESS 协议流量，包括握手解析、UUID 验证、命令分发（TCP/UDP/mux）。

### 签名

```cpp
auto vless(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
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
- [[agent/account/directory|try_acquire]] — 验证 UUID 并获取账户租约
- `protocol::vless::make_relay` — 创建 VLESS 中继器
- `protocol::vless::relay::handshake` — VLESS 握手解析 UUID 和目标地址
- `protocol::vless::to_string` — 将目标地址转换为字符串
- [[pipeline/primitives|is_mux_target]] — 检测 mux 标记地址
- [[multiplex/bootstrap|bootstrap]] — 创建多路复用会话
- [[pipeline/primitives|forward]] — TCP 命令拨号 + 隧道转发
- [[pipeline/primitives|make_datagram_router]] — UDP 命令创建路由回调
- `protocol::vless::relay::async_associate` — UDP 命令启动 UDP 中继

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表中注册为 VLESS 处理器

### 涉及的知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 握手在 Session 层完成
- [[ref/protocol/smux-yamux|Smux/Yamux]] — 多路复用协议
- [[ref/network/udp|UDP]] — UDP 中继

### 流程

1. 包装入站传输（use_global_mr=true，mux 模式下 transport 生命周期超出 session）
2. 创建 UUID 验证回调（通过 account::directory 统一认证和连接数限制）
3. 创建 VLESS 中继器并执行握手
4. 根据命令类型分发：
   - **tcp / mux**：解析目标地址 → 检测 mux 标记 → 拨号 + 隧道转发 或 进入多路复用
   - **udp**：创建 datagram_router → 启动 UDP 中继
   - **未知命令**：记录日志

### 错误处理

- 握手失败时记录日志并静默关闭连接
- UDP 命令失败时记录日志
- UUID 验证失败时记录日志并静默关闭连接

### 注意事项

- TLS 握手在 Session 层完成，本函数处理的是 TLS 剥离后的内层协议数据
- mux 模式下 inbound 会被移交给 smux_craft 并脱离 session 生命周期，因此使用全局内存池
- Mihomo smux 兼容：客户端用 mux 命令或虚假地址（`.mux.sing-box.arpa`）标记多路复用连接
- VLESS 的 tcp 和 mux 命令共享相同的处理逻辑
