---
layer: core
source: "I:/code/Prism/include/prism/protocol/tls/feature_bitmap.hpp"
---

# TLS ClientHello 特征位图

> 源码位置: `I:/code/Prism/include/prism/protocol/tls/feature_bitmap.hpp`

## 概述

TLS ClientHello 特征位图，将关键特征压缩为位图，支持快速位运算匹配。用于分层检测管道的 Layer 0 快速检测阶段。

## 命名空间

```cpp
namespace psm::protocol::tls
```

## 特征位枚举

### feature_bit

TLS ClientHello 特征位枚举，每个位代表一个特征，支持位运算快速匹配。

```cpp
enum feature_bit : std::uint32_t
{
    // 基础特征
    has_sni = 1 << 0,                    // 有 SNI 扩展
    sni_matched_config = 1 << 1,         // SNI 匹配配置中的 server_names
    has_x25519 = 1 << 2,                 // 有 X25519 key_share
    has_full_session_id = 1 << 3,        // session_id_len == 32

    // 确定性标记（独占特征）
    reality_marker_01_08_02 = 1 << 4,    // Reality 独占标记
    shadowtls_hmac_valid = 1 << 5,       // ShadowTLS HMAC 有效

    // 结构特征
    session_id_non_standard = 1 << 6,    // session_id 长度非标准
    has_ech = 1 << 7,                    // 有 ECH 扩展
    has_esni = 1 << 8,                   // 有 ESNI 扩展（已废弃）
    greased_extensions = 1 << 9,         // 有 GREASE 扩展

    // 扩展组合
    has_supported_versions = 1 << 10,    // 有 supported_versions 扩展
    has_alpn = 1 << 11,                  // 有 ALPN 扩展
    has_psk = 1 << 12,                   // 有 PSK 扩展
    has_signature_algorithms = 1 << 13,  // 有 signature_algorithms 扩展

    // 高级特征
    key_share_multiple = 1 << 14,        // 有多个 key_share
    early_data_attempt = 1 << 15,        // 尝试 0-RTT early data

    // 保留位（16-31）
    reserved_16 = 1 << 16,
    reserved_17 = 1 << 17,
    reserved_18 = 1 << 18,
    reserved_19 = 1 << 19,
    reserved_20_to_31 = 0xFFF00000
};
```

## 核心函数

### build_feature_bitmap

构建特征位图。

```cpp
[[nodiscard]] inline auto build_feature_bitmap(const client_hello_features &features) noexcept
    -> std::uint32_t;
```

**参数**:
- `features`: ClientHello 特征结构

**返回**: 特征位图（32 位）

**实现**:
- 检查 `server_name` 是否存在 → 设置 `has_sni`
- 检查 `has_x25519` → 设置 `has_x25519`
- 检查 `session_id_len` == 32 → 设置 `has_full_session_id`
- 检查 `session_id_len` 非 32 且 > 0 → 设置 `session_id_non_standard`
- 检查 Reality 独占标记（session_id[0:3] == [0x01, 0x08, 0x02]）
- 检查 `versions` 是否存在 → 设置 `has_supported_versions`

**注意**: 不检查 SNI 是否匹配配置（`sni_matched_config` 位需要在路由阶段设置）

### has_feature

检查位图中是否有指定特征。

```cpp
[[nodiscard]] inline auto has_feature(std::uint32_t bitmap, feature_bit bit) noexcept -> bool
{
    return (bitmap & bit) != 0;
}
```

### has_all_features

检查位图中是否包含所有指定特征。

```cpp
[[nodiscard]] inline auto has_all_features(std::uint32_t bitmap, std::uint32_t bits) noexcept -> bool
{
    return (bitmap & bits) == bits;
}
```

## Reality 独占标记

```cpp
// session_id[0] == 0x01, session_id[1] == 0x08, session_id[2] == 0x02
if (features.session_id.size() >= 3 &&
    features.session_id[0] == 0x01 &&
    features.session_id[1] == 0x08 &&
    features.session_id[2] == 0x02)
{
    bitmap |= reality_marker_01_08_02;
}
```

此标记是 Reality 协议的独占特征，用于快速识别 Reality 连接。

## 调用链

- [[core/protocol/tls/types|Types]] - client_hello_features 结构定义
- [[core/protocol/tls/signal|Signal]] - 解析器提供特征数据
- [[stealth/reality|Reality]] - Reality 协议检测使用位图
- [[stealth/shadowtls|ShadowTLS]] - ShadowTLS 协议检测
- [[stealth/ech|ECH]] - ECH 协议检测