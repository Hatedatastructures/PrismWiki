---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/auth.hpp
title: Reality Auth
---

# Reality Auth

Reality 协议的客户端认证逻辑。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/auth.hpp`

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
- [[crypto-x25519]] ← 密钥交换
- [[crypto-hkdf]] ← 密钥派生
- [[handshake]] ← 调用认证