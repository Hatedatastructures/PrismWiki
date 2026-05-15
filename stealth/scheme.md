---
title: "scheme.hpp — stealth 模块伪装方案基类"
source: "include/prism/stealth/scheme.hpp"
module: "stealth"
type: api
tags: [stealth, scheme, 基类, 分层检测]
related: [stealth/executor, stealth/registry, stealth/reality/scheme, stealth/shadowtls/scheme, protocol/tls/types, protocol/tls/feature_bitmap]
created: 2026-05-15
updated: 2026-05-15
---

# scheme.hpp

> 源码: `include/prism/stealth/scheme.hpp`
> 实现: header-only
> 模块: [[stealth|stealth]]

## 概述

定义 `stealth_scheme` 抽象基类，每个方案代表一种传输层伪装方式（如 Reality、ShadowTLS、Standard TLS）。调用方通过 `handshake()` 接口完成握手和协议检测，获得最终传输层和检测到的协议类型。

**分层检测架构**:
- **Tier 0**: `sniff()` — 零成本字节比较（Reality session_id 标记）
- **Tier 1**: `verify()` — 有成本验证（ShadowTLS HMAC）
- **Tier 2**: `guess()` — 模糊匹配（Restls/TrustTunnel）
- **handshake()** — 执行握手，失败则 fallback 到真实 TLS

## 依赖关系

| 依赖方向 | 模块 | 说明 |
| ---- | ---- | ---- |
| 依赖 | [[channel/transport/transmission|transmission]] | 使用 shared_transmission 传输层抽象 |
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[memory/container|container]] | 使用 memory::string/vector PMR 容器 |
| 依赖 | [[protocol/analysis|analysis]] | 使用 protocol_type 协议类型枚举 |
| 依赖 | [[protocol/tls/types|types]] | 使用 client_hello_features 结构 |
| 依赖 | [[protocol/tls/feature_bitmap|feature_bitmap]] | 使用特征位图 |
| 被依赖 | [[stealth/reality/scheme|reality/scheme]] | Reality 方案实现 |
| 被依赖 | [[stealth/shadowtls/scheme|shadowtls/scheme]] | ShadowTLS 方案实现 |
| 被依赖 | [[stealth/ech/decrypt|ech/decrypt]] | ECH 方案实现 |
| 被依赖 | [[stealth/anytls/scheme|anytls/scheme]] | AnyTLS 方案实现 |
| 被依赖 | [[stealth/restls/scheme|restls/scheme]] | Restls 方案实现 |
| 被依赖 | [[stealth/trusttunnel/scheme|trusttunnel/scheme]] | TrustTunnel 方案实现 |
| 被依赖 | [[stealth/executor|executor]] | 方案执行器 |

## 命名空间

`psm::stealth`

---

## 结构体: sniff_result

> 源码: `include/prism/stealth/scheme.hpp:59`

### 概述

Tier 0 快速检测结果。零成本字节比较，返回是否命中和是否独占。用于最快的特征匹配，例如 Reality 检查 `session_id[0:3] == [0x01, 0x08, 0x02]`。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| hit | bool | 是否命中此方案 | false |
| solo | bool | 是否独占命中（命中则不再检测其他方案） | false |
| hint | uint16_t | 评分提示（供 Tier 2 参考，范围 0-1000） | 0 |
| note | memory::string | 检测原因（用于日志和调试） | "" |

---

## 结构体: verify_result

> 源码: `include/prism/stealth/scheme.hpp:84`

### 概述

伪装方案检测结果（评分制）。使用评分制，支持优先级排序。`solo_flag` 非零表示独占命中，不再检测其他方案。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| score | uint16_t | 评分（0-1000，越高越确定） | 0 |
| solo_flag | uint16_t | 独占标记（非零表示独占，跳过其他方案） | 0 |
| note | memory::string | 检测原因（用于日志和调试） | "" |

---

## 结构体: handshake_result

> 源码: `include/prism/stealth/scheme.hpp:105`

