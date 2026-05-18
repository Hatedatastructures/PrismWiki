---
title: "TLS 加密"
category: "crypto"
type: ref
layer: ref
module: ref
source: "RFC 8446"
tags: [密码学, tls, 加密, 密钥调度, 记录层]
created: 2026-05-17
updated: 2026-05-17
---

# TLS 加密

**类别**: 密码学

## 概述

TLS 1.3 的加密层建立在 AEAD 算法和 HKDF 密钥派生之上。本文解析 TLS 1.3 的密钥调度、记录层加密和密钥更新机制。

### TLS 1.3 加密特性

TLS 1.3 加密的核心特性：

| 特性 | 说明 |
|------|------|
| **AEAD 强制** | 所有记录层加密使用 AEAD |
| **握手加密** | ClientHello/ServerHello 后所有消息加密 |
| **密钥分离** | 握手和应用阶段使用不同密钥 |
| **密钥更新** | 支持运行时密钥轮换 |
| **0-RTT 加密** | 使用 PSK 的早期数据加密 |

### 密码套件

TLS 1.3 定义的密码套件：

| 密码套件 | AEAD 算法 | 哈希算法 |
|----------|-----------|----------|
| TLS_AES_128_GCM_SHA256 | AES-128-GCM | SHA-256 |
| TLS_AES_256_GCM_SHA384 | AES-256-GCM | SHA-384 |
| TLS_CHACHA20_POLY1305_SHA256 | ChaCha20-Poly1305 | SHA-256 |
| TLS_AES_128_CCM_SHA256 | AES-128-CCM | SHA-256 |
| TLS_AES_128_CCM_8_SHA256 | AES-128-CCM (8B tag) | SHA-256 |

## 密钥调度

### 密钥调度流程

TLS 1.3 密钥调度的完整流程：

```
TLS 1.3 密钥调度：

ECDHE 共享密钥 (32B)
       │
       ▼ HKDF-Extract(0, shared)
early_secret (32B)
       │
       ├──── HKDF-Expand-Label(early_secret, "binder", "", 32)
       │     → binder_key (PSK 验证)
       │
       ├──── HKDF-Expand-Label(early_secret, "res binder", "", 32)
       │     → resumption_binder_key
       │
       ▼ HKDF-Expand-Label(early_secret, "derived", "", 32)
derived_secret (32B)
       │
       ▼ HKDF-Extract(derived_secret, handshake_transcript_hash)
handshake_secret (32B)
       │
       ├──── HKDF-Expand-Label(handshake_secret, "c hs traffic", hash, 32+12)
       │     → client_handshake_traffic_secret
       │         → client_handshake_key (16/32B)
       │         → client_handshake_iv (12B)
       │
       ├──── HKDF-Expand-Label(handshake_secret, "s hs traffic", hash, 32+12)
       │     → server_handshake_traffic_secret
       │         → server_handshake_key (16/32B)
       │         → server_handshake_iv (12B)
       │
       ▼ HKDF-Expand-Label(handshake_secret, "derived", "", 32)
derived_secret (32B)
       │
       ▼ HKDF-Extract(derived_secret, application_transcript_hash)
master_secret (32B)
       │
       ├──── HKDF-Expand-Label(master_secret, "c ap traffic", hash, 32+12)
       │     → client_application_traffic_secret
       │         → client_application_key (16/32B)
       │         → client_application_iv (12B)
       │
       ├──── HKDF-Expand-Label(master_secret, "s ap traffic", hash, 32+12)
       │     → server_application_traffic_secret
       │         → server_application_key (16/32B)
       │         → server_application_iv (12B)
       │
       └─ HKDF-Expand-Label(master_secret, "res master", hash, 32)
            → resumption_master_secret (PSK 恢复)
```

### 密钥派生细节

从 traffic_secret 派生 AEAD 密钥：

```cpp
// 从 traffic_secret 派生 key 和 iv

// AEAD 密钥
key = HKDF-Expand-Label(traffic_secret, "key", "", key_length)

// AEAD IV
iv = HKDF-Expand-Label(traffic_secret, "iv", "", 12)

// 参数：
// traffic_secret: 32 字节（SHA-256 输出）
// key_length: 16 (AES-128) 或 32 (AES-256/ChaCha20)
// iv_length: 12 字节
```

### Transcript Hash

Transcript Hash 绑定密钥到具体握手：

```
Transcript Hash：

握手 transcript：
- ClientHello (不含 binders)
- ServerHello
- EncryptedExtensions
- CertificateRequest (如果存在)
- Certificate
- CertificateVerify
- Finished

计算：
hash = SHA-256(transcript)

作用：
- 绑定密钥到具体握手内容
- 防止握手消息被篡改
- 密钥派生的 context 参数
```

## 记录层加密

### TLS 1.3 记录格式

TLS 1.3 加密记录的格式：

```
TLS 1.3 记录结构：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│  ContentType  │  LegacyVersion │           Length             │
│   (1 byte)    │   (2 bytes)     │          (2 bytes)          │
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│                                                               │
│                    Encrypted Data                             │
│                                                               │
│              (AEAD ciphertext + auth tag)                     │
│                                                               │
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

LegacyVersion: 固定为 0x0303 (TLS 1.2 legacy)
Length: AEAD 输出长度（密文 + 16B tag）
ContentType: 被加密的实际类型（如 application_data=23）
```

### AEAD 参数构造

TLS 1.3 AEAD 加密的参数：

```cpp
// TLS 1.3 AEAD 参数

// Nonce 构造
nonce = iv XOR sequence_number

// sequence_number: 64 位计数器，从 0 开始
// iv: 12 字节（HKDF 派生）

// AAD 构造
AAD = sequence_number || 
      ContentType || 
      LegacyVersion (0x0303) || 
      Length

// 注意：AAD 中的 ContentType 是实际类型
//      外部记录头的 ContentType 固定为 23 (application_data)
```

