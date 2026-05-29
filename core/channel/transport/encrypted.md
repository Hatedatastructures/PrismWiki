---
tags: [transport, encrypted]
layer: core
source: include/prism/transport/encrypted.hpp
title: encrypted — TLS 加密传输层
module: transport
updated: 2026-05-27
---

# encrypted — TLS 加密传输层

将 `ssl::stream<connector>` 适配为 [[core/channel/transport/transmission|transmission]] 接口。持有 TLS 流的共享所有权，使协议装饰器能透明操作加密连接。

## 核心接口

| 方法 | 说明 |
|------|------|
| `encrypted(shared_stream)` | 从已握手的 TLS 流构造 |
| `transport_type()` | 返回 `type::tcp` |
| `next_layer()` | 返回 nullptr（叶子节点，connector 非 transmission） |
| `executor()` | TLS 流的执行器 |
| `async_read_some(buf, ec)` | TLS 异步读取 |
| `async_write_some(buf, ec)` | TLS 异步写入 |
| `close()` | 跳过 SSL_shutdown，直接关闭底层 socket |
| `cancel()` | 取消底层传输的挂起操作 |
| `stream()` | TLS 流引用，用于直接操作 |
| `release()` | 释放 TLS 流所有权 |
| `ssl_handshake(inbound, ctx)` | 静态工厂：握手成功返回 TLS 流，失败恢复原始传输 |

### 类型

| 类型 | 说明 |
|------|------|
| `connector_type` | `transport::connector`，封装 socket + transmission |
| `stream_type` | `ssl::stream<connector_type>` |
| `shared_stream` | `shared_ptr<stream_type>` |

## 设计决策

### 为什么 close() 跳过 SSL_shutdown？

SSL_shutdown 在非阻塞模式下可能阻塞等待对端 close_notify 响应，增加关闭延迟。代理场景中快速释放资源比优雅关闭更重要——对端收到 TCP RST 而非 close_notify 是可接受的。

**后果**: 对端日志可能出现 "unexpected EOF" 或 "connection reset" 警告。

**源码依据**: `encrypted.hpp:141-144`

### 为什么 ssl_handshake 是静态工厂方法返回三元组？

TLS 握手可能失败（客户端不支持、证书错误），此时需要从 connector 中恢复原始 transmission 的所有权。返回 `(fault::code, shared_stream, shared_transmission)`：成功时 stream 有效、transmission 为 nullptr；失败时 stream 为 nullptr、transmission 从 connector 的 release() 恢复。

**后果**: 调用方在握手失败时不丢失传输层，可尝试其他方案（如降级到明文协议）。

**源码依据**: `encrypted.hpp:200-201`

### 为什么 encrypted 的 next_layer() 返回 nullptr？

encrypted 内部持有 `ssl::stream<connector>`，而 connector 不是 transmission 子类。encrypted 在装饰器链中是叶子节点，`lowest_layer<>()` 无法穿透它获取底层 TCP socket。

**后果**: 需要访问底层 TCP socket 时必须通过 `stream().lowest_layer().transmission()` 显式穿透 ssl::stream 和 connector。

**源码依据**: `encrypted.hpp:78-86`

## 约束

### TLS 流必须在构造前完成握手

**类型**: 状态前置

**规则**: 构造函数接收的 shared_stream 必须已完成 TLS 握手。未握手的流会导致后续读写失败。

**违反后果**: async_read_some/async_write_some 返回 TLS 协议错误。

**源码依据**: `encrypted.hpp:57` `@details`

### 关闭后不可再调用任何方法

**类型**: 状态前置

**规则**: close() 或 release() 后对象不再持有有效流，后续调用是未定义行为。

**违反后果**: 解引用空指针，崩溃。

**源码依据**: `encrypted.hpp:43` `@warning`

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/channel/transport/transmission|transmission]] | 抽象基类 |
| [[core/transport/adapter/connector|connector]] | 底层连接器 |

### 被引用

| 模块 | 使用方式 |
|------|----------|
| recognition 方案执行器 | 握手后用 `ssl_handshake` 创建 encrypted |
| protocol 处理器 | 通过 transmission 接口透明读写 TLS 数据 |
