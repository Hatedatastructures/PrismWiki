---
title: "config — ShadowTLS v3 服务端配置"
source: "include/prism/stealth/shadowtls/config.hpp"
module: "stealth"
submodule: "shadowtls"
type: api
tags: [stealth, shadowtls, config, 配置]
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/stealth/shadowtls/config.hpp`
> 模块: [[stealth|stealth]] > [[stealth/shadowtls|shadowtls]]

## 概述

ShadowTLS v3 服务端配置。包含用户列表、握手目标、SNI 白名单和严格模式。ShadowTLS v3 是一种 TLS 伪装协议，通过将代理流量包装成正常的 TLS 1.3 连接来对抗深度包检测。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[stealth/shadowtls/scheme|scheme]] | 方案使用配置 |
| 被依赖 | [[stealth/shadowtls/handshake|handshake]] | 握手使用配置 |
| 被依赖 | [[stealth/shadowtls/auth|auth]] | 认证使用配置 |

## 命名空间

`psm::stealth::shadowtls`

---

## 结构体: user

> 源码: `include/prism/stealth/shadowtls/config.hpp:21`

### 概述

ShadowTLS v3 用户配置。v3 支持多用户认证，每个用户有独立的名称和密码。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| name | std::string | 用户名称 |
| password | std::string | 认证密码 |

---

## 结构体: config

> 源码: `include/prism/stealth/shadowtls/config.hpp:34`

### 概述

ShadowTLS v3 服务端配置。包含用户列表、握手目标、SNI 白名单和严格模式。v3 版本必须配置至少一个用户，v2 兼容模式使用单一 `password`。`server_names` 为 SNI 白名单，只有匹配的 ClientHello 才会执行认证。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| version | int | 协议版本 (2 或 3) | 3 |
| password | std::string | v2 兼容密码 | "" |
| users | std::vector\<user\> | v3 多用户 | 空 |
| handshake_dest | std::string | 握手后端目标 host:port | "" |
| server_names | std::vector\<std::string\> | SNI 白名单 | 空 |
| strict_mode | bool | 严格模式：仅 TLS 1.3 | true |
| handshake_timeout_ms | std::uint32_t | 握手超时（毫秒） | 5000 |

### 设计意图

- **version**: 支持 v2 和 v3 两个版本，v3 支持多用户认证
- **password**: v2 兼容模式使用单一密码
- **users**: v3 支持多用户，每个用户有独立的名称和密码
- **handshake_dest**: 握手后端目标，格式为 `host:port`
- **server_names**: SNI 白名单，只有匹配的 ClientHello 才会执行认证
- **strict_mode**: 严格模式下只接受 TLS 1.3 连接
- **handshake_timeout_ms**: 握手超时时间，防止长时间阻塞

### 生命周期

1. **构造**: 由配置加载器从 `configuration.json` 解析
2. **使用**: 被 ShadowTLS 方案和握手函数读取
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 配置在启动时加载，运行时只读，线程安全

### 异常安全

- 不抛异常，配置解析失败时返回默认值

---

## 函数: enabled()

> 源码: `include/prism/stealth/shadowtls/config.hpp:50`

### 功能

检查配置是否启用。v3 需要 `users` + `handshake_dest` + `server_names` 都非空；v2 需要 `password` + `handshake_dest` + `server_names` 都非空。

### 签名

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool;
```

### 返回值

`bool` — 配置完整返回 `true`，否则返回 `false`

### 调用链

**被调用**（向上）:
- [[stealth/shadowtls/scheme|active()]] — 判断方案是否启用

### 注意事项

- v3 必须配置至少一个用户
- v2 必须配置 password
- handshake_dest 和 server_names 必须非空

---

## JSON 序列化

配置文件使用 glaze 库进行 JSON 序列化/反序列化。

### user 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| name | name | string |
| password | password | string |

### config 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| version | version | int |
| password | password | string |
| users | users | vector\<user\> |
| handshake_dest | handshake_dest | string |
| server_names | server_names | vector\<string\> |
| strict_mode | strict_mode | bool |
| handshake_timeout_ms | handshake_timeout_ms | uint32_t |

### 配置示例

```json
{
  "stealth": {
    "shadowtls": {
      "version": 3,
      "users": [
        {"name": "user1", "password": "pass1"},
        {"name": "user2", "password": "pass2"}
      ],
      "handshake_dest": "example.com:443",
      "server_names": ["example.com", "www.example.com"],
      "strict_mode": true,
      "handshake_timeout_ms": 5000
    }
  }
}
```

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — ShadowTLS 基于 TLS 1.3 协议伪装
- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — ShadowTLS 使用 HMAC-SHA1 进行认证
