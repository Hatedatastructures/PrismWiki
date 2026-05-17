---
title: "AEAD 认证加密原理"
category: "crypto"
type: ref
layer: ref
module: ref
source: "NIST SP 800-38D, RFC 5116"
tags: [密码学, aead, 认证加密, aes-gcm, chacha20]
created: 2026-05-17
updated: 2026-05-17
---

# AEAD 认证加密原理

**类别**: 密码学

## 概述

AEAD（Authenticated Encryption with Associated Data，认证加密）是现代密码学的核心概念，将加密和认证合并为单一操作，同时提供数据的机密性、完整性和真实性保证。AEAD 解决了传统"先加密后认证"方案的安全问题，是 TLS 1.3、IPsec、WireGuard 等现代协议的加密基础。

### 核心特性

AEAD 提供三重保证：

| 特性 | 说明 | 作用 |
|------|------|------|
| **机密性** | 数据加密，无法窃听 | 防止信息泄露 |
| **完整性** | 认证标签，无法篡改 | 防止数据修改 |
| **真实性** | 验证发送方 | 防止伪造消息 |

### 关键创新：AAD

AEAD 的关键创新是支持附加认证数据（AAD）：

```
AEAD 输入输出结构：
┌─────────────────────────────────────────────────────┐
│ 输入：                                               │
│   - 明文 (Plaintext)：需要加密的数据                  │
│   - 密钥 (Key)：加密密钥                              │
│   - Nonce：一次性随机数                               │
│   - AAD (Associated Data)：附加认证数据              │
│                                                      │
│ 输出：                                               │
│   - 密文 (Ciphertext)：加密后的数据                   │
│   - 认证标签 (Authentication Tag)：16 字节           │
└─────────────────────────────────────────────────────┘

AAD 特性：
- 不加密，但参与认证
- 可用于协议头部、地址信息等
- 接收方必须提供相同 AAD 才能解密
```

AAD 的典型应用场景：

| 场景 | AAD 内容 | 作用 |
|------|----------|------|
| TLS 记录层 | 序列号、记录类型 | 防止记录篡改 |
| IPsec | SPI、序列号 | 防止包重放 |
| WireGuard | 随机 nonce | 绑定 nonce |

## 算法原理

### 为什么需要 AEAD

传统加密方案存在安全问题：

```
传统方案的问题：

方案 1: 先加密后 MAC (Encrypt-then-MAC)
  加密 → MAC(密文)
  问题：需要两次操作，实现复杂

方案 2: 先 MAC 后加密 (MAC-then-Encrypt)
  MAC(明文) → 加密(MAC || 明文)
  问题：TLS 1.2 CBC 模式的 Lucky13 攻击

方案 3: 加密并 MAC (Encrypt-and-MAC)
  加密(明文) || MAC(明文)
  问题：MAC 可能泄露明文信息

AEAD 解决方案：
  单一操作同时完成加密和认证
  - 无需分开实现
  - 无需担心操作顺序
  - 安全性经过严格证明
```

### AEAD 接口定义

标准 AEAD 接口（RFC 5116）：

```cpp
// 加密接口
seal(key, nonce, plaintext, aad) → (ciphertext, tag)

// 解密接口
open(key, nonce, ciphertext, tag, aad) → plaintext or FAIL

// 参数约束：
// - key: 算法特定长度（16/32 字节）
// - nonce: 算法特定长度（12/24 字节）
// - tag: 通常 16 字节
```

### 安全要求

AEAD 的安全要求：

```
安全要求：

1. Nonce 唯一性：
   相同密钥 + 相同 nonce + 不同消息 → 安全失效
   
   AES-GCM: nonce 重用泄露认证密钥 H
   ChaCha20: nonce 重用泄露明文 XOR
   
2. 密钥使用限制：
   AES-GCM (12B nonce): 最多 2^32 次加密
   XChaCha20 (24B nonce): 最多 2^192 次加密
   
3. 标签验证：
   必须先验证标签再解密
   验证失败必须立即丢弃，不泄露任何信息
```

## 主要算法

### AES-GCM

AES-GCM 是最广泛使用的 AEAD 算法：

```
AES-GCM 结构：
┌─────────────────────────────────────────────┐
│                                             │
│   GCM = CTR 模式加密 + GHASH 认证            │
│                                             │
│   加密：AES-CTR(明文) → 密文                 │
│   认证：GHASH(AAD || 密文) → 标签            │
│                                             │
│   CTR 计数器：nonce || counter               │
│   GHASH：GF(2^128) 上的多项式运算            │
│                                             │
└─────────────────────────────────────────────┘

参数：
- 密钥长度：16/24/32 字节（AES-128/192/256）
- Nonce 长度：12 字节（推荐）
- 标签长度：16 字节（推荐 128 位）
- 分组大小：16 字节
```

AES-GCM 性能特性：

| 特性 | 说明 |
|------|------|
| 硬件加速 | AES-NI + PCLMULQDQ，可达 10 GB/s |
| 并行性 | 加密可完全并行，认证可部分并行 |
| 安全强度 | AES-128: 128 位，AES-256: 256 位 |

详见 [[ref/crypto/aes-gcm|AES-GCM]]。

### ChaCha20-Poly1305

ChaCha20-Poly1305 是软件优化的 AEAD 算法：

