---
layer: core
source: I:/code/Prism/include/prism/stealth/anytls/config.hpp
title: AnyTLS 伪装方案配置
---

# AnyTLS 伪装方案配置

> 源码位置: `I:/code/Prism/include/prism/stealth/anytls/config.hpp`

## 概述

AnyTLS 是一种 TLS 伪装协议，使用标准 TLS 证书，通过应用层认证实现代理功能。可以叠加 ECH 加密 ClientHello SNI。

协议参考: https://github.com/anytls/anytls

## 命名空间

```cpp
namespace psm::stealth::anytls
```

## 结构定义

### user

AnyTLS 用户配置。

```cpp
struct user
{
    memory::string username;  ///< 用户名
    memory::string password;  ///< 密码
};
```

### config

AnyTLS 服务端配置。

```cpp
struct config
{
    // === SNI 配置（必需）===
    memory::vector<memory::string> server_names;  ///< SNI 白名单

    // === TLS 证书（必需）===
    memory::string certificate;      ///< 证书文件路径（PEM）
    memory::string private_key;      ///< 私钥文件路径（PEM）

    // === 用户认证 ===
    memory::vector<user> users;      ///< 用户列表

    // === ECH 配置（可选）===
    memory::string ech_key;          ///< ECH 密钥（base64，可叠加）

    // === Padding 配置（可选）===
    memory::string padding_scheme;   ///< Padding 方案字符串

    // === 超时配置 ===
    std::uint32_t handshake_timeout_ms{5000};        ///< 握手超时（毫秒）
    std::uint32_t idle_session_timeout_ms{30000};    ///< 空闲会话超时（毫秒）
};
```

## 配置项说明

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `server_names` | vector<string> | 是 | SNI 白名单，只有匹配的 ClientHello 才会执行认证 |
| `certificate` | string | 是 | TLS 证书文件路径（PEM 格式） |
| `private_key` | string | 是 | TLS 私钥文件路径（PEM 格式） |
| `users` | vector<user> | 是 | 用户认证列表 |
| `ech_key` | string | 否 | ECH 密钥（base64 编码），用于叠加 ECH 加密 |
| `padding_scheme` | string | 否 | Padding 方案，用于隐藏流量特征 |
| `handshake_timeout_ms` | uint32_t | 否 | 握手超时，默认 5000ms |
| `idle_session_timeout_ms` | uint32_t | 否 | 空闲会话超时，默认 30000ms |

## 配置方法

### enabled

检查配置是否有效。

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool
{
    return !server_names.empty()
        && !certificate.empty()
        && !private_key.empty()
        && !users.empty();
}
```

**要求**: `server_names` + `certificate` + `private_key` + `users` 均非空

## JSON 序列化

使用 Glaze 库进行 JSON 序列化：

```json
{
    "server_names": ["anytls.example.com"],
    "certificate": "/path/to/cert.pem",
    "private_key": "/path/to/key.pem",
    "users": [
        {"username": "user1", "password": "pass1"}
    ],
    "ech_key": "base64_encoded_ech_key",
    "padding_scheme": "random",
    "handshake_timeout_ms": 5000,
    "idle_session_timeout_ms": 30000
}
```

## ECH 叠加

AnyTLS 支持 ECH（Encrypted Client Hello）叠加：

1. 如果配置了 `ech_key`，在 Tier 1 的 `verify()` 中解密 ECH payload
2. 解密后获取真实的 inner ClientHello SNI
3. 支持隐藏真实代理域名，防止 SNI 泄露

## 与其他方案对比

| 特性 | AnyTLS | ShadowTLS | Restls |
|------|--------|-----------|--------|
| 认证位置 | 应用数据 | SessionID | 应用数据 |
| TLS 证书 | 标准 | 后端代理 | 后端代理 |
| ECH 支持 | 是 | 否 | 否 |
| 多用户 | 是 | 是 (v3) | 否 |

## 调用链

```
config::enabled() <- anytls::scheme::active()
config::server_names <- anytls::scheme::snis()
config::ech_key <- anytls::scheme::verify() -> ech::decrypt_ech_payload()
config <- anytls::scheme::handshake()
```

## 依赖

- [[core/stealth/anytls/scheme|AnyTLS 方案]] - 使用此配置
- [[core/stealth/ech/config|ECH 配置]] - ECH 密钥格式
- [[core/stealth/ech/decrypt|ECH 解密]] - ECH 解密接口