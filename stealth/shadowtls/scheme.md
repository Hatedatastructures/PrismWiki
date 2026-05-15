---
title: "scheme — ShadowTLS v3 伪装方案"
source: "include/prism/stealth/shadowtls/scheme.hpp"
module: "stealth"
submodule: "shadowtls"
type: api
tags: [stealth, shadowtls, scheme, 方案, tier1]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/shadowtls/handshake|handshake]]"
  - "[[stealth/shadowtls/auth|auth]]"
  - "[[stealth/shadowtls/config|config]]"
  - "[[stealth/scheme|scheme 基类]]"
  - "[[stealth/executor|executor]]"
  - "[[stealth/registry|registry]]"
---

# scheme.hpp

> 源码: `include/prism/stealth/shadowtls/scheme.hpp`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp`
> 模块: [[stealth|stealth]] > [[stealth/shadowtls|shadowtls]]

## 概述

ShadowTLS v3 伪装方案。封装 ShadowTLS 握手和协议检测逻辑，继承 [[stealth/scheme|stealth_scheme]] 基类。ShadowTLS 是 Tier 1 方案，需要 HMAC 验证确认身份。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 继承 stealth_scheme 基类 |
| 依赖 | [[stealth/shadowtls/auth|auth]] | 使用 HMAC 验证函数 |
| 依赖 | [[stealth/shadowtls/handshake|handshake]] | 使用握手函数 |
| 依赖 | [[stealth/shadowtls/config|config]] | 使用 ShadowTLS 配置 |
| 依赖 | [[agent/config|config]] | 使用 psm::config 配置 |
| 依赖 | [[pipeline/primitives|primitives]] | 使用 find_reliable 穿透包装层 |
| 依赖 | [[protocol/analysis|analysis]] | 使用 detect_tls 检测内层协议 |
| 被依赖 | [[stealth/registry|registry]] | 注册到方案注册表 |
| 被依赖 | [[stealth/executor|executor]] | 被执行器调用 |

## 命名空间

`psm::stealth::shadowtls`

---

## 类: scheme

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:14`

### 概述

ShadowTLS v3 伪装方案实现。继承 `stealth_scheme` 基类，实现 Tier 1 检测（HMAC 验证）和握手逻辑。ShadowTLS 使用标准 TLS 外层，认证发生在 ClientHello 阶段。

### 类层次

```
stealth_scheme [[stealth/scheme|scheme]]
  └── shadowtls::scheme
```

### 设计意图

ShadowTLS 是 Tier 1 方案，需要 HMAC 验证确认身份。与 Reality 不同，ShadowTLS 使用标准 TLS 外层，不需要伪造证书。认证发生在 ClientHello 阶段，通过 SessionID 中的 HMAC 标签验证客户端身份。

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 返回 "shadowtls" | 下文 |
| tier() | 返回 1（Tier 1） | 下文 |
| unique() | 返回 false（无独占特征） | 下文 |
| active() | 判断方案是否启用 | 下文 |
| snis() | 获取 SNI 白名单 | 下文 |
| sniff() | Tier 0 快速检测 | 下文 |
| verify() | Tier 1 详细检测（HMAC 验证） | 下文 |
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

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:18`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:30`

### 功能说明

返回方案名称 "shadowtls"，用于日志记录、调试和方案注册表索引。

### 签名

```cpp
[[nodiscard]] auto name() const noexcept -> std::string_view override;
```

### 参数

无。

### 返回值

`std::string_view` — 固定返回 `"shadowtls"`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/registry|registry]] — 注册方案时获取名称
- [[stealth/executor|executor]] — 日志记录时获取方案名

### 知识域

无特定知识域。

---

## 函数: tier()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:20`

### 功能说明

返回检测层级 1（Tier 1）。ShadowTLS 需要 HMAC 验证确认身份，属于 Tier 1 方案。Tier 1 在 Tier 0 快速检测之后执行，涉及密码学运算。

### 签名

```cpp
[[nodiscard]] auto tier() const noexcept -> std::uint8_t override;
```

### 参数

无。

### 返回值

`std::uint8_t` — 固定返回 `1`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/executor|executor]] — 确定检测执行顺序

### 知识域

- [[stealth/scheme|scheme 基类]] — 分层检测架构

---

## 函数: unique()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:22`

### 功能说明

返回 `false`，表示 ShadowTLS 没有 Tier 0 独占特征。ShadowTLS 的 SessionID 长度虽然是 32 字节（非标准），但不能仅凭此独占判断，需要 Tier 1 HMAC 验证确认。

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

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:26`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:20`

### 功能说明

判断 ShadowTLS 方案是否在当前配置下启用。v3 模式需要至少一个用户、握手目标和 SNI 白名单；v2 兼容模式需要密码、握手目标和 SNI 白名单。

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

- 读取 `cfg.stealth.shadowtls` 配置字段

