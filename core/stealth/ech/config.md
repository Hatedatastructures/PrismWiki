---
tags: [stealth, ech, config]
layer: core
source: I:/code/Prism/include/prism/stealth/ech/config.hpp
title: ECH（Encrypted Client Hello）配置
---

# ECH（Encrypted Client Hello）配置

> 源码位置: `I:/code/Prism/include/prism/stealth/ech/config.hpp`

## 设计决策（WHY）

### 为什么 ECH key 是独立的配置而非嵌入其他方案

ECH 是一个通用的 TLS 扩展，可以叠加在任意伪装方案上。将其配置独立出来避免了在每个方案中重复 ECH 相关字段。AnyTLS 和 TrustTunnel 都可以引用同一个 ECH 配置。

### 为什么 `enabled()` 只检查 `ech_key`

ECH 的最小配置只需要密钥。`public_name` 有合理的默认值（空字符串表示不验证），不阻止 ECH 启用。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `ech_key` 格式必须正确 | HPKE 密钥格式 | 格式错误导致解密失败 |
| ECH 当前解密未实现 | decrypt.cpp 返回 not_supported | 配置 ECH 不会生效 |
| `public_name` 用于 ECH 的 outer SNI | 回退机制 | 不匹配可能导致客户端重试 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| ech_key 格式错误 | 配置失误 | ECH 解密失败 |
| 解密未实现 | 当前代码状态 | 返回 `not_supported` |
| public_name 与外层证书不匹配 | 配置不一致 | 客户端可能拒绝连接 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `ech::config` → `psm::config` | 嵌入 | `cfg.stealth.ech` 路径 |
| `anytls::scheme` → `ech::config` | 读取 | ECH 检测时使用 |
| `ech::decrypt` → `ech::config` | 依赖 | 解密使用 ech_key |

## 概述

ECH 是 TLS 扩展，加密 ClientHello 中的 SNI，防止 SNI 泄露。可以叠加在任意 TLS 伪装协议上（如 AnyTLS、TrustTunnel）。

协议参考: https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni

## 命名空间

```cpp
namespace psm::stealth::ech
```

## 结构定义

### config

ECH 服务端配置。

```cpp
struct config
{
    memory::string ech_key;         ///< ECH 密钥（base64 编码）
    memory::string public_name;     ///< 公开的伪装域名

    /**
     * @brief 检查是否启用
     * @return ech_key 非空时返回 true
     */
    [[nodiscard]] auto enabled() const noexcept -> bool
    {
        return !ech_key.empty();
    }
};
```

## 配置项说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `ech_key` | string | ECH 密钥（base64 编码），包含私钥和配置信息 |
| `public_name` | string | 公开的伪装域名，用于 outer ClientHello |

## 配置方法

### enabled

检查是否启用 ECH。

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool
{
    return !ech_key.empty();
}
```

## JSON 序列化

使用 Glaze 库进行 JSON 序列化：

```json
{
    "ech_key": "base64_encoded_ech_key",
    "public_name": "www.example.com"
}
```

## ECH 密钥格式

ECH 密钥为 base64 编码的 ECHConfigList 结构，包含：

| 组件 | 说明 |
|------|------|
| `public_name` | 外层 ClientHello 的 SNI |
| `public_key` | KEM 公钥 |
| `kem_id` | KEM 标识符 |
| `cipher_suites` | 支持的 KDF/AEAD 组合 |
| `max_name_length` | 最大内层 SNI 长度 |

## ECH 工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│ Client                                                          │
├─────────────────────────────────────────────────────────────────┤
│ 1. 获取 ECH 配置（包含 public_name 和 public_key）               │
│ 2. 构造 inner ClientHello（真实 SNI）                            │
│ 3. 使用 public_key 加密 inner ClientHello                        │
│ 4. 构造 outer ClientHello（public_name 作为 SNI）                │
│ 5. 发送 outer ClientHello + ECH 扩展                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Server                                                          │
├─────────────────────────────────────────────────────────────────┤
│ 1. 解析 ECH 扩展                                                 │
│ 2. 使用 ech_key 解密 inner ClientHello                          │
│ 3. 获取真实 SNI，继续 TLS 握手                                   │
└─────────────────────────────────────────────────────────────────┘
```

## ECH 与 SNI 隐藏

| 方式 | SNI 可见性 | 说明 |
|------|------------|------|
| 无 ECH | 明文 | SNI 可被 DPI 检测 |
| 有 ECH | 加密 | outer SNI 为伪装域名 |

## 可叠加方案

ECH 可叠加在以下方案上：

| 方案 | 配置字段 | 说明 |
|------|----------|------|
| AnyTLS | `ech_key` | 加密 SNI |
| TrustTunnel | `ech_key` | 加密 SNI |

## 与 Reality 的区别

| 特性 | ECH | Reality |
|------|-----|---------|
| SNI 加密 | 是 | 否（使用后端 SNI） |
| 证书验证 | 需要 | 不需要 |
| 指纹特征 | 标准扩展 | 特殊 key_share |
| 叠加性 | 可叠加 | 独立方案 |

## 调用链

```
config::enabled() <- anytls::scheme::verify()
config::ech_key -> ech::decrypt_ech_payload()
```

## 依赖

- [[core/stealth/ech/decrypt|ECH 解密]] - ECH 解密接口
- [[core/stealth/anytls/config|AnyTLS 配置]] - ECH 叠加支持