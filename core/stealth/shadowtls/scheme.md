---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/scheme.hpp
title: ShadowTLS v3 伪装方案
tags:
  - stealth
  - shadowtls
  - tier1
  - HMAC
  - scheme
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

## 设计决策（WHY）

### 为什么 ShadowTLS 是 Tier 1 而非 Tier 0

ShadowTLS 的 SessionID HMAC 是确定性特征——如果密码正确，HMAC 匹配是确定性的。但 HMAC-SHA1 计算是有成本的（相对于零成本的字节比较），因此归为 Tier 1。Reality 的 `[01:08:02]` 标记是纯字节比较，不需要任何计算，归为 Tier 0。

### 为什么 `verify()` 需要原始 ClientHello 字节

HMAC 计算需要 ClientHello 的原始字节（不是解析后的结构），因为 HMAC 输入包含 `ClientHello[10:hmac_index] + 00000000 + ClientHello[hmac_index+4:]`。如果只传解析后的结构，原始字节中的精确偏移信息会丢失。

### 为什么 ShadowTLS 需要连接后端服务器

ShadowTLS 的伪装原理是**代理真实 TLS 握手**。服务端将客户端的 ClientHello 转发给真实 TLS 服务器，获得真实的 ServerHello，再返回给客户端。这样在中间人看来，连接完全是一个到后端服务器的正常 TLS 握手。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| SessionID 必须恰好 32 字节 | ShadowTLS 协议 | 过短或过长都无法嵌入 HMAC |
| `verify()` 需要 `raw` 完整 ClientHello | HMAC 偏移计算 | 缺少 TLS 记录头导致偏移错误 |
| 后端服务器必须支持 TLS 1.3（strict_mode） | 配置选项 | TLS 1.2 后端在 strict_mode 下被拒绝 |
| HMAC 标签仅 4 字节 | ShadowTLS v3 设计 | 碰撞概率 1/2^32，安全但非极高 |

## 失败场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| SessionID 长度不等于 32 | `sniff()` 阶段 | `hit=false`，不进入 verify |
| HMAC 不匹配 | 密码错误 | `score=0`，不作为候选 |
| 后端服务器不可达 | `handshake_dest` 无效 | `handshake()` 失败 |
| 后端返回 TLS 1.2 | strict_mode=true | 握手拒绝 |
| 写入 ClientHello 到后端后失败 | 已向网络写入 | `polluted=true`，不可 rewind |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `scheme` → `config` | 依赖 | `active()` 检查 `enabled()`，`snis()` 读取 `server_names` |
| `scheme` → `auth` | 调用 | `verify()` 调用 `verify_client_hello()` |
| `scheme` → `handshake` | 调用 | `handshake()` 委托给 shadowtls::handshake |
| `scheme` → `recognition::tls` | 依赖 | `sniff()` 检查 session_id 长度特征 |

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