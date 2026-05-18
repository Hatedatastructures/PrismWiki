---
title: "TLS 握手流程"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 8446"
tags: [协议, tls, 握手, clienthello, serverhello, key_share]
created: 2026-05-17
updated: 2026-05-17
---

# TLS 握手流程

**类别**: 协议

## 概述

TLS 1.3 握手流程相比 TLS 1.2 大幅简化，从 2-RTT 减少到 1-RTT，并在握手过程中加密大部分消息。本文详细解析 TLS 1.3 的完整握手流程、消息格式和密钥派生。

### 握手流程对比

TLS 1.3 与 TLS 1.2 的握手对比：

```
TLS 1.2 握手（2-RTT）：

Client                                    Server
  |---------- ClientHello ------------------>| (1)
  |<--------- ServerHello -------------------| (2)
  |<--------- Certificate -------------------|
  |<--------- ServerHelloDone ---------------|
  |---------- ClientKeyExchange ------------>| (3)
  |---------- ChangeCipherSpec ------------->|
  |---------- Finished ---------------------->| (4)
  |<--------- ChangeCipherSpec --------------|
  |<--------- Finished ----------------------|
  |                                          |
  |<========= Application Data ==============>|

TLS 1.3 握手（1-RTT）：

Client                                    Server
  |---------- ClientHello ------------------>| (1)
  |          (key_share)                    |
  |                                          |
  |<--------- ServerHello -------------------| (2)
  |          (key_share)                    | ← 握手密钥已计算
  |<--------- EncryptedExtensions -----------| ← 加密开始
  |<--------- Certificate -------------------|
  |<--------- CertificateVerify -------------|
  |<--------- Finished ----------------------|
  |                                          | ← 应用密钥已计算
  |---------- Finished --------------------->| (3)
  |                                          |
  |<========= Application Data ==============>|

RTT 节省：TLS 1.3 比 TLS 1.2 少一个 RTT
```

## 1-RTT 握手

### 完整流程

TLS 1.3 标准 1-RTT 握手：

```
TLS 1.3 1-RTT 握手：

Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |                                          |
  |  内容：                                   |
  |  - supported_versions: 0x0304            |
  |  - supported_groups: x25519              |
  |  - key_share: x25519 公钥 (32B)          |
  |  - signature_algorithms                  |
  |  - cipher_suites                         |
  |                                          |
  |<--------- ServerHello -------------------|
  |                                          |
  |  内容：                                   |
  |  - supported_version: 0x0304             |
  |  - key_share: x25519 公钥 (32B)          |
  |  - cipher_suite: TLS_AES_128_GCM_SHA256  |
  |                                          |
  |  ← 此时双方计算 ECDHE 共享密钥            |
  |  ← HKDF 派生 handshake_secret            |
  |  ← 派生握手密钥                          |
  |                                          |
  |<--------- [握手消息加密] -----------------|
  |                                          |
  |  EncryptedExtensions:                    |
  |  - server_name (如果提供)                 |
  |  - alpn_protocol (如果协商)              |
  |                                          |
  |  Certificate:                            |
  |  - 服务器证书链                          |
  |                                          |
  |  CertificateVerify:                      |
  |  - 证书签名验证                          |
  |                                          |
  |  Finished:                               |
  |  - 握手完成验证                          |
  |                                          |
  |  ← 此时计算 application_secret           |
  |                                          |
  |---------- [握手密钥加密] -----------------|
  |                                          |
  |  Finished:                               |
  |  - 客户端握手完成验证                    |
  |                                          |
  |<========= Application Data ==============>|
```

### ClientHello 格式

ClientHello 消息结构：

```
ClientHello 结构（TLS 1.3）：

struct {
    ProtocolVersion legacy_version = 0x0303;  // TLS 1.2 legacy
    Random random;                            // 32 字节随机数
    opaque legacy_session_id<0..32>;          // 兼容性 session ID
    CipherSuite cipher_suites<2..65534>;      // 密码套件列表
    opaque legacy_compression_methods<1..2>;  // 压缩（无）
    Extension extensions<0..2^16-1>;          // 扩展列表
} ClientHello;

关键扩展：
- supported_versions: [0x0304]  // TLS 1.3
- supported_groups: [x25519, secp256r1]
- key_share: [(x25519, <公钥>)]
- signature_algorithms: [rsa_pss_rsae_sha256, ecdsa_secp256r1_sha256]
- server_name: <域名>
- alpn: [h2, http/1.1]
```

