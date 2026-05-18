---
title: "HKDF 理论"
category: "crypto"
type: ref
layer: ref
module: ref
source: "RFC 5869, RFC 8446"
tags: [密码学, hkdf, 密钥派生, hmac, sha256]
created: 2026-05-17
updated: 2026-05-17
---

# HKDF 理论

**类别**: 密码学

## 概述

HKDF（HMAC-based Key Derivation Function）是基于 HMAC 的密钥派生函数，由 RFC 5869 定义。HKDF 将输入密钥材料（IKM）转换为符合密码学要求的派生密钥，是 TLS 1.3 密钥调度的核心算法。

### 为什么需要密钥派生

直接使用共享密钥加密存在问题：

```
共享密钥的直接使用问题：

问题 1：密钥长度不匹配
- X25519 输出 32 字节
- AES-128-GCM 需要 16 字节密钥
- AES-256-GCM 需要 32 字节密钥 + 12 字节 nonce

问题 2：密钥用途混淆
- 同一密钥用于加密、认证、IV
- 可能导致安全漏洞

问题 3：密钥分离
- 需要多个独立密钥（客户端/服务端）
- 不能简单分割共享密钥

HKDF 解决方案：
- 输入任意长度的密钥材料
- 输出符合要求的派生密钥
- 通过 info 参数实现密钥分离
```

### HKDF 结构

HKDF 由两步组成：

```
HKDF 结构：

Extract 步骤：
┌─────────────────────────────────────────────┐
│  输入：                                      │
│    - salt: 盐值（可空）                      │
│    - IKM: 输入密钥材料                       │
│                                             │
│  输出：                                      │
│    - PRK: 伪随机密钥（32 字节）              │
│                                             │
│  操作：PRK = HMAC-Hash(salt, IKM)           │
└─────────────────────────────────────────────┘

Expand 步骤：
┌─────────────────────────────────────────────┐
│  输入：                                      │
│    - PRK: Extract 输出                       │
│    - info: 上下文信息                        │
│    - L: 输出长度                             │
│                                             │
│  输出：                                      │
│    - OKM: 输出密钥材料                       │
│                                             │
│  操作：迭代 HMAC 生成所需长度                │
└─────────────────────────────────────────────┘

两步分离的意义：
- Extract: 将 IKM 转换为均匀分布的 PRK
- Expand: 从 PRK 派生多个独立密钥
```

## Extract 步骤

### HMAC 基础

HKDF 基于 HMAC（Hash-based MAC）：

```
HMAC 定义：
HMAC(K, text) = H((K ⊕ opad) || H((K ⊕ ipad) || text))

其中：
- H: 哈希函数（SHA-256）
- K: 密钥
- opad: 0x5c 重复的块
- ipad: 0x36 重复的块
- ||: 拼接
- ⊕: 异或

输出长度：
- HMAC-SHA256: 32 字节
- HMAC-SHA512: 64 字节
```

### Extract 操作

Extract 步骤的实现：

```cpp
// HKDF-Extract
HKDF-Extract(salt, IKM) → PRK

// 实现
PRK = HMAC-SHA256(salt, IKM)

// 参数：
// salt: 盐值，用于增强安全性
//       - 可为空（使用全零）
//       - 建议使用随机盐
// IKM: 输入密钥材料
//      - 可为任意长度
//      - 如 X25519 共享密钥（32 字节）

// 输出：
// PRK: 伪随机密钥
//      - 固定 32 字节（SHA-256 输出长度）
```

### Salt 的作用

Salt 的安全作用：

| Salt 值 | 安全性 | 说明 |
|---------|--------|------|
| 空（全零） | 较弱 | IKM 熵直接传递到 PRK |
| 固定值 | 中等 | 提供域分离 |
| 随机值 | 强 | 提供额外的熵 |

