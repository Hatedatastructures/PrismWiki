---
title: "native — 原生 TLS 伪装方案（兜底）"
source: "include/prism/stealth/native.hpp"
module: "stealth"
type: api
tags: [stealth, native, 方案, tier2, 兜底]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/scheme|scheme]]"
  - "[[stealth/executor|executor]]"
  - "[[stealth/registry|registry]]"
  - "[[channel/transport/encrypted|encrypted]]"
  - "[[protocol/analysis|analysis]]"
---

# native.hpp

> 源码: `include/prism/stealth/native.hpp`
> 实现: `src/prism/stealth/native.cpp`
> 模块: [[stealth|stealth]]

## 概述

原生 TLS 伪装方案（兜底）。封装标准 TLS 握手和内层协议检测，继承 [[stealth/scheme|stealth_scheme]] 基类。Native 是 Tier 2 方案，作为兜底处理无法匹配其他方案的 TLS 连接。使用标准 BoringSSL TLS 握手，不进行任何伪装。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 继承 stealth_scheme 基类 |
| 依赖 | [[pipeline/primitives|primitives]] | 使用 ssl_handshake 原语 |
| 依赖 | [[channel/transport/encrypted|encrypted]] | 创建加密传输层 |
| 依赖 | [[protocol/analysis|analysis]] | 使用 detect_tls 检测内层协议 |
| 被依赖 | [[stealth/registry|registry]] | 注册到方案注册表 |
| 被依赖 | [[stealth/executor|executor]] | 被执行器调用作为兜底 |

## 命名空间

`psm::stealth::schemes`

---

## 类: native

> 源码: `include/prism/stealth/native.hpp:13`

### 概述

原生 TLS 伪装方案实现。Native 是 Tier 2 方案，作为兜底处理无法匹配其他方案的 TLS 连接。权重分最低（50），确保只有在其他方案都不匹配时才使用。

### 类层次

```
stealth_scheme [[stealth/scheme|scheme]]
  └── schemes::native
```

### 设计意图

Native 是兜底方案，用于处理无法匹配其他方案的 TLS 连接。权重分最低（50），确保只有在其他方案都不匹配时才使用。握手流程：标准 TLS 握手 → 读取内层数据 → 检测内层协议类型。

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| name() | 返回 "native" | 下文 |
| tier() | 返回 2（Tier 2） | 下文 |
| unique() | 返回 false（无独占特征） | 下文 |
| active() | 始终返回 true | 下文 |
| guess() | Tier 2 模糊检测，返回权重分 50 | 下文 |
| handshake() | 执行标准 TLS 握手并检测内层协议 | 下文 |
| weight() | 返回权重分 50 | 下文 |

### 生命周期

1. **构造**: 由 [[stealth/registry|registry]] 在启动时创建
2. **使用**: 被 [[stealth/executor|executor]] 调用，作为兜底方案
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 所有检测函数是线程安全的
- `handshake()` 涉及 I/O 操作，需要在协程中调用

### 异常安全

- 所有函数不抛异常，错误通过返回值报告

---

## 函数: name()

> 源码: `include/prism/stealth/native.hpp:17`
> 实现: `src/prism/stealth/native.cpp:22`

### 功能

返回方案名称 "native"，用于日志记录和调试。

### 签名

```cpp
[[nodiscard]] auto name() const noexcept -> std::string_view override;
```

### 参数

无

### 返回值

`std::string_view` — "native"

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/executor|find_scheme]] — 按名称查找方案
- [[stealth/registry|scheme_registry::find]] — 注册表查找方案

### 知识域

- [[stealth/scheme|stealth_scheme::name()]] — 基类接口

---

## 函数: tier()

> 源码: `include/prism/stealth/native.hpp:19`

### 功能

返回检测层级 2（Tier 2）。Native 无 ClientHello 独占特征，属于 Tier 2 模糊匹配方案。

### 签名

```cpp
[[nodiscard]] auto tier() const noexcept -> std::uint8_t override;
```

### 参数

无

### 返回值

`std::uint8_t` — 2

### 调用（向下）

- 无

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别模块按 tier 分层检测

### 知识域

- [[stealth/scheme|stealth_scheme]] — 分层检测架构（Tier 0/1/2）

