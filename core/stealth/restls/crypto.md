---
layer: core
source: include/prism/stealth/restls/crypto.hpp
title: Restls Crypto
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, restls, crypto, blake3, auth-mac, mask]
---
# Restls Crypto

> 源码位置: include/prism/stealth/restls/crypto.hpp

## 设计决策（WHY）

### 为什么使用 BLAKE3 keyed mode 而非 HMAC

BLAKE3 keyed mode 使用 blake3_hasher_init_keyed 初始化，提供与 HMAC 等价的 MAC 功能。优势：BLAKE3 原生支持 SIMD 并行化，在支持 AVX2/AVX-512 的 CPU 上比 HMAC-SHA1 快 5-10 倍。且 BLAKE3 的 keyed mode 不需要 HMAC 的嵌套构造，更简洁高效。

### 为什么握手 MAC 16 字节而应用数据 MAC 8 字节

握手阶段（hs_maclen=16）的 MAC 用于一次性 XOR 混淆整个 TLS encrypted record，需要更大的 MAC 确保混淆质量。应用数据阶段（appdata_maclen=8）每帧都重新计算 MAC，帧级别验证，8 字节足够。

### 为什么 mask 长度只有 4 字节

mask 仅用于 XOR 混淆 payload 的长度字段和命令字段，4 字节足够覆盖。更长的 mask 不会增加安全性。

### 为什么 auth_mac 输入包含 direction_string

direction_string 区分读写方向：server-to-client 和 client-to-server 各 16 字节。即使 payload 完全相同，不同方向的 auth_mac 也不同，防止方向混淆攻击。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| derive_secret 使用固定上下文 | restls-traffic-key | 硬编码字符串 |
| server_mask 输出固定 16 字节 | XOR 混淆 TLS 记录 | |
| auth_mac 固定 8 字节 | appdata_maclen | 截断输出 |
| mask 固定 4 字节 | mask_len | XOR 混淆密钥 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| derive_secret 输入为空 | 空密码 | 产生确定性弱密钥 |
| BLAKE3 初始化失败 | 系统资源不足 | 未定义行为 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| crypto -> crypto::blake3 | 调用 | derive_key / keyed_hasher |
| handshake -> crypto | 调用 | derive_secret / compute_server_mask |
| transport -> crypto | 调用 | compute_auth_mac / compute_mask |
