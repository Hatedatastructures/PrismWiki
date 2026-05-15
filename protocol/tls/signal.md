---
title: "signal — TLS ClientHello 解析器"
source: "include/prism/protocol/tls/signal.hpp"
module: "protocol"
type: api
tags: [protocol, tls, signal, ClientHello, 解析, SNI, 协程]
related:
  - "[[protocol/tls/types|types]]"
  - "[[protocol/tls/feature_bitmap|feature_bitmap]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[stealth/reality/scheme|reality]]"
created: 2026-05-15
updated: 2026-05-15
---

# signal.hpp

> 源码: `include/prism/protocol/tls/signal.hpp`
> 模块: [[protocol|protocol]] > tls

## 概述

TLS ClientHello 解析器。解析 TLS 记录层的 ClientHello 消息，提取 SNI、key_share 公钥、session_id 和 supported_versions 等关键字段。解析器是无状态的，所有方法均为纯函数。该模块是中立的共享层，供 [[recognition|recognition]] 和 [[stealth|stealth]] 模块共同使用。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/tls/types|types]] | TLS 共享类型（`client_hello_features`） |
| 依赖 | [[channel/transport/transmission|transmission]] | 传输层读取接口 |
| 依赖 | [[memory|memory]] | PMR 容器 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[recognition/recognition|recognition]] | 协议识别调用解析 |
| 被依赖 | [[stealth/reality/handshake|reality]] | Reality 握手调用解析 |

## 命名空间

`psm::protocol::tls`

---

## 函数: read_tls_record()

### 功能说明

从传输层读取完整的 TLS 记录（含 5 字节 record header）。调用方应确保 transport 已包装 preview（如有预读数据）。该函数是协程，通过 `co_await` 异步读取数据。

### 签名

```cpp
auto read_tls_record(channel::transport::transmission &transport)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `channel::transport::transmission &` | 底层传输层（应包含预读数据） |

### 返回值

`net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>` — 异步操作，返回错误码和完整 TLS 记录字节数组。

### 调用（向下）

- `transport.async_read_some()` — 传输层异步读取
- `memory::vector` — PMR 缓冲区管理

### 被调用（向上）

- [[recognition/recognition|recognition]] — 识别流程读取 TLS 记录
- [[stealth/reality/handshake|reality]] — Reality 握手读取 ClientHello

### 知识域

TLS 记录层、协程异步 I/O

---

## 函数: read_tls_record()（带预读）

### 功能说明

从传输层读取完整的 TLS 记录，使用已预读的数据作为前缀。当 probe 阶段已预读部分数据时，使用此重载避免重复读取。

### 签名

```cpp
auto read_tls_record(channel::transport::transmission &transport, std::span<const std::byte> preread)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `channel::transport::transmission &` | 底层传输层 |
| `preread` | `std::span<const std::byte>` | 已预读的数据 |

### 返回值

`net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>` — 异步操作，返回错误码和完整 TLS 记录字节数组。

### 调用（向下）

- `transport.async_read_some()` — 传输层异步读取
- `memory::vector` — PMR 缓冲区管理

### 被调用（向上）

- [[recognition/recognition|recognition]] — 识别流程（带 probe 预读数据）

### 知识域

TLS 记录层、协程异步 I/O

---

## 函数: parse_client_hello()

### 功能说明

从完整的 TLS 记录中提取 ClientHello 消息的关键字段，包括 SNI、key_share 公钥、session_id 和 supported_versions。该函数是纯同步函数，不执行 I/O 操作。解析结果通过 `client_hello_features` 结构返回。

### 签名

```cpp
[[nodiscard]] auto parse_client_hello(std::span<const std::uint8_t> record)
    -> std::pair<fault::code, client_hello_features>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `record` | `std::span<const std::uint8_t>` | 完整的 TLS 记录（含 record header） |

### 返回值

`std::pair<fault::code, client_hello_features>` — 错误码和解析后的特征结构。解析失败时错误码指示具体原因。

### 调用（向下）

- [[protocol/tls/types|types]] — 使用 `client_hello_features` 结构和常量
- TLS 记录头解析（ContentType、Version、Length）
- Handshake 消息解析（ClientHello 字段遍历）

### 被调用（向上）

- [[recognition/recognition|recognition]] — 识别流程解析 ClientHello
- [[stealth/reality/handshake|reality]] — Reality 握手解析 ClientHello
- [[stealth/shadowtls/auth|shadowtls]] — ShadowTLS 认证解析 ClientHello

### 知识域

TLS ClientHello 结构、RFC 8446 Section 4.1.2、SNI 扩展解析

## 相关页面

- [[protocol/tls/types|types]] — TLS 共享类型
- [[protocol/tls/feature_bitmap|feature_bitmap]] — 特征位图
- [[stealth/reality/scheme|reality]] — Reality 伪装方案
- [[channel/transport/transmission|transmission]] — 传输层接口
