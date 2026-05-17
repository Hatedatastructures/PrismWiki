---
title: "http.hpp — HTTP 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/http.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# http.hpp

> 源码: `include/prism/pipeline/protocols/http.hpp`
> 实现: `src/prism/pipeline/protocols/http.cpp`
> 模块: Pipeline / protocols

## 概述

HTTP 协议处理管道，处理 HTTP 代理请求。支持 CONNECT 方法（HTTPS 隧道）和普通 HTTP 请求转发。

## 命名空间

`psm::pipeline`

---

## 函数: http()

> 源码位置: `http.hpp:33`
> 实现位置: `http.cpp:11`

### 功能

HTTP 代理协议处理函数，创建 HTTP 中继器执行握手，连接上游服务器，建立隧道转发。

### 签名

```cpp
auto http(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文 |
| data | `std::span<const std::byte>` | 预读数据（HTTP 请求头首字节） |

### 返回值

`net::awaitable<void>` — 异步操作对象

### 处理流程

```
+------------------+
| 重置帧内存池      |
| frame_arena.reset() |
+------------------+
         │
         ▼
+------------------+
| 包装入站传输      |
| wrap_with_preview() |
+------------------+
         │
         ▼
+------------------+
| 创建 HTTP relay   |
| http::make_relay() |
+------------------+
         │
         ▼
+------------------+
| 执行握手          |
| relay::handshake() |
| 读取请求头 + 解析  |
+------------------+
         │
         ▼
+------------------+
| 解析目标地址      |
| analysis::resolve() |
+------------------+
         │
         ▼
    +----+----+
    │         │
CONNECT   普通 HTTP
    │         │
    ▼         ▼
拨号上游    重写 URI
发送 200    转发请求体
    │         │
    +----+----+
         │
         ▼
+------------------+
| 建立双向隧道      |
| tunnel()         |
+------------------+
```

### 调用链（向下）

- [[core/pipeline/primitives#wrap_with_preview|wrap_with_preview]] — 包装入站传输
- `protocol::http::make_relay` — 创建 HTTP 中继器
- `protocol::http::relay::handshake` — 执行握手
- `protocol::analysis::resolve` — 解析目标地址
- [[core/pipeline/primitives#dial|dial]] — 拨号上游
- `protocol::http::relay::write_connect_success` — 发送 200 响应
- `protocol::http::relay::write_bad_gateway` — 发送 502 响应
- [[core/pipeline/primitives#tunnel|tunnel]] — 建立隧道

### 被调用（向上）

- [[agent/dispatch/table|dispatch]] — 协议分发表注册为 HTTP 处理器

### 实现细节

```cpp
auto http(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>
{
    // 重置帧内存池
    ctx.frame_arena.reset();
    
    // 包装入站传输（处理预读数据）
    auto inbound = primitives::wrap_with_preview(ctx, data);
    
    // 创建 HTTP 中继器
    auto relay = protocol::http::make_relay(std::move(inbound), 
                                            ctx.account_directory_ptr);
    
    // 执行握手
    auto [ec, req] = co_await relay->handshake();
    if (fault::failed(ec)) {
        trace::warn("{} handshake failed: {}", HttpStr, fault::describe(ec));
        co_return;
    }
    
    // 解析目标地址
    const auto target = protocol::analysis::resolve(req);
    
    // 连接上游
    auto [dial_ec, outbound] = ctx.outbound_proxy
        ? co_await primitives::dial(*ctx.outbound_proxy, target, ...)
        : co_await primitives::dial(ctx.worker.router, "HTTP", target, ...);
    
    if (fault::failed(dial_ec) || !outbound) {
        co_await relay->write_bad_gateway();
        co_return;
    }
    
    // 命令分发
    if (req.method == "CONNECT") {
        co_await relay->write_connect_success();
        co_await primitives::tunnel(relay->release(), outbound, ctx);
    } else {
        co_await relay->forward(req, outbound, ctx.frame_arena.get());
        co_await primitives::tunnel(relay->release(), outbound, ctx);
    }
}
```

### 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| 握手失败 | 记录日志 + 静默关闭 |
| 认证失败 | relay 内部发送 407/403 |
| 上游连接失败 | 发送 502 Bad Gateway |
| 隧道传输错误 | 隧道层自动处理 |

---

## HTTP 协议特点

### CONNECT 方法

用于建立 HTTPS 隧道或通过 HTTP 代理访问非 HTTP 服务：

```
客户端 -> CONNECT example.com:443 -> 代理 -> DNS + TCP 连接 -> 目标
客户端 <- 200 Connection Established <- 代理
客户端 -> TLS ClientHello (加密) -> 代理 -> 透明转发 -> 目标
```

### 普通 HTTP 请求转发

需要重写 URI（绝对路径转为相对路径）：

```
原始请求: GET http://example.com/path HTTP/1.1
重写后:   GET /path HTTP/1.1
```

### Basic 认证

通过 `Proxy-Authorization` 头传递用户凭证：

```
Proxy-Authorization: Basic base64(username:password)
```

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/pipeline/primitives|管道原语]]
- [[protocol/http/relay|HTTP 中继器]]
- [[protocol/analysis|目标地址解析]]