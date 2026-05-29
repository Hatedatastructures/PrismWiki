---
tags: [stealth, anytls, scheme]
layer: core
source: include/prism/stealth/anytls/scheme.hpp
title: AnyTLS 伪装方案
---

# AnyTLS 伪装方案

> 源码位置: `include/prism/stealth/anytls/scheme.hpp`

## 概述

实现 `stealth_scheme` 接口，用于在 TLS 方案管道中处理 AnyTLS 连接。AnyTLS 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。可叠加 ECH 加密 ClientHello SNI。

## 设计决策（WHY）

### 为什么 AnyTLS 是 Tier 2 但有 verify() 实现

AnyTLS 的 `verify()` 仅在启用 ECH 时生效（检查 ClientHello 中是否有 ECH 扩展）。没有 ECH 时，AnyTLS 完全依赖 SNI 匹配（Tier 2 guess）。这种"有条件 Tier 1"的设计让 AnyTLS 在 ECH 启用时获得更高的检测优先级。

### 为什么 AnyTLS 支持内部多路复用

AnyTLS 在 TLS 隧道内部实现了自己的多路复用协议（frame/session/stream_transport 层）。这允许单个 TLS 连接上承载多个代理流，减少连接数和握手开销。其他伪装方案（如 Reality、ShadowTLS）不支持内部多路复用。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| ECH 解密当前未实现 | `decrypt.cpp` 返回 `not_supported` | ECH 相关检测不可用 |
| 内部多路复用需要 frame 协议 | mux 层 | 增加了复杂性 |
| 标准 TLS 证书必须配置 | 与 Reality 不同 | 不使用合成证书 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| ECH 解密失败 | ECH 启用但未实现 | 返回 `not_supported` |
| SNI 不匹配 | 无 ECH 且 SNI 不在白名单 | 跳过 AnyTLS |
| TLS 握手失败 | 证书无效 | 连接中断 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `scheme` → `config` | 依赖 | `active()`/`snis()` 读取配置 |
| `scheme` → `ech` | 调用 | ECH 解密检测 |
| `scheme` → `anytls::session` | 创建 | 握手成功后创建内部多路复用会话 |

## 命名空间

```cpp
namespace psm::stealth::anytls
```

## 类定义

### scheme

AnyTLS 伪装方案实现类，继承自 `stealth_scheme`。

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

    // === Tier 1: 详细检测（ECH 解密）===
    [[nodiscard]] auto verify(const protocol::tls::client_hello_features &features,
                               std::span<const std::byte> raw,
                               const psm::config &cfg) const
        -> verify_result override;

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

AnyTLS 属于 **Tier 2** 方案，无 ClientHello 独占特征，但支持 ECH 叠加。

| 层级 | 方法 | 说明 |
|------|------|------|
| Tier 1 | `verify()` | ECH 解密（如配置了 ech_key） |
| Tier 2 | `guess()` | 基于 SNI 的模糊匹配 |

## 工作流程

AnyTLS 使用标准 TLS 证书，通过应用层认证实现代理功能：

1. **执行标准 TLS 握手**（使用配置的证书）
2. **读取 TLS 应用数据**（客户端首帧）
3. **解析 AnyTLS 认证帧，验证用户身份**
4. **认证成功后，检测内层协议**

## ECH 支持

如果配置了 `ech_key`，AnyTLS 可以叠加 ECH 加密：

- ECH 解密在 Tier 1 的 `verify()` 中执行
- 解密后获取真实的 inner ClientHello SNI
- 支持隐藏真实代理域名

## 认证流程

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │  ClientHello (SNI match / ECH)  │
       │─────────────────────────────────>│
       │                                  │
       │  ServerHello + Certificate + ... │
       │<─────────────────────────────────│
       │                                  │
       │  TLS Finished                    │
       │─────────────────────────────────>│
       │                                  │
       │  Application Data (Auth Frame)   │
       │─────────────────────────────────>│
       │                                  │  验证用户身份
       │                                  │
       │  Application Data (Response)     │
       │<─────────────────────────────────│
       │                                  │
       │  Proxy Traffic                   │
       │<─────────────────────────────────>│
```

## 调用链

```
stealth/scheme::verify -> anytls::scheme::verify -> ech::decrypt_ech_payload (if ech_key set)
stealth/scheme::guess -> anytls::scheme::guess
stealth/scheme::handshake -> anytls::scheme::handshake
```

## 依赖

- [[core/stealth/anytls/config|AnyTLS 配置]] - 服务端配置
- [[core/stealth/ech/decrypt|ECH 解密]] - ECH 解密接口
- [[core/stealth/scheme|Stealth 方案基类]] - stealth_scheme 接口