详见 [[ref/protocol/tls-clienthello|TLS ClientHello]]。

### ServerHello 格式

ServerHello 消息结构：

```
ServerHello 结构（TLS 1.3）：

struct {
    ProtocolVersion legacy_version = 0x0303;  // TLS 1.2 legacy
    Random random;                            // 32 字节随机数
    opaque legacy_session_id_echo<0..32>;     // 回显 session ID
    CipherSuite cipher_suite;                 // 选定的密码套件
    opaque legacy_compression_method = 0;     // 无压缩
    Extension extensions<0..2^16-1>;          // 扩展列表
} ServerHello;

关键扩展：
- supported_version: 0x0304  // 确认 TLS 1.3
- key_share: (x25519, <公钥>)

随机数特殊值：
- HelloRetryRequest: random = SHA-256("HelloRetryRequest")
- 正常 ServerHello: 随机生成
```

详见 [[ref/protocol/tls-serverhello|TLS ServerHello]]。

### EncryptedExtensions

EncryptedExtensions 消息：

```
EncryptedExtensions 结构：

struct {
    Extension extensions<0..2^16-1>;
} EncryptedExtensions;

常见扩展：
- server_name: 确认 SNI
- alpn: 协议协商结果
- max_early_data_size: 0-RTT 数据限制
- supported_groups: 可选补充

注意：
- 从此消息开始加密
- 不包含安全敏感信息
```

### Certificate

Certificate 消息：

```
Certificate 结构（TLS 1.3）：

struct {
    opaque certificate_request_context<0..255>;  // 空（非请求）
    CertificateEntry certificate_list<0..2^24-1>;
} Certificate;

CertificateEntry {
    opaque cert_data<1..2^24-1>;      // DER 编码证书
    Extension extensions<0..2^16-1>; // 证书扩展
}

变化：
- 不使用证书链压缩（原 CertificateChain）
- 每个证书单独扩展
- context 字段用于请求响应匹配
```

### CertificateVerify

CertificateVerify 消息：

```
CertificateVerify 结构：

struct {
    SignatureScheme algorithm;
    opaque signature<0..2^16-1>;
} CertificateVerify;

签名数据：
transcript-hash = SHA-256(ClientHello..Certificate)
signature_input = "TLS 1.3, server CertificateVerify" || 
                  0x00 0x00 0x00 || 
                  transcript-hash

签名算法：
- rsa_pss_rsae_sha256: RSA-PSS 签名
- ecdsa_secp256r1_sha256: ECDSA 签名
- ed25519: Ed25519 签名
```

### Finished

Finished 消息：

```
Finished 结构：

struct {
    opaque verify_data<0..255>;
} Finished;

verify_data 计算：
finished_key = HKDF-Expand-Label(secret, "finished", "", 32)
verify_data = HMAC(finished_key, transcript-hash)

客户端 Finished：
- secret = client_handshake_traffic_secret
- transcript = ClientHello..CertificateVerify

服务端 Finished：
- secret = server_handshake_traffic_secret
- transcript = ClientHello..server Finished 之前的所有消息
```

## 密钥派生

### 密钥计算流程

握手过程中的密钥计算：

```
密钥计算时间点：

1. ServerHello 接收后：
   - 计算共享密钥 = X25519(client_private, server_public)
   - handshake_secret = HKDF-Extract(derived_secret, shared)
   - 派生 client/server handshake key/iv
   - 开始解密服务端握手消息

2. 服务端 Finished 接收后：
   - 验证 server Finished
   - master_secret = HKDF-Extract(derived_secret, app_transcript)
   - 派生 client/server application key/iv
   - 切换到应用密钥

3. 客户端 Finished 发送后：
   - 使用应用密钥加密后续数据
```

详见 [[ref/crypto/tls-crypto|TLS 加密]]。

## HelloRetryRequest

### 触发条件

HelloRetryRequest 的触发：

