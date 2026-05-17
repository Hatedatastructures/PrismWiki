---
layer: core
source: I:/code/Prism/include/prism/stealth/restls/scheme.hpp
---

# Restls 伪装方案

> 源码位置: `I:/code/Prism/include/prism/stealth/restls/scheme.hpp`

## 概述

实现 `stealth_scheme` 接口，用于在 TLS 方案管道中处理 Restls 连接。Restls 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。

## 命名空间

```cpp
namespace psm::stealth::restls
```

## 类定义

### scheme

Restls 伪装方案实现类，继承自 `stealth_scheme`。

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
    auto handshake(stealth::handshake_context ctx) 
        -> net::awaitable<stealth::handshake_result> override;

protected:
    [[nodiscard]] auto weight() const noexcept
        -> std::uint16_t override { return 100; }
};
```

## 方案层级

Restls 属于 **Tier 2** 方案，无 ClientHello 独占特征，依赖 SNI 匹配进行模糊检测。

| 层级 | 方法 | 说明 |
|------|------|------|
| Tier 2 | `guess()` | 基于 SNI 的模糊匹配 |

## 工作流程

Restls 通过模拟真实 TLS 流量来隐藏代理特征：

1. **读取客户端 TLS ClientHello**
2. **建立到后端 TLS 服务器的连接**（必须是 TLS 1.2 或 TLS 1.3 服务器）
3. **在 TLS 应用数据中验证客户端身份**
4. **认证成功后，使用 restls-script 控制流量模式**

## Restls Script 语法

流量控制脚本用于隐藏代理特征：

| 语法 | 说明 |
|------|------|
| `300?100` | 发送 300 字节，等待 100ms |
| `400~100` | 等待 100ms 后发送 400 字节 |
| `<1` | 等待客户端数据 |

## 认证流程

```
┌─────────────┐                    ┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │                    │   Backend   │
└──────┬──────┘                    └──────┬──────┘                    └──────┬──────┘
       │                                  │                                  │
       │  ClientHello (SNI match)         │                                  │
       │─────────────────────────────────>│                                  │
       │                                  │                                  │
       │                                  │  TLS Handshake                   │
       │                                  │─────────────────────────────────>│
       │                                  │                                  │
       │                                  │  TLS Handshake                   │
       │                                  │<─────────────────────────────────│
       │                                  │                                  │
       │  TLS Handshake (forwarded)       │                                  │
       │<─────────────────────────────────>│                                  │
       │                                  │                                  │
       │  Application Data (auth)         │                                  │
       │─────────────────────────────────>│                                  │
       │                                  │  验证认证信息                     │
       │                                  │                                  │
       │  Application Data (script ctrl)  │                                  │
       │<─────────────────────────────────>│  Relay + Script Control         │
       │                                  │─────────────────────────────────>│
```

## 调用链

```
stealth/scheme::guess -> restls::scheme::guess
stealth/scheme::handshake -> restls::scheme::handshake
```

## 依赖

- [[core/stealth/restls/config|Restls 配置]] - 服务端配置
- [[core/stealth/scheme|Stealth 方案基类]] - stealth_scheme 接口