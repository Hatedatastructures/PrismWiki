---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/scheme.hpp
title: ShadowTLS v3 伪装方案
---

# ShadowTLS v3 伪装方案

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/scheme.hpp`

## 概述

封装 ShadowTLS 握手和协议检测逻辑，继承 `stealth_scheme` 基类。ShadowTLS 是 Tier 1 方案，需要 HMAC 验证确认身份。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 类定义

### scheme

ShadowTLS v3 伪装方案实现类，继承自 `stealth_scheme`。

```cpp
class scheme final : public stealth_scheme
{
public:
    // === 基本信息 ===
    [[nodiscard]] auto name() const noexcept
        -> std::string_view override;
    [[nodiscard]] auto tier() const noexcept
        -> std::uint8_t override { return 1; }
    [[nodiscard]] auto unique() const noexcept
        -> bool override { return false; }

    // === 配置检查 ===
    [[nodiscard]] auto active(const psm::config &cfg) const noexcept
        -> bool override;
    [[nodiscard]] auto snis(const psm::config &cfg) const
        -> memory::vector<memory::string> override;

    // === Tier 0: 快速检测 ===
    [[nodiscard]] auto sniff(std::uint32_t bitmap,
                              const protocol::tls::client_hello_features &features) const
        -> sniff_result override;

    // === Tier 1: 详细检测（HMAC 验证）===
    [[nodiscard]] auto verify(const protocol::tls::client_hello_features &features,
                               std::span<const std::byte> raw,
                               const psm::config &cfg) const
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

ShadowTLS 属于 **Tier 1** 方案，具备 ClientHello 独占特征（SessionID HMAC）。

| 层级 | 方法 | 说明 |
|------|------|------|
| Tier 0 | `sniff()` | 快速特征检测 |
| Tier 1 | `verify()` | HMAC 验证确认身份 |

## 工作流程

1. **sniff()**: 检查 SessionID 长度是否为 32 字节（ShadowTLS 要求）
2. **verify()**: 验证 SessionID 后 4 字节的 HMAC-SHA1 标签
3. **handshake()**: 与后端服务器完成 TLS 握手，处理数据帧的 HMAC 验证和 XOR 解密

## 认证机制

ShadowTLS v3 在 TLS ClientHello 的 SessionID 字段中嵌入 4 字节 HMAC 标签：

```
SessionID[28:32] = HMAC-SHA1(password, ClientHello[10:hmac_index] + 00000000 + ClientHello[hmac_index+4:])[:4]
```

## 调用链

```
stealth/scheme::sniff -> shadowtls::scheme::sniff
stealth/scheme::verify -> shadowtls::scheme::verify -> shadowtls::auth::verify_client_hello
stealth/scheme::handshake -> shadowtls::scheme::handshake -> shadowtls::handshake::handshake
```

## 依赖

- [[core/stealth/shadowtls/config|ShadowTLS 配置]] - 服务端配置
- [[core/stealth/shadowtls/auth|ShadowTLS 认证]] - HMAC 验证逻辑
- [[core/stealth/shadowtls/handshake|ShadowTLS 握手]] - 服务端握手流程
- [[core/stealth/shadowtls/constants|ShadowTLS 常量]] - 协议常量
- [[core/protocol/tls/types|TLS 类型]] - ClientHello 特征结构