---
title: "http.hpp — HTTP 协议处理管道"
source: "include/prism/pipeline/protocols/http.hpp"
module: "pipeline"
type: api
tags: [pipeline, protocols, http, HTTP, proxy]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[pipeline/primitives|primitives]]"
  - "[[agent/dispatch/table|dispatch table]]"
  - "[[protocol/http/relay|HTTP relay]]"
  - "[[protocol/analysis|analysis]]"
  - "[[ref/protocol/http-connect|HTTP 代理协议]]"
---

# http.hpp

> 源码: `include/prism/pipeline/protocols/http.hpp`
> 实现: `src/prism/pipeline/protocols/http.cpp`
> 模块: [[pipeline|Pipeline]] / protocols

## 概述

声明 HTTP 代理协议的会话处理函数，包括请求解析、代理认证、上游连接建立以及双向隧道转发。支持 CONNECT 方法（HTTPS 隧道）和普通 HTTP 请求转发。协议级逻辑由 `http::relay` 封装，pipeline 仅负责编排。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context 定义 |
| 依赖 | [[pipeline/primitives|primitives]] | 隧道原语（dial, tunnel, wrap_with_preview） |
| 依赖 | [[protocol/http/relay|relay]] | HTTP 中继器（握手、解析、转发） |
| 依赖 | [[protocol/analysis|analysis]] | 目标地址解析 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[agent/dispatch/table|table]] | handler_table 注册 http |

## 命名空间

`psm::pipeline`

---

## 函数: http()

> 源码: `include/prism/pipeline/protocols/http.hpp:33`
> 实现: `src/prism/pipeline/protocols/http.cpp:11`

### 功能

HTTP 代理协议处理函数，创建 HTTP 中继器执行握手（读取请求头、解析、认证），连接上游，并根据请求方法建立隧道转发。

### 签名

```cpp
auto http(session_context &ctx, std::span<const std::byte> data) -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文，包含入站传输和配置信息 |
| data | `std::span<const std::byte>` | 预读数据，协议检测时读取的初始数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象，处理完成后返回。

### 调用（向下）

- [[pipeline/primitives|wrap_with_preview]] — 包装入站传输（如有预读数据则用 preview 装饰器重放）
- [[protocol/http/relay|make_relay]] — 创建 HTTP 中继器
- [[protocol/http/relay|relay::handshake]] — 读取请求头、解析、认证
- [[protocol/analysis|analysis::resolve]] — 解析目标地址
- [[pipeline/primitives|dial]]（出站代理重载） — 通过出站代理拨号
- [[pipeline/primitives|dial]]（路由器重载） — 通过路由器拨号
- [[protocol/http/relay|relay::write_connect_success]] — CONNECT 方法发送 200 响应
- [[protocol/http/relay|relay::write_bad_gateway]] — 拨号失败时发送 502 响应
- [[protocol/http/relay|relay::forward]] — 普通 HTTP 请求重写 URI 后转发
- [[pipeline/primitives|tunnel]] — 建立双向隧道

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表中注册为 HTTP 处理器

### 涉及的知识域

- [[ref/protocol/http-connect|HTTP 代理协议]] — HTTP CONNECT 方法和普通代理转发
- [[ref/protocol/socks5-rfc1928|CONNECT 方法]] — CONNECT 隧道语义

### 流程

1. 重置帧内存池
2. 包装入站传输（如有预读数据则用 preview 装饰器重放）
3. 创建 HTTP 中继并握手（读取请求头 + 解析 + 认证）
4. 解析目标地址
5. 连接目标服务器（优先出站代理，回退路由器）
6. 按方法分发：
   - **CONNECT**：发送 200 响应，释放传输层，进入隧道
   - **普通请求**：重写 URI 后转发原始数据，然后进入隧道

### 错误处理

- 握手失败时静默关闭连接
- 拨号失败时向客户端发送 502 Bad Gateway 响应
- CONNECT 响应发送失败时静默关闭连接

### 注意事项

- 协议级逻辑由 `http::relay` 封装，pipeline 仅负责编排
- 请求解析失败或连接建立失败时静默关闭连接，不抛出异常
