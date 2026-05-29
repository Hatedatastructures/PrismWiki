---
tags: [stealth, restls, scheme]
layer: core
source: include/prism/stealth/restls/scheme.hpp
title: Restls 伪装方案
---

# Restls 伪装方案

> 源码位置: `include/prism/stealth/restls/scheme.hpp`

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

## 设计决策（WHY）

### 为什么 Restls 是 Tier 2 而非 Tier 1

Restls 的认证信息嵌入在 TLS 应用数据中（而非 ClientHello），无法在 `verify()` 阶段（只有 ClientHello 数据）进行检测。只有 SNI 匹配可以作为检测依据，因此归为 Tier 2。

### 为什么 Restls 支持 TLS 1.2 和 TLS 1.3

与 ShadowTLS 不同，Restls 不代理完整的 TLS 握手。它连接后端 TLS 服务器获取 ServerHello 中的 `server_random`，然后用 `server_random` 和密码派生认证密钥。TLS 1.2 服务器也提供 `server_random`，因此兼容。

### 为什么使用 BLAKE3 而非 HMAC-SHA1

Restls 使用 BLAKE3 keyed hash 作为 MAC 函数。BLAKE3 在硬件上比 SHA1 快得多（支持 SIMD 并行），且安全性更强（SHA1 已被破解用于碰撞攻击）。BLAKE3 的 keyed mode 天然适合 MAC 用途，不需要 HMAC 的嵌套构造。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 后端必须是 TLS 1.2 或 1.3 服务器 | 握手依赖 `server_random` | 非 TLS 服务器无法提供 |
| SNI 匹配是唯一检测依据 | Tier 2 | 多个方案可能共享 SNI |
| 握手后 `polluted=true` | ServerHello 转发 | 不可 rewind |
| 单密码认证 | 配置结构 | 无多用户支持 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 后端不可达 | `host` 配置错误 | 握手失败 |
| SNI 不匹配 | 客户端 SNI 不在白名单 | 跳过 Restls |
| 密码错误 | BLAKE3 认证 MAC 不匹配 | 认证失败 |
| `version_hint` 与实际不符 | TLS 1.2 后端但 hint 为 tls13 | XOR 偏移错误，认证必然失败 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `scheme` → `config` | 依赖 | `active()`/`snis()`/`handshake()` 读取配置 |
| `scheme` → `restls::handshake` | 调用 | `handshake()` 委托给 restls::handshake |
| `scheme` → `restls::crypto` | 间接 | 握手中使用 BLAKE3 密钥派生 |

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