```
ChaCha20-Poly1305 结构：
┌─────────────────────────────────────────────┐
│                                             │
│   ChaCha20: 流加密，生成密钥流               │
│   Poly1305: 认证，计算 MAC                   │
│                                             │
│   加密：明文 XOR ChaCha20-密钥流 → 密文      │
│   认证：Poly1305(AAD || 密文) → 标签         │
│                                             │
└─────────────────────────────────────────────┘

参数：
- 密钥长度：32 字节
- Nonce 长度：12 字节
- 标签长度：16 字节
```

ChaCha20-Poly1305 特性：

| 特性 | 说明 |
|------|------|
| 无需硬件加速 | 纯软件实现性能优异 |
| 常量时间 | 所有操作常量时间，抗侧信道 |
| 移动设备 | 适合无 AES-NI 的设备 |

详见 [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]。

### XChaCha20-Poly1305

XChaCha20-Poly1305 扩展 nonce 长度：

```
XChaCha20 结构：
┌─────────────────────────────────────────────┐
│                                             │
│   扩展 nonce: 24 字节                        │
│                                             │
│   子密钥派生：                               │
│   HChaCha20(key, nonce[0:16]) → subkey      │
│                                             │
│   加密：                                     │
│   ChaCha20(subkey, nonce[16:24] || counter) │
│                                             │
└─────────────────────────────────────────────┘

参数：
- 密钥长度：32 字节
- Nonce 长度：24 字节
- 标签长度：16 字节
```

XChaCha20 优势：

| 特性 | 说明 |
|------|------|
| 长 nonce | 24 字节，随机 nonce 无碰撞风险 |
| 使用限制 | ~2^192 次加密，几乎无限 |
| UDP 适用 | 适合随机 nonce 的 UDP 场景 |

## Nonce 管理

### Nonce 管理策略

Nonce 管理是 AEAD 安全的关键：

```
Nonce 管理策略：

TCP 流场景（有序）：
- 计数器 nonce，从 0 开始
- 每次加密后递增
- 小端序递增，便于实现

nonce[0] = 0 → 1 → 2 → ... → 255 → 0 (进位)
nonce[1] = 0 → ... → 0 → 1 (进位)

UDP 包场景（无序）：
- 随机 nonce
- 需要足够长防碰撞
- 12 字节：每密钥最多 2^32 包
- 24 字节：几乎无碰撞风险
```

### Prism 实现

Prism 的 AEAD nonce 管理：

```cpp
// TCP 流：自动 nonce
class aead_context {
    std::array<uint8_t, 12> nonce_;
    
    // seal: 使用当前 nonce，然后递增
    auto seal(output, plaintext, aad);
    
    // open: 使用当前 nonce，然后递增
    auto open(output, ciphertext, aad);
    
    // nonce 递增（小端序）
    void increment_nonce();
};

// UDP 包：显式 nonce
auto seal(output, plaintext, explicit_nonce, aad);
auto open(output, ciphertext, explicit_nonce, aad);
```

## 在协议中的应用

### TLS 1.3

TLS 1.3 使用 AEAD 加密记录层：

```
TLS 1.3 记录结构：
┌───────────────────────────────────────────────┐
│ 0  │ ContentType │  LegacyVersion │ Length    │
│    │ (1 byte)    │ (2 bytes)      │ (2 bytes) │
├───────────────────────────────────────────────┤
│                 Encrypted Data                 │
│            (AEAD ciphertext + tag)             │
├───────────────────────────────────────────────┤
│                   AAD                          │
│  序列号(8B) || ContentType || LegacyVersion || │
│  Length                                        │
└───────────────────────────────────────────────┘

AAD = 序列号 || 记录头部
Nonce = 序列号 || client/server_write_iv
```

### SS2022

Shadowsocks 2022 使用 AEAD 加密 payload：

```
SS2022 TCP 数据包：
┌───────────────────────────────────────────────┐
│                 密文 payload                   │
│            (AEAD ciphertext + tag)             │
└───────────────────────────────────────────────┘

Nonce: 计数器，从 0 开始，小端序递增
AAD: 空（不使用）
```

## 安全最佳实践

### 使用建议

| 建议 | 说明 |
|------|------|
| 标签长度 | 使用 16 字节（128 位），避免截断 |
| Nonce 唯一 | 绝对避免 nonce 重用 |
| 先验证后解密 | 验证标签后才解密，不泄露任何信息 |
| 密钥轮换 | 定期更换密钥，限制使用次数 |
| AAD 使用 | 将协议头部作为 AAD，增强安全性 |

### 错误处理

```cpp
// 正确的错误处理
auto ec = ctx.open(decrypted, ciphertext, aad);
if (ec != fault::code::success) {
    // 验证失败
    // 1. 不返回部分解密结果
    // 2. 不泄露具体错误原因
    // 3. 记录日志（不包含密钥材料）
    log_error("AEAD authentication failed");
    return ec;
}
```

## 参见

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-GCM 详细原理
- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — ChaCha 详细原理
- [[crypto/aead|aead]] — AEAD 实现接口
- [[ref/crypto/tls-crypto|TLS 加密]] — TLS 1.3 加密
- [[ref/crypto/overview|密码学原理概览]] — 密码学索引