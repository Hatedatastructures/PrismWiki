---
title: Stealth 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [stealth, tls, camouflage, reality, scheme, registry, overview, design]
related: [[recognition], [protocol], [channel/overview], [resolve/overview], [crypto/overview], [memory/overview]]
---

# Stealth 模块

## 概述

Stealth 是 Prism 的传输层伪装方案层，位于 [[channel]] 传输通道之上、
[[multiplex]] 多路复用之下。核心使命是让代理流量伪装成合法的 TLS 连接，
抵御流量审查和主动探测。

Stealth 采用三层分层检测架构，按检测成本从低到高排列：

- **Tier 0**：零成本字节比较 — 例如 Reality 检查 session_id 标记字节
- **Tier 1**：有成本验证 — 例如 ShadowTLS HMAC 验证、AnyTLS ECH 解密
- **Tier 2**：模糊匹配 — 例如 Restls/TrustTunnel/Native，依赖 SNI 路由和权重评分

检测层级之间遵循"独占优先"原则：Tier 0 方案命中后立即终止，不再执行后续检测。

### 核心抽象

#### 方案基类（stealth_scheme）

所有伪装方案继承 `stealth_scheme` 抽象基类，实现统一的检测与执行接口。
身份查询方法 `name()`/`tier()`/`unique()` 标识方案元信息；
配置方法 `active(cfg)` 和 `snis(cfg)` 判断方案启用状态和 SNI 白名单；
三级检测方法 `sniff()`/`verify()`/`guess()` 分别对应 Tier 0/1/2；
`handshake(context)` 执行握手并返回最终传输层与检测到的内层协议。

#### 注册表（scheme_registry）

单例模式，管理所有方案的注册和查询。启动阶段调用 `register_all_schemes()`
注册全部内置方案，运行时只读无需同步。识别模块通过 `all()` 获取方案列表，
按注册顺序（即默认优先级）进行检测。

#### 执行器（scheme_executor）

按候选方案列表依次尝试执行。每个方案执行后通过 `detected` 类型判断匹配：
返回 TLS 表示不匹配，继续下一个方案；返回具体协议表示匹配，终止执行。

执行器内部维护 snapshot 回滚机制：通过 `ensure_snapshot()` 包装传输层，
方案间数据隔离；失败时 `try_rewind()` 回滚到同一起点，下一个方案可从
相同位置重试。全部方案失败时自动执行 Native 兜底。

#### ECH 支持

`ech::decrypt_ech_payload()` 使用 HPKE 解密 ECH outer payload，
获取 inner ClientHello 中的真实 SNI。ECH 是叠加层，可在任意 TLS 伪装
协议之上使用。AnyTLS 方案在 Tier 1 的 `verify()` 中调用此接口。

### 内置方案概览

| 方案 | Tier | 独占 | 检测机制 | 权重 |
|------|------|------|----------|------|
| Reality | 0 | 是 | session_id 字节标记 | 450 |
| ShadowTLS | 1 | 否 | HMAC 验证 | 100 |
| AnyTLS | 2 | 否 | ECH 解密(Tier1) + SNI 匹配 | 100 |
| Restls | 2 | 否 | SNI 匹配 | 100 |
| TrustTunnel | 2 | 否 | SNI 匹配，支持 TCP/UDP | 100 |
| Native | 2 | 否 | 标准 TLS 握手（兜底） | 50 |

### 设计原则

- **分层检测**：按成本递增依次检测，Tier 0 仅做字节比较避免不必要的计算
- **独占优先**：Tier 0 方案命中后立即终止，不再检测其他方案
- **评分制**：`verify_result` 使用 0-1000 分评分，支持跨方案优先级排序
- **snapshot 回滚**：方案间数据隔离，失败时可回滚到同一起点重试
- **注册表模式**：方案与执行器解耦，新增方案只需在 `register_all_schemes()` 中注册一行

### 与其他模块的关系

- **上游**：[[recognition]] 通过 `scheme_registry` 和 `scheme_executor` 驱动检测和执行
- **上游**：[[protocol]] 的 TLS 分析模块提供 `client_hello_features` 供 Tier 0/1 检测使用
- **下游**：使用 [[channel]] 的 `shared_transmission` 和 snapshot 装饰器
- **下游**：使用 [[resolve]] 的 `router` 建立 fallback 连接（如 Restls 连接后端 TLS 服务器）

## 详细设计

### 数据结构

`sniff_result`（Tier 0）：`hit` 命中、`solo` 独占、`hint` 评分提示（0-1000）、`note` 原因。

`verify_result`（Tier 1/2）：`score` 评分（0-1000）、`solo_flag` 独占标记、`note` 原因。

`handshake_result`：`transport` 传输层、`detected` 内层协议、`preread` 预读数据、
`error` 错误码、`scheme` 方案名。

`handshake_context`：`inbound` 传输层、`cfg` 配置、`router` 路由器、
`session` 会话上下文、`preread` ClientHello 字节。

### 方案基类接口

**文件**: `stealth/scheme.hpp`

