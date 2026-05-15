---
title: "feature_bitmap — TLS ClientHello 特征位图"
source: "include/prism/protocol/tls/feature_bitmap.hpp"
module: "protocol"
type: api
tags: [protocol, tls, feature_bitmap, 指纹, ClientHello, 位图, 快速匹配]
related:
  - "[[protocol/tls/types|types]]"
  - "[[protocol/tls/signal|signal]]"
  - "[[stealth/reality/scheme|reality]]"
  - "[[stealth/shadowtls/scheme|shadowtls]]"
created: 2026-05-15
updated: 2026-05-15
---

# feature_bitmap.hpp

> 源码: `include/prism/protocol/tls/feature_bitmap.hpp`
> 模块: [[protocol|protocol]] > tls

## 概述

TLS ClientHello 特征位图。将关键特征压缩为 32 位位图，支持快速位运算匹配。用于分层检测管道的 Layer 0 快速检测阶段，在进入更昂贵的方案验证前快速过滤候选方案。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/tls/types|types]] | `client_hello_features` 结构 |
| 被依赖 | [[stealth/reality/scheme|reality]] | Reality 方案快速检测 |
| 被依赖 | [[stealth/shadowtls/scheme|shadowtls]] | ShadowTLS 方案快速检测 |

## 命名空间

`psm::protocol::tls`

---

## 枚举: feature_bit

每个位代表一个 ClientHello 特征，支持位运算快速匹配。枚举类型为 `std::uint32_t`。

### 基础特征

| 值 | 位 | 说明 |
|----|------|------|
| `has_sni` | `1 << 0` | 有 SNI 扩展 |
| `sni_matched_config` | `1 << 1` | SNI 匹配配置中的 `server_names`（需要在路由阶段设置） |
| `has_x25519` | `1 << 2` | 有 X25519 key_share |
| `has_full_session_id` | `1 << 3` | `session_id_len == 32` |

### 确定性标记（独占特征）

| 值 | 位 | 说明 |
|----|------|------|
| `reality_marker_01_08_02` | `1 << 4` | Reality 独占标记: `session_id[0:3] == [0x01, 0x08, 0x02]` |
| `shadowtls_hmac_valid` | `1 << 5` | ShadowTLS HMAC 有效（需要 Layer 1 验证） |

### 结构特征

| 值 | 位 | 说明 |
|----|------|------|
| `session_id_non_standard` | `1 << 6` | session_id 长度非标准（!= 32） |
| `has_ech` | `1 << 7` | 有 ECH (Encrypted Client Hello) 扩展 |
| `has_esni` | `1 << 8` | 有 ESNI 扩展（旧版，已废弃） |
| `greased_extensions` | `1 << 9` | 有 GREASE 扩展（RFC 8701） |

### 扩展组合

| 值 | 位 | 说明 |
|----|------|------|
| `has_supported_versions` | `1 << 10` | 有 `supported_versions` 扩展 |
| `has_alpn` | `1 << 11` | 有 ALPN 扩展 |
| `has_psk` | `1 << 12` | 有 PSK 扩展 |
| `has_signature_algorithms` | `1 << 13` | 有 `signature_algorithms` 扩展 |

### 高级特征

| 值 | 位 | 说明 |
|----|------|------|
| `key_share_multiple` | `1 << 14` | 有多个 key_share |
| `early_data_attempt` | `1 << 15` | 尝试 0-RTT early data |

### 保留位

| 值 | 位 | 说明 |
|----|------|------|
| `reserved_16` ~ `reserved_19` | `1 << 16` ~ `1 << 19` | 保留位 |
| `reserved_20_to_31` | `0xFFF00000` | 保留位 20-31 |

---

## 函数: build_feature_bitmap()

### 功能说明

从 `client_hello_features` 结构构建 32 位特征位图。检查 SNI、X25519、session_id、Reality 标记和 supported_versions 等特征，设置对应的位。不检查 SNI 是否匹配配置（`sni_matched_config` 位需要在路由阶段设置）。

### 签名

```cpp
[[nodiscard]] inline auto build_feature_bitmap(const client_hello_features &features) noexcept
    -> std::uint32_t;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `features` | `const client_hello_features &` | ClientHello 特征结构 |

### 返回值

`std::uint32_t` — 特征位图（32 位），每个位代表一个特征。

### 调用（向下）

- [[protocol/tls/types|types]] — 读取 `client_hello_features` 字段

### 被调用（向上）

- [[stealth/reality/scheme|reality]] — Reality 快速检测
- [[stealth/shadowtls/scheme|shadowtls]] — ShadowTLS 快速检测
- [[recognition/recognition|recognition]] — 协议识别流程

### 知识域

位图算法、TLS 特征提取

---

## 函数: has_feature()

### 功能说明

检查位图中是否有指定的单个特征位。使用位与运算判断。

### 签名

```cpp
[[nodiscard]] inline auto has_feature(std::uint32_t bitmap, feature_bit bit) noexcept -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `bitmap` | `std::uint32_t` | 特征位图 |
| `bit` | `feature_bit` | 要检查的特征位 |

### 返回值

`bool` — `true` 表示存在该特征，`false` 表示不存在。

### 调用（向下）

无（纯位运算）

### 被调用（向上）

- [[stealth/reality/scheme|reality]] — Reality 特征检查
- [[stealth/shadowtls/scheme|shadowtls]] — ShadowTLS 特征检查

### 知识域

位运算

---

## 函数: has_all_features()

### 功能说明

检查位图中是否包含所有指定的特征位。使用位与和等于运算判断。

### 签名

```cpp
[[nodiscard]] inline auto has_all_features(std::uint32_t bitmap, std::uint32_t bits) noexcept -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `bitmap` | `std::uint32_t` | 特征位图 |
| `bits` | `std::uint32_t` | 要检查的特征位组合 |

### 返回值

`bool` — `true` 表示所有指定位都已设置，`false` 表示有缺失。

### 调用（向下）

无（纯位运算）

### 被调用（向上）

- 方案检测管道中的组合条件检查

### 知识域

位运算

## 相关页面

- [[protocol/tls/types|types]] — TLS 共享类型
- [[protocol/tls/signal|signal]] — ClientHello 解析器
- [[stealth/reality/scheme|reality]] — Reality 伪装方案
