---
title: "table.hpp — 协议处理函数表"
source: "include/prism/agent/dispatch/table.hpp"
module: "agent"
type: api
tags: [agent, dispatch, 分发, 协议, 编译期, 零虚函数]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/session/session|会话管理]]"
  - "[[pipeline/protocols|协议处理管道]]"
  - "[[protocol/analysis|协议分析]]"
  - "[[ref/programming/constexpr|编译期优化]]"
---

# table.hpp

> 源码: `include/prism/agent/dispatch/table.hpp`
> 模块: [[agent|Agent]] / dispatch

## 概述

使用编译期数组替代虚函数 + 工厂模式，实现协议分发。每个 handler 函数直接调用对应的 pipeline 函数，无额外包装层。零虚函数、零动态分配、编译时可内联。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context\|context]] | session_context 定义 |
| 依赖 | [[protocol/analysis\|analysis]] | protocol_type 枚举 |
| 依赖 | [[pipeline/protocols\|protocols]] | 各协议处理函数（http, socks5, trojan, vless, shadowsocks） |
| 被依赖 | [[agent/session/session\|session]] | 会话分流时调用 dispatch |

## 命名空间

`psm::agent::dispatch`

---

## 类型别名: handler_func

**功能说明**: 协议处理函数类型定义，统一所有 handler 的签名格式。

**签名**:
```cpp
using handler_func = net::awaitable<void>(session_context &, std::span<const std::byte>);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `ctx` | `session_context &` | 会话上下文 |
| `data` | `std::span<const std::byte>` | 预读数据 |

**返回值**: `net::awaitable<void>` 协程

**调用（向下）**: 无（类型别名）

**被调用（向上）**: `handler_table` 数组中存储指向此类型的函数指针

**涉及的知识域**: [[ref/programming/constexpr\|编译期优化]]

---

## 函数: handle_unknown()

**功能说明**: Unknown 协议处理器，当协议无法识别时直接在入站和出站传输之间建立双向 TCP 隧道。

**签名**:
```cpp
inline auto handle_unknown(session_context &ctx, std::span<const std::byte> /*data*/)
    -> net::awaitable<void>;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `ctx` | `session_context &` | 会话上下文 |
| `data` | `std::span<const std::byte>` | 预读数据（未使用） |

**返回值**: `net::awaitable<void>` 协程

**调用（向下）**:
- 检查 `ctx.inbound` 和 `ctx.outbound` 有效性
- [[pipeline/primitives\|pipeline::primitives::tunnel()]] 建立双向隧道

**被调用（向上）**: `handler_table[0]`（unknown）和 `handler_table[6]`（tls fallback）；`dispatch()` 越界 fallback 时调用

**涉及的知识域**: [[ref/network/tcp\|TCP 传输]]

---

## 常量: handler_table

**功能说明**: 编译期协议处理函数表，数组索引对应 `protocol_type` 枚举值，实现 O(1) 分发。

**签名**:
```cpp
inline constexpr std::array<handler_func *, static_cast<std::size_t>(::psm::protocol::protocol_type::tls) + 1>
    handler_table;
```

**参数**: 无

**返回值**: 无（常量表达式）

**映射关系**:

| 索引 | 协议类型 | 处理函数 | 说明 |
|------|----------|----------|------|
| 0 | unknown | `handle_unknown` | 原始 TCP 透传 |
| 1 | http | `pipeline::http` | HTTP 代理协议 |
| 2 | socks5 | `pipeline::socks5` | SOCKS5 协议 |
| 3 | trojan | `pipeline::trojan` | Trojan 协议 |
| 4 | vless | `pipeline::vless` | VLESS 协议 |
| 5 | shadowsocks | `pipeline::shadowsocks` | SS2022 协议 |
| 6 | tls | `handle_unknown` | TLS 不应直接到达此表，由 stage chain 处理 |

**调用（向下）**: 无（常量数组）

**被调用（向上）**: `dispatch()` 通过索引访问

**涉及的知识域**: [[ref/programming/constexpr\|编译期优化]]

---

## 函数: dispatch()

**功能说明**: 根据协议类型枚举值索引函数表，调用对应的协议处理函数。索引越界时 fallback 到 unknown 处理器。

**签名**:
```cpp
inline auto dispatch(session_context &ctx, psm::protocol::protocol_type type, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `ctx` | `session_context &` | 会话上下文，传递给目标 handler |
| `type` | `psm::protocol::protocol_type` | 检测到的协议类型枚举值 |
| `data` | `std::span<const std::byte>` | 预读数据，传递给目标 handler |

**返回值**: `net::awaitable<void>` 协程

**调用（向下）**:
- `static_cast<std::size_t>(type)` 将枚举转为数组索引
- `handler_table[idx](ctx, data)` 调用对应的协议处理函数
- 越界时 fallback 到 `handler_table[unknown]`

**被调用（向上）**: [[agent/session/session\|session::diversion()]] 在完成协议识别后调用分发

**涉及的知识域**:
- [[ref/programming/constexpr\|编译期优化]]
- [[ref/programming/boost-asio\|Boost.Asio 协程]]

---

## 知识域

- [[ref/programming/constexpr\|编译期优化]]
- [[ref/programming/boost-asio\|Boost.Asio 协程]]
- [[ref/network/tcp\|TCP 传输]]
