---
title: "shadowsocks.hpp — SS2022 Pipeline 处理器"
source: "include/prism/pipeline/protocols/shadowsocks.hpp"
module: "pipeline"
type: api
tags: [pipeline, protocols, shadowsocks, SS2022, AEAD]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/primitives|primitives]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[protocol/shadowsocks/relay|SS2022 relay]]"
  - "[[ref/crypto/aes-gcm|AEAD 加密]]"
  - "[[ref/anti-censorship/replay-attack|防重放]]"
---

# shadowsocks.hpp

> 源码: `include/prism/pipeline/protocols/shadowsocks.hpp`
> 实现: `src/prism/pipeline/protocols/shadowsocks.cpp`
> 模块: [[pipeline|Pipeline]] / protocols

## 概述

声明 Shadowsocks 2022 协议的会话处理函数，负责 SS2022 协议握手、AEAD 解密验证、时间戳验证、地址解析和数据转发。SS2022 无正特征，通过排除法 fallback 到此处理器。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[pipeline/primitives|primitives]] | 隧道原语（forward, wrap_with_preview） |
| 依赖 | [[protocol/shadowsocks/relay|relay]] | SS2022 中继器（握手、AEAD 加解密、salt pool） |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[agent/dispatch/table|table]] | handler_table 注册 shadowsocks |

## 命名空间

`psm::pipeline`

---

## 函数: shadowsocks()

> 源码: `include/prism/pipeline/protocols/shadowsocks.hpp:29`
> 实现: `src/prism/pipeline/protocols/shadowsocks.cpp:10`

### 功能

SS2022 协议处理函数，执行 AEAD 解密验证、时间戳验证、地址解析，并通过乐观响应后建立双向隧道转发。

### 签名

```cpp
auto shadowsocks(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，包含入站传输和配置信息 |
| data | `std::span<const std::byte>` | 预读数据，协议检测时读取的初始数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象，处理完成后返回。

### 调用（向下）

- [[pipeline/primitives|wrap_with_preview]] — 包装入站传输（use_global_mr=true）
- `protocol::shadowsocks::make_relay` — 创建 SS2022 中继器（含 salt pool）
- `protocol::shadowsocks::relay::handshake` — AEAD 解密验证、时间戳验证、地址解析
- `protocol::shadowsocks::relay::acknowledge` — 乐观响应（先发送 acknowledge 再拨号）
- [[pipeline/primitives|forward]] — 拨号 + 隧道转发（relay 本身作为 inbound，AEAD 加解密持续进行）

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表中注册为 SS2022 处理器

### 涉及的知识域

- [[ref/crypto/aes-gcm|AEAD 加密]] — AES-128/256-GCM 认证加密
- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — SS2022 支持的 AEAD 算法
- [[ref/crypto/blake3|BLAKE3]] — SS2022 密钥派生
- [[ref/anti-censorship/replay-attack|防重放]] — salt pool 防重放机制

### 流程

1. 包装入站传输（use_global_mr=true）
2. 获取或创建 worker 线程独占的 salt pool（thread_local，无需锁）
3. 创建 SS2022 中继器
4. 执行握手：AEAD 解密请求 → 验证时间戳 → 解析目标地址
5. 乐观响应：先发送 acknowledge 再拨号（与 mihomo 一致）
6. 拨号 + 隧道转发（relay 本身作为 inbound，AEAD 加解密持续进行）

### 错误处理

- 握手失败时记录日志并静默关闭连接
- acknowledge 失败时记录日志并静默关闭连接

### 注意事项

- SS2022 无正特征，通过排除法 fallback 到此处理器
- salt pool 使用 thread_local 保证每个工作线程独立实例，无需锁
- relay 本身作为 inbound 传入 tunnel，AEAD 加解密在 relay 层透明进行
- 乐观响应模式：先发送 acknowledge 再拨号，减少 RTT