### 加密过程

TLS 1.3 记录加密过程：

```cpp
// TLS 1.3 记录加密

// 输入
plaintext = ContentType || data
key = traffic_key
iv = traffic_iv
seq = sequence_number

// 构造参数
nonce = iv XOR pad64(seq)  // 64 位扩展到 96 位
aad = seq || record_header  // 13 字节

// AEAD 加密
ciphertext, tag = AEAD.seal(key, nonce, plaintext, aad)

// 输出
record = TLSRecord {
    type: 23 (application_data),
    version: 0x0303,
    length: len(ciphertext) + 16,
    data: ciphertext || tag
}
```

### 解密过程

TLS 1.3 记录解密过程：

```cpp
// TLS 1.3 记录解密

// 输入
record = TLSRecord {...}
key = traffic_key
iv = traffic_iv
seq = sequence_number

// 构造参数
nonce = iv XOR pad64(seq)
aad = seq || record_header

// AEAD 解密
plaintext = AEAD.open(key, nonce, ciphertext, tag, aad)

// 解析
actual_type = plaintext[0]
data = plaintext[1:]

// 更新序列号
seq += 1
```

## 密钥更新

### KeyUpdate 消息

TLS 1.3 支持运行时密钥更新：

```
KeyUpdate 消息流程：

发送方：
1. 发送 KeyUpdate 消息
2. 计算新 traffic_secret
   new_secret = HKDF-Expand-Label(old_secret, "update", "", 32)
3. 派生新 key/iv
4. 使用新密钥加密后续数据

接收方：
1. 接收 KeyUpdate 消息
2. 计算新 traffic_secret
3. 派生新 key/iv
4. 使用新密钥解密后续数据
```

### KeyUpdate 消息格式

```
KeyUpdate 消息结构：

struct {
    KeyUpdateRequest request_update;
} KeyUpdate;

enum {
    update_not_requested(0),
    update_requested(1),
    (255)
} KeyUpdateRequest;

update_not_requested: 单向更新
update_requested: 要求对方也更新
```

### 密钥更新机制

```cpp
// 密钥更新机制

// 发起更新
void send_key_update() {
    // 发送 KeyUpdate 消息
    send(KeyUpdate{update_not_requested});
    
    // 计算新密钥
    new_traffic_secret = HKDF-Expand-Label(
        current_traffic_secret, "update", "", 32
    );
    new_key = HKDF-Expand-Label(new_traffic_secret, "key", "", key_len);
    new_iv = HKDF-Expand-Label(new_traffic_secret, "iv", "", 12);
    
    // 切换密钥
    current_key = new_key;
    current_iv = new_iv;
    sequence_number = 0;  // 重置序列号
}

// 接收更新
void on_key_update(KeyUpdate msg) {
    // 计算新密钥
    ...
    
    // 如果对方请求更新，也发送 KeyUpdate
    if (msg.request_update == update_requested) {
        send(KeyUpdate{update_not_requested});
        ...
    }
}
```

## 0-RTT 加密

### 0-RTT 密钥

使用 PSK 的 0-RTT 加密：

```
0-RTT 密钥派生：

PSK (resumption PSK)
       │
       ▼ HKDF-Extract(0, PSK)
early_secret
       │
       ├──── HKDF-Expand-Label(early_secret, "binder", "", 32)
       │     → binder_key
       │
       ▼ HKDF-Expand-Label(early_secret, "ext binder", "", 32)
external_bind_key  (外部 PSK)
       │
       ▼ HKDF-Extract(derived_secret, ECDHE_shared)
handshake_secret
       │
       ▼ HKDF-Expand-Label(handshake_secret, "c e traffic", hash, 32)
client_early_traffic_secret
       │
       └─ HKDF-Expand-Label(client_early_traffic_secret, "key", "", key_len)
          → client_early_key

0-RTT 数据使用 client_early_key 加密
```

### 0-RTT 安全限制

```
0-RTT 安全限制：

限制 1：无前向保密
- 0-RTT 密钥派生自 PSK
- PSK 泄露 → 0-RTT 数据可解密

限制 2：可重放
- 0-RTT 数据可被攻击者重放
- 服务端需实现重放检测

限制 3：有限数据
- 不应发送敏感数据
- 不应发送状态改变操作

建议：
- 仅发送幂等请求
- 使用服务端提供的 max_early_data_size
- 服务端实现重放缓存
```

## 在 Prism 中的应用

### Reality TLS 加密

Reality 使用 TLS 1.3 加密机制：

```
Reality TLS 1.3 加密：

握手阶段：
- 使用 X25519 密钥交换
- 使用 HKDF 密钥派生
- 使用 AES-GCM 加密握手消息

记录层加密：
- 标准 TLS 1.3 AEAD 加密
- AAD 包含序列号和记录头
- Nonce = iv XOR sequence_number

密钥更新：
- 支持标准 TLS 1.3 密钥更新
- 可扩展连接时长
```

详见 [[core/stealth/reality/handshake|Reality]]。

## 参见

- [[ref/protocol/tls-handshake|TLS 握手流程]] — TLS 协议详解
- [[ref/crypto/aead-basics|AEAD 原理]] — AEAD 基础
- [[ref/crypto/hkdf-theory|HKDF 理论]] — 密钥派生
- [[ref/crypto/aes-gcm|AES-GCM]] — AES-GCM 原理
- [[ref/protocol/tls-1.3|TLS 1.3 协议]] — TLS 1.3 完整规范
- [[core/crypto/hkdf|hkdf]] — HKDF 实现
- [[core/crypto/aead|aead]] — AEAD 实现