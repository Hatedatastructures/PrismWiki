---
tags: [recognition, recognition]
layer: core
module: recognition
source: I:/code/Prism/include/prism/recognition/recognition.hpp
title: recognition.hpp
---

# recognition.hpp

Recognition 模块聚合头文件，提供统一的协议识别入口。

## 核心类型

### recognize_context

完整识别流程输入上下文。

| 字段 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 传输层 |
| `cfg` | `const psm::config *` | 全局配置 |
| `router` | `connect::router *` | 路由器（fallback 用） |
| `session` | `context::session *` | 会话上下文 |
| `frame_arena` | `memory::frame_arena *` | 帧内存池 |

### recognize_result

完整识别流程输出结果。

| 字段 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 最终传输层 |
| `detected` | `protocol::protocol_type` | 检测到的协议类型 |
| `preread` | `memory::vector<std::byte>` | 预读数据 |
| `error` | `fault::code` | 错误码 |
| `executed_scheme` | `memory::string` | 成功执行的方案名称 |
| `success` | `bool` | 是否成功识别 |

### identify_context / identify_result

TLS 识别的输入/输出结构，字段类似 `recognize_context/result`，但增加了 `preread`（`std::span<const std::byte>`）用于传递 Probe 阶段的预读数据。

## 核心函数

### recognize()

```cpp
auto recognize(recognize_context ctx) -> net::awaitable<recognize_result>;
```

执行完整协议识别流程（Probe → Identify）。

### identify()

```cpp
auto identify(identify_context ctx) -> net::awaitable<identify_result>;
```

执行 TLS 伪装方案识别（Read → Parse → Detect → Execute）。

## 设计决策

### 为什么 recognize() 和 identify() 是独立函数而非类方法？

`recognize()` 是无状态的顶层协调函数，所有状态通过 `recognize_context` 传入。这避免了类实例生命周期管理（特别是在协程中的 `shared_from_this()` 问题），也使得调用方（session）无需持有识别器对象。

**后果**: 两次 `recognize()` 调用之间不共享状态，每次调用独立处理。

## 约束

### 指针成员非空保证

**类型**: 状态前置

**规则**: `recognize_context` 中的 `cfg`、`session`、`frame_arena` 必须非空。`router` 在需要 fallback 时必须非空。

**违反后果**: 空指针解引用，崩溃。

**源码依据**: `recognition.hpp:65-74`

### transport 所有权

**类型**: 生命周期

**规则**: `recognize_context::transport` 通过值传递（`shared_transmission` 即 `shared_ptr`），引用计数保证传输层在识别期间存活。

**违反后果**: 无直接风险（`shared_ptr` 保护）。

## 引用关系

### 被引用

- [[core/instance/worker/launch|launch]]：调用 `recognize()` 入口

### 依赖

- [[core/recognition/confidence|confidence]]：置信度枚举
- [[core/recognition/result|result]]：分析结果
- [[core/recognition/layered_pipeline|layered_pipeline]]：分层检测管道
- [[core/recognition/scheme-route-table|scheme-route-table]]：SNI 路由表
- [[core/recognition/probe/probe|probe]]：外层协议探测
- [[core/recognition/probe/analyzer|analyzer]]：协议类型检测
- [[core/stealth/scheme|stealth::scheme]]：伪装方案
- [[core/transport/transmission|transport]]：传输层
- [[core/protocol/types|protocol::protocol_type]]：协议类型枚举