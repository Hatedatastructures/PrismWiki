---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/config.hpp
title: ShadowTLS v3 服务端配置
tags:
  - stealth
  - shadowtls
  - config
---

# ShadowTLS v3 服务端配置

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/config.hpp`

## 概述

ShadowTLS v3 是一种 TLS 伪装协议，通过将代理流量包装成正常的 TLS 1.3 连接来对抗深度包检测。服务端在 TLS ClientHello 的 SessionID 中验证客户端身份，认证成功后透传后续 TLS 流量。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 设计决策（WHY）

### 为什么 v3 使用多用户而非 v2 的单密码

v2 的单密码意味着所有客户端共享同一认证凭据，一旦泄露所有客户端都不安全。v3 的多用户支持允许为每个客户端分配独立的用户名/密码对，便于轮换和撤销。

### 为什么 `handshake_dest` 是必需的

ShadowTLS 的核心是代理真实 TLS 握手到后端服务器。没有 `handshake_dest`，服务端无法获取真实的 ServerHello 来伪装 TLS 握手。

### 为什么 `strict_mode` 默认为 true

TLS 1.2 在某些特征上与 TLS 1.3 不同（如记录层版本号、扩展格式）。如果后端返回 TLS 1.2 的 ServerHello，代理给客户端后可能导致指纹不一致。`strict_mode=true` 确保只代理 TLS 1.3 握手。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| v2 和 v3 配置字段共存 | 兼容性 | `password` 仅 v2 使用，`users` 仅 v3 使用 |
| `handshake_dest` 格式 host:port | 解析器 | IPv6 需方括号 |
| `strict_mode=true` 时后端必须 TLS 1.3 | 握手逻辑 | TLS 1.2 后端被拒绝 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| `users` 和 `password` 都为空 | 配置不完整 | `enabled()` 返回 false |
| `handshake_dest` 不可达 | 后端服务器宕机 | 握手失败 |
| `server_names` 为空 | 未配置 SNI | `enabled()` 返回 false |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `config` → `psm::config` | 嵌入 | `cfg.stealth.shadowtls` 路径 |
| `shadowtls::scheme` → `config` | 调用 | `active()`/`snis()`/`handshake()` |
| `shadowtls::auth` → `config` | 依赖 | 使用 password/users 验证 |

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