```
TLS 1.3 的 Salt 使用：

early_secret = HKDF-Extract(0, IKM)  // salt 为全零

handshake_secret = HKDF-Extract(
    derived_secret,  // 来自 early_secret 派生
    handshake_transcript_hash
)

master_secret = HKDF-Extract(
    derived_secret,  // 来自 handshake_secret 派生
    application_transcript_hash
)
```

## Expand 步骤

### Expand 操作

Expand 步骤的实现：

```cpp
// HKDF-Expand
HKDF-Expand(PRK, info, L) → OKM

// 实现（迭代）
N = ceil(L / HashLen)  // 需要的迭代次数
T(0) = empty
T(1) = HMAC(PRK, T(0) || info || 0x01)
T(2) = HMAC(PRK, T(1) || info || 0x02)
...
T(N) = HMAC(PRK, T(N-1) || info || N)

OKM = T(1) || T(2) || ... || T(N) 的前 L 字节

// 约束：
// L <= 255 × HashLen = 255 × 32 = 8160 字节
```

### Info 的作用

Info 参数实现密钥分离：

```
Info 的作用：

密钥分离：
HKDF-Expand(PRK, "client key", 16) → client_key
HKDF-Expand(PRK, "server key", 16) → server_key
HKDF-Expand(PRK, "client iv", 12) → client_iv
HKDF-Expand(PRK, "server iv", 12) → server_iv

不同 info → 不同输出（即使 PRK 相同）

域隔离：
- 不同用途使用不同 info
- 防止密钥混淆
- 防止跨用途攻击
```

### 输出长度限制

Expand 的输出长度限制：

```
输出长度限制：

最大长度：L = 255 × HashLen = 8160 字节

原因：
- 每次迭代输出 HashLen 字节
- 迭代计数器为单字节（0x01 到 0xff）
- 最多 255 次迭代

超出限制：
- 返回错误
- 不支持超长输出

TLS 1.3 典型输出：
- 密钥：16/32 字节
- IV：12 字节
- 均在限制内
```

## TLS 1.3 HKDF-Expand-Label

### 定义

TLS 1.3 定义了专用的 HKDF-Expand-Label：

```cpp
// TLS 1.3 HKDF-Expand-Label
HKDF-Expand-Label(Secret, Label, Context, Length) → Key

// 构造 HkdfLabel 结构
struct HkdfLabel {
    uint16 length;           // 输出长度（网络序）
    opaque label<7..255>;    // "tls13 " + Label
    opaque context<0..255>;  // Context（transcript hash）
};

// 实现
HkdfLabel = Length || "tls13 " + Label || Context
Key = HKDF-Expand(Secret, HkdfLabel, Length)
```

### 标签前缀

"tls13 " 前缀的意义：

```
标签前缀 "tls13"：

目的：
- 域隔离
- TLS 1.3 派生的密钥与其他协议不同
- 即使 Secret 相同，"tls13 " 前缀确保不同输出

示例：
Label = "c hs traffic"
完整 label = "tls13 c hs traffic"
输出 = client handshake traffic key

Label = "s ap traffic"
完整 label = "tls13 s ap traffic"
输出 = server application traffic key
```

### 在 TLS 1.3 中的应用

TLS 1.3 密钥派生使用 HKDF-Expand-Label：

```
TLS 1.3 密钥派生流程：

ECDHE 共享密钥
       │
       ▼ HKDF-Extract(0, shared)
early_secret
       │
       ▼ HKDF-Expand-Label(early_secret, "derived", "", 32)
derived_secret
       │
       ▼ HKDF-Extract(derived_secret, handshake_transcript)
handshake_secret
       │
       ├──── HKDF-Expand-Label(handshake_secret, "c hs traffic", hash, 32)
       │     → client_handshake_key/iv
       │
       ├──── HKDF-Expand-Label(handshake_secret, "s hs traffic", hash, 32)
       │     → server_handshake_key/iv
       │
       ▼ HKDF-Expand-Label(handshake_secret, "derived", "", 32)
derived_secret
       │
       ▼ HKDF-Extract(derived_secret, application_transcript)
master_secret
       │
       ├──── HKDF-Expand-Label(master_secret, "c ap traffic", hash, 32)
       │     → client_application_key/iv
       │
       ├──── HKDF-Expand-Label(master_secret, "s ap traffic", hash, 32)
       │     → server_application_key/iv
       │
       └─ HKDF-Expand-Label(master_secret, "res master", hash, 32)
            → resumption_master_secret
```

