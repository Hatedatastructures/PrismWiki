---
title: "config — TrustTunnel 伪装方案配置"
source: "include/prism/stealth/trusttunnel/config.hpp"
module: "stealth"
submodule: "trusttunnel"
type: api
tags: [stealth, trusttunnel, config, 配置]
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/stealth/trusttunnel/config.hpp`
> 模块: [[stealth|stealth]] > [[stealth/trusttunnel|trusttunnel]]

## 概述

TrustTunnel 伪装方案配置。TrustTunnel 是一种支持 TCP 和 HTTP/3（QUIC）的 TLS 伪装协议。使用标准 TLS 证书，支持 BBR 拥塞控制。

**协议参考**: https://github.com/trusttunnel/trusttunnel

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::string/vector PMR 容器 |
| 被依赖 | [[stealth/trusttunnel/scheme|scheme]] | 方案使用配置 |

## 命名空间

`psm::stealth::trusttunnel`

---

## 结构体: user

> 源码: `include/prism/stealth/trusttunnel/config.hpp:20`

### 概述

TrustTunnel 用户配置。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| username | memory::string | 用户名 |
| password | memory::string | 密码 |

---

## 枚举: network_type

> 源码: `include/prism/stealth/trusttunnel/config.hpp:30`

### 概述

传输网络类型枚举。

### 枚举值

| 值 | 说明 |
|-----|------|
| tcp | HTTP/2 (TCP) |
| udp | HTTP/3 (QUIC) |
| both | 同时支持 TCP 和 UDP |

---

## 枚举: congestion_controller

> 源码: `include/prism/stealth/trusttunnel/config.hpp:41`

### 概述

拥塞控制算法枚举。

### 枚举值

| 值 | 说明 |
|-----|------|
| cubic | CUBIC 拥塞控制 |
| bbr | BBR 拥塞控制 |
| new_reno | New Reno 拥塞控制 |

---

## 结构体: config

> 源码: `include/prism/stealth/trusttunnel/config.hpp:61`

### 概述

TrustTunnel 服务端配置。包含 TLS 证书、SNI 白名单、用户认证和网络配置。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| server_names | memory::vector\<memory::string\> | SNI 白名单 | 空 |
| certificate | memory::string | 证书文件路径（PEM） | "" |
| private_key | memory::string | 私钥文件路径（PEM） | "" |
| users | memory::vector\<user\> | 用户列表 | 空 |
| network | network_type | 传输网络类型 | both |
| congestion | congestion_controller | 拥塞控制算法 | bbr |
| handshake_timeout_ms | std::uint32_t | 握手超时（毫秒） | 5000 |
| idle_timeout_ms | std::uint32_t | 空闲超时（毫秒） | 30000 |

### 设计意图

- **server_names**: SNI 白名单，只有匹配的 ClientHello 才会执行认证
- **certificate**: TLS 证书文件路径（PEM 格式）
- **private_key**: TLS 私钥文件路径（PEM 格式）
- **users**: 用户认证列表
- **network**: 传输网络类型，支持 TCP、UDP 或同时支持
- **congestion**: 拥塞控制算法，支持 cubic、bbr、new_reno
- **handshake_timeout_ms**: 握手超时时间
- **idle_timeout_ms**: 空闲超时时间

### 生命周期

1. **构造**: 由配置加载器从 `configuration.json` 解析
2. **使用**: 被 TrustTunnel 方案和握手函数读取
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 配置在启动时加载，运行时只读，线程安全

### 异常安全

- 不抛异常，配置解析失败时返回默认值

---

## 函数: enabled()

> 源码: `include/prism/stealth/trusttunnel/config.hpp:85`

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
- [[stealth/trusttunnel/scheme|active()]] — 判断方案是否启用

---

## JSON 序列化

配置文件使用 glaze 库进行 JSON 序列化/反序列化。

### user 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| username | username | string |
| password | password | string |

### network_type 枚举映射

| JSON 值 | 枚举值 |
|---------|--------|
| "tcp" | network_type::tcp |
| "udp" | network_type::udp |
| "both" | network_type::both |

### congestion_controller 枚举映射

| JSON 值 | 枚举值 |
|---------|--------|
| "cubic" | congestion_controller::cubic |
| "bbr" | congestion_controller::bbr |
| "new_reno" | congestion_controller::new_reno |

### config 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| server_names | server_names | vector\<string\> |
| certificate | certificate | string |
| private_key | private_key | string |
| users | users | vector\<user\> |
| network | network | network_type |
| congestion | congestion | congestion_controller |
| handshake_timeout_ms | handshake_timeout_ms | uint32_t |
| idle_timeout_ms | idle_timeout_ms | uint32_t |

### 配置示例

```json
{
  "stealth": {
    "trusttunnel": {
      "server_names": ["example.com", "www.example.com"],
      "certificate": "/path/to/cert.pem",
      "private_key": "/path/to/key.pem",
      "users": [
        {"username": "user1", "password": "pass1"},
        {"username": "user2", "password": "pass2"}
      ],
      "network": "both",
      "congestion": "bbr",
      "handshake_timeout_ms": 5000,
      "idle_timeout_ms": 30000
    }
  }
}
```

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TrustTunnel 基于 TLS 1.3 协议伪装
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — TrustTunnel 支持 TCP/QUIC 双栈传输

## 相关链接

- [[stealth/trusttunnel/scheme|scheme]] — TrustTunnel 方案，使用此配置
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — TCP/UDP 传输选择
