---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/config.hpp
title: ShadowTLS v3 服务端配置
---

# ShadowTLS v3 服务端配置

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/config.hpp`

## 概述

ShadowTLS v3 是一种 TLS 伪装协议，通过将代理流量包装成正常的 TLS 1.3 连接来对抗深度包检测。服务端在 TLS ClientHello 的 SessionID 中验证客户端身份，认证成功后透传后续 TLS 流量。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 结构定义

### user

ShadowTLS v3 用户配置。

```cpp
struct user
{
    std::string name;     ///< 用户名称
    std::string password; ///< 认证密码
};
```

v3 版本支持多用户认证，每个用户有独立的名称和密码。

### config

ShadowTLS v3 服务端配置。

```cpp
struct config
{
    int version{3};                                  ///< 协议版本 (2 或 3)
    std::string password;                            ///< v2 兼容密码
    std::vector<user> users;                         ///< v3 多用户
    std::string handshake_dest;                      ///< 握手后端目标 host:port
    std::vector<std::string> server_names;           ///< SNI 白名单
    bool strict_mode{true};                          ///< 严格模式：仅 TLS 1.3
    std::uint32_t handshake_timeout_ms{5000};        ///< 握手超时（毫秒）
};
```

## 配置项说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | int | 协议版本，支持 2 或 3 |
| `password` | string | v2 兼容密码（仅 v2 模式使用） |
| `users` | vector<user> | v3 多用户列表 |
| `handshake_dest` | string | 握手后端目标服务器地址（host:port） |
| `server_names` | vector<string> | SNI 白名单，只有匹配的 ClientHello 才会执行认证 |
| `strict_mode` | bool | 严格模式，默认仅接受 TLS 1.3 |
| `handshake_timeout_ms` | uint32_t | 握手超时时间，默认 5000ms |

## 配置方法

### enabled

检查配置是否有效。

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool
{
    if (version == 3)
        return !users.empty() && !handshake_dest.empty() && !server_names.empty();
    return !password.empty() && !handshake_dest.empty() && !server_names.empty();
}
```

**v3 模式要求**: `users` + `handshake_dest` + `server_names` 均非空
**v2 模式要求**: `password` + `handshake_dest` + `server_names` 均非空

## JSON 序列化

使用 Glaze 库进行 JSON 序列化：

```json
{
    "version": 3,
    "password": "",
    "users": [
        {"name": "user1", "password": "pass1"}
    ],
    "handshake_dest": "www.example.com:443",
    "server_names": ["shadow.example.com"],
    "strict_mode": true,
    "handshake_timeout_ms": 5000
}
```

## 版本差异

| 特性 | v2 | v3 |
|------|----|----|
| 用户认证 | 单密码 | 多用户 |
| SessionID HMAC | 4 字节 | 4 字节 |
| 数据帧处理 | 无 | HMAC 验证 + XOR 加密 |
| 兼容性 | 广泛 | 推荐 |

## 调用链

```
config::enabled() <- shadowtls::scheme::active()
config::server_names <- shadowtls::scheme::snis()
config <- shadowtls::handshake::handshake()
```

## 依赖

- [[core/stealth/shadowtls/scheme|ShadowTLS 方案]] - 使用此配置
- [[core/stealth/shadowtls/handshake|ShadowTLS 握手]] - 使用此配置