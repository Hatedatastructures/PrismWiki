---
title: "handshake — ShadowTLS v3 服务端握手"
source: "include/prism/stealth/shadowtls/handshake.hpp"
module: "stealth"
submodule: "shadowtls"
type: api
tags: [stealth, shadowtls, handshake, 握手]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/shadowtls/auth|auth]]"
  - "[[stealth/shadowtls/scheme|scheme]]"
  - "[[stealth/shadowtls/constants|constants]]"
  - "[[stealth/shadowtls/config|config]]"
  - "[[ref/protocol/tls-1.3|TLS 1.3]]"
---

# handshake.hpp

> 源码: `include/prism/stealth/shadowtls/handshake.hpp`
> 实现: `src/prism/stealth/shadowtls/handshake.cpp`
> 模块: [[stealth|stealth]] > [[stealth/shadowtls|shadowtls]]

## 概述

ShadowTLS v3 服务端握手。执行完整的 ShadowTLS v3 握手流程：验证 SessionID 中的 HMAC 标签、与后端服务器完成 TLS 握手、处理数据帧的 HMAC 验证和 XOR 解密。

**与 Reality 的区别**：ShadowTLS 使用标准 TLS 外层，认证发生在 ClientHello 阶段，不需要伪造证书。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/shadowtls/auth|auth]] | 使用 HMAC 验证函数 |
| 依赖 | [[stealth/shadowtls/config|config]] | 使用 ShadowTLS 配置 |
| 依赖 | [[stealth/shadowtls/constants|constants]] | 使用协议常量 |
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 依赖 | [[trace|trace]] | 日志记录 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[stealth/shadowtls/scheme|scheme]] | 方案执行中调用握手函数 |

## 命名空间

`psm::stealth::shadowtls`

---

## 结构体: handshake_result

> 源码: `include/prism/stealth/shadowtls/handshake.hpp:28`

### 概述

ShadowTLS 握手结果。包含认证状态、错误码、客户端首帧数据和匹配的用户名。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| authenticated | `bool` | 是否认证成功 | `false` |
| error | `std::error_code` | 错误码 | 无错误 |
| client_first_frame | `std::vector<std::byte>` | 客户端首帧数据（认证后） | 空 |
| matched_user | `std::string_view` | 匹配的用户名 | 空 |

---

## 函数: handshake()

> 源码: `include/prism/stealth/shadowtls/handshake.hpp:47`
> 实现: `src/prism/stealth/shadowtls/handshake.cpp:431`

### 功能说明

执行完整的 ShadowTLS v3 握手流程。这是 ShadowTLS 方案的核心函数，负责验证客户端身份、与后端服务器建立 TLS 连接、处理数据帧的 HMAC 验证和 XOR 解密。

**完整流程**：
1. 验证 ClientHello 中 SessionID 的 HMAC 标签（v3 遍历多用户，v2 使用单一密码）
2. 解析 `handshake_dest`（host:port），建立到后端服务器的 TCP 连接
3. 转发 ClientHello 到后端服务器
4. 读取后端 ServerHello，返回给客户端
5. 从 ServerHello 提取 ServerRandom（32 字节）
6. strict_mode 下检查后端是否支持 TLS 1.3
7. 启动后台协程：后端 -> 客户端（带 XOR + 累积 HMAC 修改）
8. 前台：读取客户端帧直到 HMAC 匹配，剥离 HMAC 返回首帧数据
9. 关闭后端 socket，等待 relay 协程退出

### 签名

