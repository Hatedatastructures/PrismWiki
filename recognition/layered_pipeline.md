---
title: "layered_pipeline.hpp — 分层检测管道"
source: "include/prism/recognition/layered_pipeline.hpp"
module: "recognition"
type: api
tags: [recognition, layered_pipeline, 分层检测, 管道]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/recognition
  - recognition/scheme_route_table
  - recognition/confidence
  - stealth/scheme
  - stealth/registry
  - protocol/tls/feature_bitmap
---

# layered_pipeline.hpp

> 源码: `include/prism/recognition/layered_pipeline.hpp` + `src/prism/recognition/layered_pipeline.cpp`
> 模块: [[recognition|Recognition]]

## 概述

按成本和确定性分层执行检测，避免不必要的计算。Tier 0 为零成本字节比较（如 Reality session_id 标记），Tier 1 为有成本验证（如 ShadowTLS HMAC），Tier 2 为模糊匹配（依赖 SNI 路由）。管道按层级顺序执行，独占命中时短路返回，否则收集候选列表按评分排序。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container\|container]] | PMR 容器类型 |
| 依赖 | [[protocol/tls/types\|types]] | ClientHello 特征结构 |
| 依赖 | [[protocol/tls/feature_bitmap\|feature_bitmap]] | 特征位图 |
| 依赖 | [[stealth/scheme\|scheme]] | 伪装方案接口 |
| 依赖 | [[stealth/registry\|registry]] | 方案注册表 |
| 依赖 | [[recognition/scheme_route_table\|scheme_route_table]] | SNI 路由匹配结果 |
| 被依赖 | [[recognition/recognition\|recognition]] | identify 流程使用管道 |

## 命名空间

`psm::recognition`

---

## 结构体: candidate_entry

### 概述

检测候选条目，包含方案名称、评分、层级和确定性标记。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `name` | 方案名称 |
| `std::uint16_t` | `score` | 评分（0-1000，越高越确定） |
| `std::uint8_t` | `tier` | 检测层级 (0-2) |
| `bool` | `is_deterministic` | 是否确定性命中（独占特征） |

---

## 结构体: pipeline_result

### 概述

分层检测管道结果，包含确定性命中信息和候选列表。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `bool` | `deterministic_hit` | 是否确定性命中（独占特征） |
| `memory::string` | `exclusive_scheme` | 独占命中的方案名称 |
| `memory::vector<candidate_entry>` | `candidates` | 候选列表（按评分排序） |
| `memory::string` | `reason` | 检测原因（用于日志） |

---

## 类: layered_detection_pipeline

### 概述

分层检测管道，按成本分层执行检测。构造时从 scheme 注册表中按 `tier()` 值分组到三个层级列表，同时记录 native 兜底方案。

### 设计意图

执行顺序：
1. 先执行 Tier 0，检查独占特征
2. 如果独占命中，直接返回单一候选
3. 否则执行 Tier 1，检查 HMAC 等有成本验证
4. 如果确定性命中，返回单一候选
5. 否则执行 Tier 2，返回多候选列表（含 native 兜底）

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `std::vector<stealth::shared_scheme>` | `tier0_schemes_` | Tier 0 方案列表（有独占特征） |
| `std::vector<stealth::shared_scheme>` | `tier1_schemes_` | Tier 1 方案列表（需要 HMAC/解密验证） |
| `std::vector<stealth::shared_scheme>` | `tier2_schemes_` | Tier 2 方案列表（模糊匹配） |
| `stealth::shared_scheme` | `native_scheme_` | Native 兜底方案 |

---

### 构造函数: layered_detection_pipeline()

#### 功能说明

从 stealth 注册表构建管道。遍历所有已注册的 scheme，根据其 `tier()` 返回值分配到 `tier0_schemes_`、`tier1_schemes_` 或 `tier2_schemes_` 列表，同时记录名为 "native" 的兜底方案。

#### 签名

```cpp
explicit layered_detection_pipeline(const std::vector<stealth::shared_scheme> &schemes);
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `schemes` | `const std::vector<stealth::shared_scheme> &` | 输入 | 已注册的方案列表（来自 registry.all()） |

#### 返回值

无（构造函数）。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `scheme->tier()` | [[stealth/scheme\|scheme]] | 获取方案的检测层级 (0/1/2) |
| `scheme->name()` | [[stealth/scheme\|scheme]] | 获取方案名称，匹配 "native" 兜底 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `identify()` | [[recognition/recognition\|recognition]] | 构建管道实例后调用 detect() |

#### 知识域

- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[stealth/scheme\|伪装方案接口]]

---

### 成员函数: detect()

#### 功能说明

执行分层检测。按 Tier 0 -> Tier 1 -> Tier 2 顺序执行检测，每层短路检查：Tier 0 独占命中直接返回，否则进入 Tier 1；Tier 1 确定性命中直接返回，否则进入 Tier 2 返回模糊匹配候选列表。

#### 签名

```cpp
[[nodiscard]] auto detect(
    std::uint32_t bitmap,
    const protocol::tls::client_hello_features &features,
    std::span<const std::byte> raw,
    const psm::config &cfg,
    const std::vector<stealth::shared_scheme> &matched_schemes) const
    -> pipeline_result;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `bitmap` | `std::uint32_t` | 输入 | 特征位图（由 `build_feature_bitmap()` 生成） |
