---
title: "scheme — Reality 伪装方案"
source: "include/prism/stealth/reality/scheme.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, scheme, 方案, tier0]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/scheme|scheme]]"
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/executor|executor]]"
  - "[[stealth/registry|registry]]"
  - "[[protocol/tls/feature_bitmap|feature_bitmap]]"
---

# scheme.hpp

> 源码: `include/prism/stealth/reality/scheme.hpp`
> 实现: `src/prism/stealth/reality/scheme.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality 伪装方案类。封装 Reality TLS 握手和协议检测逻辑，继承 [[stealth/scheme|stealth_scheme]] 基类。Reality 是 Tier 0 方案，有独占特征（session_id 标记 `[0x01, 0x08, 0x02]`），命中后跳过其他方案。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 继承 stealth_scheme 基类 |
| 依赖 | [[stealth/reality/handshake|handshake]] | 调用 reality::handshake() |
| 依赖 | [[protocol/tls/feature_bitmap|feature_bitmap]] | 使用特征位图进行 Tier 0 检测 |
| 依赖 | [[config|config]] | 使用 psm::config 配置 |
| 被依赖 | [[stealth/registry|registry]] | 注册到方案注册表 |
| 被依赖 | [[stealth/executor|executor]] | 被执行器调用 |

## 命名空间

`psm::stealth::reality`

---

## 类: scheme

> 源码: `include/prism/stealth/reality/scheme.hpp:14`

### 概述

Reality 伪装方案实现。继承 [[stealth/scheme|stealth_scheme]] 基类，实现 Tier 0 检测（独占特征）和握手逻辑。Reality 是最快的检测方案，通过 session_id 标记实现零成本字节比较。

### 类层次

```
stealth_scheme [[stealth/scheme|scheme]]
  └── reality::scheme
```

### 设计意图

Reality 是 Tier 0 方案，有独占特征（session_id 标记）。这意味着：
- Tier 0 检测只需字节比较，不涉及 HMAC 或解密
- 命中后跳过其他方案（solo=true）
- 是最快的检测方式，适合 99% 的流量

Reality 的核心思想是：客户端在 ClientHello 的 SessionID 中嵌入特殊标记，服务端通过字节比较识别。

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 返回 "reality" | 下文 |
| tier() | 返回 0（Tier 0） | 下文 |
| unique() | 返回 true（有独占特征） | 下文 |
| active() | 判断方案是否启用 | 下文 |
| snis() | 获取 SNI 白名单 | 下文 |
| sniff() | Tier 0 快速检测（session_id 标记） | 下文 |
| handshake() | 执行 Reality 握手 | 下文 |
| weight() | 返回权重分 450 | 下文 |

### 生命周期

1. **构造**: 由 [[stealth/registry|registry]] 在启动时创建
2. **使用**: 被 [[stealth/executor|executor]] 调用，执行检测和握手
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 所有检测函数是线程安全的，只读取配置
- `handshake()` 涉及 I/O 操作，需要在协程中调用

### 异常安全

- 所有函数不抛异常，错误通过返回值报告

---

## 函数: name()

> 源码: `include/prism/stealth/reality/scheme.hpp:18`
> 实现: `src/prism/stealth/reality/scheme.cpp:108`

### 功能

返回方案名称 "reality"，用于日志记录和调试。

### 签名

```cpp
[[nodiscard]] auto name() const noexcept -> std::string_view override;
```

### 参数

无

### 返回值

`std::string_view` — "reality"

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/executor|find_scheme]] — 按名称查找方案
- [[stealth/registry|scheme_registry::find]] — 注册表查找方案

### 知识域

- [[stealth/scheme|stealth_scheme::name()]] — 基类接口

---

## 函数: tier()

> 源码: `include/prism/stealth/reality/scheme.hpp:20`

### 功能

返回检测层级 0（Tier 0）。Reality 有独占特征，属于 Tier 0 方案，检测速度最快。

### 签名

```cpp
[[nodiscard]] auto tier() const noexcept -> std::uint8_t override;
```

### 参数

无

### 返回值

`std::uint8_t` — 0

### 调用（向下）

- 无

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别模块按 tier 分层检测

### 知识域

- [[stealth/scheme|stealth_scheme]] — 分层检测架构（Tier 0/1/2）

---

## 函数: unique()

> 源码: `include/prism/stealth/reality/scheme.hpp:22`

### 功能

返回 `true`，表示 Reality 有独占特征。Tier 0 命中后跳过其他方案（solo=true）。

### 签名

```cpp
[[nodiscard]] auto unique() const noexcept -> bool override;
```

### 参数

无

### 返回值

`bool` — `true`

### 调用（向下）

- 无

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别模块判断是否独占

### 知识域

- [[stealth/scheme|stealth_scheme::unique()]] — 独占特征接口

---

## 函数: active()

> 源码: `include/prism/stealth/reality/scheme.hpp:26`
> 实现: `src/prism/stealth/reality/scheme.cpp:15`

### 功能

判断 Reality 方案是否在当前配置下启用。需要配置中 `stealth.reality` 节点的 `enabled()` 返回 true（需要 `dest`、`server_names`、`private_key`、`short_ids` 等参数）。

### 签名

```cpp
[[nodiscard]] auto active(const psm::config &cfg) const noexcept -> bool override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置 |

### 返回值

`bool` — 方案是否启用

