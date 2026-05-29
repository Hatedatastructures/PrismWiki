---
layer: core
source: include/prism/stealth/anytls/padding.hpp
title: AnyTLS Padding
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, anytls, padding, traffic-shaping]
---
# AnyTLS Padding

## 设计决策（WHY）

### 为什么使用 MD5 哈希生成 PRNG 种子

MD5 将 padding_scheme 字符串转换为 128 位种子。MD5 的输出分布均匀，确保不同 scheme 字符串产生不同的填充模式。MD5 已不推荐用于安全场景，但这里仅用于 PRNG 种子生成，不涉及安全性。

### 为什么使用 mt19937 而非密码学安全 PRNG

填充大小不需要密码学安全性——其目的是打破流量长度特征，而非加密。mt19937 比密码学 PRNG（如 ChaCha20）快得多，在热路径上性能更好。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 填充大小由 scheme 字符串决定 | 确定性 | 相同 scheme 相同填充 |
| mt19937 输出范围有限 | PRNG 特性 | 填充大小有上限 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| padding_scheme 为空 | 未配置 | 无填充 |
| 填充导致帧超限 | 随机值过大 | 帧被截断 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| padding -> config | 依赖 | 读取 padding_scheme |
| session -> padding | 调用 | 构造帧时添加填充 |
