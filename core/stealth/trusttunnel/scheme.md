---
layer: core
source: I:/code/Prism/include/prism/stealth/trusttunnel/scheme.hpp
title: TrustTunnel 伪装方案
---

# TrustTunnel 伪装方案

> 源码位置: `I:/code/Prism/include/prism/stealth/trusttunnel/scheme.hpp`

## 概述

实现 `stealth_scheme` 接口，用于在 TLS 方案管道中处理 TrustTunnel 连接。TrustTunnel 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。支持 TCP（HTTP/2）和 UDP（HTTP/3/QUIC）两种传输模式。

## 命名空间

```cpp
namespace psm::stealth::trusttunnel
```

## 类定义

### scheme

TrustTunnel 伪装方案实现类，继承自 `stealth_scheme`。

```cpp
class scheme final : public stealth_scheme
{
public:
    // === 基本信息 ===
    [[nodiscard]] auto name() const noexcept
        -> std::string_view override;
    [[nodiscard]] auto tier() const noexcept
        -> std::uint8_t override { return 2; }
    [[nodiscard]] auto unique() const noexcept
        -> bool override { return false; }

    // === 配置检查 ===
    [[nodiscard]] auto active(const psm::config &cfg) const noexcept
        -> bool override;
    [[nodiscard]] auto snis(const psm::config &cfg) const
        -> memory::vector<memory::string> override;

    // === Tier 2: 模糊检测 ===
    [[nodiscard]] auto guess(const psm::config &cfg) const
        -> verify_result override;

    // === 执行 ===
    [[nodiscard]] auto handshake(stealth::handshake_context ctx)
        -> net::awaitable<stealth::handshake_result> override;

protected:
    [[nodiscard]] auto weight() const noexcept
        -> std::uint16_t override { return 100; }
};
```

## 方案层级

TrustTunnel 属于 **Tier 2** 方案，无 ClientHello 独占特征，依赖 SNI 匹配。

| 层级 | 方法 | 说明 |
|------|------|------|
| Tier 2 | `guess()` | 基于 SNI 的模糊匹配 |

## 工作流程

TrustTunnel 使用标准 TLS 证书，支持 TCP 和 UDP 传输：

1. **执行标准 TLS 握手**（使用配置的证书）
2. **读取 TLS 应用数据**（客户端首帧）
3. **验证用户身份**
4. **根据网络配置选择传输模式**
5. **认证成功后检测内层协议**

## 传输模式

| 模式 | 协议 | 说明 |
|------|------|------|
| `tcp` | HTTP/2 | TCP 传输 |
| `udp` | HTTP/3 (QUIC) | UDP 传输 |
| `both` | HTTP/2 + HTTP/3 | 同时支持 TCP 和 UDP |

## 拥塞控制

支持多种拥塞控制算法：

| 算法 | 说明 |
|------|------|
| `cubic` | 经典 TCP 拥塞控制 |
| `bbr` | Google BBR，推荐 |
| `new_reno` | 传统 Reno 改进版 |

## 认证流程

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │  ClientHello (SNI match)         │
       │─────────────────────────────────>│
       │                                  │
       │  TLS/QUIC Handshake              │
       │<─────────────────────────────────>│
       │                                  │
       │  HTTP/2 or HTTP/3 Connection     │
       │<─────────────────────────────────>│
       │                                  │
       │  Auth Frame (username/password)  │
       │─────────────────────────────────>│
       │                                  │  验证用户身份
       │                                  │
       │  Proxy Traffic                   │
       │<─────────────────────────────────>│
```

## 调用链

```
stealth/scheme::guess -> trusttunnel::scheme::guess
stealth/scheme::handshake -> trusttunnel::scheme::handshake
```

## 依赖

- [[core/stealth/trusttunnel/config|TrustTunnel 配置]] - 服务端配置
- [[core/stealth/scheme|Stealth 方案基类]] - stealth_scheme 接口