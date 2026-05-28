---
tags: [channel, transport, transmission]
layer: core
source: I:/code/Prism/include/prism/transport/transmission.hpp
title: transmission — 传输层流式抽象接口
module: transport
updated: 2026-05-27
---

# transmission — 传输层流式抽象接口

传输层核心抽象基类，参照 Boost.Asio AsyncStream 概念设计。所有具体传输（TCP、UDP）和协议装饰器（preview、snapshot、seal）都继承此接口。

## 核心接口

| 方法 | 说明 |
|------|------|
| `transport_type()` | 传输类型标识（tcp/udp），装饰器沿 next_layer() 委托到底层 |
| `executor()` | 获取关联执行器（纯虚） |
| `get_executor()` | Asio Concept 兼容，委托给 `executor()` |
| `async_read_some(buf, ec)` | 异步部分读取（纯虚，核心操作） |
| `async_write_some(buf, ec)` | 异步部分写入（纯虚，核心操作） |
| `async_read_some(buf, handler)` | Completion-handler 风格读取，为 ssl::stream 适配 |
| `async_write_some(buf, handler)` | Completion-handler 风格写入 |
| `close()` | 关闭传输层（纯虚） |
| `cancel()` | 取消挂起异步操作（纯虚） |
| `next_layer()` | 装饰器链导航，叶子节点返回 nullptr |
| `lowest_layer<T>()` | 沿链走到链底，dynamic_cast 为目标类型 |

### 自由函数（非成员方法）

| 函数 | 说明 |
|------|------|
| `transport::async_write(t, buf, ec)` | 循环 async_write_some 直到全部写入 |
| `transport::async_read(t, buf, ec)` | 循环 async_read_some 直到缓冲区填满 |

### 类型定义

| 名称 | 说明 |
|------|------|
| `shared_transmission` | `shared_ptr<transmission>`，传输层生命周期管理 |
| `type` 枚举 | `tcp` / `udp`，替代旧的 `is_reliable()` |

## 设计决策

### 为什么 async_write/async_read 是自由函数而不是成员方法？

Boost.Asio 的设计模式：组合操作（完整读写）不是流接口的一部分，而是外部算法。`transmission` 只定义最小核心操作（async_read_some、async_write_some），完整读写由 `transport::async_write()` / `transport::async_read()` 自由函数提供。好处是子类不需要重写完整读写逻辑，所有传输类型共享同一份实现。

**后果**: 调用方式为 `co_await transport::async_write(*trans, buf, ec)` 而非 `co_await trans->async_write(buf, ec)`。

### 为什么用 transport_type() 枚举替代 is_reliable()？

`is_reliable()` 只区分可靠/不可靠，无法区分 TCP 上的加密传输。`transport_type` 枚举明确标识底层协议，装饰器通过 `next_layer()` 链委托到底层获取真实类型。

**后果**: 所有使用 `is_reliable()` 的地方需要迁移到 `transport_type()`。

### 为什么提供 completion-handler 风格的重载？

BoringSSL 的 `ssl::stream` 使用 Asio completion-token 模式（回调），不是 `co_await`。为让 `ssl::stream` 直接包装 transmission，需要提供 `any_completion_handler` 签名的重载。

默认实现通过 `co_spawn` 桥接到 awaitable 接口。热路径实现（reliable、preview）直接覆写，避免协程开销。

**后果**: 每次桥接有一次 co_spawn 开销，但仅 ssl::stream 路径使用，热路径无影响。

### 为什么用 next_layer()/lowest_layer<T>() 替代 dynamic_cast 链式解包？

旧方式是 `dynamic_cast<preview*>(trans)` / `dynamic_cast<snapshot*>(trans)` 逐类型剥壳。`next_layer()` 提供统一的装饰器链导航，`lowest_layer<T>()` 一次走到底再转型。

**后果**: `lowest_layer<T>()` 内部仍使用 `dynamic_cast`，但只做一次而非 N 次。RTTI 编译选项不可关闭。

## 约束

### 装饰器必须正确转发 next_layer()

**类型**: 接口契约

**规则**: 所有装饰器（preview、snapshot、seal、protocol conn）必须覆写 `next_layer()` 返回被包装的内层传输。叶子节点（reliable、unreliable）返回 nullptr。

**违反后果**: `transport_type()` 和 `lowest_layer<T>()` 导航断裂，加密层无法找到底层 TCP 传输。

**源码依据**: `transmission.hpp:185-189`

### 错误码转换的 category 匹配

**类型**: 状态前置

**规则**: `detail::to_ec()` 将 `std::error_code` 转为 `boost::system::error_code`。fault category 的错误码映射到 `boost::system::category()`，其他映射到 `generic_category()`。

**违反后果**: 如果 fault category 判断错误，Asio 层无法正确识别错误类型。

**源码依据**: `transmission.hpp:36-44`

## 引用关系

### 继承

- [[core/transport/reliable|reliable]] — TCP 可靠传输（叶子节点，type::tcp）
- [[core/transport/unreliable|unreliable]] — UDP 不可靠传输（叶子节点，type::udp）
- [[core/transport/encrypted|encrypted]] — TLS 加密传输（装饰器）
- [[core/transport/preview|preview]] — 预读重放装饰器
- [[core/transport/snapshot|snapshot]] — 传输层快照装饰器
- [[core/crypto/seal|seal]] — 加密封装装饰器