```
HelloRetryRequest 触发条件：

条件 1：key_share 不匹配
- 客户端未提供服务端支持的 group
- 服务端要求客户端重新提供

条件 2：Cookie 验证
- 服务端需要 cookie 验证客户端
- 防止 DoS 攻击

流程：
Client                                    Server
  |---------- ClientHello ------------------>|
  |          (key_share: secp256r1)          |
  |                                          |
  |<--------- HelloRetryRequest -------------|
  |          (selected_group: x25519)        |
  |                                          |
  |---------- ClientHello ------------------>|
  |          (key_share: x25519)             |
  |                                          |
  |<--------- ServerHello -------------------|
  |          ...                             |
```

### HelloRetryRequest 格式

```
HelloRetryRequest 结构：

struct {
    ProtocolVersion legacy_version = 0x0303;
    Random random = SHA-256("HelloRetryRequest");  // 特殊值
    opaque legacy_session_id_echo<0..32>;
    CipherSuite cipher_suite;
    opaque legacy_compression_method = 0;
    Extension extensions<0..2^16-1>;
} HelloRetryRequest;

关键扩展：
- supported_version: 0x0304
- key_share: (selected_group, empty) 或 cookie
```

## 0-RTT 握手

### PSK 握手

使用 PSK 的 0-RTT 握手：

```
0-RTT 握手流程：

Client                                    Server
  |---------- ClientHello ------------------>|
  |          (pre_shared_key)                |
  |          (key_share)                     |
  |---------- Early Data ------------------->| ← 0-RTT 数据
  |---------- EndOfEarlyData --------------->|
  |                                          |
  |<--------- ServerHello -------------------|
  |          (pre_shared_key)                |
  |<--------- EncryptedExtensions -----------|
  |<--------- Finished ----------------------|
  |                                          |
  |---------- Finished --------------------->|
  |                                          |
  |<========= Application Data ==============>|

0-RTT 数据使用 PSK 派生的 early_key 加密
```

### PSK 扩展格式

```
pre_shared_key 扩展结构：

struct {
    opaque identity<0..2^16-1>;  // PSK identity
    uint32 obfuscated_ticket_age;  // 时间（减去 age mask）
} PskIdentity;

struct {
    opaque binder<0..255>;  // HMAC 验证值
} PskBinderEntry;

struct {
    PskIdentity identities<7..2^16-1>;
    PskBinderEntry binders<33..2^16-1>;
} OfferedPsks;

binder 计算：
binder_key = HKDF-Expand-Label(early_secret, "res binder", "", 32)
binder = HMAC(binder_key, ClientHello 不含 binders 的 hash)
```

## 会话恢复

### PSK 生成

会话恢复 PSK 的生成：

```
PSK 生成（握手结束时）：

resumption_master_secret = HKDF-Expand-Label(
    master_secret, "res master", transcript_hash, 32
)

PSK = HKDF-Expand-Label(
    resumption_master_secret, "resumption", ticket_nonce, 32
)

Ticket 内容：
- PSK identity（ticket 或 session ID）
- PSK 本身（服务端存储）
- cipher_suite
- 创建时间
- max_early_data_size（如果支持 0-RTT）
```

## 在 Prism 中的应用

### Reality 握手

Reality 使用标准 TLS 1.3 握手：

```
Reality 握手流程：

1. 客户端发送 ClientHello
   - 包含 Reality 特定的 key_share
   - 包含服务端 publicKey 作为 "shortId"

2. 服务端检测 Reality 客户端
   - 验证 shortId 匹配
   - 使用 privateKey 计算 ECDHE

3. 服务端响应 ServerHello
   - 标准 TLS 1.3 格式
   - 继续完成握手

4. 后续加密通信
   - 标准 TLS 1.3 AEAD 加密
```

详见 [[core/stealth/reality/handshake|Reality]]。

## 参见

- [[ref/protocol/tls-1.3|TLS 1.3 协议]] — TLS 1.3 完整规范
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 解析
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 解析
- [[ref/protocol/tls-extensions|TLS 扩展]] — 扩展详解
- [[ref/crypto/tls-crypto|TLS 加密]] — TLS 1.3 加密
- [[ref/crypto/key-exchange|密钥交换原理]] — X25519 原理
- [[core/stealth/reality/handshake|Reality]] — Reality TLS 伪装