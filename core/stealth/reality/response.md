---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/response.hpp
---

# Reality Response

TLS 1.3 ServerHello 生成器。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/response.hpp`

## 生成结果

### server_hello_result

```cpp
struct server_hello_result
{
    memory::vector<std::uint8_t> server_hello_msg;              // ServerHello 消息
    memory::vector<std::uint8_t> server_hello_record;           // ServerHello TLS 记录
    memory::vector<std::uint8_t> change_cipher_spec_record;     // CCS 兼容性记录
    memory::vector<std::uint8_t> encrypted_handshake_record;    // 加密握手记录
    memory::vector<std::uint8_t> encrypted_handshake_plaintext; // 加密前明文
};
```

| 字段 | 说明 |
|------|------|
| `server_hello_msg` | ServerHello 握手消息（含 handshake header） |
| `server_hello_record` | ServerHello TLS 记录（含 record header） |
| `change_cipher_spec_record` | ChangeCipherSpec 兼容性记录 |
| `encrypted_handshake_record` | 加密后的握手记录 |
| `encrypted_handshake_plaintext` | 加密前的握手消息明文 |

## 函数接口

### generate_server_hello

```cpp
[[nodiscard]] auto generate_server_hello(
    const client_hello_info &client_hello,
    std::span<const std::uint8_t> server_ephemeral_public,
    const key_material &handshake_keys,
    std::span<const std::uint8_t> dest_certificate,
    std::span<const std::uint8_t> client_hello_msg,
    std::span<const std::uint8_t> auth_key = {})
    -> std::pair<fault::code, server_hello_result>
```

生成完整的 TLS 握手响应。

#### 参数说明

| 参数 | 用途 |
|------|------|
| `client_hello` | 客户端信息 |
| `server_ephemeral_public` | 服务端临时公钥（key_share） |
| `handshake_keys` | 握手密钥（用于加密） |
| `dest_certificate` | 目标网站证书（DER） |
| `client_hello_msg` | 完整 ClientHello 消息 |
| `auth_key` | 认证密钥（用于 Ed25519 签名） |

### make_tls_record

```cpp
[[nodiscard]] auto make_tls_record(std::uint8_t content_type,
                                   std::span<const std::uint8_t> payload)
    -> memory::vector<std::uint8_t>
```

构造 TLS 记录格式：

```
[ContentType: 1B][Version: 2B][Length: 2B][Payload]
```

### encrypt_tls_record

```cpp
[[nodiscard]] auto encrypt_tls_record(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> iv,
    std::uint64_t sequence,
    std::uint8_t content_type,
    std::span<const std::uint8_t> plaintext)
    -> std::pair<fault::code, memory::vector<std::uint8_t>>
```

加密 TLS 1.3 记录。

#### 加密流程

```
nonce = IV XOR sequence
AAD = record_header
ciphertext = AES-128-GCM(key, nonce, plaintext + content_type, AAD)
output = record_header + ciphertext + auth_tag
```

## ServerHello 结构

```
ServerHello (handshake header + body):
[Type: 1B][Length: 3B][Version: 2B][Random: 32B][SessionID: 1+len][CipherSuite: 2B][CompressionMethod: 1B][Extensions: 2+list]

Extensions:
- key_share (0x0033): X25519 公钥
- supported_versions (0x002B): TLS 1.3 (0x0304)
```

## 加密握手消息内容

`encrypted_handshake_record` 包含：

```
EncryptedExtensions
Certificate
CertificateVerify
Finished
```

### 伪造证书

Reality 使用 `auth_key` 生成 Ed25519 签名的合成证书：

- 证书内容使用目标网站的真实证书
- 签名使用 Reality 派生的 auth_key
- 避免 CA 验证问题

## 调用链

- [[handshake]] ← 调用生成
- [[request]] ← 客户端信息
- [[keygen]] ← 密钥材料
- [[constants]] ← TLS 常量
- [[crypto-aead]] ← 加密
- [[crypto-x25519]] ← Ed25519 签名