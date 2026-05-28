---
tags: [transport, reliable]
layer: core
source: I:/code/Prism/include/prism/transport/reliable.hpp
title: reliable — TCP 可靠传输
module: transport
updated: 2026-05-27
---

# reliable — TCP 可靠传输

TCP 传输层实现，继承 [[core/transport/transmission|transmission]]。封装 `tcp::socket`，支持连接池复用。所有异步操作返回 `net::awaitable`。

## 核心接口

| 方法 | 说明 |
|------|------|
| `reliable(executor)` | 从执行器构造，socket 未打开 |
| `reliable(socket)` | 从已连接 socket 构造 |
| `reliable(pooled_connection)` | 从连接池连接构造 |
| `async_read_some(buf, ec)` | 异步读取，Boost 错误码→项目错误码 |
| `async_write_some(buf, ec)` | 异步写入 |
| `async_write_scatter(bufs, count, ec)` | scatter-gather 写入，`count==2` 时 `[[likely]]` 优化 |
| `shutdown_write()` | TCP 半关闭 |
| `close()` | 连接池连接仅 cancel，普通连接 close |
| `cancel()` | 取消挂起的异步操作 |
| `native_socket()` | 直接访问底层 socket |

## 设计决策

### 为什么 close() 对连接池连接只 cancel 不关闭？

`pooled_connection` 是 RAII 包装器。`close()` 时仅 `cancel()` 挂起操作，保持 socket 打开。析构时 `pooled_.reset()` 触发 `recycle()` 归还连接池。`recycle()` 通过 `healthy_fast()` 检测 socket 健康状态，健康则复用。

**后果**: 连接池连接的 `close()` 语义与普通连接不同 — socket 未关闭，可被复用。

### 为什么 async_write_scatter 对 count==2 优化？

代理场景中最常见的 scatter 模式是帧头 + 载荷（如 Trojan 帧头 56 字节 + 数据）。`[[likely]]` 分支用 `std::array<const_buffer, 2>` 构造 buffer 序列，底层 `async_write` 映射为单次 `WSASend`/`writev` 系统调用。避免两次 write 导致的额外系统调用和 TLS 记录开销。

**后果**: `count > 2` 的路径没有优化（逐个写入），但实际场景中不存在。

### 为什么继承 enable_shared_from_this？

`reliable` 的异步操作需要在协程挂起期间保持对象存活。`shared_from_this()` 让协程安全持有 `shared_ptr<reliable>`。

**后果**: 必须通过 `shared_ptr` 管理 `reliable` 的生命周期，不能栈上构造。

## 约束

### native_socket() 的双路径

**类型**: 状态前置

**规则**: `native_socket()` 根据是否为连接池连接返回不同来源的 socket。`pooled_.valid()` 时返回池连接的 socket，否则返回 `socket_` 的 socket。两者不会同时存在。

**违反后果**: 对已 `close()` 的普通连接调用 `native_socket()` 返回已关闭的 socket，后续操作返回 `bad_descriptor`。

**源码依据**: `reliable.hpp` native_socket 实现

## 工厂函数

| 函数 | 说明 |
|------|------|
| `make_reliable(executor)` | 创建未打开的传输层 |
| `make_reliable(socket)` | 从已连接 socket 创建 |
| `make_reliable(pooled)` | 从连接池连接创建 |

## 引用关系

### 依赖

- [[core/transport/transmission|transmission]]：抽象基类
- [[core/connect/pool/pool|pooled_connection]]：连接池 RAII 包装器

### 被引用

- [[core/transport/encrypted|encrypted]]：TLS 传输层包装 reliable
- [[core/instance/worker/launch|launch]]：`make_reliable(socket)` 创建入站传输
- [[core/outbound/direct|direct]]：`make_reliable(pooled)` 创建出站传输
