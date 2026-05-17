---
title: "密码学原理概览"
category: "crypto"
type: ref
layer: ref
module: ref
source: ""
tags: [密码学, 概览, 加密, 密钥派生, 密钥交换]
created: 2026-05-17
updated: 2026-05-17
---

# 密码学原理概览

**类别**: 密码学

## 概述

密码学原理目录提供加密算法、密钥交换、密钥派生等技术的基础原理说明。这些原理是理解现代安全协议（如 TLS 1.3）和代理协议加密机制的基础。

### 核心领域

密码学原理涵盖以下核心领域：

| 领域 | 内容 | 关键算法 |
|------|------|----------|
| 认证加密 | AEAD 原理、算法对比 | AES-GCM、ChaCha20-Poly1305 |
| 密钥交换 | ECDHE 原理、前向保密 | X25519、Curve25519 |
| 密钥派生 | HKDF 理论、BLAKE3 | HKDF-SHA256、BLAKE3 |
| TLS 加密 | TLS 1.3 密钥调度 | HKDF-Expand-Label |

### 在 Prism 中的应用

Prism 项目中密码学原理的应用场景：

| 场景 | 使用算法 | 应用模块 |
|------|----------|----------|
| Reality TLS 伪装 | X25519 + HKDF + AES-GCM | [[stealth/reality]] |
| SS2022 加密 | BLAKE3 + AES-GCM/ChaCha20 | [[protocol/shadowsocks]] |
| Trojan 认证 | SHA-224 | [[protocol/trojan]] |
| TLS 记录层加密 | AES-GCM | [[channel/transport/encrypted]] |

## 核心概念

### 认证加密 (AEAD)

AEAD（Authenticated Encryption with Associated Data）是现代密码学的核心概念，同时提供：

1. **机密性**：数据加密，防止窃听
2. **完整性**：认证标签，防止篡改
3. **关联数据**：AAD 参与认证但不加密

```
AEAD 加密输出结构：
┌────────────────────────────────────────────┐
│  密文 (Ciphertext)  │  认证标签 (Tag 16B) │
├─────────────────────┴──────────────────────┤
│  输入：明文 + 密钥 + Nonce + AAD            │
└────────────────────────────────────────────┘
```

详见 [[ref/crypto/aead-basics|AEAD 原理]]。

### 密钥交换 (Key Exchange)

密钥交换用于在不安全信道上协商共享密钥：

```
ECDHE 密钥交换流程：
客户端                                    服务端
   │                                        │
   │──── 生成临时密钥对 (私钥A, 公钥A) ────  │
   │                                        │
   │──────────── 公钥A ────────────────────>│
   │                                        │──── 生成临时密钥对 (私钥B, 公钥B)
   │<──────────── 公钥B ────────────────────│
   │                                        │
   │──── 计算共享密钥 = X25519(A私, B公) ─── │──── 计算共享密钥 = X25519(B私, A公)
   │                                        │
   └── 双方得到相同的共享密钥                └
```

前向保密特性：即使长期私钥泄露，历史会话也无法解密。

详见 [[ref/crypto/key-exchange|密钥交换原理]]。

### 密钥派生 (Key Derivation)

密钥派生将共享密钥转换为可用的加密密钥：

```
HKDF 两步流程：
共享密钥 (IKM)
      │
      ▼ HKDF-Extract(salt, IKM)
伪随机密钥 (PRK, 32B)
      │
      ▼ HKDF-Expand(PRK, info, length)
派生密钥 (OKM)
```

TLS 1.3 使用 HKDF-Expand-Label 进行密钥调度。

详见 [[ref/crypto/hkdf-theory|HKDF 理论]]。

## 算法对比

### AEAD 算法对比

| 算法 | 密钥长度 | Nonce 长度 | 硬件加速 | 适用场景 |
|------|----------|------------|----------|----------|
| AES-128-GCM | 16 字节 | 12 字节 | AES-NI | TLS 1.3 默认 |
| AES-256-GCM | 32 字节 | 12 字节 | AES-NI | 高安全需求 |
| ChaCha20-Poly1305 | 32 字节 | 12 字节 | 无需 | 移动设备、嵌入式 |
| XChaCha20-Poly1305 | 32 字节 | 24 字节 | 无需 | UDP、随机 nonce |

### 密钥交换算法对比

| 算法 | 安全强度 | 密钥长度 | 计算速度 | 标准 |
|------|----------|----------|----------|------|
| X25519 | 128 位 | 32 字节 | 快 | RFC 7748 |
| ECDHE P-256 | 128 位 | 32 字节 | 中 | NIST |
| ECDHE P-384 | 192 位 | 48 字节 | 慢 | NIST |

### 密钥派生算法对比

| 算法 | 步骤 | 性能 | 适用场景 |
|------|------|------|----------|
| HKDF-SHA256 | Extract + Expand | 中 | TLS 1.3 |
| BLAKE3 derive_key | 单步 | 快 | SS2022 |

## 安全考量

### Nonce 管理

Nonce (Number used ONCE) 是 AEAD 安全的关键：

```
Nonce 重用风险：
相同密钥 + 相同 Nonce + 不同消息 → 认证密钥泄露 → 可伪造消息

安全策略：
1. TCP 流：计数器 nonce，从 0 递增
2. UDP 包：随机 nonce，足够长防碰撞
```

### 密钥生命周期

密钥的安全生命周期：

```
密钥生命周期：
生成 → 使用 → 派生子密钥 → 丢弃
      │         │
      │         └── 使用次数限制
      │             AES-GCM: ~2^32 次 (12B nonce)
      │             XChaCha: ~2^192 次 (24B nonce)
      │
      └── 定期更换
          TLS 1.3: 每次握手新密钥
```

### 前向保密

前向保密的实现条件：

1. 使用临时密钥交换（ECDHE）
2. 会话结束后销毁临时私钥
3. 不使用静态密钥加密

## 子目录索引

| 文件 | 内容 | 链接 |
|------|------|------|
| AEAD 原理 | 认证加密基础概念 | [[ref/crypto/aead-basics]] |
| 密钥交换原理 | X25519 ECDHE | [[ref/crypto/key-exchange]] |
| HKDF 理论 | 密钥派生函数 | [[ref/crypto/hkdf-theory]] |
| TLS 加密 | TLS 1.3 密钥调度 | [[ref/crypto/tls-crypto]] |
| AES-GCM | AES-GCM 详细原理 | [[ref/crypto/aes-gcm]] |
| ChaCha20-Poly1305 | ChaCha 详细原理 | [[ref/crypto/chacha20-poly1305]] |
| X25519 | Curve25519 详细原理 | [[ref/crypto/x25519]] |
| HKDF | HKDF 详细原理 | [[ref/crypto/hkdf]] |
| BLAKE3 | BLAKE3 详细原理 | [[ref/crypto/blake3]] |
| ECDHE | ECDHE 详细原理 | [[ref/crypto/ecdhe]] |

## 参见

- [[crypto/overview|Crypto 模块]] — 加密算法实现层
- [[ref/protocol/tls-handshake|TLS 握手流程]] — TLS 协议
- [[stealth/reality|Reality]] — TLS 伪装方案
- [[ref/overview|参考资料概览]] — 参考资料层索引