### 被调用（向上）

- [[stealth/registry|registry]] — 启动时过滤未启用的方案
- [[stealth/executor|executor]] — 检测前确认方案可用

### 知识域

- [[stealth/shadowtls/config|config]] — ShadowTLS 配置结构

---

## 函数: snis()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:28`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:35`

### 功能说明

获取 ShadowTLS 的 SNI 白名单。用于 SNI 路由阶段，只有匹配白名单的 SNI 才会进入此方案的检测管道。白名单从 `cfg.stealth.shadowtls.server_names` 读取。

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

- 读取 `cfg.stealth.shadowtls.server_names`

### 被调用（向上）

- [[stealth/registry|registry]] — 构建 SNI 路由表

### 知识域

- [[stealth/shadowtls/config|config]] — server_names 配置
- [[recognition/scheme_route_table|scheme_route_table]] — SNI 路由机制

---

## 函数: sniff()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:32`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:44`

### 功能说明

Tier 0 快速检测。检查 ClientHello 的 SessionID 长度是否为非标准值（ShadowTLS 要求 32 字节）。命中时返回 `hit=true, solo=false`（不能独占，需要 Tier 1 HMAC 验证），hint 分 150。

### 签名

```cpp
[[nodiscard]] auto sniff(std::uint32_t bitmap,
                          const protocol::tls::client_hello_features &features) const
    -> sniff_result override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| bitmap | `std::uint32_t` | ClientHello 特征位图 |
| features | `const protocol::tls::client_hello_features&` | ClientHello 解析后的特征结构 |

### 返回值

`sniff_result` — 快速检测结果：
- `hit=true, solo=false, hint=150` — SessionID 非标准长度
- `hit=false` — 未命中

### 调用（向下）

- 使用 `protocol::tls::has_feature(bitmap, session_id_non_standard)` 检查位图

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 0 检测阶段

### 知识域

- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 字段结构
- [[protocol/tls/feature_bitmap|feature_bitmap]] — 特征位图定义

---

## 函数: verify()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:37`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:63`

### 功能说明

Tier 1 详细检测。执行 HMAC 验证确认客户端身份。要求 `session_id_len == 32` 且 `raw_client_hello >= 76` 字节。v3 模式遍历所有用户密码进行 HMAC 验证；v2 兼容模式使用单一密码。HMAC 验证通过返回高分 900 + 独占标记，失败返回低分 50。

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
| raw | `std::span<const std::byte>` | 原始 ClientHello 字节（含 TLS 记录头） |
| cfg | `const psm::config&` | 服务器全局配置 |

### 返回值

`verify_result` — 详细检测结果：
- `score=900, solo_flag=0xFFFF` — HMAC 验证通过（独占）
- `score=50, solo_flag=0` — HMAC 不匹配

### 调用（向下）

- [[stealth/shadowtls/auth|verify_client_hello]] — 验证 ClientHello HMAC

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 1 检测阶段

### 知识域

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[ref/protocol/tls-sessionid|TLS SessionID]] — TLS SessionID 字段
- [[stealth/shadowtls/config|config]] — 多用户密码配置

---

## 函数: handshake()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:43`
> 实现: `src/prism/stealth/shadowtls/scheme.cpp:107`

### 功能说明

执行 ShadowTLS 握手。从 `handshake_context` 中提取底层 reliable transport 的 raw socket，将 Recognition 层预读的 ClientHello 传递给 [[stealth/shadowtls/handshake|shadowtls::handshake()]]。认证成功后解析内层协议（通过 `protocol::analysis::detect_tls`），用 `preview` 包装传输层以回放内层首帧数据。认证失败时将 detected 设为 `tls` 表示"不是我"，传递给下一个 scheme。

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
- `transport` — 最终传输层（可能包装为 preview）
- `detected` — 检测到的内层协议类型
- `preread` — 内层预读数据
- `error` — 错误码

### 调用（向下）

- [[pipeline/primitives|find_reliable]] — 穿透 snapshot/preview 包装层找到底层 TCP socket
- [[stealth/shadowtls/handshake|handshake]] — 执行 ShadowTLS 握手流程
- [[protocol/analysis|detect_tls]] — 检测内层协议类型
- [[pipeline/primitives|preview]] — 包装传输层以回放预读数据

### 被调用（向上）

- [[stealth/executor|executor]] — 执行握手阶段

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — HMAC-SHA1 消息认证码
- [[stealth/shadowtls/handshake|handshake]] — ShadowTLS 握手流程

---

## 函数: weight()

> 源码: `include/prism/stealth/shadowtls/scheme.hpp:47`

### 功能说明

返回权重分 100。Tier 2 模糊检测时用于优先级排序。ShadowTLS 不参与 Tier 2 检测（已有 Tier 1 HMAC 验证），此函数仅满足接口要求。

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