## 与 BLAKE3 的对比

### 对比

| 特性 | HKDF-SHA256 | BLAKE3 derive_key |
|------|-------------|-------------------|
| 步骤 | Extract + Expand 两步 | 单步 |
| 性能 | 中（~200 MB/s） | 快（~1 GB/s） |
| 输出长度 | 最多 8160 字节 | 任意长度 |
| 标准 | RFC 5869 | BLAKE3 规范 |
| 应用 | TLS 1.3 | SS2022 |

### SS2022 使用 BLAKE3

SS2022 选择 BLAKE3 的原因：

```
SS2022 密钥派生：

subkey = BLAKE3-derive_key(
    context = "shadowsocks 2022 session subkey",
    key_material = PSK || salt,
    length = 32
)

选择 BLAKE3 的原因：
1. 单步操作，无需 Extract + Expand
2. 性能更高
3. SIP022 规范要求
4. context 参数实现域隔离

对比 HKDF：
- HKDF: Extract + Expand（两步）
- BLAKE3: derive_key（单步）
```

## Prism 实现

### HKDF 接口

Prism 的 HKDF 实现：

```cpp
// hkdf.hpp

// HMAC
auto hmac_sha256(key, data) → hash;  // 32 字节
auto hmac_sha512(key, data) → hash;  // 64 字节

// HKDF-Extract
auto hkdf_extract(salt, ikm) → prk;  // 32 字节

// HKDF-Expand
auto hkdf_expand(prk, info, length) → okm;

// TLS 1.3 HKDF-Expand-Label
auto hkdf_expand_label(secret, label, context, length) → key;
  // 自动添加 "tls13 " 前缀

// SHA-256 多块重载
auto sha256(data1) → hash;
auto sha256(data1, data2) → hash;  // 避免临时拼接
auto sha256(data1, data2, data3) → hash;
```

详见 [[core/crypto/hkdf|hkdf]]。

## 安全考量

### 输入熵要求

HKDF 的输入熵要求：

```
输入熵要求：

IKM 熵：
- 必须具有足够熵
- X25519 共享密钥：~128 位熵（足够）
- 低熵输入（如密码）：需要强 Salt

Salt 作用：
- 增强 IKM 熵不足的情况
- 提供域分离
- 防止 IKM 泄露时 PRK 泄露

PRK 安全：
- PRK 必须保密
- PRK 泄露 → 所有派生密钥泄露
```

### 密钥分离

密钥分离的最佳实践：

```
密钥分离原则：

1. 不同用途不同 info
   - 加密密钥：info = "encryption key"
   - 认证密钥：info = "authentication key"
   - IV：info = "iv"

2. 不同方向不同 info
   - 客户端：info 包含 "client"
   - 服务端：info 包含 "server"

3. 不同阶段不同 info
   - 握手阶段：info 包含 "handshake"
   - 应用阶段：info 包含 "application"

4. 绑定上下文
   - info 包含 transcript hash
   - 绑定到具体会话
```

## 参见

- [[ref/crypto/hkdf|HKDF]] — HKDF 详细原理
- [[ref/crypto/tls-crypto|TLS 加密]] — TLS 1.3 密钥调度
- [[ref/crypto/blake3|BLAKE3]] — BLAKE3 密钥派生
- [[ref/crypto/hmac-sha256|HMAC-SHA256]] — HMAC 原理
- [[core/crypto/hkdf|hkdf]] — HKDF 实现接口
- [[ref/protocol/tls-handshake|TLS 握手流程]] — TLS 协议