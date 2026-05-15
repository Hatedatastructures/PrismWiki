---
title: "handshake — Reality 握手状态机"
source: "include/prism/stealth/reality/handshake.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, handshake, 握手, 状态机]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/scheme|scheme]]"
  - "[[stealth/reality/auth|auth]]"
  - "[[stealth/reality/keygen|keygen]]"
  - "[[stealth/reality/seal|seal]]"
  - "[[stealth/reality/response|response]]"
  - "[[stealth/reality/request|request]]"
---

# handshake.hpp

> 源码: `include/prism/stealth/reality/handshake.hpp`
> 实现: `src/prism/stealth/reality/handshake.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality 握手状态机。Reality 协议的核心入口，协调 [[stealth/reality/request|ClientHello 解析]]、[[stealth/reality/auth|认证]]、[[stealth/reality/response|ServerHello 生成]]、[[stealth/reality/keygen|密钥派生]] 和回退逻辑。由 [[stealth/reality/scheme|scheme]] 层调用。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | 使用 session_context |
| 依赖 | [[channel/transport/transmission|transmission]] | 使用 shared_transmission |
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 依赖 | [[stealth/reality/auth|auth]] | 使用 authenticate() 认证函数 |
| 依赖 | [[stealth/reality/keygen|keygen]] | 使用密钥派生函数 |
| 依赖 | [[stealth/reality/response|response]] | 使用 generate_server_hello() |
| 依赖 | [[stealth/reality/seal|seal]] | 使用 seal 创建加密传输层 |
| 依赖 | [[stealth/reality/config|config]] | 使用 Reality 配置 |
| 依赖 | [[resolve/router|router]] | 使用路由器进行 fallback |
| 依赖 | [[crypto/base64|base64]] | 解码私钥 |
| 依赖 | [[crypto/x25519|x25519]] | TLS ECDH 密钥交换 |
| 被依赖 | [[stealth/reality/scheme|scheme]] | 方案执行中调用握手函数 |

## 命名空间

`psm::stealth::reality`

---

## 枚举: handshake_result_type

> 源码: `include/prism/stealth/reality/handshake.hpp:31`

### 概述

握手结果类型枚举。表示 Reality 握手的四种可能结果。

### 枚举值

| 值 | 说明 |
|-----|------|
| authenticated | Reality 认证成功，返回 [[stealth/reality/seal|seal]] 加密传输层 |
| not_reality | 非 Reality 客户端（SNI 不匹配），应走标准 TLS |
| fallback | 回退到 dest 服务器，透明代理已完成 |
| failed | 错误 |

---

## 结构体: handshake_result

> 源码: `include/prism/stealth/reality/handshake.hpp:49`

### 概述

握手结果。包含握手结果类型、加密传输层（认证成功时）、内层预读数据、原始 TLS 记录（非 Reality 时）和错误码。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| type | handshake_result_type | 握手结果类型 |
| encrypted_transport | shared_transmission | type==authenticated 时为 [[stealth/reality/seal|seal]] 加密传输层 |
| inner_preread | memory::vector\<std::byte\> | type==authenticated 时为内层预读数据 |
| raw_tls_record | memory::vector\<std::byte\> | type==not_reality 时为原始 ClientHello TLS record |
| error | fault::code | 错误码（默认 success） |

---

## 函数: handshake()

> 源码: `include/prism/stealth/reality/handshake.hpp:67`
> 实现: `src/prism/stealth/reality/handshake.cpp:399`

### 功能

执行 Reality 握手。读取 ClientHello，尝试 Reality 认证，成功则建立 [[stealth/reality/seal|seal]] 加密传输层，失败则回退到 dest 服务器的标准 TLS 或直接透传。这是 Reality 协议的核心入口函数。

### 签名

```cpp
auto handshake(channel::transport::shared_transmission inbound,
               const psm::config &cfg,
               psm::agent::session_context &session)
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| inbound | shared_transmission | 入站传输层（不转移所有权，仅使用） |
| cfg | const psm::config& | 服务器配置 |
| session | psm::agent::session_context& | 会话上下文（用于 fallback 等需要 router 的场景） |

### 返回值

`net::awaitable<handshake_result>` — 异步操作，返回握手结果

### 错误处理

- `handshake_result_type::authenticated` — 认证成功
- `handshake_result_type::not_reality` — 非 Reality 客户端
- `handshake_result_type::fallback` — 回退到 dest 服务器
- `handshake_result_type::failed` — 错误

### 调用（向下）

- protocol::tls::read_tls_record — 读取 ClientHello TLS 记录
- protocol::tls::parse_client_hello — 解析 ClientHello 特征
- [[crypto/base64|base64_decode]] — 解码私钥
- [[stealth/reality/auth|authenticate]] — Reality 认证
- [[crypto/x25519|x25519]] — TLS ECDH 密钥交换
- [[stealth/reality/response|generate_server_hello]] — 生成 ServerHello
- [[stealth/reality/keygen|derive_handshake_keys]] — 派生握手密钥
- [[stealth/reality/keygen|compute_finished_verify_data]] — 计算 Finished
- [[stealth/reality/response|encrypt_tls_record]] — 加密握手记录
- [[stealth/reality/keygen|derive_application_keys]] — 派生应用密钥
- [[stealth/reality/seal|seal]] — 创建加密传输层
- fallback_to_dest() — 回退到 dest

