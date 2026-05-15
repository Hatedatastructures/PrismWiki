---
title: "scheme — AnyTLS 伪装方案"
source: "include/prism/stealth/anytls/scheme.hpp"
module: "stealth"
submodule: "anytls"
type: api
tags: [stealth, anytls, scheme, 方案, tier2, ech]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/anytls/config|config]]"
  - "[[stealth/ech/decrypt|decrypt]]"
  - "[[stealth/scheme|scheme 基类]]"
  - "[[stealth/executor|executor]]"
  - "[[stealth/registry|registry]]"
---

# scheme.hpp

> 源码: `include/prism/stealth/anytls/scheme.hpp`
> 实现: `src/prism/stealth/anytls/scheme.cpp`
> 模块: [[stealth|stealth]] > [[stealth/anytls|anytls]]

## 概述

AnyTLS 伪装方案类。实现 [[stealth/scheme|stealth_scheme]] 接口，用于在 TLS 方案管道中处理 AnyTLS 连接。AnyTLS 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。可叠加 [[stealth/ech/decrypt|ECH]] 加密 ClientHello SNI。

**当前状态**：基础框架已实现，认证逻辑待完善。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 继承 stealth_scheme 基类 |
| 依赖 | [[stealth/anytls/config|config]] | 使用 AnyTLS 配置 |
| 依赖 | [[stealth/ech/decrypt|decrypt]] | 使用 ECH 解密函数 |
| 依赖 | [[agent/config|config]] | 使用 psm::config 配置 |
| 依赖 | [[protocol/tls/feature_bitmap|feature_bitmap]] | 使用特征位图检查 ECH 扩展 |
| 依赖 | [[fault/handling|handling]] | 错误处理 |
| 被依赖 | [[stealth/registry|registry]] | 注册到方案注册表 |
| 被依赖 | [[stealth/executor|executor]] | 被执行器调用 |

## 命名空间

`psm::stealth::anytls`

---

## 类: scheme

> 源码: `include/prism/stealth/anytls/scheme.hpp:31`

### 概述

AnyTLS 伪装方案实现。AnyTLS 使用标准 TLS 证书，通过应用层认证实现代理功能。服务端在 TLS 握手完成后，从 TLS 应用数据中读取认证信息。

### 类层次

```
stealth_scheme [[stealth/scheme|scheme]]
  └── anytls::scheme
```

### 设计意图

AnyTLS 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。工作流程：
1. 执行标准 TLS 握手（使用配置的证书）
2. 读取 TLS 应用数据（客户端首帧）
3. 解析 AnyTLS 认证帧格式: `[password_length:2][password:N][padding:variable]`
4. 验证用户身份
5. 认证成功后，检测内层协议

**ECH 支持**：
- 如果配置了 `ech_key`，可以叠加 ECH 加密
- ECH 解密在 Tier 1 的 `verify` 中执行

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 返回 "anytls" | 下文 |
| tier() | 返回 2（Tier 2） | 下文 |
| unique() | 返回 false（无独占特征） | 下文 |
| active() | 判断方案是否启用 | 下文 |
| snis() | 获取 SNI 白名单 | 下文 |
| verify() | Tier 1 详细检测（ECH 解密） | 下文 |
| guess() | Tier 2 模糊检测 | 下文 |
| handshake() | 执行握手 | 下文 |
| weight() | 返回权重分 100 | 下文 |

### 生命周期

1. **构造**: 由 `registry` 在启动时创建
2. **使用**: 被 `executor` 调用，执行检测和握手
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 所有检测函数是线程安全的，只读取配置
- `handshake()` 涉及 I/O 操作，需要在协程中调用

### 异常安全

- 所有函数不抛异常，错误通过返回值报告

---

## 函数: name()

> 源码: `include/prism/stealth/anytls/scheme.hpp:35`
> 实现: `src/prism/stealth/anytls/scheme.cpp:23`

### 功能说明

返回方案名称 "anytls"，用于日志记录、调试和方案注册表索引。

### 签名

```cpp
[[nodiscard]] auto name() const noexcept -> std::string_view override;
```