### 概述

伪装方案执行结果。包含执行后的传输层、检测到的内层协议和预读数据。这是 `handshake()` 的返回值。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| transport | shared_transmission | 最终传输层 |
| detected | protocol::protocol_type | 检测到的内层协议 |
| preread | memory::vector\<std::byte\> | 内层预读数据 |
| error | fault::code | 错误码（默认 success） |
| scheme | memory::string | 成功执行的方案名 |

---

## 结构体: handshake_context

> 源码: `include/prism/stealth/scheme.hpp:120`

### 概述

伪装方案执行上下文。封装 `handshake()` 所需的所有参数，避免参数过长。调用方应在调用前用 `preview` 包装 `inbound`（如有预读数据）。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| inbound | shared_transmission | 当前传输层（应包含预读数据） |
| cfg | const psm::config* | 服务器配置 |
| router | resolve::router* | 路由器（fallback 用） |
| session | agent::session_context* | 会话上下文 |
| preread | memory::vector\<std::byte\> | 来自 identify 的 preread 数据（完整 ClientHello） |

---

## 类: stealth_scheme

> 源码: `include/prism/stealth/scheme.hpp:142`

### 概述

传输层伪装方案抽象基类。支持分层检测架构：Tier 0 快速检测、Tier 1 详细检测、Tier 2 模糊检测。每个子类代表一种伪装方案（如 Reality、ShadowTLS），通过虚函数实现具体的检测和握手逻辑。

### 类层次

```
stealth_scheme (抽象基类)
  ├── reality_scheme [[stealth/reality/scheme|reality/scheme]]
  ├── shadowtls_scheme [[stealth/shadowtls/scheme|shadowtls/scheme]]
  ├── ech_scheme [[stealth/ech/decrypt|ech/decrypt]]
  ├── anytls_scheme [[stealth/anytls/scheme|anytls/scheme]]
  ├── restls_scheme [[stealth/restls/scheme|restls/scheme]]
  └── trusttunnel_scheme [[stealth/trusttunnel/scheme|trusttunnel/scheme]]
```

### 设计意图

采用分层检测架构，平衡检测速度和准确性：
- **Tier 0** 是最快的，只做字节比较，适合独占特征检测（如 Reality 的 session_id 标记）
- **Tier 1** 涉及 HMAC 验证或解密，延迟执行，适合需要密码学验证的方案（如 ShadowTLS）
- **Tier 2** 是兜底方案，无独占特征，依赖 SNI 匹配（如 Restls、TrustTunnel）

这种设计使得 99% 的流量在 Tier 0 就能确定方案，只有少数流量需要 Tier 1/2 验证。

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 方案名称（用于日志） | 下文 |
| tier() | 检测层级（0-2） | 下文 |
| unique() | 是否有独占特征 | 下文 |
| active() | 判断方案是否启用 | 下文 |
| snis() | 获取 SNI 白名单 | 下文 |
| sniff() | Tier 0 快速检测 | 下文 |
| verify() | Tier 1 详细检测 | 下文 |
| guess() | Tier 2 模糊检测 | 下文 |
| handshake() | 执行握手 | 下文 |

### 生命周期

1. **构造**: 由 `registry` 在启动时创建，存储为 `shared_ptr`
2. **使用**: 由 `executor` 调用，执行检测和握手
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 所有检测函数（`sniff()`, `verify()`, `guess()`）是线程安全的，只读取配置
- `handshake()` 涉及 I/O 操作，需要在协程中调用

### 异常安全

- 所有函数不抛异常，错误通过返回值报告

---

## 函数: name()

> 源码: `include/prism/stealth/scheme.hpp:150`

### 功能

返回方案的名称，用于日志记录和调试。每个子类实现此函数返回唯一的方案名（如 "reality", "shadowtls"）。

### 签名

```cpp
[[nodiscard]] virtual auto name() const noexcept -> std::string_view = 0;
```

### 参数

无

### 返回值

