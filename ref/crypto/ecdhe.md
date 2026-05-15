---
title: "ECDHE"
category: "crypto"
type: ref
tags: [密码学, ecdhe, 密钥交换, 前向保密]
created: 2026-05-15
updated: 2026-05-15
---

# ECDHE

**类别**: 密码学

## 概述

ECDHE（Elliptic Curve Diffie-Hellman Ephemeral）是基于椭圆曲线的临时 Diffie-Hellman 密钥交换协议，提供前向保密。

## 原理

### ECDHE 结构

ECDHE 结合了 ECDH 和临时密钥。

**密钥交换流程**：
1. 双方生成临时密钥对
2. 交换公钥
3. 计算共享密钥
4. 删除临时私钥

### 前向保密

**特性**：
- 每次会话使用新的临时密钥
- 即使长期私钥泄露，历史会话仍安全
- 删除临时私钥后无法恢复共享密钥

### 安全性

- 基于椭圆曲线离散对数问题（ECDLP）
- 提供前向保密
- 128 位安全级别（使用 256 位曲线）

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| x25519_keypair_generate | 生成临时密钥对 | [[crypto/x25519|x25519]] |
| x25519_shared_secret | 计算共享密钥 | [[crypto/x25519|x25519]] |
| derive_handshake_keys | TLS 1.3 密钥派生 | [[stealth/reality/keygen|keygen]] |

## 参考资料

- [RFC 4492 - Elliptic Curve Cryptography (ECC) Cipher Suites for TLS](https://tools.ietf.org/html/rfc4492)

## 相关知识

- [[ref/crypto/x25519|X25519]] — 具体的 ECDHE 实现
- [[ref/protocol/tls-1.3|TLS 1.3]] — 使用 ECDHE 进行密钥交换
