---
title: "udp_relay — 共享 UDP 中继辅助工具"
source: "include/prism/protocol/common/udp_relay.hpp"
module: "protocol"
type: api
tags: [protocol, common, udp_relay, UDP, 中继, 数据报]
related:
  - "[[protocol/trojan/relay|trojan_relay]]"
  - "[[protocol/vless/relay|vless_relay]]"
  - "[[protocol/socks5/stream|socks5_relay]]"
created: 2026-05-15
updated: 2026-05-15
---

# udp_relay.hpp

> 源码: `include/prism/protocol/common/udp_relay.hpp`
> 模块: [[protocol|protocol]] > common

## 概述

共享 UDP 中继辅助工具。提供协议无关的 UDP 数据报中继基础设施，包括缓冲区管理和数据报转发函数。被 Trojan 和 VLESS 的 UDP over TLS 实现共用。所有操作基于 `boost::asio::awaitable`，支持异步无阻塞处理。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[memory|memory]] | PMR 容器 |
| 依赖 | [[fault|fault]] | 错误码 |
| 依赖 | [[trace|trace]] | 日志 |
| 被依赖 | [[protocol/trojan/relay|trojan_relay]] | Trojan UDP 中继 |
| 被依赖 | [[protocol/vless/relay|vless_relay]] | VLESS UDP 中继 |

## 命名空间

`psm::protocol::common`

---

## 结构体: udp_buffers

### 功能说明

UDP 会话缓冲区集合。封装 UDP 帧循环所需的所有缓冲区，使用 PMR 分配器避免热路径堆分配。

### 构造函数

```cpp
explicit udp_buffers(const std::size_t max_datagram);
```

#### 功能说明

构造 UDP 缓冲区集合。根据 `max_datagram` 大小预分配 `recv`、`send`、`response` 三个缓冲区，避免 UDP 帧循环中的动态扩容。使用 PMR 分配器与线程局部内存池对齐。

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `max_datagram` | `std::size_t` | 数据报最大长度（字节） |

#### 返回值

无（构造函数）

#### 调用（向下）

- `memory::vector<std::byte>` 构造函数（PMR 分配器）

#### 被调用（向上）

- [[protocol/trojan/relay|trojan_relay]] `async_associate()` 初始化缓冲区
- [[protocol/vless/relay|vless_relay]] `async_associate()` 初始化缓冲区

#### 知识域

- [[memory|memory]] PMR 内存分配
- [[protocol/common/udp_relay|udp_relay]] UDP 数据报中继

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `recv` | `memory::vector<std::byte>` | 接收缓冲区 |
| `send` | `memory::vector<std::byte>` | 发送缓冲区 |
| `response` | `memory::vector<std::byte>` | 响应缓冲区 |

---

## 函数: relay_udp_packet()

### 功能说明

通过延迟打开的 UDP socket 转发数据报到目标地址，然后等待并接收单个响应。Socket 首次调用时按目标协议族打开，后续调用复用同一 socket，避免每包 open/close 的系统调用开销。

### 签名

```cpp
inline auto relay_udp_packet(net::ip::udp::socket &udp_socket,
                             const net::ip::udp::endpoint &target_ep,
                             std::span<const std::byte> payload,
                             udp_buffers &buf)
    -> net::awaitable<std::tuple<fault::code, std::size_t, net::ip::udp::endpoint>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `udp_socket` | `net::ip::udp::socket &` | UDP socket 引用，延迟打开并复用 |
| `target_ep` | `const net::ip::udp::endpoint &` | 目标端点 |
| `payload` | `std::span<const std::byte>` | 载荷数据 |
| `buf` | `udp_buffers &` | 缓冲区集合 |

### 返回值

`net::awaitable<std::tuple<fault::code, std::size_t, net::ip::udp::endpoint>>` — 错误码、响应数据长度、发送者端点。

### 调用（向下）

- `udp_socket.open()` — 延迟打开 socket
- `udp_socket.async_send_to()` — 发送数据报
- `udp_socket.async_receive_from()` — 接收响应

### 被调用（向上）

- [[protocol/trojan/relay|trojan_relay]] `udp_frame_loop()` 转发 UDP 数据
- [[protocol/vless/relay|vless_relay]] `udp_frame_loop()` 转发 UDP 数据

### 知识域

UDP socket 管理、延迟打开、协程异步 I/O

## 相关页面

- [[protocol/trojan/relay|trojan_relay]] — Trojan 中继器
- [[protocol/vless/relay|vless_relay]] — VLESS 中继器
- [[protocol/socks5/stream|socks5_relay]] — SOCKS5 中继器