| `features` | `const client_hello_features &` | 输入 | ClientHello 特征结构 |
| `raw` | `std::span<const std::byte>` | 输入 | 原始 ClientHello 字节 |
| `cfg` | `const psm::config &` | 输入 | 全局配置（用于 `scheme->active()` 检查） |
| `matched_schemes` | `const std::vector<shared_scheme> &` | 输入 | SNI 路由匹配的方案（Tier 2 优先使用） |

#### 返回值

`pipeline_result` — 包含确定性命中信息和按评分排序的候选列表。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `detect_tier0()` | 本类 | 零成本字节比较检测 |
| `detect_tier1()` | 本类 | 有成本 HMAC/解密验证检测 |
| `detect_tier2()` | 本类 | 模糊匹配检测 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `identify()` | [[recognition/recognition\|recognition]] | Phase 4: 执行分层检测获取候选列表 |

#### 知识域

- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[ref/anti-censorship/probing\|探测对抗]]

---

### 私有成员函数: detect_tier0()

#### 功能说明

执行 Tier 0 零成本检测。遍历 Tier 0 方案列表，对每个激活的方案调用 `sniff()` 进行字节级比较。独占命中（`solo=true`）时短路返回确定性结果，非独占命中添加到候选列表。

#### 签名

```cpp
[[nodiscard]] auto detect_tier0(
    std::uint32_t bitmap,
    const protocol::tls::client_hello_features &features,
    const psm::config &cfg) const
    -> pipeline_result;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `bitmap` | `std::uint32_t` | 输入 | 特征位图 |
| `features` | `const client_hello_features &` | 输入 | ClientHello 特征结构 |
| `cfg` | `const psm::config &` | 输入 | 全局配置 |

#### 返回值

`pipeline_result` — 独占命中时 `deterministic_hit=true`，否则收集非独占候选。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `scheme->active()` | [[stealth/scheme\|scheme]] | 检查方案是否在当前配置下激活 |
| `scheme->sniff()` | [[stealth/scheme\|scheme]] | 零成本字节比较，返回 hit/solo/hint |
| `scheme->name()` | [[stealth/scheme\|scheme]] | 获取方案名称 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `detect()` | 本类 | 分层检测第一层 |

#### 知识域

- [[ref/protocol/tls-sessionid\|TLS Session ID]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]

---

### 私有成员函数: detect_tier1()

#### 功能说明

执行 Tier 1 有成本检测。遍历 Tier 1 方案列表，对每个激活的方案调用 `verify()` 执行 HMAC 校验等有成本验证。`solo_flag != 0` 时短路返回确定性结果，否则按评分收集候选。

#### 签名

```cpp
[[nodiscard]] auto detect_tier1(
    const protocol::tls::client_hello_features &features,
    std::span<const std::byte> raw,
    const psm::config &cfg) const
    -> pipeline_result;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `features` | `const client_hello_features &` | 输入 | ClientHello 特征结构 |
| `raw` | `std::span<const std::byte>` | 输入 | 原始 ClientHello 字节 |
| `cfg` | `const psm::config &` | 输入 | 全局配置 |

#### 返回值

`pipeline_result` — 确定性命中时 `deterministic_hit=true`，否则收集候选。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `scheme->active()` | [[stealth/scheme\|scheme]] | 检查方案是否激活 |
| `scheme->verify()` | [[stealth/scheme\|scheme]] | 有成本验证，返回 score/solo_flag/note |
| `scheme->name()` | [[stealth/scheme\|scheme]] | 获取方案名称 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `detect()` | 本类 | 分层检测第二层（Tier 0 未独占命中时） |

#### 知识域

- [[ref/crypto/hmac-sha256\|HMAC-SHA256]]
- [[stealth/shadowtls/handshake\|ShadowTLS 握手]]

---

### 私有成员函数: detect_tier2()

#### 功能说明

执行 Tier 2 模糊匹配检测。优先使用 SNI 路由匹配的方案列表，无匹配时回退到全部 Tier 2 方案。始终将 native 兜底方案追加到候选末尾，最终按评分降序排序。

#### 签名

```cpp
[[nodiscard]] auto detect_tier2(
    const psm::config &cfg,
    const std::vector<stealth::shared_scheme> &matched_schemes) const
    -> pipeline_result;
```

#### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `cfg` | `const psm::config &` | 输入 | 全局配置 |
| `matched_schemes` | `const std::vector<shared_scheme> &` | 输入 | SNI 路由匹配的方案 |

#### 返回值

`pipeline_result` — 包含按评分降序排列的候选列表（含 native 兜底）。

#### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `scheme->active()` | [[stealth/scheme\|scheme]] | 检查方案是否激活 |
| `scheme->guess()` | [[stealth/scheme\|scheme]] | 模糊匹配，返回 score |
| `scheme->name()` | [[stealth/scheme\|scheme]] | 获取方案名称 |

#### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `detect()` | 本类 | 分层检测第三层（Tier 0/1 未确定命中时） |

#### 知识域

- [[recognition/scheme_route_table\|SNI 路由表]]
- [[ref/anti-censorship/probing\|探测对抗]]

---

## 知识域

- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[ref/anti-censorship/probing\|探测对抗]]
- [[stealth/scheme\|伪装方案接口]]
