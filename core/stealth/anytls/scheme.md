---
layer: core
source: I:/code/Prism/include/prism/stealth/anytls/scheme.hpp
---

# AnyTLS 伪装方案

> 源码位置: `I:/code/Prism/include/prism/stealth/anytls/scheme.hpp`

## 概述

实现 `stealth_scheme` 接口，用于在 TLS 方案管道中处理 AnyTLS 连接。AnyTLS 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。可叠加 ECH 加密 ClientHello SNI。

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