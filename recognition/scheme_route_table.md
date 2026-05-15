---
title: "scheme_route_table.hpp — SNI 路由表"
source: "include/prism/recognition/scheme_route_table.hpp"
module: "recognition"
type: api
tags: [recognition, scheme_route_table, SNI, 路由]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/recognition
  - recognition/layered_pipeline
  - stealth/reality/config
  - stealth/shadowtls/config
  - stealth/restls/config
  - stealth/anytls/config
  - stealth/trusttunnel/config
---

# scheme_route_table.hpp

> 源码: `include/prism/recognition/scheme_route_table.hpp` + `src/prism/recognition/scheme_route_table.cpp`
> 模块: [[recognition|Recognition]]

## 概述

根据 ClientHello SNI 快速路由到对应伪装方案。从配置构建，启动时初始化，支持多方案共享同一 SNI（返回多个候选）。路由表覆盖 Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel 五种方案的 `server_names` 配置。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container\|container]] | PMR 容器类型 |
| 依赖 | [[agent/config\|全局配置]] | 读取 stealth 配置 |
| 被依赖 | [[recognition/recognition\|recognition]] | identify 流程使用路由表 |
| 被依赖 | [[recognition/layered_pipeline\|layered_pipeline]] | Tier 2 使用匹配结果 |

## 命名空间

`psm::recognition`

---

## 类: scheme_route_table

### 概述

SNI 路由表，从各伪装方案的 `server_names` 配置构建路由表，实现 SNI 到方案名称的快速映射。

### 设计意图

**路由规则**：
- Reality `server_names` -> "reality"
- ShadowTLS `server_names` -> "shadowtls"
- Restls `server_names` -> "restls"
- AnyTLS `server_names` -> "anytls"
- TrustTunnel `server_names` -> "trusttunnel"
- 未匹配任何 SNI -> 返回空列表（后续 fallback 到 native）

**多方案共享 SNI**：
- 同一 SNI 可配置在多个方案中，返回多个候选
- 例如："example.com" 同时配置在 reality 和 shadowtls
- 执行时按优先级顺序尝试

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::map<memory::string, memory::vector<memory::string>>` | `route_map_` | SNI 到方案名称列表映射 |

---

### 静态成员函数: build()

#### 功能说明

从配置构建路由表。遍历 Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel 五种方案的 `server_names` 配置，检查各方案是否 `enabled()`，对每个启用方案的 SNI 调用 `add_route()` 注册映射。

#### 签名

```cpp
static auto build(const psm::config &cfg) -> scheme_route_table;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `cfg` | `const psm::config &` | 输入 | 全局配置，包含各 stealth 方案的 server_names |

#### 返回值

`scheme_route_table` — 构建完成的路由表实例。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `cfg.stealth.reality.enabled()` | [[stealth/reality/config\|reality config]] | 检查 Reality 是否启用 |
| `cfg.stealth.shadowtls.enabled()` | [[stealth/shadowtls/config\|shadowtls config]] | 检查 ShadowTLS 是否启用 |
| `cfg.stealth.restls.enabled()` | [[stealth/restls/config\|restls config]] | 检查 Restls 是否启用 |
| `cfg.stealth.anytls.enabled()` | [[stealth/anytls/config\|anytls config]] | 检查 AnyTLS 是否启用 |
| `cfg.stealth.trusttunnel.enabled()` | [[stealth/trusttunnel/config\|trusttunnel config]] | 检查 TrustTunnel 是否启用 |
| `add_route()` | 本类 | 注册单个 SNI -> 方案映射 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `identify()` | [[recognition/recognition\|recognition]] | Phase 3: 从配置构建路由表 |

#### 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]
- [[stealth/reality/config\|Reality 配置]]

---

### 成员函数: lookup()

#### 功能说明

根据 SNI 查找匹配的方案名称列表。空 SNI 直接返回空列表，否则在 `route_map_` 中精确查找。返回的列表通常只有一个元素，多方案共享 SNI 时可能有多个。

#### 签名

```cpp
[[nodiscard]] auto lookup(std::string_view sni) const
    -> memory::vector<memory::string>;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `sni` | `std::string_view` | 输入 | ClientHello 中的 SNI 值 |

#### 返回值

`memory::vector<memory::string>` — 匹配的方案名称列表。空 SNI 或未匹配时返回空列表。

#### 调用（向下）

无（直接查表）。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `identify()` | [[recognition/recognition\|recognition]] | Phase 3: 根据 SNI 查找匹配方案 |

#### 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]

---

### 成员函数: matches_any()

#### 功能说明

检查 SNI 是否匹配任意已注册的方案。比 `lookup()` 更轻量，仅返回布尔结果，适用于快速判断是否需要进入 Tier 2 检测。

#### 签名

```cpp
[[nodiscard]] auto matches_any(std::string_view sni) const -> bool;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `sni` | `std::string_view` | 输入 | ClientHello 中的 SNI 值 |

#### 返回值

`bool` — 匹配至少一个方案时返回 `true`，空 SNI 返回 `false`。

#### 调用（向下）

无（直接查表）。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| 外部调用方 | — | 快速判断 SNI 是否命中任何方案 |

#### 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]

---

### 成员函数: all_registered_snis()

#### 功能说明

获取所有已注册的 SNI 列表。遍历 `route_map_` 的所有 key 返回，用于调试和日志输出。

#### 签名

```cpp
[[nodiscard]] auto all_registered_snis() const
    -> memory::vector<memory::string>;
```

#### 参数表格

无参数。

#### 返回值

`memory::vector<memory::string>` — 所有已注册的 SNI 字符串列表。

#### 调用（向下）

无（直接遍历 map）。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| 调试/日志 | — | 输出路由表内容 |

#### 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]

---

### 成员函数: empty()

#### 功能说明

检查路由表是否为空。无任何 SNI 注册时返回 `true`，表示所有方案均未配置 `server_names`。

#### 签名

```cpp
[[nodiscard]] auto empty() const noexcept -> bool;
```

#### 参数表格

无参数。

#### 返回值

`bool` — 无任何 SNI 注册时返回 `true`。

#### 调用（向下）

无。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| 外部调用方 | — | 检查是否有 SNI 路由配置 |

#### 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]

---

### 私有成员函数: add_route()

#### 功能说明

添加单个 SNI 到方案的映射。空 SNI 直接忽略。如果 SNI 已存在且方案名已注册则去重跳过，否则追加到现有列表或创建新条目。

#### 签名

```cpp
auto add_route(std::string_view sni, std::string_view scheme_name) -> void;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `sni` | `std::string_view` | 输入 | SNI 值 |
| `scheme_name` | `std::string_view` | 输入 | 方案名称（如 "reality"、"shadowtls"） |

#### 返回值

无（void）。

#### 调用（向下）

无（直接操作 map）。

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `build()` | 本类 | 遍历配置时注册每个 SNI 映射 |

#### 知识域

- [[memory/container\|PMR 容器]]

---

## 知识域

- [[ref/protocol/tls-extensions\|TLS 扩展]]
- [[stealth/reality/config\|Reality 配置]]
- [[stealth/shadowtls/config\|ShadowTLS 配置]]
