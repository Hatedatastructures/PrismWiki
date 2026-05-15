---
title: "confidence.hpp — 检测置信度枚举定义"
source: "include/prism/recognition/confidence.hpp"
module: "recognition"
type: api
tags: [recognition, confidence, 置信度, 枚举]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/result
  - recognition/layered_pipeline
  - recognition/recognition
  - stealth/scheme
---

# confidence.hpp

> 源码: `include/prism/recognition/confidence.hpp`
> 模块: [[recognition|Recognition]]

## 概述

定义 ClientHello 特征分析结果的置信度级别，用于指导方案执行顺序。置信度越高，优先执行对应方案的可能性越大。该枚举被 `analysis_result` 和分层检测管道的候选评分体系使用。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[recognition/result\|result]] | analysis_result 使用 confidence |
| 被依赖 | [[recognition/layered_pipeline\|layered_pipeline]] | 分层检测使用置信度 |
| 被依赖 | [[stealth/scheme\|scheme]] | sniff/hint 使用置信度级别 |

## 命名空间

`psm::recognition`

---

## 枚举: confidence

### 功能说明

检测置信度级别枚举。用于 `analysis_result::score` 字段和 `candidate_entry::score` 的语义映射。高置信度方案优先执行，`none` 表示无特征匹配，执行 Native 兜底方案。

### 签名

```cpp
enum class confidence : std::uint8_t { high, medium, low, none };
```

### 参数表格

无（枚举类型）。

### 返回值

无（枚举类型定义）。

### 调用（向下）

无。

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `analysis_result::score` | [[recognition/result\|result]] | 存储最高候选的置信度 |
| `candidate_entry::score` | [[recognition/layered_pipeline\|layered_pipeline]] | 候选条目的评分语义 |
| scheme `sniff()` 返回值 | [[stealth/scheme\|scheme]] | Tier 0 检测的 hint 字段 |

### 知识域

- [[ref/protocol/tls-1.3\|TLS 1.3]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]

---

## 枚举值详解

| 值 | 底层类型 | 说明 | 典型场景 |
|----|----------|------|----------|
| `high` | `std::uint8_t` | 高置信度：特征完全匹配，可直接执行对应方案 | Reality session_id 标记命中 |
| `medium` | `std::uint8_t` | 中置信度：特征部分匹配，需完整验证 | ShadowTLS HMAC 验证通过 |
| `low` | `std::uint8_t` | 低置信度：特征部分匹配但不确定，需验证 | SNI 匹配但特征不完全 |
| `none` | `std::uint8_t` | 无特征：标准 TLS，执行 Native 兜底方案 | 未匹配任何伪装方案 |

## 知识域

- [[ref/protocol/tls-1.3\|TLS 1.3]]
- [[stealth/scheme\|伪装方案接口]]
- [[recognition/result\|分析结果]]
- [[recognition/layered_pipeline\|分层检测管道]]
