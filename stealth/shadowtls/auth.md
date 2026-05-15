---
title: "auth — ShadowTLS v3 认证逻辑"
source: "include/prism/stealth/shadowtls/auth.hpp"
module: "stealth"
submodule: "shadowtls"
type: api
tags: [stealth, shadowtls, auth, hmac, 认证]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/shadowtls/handshake|handshake]]"
  - "[[stealth/shadowtls/scheme|scheme]]"
  - "[[stealth/shadowtls/constants|constants]]"
  - "[[ref/crypto/hmac-sha1|HMAC-SHA1]]"
  - "[[ref/crypto/sha256|SHA-256]]"
---

# auth.hpp

> 源码: `include/prism/stealth/shadowtls/auth.hpp`
> 实现: `src/prism/stealth/shadowtls/auth.cpp`
> 模块: [[stealth|stealth]] > [[stealth/shadowtls|shadowtls]]

## 概述

ShadowTLS v3 认证逻辑。基于 HMAC-SHA1 的认证机制，ShadowTLS v3 在 TLS ClientHello 的 SessionID 字段中嵌入 4 字节 HMAC 标签进行身份验证。

**认证算法**（参照 sing-shadowtls v3_server.go `verifyClientHello`）：
```
HMAC = HMAC-SHA1(password, ClientHello[10:hmac_index] + 00000000 + ClientHello[hmac_index+4:])[:4]
```

**握手完成后的数据帧处理**（参照 sing-shadowtls）：
- 客户端->服务端: `HMAC_Verify = HMAC-SHA1(password, serverRandom + "C" + payload)[:4]`
- 服务端->客户端: `HMAC_Write = HMAC-SHA1(password, serverRandom + "S" + payload)[:4]`（单帧独立）
- Relay（后端->客户端）: `HMAC = HMAC-SHA1(password, serverRandom + payload)[:4]`（累积，无后缀）
- `WriteKey = SHA256(password + serverRandom)`

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | OpenSSL | HMAC-SHA1 (`HMAC`, `HMAC_CTX`)、SHA-256 (`SHA256`)、恒定时间比较 (`CRYPTO_memcmp`) |
| 依赖 | [[stealth/shadowtls/constants|constants]] | 使用 `hmac_size`, `tls_header_size` 等常量 |
| 被依赖 | [[stealth/shadowtls/handshake|handshake]] | 握手流程中调用验证函数 |
| 被依赖 | [[stealth/shadowtls/scheme|scheme]] | 方案 verify() 中调用 verify_client_hello |

## 命名空间

`psm::stealth::shadowtls`

---

## 函数: verify_client_hello()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:36`
> 实现: `src/prism/stealth/shadowtls/auth.cpp:34`

### 功能说明

验证 ClientHello 中的 SessionID HMAC。从 ClientHello 帧（含 TLS 记录头）中提取 SessionID 的后 4 字节 HMAC 标签，与本地计算结果进行恒定时间比较。这是 ShadowTLS v3 的核心认证函数，用于在 TLS 握手阶段验证客户端身份。

**验证流程**：
1. 最小长度检查：TLS Header(5) + Handshake Header(4) + Version(2) + Random(32) + SessionID Length(1) + SessionID(32) = 76
2. TLS 记录类型必须是 Handshake (0x16)
3. 握手类型必须是 ClientHello (0x01)
4. SessionID 长度必须是 32
5. 构建 HMAC 计算数据：ClientHello 去掉 TLS header，SessionID 中 HMAC 位置填 0
6. 计算 `HMAC-SHA1(password, modified_data)[:4]`
7. 提取客户端 SessionID 中的 HMAC 标签
8. 使用 `CRYPTO_memcmp` 恒定时间比较

### 签名

```cpp
[[nodiscard]] auto verify_client_hello(std::span<const std::byte> client_hello,
                                        std::string_view password) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| client_hello | `std::span<const std::byte>` | ClientHello 帧数据（含 5 字节 TLS 记录头） |
| password | `std::string_view` | 认证密码 |

### 返回值

`bool` — 认证通过返回 `true`，失败返回 `false`

### 调用（向下）

- compute_hmac() — 计算 HMAC-SHA1 标签
- OpenSSL `CRYPTO_memcmp` — 恒定时间比较（防止时序攻击）

### 被调用（向上）

- [[stealth/shadowtls/handshake|handshake]] — 握手流程中验证客户端身份（多用户遍历）
- [[stealth/shadowtls/scheme|scheme::verify]] — 方案 Tier 1 检测中验证客户端身份

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[ref/protocol/tls-sessionid|TLS SessionID]] — TLS SessionID 字段结构
- [[stealth/shadowtls/constants|constants]] — `session_id_length_index`, `tls_session_id_size`, `hmac_size`
- [[ref/anti-censorship/replay-attack|重放攻击]] — 恒定时间比较防止时序侧信道

---

## 函数: compute_hmac()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:47`
> 实现: `src/prism/stealth/shadowtls/auth.cpp:18`

### 功能说明

计算 HMAC-SHA1 标签。通用的 HMAC-SHA1 计算函数，使用 OpenSSL `HMAC` 接口，返回前 4 字节作为标签。用于 ShadowTLS 的各种认证场景。

### 签名

