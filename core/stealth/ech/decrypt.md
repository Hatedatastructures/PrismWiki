---
tags: [stealth, ech, decrypt]
layer: core
source: I:/code/Prism/include/prism/stealth/ech/decrypt.hpp
title: ECH 解密接口
---

# ECH 解密接口

> 源码位置: `I:/code/Prism/include/prism/stealth/ech/decrypt.hpp`

## 设计决策（WHY）

### 为什么使用 HPKE 而非简单的公钥加密

ECH 规范（RFC 9460）使用 HPKE（Hybrid Public Key Encryption）进行 ClientHello 加密。HPKE 提供了前向安全性、密钥封装和认证加密的组合。简单的公钥加密（如 RSA 加密 SNI）无法提供这些安全属性。

### 为什么当前实现返回 `not_supported`

ECH 的 HPKE 解密需要完整的密码学实现（X25519 KEM + AES-128-GCM AEAD + HKDF），以及 ECH 特定的序列化/反序列化逻辑。这些组件尚未实现。当前代码定义了接口（`decrypt_ech_payload`），但内部直接返回 `not_supported`。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 当前未实现 | decrypt.cpp | 返回 `fault::code::not_supported` |
| 需要 HPKE 库支持 | 规范要求 | 缺少则无法实现 |
| `decrypt_result` 结构已定义但未使用 | 接口预留 | 等待实现 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 任何调用 | 当前状态 | 返回 `not_supported` |
| 未来实现时密钥不匹配 | ech_key 与客户端期望不同 | 解密失败 |
| HPKE 参数不支持 | 客户端使用不支持的 KEM/AEAD | 解密失败 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `decrypt` → `ech::config` | 依赖 | 使用 ech_key 进行解密 |
| `anytls::scheme` → `decrypt` | 调用 | `verify()` 中尝试 ECH 解密 |

## 概述

使用 HPKE (Hybrid Public Key Encryption) 解密 ECH outer payload，获取 inner ClientHello。ECH（Encrypted Client Hello）是 TLS 扩展，加密 ClientHello 中的 SNI，防止 SNI 泄露。

## 命名空间

```cpp
namespace psm::stealth::ech
```

## 结构定义

### decrypt_result

ECH 解密结果。

```cpp
struct decrypt_result
{
    bool valid{false};                          ///< 解密是否成功
    protocol::tls::client_hello_features inner_features; ///< 解密后的 inner 特征
    memory::vector<std::byte> inner_client_hello; ///< 解密后的 inner ClientHello 原始字节
    fault::code error{fault::code::success};    ///< 错误码
};
```

## 核心函数

### decrypt_ech_payload

解密 ECH outer payload。

```cpp
[[nodiscard]] auto decrypt_ech_payload(
    std::span<const std::byte> outer_payload,
    std::string_view ech_key) -> decrypt_result;
```

**参数**：
- `outer_payload`: ECH outer payload（从 ClientHello 扩展提取）
- `ech_key`: ECH 配置密钥（base64 编码）

**返回**: 解密结果，包含 inner ClientHello 和提取的特征

## ECH Outer 格式

```
┌─────────────────────────────────────────────────────────────┐
│ ECH Outer Payload                                           │
├─────────────────────────────────────────────────────────────┤
│ version (2 bytes)                                           │
│   0xfe 0x0d for TLS 1.3                                     │
├─────────────────────────────────────────────────────────────┤
│ config_id (1 byte)                                          │
│   ECH 配置标识符                                             │
├─────────────────────────────────────────────────────────────┤
│ enc (variable length)                                       │
│   KEM encapsulated key                                      │
├─────────────────────────────────────────────────────────────┤
│ payload (variable length)                                   │
│   AEAD encrypted inner ClientHello                          │
└─────────────────────────────────────────────────────────────┘
```

## HPKE 解密流程

```
┌─────────────────┐                    ┌─────────────────┐
│ ECH Outer       │                    │ HPKE 解密       │
├─────────────────┤                    ├─────────────────┤
│ version         │                    │                 │
│ config_id       │──► 匹配 ECH Key ──►│ 使用 enc 计算   │
│ enc             │                    │ shared secret   │
│ payload         │──► AEAD 解密 ────►│ 解密 payload    │
└─────────────────┘                    └─────────────────┘
                                                │
                                                ▼
                                       ┌─────────────────┐
                                       │ Inner ClientHello │
                                       │ - 真实 SNI        │
                                       │ - 真实 ALPN       │
                                       │ - 其他扩展        │
                                       └─────────────────┘
```

## 解密步骤

1. **解析 ECH outer 结构**
   - 提取 version、config_id、enc、payload
   - 验证 version 为 TLS 1.3 (0xfe 0x0d)

2. **计算共享密钥**
   - 使用 ECH 私钥和 enc 计算 HPKE shared secret

3. **AEAD 解密**
   - 使用共享密钥解密 payload，获取 inner ClientHello

4. **解析 inner ClientHello**
   - 提取真实 SNI 和其他特征
   - 返回 decrypt_result

## 使用场景

ECH 可叠加在任意 TLS 伪装协议上：

| 方案 | ECH 支持 | 说明 |
|------|----------|------|
| Reality | 否 | 已有自己的 SNI 隐藏机制 |
| AnyTLS | 是 | 可叠加 ECH 加密 SNI |
| TrustTunnel | 是 | 可叠加 ECH 加密 SNI |

## 调用链

```
anytls::scheme::verify -> ech::decrypt_ech_payload
trusttunnel::scheme::verify -> ech::decrypt_ech_payload
```

## 依赖

- [[core/stealth/ech/config|ECH 配置]] - ECH 密钥格式
- [[core/protocol/tls/types|TLS 类型]] - ClientHello 特征结构
- [[log/fault/code|错误码]] - 错误码系统