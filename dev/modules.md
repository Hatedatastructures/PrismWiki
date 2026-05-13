---
title: Prism 模块结构
created: 2026-05-13
updated: 2026-05-13
type: concept
tags: [architecture, modules, overview]
related: [[agent/overview], [agent/architecture], [protocol/overview], [pipeline/overview], [recognition/overview]]
---

# Prism 模块结构

Prism 采用模块化设计，每个模块职责清晰，通过聚合头文件组织。命名空间统一使用 `psm::` 前缀。

## 模块总览

| 模块 | 头文件 | 源文件 | 职责 |
|------|--------|--------|------|
| agent（核心） | `include/prism/agent/` | `src/prism/agent/` | 核心会话链路：监听、负载均衡、会话管理、协议分发 |
| recognition | `include/prism/recognition/` | `src/prism/recognition/` | 协议智能识别：外层探测 + TLS 伪装方案识别 |
| pipeline | `include/prism/pipeline/` | `src/prism/pipeline/` | 协议处理管道：原语 + 各协议会话编排 |
| protocol | `include/prism/protocol/` | `src/prism/protocol/` | 协议编解码：消息结构、中继器、TLS 特征 |
| resolve | `include/prism/resolve/` | `src/prism/resolve/` | DNS 解析：七阶段查询管道 |
| channel | `include/prism/channel/` | `src/prism/channel/` | 连接管理：连接池、Happy Eyeballs、传输层 |
| multiplex | `include/prism/multiplex/` | `src/prism/multiplex/` | 多路复用：smux v1 + yamux |
| stealth | `include/prism/stealth/` | `src/prism/stealth/` | TLS 伪装：Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel |
| memory | `include/prism/memory/` | header-only | 内存管理：PMR 全局池 + 线程独占池 |
| trace | `include/prism/trace/` | `src/prism/trace/` | 日志系统 |
| fault | `include/prism/fault/` | header-only | 错误码定义 |
| crypto | `include/prism/crypto/` | `src/prism/crypto/` | 加密算法 |
| exception | `include/prism/exception/` | header-only | 异常层次结构 |
| config | `include/prism/config.hpp` | header-only | 全局配置管理 |
| transformer | `include/prism/transformer/` | header-only | 数据转换 |
| outbound | `include/prism/outbound/` | header-only | 出站连接抽象 |
| loader | `include/prism/loader/` | header-only | 配置加载 |

## 模块层次关系

```
main.cpp
    ↓
agent ← loader（配置加载）
    ↓
recognition → protocol（编解码）→ pipeline（编排）
    ↓
channel（连接池）→ multiplex（多路复用）→ stealth（TLS 伪装）
    ↓
resolve（DNS）→ crypto（加密）→ memory（内存）
```

## 聚合头文件

每个主要模块都有根级聚合头文件，引入该模块所有子头文件：

`agent.hpp`, `memory.hpp`, `protocol.hpp`, `stealth.hpp`, `exception.hpp`, `fault.hpp`, `trace.hpp`, `transformer.hpp`, `config.hpp`, `recognition.hpp`, `channel.hpp`, `multiplex.hpp`, `pipeline.hpp`, `resolve.hpp`, `crypto.hpp`, `outbound.hpp`, `loader.hpp`

新增子头文件时需同步更新对应的聚合头文件。

## 请求处理数据流

```
listener → balancer → worker → launch → session
    → recognition::recognize()     [协议识别]
    → dispatch::dispatch()         [协议分发]
    → pipeline::protocol()         [协议处理]
    → primitives::tunnel()         [双向转发]
    → channel/resolve/multiplex    [连接/DNS/复用]
```

## 命名规范

- **命名空间**：`psm::` 前缀（如 `psm::resolve`、`psm::channel`）
- **文件**：snake_case（如 `connection_pool.hpp`）
- **生产代码**：类/函数/类型/结构体/枚举全部 snake_case
- **测试代码**：函数统一 PascalCase（如 `TestBasicGetRequest`）
- **头文件保护**：统一使用 `#pragma once`
- **返回类型**：使用尾随返回类型（`auto func() -> return_type`）
- **不可丢弃**：对有意义的返回值使用 `[[nodiscard]]`
- **Boost.Asio 别名**：统一 `namespace net = boost::asio;`

## 相关页面

- [[agent]] — Agent 模块概述
- [[agent/architecture]] — 架构设计
- [[protocol]] — Protocol 模块
- [[pipeline]] — Pipeline 模块
- [[recognition]] — Recognition 模块
- [[dev/cpp23-coroutines]] — 协程技术
- [[dev/pmr-memory-pool]] — PMR 内存池
