---
title: "config — AnyTLS 伪装方案配置"
source: "include/prism/stealth/anytls/config.hpp"
module: "stealth"
submodule: "anytls"
type: api
tags: [stealth, anytls, config, 配置]
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/stealth/anytls/config.hpp`
> 模块: [[stealth|stealth]] > [[stealth/anytls|anytls]]

## 概述

AnyTLS 伪装方案配置。AnyTLS 是一种 TLS 伪装协议，使用标准 TLS 证书，通过应用层认证实现代理功能。可以叠加 ECH 加密 ClientHello SNI。

**协议参考**: https://github.com/anytls/anytls

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::string/vector PMR 容器 |
| 被依赖 | [[stealth/anytls/scheme|scheme]] | 方案使用配置 |

## 命名空间

`psm::stealth::anytls`

---

## 结构体: user

> 源码: `include/prism/stealth/anytls/config.hpp:20`

### 概述

AnyTLS 用户配置。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| username | memory::string | 用户名 |
| password | memory::string | 密码 |

---

## 结构体: config

> 源码: `include/prism/stealth/anytls/config.hpp:39`

### 概述

AnyTLS 服务端配置。包含 TLS 证书、SNI 白名单、用户认证和可选的 ECH 配置。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| server_names | memory::vector\<memory::string\> | SNI 白名单 | 空 |
| certificate | memory::string | 证书文件路径（PEM） | "" |
| private_key | memory::string | 私钥文件路径（PEM） | "" |
| users | memory::vector\<user\> | 用户列表 | 空 |
| ech_key | memory::string | ECH 密钥（base64，可叠加） | "" |
| padding_scheme | memory::string | Padding 方案字符串 | "" |
| handshake_timeout_ms | std::uint32_t | 握手超时（毫秒） | 5000 |
| idle_session_timeout_ms | std::uint32_t | 空闲会话超时（毫秒） | 30000 |

### 设计意图

- **server_names**: SNI 白名单，只有匹配的 ClientHello 才会执行认证
- **certificate**: TLS 证书文件路径（PEM 格式）
- **private_key**: TLS 私钥文件路径（PEM 格式）
- **users**: 用户认证列表
- **ech_key**: 可选的 ECH 密钥（base64 编码），用于叠加 ECH 加密
- **padding_scheme**: 可选的 padding 方案，用于隐藏流量特征
- **handshake_timeout_ms**: 握手超时时间
- **idle_session_timeout_ms**: 空闲会话超时时间

### 生命周期

1. **构造**: 由配置加载器从 `configuration.json` 解析
2. **使用**: 被 AnyTLS 方案和握手函数读取
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 配置在启动时加载，运行时只读，线程安全

### 异常安全

- 不抛异常，配置解析失败时返回默认值

---

## 函数: enabled()

> 源码: `include/prism/stealth/anytls/config.hpp:65`

### 功能

检查配置是否有效。如果 `server_names`、`certificate`、`private_key` 和 `users` 都非空，返回 `true`。

### 签名

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool;
```

### 返回值

`bool` — 配置完整返回 `true`，否则返回 `false`

### 调用链

**被调用**（向上）:
- [[stealth/anytls/scheme|active()]] — 判断方案是否启用

---

## JSON 序列化

配置文件使用 glaze 库进行 JSON 序列化/反序列化。

### user 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| username | username | string |
| password | password | string |

### config 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| server_names | server_names | vector\<string\> |
| certificate | certificate | string |
| private_key | private_key | string |
| users | users | vector\<user\> |
| ech_key | ech_key | string |
| padding_scheme | padding_scheme | string |
| handshake_timeout_ms | handshake_timeout_ms | uint32_t |
| idle_session_timeout_ms | idle_session_timeout_ms | uint32_t |

### 配置示例

```json
{
  "stealth": {
    "anytls": {
      "server_names": ["example.com", "www.example.com"],
      "certificate": "/path/to/cert.pem",
      "private_key": "/path/to/key.pem",
      "users": [
        {"username": "user1", "password": "pass1"},
        {"username": "user2", "password": "pass2"}
      ],
      "ech_key": "base64_encoded_ech_key",
      "padding_scheme": "random",
      "handshake_timeout_ms": 5000,
      "idle_session_timeout_ms": 30000
    }
  }
}
```

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — AnyTLS 基于 TLS 1.3 协议伪装

## 相关链接

- [[stealth/anytls/scheme|scheme]] — AnyTLS 方案，使用此配置
- [[stealth/ech/decrypt|decrypt]] — ECH 解密，使用 ech_key
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
