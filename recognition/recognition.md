---
title: "recognition.hpp — Recognition 模块聚合头文件"
source: "include/prism/recognition/recognition.hpp"
module: "recognition"
type: api
tags: [recognition, 识别, 入口, 协议识别]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/layered_pipeline
  - recognition/scheme_route_table
  - recognition/probe/probe
  - recognition/probe/analyzer
  - recognition/result
  - recognition/confidence
  - stealth/executor
  - stealth/scheme
---

# recognition.hpp

> 源码: `include/prism/recognition/recognition.hpp` + `src/prism/recognition/recognition.cpp`
> 模块: [[recognition|Recognition]]

## 概述

Recognition 模块聚合头文件，引入识别模块所有子模块头文件，提供统一的模块入口和完整的协议识别生命周期。新增 `recognize()` 统一入口，封装外层探测 + TLS 伪装方案识别。`identify()` 仅在 TLS 场景下被调用，完成 ClientHello 读取、SNI 路由、分层检测和方案执行的完整生命周期。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[recognition/confidence\|confidence]] | 置信度枚举 |
| 依赖 | [[recognition/result\|result]] | 分析结果结构 |
| 依赖 | [[recognition/probe/probe\|probe]] | 外层协议探测 |
| 依赖 | [[recognition/probe/analyzer\|analyzer]] | 协议检测函数 |
| 依赖 | [[recognition/scheme_route_table\|scheme_route_table]] | SNI 路由表 |
| 依赖 | [[recognition/layered_pipeline\|layered_pipeline]] | 分层检测管道 |
| 依赖 | [[channel/transport/transmission\|transmission]] | 传输层抽象 |
| 依赖 | [[memory/pool\|pool]] | 帧内存池 |
| 依赖 | [[protocol/analysis\|analysis]] | 协议分析 |
| 依赖 | [[fault/code\|code]] | 错误码 |
| 依赖 | [[stealth/executor\|executor]] | 方案执行器 |
| 依赖 | [[stealth/registry\|registry]] | 方案注册表 |
| 依赖 | [[pipeline/primitives\|primitives]] | preview 原语 |
| 被依赖 | [[agent/session/session\|session]] | 会话调用 recognize |

## 命名空间

`psm::recognition`

---

## 结构体: recognize_context

### 概述

完整识别流程输入上下文，包含识别所需的所有输入，统一入口使用。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `channel::transport::shared_transmission` | `transport` | 传输层（socket 或已包装的传输） |
| `const psm::config *` | `cfg` | 全局配置 |
| `resolve::router *` | `router` | 路由器（fallback 用） |
| `agent::session_context *` | `session` | 会话上下文（供方案使用） |
| `memory::frame_arena *` | `frame_arena` | 帧内存池（用于预读数据分配） |

---

## 结构体: recognize_result

### 概述

完整识别流程输出结果，包含识别完成后的所有输出。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `channel::transport::shared_transmission` | `transport` | 最终传输层（可能被加密或包装） |
| `protocol::protocol_type` | `detected` | 检测到的协议类型 |
| `memory::vector<std::byte>` | `preread` | 预读数据 |
| `fault::code` | `error` | 执行错误码 |
| `memory::string` | `executed_scheme` | 成功执行的方案名称（仅 TLS） |
| `bool` | `success` | 是否成功识别 |

---

## 函数: recognize()

### 功能说明

执行完整协议识别流程。封装外层探测 + TLS 伪装方案识别的两阶段生命周期。Phase 1 调用 `probe::probe()` 预读 24 字节检测外层协议类型；当检测到 TLS 时进入 Phase 2 调用 `identify()` 执行 ClientHello 解析、SNI 路由、分层检测和方案执行。

### 签名

```cpp
auto recognize(recognize_context ctx) -> net::awaitable<recognize_result>;
```

### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `ctx` | `recognize_context` | 输入 | 识别上下文，包含传输层、配置、路由器、会话上下文、帧内存池 |

### 返回值

`net::awaitable<recognize_result>` — 识别结果协程。返回包含最终传输层（可能被 scheme 加密包装）、检测到的协议类型、预读数据、错误码、执行的方案名称和成功标记。

### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `probe::probe()` | [[recognition/probe/probe\|probe]] | Phase 1: 预读 24 字节检测外层协议 |
| `identify()` | 本文件 | Phase 2: 当检测到 TLS 时执行完整识别生命周期 |

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `session::diversion()` | [[agent/session/session\|session]] | 会话分流入口，调用 recognize 判断协议类型 |

### 知识域

- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[ref/protocol/socks5-rfc1928\|SOCKS5]]
- [[ref/protocol/http-connect\|HTTP CONNECT]]

---

## 结构体: identify_context

### 概述

协议识别上下文（输入参数），包含识别所需的所有输入：传输层、配置、预读数据。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `channel::transport::shared_transmission` | `transport` | 传输层（socket 或已包装的传输） |
| `const psm::config *` | `cfg` | 全局配置 |
| `std::span<const std::byte>` | `preread` | 已预读数据（来自 protocol::probe） |
| `resolve::router *` | `router` | 路由器（fallback 用） |
| `agent::session_context *` | `session` | 会话上下文（可选，供方案使用） |
| `memory::frame_arena *` | `frame_arena` | 帧内存池（用于预读数据分配） |

---

## 结构体: identify_result

### 概述

协议识别结果，包含识别完成后的所有输出：传输层、协议类型、预读数据。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `channel::transport::shared_transmission` | `transport` | 最终传输层（可能被加密或包装） |
| `protocol::protocol_type` | `detected` | 检测到的协议类型 |
| `memory::vector<std::byte>` | `preread` | 内层预读数据 |
| `fault::code` | `error` | 执行错误码 |
| `memory::string` | `executed_scheme` | 成功执行的方案名称 |
| `bool` | `success` | 是否成功识别并建立传输层 |

---

## 函数: identify()

### 功能说明

执行完整的 TLS 伪装方案识别生命周期。包含 6 个阶段：读取完整 ClientHello、解析 SNI/特征、SNI 路由表查找匹配方案、构建特征位图并执行分层检测管道、构建 preview transport、按候选顺序执行 scheme。确定性命中时直接执行单一方案，否则按评分顺序尝试多候选。

### 签名

```cpp
auto identify(identify_context ctx) -> net::awaitable<identify_result>;
```

### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `ctx` | `identify_context` | 输入 | 识别上下文，包含传输层、配置、已预读数据、路由器、会话上下文、帧内存池 |

### 返回值

`net::awaitable<identify_result>` — 识别结果协程。返回包含方案执行后的传输层（可能被加密包装）、检测到的协议类型、内层预读数据、错误码、执行的方案名称和成功标记。

### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `protocol::tls::read_tls_record()` | [[protocol/tls/types\|tls]] | Phase 1: 读取完整 TLS ClientHello |
| `protocol::tls::parse_client_hello()` | [[protocol/tls/types\|tls]] | Phase 2: 解析 ClientHello 结构提取特征 |
| `scheme_route_table::build()` | [[recognition/scheme_route_table\|scheme_route_table]] | Phase 3: 从配置构建 SNI 路由表 |
| `scheme_route_table::lookup()` | [[recognition/scheme_route_table\|scheme_route_table]] | Phase 3: 根据 SNI 查找匹配方案 |
| `protocol::tls::build_feature_bitmap()` | [[protocol/tls/feature_bitmap\|feature_bitmap]] | Phase 4: 构建特征位图 |
| `layered_detection_pipeline::detect()` | [[recognition/layered_pipeline\|layered_pipeline]] | Phase 4: 执行分层检测管道 |
| `pipeline::primitives::preview` | [[pipeline/primitives\|primitives]] | Phase 5: 构建 preview transport |
| `stealth::scheme_executor::execute()` | [[stealth/executor\|executor]] | Phase 6: 执行确定性命中方案 |
| `stealth::scheme_executor::execute_by_analysis()` | [[stealth/executor\|executor]] | Phase 6: 按候选顺序执行方案 |

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `recognize()` | 本文件 | 当 probe 检测到 TLS 时调用 identify |

### 知识域

- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[ref/protocol/tls-extensions\|TLS 扩展]]

---

## 知识域

- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[stealth/executor\|方案执行器]]
