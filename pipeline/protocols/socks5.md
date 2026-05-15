---
title: "socks5.hpp — SOCKS5 协议处理管道"
source: "include/prism/pipeline/protocols/socks5.hpp"
module: "pipeline"
type: api
tags: [pipeline, protocols, socks5, SOCKS5]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/primitives|primitives]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[protocol/socks5/relay|SOCKS5 relay]]"
  - "[[protocol/analysis|analysis]]"
  - "[[ref/protocol/socks5-rfc1928|SOCKS5 (RFC 1928)]]"
---

# socks5.hpp

> 源码: `include/prism/pipeline/protocols/socks5.hpp`
> 实现: `src/prism/pipeline/protocols/socks5.cpp`
> 模块: [[pipeline|Pipeline]] / protocols

## 概述

声明 SOCKS5 代理协议的会话处理函数，包括握手协商、请求解析、命令分发和双向隧道转发。支持 CONNECT（TCP 隧道）和 UDP_ASSOCIATE（UDP 中继）命令，不支持 BIND 命令。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[pipeline/primitives|primitives]] | 隧道原语（dial, tunnel, wrap_with_preview, make_datagram_router） |
| 依赖 | [[protocol/socks5/relay|relay]] | SOCKS5 中继器（握手、请求解析、响应） |
| 依赖 | [[protocol/analysis|analysis]] | 目标地址解析 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[agent/dispatch/table|table]] | handler_table 注册 socks5 |

## 命名空间

`psm::pipeline`

---

## 函数: socks5()

> 源码: `include/prism/pipeline/protocols/socks5.hpp:30`
> 实现: `src/prism/pipeline/protocols/socks5.cpp:12`

### 功能

SOCKS5 协议处理函数，执行握手协商、请求解析，并根据命令类型（CONNECT/UDP_ASSOCIATE）分发处理。

### 签名

```cpp
auto socks5(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，包含入站传输和配置信息 |
| data | `std::span<const std::byte>` | 预读数据，协议检测时读取的初始数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象，处理完成后返回。

### 调用（向下）

- [[pipeline/primitives|wrap_with_preview]] — 包装入站传输
- [[protocol/socks5/relay|make_relay]] — 创建 SOCKS5 中继器
- [[protocol/socks5/relay|relay::handshake]] — SOCKS5 握手协商（认证方法 + 请求解析）
- [[protocol/socks5/relay|relay::to_string]] — 将目标地址转换为字符串
- [[pipeline/primitives|dial]]（出站代理重载） — 通过出站代理拨号（CONNECT 命令）
- [[pipeline/primitives|dial]]（路由器重载） — 通过路由器拨号（CONNECT 命令）
- [[protocol/socks5/relay|relay::async_write_success]] — 拨号成功后发送 SOCKS5 成功响应
- [[protocol/socks5/relay|relay::async_write_error]] — 拨号失败时发送 SOCKS5 错误响应
- [[pipeline/primitives|tunnel]] — CONNECT 命令建立双向隧道
- [[pipeline/primitives|make_datagram_router]] — UDP_ASSOCIATE 创建路由回调
- [[protocol/socks5/relay|relay::async_associate]] — UDP_ASSOCIATE 启动 UDP 中继

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表中注册为 SOCKS5 处理器

### 涉及的知识域

- [[ref/protocol/socks5-rfc1928|SOCKS5 (RFC 1928)]] — SOCKS5 协议规范
- [[ref/network/udp|UDP]] — UDP_ASSOCIATE 命令
- [[ref/network/tcp|TCP]] — CONNECT 命令的 TCP 隧道

### 流程

1. 包装入站传输（如有预读数据则用 preview 装饰器重放）
2. 创建 SOCKS5 中继器并执行握手（认证方法协商 + 请求解析）
3. 根据命令类型分发：
   - **CONNECT**：解析目标地址 → 拨号上游 → 发送成功响应 → 建立双向隧道
   - **UDP_ASSOCIATE**：解析目标地址 → 创建 datagram_router → 启动 UDP 中继
   - **BIND**：不支持，返回 `command_not_supported` 错误

### 错误处理

- 握手失败时静默关闭连接
- CONNECT 拨号失败时发送 SOCKS5 错误响应：
  - IPv6 禁用时返回 `network_unreachable`
  - 其他失败返回 `host_unreachable`
- UDP_ASSOCIATE 失败时记录日志

### 注意事项

- SOCKS5 协议要求预读数据为空，否则握手可能失败
- CONNECT 命令先拨号上游再发送成功响应，遵循 RFC 1928 语义
