---
tags: [transport, unreliable]
layer: core
source: I:/code/Prism/include/prism/transport/unreliable.hpp
title: unreliable — UDP 数据报传输
module: transport
updated: 2026-05-27
---

# unreliable — UDP 数据报传输

UDP 传输层实现，继承 [[core/transport/transmission|transmission]]。封装 `udp::socket`，通过远程端点实现连接式操作，接收时自动过滤非远程端点数据报。

## 核心接口

| 方法 | 说明 |
|------|------|
| `unreliable(executor, remote_endpoint?)` | 从执行器构造 |
| `unreliable(socket, remote_endpoint?)` | 从已打开 UDP socket 构造 |
| `async_read_some(buf, ec)` | 接收并过滤来源，`while(true)` 循环丢弃不匹配的数据报 |
| `async_write_some(buf, ec)` | 发送到远程端点，未设置时返回 `io_error` |
| `async_write(buf, ec)` | 直接委托给 `async_write_some`（UDP 无流式语义） |
| `close()` | 关闭 UDP socket |
| `cancel()` | 取消挂起的异步操作 |
| `set_remote_endpoint(ep)` | 设置发送目标 |
| `remote_endpoint()` | 返回当前远程端点 |
| `native_socket()` | 直接访问底层 socket |

## 设计决策

### 为什么 async_read_some 有 while(true) 来源过滤？

UDP socket 的 `async_receive_from` 接收来自任何端点的数据报。如果已设置远程端点，只接受来自该端点的数据，不匹配的丢弃并继续等待。SOCKS5 UDP ASSOCIATE 场景中，多个客户端可能向同一 socket 发送，来源过滤确保只处理关联客户端的数据。

**后果**: 如果远程端点不发送数据，`async_read_some` 会无限等待（受协程取消控制）。

### 为什么首次接收自动设置远程端点？

UDP 中客户端端口通常动态分配。如果未预设远程端点，首次接收的数据报来源自动设为远程端点，后续发送操作指向该端点。这是 UDP "连接模拟"的标准模式。

**后果**: 如果第一个数据报来自非预期来源，后续所有通信指向错误端点。

## 约束

### 未设置远程端点时 async_write_some 返回错误

**类型**: 状态前置

**规则**: `async_write_some` 要求 `remote_endpoint_` 已设置，否则返回 `fault::code::io_error`。

**违反后果**: 写入总是失败。

**源码依据**: `unreliable.hpp` async_write_some 实现

## 工厂函数

| 函数 | 说明 |
|------|------|
| `make_unreliable(executor, remote?)` | 创建传输层，socket 未打开 |
| `make_unreliable(socket, remote?)` | 从已打开 socket 创建 |

## 引用关系

### 依赖

- [[core/transport/transmission|transmission]]：抽象基类

### 被引用

- [[core/protocol/socks5/udp|socks5 UDP]]：UDP ASSOCIATE 创建 unreliable 传输