`std::string_view` — 方案名称，不拥有内存

### 调用（向下）

无

### 被调用（向上）

- [[stealth/executor|executor]] — 日志记录和错误报告

### 知识域

- [[ref/programming/c++23-coroutines|C++23 协程]]

---

## 函数: tier()

> 源码: `include/prism/stealth/scheme.hpp:153`

### 功能

返回方案的检测层级。Tier 0 有独占特征（如 Reality），Tier 1 需要密码学验证（如 ShadowTLS），Tier 2 依赖 SNI 匹配（如 Restls）。

### 签名

```cpp
[[nodiscard]] virtual auto tier() const noexcept -> std::uint8_t;
```

### 参数

无

### 返回值

`std::uint8_t` — 检测层级（0, 1, 2），默认返回 2

### 调用（向下）

无

### 被调用（向上）

- [[stealth/executor|executor]] — 决定检测顺序

### 知识域

- 分层检测架构

---

## 函数: unique()

> 源码: `include/prism/stealth/scheme.hpp:159`

### 功能

判断方案是否有独占特征。如果返回 `true`，则 Tier 0 命中后不再检测其他方案。Reality 的 session_id 标记是独占特征。

### 签名

```cpp
[[nodiscard]] virtual auto unique() const noexcept -> bool;
```

### 参数

无

### 返回值

`bool` — 是否有独占特征，默认返回 `false`

### 调用（向下）

无

### 被调用（向上）

- [[stealth/executor|executor]] — 决定是否跳过其他方案

### 知识域

- 分层检测架构

---

## 函数: active()

> 源码: `include/prism/stealth/scheme.hpp:167`

### 功能

判断此方案是否在当前配置下启用。例如，Reality 方案需要配置 `dest`、`server_names`、`private_key` 等参数才能启用。

### 签名

```cpp
[[nodiscard]] virtual auto active(const psm::config &cfg) const noexcept -> bool = 0;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置 |

### 返回值

`bool` — 方案是否启用

### 调用（向下）

无（各子类读取自身配置字段）

### 被调用（向上）

- [[stealth/registry|registry]] — 启动时过滤启用的方案
- [[stealth/executor|executor]] — 检测前判断方案是否启用

### 知识域

- [[agent/config|config 配置结构]]

---

## 函数: snis()

> 源码: `include/prism/stealth/scheme.hpp:170`

### 功能

获取方案的 SNI 白名单。用于 SNI 路由，只有匹配白名单的 SNI 才会尝试此方案。默认返回空列表，表示不使用 SNI 路由。

### 签名

```cpp
[[nodiscard]] virtual auto snis(const psm::config &cfg) const
    -> memory::vector<memory::string>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置 |

### 返回值

`memory::vector<memory::string>` — SNI 白名单，默认返回空

### 调用（向下）

无（各子类读取自身配置字段）

### 被调用（向上）

- [[stealth/registry|registry]] — 构建 SNI 路由表

### 知识域

- [[recognition/scheme_route_table|SNI 路由表]]

---

## 函数: sniff()

> 源码: `include/prism/stealth/scheme.hpp:186`

### 功能

Tier 0 快速检测。只做字节比较，不涉及 HMAC 或解密。例如 Reality 检查 `session_id[0:3] == [0x01, 0x08, 0x02]`。这是最快的检测方式，适合独占特征。

### 签名

