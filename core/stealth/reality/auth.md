---
layer: core
source: include/prism/stealth/reality/auth.hpp
title: Reality Auth
tags:
  - stealth
  - reality
  - authentication
  - X25519
---

# Reality Auth

Reality 协议的客户端认证逻辑。

## 源码位置

- 头文件：`include/prism/stealth/reality/auth.hpp`

## 设计决策（WHY）

### 为什么认证分五步而非一步

Reality 认证的五步流程（SNI 匹配 → X25519 检查 → 密钥交换 → short_id 验证 → auth_key 派生）是**逐步提高成本**的策略。SNI 匹配是纯字符串比较（零成本），可以快速排除不相关的 ClientHello。只有 SNI 匹配后才会执行 X25519 密钥交换（有成本）。如果 X25519 失败，不会浪费 HKDF 计算资源。

### 为什么使用两层密钥交换

1. **静态密钥交换**（配置的 `private_key` + 客户端公钥）→ 计算认证用的 `shared_secret`
2. **临时密钥交换**（服务端生成的临时密钥对 + 客户端公钥）→ 计算 TLS ECDH 的 `shared_secret`

分离的原因是：认证密钥只需要验证客户端身份，而 TLS 握手需要前向安全性（ephemeral key）。如果只用静态密钥，一旦私钥泄露，所有历史流量都可解密。

### 为什么 `short_ids` 允许空字符串

空字符串 `""` 在 `match_short_id` 中表示"接受任意 short_id"。这是 Reality 的宽松模式——某些部署场景下不想配置 short_id，只需 SNI 匹配和密钥交换就足够了。代价是安全性降低：知道 SNI 和服务器公钥的攻击者可以构造合法的 ClientHello。

### 为什么 `auth_key` 使用 HKDF 而非直接使用 shared_secret

直接使用 X25519 的 `shared_secret` 作为认证密钥是危险的——如果直接用作 AES-GCM 密钥，X25519 输出的某些特殊值可能导致弱密钥。HKDF 将 `shared_secret` 和 `transcript_hash` 混合后提取伪随机密钥，确保密钥质量。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `private_key` 必须是有效的 X25519 私钥 | 密钥交换 | 无效私钥导致 X25519 输出全零或错误 |
| `client_public_key` 必须是 32 字节 | X25519 格式 | 从 ClientHello key_share 提取 |
| `short_ids` hex 编码 | 解析器 | 非 hex 字符导致解码失败 |
| 认证是同步操作 | 函数签名 | 不涉及网络 I/O，可以在 sniff/verify 阶段调用 |

## 失败场景

| 场景 | 触发条件 | 返回错误码 |
|------|----------|-----------|
| SNI 不匹配 | server_name 不在 server_names 中 | `reality_sni_mismatch` |
| 无 X25519 公钥 | ClientHello 无 key_share 扩展 | 认证失败 |
| X25519 密钥交换失败 | 公钥为特殊值 | 认证失败 |
| short_id 不匹配 | session_id 中的 short_id 不在白名单 | 认证失败 |
| HKDF 派生失败 | BoringSSL 内部错误 | 认证失败 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `auth` → `config` | 依赖 | 使用 `server_names`、`private_key`、`short_ids` |
| `auth` → `crypto::x25519` | 调用 | 静态密钥交换 |
| `auth` → `crypto::hkdf` | 调用 | auth_key 派生 |
| `auth` → `request` | 依赖 | 使用 `client_hello_info` 的 session_id、server_name、client_public_key |
| `handshake` → `auth` | 调用 | Stage 2 执行认证 |

## 认证流程

### authenticate

```cpp
[[nodiscard]] auto authenticate(const config &cfg, const client_hello_info &client_hello,
                                std::span<const std::uint8_t> decoded_private_key)
    -> std::pair<fault::code, auth_result>
```

执行 Reality 认证，返回认证结果。

#### 认证步骤

1. **SNI 匹配**：检查 server_name 是否在 server_names 列表中
2. **X25519 检查**：验证 key_share 扩展中包含 X25519 公钥
3. **密钥交换**：使用静态私钥计算共享密钥
4. **short_id 验证**：检查 session_id 中的 short_id
5. **派生 auth_key**：使用 HKDF 派生认证密钥

## 认证结果

### auth_result

```cpp
struct auth_result
{
    bool authenticated = false;
    std::array<std::uint8_t, 32> shared_secret{};
    crypto::x25519_keypair server_ephemeral_key;
    std::array<std::uint8_t, 32> auth_key{};
};
```

| 字段 | 说明 |
|------|------|
| `authenticated` | 认证是否成功 |
| `shared_secret` | X25519 共享密钥（用于 TLS ECDH） |
| `server_ephemeral_key` | 服务端临时密钥对（用于 ServerHello） |
| `auth_key` | HKDF 派生的认证密钥（用于 Ed25519 签名） |

## 辅助函数

### match_server_name

```cpp
[[nodiscard]] auto match_server_name(std::string_view sni,
                                     const memory::vector<memory::string> &server_names) -> bool
```

检查 SNI 是否匹配 server_names 列表。

### match_short_id

```cpp
[[nodiscard]] auto match_short_id(std::span<const std::uint8_t> short_id,
                                  const memory::vector<memory::string> &allowed_short_ids) -> bool
```

检查 short_id 是否匹配。

- 空字符串表示接受任意
- hex 编码比较

### hex_to_bytes

```cpp
[[nodiscard]] auto hex_to_bytes(std::string_view hex) -> memory::vector<std::uint8_t>
```

将 hex 编码字符串解码为字节序列。

### hex_digit

```cpp
[[nodiscard]] auto hex_digit(char c) -> int
```

将单个 hex 字符转换为数值。

## 密钥交换细节

Reality 使用两层密钥交换：

1. **静态密钥交换**：使用配置的 private_key 和客户端公钥
2. **临时密钥交换**：服务端生成临时密钥对用于 ServerHello

认证流程：
```
client_public_key (from ClientHello key_share)
+ server_private_key (from config)
→ shared_secret

shared_secret + transcript_hash
→ auth_key (via HKDF)
```

## 调用链

- [[config]] ← 配置结构体
- [[request]] ← ClientHello 解析结果
- [[constants]] ← REALITY_KEY_LEN
- [[core/crypto/x25519|crypto-x25519]] ← 密钥交换
- [[core/crypto/hkdf|crypto-hkdf]] ← 密钥派生
- [[handshake]] ← 调用认证