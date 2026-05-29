---
tags: [pipeline, trojan, protocol-handler, sha224, mux]
title: "Trojan 协议处理管道"
layer: core
source: "include/prism/pipeline/protocols/trojan.hpp"
module: pipeline
updated: 2026-05-27
---

# Trojan 协议处理管道

Trojan 协议处理器，处理 TLS 剥离后的内层流量。TLS 握手在 Session 层完成，本模块负责凭证验证（SHA224）、命令分发和多路复用引导。

## 处理流程

1. `wrap_with_preview(use_global_mr=true)` — mux 场景需要全局内存池
2. `account::try_acquire()` — SHA224 凭证验证
3. 创建 Trojan relay，执行握手（解析 SHA224 hash + 目标地址 + 命令）
4. CONNECT：检测 mux 标记 → `forward()` 或 `multiplex::bootstrap()`
5. UDP_ASSOCIATE：`make_datagram_router()` → `async_associate()`

## 协议格式

```
SHA224(password) | 地址 | CRLF | CMD
   56 bytes     | Variable | 2  | 1
```

| 命令 | 值 | 说明 |
|------|-----|------|
| CONNECT | 0x01 | TCP 隧道 |
| UDP_ASSOCIATE | 0x03 | UDP 中继 |
| mux 标记 | 0x7F | 直接 mux 模式 |

## 设计决策

### 为什么 mux 用 use_global_mr=true？

Trojan 的 mux 连接生命周期长于单个请求（跨越多个 stream）。帧竞技场（`frame_arena`）在请求结束时 reset，mux 场景下不适合管理长期存活的数据。全局内存池的生命周期与连接一致。

**后果**: mux 模式的内存分配走全局池而非帧竞技场。

### 为什么凭证用 SHA224？

Trojan 协议规范要求密码以 SHA224 哈希形式传输（56 字节 hex 字符串）。`loader::build_dir()` 预先将配置中的明文密码规范化为 SHA224，运行时直接比较哈希值。

**后果**: 密码不以明文传输，但 SHA224 哈希本身可作为凭证被重放（依赖 TLS 外层加密保护）。

### mux 检测的两种方式？

1. 命令字节为 0x7F — 直接 mux 模式，无需解析地址
2. 目标地址以 `.mux.sing-box.arpa` 结尾 — sing-box 兼容模式

**后果**: 两种模式最终都调用 `multiplex::bootstrap()`，区别在于是否需要先解析目标地址。

## 引用关系

### 被调用

- [[core/instance/dispatch/table|dispatch]] — 注册为 Trojan 处理器

### 依赖

- [[core/pipeline/primitives|primitives]] — `wrap_with_preview`, `forward`, `make_datagram_router`
- [[core/account/directory|account::directory]] — `try_acquire()` 凭证验证
- [[core/multiplex/overview|multiplex]] — `bootstrap()` 多路复用引导