```cpp
[[nodiscard]] auto compute_hmac(std::string_view key, const std::byte *data,
                                 std::size_t data_len)
    -> std::array<std::uint8_t, 4>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| key | `std::string_view` | HMAC 密钥（密码） |
| data | `const std::byte*` | 要认证的数据指针 |
| data_len | `std::size_t` | 数据长度（字节） |

### 返回值

`std::array<std::uint8_t, 4>` — 4 字节 HMAC 标签

### 调用（向下）

- OpenSSL `HMAC(EVP_sha1(), ...)` — 执行 HMAC-SHA1 计算

### 被调用（向上）

- verify_client_hello() — 验证 ClientHello
- verify_frame_hmac() — 验证数据帧（通过 HMAC_CTX 累积模式）
- compute_write_hmac() — 计算写入 HMAC

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[stealth/shadowtls/constants|constants]] — `hmac_size` = 4

---

## 函数: verify_frame_hmac()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:61`
> 实现: `src/prism/stealth/shadowtls/auth.cpp:90`

### 功能说明

验证握手后的数据帧 HMAC（客户端->服务端方向，单帧独立模式）。计算 `HMAC-SHA1(password, serverRandom + "C" + payload)[:4]`，与客户端帧头中的 4 字节 HMAC 进行恒定时间比较。

**注意**：此函数用于单帧独立 HMAC 验证。`handshake.cpp` 中的 `read_until_hmac_match()` 使用累积 HMAC 模式（参照 sing-shadowtls `hmacVerifyReset`），与此函数的独立模式不同。

### 签名

```cpp
[[nodiscard]] auto verify_frame_hmac(std::string_view password,
                                      std::span<const std::byte> server_random,
                                      std::span<const std::byte> payload,
                                      std::span<const std::uint8_t, 4> client_hmac) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| password | `std::string_view` | 密码 |
| server_random | `std::span<const std::byte>` | TLS ServerRandom（32 字节） |
| payload | `std::span<const std::byte>` | 数据帧 payload（不含 TLS header 和 HMAC） |
| client_hmac | `std::span<const std::uint8_t, 4>` | 客户端帧头中的 4 字节 HMAC |

### 返回值

`bool` — 匹配返回 `true`，不匹配返回 `false`

### 调用（向下）

- OpenSSL `HMAC_CTX` — 分步 HMAC 计算（Init -> Update(serverRandom) -> Update("C") -> Update(payload) -> Final）
- OpenSSL `CRYPTO_memcmp` — 恒定时间比较

### 被调用（向上）

- [[stealth/shadowtls/handshake|handshake]] — 数据帧验证（备用接口）

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — TLS ServerRandom 字段
- [[stealth/shadowtls/constants|constants]] — `hmac_size`

---

## 函数: compute_write_hmac()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:76`
> 实现: `src/prism/stealth/shadowtls/auth.cpp:118`

### 功能说明

生成数据帧的 HMAC 标签（服务端->客户端方向，单帧独立模式）。计算 `HMAC-SHA1(password, serverRandom + "S" + payload)[:4]`。使用 "S" 后缀区分服务端方向。

**注意**：此函数用于单帧独立 HMAC。`handshake.cpp` 中的 `relay_backend_to_client_with_modification()` 使用累积 HMAC 模式（无后缀），与此函数的独立模式不同。

### 签名

```cpp
[[nodiscard]] auto compute_write_hmac(std::string_view password,
                                       std::span<const std::byte> server_random,
                                       std::span<const std::byte> payload)
    -> std::array<std::uint8_t, 4>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| password | `std::string_view` | 密码 |
| server_random | `std::span<const std::byte>` | TLS ServerRandom（32 字节） |
| payload | `std::span<const std::byte>` | 修改后的 payload |

### 返回值

`std::array<std::uint8_t, 4>` — 4 字节 HMAC 标签

### 调用（向下）

- OpenSSL `HMAC_CTX` — 分步 HMAC 计算（Init -> Update(serverRandom) -> Update("S") -> Update(payload) -> Final）

### 被调用（向上）

- [[stealth/shadowtls/handshake|handshake]] — 生成数据帧 HMAC（备用接口）

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — TLS ServerRandom 字段

---

## 函数: compute_write_key()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:88`
> 实现: `src/prism/stealth/shadowtls/auth.cpp:148`

### 功能说明

生成写入密钥（XOR 加密用）。计算 `SHA256(password + serverRandom)`，返回 32 字节密钥。用于数据帧的 XOR 加密，在 `relay_backend_to_client_with_modification()` 中使用。

### 签名

```cpp
[[nodiscard]] auto compute_write_key(std::string_view password,
                                      std::span<const std::byte> server_random)
    -> std::vector<std::uint8_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| password | `std::string_view` | 密码 |
| server_random | `std::span<const std::byte>` | TLS ServerRandom（32 字节） |

### 返回值

`std::vector<std::uint8_t>` — 32 字节写入密钥（SHA256 输出）

### 调用（向下）

- OpenSSL `SHA256` — 分步 SHA-256 计算（Init -> Update(password) -> Update(serverRandom) -> Final）

### 被调用（向上）

- [[stealth/shadowtls/handshake|relay_backend_to_client_with_modification]] — 生成 XOR 写入密钥

### 知识域

- [[ref/crypto/sha256|SHA-256]] — SHA-256 哈希算法
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — TLS ServerRandom 字段