### 参数

无。

### 返回值

`std::string_view` — 固定返回 `"anytls"`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/registry|registry]] — 注册方案时获取名称
- [[stealth/executor|executor]] — 日志记录时获取方案名

### 知识域

无特定知识域。

---

## 函数: tier()

> 源码: `include/prism/stealth/anytls/scheme.hpp:37`

### 功能说明

返回检测层级 2（Tier 2）。AnyTLS 无 ClientHello 独占特征，属于 Tier 2 方案。Tier 2 是分层检测的最后一级，依赖 SNI 路由阶段的匹配结果。

### 签名

```cpp
[[nodiscard]] auto tier() const noexcept -> std::uint8_t override;
```

### 参数

无。

### 返回值

`std::uint8_t` — 固定返回 `2`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/executor|executor]] — 确定检测执行顺序

### 知识域

- [[stealth/scheme|scheme 基类]] — 分层检测架构

---

## 函数: unique()

> 源码: `include/prism/stealth/anytls/scheme.hpp:39`

### 功能说明

返回 `false`，表示 AnyTLS 没有独占特征。Tier 0 检测不支持，需要 Tier 2 SNI 匹配。

### 签名

```cpp
[[nodiscard]] auto unique() const noexcept -> bool override;
```

### 参数

无。

### 返回值

`bool` — 固定返回 `false`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/executor|executor]] — 判断是否跳过后续方案检测

### 知识域

- [[stealth/scheme|scheme 基类]] — Tier 0 独占机制

---

## 函数: active()

> 源码: `include/prism/stealth/anytls/scheme.hpp:43`
> 实现: `src/prism/stealth/anytls/scheme.cpp:18`

### 功能说明

判断 AnyTLS 方案是否在当前配置下启用。委托给 `cfg.stealth.anytls.enabled()` 检查配置完整性，需要证书、私钥等参数。

### 签名

```cpp
[[nodiscard]] auto active(const psm::config &cfg) const noexcept -> bool override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | `const psm::config&` | 服务器全局配置 |

### 返回值

`bool` — `true` 表示方案已正确配置并启用，`false` 表示缺少必要配置项

### 调用（向下）

- `cfg.stealth.anytls.enabled()` — 检查 AnyTLS 配置是否完整

### 被调用（向上）

- [[stealth/registry|registry]] — 启动时过滤未启用的方案
- [[stealth/executor|executor]] — 检测前确认方案可用

### 知识域

- [[stealth/anytls/config|config]] — AnyTLS 配置结构

---

## 函数: snis()

> 源码: `include/prism/stealth/anytls/scheme.hpp:45`
> 实现: `src/prism/stealth/anytls/scheme.cpp:28`

### 功能说明

获取 AnyTLS 的 SNI 白名单。用于 SNI 路由阶段，只有匹配白名单的 SNI 才会进入此方案的检测管道。白名单从 `cfg.stealth.anytls.server_names` 读取。

### 签名

```cpp
[[nodiscard]] auto snis(const psm::config &cfg) const
    -> memory::vector<memory::string> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | `const psm::config&` | 服务器全局配置 |

### 返回值

`memory::vector<memory::string>` — SNI 白名单列表，使用 PMR 分配器

### 调用（向下）

- 读取 `cfg.stealth.anytls.server_names`

### 被调用（向上）

- [[stealth/registry|registry]] — 构建 SNI 路由表

### 知识域

- [[stealth/anytls/config|config]] — server_names 配置
- [[recognition/scheme_route_table|scheme_route_table]] — SNI 路由机制

---

## 函数: verify()

> 源码: `include/prism/stealth/anytls/scheme.hpp:49`
> 实现: `src/prism/stealth/anytls/scheme.cpp:37`

### 功能说明

Tier 1 详细检测。如果配置了 ECH 密钥且 ClientHello 包含 ECH 扩展，尝试 ECH 解密。ECH 扩展存在时返回中等分 300，无 ECH 或未配置时返回低分 0。

**注意**：当前 ECH 解密逻辑为 TODO 桩，仅检测 ECH 扩展是否存在。

### 签名

