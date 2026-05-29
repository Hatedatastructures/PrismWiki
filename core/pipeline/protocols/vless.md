---
tags: [pipeline, vless, protocol-handler, uuid, mux]
title: "VLESS 协议处理管道"
layer: core
source: "include/prism/pipeline/protocols/vless.hpp"
module: pipeline
updated: 2026-05-27
---

# VLESS 协议处理管道

VLESS 协议处理器，处理 TLS 剥离后的内层流量。负责 UUID 验证、命令分发和多路复用引导。结构与 Trojan 类似，但凭证类型为 UUID。

## 处理流程

1. `wrap_with_preview(use_global_mr=true)` — mux 场景需要全局内存池
2. `account::try_acquire()` — UUID 验证
3. 创建 VLESS relay，执行握手（解析 UUID + ADDONS + 目标地址 + 命令）
4. TCP：检测 mux 标记 → `forward()` 或 `multiplex::bootstrap()`
5. UDP：`make_datagram_router()` → `async_associate()`

## 协议格式

```
VER(1) | UUID(16) | ADDONS(Variable) | CMD(1) | ADDR(Variable) | PORT(2)
```

| 命令 | 值 | 说明 |
|------|-----|------|
| tcp | 0x01 | TCP 隧道 |
| udp | 0x02 | UDP 中继 |
| mux | 检测标记 | 多路复用（`.mux.sing-box.arpa` 后缀） |

## 设计决策

### 为什么 VLESS 与 Trojan 共享相似的管道结构？

两者都是"TLS 外层 + 自定义内层"的架构。TLS 握手在 Session 层统一完成，内层只需解析凭证 + 目标地址 + 命令。共享 `forward()`、`tunnel()`、`make_datagram_router()` 等原语。

**后果**: 两者代码结构高度相似，区别仅在凭证类型（UUID vs SHA224）和协议格式。

### 为什么 VLESS 有 ADDONS 字段？

VLESS 的 ADDONS 用于扩展信息（如 XTLS 流控）。Prism 当前不解析 ADDONS 内容，跳过后继续读取命令和地址。

**后果**: 未来如果支持 XTLS，需要在握手阶段解析 ADDONS。

## 引用关系

### 被调用

- [[core/instance/dispatch/table|dispatch]] — 注册为 VLESS 处理器

### 依赖

- [[core/pipeline/primitives|primitives]] — `wrap_with_preview`, `forward`, `make_datagram_router`
- [[core/account/directory|account::directory]] — `try_acquire()` UUID 验证
- [[core/multiplex/overview|multiplex]] — `bootstrap()` 多路复用引导
