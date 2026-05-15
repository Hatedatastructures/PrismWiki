---
title: "auth — Reality 认证逻辑"
source: "include/prism/stealth/reality/auth.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, auth, 认证, x25519, hkdf]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/reality/request|request]]"
  - "[[stealth/reality/config|config]]"
  - "[[crypto/x25519|x25519]]"
  - "[[crypto/hkdf|hkdf]]"
  - "[[crypto/aead|aead]]"
---

# auth.hpp

> 源码: `include/prism/stealth/reality/auth.hpp`
> 实现: `src/prism/stealth/reality/auth.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality 认证逻辑。实现 Reality 协议的客户端认证流程：检查 SNI 是否匹配 `server_names`，检查 `key_share` 扩展是否包含 X25519 公钥，计算 X25519 共享密钥，验证 `session_id` 中的 `short_id` 和认证数据。认证成功后返回共享密钥和服务端临时密钥对。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[crypto/x25519|x25519]] | 使用 X25519 密钥交换和密钥对生成 |
| 依赖 | [[crypto/hkdf|hkdf]] | 使用 HKDF-Extract/Expand 派生认证密钥 |
| 依赖 | [[crypto/aead|aead]] | 使用 AES-256-GCM 解密 session_id |
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[stealth/reality/config|config]] | 使用 Reality 配置 |
| 依赖 | [[stealth/reality/request|request]] | 使用 client_hello_info 结构 |
| 依赖 | [[stealth/reality/constants|constants]] | 使用协议常量 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手流程中调用认证函数 |

## 命名空间

`psm::stealth::reality`

---

## 结构体: auth_result

> 源码: `include/prism/stealth/reality/auth.hpp:28`

### 概述

Reality 认证结果。包含认证状态、X25519 共享密钥、服务端临时密钥对和 HKDF 派生的认证密钥。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| authenticated | bool | 认证是否成功 |
| shared_secret | std::array\<uint8_t, 32\> | X25519 共享密钥 |
| server_ephemeral_key | crypto::x25519_keypair | 服务端临时 X25519 密钥对（用于 ServerHello 的 key_share） |
| auth_key | std::array\<uint8_t, 32\> | HKDF 派生的认证密钥（用于 Ed25519 证书签名） |

---

## 函数: authenticate()

> 源码: `include/prism/stealth/reality/auth.hpp:45`
> 实现: `src/prism/stealth/reality/auth.cpp:50`

### 功能

执行 Reality 认证。按 12 步流程验证客户端身份：SNI 匹配、X25519 公钥检查、TLS 1.3 支持检查、session_id 长度检查、X25519 密钥交换、全零共享密钥检查、HKDF 派生认证密钥、构造 AAD、AES-256-GCM 解密 session_id、验证格式标记、验证 short_id、生成服务端临时密钥对。

### 签名

```cpp
[[nodiscard]] auto authenticate(const config &cfg, const client_hello_info &client_hello,
                                std::span<const std::uint8_t> decoded_private_key)
    -> std::pair<fault::code, auth_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const config& | Reality 配置（server_names、short_ids 等） |
| client_hello | const client_hello_info& | 解析后的 ClientHello 信息 |
| decoded_private_key | std::span<const std::uint8_t> | 已 base64 解码的 32 字节私钥 |

### 返回值

`std::pair<fault::code, auth_result>` — 错误码和认证结果

### 错误处理

- `fault::code::success` — 认证成功
- `fault::code::reality_sni_mismatch` — SNI 不匹配
- `fault::code::reality_auth_failed` — 认证失败（无公钥、不支持 TLS 1.3、session_id 过短、解密失败、格式错误、short_id 不匹配）
- `fault::code::reality_key_exchange_failed` — X25519 密钥交换失败或共享密钥全零

### 调用（向下）

- match_server_name() — 检查 SNI 匹配
- [[crypto/x25519|x25519]] — 计算 X25519 共享密钥
- [[crypto/hkdf|hkdf_extract]] — HKDF-Extract 派生 PRK
- [[crypto/hkdf|hkdf_expand]] — HKDF-Expand 派生认证密钥
- [[crypto/aead|aead_context]] — AES-256-GCM 解密 session_id
- match_short_id() — 验证 short_id
- [[crypto/x25519|generate_x25519_keypair]] — 生成服务端临时密钥对

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中调用认证函数

### 知识域

- [[ref/crypto/x25519|X25519]] — X25519 密钥交换
- [[ref/crypto/hkdf|HKDF]] — HKDF 密钥派生
- [[ref/crypto/aes-gcm|AES-GCM]] — AES-256-GCM 认证加密
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — TLS ClientHello 结构