### 被调用（向上）

- [[stealth/reality/scheme|scheme::handshake()]] — 方案执行中调用

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[ref/crypto/x25519|X25519]] — X25519 密钥交换

### 流程

1. 读取 ClientHello TLS 记录
2. 解析 ClientHello 特征（SNI、key_share、session_id 等）
3. 解码 base64 私钥
4. 调用 `authenticate()` 进行 Reality 认证
   - SNI 不匹配 → `not_reality`
   - 认证失败 → `not_reality`
5. TLS ECDH 密钥交换（服务端临时密钥 + 客户端公钥）
6. 生成 ServerHello + 加密握手记录
7. 派生握手流量密钥
8. 用正确密钥重算 Finished 并加密
9. 发送 ServerHello + CCS + 加密握手记录（scatter-gather）
10. 消费客户端 CCS + Finished
11. 派生应用数据密钥
12. 创建 [[stealth/reality/seal|seal]] 加密传输层
13. 预读 64 字节内层数据
14. 返回 `authenticated` 结果

---

## 函数: fallback_to_dest()

> 源码: `include/prism/stealth/reality/handshake.hpp:81`
> 实现: `src/prism/stealth/reality/handshake.cpp:122`

### 功能

执行回退：连接 dest 服务器并透明代理。通过 [[resolve/router|router]] 解析并连接配置的 dest 目标服务器，将原始 TLS 记录转发过去，然后使用 [[pipeline/primitives|tunnel]] 双向转发数据，完成后由上层继续处理内层协议。

### 签名

```cpp
auto fallback_to_dest(psm::agent::session_context &session,
                      channel::transport::shared_transmission inbound,
                      std::span<const std::uint8_t> raw_record)
    -> net::awaitable<fault::code>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| session | psm::agent::session_context& | 会话上下文（用于 router 和 tunnel） |
| inbound | shared_transmission | 入站传输层 |
| raw_record | std::span<const std::uint8_t> | 原始 ClientHello TLS 记录字节 |

### 返回值

`net::awaitable<fault::code>` — 异步操作，返回错误码

### 调用（向下）

- parse_dest() — 解析 dest 地址
- [[resolve/router|router::async_forward]] — DNS 解析和连接
- net::async_write — 转发原始 TLS 记录
- [[pipeline/primitives|tunnel]] — 双向数据转发

### 被调用（向上）

- handshake() — 握手流程中回退

### 知识域

- [[resolve/router|router]] — DNS 解析和连接管理
- [[pipeline/primitives|tunnel]] — 双向透明转发

### 流程

1. 解析 dest 配置为 host:port
2. 通过 router 连接 dest 服务器
3. 将原始 ClientHello TLS 记录写入 dest
4. 创建 dest 传输层
5. 调用 `tunnel()` 双向转发
6. 返回 success 或错误码

---

## 函数: parse_dest()

> 源码: `include/prism/stealth/reality/handshake.hpp:94`
> 实现: `src/prism/stealth/reality/handshake.cpp:73`

### 功能

从 dest 配置中解析 host 和 port。支持三种格式：`host:port`、`host`（默认端口 443）、`[ipv6]:port`。使用 `std::from_chars` 解析端口号。

### 签名

```cpp
auto parse_dest(std::string_view dest, std::string &host, std::uint16_t &port) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| dest | std::string_view | 目标地址字符串（host:port 格式） |
| host | std::string& | 输出参数，解析出的主机名 |
| port | std::uint16_t& | 输出参数，解析出的端口号（默认 443） |

### 返回值

`bool` — 解析成功返回 `true`，格式错误返回 `false`

### 调用（向下）

- 无（纯函数）

### 被调用（向上）

- handshake() — 握手流程中解析 dest
- fallback_to_dest() — 回退流程中解析 dest

### 知识域

- 网络编程 — IPv4/IPv6 地址解析

---

## 函数: fetch_dest_certificate()

> 源码: `include/prism/stealth/reality/handshake.hpp:106`
> 实现: `src/prism/stealth/reality/handshake.cpp:170`

### 功能

从 dest 服务器获取证书。通过 [[resolve/router|router]] 解析目标地址并建立 TLS 连接，使用 BoringSSL 获取目标网站的 DER 格式证书用于 Reality 伪造。设置 `verify_none` 模式，不验证证书链。

### 签名

```cpp
auto fetch_dest_certificate(std::string_view host, std::uint16_t port, resolve::router &router)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| host | std::string_view | 目标主机名 |
| port | std::uint16_t | 目标端口 |
| router | resolve::router& | 路由器引用，用于 DNS 解析和连接 |

### 返回值

`net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>` — 异步操作，返回错误码和 DER 格式证书

### 调用（向下）

- [[resolve/router|router::async_forward]] — DNS 解析和连接
- net::ssl::stream::async_handshake — TLS 握手
- SSL_get_peer_certificate — 获取对端证书
- i2d_X509_bio — DER 编码证书

### 被调用（向上）

- handshake() — 握手流程中获取证书（当前未使用，Reality 认证客户端使用合成证书）

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 握手和证书获取
- OpenSSL API — X509 证书操作