```cpp
auto handshake(net::ip::tcp::socket &client_sock, const config &cfg,
               memory::vector<std::byte> client_hello)
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| client_sock | `net::ip::tcp::socket&` | 客户端 TCP socket |
| cfg | `const config&` | ShadowTLS 配置（含用户列表、握手目标、严格模式等） |
| client_hello | `memory::vector<std::byte>` | 已读取的完整 ClientHello 帧（含 5 字节 TLS 记录头） |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回握手结果：
- `authenticated=true` — 认证成功，`client_first_frame` 包含内层首帧
- `authenticated=false` — 认证失败，`error` 设置具体错误码

### 错误处理

| 错误场景 | error 码 |
|----------|----------|
| ClientHello 为空 | `std::errc::invalid_argument` |
| HMAC 验证失败（所有用户都不匹配） | `std::errc::permission_denied` |
| 后端连接失败 | `std::errc::connection_refused` |
| 转发 ClientHello 失败 | `std::errc::connection_aborted` |
| 读取 ServerHello 失败 | `std::errc::connection_aborted` |
| 提取 ServerRandom 失败 | `std::errc::protocol_error` |
| strict_mode 下后端非 TLS 1.3 | `std::errc::protocol_not_supported` |
| HMAC 匹配阶段失败 | `std::errc::protocol_error` |

### 调用（向下）

- [[stealth/shadowtls/auth|verify_client_hello]] — 验证 ClientHello HMAC（多用户遍历）
- [[stealth/shadowtls/auth|compute_write_key]] — 生成 XOR 写入密钥 `SHA256(password + serverRandom)`
- `read_tls_frame()` — 从 socket 读取完整 TLS 记录帧（内部静态函数）
- `extract_server_random()` — 从 ServerHello 提取 32 字节 ServerRandom（内部静态函数）
- `is_server_hello_tls13()` — 检查 ServerHello 是否支持 TLS 1.3（内部静态函数）
- `read_until_hmac_match()` — 持续读取客户端帧直到 HMAC 匹配（内部静态函数）
- `relay_backend_to_client_with_modification()` — 后端到客户端的 XOR + 累积 HMAC 转发（内部静态函数）
- `xor_with_key()` — XOR 加密辅助函数（内部静态函数）

### 被调用（向上）

- [[stealth/shadowtls/scheme|scheme::handshake]] — 方案执行中调用握手函数

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[ref/protocol/tls-sessionid|TLS SessionID]] — TLS SessionID 字段
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — ServerRandom 提取
- [[stealth/shadowtls/constants|constants]] — 协议常量定义

---

## 内部函数: read_tls_frame()

> 源码: `src/prism/stealth/shadowtls/handshake.cpp:46`

### 功能说明

从 socket 读取一个完整的 TLS 记录帧。先读取 5 字节 TLS 记录头，解析 payload 长度，再读取完整 payload。返回包含 TLS 头的完整帧。

### 签名

```cpp
static auto read_tls_frame(net::ip::tcp::socket &sock)
    -> net::awaitable<std::optional<std::vector<std::byte>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| sock | `net::ip::tcp::socket&` | 目标 socket |

### 返回值

`net::awaitable<std::optional<std::vector<std::byte>>>` — 协程，返回完整 TLS 帧或 `nullopt`（读取失败）

### 调用（向下）

- `net::async_read` — 异步读取 TLS 记录头和 payload

### 被调用（向上）

- `handshake()` — 读取 ServerHello
- `read_until_hmac_match()` — 读取客户端帧
- `relay_backend_to_client_with_modification()` — 读取后端帧

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 记录层格式

---

## 内部函数: extract_server_random()

> 源码: `src/prism/stealth/shadowtls/handshake.cpp:88`

### 功能说明

从 ServerHello 帧中提取 32 字节 ServerRandom。验证帧格式（Content Type = Handshake, Handshake Type = ServerHello），然后从固定偏移提取 Random 字段。

### 签名

```cpp
static auto extract_server_random(std::span<const std::byte> server_hello)
    -> std::optional<std::array<std::byte, tls_random_size>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| server_hello | `std::span<const std::byte>` | ServerHello TLS 帧 |

### 返回值

`std::optional<std::array<std::byte, 32>>` — 32 字节 ServerRandom 或 `nullopt`

### 调用（向下）

无内部调用。

### 被调用（向上）

- `handshake()` — 提取 ServerRandom 用于后续 HMAC 和 XOR 计算

### 知识域

- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — ServerRandom 字段位置

---

## 内部函数: is_server_hello_tls13()

> 源码: `src/prism/stealth/shadowtls/handshake.cpp:107`

### 功能说明

检查 ServerHello 是否协商了 TLS 1.3。解析 ServerHello 扩展列表，查找 `supported_versions` 扩展（类型 43），检查版本号是否为 `0x0304`。

### 签名

```cpp
static bool is_server_hello_tls13(std::span<const std::byte> server_hello);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| server_hello | `std::span<const std::byte>` | ServerHello TLS 帧 |

### 返回值

`bool` — `true` 表示 TLS 1.3，`false` 表示非 TLS 1.3 或解析失败

### 调用（向下）

无内部调用。

### 被调用（向上）

- `handshake()` — strict_mode 下验证后端 TLS 版本

### 知识域

