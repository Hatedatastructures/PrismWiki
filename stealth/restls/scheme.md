---
title: "scheme — Restls 伪装方案"
source: "include/prism/stealth/restls/scheme.hpp"
module: "stealth"
submodule: "restls"
type: api
tags: [stealth, restls, scheme, 方案, tier2]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/restls/config|config]]"
  - "[[stealth/scheme|scheme 基类]]"
  - "[[stealth/executor|executor]]"
  - "[[stealth/registry|registry]]"
  - "[[pipeline/primitives|primitives]]"
---

# scheme.hpp

> 源码: `include/prism/stealth/restls/scheme.hpp`
> 实现: `src/prism/stealth/restls/scheme.cpp`
> 模块: [[stealth|stealth]] > [[stealth/restls|restls]]

## 概述

Restls 伪装方案类。实现 [[stealth/scheme|stealth_scheme]] 接口，用于在 TLS 方案管道中处理 Restls 连接。Restls 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。

Restls 通过模拟真实 TLS 流量来隐藏代理特征。服务端与后端 TLS 服务器建立连接，在 TLS 应用数据中嵌入认证信息，使用 `restls-script` 控制流量模式。

**当前状态**：基础框架已实现，认证逻辑待完善。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 继承 stealth_scheme 基类 |
| 依赖 | [[stealth/restls/config|config]] | 使用 Restls 配置 |
| 依赖 | [[agent/config|config]] | 使用 psm::config 配置 |
| 依赖 | [[pipeline/primitives|primitives]] | 使用 find_reliable 穿透包装层 |
| 依赖 | [[protocol/analysis|analysis]] | 协议分析 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[stealth/registry|registry]] | 注册到方案注册表 |
| 被依赖 | [[stealth/executor|executor]] | 被执行器调用 |

## 命名空间

`psm::stealth::restls`

---

## 类: scheme

> 源码: `include/prism/stealth/restls/scheme.hpp:26`

### 概述

Restls 伪装方案实现。Restls 通过模拟真实 TLS 流量来隐藏代理特征。服务端与后端 TLS 服务器建立连接，在 TLS 应用数据中嵌入认证信息。

### 类层次

```
stealth_scheme [[stealth/scheme|scheme]]
  └── restls::scheme
```

### 设计意图

Restls 是 Tier 2 方案，无 ClientHello 独占特征，依赖 SNI 匹配。工作流程：
1. 读取客户端 TLS ClientHello
2. 建立到后端 TLS 服务器的连接
3. 在 TLS 应用数据中验证客户端身份
4. 认证成功后，使用 `restls-script` 控制流量模式

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 返回 "restls" | 下文 |
| tier() | 返回 2（Tier 2） | 下文 |
| unique() | 返回 false（无独占特征） | 下文 |
| active() | 判断方案是否启用 | 下文 |
| snis() | 获取 SNI 白名单 | 下文 |
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

> 源码: `include/prism/stealth/restls/scheme.hpp:30`
> 实现: `src/prism/stealth/restls/scheme.cpp:29`

### 功能说明

返回方案名称 "restls"，用于日志记录、调试和方案注册表索引。

### 签名

```cpp
[[nodiscard]] auto name() const noexcept -> std::string_view override;
```

### 参数

无。

### 返回值

`std::string_view` — 固定返回 `"restls"`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/registry|registry]] — 注册方案时获取名称
- [[stealth/executor|executor]] — 日志记录时获取方案名

### 知识域

无特定知识域。

---

## 函数: tier()

> 源码: `include/prism/stealth/restls/scheme.hpp:32`

### 功能说明

返回检测层级 2（Tier 2）。Restls 无 ClientHello 独占特征，属于 Tier 2 方案。Tier 2 是分层检测的最后一级，依赖 SNI 路由阶段的匹配结果。

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

> 源码: `include/prism/stealth/restls/scheme.hpp:34`

### 功能说明

返回 `false`，表示 Restls 没有独占特征。Tier 0 检测不支持，需要 Tier 2 SNI 匹配。

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

> 源码: `include/prism/stealth/restls/scheme.hpp:38`
> 实现: `src/prism/stealth/restls/scheme.cpp:24`

### 功能说明

判断 Restls 方案是否在当前配置下启用。委托给 `cfg.stealth.restls.enabled()` 检查配置完整性，需要 `host`、`password`、`restls_script` 等参数。

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

- `cfg.stealth.restls.enabled()` — 检查 Restls 配置是否完整

### 被调用（向上）

- [[stealth/registry|registry]] — 启动时过滤未启用的方案
- [[stealth/executor|executor]] — 检测前确认方案可用

### 知识域

- [[stealth/restls/config|config]] — Restls 配置结构

---

## 函数: snis()

> 源码: `include/prism/stealth/restls/scheme.hpp:40`
> 实现: `src/prism/stealth/restls/scheme.cpp:34`

### 功能说明

获取 Restls 的 SNI 白名单。用于 SNI 路由阶段，只有匹配白名单的 SNI 才会进入此方案的检测管道。白名单从 `cfg.stealth.restls.server_names` 读取。

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

- 读取 `cfg.stealth.restls.server_names`

### 被调用（向上）

- [[stealth/registry|registry]] — 构建 SNI 路由表

### 知识域

- [[stealth/restls/config|config]] — server_names 配置
- [[recognition/scheme_route_table|scheme_route_table]] — SNI 路由机制

---

## 函数: guess()

> 源码: `include/prism/stealth/restls/scheme.hpp:44`
> 实现: `src/prism/stealth/restls/scheme.cpp:43`

### 功能说明

Tier 2 模糊检测。Restls 无 ClientHello 独占特征，依赖 SNI 匹配。SNI 路由阶段已过滤不匹配的连接，这里返回基础分 100 供优先级排序。

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
- `score=100, solo_flag=0, note="Restls: rely on SNI match"`

### 调用（向下）

无内部调用。

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 2 检测阶段

### 知识域

- [[stealth/scheme|scheme 基类]] — Tier 2 模糊检测机制

---

## 函数: handshake()

> 源码: `include/prism/stealth/restls/scheme.hpp:48`
> 实现: `src/prism/stealth/restls/scheme.cpp:54`

### 功能说明

执行 Restls 握手。获取底层 reliable transport 的 raw socket，然后执行 Restls 握手流程。

**当前状态**：框架已实现，认证逻辑待完善（TODO 桩）。当前直接返回 `detected=tls` 表示"不是我"，传递给下一个 scheme。

**计划实现的流程**：
1. 读取客户端 TLS ClientHello
2. 建立到后端 TLS 服务器的连接
3. 在 TLS 应用数据中验证客户端身份
4. 认证成功后，使用 `restls-script` 控制流量模式

### 签名

```cpp
auto handshake(stealth::handshake_context ctx)
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

- [[pipeline/primitives|find_reliable]] — 穿透 snapshot/preview 包装层找到底层 TCP socket

### 被调用（向上）

- [[stealth/executor|executor]] — 执行握手阶段

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[pipeline/primitives|primitives]] — 传输层包装层穿透

---

## 函数: weight()

> 源码: `include/prism/stealth/restls/scheme.hpp:51`

### 功能说明

返回权重分 100。Tier 2 模糊检测时用于优先级排序。与其他 Tier 2 方案（AnyTLS、TrustTunnel）权重相同。

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
