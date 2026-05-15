---
title: "request — TLS ClientHello 解析器"
source: "include/prism/stealth/reality/request.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, request, clienthello, 解析]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/auth|auth]]"
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/reality/response|response]]"
  - "[[stealth/reality/constants|constants]]"
  - "[[memory/container|container]]"
---

# request.hpp

> 源码: `include/prism/stealth/reality/request.hpp`
> 实现: `src/prism/stealth/reality/request.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

TLS ClientHello 解析器。解析 TLS 记录层的 ClientHello 消息，提取 SNI、key_share 公钥、session_id 和 supported_versions 等关键字段，用于 Reality 协议的认证前置步骤：从客户端的 ClientHello 中获取认证所需的全部信息。

**注意**：解析器是无状态的，所有方法均为纯函数。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::vector/string PMR 容器 |
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[channel/transport/transmission|transmission]] | read_tls_record 使用传输层接口 |
| 被依赖 | [[stealth/reality/auth|auth]] | 认证函数使用解析结果 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手函数使用解析结果 |
| 被依赖 | [[stealth/reality/response|response]] | ServerHello 生成使用 client_hello_info |

## 命名空间

`psm::stealth::reality`

---

## 结构体: client_hello_info

> 源码: `include/prism/stealth/reality/request.hpp:35`

### 概述

ClientHello 解析结果。存储从 TLS ClientHello 消息中提取的认证所需字段。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| raw_message | memory::vector\<uint8_t\> | 完整 ClientHello handshake 消息字节，用于 transcript hash 计算 |
| random | std::array\<uint8_t, 32\> | 客户端随机数（32 字节），后 12 字节用作 AEAD nonce |
| session_id | memory::vector\<uint8_t\> | session_id（包含 Reality 的 short_id 和认证数据） |
| server_name | memory::string | SNI 服务器名称 |
| client_public_key | std::array\<uint8_t, 32\> | key_share 扩展中的 X25519 公钥（32 字节） |
| has_client_public_key | bool | 是否包含客户端公钥 |
| supported_versions | memory::vector\<uint16_t\> | 客户端支持的 TLS 版本列表 |

---

## 函数: read_tls_record()

> 源码: `include/prism/stealth/reality/request.hpp:54`
> 实现: `src/prism/stealth/reality/request.cpp`

### 功能

读取完整的 TLS 记录。从传输层读取 5 字节 TLS 记录头获取长度，再读取剩余载荷数据，组装为完整 TLS 记录。调用方应确保 transport 已包装 preview（如有预读数据）。

### 签名

```cpp
auto read_tls_record(channel::transport::transmission &transport)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| transport | channel::transport::transmission& | 底层传输（应包含预读数据） |

### 返回值

`net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>` — 异步操作，返回错误码和完整 TLS 记录（含 5 字节 record header）

### 调用（向下）

- [[channel/transport/transmission|transmission::async_read_some]] — 异步读取传输层数据

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中读取 TLS 记录

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 记录层格式（ContentType + Version + Length + Fragment）

---

## 函数: parse_client_hello()

> 源码: `include/prism/stealth/reality/request.hpp:64`
> 实现: `src/prism/stealth/reality/request.cpp`

### 功能

解析 ClientHello。从完整的 TLS 记录中提取 ClientHello 消息的关键字段，包括 random、session_id、SNI、key_share 公钥和 supported_versions。这是 Reality 认证的前置步骤。

### 签名

```cpp
[[nodiscard]] auto parse_client_hello(std::span<const std::uint8_t> raw_tls_record)
    -> std::pair<fault::code, client_hello_info>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| raw_tls_record | std::span<const std::uint8_t> | 完整的 TLS 记录（含 record header） |

### 返回值

`std::pair<fault::code, client_hello_info>` — 错误码和解析结果

### 错误处理

- `fault::code::success` — 解析成功
- `fault::code::invalid_tls_record` — 无效的 TLS 记录（长度不匹配、非 handshake 类型）
- `fault::code::invalid_client_hello` — 无效的 ClientHello（非 ClientHello 消息类型）

### 调用（向下）

- 无（纯函数，只做二进制解析）

### 被调用（向上）

- [[stealth/reality/auth|authenticate]] — 认证流程中解析 ClientHello
- [[stealth/reality/handshake|handshake]] — 握手流程中解析 ClientHello

### 知识域

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — TLS ClientHello 结构
- [[ref/protocol/tls-extensions|TLS 扩展]] — TLS 扩展（SNI=0x0000, key_share=0x0033, supported_versions=0x002b）

### 流程

1. 解析 TLS 记录头（5 字节：ContentType + Version + Length）
2. 验证 ContentType 为 handshake (0x16)
3. 解析 ClientHello handshake 消息头（4 字节：Type + Length）
4. 验证 Type 为 ClientHello (0x01)
5. 提取 client_version（2 字节）
6. 提取 random（32 字节）
7. 提取 session_id（变长，1 字节长度前缀）
8. 提取 cipher_suites（变长）
9. 提取 compression_methods（变长）
10. 遍历扩展列表，提取：
    - SNI (0x0000) → server_name
    - key_share (0x0033) → X25519 公钥
    - supported_versions (0x002b) → 版本列表
11. 返回解析结果

### 注意事项

- 解析器是无状态的，所有方法均为纯函数
- 不验证字段的合法性，只做提取
- 返回的 raw_message 包含完整的 ClientHello handshake 消息（不含 record header）