### 调用（向下）

- [[config|config::stealth::reality::enabled()]] — 检查 Reality 配置是否完整

### 被调用（向上）

- [[stealth/executor|execute_pipeline]] — 管道执行前检查方案是否启用

### 知识域

- [[config|psm::config]] — 服务器配置结构

---

## 函数: snis()

> 源码: `include/prism/stealth/reality/scheme.hpp:28`
> 实现: `src/prism/stealth/reality/scheme.cpp:20`

### 功能

获取 Reality 的 SNI 白名单。从配置中提取 `stealth.reality.server_names` 列表，用于 SNI 路由，只有匹配白名单的 SNI 才会尝试此方案。

### 签名

```cpp
[[nodiscard]] auto snis(const psm::config &cfg) const
    -> memory::vector<memory::string> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置 |

### 返回值

`memory::vector<memory::string>` — SNI 白名单列表

### 调用（向下）

- 无（遍历配置中的 server_names）

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别模块构建 SNI 路由表

### 知识域

- [[config|config::stealth::reality]] — Reality 配置

---

## 函数: sniff()

> 源码: `include/prism/stealth/reality/scheme.hpp:32`
> 实现: `src/prism/stealth/reality/scheme.cpp:29`

### 功能

Tier 0 快速检测。通过特征位图检查 ClientHello 中的 Reality 标记。分四级置信度：独占标记 `[01:08:02]`（hint=950, solo=true）→ X25519 + session_id=32（hint=450）→ X25519 + 非标准 session_id（hint=400）→ 仅 X25519（hint=200）→ 仅 SNI + session_id=32（hint=100）。

### 签名

```cpp
[[nodiscard]] auto sniff(std::uint32_t bitmap,
                          const protocol::tls::client_hello_features &features) const
    -> sniff_result override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| bitmap | std::uint32_t | 特征位图（由 recognition 层计算） |
| features | const protocol::tls::client_hello_features& | ClientHello 特征结构 |

### 返回值

`sniff_result` — 快速检测结果，命中时返回 `{hit=true, solo=true/false, hint=N}`

### 调用（向下）

- [[protocol/tls/feature_bitmap|has_feature()]] — 检查单个特征位
- [[protocol/tls/feature_bitmap|has_all_features()]] — 检查多个特征位

### 被调用（向上）

- [[recognition/recognition|recognize]] — Tier 0 检测阶段

### 知识域

- [[protocol/tls/feature_bitmap|feature_bitmap]] — ClientHello 特征位图
- [[ref/protocol/tls-sessionid|TLS SessionID]] — TLS SessionID 字段

### 流程

1. 检查 `reality_marker_01_08_02` 特征位 → 独占命中（hint=950, solo=true）
2. 检查 `has_x25519 | has_full_session_id` → 高置信度（hint=450）
3. 检查 `has_x25519 + session_id_non_standard` → 中置信度（hint=400）
4. 检查 `has_x25519` → 中低置信度（hint=200）
5. 检查 `has_sni | has_full_session_id` → 低置信度（hint=100）
6. 检查 `has_sni` → 最低置信度（hint=100）
7. 无命中返回 `{hit=false}`

---

## 函数: handshake()

> 源码: `include/prism/stealth/reality/scheme.hpp:40`
> 实现: `src/prism/stealth/reality/scheme.cpp:113`

### 功能

执行 Reality 握手。调用 [[stealth/reality/handshake|reality::handshake()]] 完成完整的握手流程，根据结果类型映射到 [[stealth/scheme|handshake_result]]：认证成功返回 seal 加密传输层和 VLESS 协议；非 Reality 返回 TLS 类型和原始记录；回退完成返回空结果；失败返回错误码。

### 签名

```cpp
[[nodiscard]] auto handshake(stealth::handshake_context ctx)
    -> net::awaitable<stealth::handshake_result> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | stealth::handshake_context | 执行上下文（传输层、预读数据、配置、路由器、会话） |

### 返回值

`net::awaitable<stealth::handshake_result>` — 协程，返回握手结果

### 调用（向下）

- [[stealth/reality/handshake|reality::handshake()]] — Reality 握手状态机

### 被调用（向上）

- [[stealth/executor|execute_single]] — 执行器调用方案握手

### 知识域

- [[stealth/reality/handshake|handshake_result_type]] — Reality 握手结果类型
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程

### 流程

1. 检查 session 上下文是否有效
2. 调用 `reality::handshake()` 执行 Reality 握手
3. **authenticated**: transport = seal, detected = vless, preread = inner_preread
4. **not_reality**: transport = inbound, preread = raw_tls_record, detected = tls
5. **fallback**: 空结果（透明代理已完成）
6. **failed**: 保留原始 transport，返回错误码

---

## 函数: weight()

> 源码: `include/prism/stealth/reality/scheme.hpp:44`

### 功能

返回权重分 450。Reality 不参与 Tier 2 guess 检测（通过 Tier 0 sniff 独占），此权重分仅用于 fallback 场景。

### 签名

```cpp
[[nodiscard]] auto weight() const noexcept -> std::uint16_t override;
```

### 参数

无

### 返回值

`std::uint16_t` — 450

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/scheme|stealth_scheme::guess()]] — 基类默认 guess() 使用 weight()

### 知识域

- [[stealth/scheme|stealth_scheme::weight()]] — 权重分接口