```cpp
[[nodiscard]] virtual auto sniff(std::uint32_t bitmap,
                                  const protocol::tls::client_hello_features &features) const
    -> sniff_result;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| bitmap | std::uint32_t | 特征位图 |
| features | const protocol::tls::client_hello_features& | ClientHello 特征结构 |

### 返回值

`sniff_result` — 快速检测结果，默认返回 `{hit=false, solo=false, hint=0, note="no sniff"}`

### 调用（向下）

无（只做字节比较）

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 0 检测阶段

### 知识域

- [[protocol/tls/types|client_hello_features]] — ClientHello 特征结构
- [[protocol/tls/feature_bitmap|feature_bitmap]] — 特征位图

---

## 函数: verify()

> 源码: `include/prism/stealth/scheme.hpp:205`

### 功能

Tier 1 详细检测。涉及 HMAC 验证或解密，延迟执行。例如 ShadowTLS HMAC 验证、AnyTLS ECH 解密。比 Tier 0 慢，但更准确。

### 签名

```cpp
[[nodiscard]] virtual auto verify(const protocol::tls::client_hello_features &features,
                                   std::span<const std::byte> raw,
                                   const psm::config &cfg) const
    -> verify_result;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| features | const protocol::tls::client_hello_features& | ClientHello 特征结构 |
| raw | std::span<const std::byte> | 原始 ClientHello 字节 |
| cfg | const psm::config& | 服务器配置 |

### 返回值

`verify_result` — 详细检测结果，默认返回 `{score=0, solo_flag=0, note="no verify"}`

### 调用（向下）

- [[stealth/shadowtls/auth|verify_client_hello]] — ShadowTLS HMAC 验证
- [[stealth/reality/auth|auth]] — Reality 验证

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 1 检测阶段

### 知识域

- [[ref/protocol/tls-sessionid|tls-sessionid]] — SessionID 字段
- [[ref/protocol/tls-serverrandom|tls-serverrandom]] — ServerRandom 字段
- [[ref/crypto/hmac-sha256|HMAC-SHA256]]

---

## 函数: guess()

> 源码: `include/prism/stealth/scheme.hpp:223`

### 功能

Tier 2 模糊检测。无 ClientHello 独占特征，依赖 SNI 匹配。例如 Restls、TrustTunnel、Native。返回权重分，供优先级排序。

### 签名

```cpp
[[nodiscard]] virtual auto guess(const psm::config &cfg) const
    -> verify_result;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置 |

### 返回值

`verify_result` — 模糊检测结果，默认返回权重分

### 调用（向下）

- weight() — 获取权重分

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 2 检测阶段

### 知识域

- 分层检测架构

---

## 函数: handshake()

> 源码: `include/prism/stealth/scheme.hpp:237`

### 功能

执行握手。根据方案类型，完成 TLS 握手或代理协议握手，返回最终传输层和检测到的内层协议。如果握手失败，返回错误码。

### 签名

```cpp
[[nodiscard]] virtual auto handshake(handshake_context ctx)
    -> net::awaitable<handshake_result> = 0;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | handshake_context | 执行上下文（传输层、预读数据、配置、路由器、会话） |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回握手结果

### 错误处理

返回 `handshake_result.error` 为非 `success` 的错误码：
- `fault::code::handshake_failed` — 握手失败
- `fault::code::protocol_error` — 协议错误

### 调用（向下）

- [[stealth/reality/handshake|reality/handshake]] — Reality 握手
- [[stealth/shadowtls/handshake|shadowtls/handshake]] — ShadowTLS 握手

### 被调用（向上）

- [[stealth/executor|executor]] — 执行握手阶段

### 知识域

- [[ref/protocol/tls-1.3|tls-1.3]] — TLS 1.3 握手流程

---

## 函数: weight()

> 源码: `include/prism/stealth/scheme.hpp:242`

### 功能

返回方案的权重分。Tier 2 模糊检测时使用，用于优先级排序。默认返回 100，子类可覆盖。

### 签名

```cpp
[[nodiscard]] virtual auto weight() const noexcept -> std::uint16_t;
```

### 参数

无

### 返回值

`std::uint16_t` — 权重分，默认返回 100

### 调用（向下）

无

### 被调用（向上）

- guess() — Tier 2 检测时使用

### 知识域

- 分层检测架构

---

## 类型别名: shared_scheme

> 源码: `include/prism/stealth/scheme.hpp:248`

```cpp
using shared_scheme = std::shared_ptr<stealth_scheme>;
```

方案的共享指针类型，由 `registry` 管理生命周期。
