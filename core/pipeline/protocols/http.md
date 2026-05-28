---
tags: [pipeline, http, protocol-handler, connect, forward]
title: "HTTP 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/http.hpp"
module: pipeline
updated: 2026-05-27
---

# HTTP 协议处理管道

HTTP 代理协议处理器。支持 CONNECT 方法（HTTPS 隧道）和普通 HTTP 请求转发。无内层加密，是最简单的协议管道。

## 处理流程

1. `wrap_with_preview()` 包装入站传输（含预读的 HTTP 请求行）
2. 创建 HTTP relay，执行握手（读取请求头 + Basic Auth 验证）
3. `analysis::resolve()` 解析目标地址
4. CONNECT 方法：拨号上游 → 发送 200 → `tunnel()` 双向转发
5. 普通 HTTP：重写 URI（绝对路径→相对路径）→ 转发请求体 → `tunnel()`

## 设计决策

### 为什么 HTTP 不需要 mux？

HTTP 代理每次请求都是独立的 TCP 连接，协议本身没有多路复用概念。CONNECT 方法建立隧道后就是透传 TCP，上层协议（TLS）才有多路复用的可能。

**后果**: HTTP 管道最简单，不涉及 mux 检测。

### 为什么普通 HTTP 需要重写 URI？

客户端发送 `GET http://example.com/path HTTP/1.1`，但上游服务器期望 `GET /path HTTP/1.1`。代理需要将绝对 URI 转为相对路径。

**后果**: 非 CONNECT 请求需要 relay 解析并重写请求行。

## 错误处理

| 场景 | 响应 |
|------|------|
| 握手失败 | 静默关闭连接 |
| 认证失败 | relay 发送 407 Proxy Auth Required |
| 上游连接失败 | 发送 502 Bad Gateway |

## 引用关系

### 被调用

- [[core/instance/dispatch/table|dispatch]] — 注册为 HTTP 处理器

### 依赖

- [[core/pipeline/primitives|primitives]] — `wrap_with_preview`, `dial`, `tunnel`
