---
title: 传输层接口详解
module: "channel"
namespace: psm::channel::transport
files:
  - include/prism/channel/transport/transmission.hpp
  - include/prism/channel/transport/reliable.hpp
  - include/prism/channel/transport/encrypted.hpp
  - include/prism/channel/transport/unreliable.hpp
  - include/prism/channel/transport/snapshot.hpp
see_also:
  - "[[channel]]"
  - "[[stealth]]"
  - "[[memory/pool]]"
---

# 传输层接口详解

传输层是 [[channel]] 模块的核心基础设施，采用分层流式架构，为上层协议（Trojan、VLESS、Shadowsocks）
提供统一的异步读写抽象。所有实现基于纯协程设计，使用 `net::awaitable<T>` 作为异步操作返回类型。

## 分层架构

```
  snapshot (可回滚装饰器)
      │
  encrypted (TLS 加密层)
      │
  reliable (TCP 可靠传输)  /  unreliable (UDP 不可靠传输)
      │
  transmission (抽象基类)
```

上层代码通过 `shared_transmission`（即 `std::shared_ptr<transmission>`）持有传输层实例，
实现类型擦除和生命周期管理。

## 基类：transmission

`transmission` 是所有传输层的纯虚基类，定义了以下核心接口：

| 方法 | 纯虚 | 说明 |
|------|:----:|------|
| `async_read_some(span<byte>, ec)` | ✓ | 异步读取部分数据 |
| `async_write_some(span<const byte>, ec)` | ✓ | 异步写入部分数据 |
| `executor()` | ✓ | 返回关联的 Boost.Asio 执行器 |
| `close()` | ✓ | 关闭传输层 |
| `cancel()` | ✓ | 取消所有挂起的异步操作 |
| `is_reliable()` | | 可靠性标识，默认 `false` |
| `shutdown_write()` | | 半关闭写端，默认空实现 |
| `async_write(span, ec)` | | 完整写入，循环调用 `async_write_some` |
| `async_read(span, ec)` | | 完整读取，循环调用 `async_read_some` |
| `async_write_scatter(buffers, count, ec)` | | scatter-gather 写入 |

所有异步方法通过 `std::error_code&` 参数返回错误，避免异常开销。
`async_write` 和 `async_read` 是带循环的完整版本，子类（如 UDP）可重写。

## reliable — TCP 可靠传输

`reliable` 封装 `boost::asio::ip::tcp::socket`，是基于 TCP 的具体传输实现。

**核心特性：**

- `is_reliable()` 返回 `true`
- `native_socket()` 提供底层 socket 直接访问（如设置 `TCP_NODELAY`）
- `shutdown_write()` 调用 `socket.shutdown(shutdown_send)` 实现半关闭
- `async_write_scatter()` 对双缓冲区（帧头+载荷）场景使用原生 `async_write`，
  映射为单次 `WSASend`/`writev` 系统调用

**连接池复用：** 第三个构造函数接收 `pooled_connection`，析构或 `close()` 时
socket 不直接关闭，而是归还到 [[channel]] 连接池。池连接通过
`healthy_fast()` 检测健康状态，不健康连接自动销毁。

**工厂函数：**

```cpp
make_reliable(executor)         // 创建空 socket
make_reliable(tcp::socket)      // 从已有 socket
make_reliable(pooled_connection) // 从连接池
```

## encrypted — TLS 加密传输层

`encrypted` 将 `ssl::stream<connector>` 适配为 `transmission` 接口，
用于 TLS 加密场景。

**核心特性：**

- 持有 `ssl::stream` 的 `shared_ptr`（`shared_stream`）
- `is_reliable()` 返回 `true`（TLS 基于 TCP）
- `async_write_scatter()` 将多缓冲区合并为单次 `SSL_write`，
  避免两次加密操作和额外的 TLS 记录头开销
- `close()` 先调用 `SSL_shutdown` 发送 `close_notify`，再关闭底层传输
- `release()` 移动返回 `shared_stream` 所有权，用于协议装饰器接管 TLS 流

**工厂函数：**

```cpp
make_encrypted(shared_stream ssl_stream)
```

> 典型用法：TLS 握手完成后，将 `ssl::stream` 包装为 `encrypted`，
> 再交给 Trojan/VLESS 等协议装饰器处理。

## unreliable — UDP 不可靠传输

`unreliable` 封装 `boost::asio::ip::udp::socket`，模拟流式语义。

**核心特性：**

- `is_reliable()` 返回 `false`（默认值）
- 内部维护 `remote_endpoint_`，所有 `send_to` 指向该端点
- `async_read_some()` 使用 `async_receive_from`，自动过滤来源：
  - 若未设置远程端点，首次接收自动设为远程端点
  - 来源不匹配则丢弃并继续等待
- `async_write()` 重写为直接委托 `async_write_some()`（UDP 一次发送完成）

**工厂函数：**

```cpp
make_unreliable(executor, optional<endpoint>)
make_unreliable(udp::socket, optional<endpoint>)
```

## snapshot — 可回滚传输装饰器

`snapshot` 是传输层装饰器，自动捕获所有读取的字节到内部缓冲区，
支持 `rewind()` 回到起点重新读取。

**设计目的：** TLS 伪装方案的依次尝试——每个 scheme 读取的数据被 snapshot 捕获，
失败时 rewind，下一个 scheme 从同一起点重试。

**读取两阶段：**

1. 若 `captured_` 中有未回放数据（`read_pos_ < captured_.size()`），
   从缓冲区同步复制（零开销回放）
2. 否则从内层传输异步读取，同时追加到 `captured_`

**安全约束：**

- `rewind()` 仅在 `wrote_ == false` 时安全（通过 `can_rewind()` 检查）
- 认证阶段是纯读取，安全 rewind
- 一旦写入（如 TLS 握手），transport 状态不可恢复

内部使用 `memory::vector<std::byte>` 存储捕获数据，支持 PMR 内存资源注入。

**工厂函数：**

```cpp
make_snapshot(shared_transmission inner, memory::resource_pointer mr)
```

## 错误码映射

所有实现类在 Boost.Asio 错误码和项目错误码之间自动映射：

```
boost::system::error_code  →  fault::to_code()  →  fault::make_error_code()
```

自定义错误类别（`psm::fault::category()`）映射到 `boost::system::category()`，
通用类别映射到 `boost::system::generic_category()`。

## Boost.Asio 适配

`transmission.hpp` 还提供两个自由函数模板，将 `shared_transmission`
适配为标准 Boost.Asio CompletionToken 接口：

```cpp
async_read_some(shared_transmission, MutableBufferSequence, CompletionToken)
async_write_some(shared_transmission, ConstBufferSequence, CompletionToken)
```

内部通过 `async_initiate` 桥接，在独立协程中执行实际 IO 并转换错误码。
