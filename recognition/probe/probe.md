---
title: "probe.hpp — 外层协议探测"
source: "include/prism/recognition/probe/probe.hpp"
module: "recognition"
type: api
tags: [recognition, probe, 探测, 协议探测]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/recognition
  - recognition/probe/analyzer
  - channel/transport/transmission
  - fault/code
  - pipeline/primitives
---

# probe.hpp

> 源码: `include/prism/recognition/probe/probe.hpp`
> 模块: [[recognition|Recognition]] / probe

## 概述

从传输层预读数据检测协议类型（HTTP/SOCKS5/TLS/Shadowsocks）。迁移自 `protocol/probe.hpp`，职责下沉到 recognition 模块。`probe()` 是协程函数，通过 `async_read_some()` 预读最多 24 字节后调用 `detect()` 判断协议类型。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[fault/code\|code]] | 错误码 |
| 依赖 | [[fault/handling\|handling]] | 错误处理（`to_code()`） |
| 依赖 | [[protocol/analysis\|analysis]] | protocol_type 枚举 |
| 依赖 | [[channel/transport/transmission\|transmission]] | 传输层抽象 |
| 依赖 | [[memory/container\|container]] | PMR 容器类型 |
| 依赖 | [[recognition/probe/analyzer\|analyzer]] | 协议检测函数 |
| 被依赖 | [[recognition/recognition\|recognition]] | recognize 流程调用 probe |

## 命名空间

`psm::recognition::probe`

---

## 结构体: probe_result

### 概述

外层协议探测结果，包含检测到的协议类型、预读数据和错误码。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `protocol::protocol_type` | `type` | 检测到的协议类型 |
| `std::array<std::byte, 32>` | `pre_read_data` | 预读数据缓冲区（最大 32 字节） |
| `std::size_t` | `pre_read_size` | 实际预读数据大小 |
| `fault::code` | `ec` | 错误代码 |

---

### 成员函数: success()

#### 功能说明

检测探测是否成功。错误码为 `fault::code::success` 且协议类型不是 `unknown` 时返回 `true`。

#### 签名

```cpp
[[nodiscard]] auto success() const noexcept -> bool;
```

#### 参数表格

无参数。

#### 返回值

`bool` — 探测成功返回 `true`。

#### 调用（向下）

无。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `recognize()` | [[recognition/recognition\|recognition]] | 非 TLS 路径检查探测是否成功 |

#### 知识域

- [[fault/code\|错误码]]

---

### 成员函数: preload_view()

#### 功能说明

获取预读数据的字符串视图。将内部缓冲区按实际读取大小转换为 `std::string_view`，适用于文本协议（HTTP）的特征匹配。

#### 签名

```cpp
[[nodiscard]] auto preload_view() const noexcept -> std::string_view;
```

#### 参数表格

无参数。

#### 返回值

`std::string_view` — 预读数据的字符串视图。

#### 调用（向下）

无。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| 外部调用方 | — | 获取文本格式的预读数据 |

#### 知识域

- [[channel/transport/transmission\|传输层]]

---

### 成员函数: preload_bytes()

#### 功能说明

获取预读数据的字节视图。将内部缓冲区按实际读取大小转换为 `std::span<const std::byte>`，适用于二进制协议（TLS ClientHello）的后续处理。

#### 签名

```cpp
[[nodiscard]] auto preload_bytes() const noexcept -> std::span<const std::byte>;
```

#### 参数表格

无参数。

#### 返回值

`std::span<const std::byte>` — 预读数据的字节视图。

#### 调用（向下）

无。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `recognize()` | [[recognition/recognition\|recognition]] | TLS 路径获取预读字节传入 identify |

#### 知识域

- [[channel/transport/transmission\|传输层]]

---

## 函数: probe()

### 功能说明

外层协议探测协程。从传输层异步预读最多 `max_peek_size` 字节（默认 24），读取失败返回错误码，读取 0 字节返回 `eof`，成功后调用 `detect()` 检测协议类型并返回探测结果。

### 签名

```cpp
inline auto probe(channel::transport::transmission &transport, const std::size_t max_peek_size = 24)
    -> net::awaitable<probe_result>;
```

### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `transport` | `channel::transport::transmission &` | 输入 | 传输层对象（socket 或已包装的传输） |
| `max_peek_size` | `std::size_t` | 输入 | 最大预读字节数，默认 24，不超过缓冲区容量 32 |

### 返回值

`net::awaitable<probe_result>` — 探测结果协程。返回包含检测到的协议类型、预读数据和错误码。

### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `transport.async_read_some()` | [[channel/transport/transmission\|transmission]] | 异步预读数据 |
| `fault::to_code()` | [[fault/handling\|handling]] | 将 std::error_code 转换为 fault::code |
| `detect()` | [[recognition/probe/analyzer\|analyzer]] | 从预读数据检测协议类型 |

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `recognize()` | [[recognition/recognition\|recognition]] | Phase 1: 外层协议探测 |

### 知识域

- [[ref/protocol/socks5-rfc1928\|SOCKS5]]
- [[ref/protocol/http-connect\|HTTP]]
- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[ref/programming/c++23-coroutines\|C++23 协程]]

---

## 知识域

- [[ref/protocol/socks5-rfc1928\|SOCKS5]]
- [[ref/protocol/http-connect\|HTTP]]
- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[pipeline/primitives\|预读探测]]