### 流程

1. 检查 SNI 是否匹配 `server_names`（空 SNI 跳过检查）
2. 检查 `key_share` 扩展是否包含 X25519 公钥
3. 检查客户端是否支持 TLS 1.3
4. 检查 `session_id` 长度是否 >= 32 字节
5. 计算 X25519 共享密钥（服务端私钥 + 客户端公钥）
6. 检查共享密钥是否为全零（低阶点攻击防护）
7. HKDF-Extract(random[0:20], shared_secret) → PRK
8. HKDF-Expand(PRK, "REALITY", 32) → auth_key
9. 构造 AAD（原始 ClientHello，session_id 字段清零）
10. AES-256-GCM 解密 session_id（nonce = random[20:32]）
11. 验证解密后 session_id[0] == 0x01（版本标记）
12. 验证 short_id 匹配
13. 生成服务端临时 X25519 密钥对
14. 返回认证结果

---

## 函数: match_server_name()

> 源码: `include/prism/stealth/reality/auth.hpp:56`
> 实现: `src/prism/stealth/reality/auth.cpp:12`

### 功能

检查 SNI 是否匹配。遍历 `server_names` 列表检查是否有完全匹配项。SNI 为空时返回 false。

### 签名

```cpp
[[nodiscard]] auto match_server_name(std::string_view sni,
                                     const memory::vector<memory::string> &server_names) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| sni | std::string_view | 客户端 ClientHello 中的 SNI 值 |
| server_names | const memory::vector<memory::string>& | 配置中允许的 SNI 列表 |

### 返回值

`bool` — 匹配返回 `true`，不匹配返回 `false`

### 调用（向下）

- 无（纯比较函数）

### 被调用（向上）

- authenticate() — 认证流程中检查 SNI

### 知识域

- [[ref/protocol/tls-extensions|TLS 扩展]] — SNI 扩展

---

## 函数: match_short_id()

> 源码: `include/prism/stealth/reality/auth.hpp:67`
> 实现: `src/prism/stealth/reality/auth.cpp:26`

### 功能

匹配 short_id。遍历 `allowed_short_ids` 检查是否有匹配项，空字符串表示接受任意 short_id。将 hex 编码的配置值转换为字节后进行前缀匹配（客户端 short_id 长度 >= 配置值长度时比较前缀）。

### 签名

```cpp
[[nodiscard]] auto match_short_id(std::span<const std::uint8_t> short_id,
                                  const memory::vector<memory::string> &allowed_short_ids) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| short_id | std::span<const std::uint8_t> | 客户端发送的 short_id 字节 |
| allowed_short_ids | const memory::vector<memory::string>& | 配置中允许的 short_id 列表（hex 编码） |

### 返回值

`bool` — 匹配返回 `true`，不匹配返回 `false`

### 调用（向下）

- hex_to_bytes() — 将 hex 编码转换为字节

### 被调用（向上）

- authenticate() — 认证流程中检查 short_id

### 知识域

- [[stealth/reality/config|config::short_ids]] — 配置中的 short_id 列表

---

## 函数: hex_to_bytes()

> 源码: `include/prism/stealth/reality/auth.hpp:76`
> 实现: `src/prism/stealth/reality/auth.cpp:189`

### 功能

十六进制字符串转字节。将 hex 编码的字符串解码为原始字节序列。每两个字符转换为一个字节，奇数长度或非法字符时返回空向量。

### 签名

```cpp
[[nodiscard]] auto hex_to_bytes(std::string_view hex) -> memory::vector<std::uint8_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| hex | std::string_view | 十六进制字符串 |

### 返回值

`memory::vector<std::uint8_t>` — 解码后的字节向量，解码失败返回空向量

### 调用（向下）

- hex_digit() — 单个字符转换

### 被调用（向上）

- match_short_id() — 匹配 short_id 时转换配置值

### 知识域

- 编码 — 十六进制编解码

---

## 函数: hex_digit()

> 源码: `include/prism/stealth/reality/auth.hpp:84`
> 实现: `src/prism/stealth/reality/auth.cpp:208`

### 功能

单个十六进制字符转数值。将 0-9/a-f/A-F 字符转换为对应的数值 0-15，非法字符返回 -1。

### 签名

```cpp
[[nodiscard]] auto hex_digit(char c) -> int;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| c | char | 十六进制字符 |

### 返回值

`int` — 对应的数值（0-15），非法字符返回 -1

### 调用（向下）

- 无（纯函数）

### 被调用（向上）

- hex_to_bytes() — 十六进制字符串转字节

### 知识域

- 编码 — 十六进制字符映射
