---
layer: core
source: I:/code/Prism/include/prism/instance/worker/tls.hpp
title: tls — TLS 上下文初始化模块
module: instance/worker
tags: [instance, worker, tls, ssl, boringssl]
updated: 2026-05-27
---

# tls — TLS 上下文初始化模块

Worker 初始化阶段调用的 TLS 上下文工厂。根据配置加载证书链和私钥，设置 GREASE 扩展和 ALPN。未配置证书时返回空指针，表示明文模式。

## 核心接口

| 函数 | 说明 |
|------|------|
| `make(cfg)` → `shared_context` | 根据配置创建 TLS 上下文，证书/私钥任一为空则返回 nullptr |
| `configure(ctx, cert, key)` | 初始化 TLS 上下文：加载证书链 + 私钥 + GREASE + ALPN |

### 类型

| 类型 | 说明 |
|------|------|
| `shared_context` | `shared_ptr<ssl::context>`，所有 worker 共享同一实例 |

## 设计决策

### 为什么所有 worker 共享同一个 ssl::context？

`ssl::context` 是线程安全的只读对象（创建后不再修改）。所有 worker 共享同一个实例避免重复加载证书/私钥，节省内存和启动时间。每个连接持有独立的 `ssl::stream`，不共享可变状态。

**后果**: 热更新证书需要替换整个 shared_context 指针，不能在运行中修改现有 context。

### 为什么证书/私钥加载失败用异常而非错误码？

`make()` 在 Worker 启动阶段调用，属于"致命路径"——没有有效的 TLS 上下文就无法服务 HTTPS 流量。启动路径异常符合 Prism 的双轨错误策略：热路径用 `fault::code`，启动路径用异常。

**后果**: 调用方（worker 构造）必须 try-catch，否则整个进程终止。

### 为什么设置 verify_none？

服务端模式不验证客户端证书（Prism 是代理服务器，不是 CA 验证服务）。客户端身份验证由上层协议（Trojan 密码、VLESS UUID、SS2022 PSK）完成，不依赖 TLS 客户端证书。

**后果**: 任何客户端都能完成 TLS 握手，安全性依赖应用层协议。

### configure 的配置顺序

| 阶段 | 操作 | 失败行为 |
|------|------|----------|
| 1 | `set_options(default_workarounds)` | — |
| 2 | `set_verify_mode(verify_none)` | — |
| 3 | `use_certificate_chain_file(cert)` | 抛 `exception::protocol` |
| 4 | `use_private_key_file(key, pem)` | 抛 `exception::protocol` |
| 5 | `set_options(enable_grease)` | — |
| 6 | `set_alpn_protocols({"h2", "http/1.1"})` | — |

## 约束

### configure 必须在 worker 启动前成功

**类型**: 调用顺序

**规则**: `make()` 在 worker 构造时调用。如果抛出异常，worker 无法启动。

**违反后果**: 进程启动失败（这是预期行为——配置错误应尽早暴露）。

**源码依据**: `tls.hpp:9` `@note`

### 证书和私钥必须同时提供

**类型**: 状态前置

**规则**: `make()` 要求 `cfg` 中 `tls_cert` 和 `tls_key` 都非空。任一为空则返回 nullptr（明文模式）。

**违反后果**: 无意中漏配一个字段会导致明文模式而非 TLS 模式，客户端 TLS 连接会被拒绝。

### 不支持加密私钥

**类型**: 资源限制

**规则**: `use_private_key_file` 只支持未加密的 PEM 格式私钥。

**违反后果**: 加载加密私钥会抛出异常。

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/instance/config|config]] | 提供证书/私钥路径配置 |

### 被引用

| 模块 | 使用方式 |
|------|----------|
| [[core/instance/worker/worker|worker]] | 构造时调用 `tls::make(cfg)` 获取 `ssl_ctx_` |
| [[core/instance/worker/launch|launch]] | session 启动时检查 `ssl_ctx_` 是否存在以决定 TLS/明文路径 |
