---
tags: [stealth, trusttunnel, config]
layer: core
source: I:/code/Prism/include/prism/stealth/trusttunnel/config.hpp
title: TrustTunnel 伪装方案配置
---

# TrustTunnel 伪装方案配置

> 源码位置: `I:/code/Prism/include/prism/stealth/trusttunnel/config.hpp`

## 设计决策（WHY）

### 为什么 `network_type` 默认为 `both`

同时支持 TCP 和 UDP 可以覆盖更多客户端场景。HTTP/2 (TCP) 作为基线保证兼容性，HTTP/3 (QUIC/UDP) 在支持的客户端上提供更好的性能。

### 为什么 `congestion` 默认为 `bbr`

BBR 是 Google 开发的拥塞控制算法，在高带宽长距离网络（如跨国代理）上比 Cubic 好得多。作为代理服务器，这个场景是典型使用场景。

### 为什么 `users` 参与但 `short_ids` 不存在于 TrustTunnel

TrustTunnel 使用标准的 username/password 认证（类似 HTTP Basic Auth），而非 ShadowTLS 的 HMAC 或 Reality 的 short_id。这是因为认证发生在 TLS 隧道内部，外层已有加密保护，无需复杂的密码学认证。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 证书和私钥必须配对 | TLS 握手 | 不匹配会导致握手失败 |
| users 列表不能为空 | `enabled()` 检查 | 无用户 = 未启用 |
| network_type::udp 需要 QUIC 库支持 | 传输依赖 | 无 QUIC 库则 UDP 模式不可用 |
| handshake_timeout_ms 和 idle_timeout_ms 单位为毫秒 | 配置格式 | 0 表示无限等待（危险） |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 证书文件不存在 | 路径配置错误 | TLS 握手失败 |
| 私钥与证书不匹配 | 配置失误 | TLS 握手失败 |
| users 为空 | 未配置用户 | `enabled()` 返回 false |
| 空壳状态 | 当前实现不完整 | 方案不可用 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `config` → `psm::config` | 嵌入 | `cfg.stealth.trusttunnel` 路径 |
| `trusttunnel::scheme` → `config` | 调用 | `active()`/`snis()` |

## 概述

TrustTunnel 是一种支持 TCP 和 HTTP/3（QUIC）的 TLS 伪装协议。使用标准 TLS 证书，支持 BBR 拥塞控制。

协议参考: https://github.com/trusttunnel/trusttunnel

## 命名空间

```cpp
namespace psm::stealth::trusttunnel
```

## 结构定义

### user

TrustTunnel 用户配置。

```cpp
struct user
{
    memory::string username;  ///< 用户名
    memory::string password;  ///< 密码
};
```

### network_type

传输网络类型枚举。

```cpp
enum class network_type : std::uint8_t
{
    tcp,    ///< HTTP/2 (TCP)
    udp,    ///< HTTP/3 (QUIC)
    both    ///< 同时支持 TCP 和 UDP
};
```

### congestion_controller

拥塞控制算法枚举。

```cpp
enum class congestion_controller : std::uint8_t
{
    cubic,
    bbr,
    new_reno
};
```

### config

TrustTunnel 服务端配置。

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

    // === 网络配置 ===
    network_type network{network_type::both};              ///< 传输网络类型
    congestion_controller congestion{congestion_controller::bbr}; ///< 拥塞控制

    // === 超时配置 ===
    std::uint32_t handshake_timeout_ms{5000};  ///< 握手超时（毫秒）
    std::uint32_t idle_timeout_ms{30000};      ///< 空闲超时（毫秒）
};
```

## 配置项说明

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `server_names` | vector<string> | 是 | SNI 白名单 |
| `certificate` | string | 是 | TLS 证书文件路径（PEM 格式） |
| `private_key` | string | 是 | TLS 私钥文件路径（PEM 格式） |
| `users` | vector<user> | 是 | 用户认证列表 |
| `network` | network_type | 否 | 传输网络类型，默认 `both` |
| `congestion` | congestion_controller | 否 | 拥塞控制算法，默认 `bbr` |
| `handshake_timeout_ms` | uint32_t | 否 | 握手超时，默认 5000ms |
| `idle_timeout_ms` | uint32_t | 否 | 空闲超时，默认 30000ms |

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
    "server_names": ["trust.example.com"],
    "certificate": "/path/to/cert.pem",
    "private_key": "/path/to/key.pem",
    "users": [
        {"username": "user1", "password": "pass1"}
    ],
    "network": "both",
    "congestion": "bbr",
    "handshake_timeout_ms": 5000,
    "idle_timeout_ms": 30000
}
```

## 网络类型详解

| 值 | 说明 | 协议栈 |
|----|------|--------|
| `tcp` | 仅 TCP | TLS 1.3 + HTTP/2 |
| `udp` | 仅 UDP | QUIC + HTTP/3 |
| `both` | 双栈 | TLS 1.3 + HTTP/2 + QUIC + HTTP/3 |

## 拥塞控制算法

| 算法 | 特点 | 推荐场景 |
|------|------|----------|
| `cubic` | 经典 TCP 友好 | 传统网络环境 |
| `bbr` | 高吞吐低延迟 | 推荐，现代网络 |
| `new_reno` | 简单可靠 | 兼容性要求 |

## 与其他方案对比

| 特性 | TrustTunnel | AnyTLS | ShadowTLS |
|------|-------------|--------|-----------|
| TCP 支持 | HTTP/2 | TLS | TLS |
| UDP 支持 | HTTP/3 | 无 | 无 |
| 拥塞控制 | 可配置 | 无 | 无 |
| 多用户 | 是 | 是 | 是 (v3) |

## 调用链

```
config::enabled() <- trusttunnel::scheme::active()
config::server_names <- trusttunnel::scheme::snis()
config <- trusttunnel::scheme::handshake()
```

## 依赖

- [[core/stealth/trusttunnel/scheme|TrustTunnel 方案]] - 使用此配置