```cpp
[[nodiscard]] auto verify(const protocol::tls::client_hello_features &features,
                           std::span<const std::byte> raw,
                           const psm::config &cfg) const
    -> verify_result override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| features | `const protocol::tls::client_hello_features&` | ClientHello 特征结构 |
| raw | `std::span<const std::byte>` | 原始 ClientHello 字节 |
| cfg | `const psm::config&` | 服务器全局配置 |

### 返回值

`verify_result` — 详细检测结果：
- `score=300, solo_flag=0` — ECH 扩展存在且密钥已配置
- `score=0, solo_flag=0` — 无 ECH 或未配置密钥

### 调用（向下）

- [[protocol/tls/feature_bitmap|build_feature_bitmap]] — 构建特征位图
- [[protocol/tls/feature_bitmap|has_feature]] — 检查 ECH 扩展标记

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 1 检测阶段

### 知识域

- [[ref/protocol/tls-extensions|TLS 扩展]] — ECH 扩展
- [[stealth/ech/decrypt|decrypt]] — ECH 解密（计划中）

---

## 函数: guess()

> 源码: `include/prism/stealth/anytls/scheme.hpp:55`
> 实现: `src/prism/stealth/anytls/scheme.cpp:67`

### 功能说明

Tier 2 模糊检测。AnyTLS 无 ClientHello 独占特征，依赖 SNI 匹配。SNI 路由阶段已过滤不匹配的连接，这里返回基础分 100 供优先级排序。

### 签名

```cpp
[[nodiscard]] auto guess(const psm::config &cfg) const
    -> verify_result override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | `const psm::config&` | 服务器全局配置 |

### 返回值

`verify_result` — 模糊检测结果：
- `score=100, solo_flag=0, note="AnyTLS: rely on SNI match"`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 2 检测阶段

### 知识域

- [[stealth/scheme|scheme 基类]] — Tier 2 模糊检测机制

---

## 函数: handshake()

> 源码: `include/prism/stealth/anytls/scheme.hpp:59`
> 实现: `src/prism/stealth/anytls/scheme.cpp:78`

### 功能说明

执行 AnyTLS 握手。执行标准 TLS 握手，读取应用数据，解析认证帧，验证用户身份，检测内层协议。

**当前状态**：框架已实现，认证逻辑待完善（TODO 桩）。当前直接返回 `detected=tls` 表示"不是我"，传递给下一个 scheme。

**计划实现的流程**：
1. 执行标准 TLS 握手（使用配置的证书）
2. 读取 TLS 应用数据（客户端首帧）
3. 解析 AnyTLS 认证帧格式: `[password_length:2][password:N][padding:variable]`
4. 验证用户身份
5. 认证成功后检测内层协议

### 签名

```cpp
[[nodiscard]] auto handshake(stealth::handshake_context ctx)
    -> net::awaitable<stealth::handshake_result> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `stealth::handshake_context` | 执行上下文，包含 inbound 传输层、预读数据、配置、路由器、会话 |

### 返回值

`net::awaitable<stealth::handshake_result>` — 协程，返回握手结果：
- `detected=tls` — 当前实现始终返回（TODO 桩）
- `transport` — 原始传输层（未修改）
- `error=not_supported` — session 为空时

### 调用（向下）

- 读取 `ctx.cfg->stealth.anytls` 配置

### 被调用（向上）

- [[stealth/executor|executor]] — 执行握手阶段

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[stealth/anytls/config|config]] — AnyTLS 配置

---

## 函数: weight()

> 源码: `include/prism/stealth/anytls/scheme.hpp:63`

### 功能说明

返回权重分 100。Tier 2 模糊检测时用于优先级排序。与其他 Tier 2 方案（Restls、TrustTunnel）权重相同。

### 签名

```cpp
[[nodiscard]] auto weight() const noexcept -> std::uint16_t override;
```

### 参数

无。

### 返回值

`std::uint16_t` — 固定返回 `100`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/scheme|scheme 基类]] — 默认 `guess()` 实现使用 weight()

### 知识域

- [[stealth/scheme|scheme 基类]] — 权重分机制