---

## 函数: unique()

> 源码: `include/prism/stealth/native.hpp:21`

### 功能

返回 `false`，表示 Native 没有独占特征。Tier 0 命中后不会跳过其他方案。

### 签名

```cpp
[[nodiscard]] auto unique() const noexcept -> bool override;
```

### 参数

无

### 返回值

`bool` — `false`

### 调用（向下）

- 无

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别模块判断是否独占

### 知识域

- [[stealth/scheme|stealth_scheme::unique()]] — 独占特征接口

---

## 函数: active()

> 源码: `include/prism/stealth/native.hpp:25`
> 实现: `src/prism/stealth/native.cpp:17`

### 功能

判断 Native 方案是否启用。始终返回 `true`，因为 Native 是兜底方案，不能被禁用。

### 签名

```cpp
[[nodiscard]] auto active(const psm::config &cfg) const noexcept -> bool override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置（未使用） |

### 返回值

`bool` — 始终返回 `true`

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/executor|execute_pipeline]] — 管道执行前检查方案是否启用

### 知识域

- [[stealth/scheme|stealth_scheme::active()]] — 方案启用检查接口

---

## 函数: guess()

> 源码: `include/prism/stealth/native.hpp:29`
> 实现: `src/prism/stealth/native.cpp:27`

### 功能

Tier 2 模糊检测。返回权重分（50），供优先级排序。权重分最低，确保只有在其他方案都不匹配时才使用。返回 `solo_flag=0` 表示不独占。

### 签名

```cpp
[[nodiscard]] auto guess(const psm::config &cfg) const
    -> verify_result override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | const psm::config& | 服务器配置（未使用） |

### 返回值

`verify_result` — 模糊检测结果，`{score=50, solo_flag=0, note="native TLS fallback"}`

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/executor|executor]] — Tier 2 检测阶段

### 知识域

- [[stealth/scheme|verify_result]] — 检测结果结构体
- [[stealth/scheme|stealth_scheme::guess()]] — Tier 2 模糊检测接口

---

## 函数: handshake()

> 源码: `include/prism/stealth/native.hpp:33`
> 实现: `src/prism/stealth/native.cpp:37`

### 功能

执行标准 TLS 握手并检测内层协议。使用 BoringSSL 进行标准 TLS 握手，握手成功后读取内层数据并调用 [[protocol/analysis|detect_tls]] 检测内层协议类型（如 Trojan、VLESS 等）。检测到的协议类型和加密传输层写入结果。

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

`net::awaitable<stealth::handshake_result>` — 协程，返回握手结果（传输层、检测到的协议、预读数据）

### 调用（向下）

- [[pipeline/primitives|ssl_handshake]] — 执行标准 TLS 握手
- [[channel/transport/encrypted|encrypted]] — 创建加密传输层包装
- [[protocol/analysis|detect_tls]] — 检测内层协议类型

### 被调用（向上）

- [[stealth/executor|execute_single]] — 执行器调用方案握手
- [[stealth/executor|execute_by_analysis]] — 全部方案失败时 native 兜底

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 握手流程
- [[protocol/analysis|protocol analysis]] — 内层协议检测

### 流程

1. 检查 session 上下文是否有效
2. 调用 `ssl_handshake()` 执行标准 TLS 握手
3. 创建 `encrypted` 加密传输层包装
4. 注册 cancel/close 回调到 session
5. 循环读取内层数据（最多 128 字节）
6. 调用 `detect_tls()` 检测内层协议
7. 检测成功：返回 transport + detected + preread
8. 检测失败：返回 `fault::code::protocol_error`

---

## 函数: weight()

> 源码: `include/prism/stealth/native.hpp:37`

### 功能

返回权重分 50。Tier 2 模糊检测时使用，权重分最低，确保只有在其他方案都不匹配时才使用。

### 签名

```cpp
[[nodiscard]] auto weight() const noexcept -> std::uint16_t override;
```

### 参数

无

### 返回值

`std::uint16_t` — 50

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/scheme|stealth_scheme::guess()]] — 基类默认 guess() 使用 weight()

### 知识域

- [[stealth/scheme|stealth_scheme::weight()]] — 权重分接口
