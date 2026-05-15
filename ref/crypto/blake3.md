---
title: "BLAKE3"
category: "crypto"
type: ref
tags: [密码学, blake3, 哈希, 密钥派生]
created: 2026-05-15
updated: 2026-05-15
---

# BLAKE3

**类别**: 密码学

## 概述

BLAKE3 是 BLAKE2 的改进版本，是一种快速、安全的加密哈希函数。

## 原理

### BLAKE3 结构

BLAKE3 基于 BLAKE2s，使用 Merkle 树结构。

**特性**：
- 256 位输出
- 支持任意长度输入
- 支持密钥派生
- 支持可变长度输出
- 并行计算

### 哈希计算

**处理流程**：
1. 将输入分成 1024 字节的块
2. 对每个块进行 BLAKE3 压缩
3. 使用 Merkle 树组合结果
4. 输出 256 位哈希值

### 密钥派生

BLAKE3 支持密钥派生模式：
```
output = BLAKE3::keyed(key, input)
```

### 安全性

- 128 位安全级别
- 抗碰撞攻击
- 抗原像攻击

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| blake3_hash | BLAKE3 哈希计算 | [[crypto/blake3|blake3]] |
| blake3_keyed | BLAKE3 密钥派生 | [[crypto/blake3|blake3]] |

## 参考资料

- [BLAKE3 - one function, fast everywhere](https://blake3.io/)

## 相关知识

- [[ref/crypto/sha256|SHA-256]] — 替代的哈希函数
- [[ref/crypto/hkdf|HKDF]] — 替代的密钥派生函数