```
class stealth_scheme:
  virtual name() -> string_view = 0
  virtual tier() -> uint8           // 默认 2
  virtual unique() -> bool          // 默认 false
  virtual active(const config&) -> bool = 0
  virtual snis(const config&) -> vector<string>
  virtual sniff(bitmap, features) -> sniff_result          // Tier 0
  virtual verify(features, raw, config) -> verify_result   // Tier 1
  virtual guess(config) -> verify_result                   // Tier 2
  virtual handshake(context) -> awaitable<handshake_result> = 0
protected:
  virtual weight() -> uint16  // 默认 100，Tier 2 使用
```

### 注册表（scheme_registry）

**文件**: `stealth/registry.hpp` + `.cpp`

```
class scheme_registry:
  static instance() -> scheme_registry&
  add(shared_scheme)
  all() -> const vector<shared_scheme>&    // 按注册顺序
  find(name) -> shared_scheme

auto register_all_schemes() -> void
```

### 执行器（scheme_executor）

**文件**: `stealth/executor.hpp` + `.cpp`

```
class scheme_executor:
  scheme_executor(const registry&)
  execute_by_analysis(analysis_result, context) -> awaitable<handshake_result>
  execute(candidates, context) -> awaitable<handshake_result>
private:
  execute_pipeline(order, context)      // 核心循环
  execute_single(scheme, context)       // 单方案执行
  ensure_snapshot(context)              // 包装 snapshot
  try_rewind(context)                   // 回滚 snapshot
  pass_through(context, result)         // 传递 preread
```

流程：ensure_snapshot → 依次执行 → `detected!=TLS` 匹配返回 →
`detected==TLS` 则 rewind + pass_through → 全部失败执行 native 兜底。

### 各方案实现

#### Reality（Tier 0，独占，权重 450）

**文件**: `stealth/reality/scheme.hpp`

`sniff()` 检查 `session_id[0:3] == [0x01, 0x08, 0x02]`，命中则 `solo=true`。不实现 verify/guess。

实现：auth/handshake/keygen/request/response/seal/config/constants（8 个组件）。

#### ShadowTLS v3（Tier 1，权重 100）

**文件**: `stealth/shadowtls/scheme.hpp`

`sniff()` Tier 0 快速排除，`verify()` HMAC 验证。

实现：auth/handshake/config/constants（4 个组件）。

#### AnyTLS（Tier 2，权重 100）

**文件**: `stealth/anytls/scheme.hpp`

标准证书 + 应用层认证。`verify()` ECH 解密，`guess()` SNI 匹配。

#### Restls（Tier 2，权重 100）

**文件**: `stealth/restls/scheme.hpp`

`guess()` SNI 匹配。连接后端 TLS 服务器，应用数据中验证身份，restls-script 控制流量模式。

#### TrustTunnel（Tier 2，权重 100）

**文件**: `stealth/trusttunnel/scheme.hpp`

`guess()` SNI 匹配。支持 TCP（HTTP/2）和 UDP（HTTP/3）。

#### Native（Tier 2 兜底，权重 50）

**文件**: `stealth/native.hpp`

权重最低，标准 TLS 握手 + 内层协议自动检测。

### ECH 解密

**文件**: `stealth/ech/decrypt.hpp` + `.cpp`

```
struct decrypt_result {
  bool valid; client_hello_features inner_features;
  vector<byte> inner_client_hello; fault::code error;
}
auto decrypt_ech_payload(outer_payload, ech_key) -> decrypt_result
```

HPKE：解析 ECH outer → shared secret → AEAD 解密 → 解析 inner ClientHello。

`ech::config`：`ech_key`(base64) + `public_name`。`enabled()` 检查非空。glaze JSON 序列化。

## 文件清单

| 文件 | 说明 |
|------|------|
| `stealth/scheme.hpp` | 方案基类 + 结果/上下文结构体 |
| `stealth/registry.hpp` / `.cpp` | 方案注册表 + `register_all_schemes()` |
| `stealth/executor.hpp` / `.cpp` | 方案执行器（snapshot 回滚） |
| `stealth/native.hpp` / `.cpp` | Native TLS 兜底方案 |
| `stealth/reality/*.hpp` + `.cpp` | Reality 方案（8 个组件） |
| `stealth/shadowtls/*.hpp` + `.cpp` | ShadowTLS 方案（4 个组件） |
| `stealth/restls/scheme.hpp` + `config.hpp` | Restls 方案 |
| `stealth/anytls/scheme.hpp` + `config.hpp` | AnyTLS 方案 |
| `stealth/trusttunnel/scheme.hpp` + `config.hpp` | TrustTunnel 方案 |
| `stealth/ech/config.hpp` + `decrypt.hpp` / `.cpp` | ECH 配置与 HPKE 解密 |
| `stealth.hpp` | 聚合头文件 |

## 相关链接

- [[stealth/reality]] — Reality 协议详解
- [[docs/stealth/pki-certificates]] — PKI 证书详解
- [[recognition]] — Recognition 模块
- [[protocol]] — Protocol 模块
- [[channel]] — Channel 模块
- [[crypto]] — Crypto 模块
- [[memory]] — Memory 模块
