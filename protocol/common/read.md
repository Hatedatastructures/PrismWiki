---
title: "read — 共享 I/O 读取工具函数"
source: "include/prism/protocol/common/read.hpp"
module: "protocol"
type: api
tags: [protocol, common, read, 协程, 批量读取, 异步IO]
related:
  - "[[protocol/trojan/relay|trojan_relay]]"
  - "[[protocol/vless/relay|vless_relay]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# read.hpp

> 源码: `include/prism/protocol/common/read.hpp`
> 模块: [[protocol|protocol]] > common

## 概述

共享 I/O 读取工具函数。提供跨协议的通用读取辅助函数，包括 `read_at_least` 和 `read_remaining`。这些函数封装了从传输层批量读取数据的协程逻辑，被 Trojan 和 VLESS relay 共同使用，消除重复代码。所有函数返回 `net::awaitable`，遵循项目纯协程设计。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/transport/transmission|transmission]] | 传输层读取接口 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[protocol/trojan/relay|trojan_relay]] | Trojan 中继器使用读取工具 |
| 被依赖 | [[protocol/vless/relay|vless_relay]] | VLESS 中继器使用读取工具 |

## 命名空间

`psm::protocol::common`

---

## 函数: read_at_least()

### 功能说明

批量读取至少指定数量的字节。循环调用 `async_read_some` 直到读取至少 `min_size` 字节。遇到错误或 EOF 时提前返回。

### 签名

```cpp
inline auto read_at_least(channel::transport::transmission &transport,
                          const std::span<std::byte> buffer,
                          const std::size_t min_size)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `channel::transport::transmission &` | 传输层引用 |
| `buffer` | `std::span<std::byte>` | 输出缓冲区 |
| `min_size` | `std::size_t` | 最小读取字节数 |

### 返回值

`net::awaitable<std::pair<fault::code, std::size_t>>` — 错误码和实际读取字节数。

### 调用（向下）

- `transport.async_read_some()` — 传输层异步读取

### 被调用（向上）

- [[protocol/trojan/relay|trojan_relay]] `handshake()` 读取凭据
- [[protocol/vless/relay|vless_relay]] `handshake()` 读取请求头

### 知识域

协程循环读取、EOF 处理

---

## 函数: read_remaining()

### 功能说明

精确补读剩余字节。从 `current` 位置继续读取，直到达到 `target` 字节。用于在部分读取后补全数据。

### 签名

```cpp
inline auto read_remaining(channel::transport::transmission &transport,
                           const std::span<std::byte> buffer,
                           std::size_t current,
                           const std::size_t target)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `channel::transport::transmission &` | 传输层引用 |
| `buffer` | `std::span<std::byte>` | 输出缓冲区 |
| `current` | `std::size_t` | 当前已读字节数 |
| `target` | `std::size_t` | 目标字节数 |

### 返回值

`net::awaitable<std::pair<fault::code, std::size_t>>` — 错误码和最终读取字节数。

### 调用（向下）

- `transport.async_read_some()` — 传输层异步读取

### 被调用（向上）

- [[protocol/trojan/relay|trojan_relay]] 补读地址数据
- [[protocol/vless/relay|vless_relay]] 补读请求数据

### 知识域

协程循环读取、缓冲区管理

## 相关页面

- [[protocol/trojan/relay|trojan_relay]] — Trojan 中继器
- [[protocol/vless/relay|vless_relay]] — VLESS 中继器
- [[channel/transport/transmission|transmission]] — 传输层接口
