---
title: Core 层总览
created: 2026-05-17
updated: 2026-05-17
layer: core
source: "I:/code/Prism/include/prism/"
tags: [architecture, overview]
---

# Core 层总览

Core 层是知识库最详细的层级，包含 Prism 所有模块的实现细节。

## 职责

- 函数实现详解（复杂函数逐行解释）
- 调用链关系
- 状态变化说明
- 错误处理逻辑

## 详细程度

- **复杂函数**: 逐行解释，说明每行代码作用、设计决策
- **简单函数**: 只写用途，不逐行解释
- **枚举/常量**: 列出值和含义，不逐行解释

复杂判断标准（综合）：
1. 代码行数 > 50
2. 逻辑分支 > 3
3. 涉及专业知识（加密/协议/协程）
4. 有复杂状态转换

## 模块列表

| 模块 | 职责 | 源码位置 |
|------|------|----------|
| [[core/instance/overview|Agent]] | 前端监听、会话管理、负载均衡 | `include/prism/instance/` |
| [[core/connect/overview|Channel]] | 连接池、拨号、Happy Eyeballs | `include/prism/connect/` |
| [[core/pipeline/overview|Pipeline]] | 协议处理器、管道原语 | `include/prism/pipeline/` |
| [[core/protocol/overview|Protocol]] | 协议格式、常量、配置 | `include/prism/protocol/` |
| [[core/stealth/overview|Stealth]] | TLS 伪装（Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel/ECH） | `include/prism/stealth/` |
| [[core/recognition/overview|Recognition]] | 协议识别、探测分析 | `include/prism/recognition/` |
| [[core/resolve/overview|Resolve]] | DNS 解析、缓存、规则 | `include/prism/resolve/` |
| [[core/multiplex/overview|Multiplex]] | Smux/Yamux 多路复用 | `include/prism/multiplex/` |
| [[core/crypto/overview|Crypto]] | AEAD/HKDF/X25519/Blake3 加密模块 | `include/prism/crypto/` |
| [[outbound/overview|Outbound]] | 出站代理、直连 | `include/prism/outbound/` |
| [[core/memory/overview|Memory]] | PMR 内存池 | `include/prism/memory/` |
| [[core/fault/overview|Fault]] | 错误码系统 | `include/prism/fault/` |
| [[core/exception/overview|Exception]] | 异常体系 | `include/prism/exception/` |
| [[core/trace/overview|Trace]] | 日志系统 | `include/prism/trace/` |
| [[core/transformer/overview|Transformer]] | JSON 序列化 | `include/prism/transformer/` |
| [[core/loader/overview|Loader]] | 配置加载 | `include/prism/loader/` |

## 架构总览

Prism 采用六阶段流水线架构：

```
入站 (Incoming) → 识别 (Recognition) → 分发 (Dispatch) → 管道 (Pipeline) → 通道 (Channel) → 出站 (Outbound)
```

详见 [[architecture|六阶段流水线架构]]

## 启动流程

启动顺序 9 步：

1. `memory::system::enable_global_pooling()` — 初始化全局内存池
2. `stealth::register_all_schemes()` — 注册所有 TLS 伪装方案
3. 加载配置文件 (`loader::load()`)
4. 初始化日志系统 (`trace::init()`)
5. 构建账户目录 (`loader::build_account_directory()`)
6. 创建 worker 线程池
7. 构建负载均衡器 (`balancer`)
8. 启动监听器 (`listener`)
9. 启动所有线程

详见 [[startup|启动流程详解]]

## 协议处理流程

```
listener 接收连接 → balancer 选择 worker → worker 启动会话
→ session 调用 recognition::recognize() → dispatch 获取 handler
→ handler 调用 pipeline → router 建立上游 → tunnel 双向转发
```

详见 [[flow|协议处理流程详解]]

## 基础设施

以下模块提供底层支持：

- [[core/memory/overview|Memory]] — PMR 内存池，热路径零堆分配
- [[core/fault/overview|Fault]] — 错误码枚举，热路径返回错误码而非抛异常
- [[core/exception/overview|Exception]] — 异常层次，启动/致命错误使用
- [[core/trace/overview|Trace]] — 日志系统，spdlog 实现
- [[core/transformer/overview|Transformer]] — JSON 序列化，glaze 库
- [[core/loader/overview|Loader]] — 配置加载适配器

详见 [[infrastructure|基础设施总览]]