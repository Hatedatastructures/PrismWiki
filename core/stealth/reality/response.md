---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/response.hpp
title: Reality Response
tags:
  - stealth
  - reality
  - ServerHello
  - TLS-record
---

# Reality Response

TLS 1.3 ServerHello 生成器。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/response.hpp`

## 设计决策（WHY）

### 为什么需要 `change_cipher_spec_record`

TLS 1.3 为了兼容中间件，规定在握手初期发送一个 CCS（ChangeCipherSpec）记录。这个记录在 TLS 1.3 语义上被忽略，但某些旧版中间件如果看不到 CCS 会断开连接。Reality 必须模拟完整的 TLS 1.3 行为，因此 CCS 记录不可省略。

### 为什么使用合成 Ed25519 证书

Reality 不拥有 dest 网站的私钥，无法生成有效的 CA 签名证书。使用 `auth_key` 生成的 Ed25519 自签名证书有两个好处：
1. 不需要去 dest 服务器获取证书（省去网络延迟）
2. 客户端通过 Reality 协议已经验证了服务器身份，不需要 CA 验证

客户端的 Reality 实现知道证书是合成的，会跳过 CA 验证，直接使用 `auth_key` 确认服务器身份。

### 为什么 `server_hello_result` 同时保存明文和密文

`encrypted_handshake_plaintext` 和 `encrypted_handshake_record` 分别保存加密前和加密后的握手记录。明文被保留是因为后续的 `derive_and_encrypt_finished` 需要提取 EE + Cert + CertVerify 部分来计算 transcript hash，然后重算 Finished。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| ServerHello 必须包含 key_share (X25519) 和 supported_versions (TLS 1.3) | TLS 1.3 规范 | 缺少扩展导致客户端拒绝 |
| AES-128-GCM 密钥 16 字节、IV 12 字节、Tag 16 字节 | TLS 1.3 密码套件 | 固定不可变 |
| 加密序列号从 0 开始 | TLS 1.3 规范 | 第一个加密记录 sequence=0 |
| 合成证书有效期仅 1 小时 | 实现约定 | 防止长期缓存 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| AES-128-GCM 加密失败 | BoringSSL 错误 | 返回错误，握手终止 |
| ServerHello 构造越界 | 缓冲区不足 | 未定义行为（应被长度检查捕获） |
| Ed25519 签名失败 | auth_key 无效 | 证书无效，客户端拒绝 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `response` → `crypto::aead` | 调用 | `encrypt_tls_record` 使用 AEAD 加密 |
| `response` → `keygen` | 依赖 | 接收 `key_material` 进行加密 |
| `response` → `request` | 依赖 | 使用 `client_hello_info` 构造 ServerHello |
| `handshake` → `response` | 调用 | Stage 3/4 生成握手响应 |
| `seal` ← `response` | 被依赖 | seal 使用 `key_material` 中的应用密钥 |

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
- [[core/crypto/aead|crypto-aead]] ← 加密
- [[core/crypto/x25519|crypto-x25519]] ← Ed25519 签名