- [[ref/protocol/tls-extensions|TLS 扩展]] — supported_versions 扩展
- [[stealth/shadowtls/constants|constants]] — `extension_supported_versions`, `tls_version_1_3`

---

## 内部函数: read_until_hmac_match()

> 源码: `src/prism/stealth/shadowtls/handshake.cpp:186`

### 功能说明

持续读取客户端 TLS 帧直到 HMAC 匹配。这是 ShadowTLS v3 握手的关键步骤：客户端在 Application Data 帧中嵌入 HMAC 标签证明身份。

**处理逻辑**：
- 非 Application Data 帧（如 TLS 握手记录）：原样转发到后端
- Application Data 帧：提取前 4 字节 HMAC，计算 `HMAC-SHA1(password, serverRandom + "C" + payload)` 进行比较
- HMAC 匹配：剥离 HMAC 头，返回首帧数据
- HMAC 不匹配：原样转发到后端

使用 OpenSSL HMAC_CTX 的 Reset/Update/Final 模式，每帧重置到 `HMAC(password, serverRandom + "C")` 状态。

### 签名

```cpp
static auto read_until_hmac_match(
    net::ip::tcp::socket &client_sock,
    net::ip::tcp::socket &backend_sock,
    std::string_view password,
    std::span<const std::byte> server_random)
    -> net::awaitable<std::optional<std::vector<std::byte>>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| client_sock | `net::ip::tcp::socket&` | 客户端 socket |
| backend_sock | `net::ip::tcp::socket&` | 后端 socket（转发不匹配帧） |
| password | `std::string_view` | 用户密码 |
| server_random | `std::span<const std::byte>` | 32 字节 ServerRandom |

### 返回值

`net::awaitable<std::optional<std::vector<std::byte>>>` — 协程，HMAC 匹配时返回剥离 HMAC 后的首帧数据，失败返回 `nullopt`

### 调用（向下）

- `read_tls_frame()` — 读取 TLS 帧
- `net::async_write` — 转发不匹配帧到后端
- OpenSSL `HMAC_CTX` — HMAC 计算

### 被调用（向上）

- `handshake()` — 握手阶段读取客户端认证帧

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[stealth/shadowtls/constants|constants]] — `hmac_size`, `tls_hmac_header_size`

---

## 内部函数: relay_backend_to_client_with_modification()

> 源码: `src/prism/stealth/shadowtls/handshake.cpp:308`

### 功能说明

转发后端服务器数据到客户端（带 XOR + 累积 HMAC 修改）。参照 sing-shadowtls `copyByFrameWithModification`。

**处理逻辑**：
- 非 Application Data 帧（如 ChangeCipherSpec）：原样转发
- Application Data 帧：
  1. XOR 加密 payload（使用 `SHA256(password + serverRandom)` 作为密钥）
  2. 累积 HMAC 更新（不 Reset，跨帧累积）
  3. 取当前累积 HMAC 的前 4 字节作为标签
  4. 将 HMAC 标签写回累积上下文
  5. 构建新帧：`[TLS Header(5)] [HMAC(4)] [XOR'd payload]`
  6. 发送到客户端

**帧格式**：`[0x17 0x03 0x03 len_hi len_lo] [hmac_tag(4)] [xor'd_payload]`

### 签名

```cpp
static auto relay_backend_to_client_with_modification(
    net::ip::tcp::socket &backend_sock,
    net::ip::tcp::socket &client_sock,
    std::string_view password,
    std::span<const std::byte> server_random) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| backend_sock | `net::ip::tcp::socket&` | 后端 socket（数据源） |
| client_sock | `net::ip::tcp::socket&` | 客户端 socket（数据目标） |
| password | `std::string_view` | 用户密码 |
| server_random | `std::span<const std::byte>` | 32 字节 ServerRandom |

### 返回值

`net::awaitable<void>` — 协程，直到后端关闭或写入失败时退出

### 调用（向下）

- [[stealth/shadowtls/auth|compute_write_key]] — 生成 XOR 写入密钥
- `read_tls_frame()` — 读取后端帧
- `xor_with_key()` — XOR 加密 payload
- `net::async_write` — 发送修改后的帧到客户端

### 被调用（向上）

- `handshake()` — 作为后台协程启动

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — 累积 HMAC 模式
- [[ref/crypto/sha256|SHA-256]] — XOR 密钥派生
- [[stealth/shadowtls/constants|constants]] — 帧格式常量
