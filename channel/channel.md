---
title: Channel 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [channel, transport, connection-pool, happy-eyeballs, overview]
related: [[multiplex], [stealth/overview], [memory/overview], [fault/overview]]
---

# Channel 模块

## 模块定位

Channel 是 Prism 的传输通道层，位于分层架构的最底层。
向上为 [[multiplex]] 多路复用和 [[stealth]] 伪装方案
提供统一的传输抽象，向下对接 Boost.Asio 的 TCP/UDP socket。

Channel 承担四项核心职责：传输抽象（统一的分层流式接口）、
连接管理（连接池复用与 Happy Eyeballs 竞速）、健康检测（socket 有效性验证）、
协议适配（connector 桥接 Boost.Asio 概念）。

## 核心功能

### 传输抽象

`transmission` 是所有传输层的纯虚接口，定义 `async_read_some`、
`async_write_some`、`close`、`cancel` 等基本操作。
所有异步操作返回 `net::awaitable<T>`，错误通过 `std::error_code&` 参数返回。
基类提供默认的 `async_write`（循环写入）和 `async_write_scatter`（逐段写入）实现，
子类可重写以利用原生 scatter-gather I/O。

具体实现：
- `reliable`：TCP 可靠流式传输，支持连接池复用，`async_write_scatter`
  对 2 缓冲区使用 `net::async_write` 原生写入
- `encrypted`：TLS 加密传输，封装 `ssl::stream<connector>`，
  `async_write_scatter` 合并为单次 `SSL_write` 减少 TLS 记录数
- `unreliable`：UDP 数据报传输，维护 `remote_endpoint` 模拟连接语义，
  接收时自动过滤非目标端点数据报
- `snapshot`：可回滚装饰器，两阶段读取（Phase 1 回放 captured_，Phase 2 从内层读取），
  用于 [[stealth]] 方案依次尝试

### 连接池

`connection_pool` 管理到目标服务器的 TCP 连接复用。
内部以 `endpoint_key`（port/family/address）为键、LIFO 栈为值的哈希表缓存连接。
`pooled_connection` 是 RAII 包装器，析构时自动调用 `recycle()` 归还连接，
`release()` 可放弃所有权不归还池。

`async_acquire` 流程：查找缓存 → 过期检查 → 健康检测 → 未命中时通过
co_spawn + timer 创建新连接。支持后台定时清理协程。

设计为线程局部使用，不跨线程共享。

### Happy Eyeballs 竞速

`address_racer` 实现 RFC 8305 地址竞速算法。当 DNS 返回
多个 IP 地址时并发尝试连接，第一个 IP 立即尝试，后续 IP 延迟 250ms。
共享状态 `race_context` 在单线程 strand 内无挂起点，不需要互斥锁。

### 健康检测

`health` 模块提供 socket 有效性验证，在连接池复用前执行。
检测维度包括 SO_ERROR、available 和 MSG_PEEK FIN 检测。
返回 `socket_state` 枚举（healthy/has_data/fin/error/invalid）。
`healthy_fast` 提供快速布尔版本，用于连接池归还前的轻量检查。

检测流程依次为：SO_ERROR 获取待处理错误码 → available 检查待读数据量 →
MSG_PEEK 非阻塞预读检测对端 FIN。任一步骤判定不可用即短路返回。

### 协议适配

`connector` 将 `transmission` 适配为 Boost.Asio 的
AsyncReadStream/AsyncWriteStream 概念。支持预读数据注入：
首次 `async_read_some` 优先从 `preread_buffer_` 返回，
避免协议检测阶段已读数据丢失。用于 `ssl::stream<connector>` 场景。

## 设计原则

- **协程优先**：所有异步操作返回 `net::awaitable<T>`，禁止阻塞
- **错误码返回**：热路径使用 `fault::code`，不抛异常
- **分层装饰**：`transmission` 接口支持任意层叠装饰（reliable → encrypted → snapshot）
- **PMR 内存**：热路径容器使用 `memory::vector`/`memory::unordered_map`
- **工厂模式**：`make_reliable`/`make_encrypted`/`make_unreliable`/`make_snapshot` 统一创建接口

## 核心类与实现细节

### transmission（纯虚基类）

定义在 `channel/transport/transmission.hpp`，命名空间 `psm::channel::transport`。
所有传输层的统一接口，`shared_transmission = shared_ptr<transmission>`。

核心方法：`is_reliable()`（默认 false，TCP/TLS 重写）、`executor()`（纯虚）、
`async_read_some` / `async_write_some`（纯虚）、`shutdown_write()`（默认空）、
`close()` / `cancel()`（纯虚）、`async_write`（循环写入）、`async_read`（循环读取）、
`async_write_scatter`（默认逐段，子类可重写为原生 scatter-gather）。

同文件提供 Asio 适配函数 `async_read_some`/`async_write_some`，
通过 `async_initiate` 桥接，支持任意 CompletionToken，内部做 `fault` → `boost::system` 错误码映射。

### reliable（TCP 实现）

`channel/transport/reliable.hpp`，header-only。继承 `transmission` + `enable_shared_from_this`。
命名空间 `psm::channel::transport`。

三种构造方式：
- `reliable(executor)` — 空 socket，后续 open/accept
- `reliable(socket_type)` — 已连接 socket
- `reliable(pooled_connection)` — 连接池复用

关键行为：
- `close()` 池连接时 cancel 挂起操作但不关闭 socket，析构时通过 `pooled_.reset()` → `recycle()` 归还；非池连接直接 close
- `async_write_scatter` 对 2 缓冲区使用 `net::async_write` 原生写入（`[[likely]]` 优化路径）

工厂函数：`make_reliable(executor)` / `make_reliable(socket)` / `make_reliable(pooled)`

### encrypted（TLS 实现）

`channel/transport/encrypted.hpp`，header-only。继承 `transmission`。
持有 `shared_ptr<ssl::stream<connector>>`（`shared_stream` 类型别名）。

关键行为：
- `close()` 调用 `SSL_shutdown` 发送 close_notify，再关闭底层传输
- `async_write_scatter` 合并为单次 `SSL_write`，减少 TLS 记录数
- `release()` 移动返回内部 TLS 流共享指针
- `stream()` 返回 TLS 流引用

工厂函数：`make_encrypted(shared_stream)`

### unreliable（UDP 实现）

`channel/transport/unreliable.hpp`，header-only。继承 `transmission` + `enable_shared_from_this`。

关键行为：
- `async_read_some`：`async_receive_from` + 来源过滤，不匹配 `remote_endpoint_` 则丢弃并继续等待
- 未设置 `remote_endpoint_` 时首次接收自动设置
- `async_write` 直接委托 `async_write_some`（UDP 数据报一次完成）

工厂函数：`make_unreliable(executor)` / `make_unreliable(socket)`

### snapshot（可回滚装饰器）

`channel/transport/snapshot.hpp`，header-only。继承 `transmission`。
用于 [[stealth]] 方案依次尝试。

两阶段读取：Phase 1 从 `captured_` 回放（`read_pos_ < captured_.size()`，
同步 memcpy），Phase 2 从内层读取并追加到 `captured_`。
写入时标记 `wrote_=true` 禁止 rewind。

- `rewind()`：`read_pos_` 归零
- `can_rewind()`：仅在未写入时返回 true
- `inner()`：返回内层传输

工厂函数：`make_snapshot(inner, mr)`

### connector（Boost.Asio 适配器）

`channel/adapter/connector.hpp`，header-only。命名空间 `psm::channel`。
将 `transmission` 适配为 Boost.Asio AsyncReadStream/AsyncWriteStream 概念。

- `async_read_some`：优先从 `preread_buffer_` 返回（同步 memcpy），消费完毕后委托给传输层
- `async_write_some`：直接委托传输层的 Asio 适配函数
- `async_write`/`async_read`：委托给 `transmission` 虚函数
- `lowest_layer()` 返回自身引用，满足 Asio lowest_layer 概念
- `transmission()` 返回底层传输引用，`release()` 移动返回传输指针

用于 `ssl::stream<connector>` 场景，使 TLS 流建立在 transmission 之上。

### connection_pool（连接池）

`channel/connection/pool.hpp` + `.cpp`。命名空间 `psm::channel`。

核心接口：
- `async_acquire(endpoint)` → `awaitable<pair<fault::code, pooled_connection>>`
- `recycle(socket*, endpoint)`：IPv6 过滤 + 健康检测 + 容量检查后入栈或关闭
- `start()`：启动后台清理协程
- `stats()` → `pool_stats`（idle_count/endpoint_count 实时计算，其余累计计数器）
- `get_config()` → `const config&`

缓存结构：`unordered_map<endpoint_key, vector<idle_item>, endpoint_hash>`，LIFO 栈式。
`endpoint_key` 包含 port/family/address，FNV-1a 哈希。`idle_item` 含 socket 指针和时间戳。

config 字段：max_cache_per_endpoint(32)、connect_timeout_ms(300)、max_idle_seconds(30)、cleanup_interval_sec(10)、recv/send_buffer_size(65536)、tcp_nodelay(true)、keep_alive(true)、cache_ipv6(false)。

### pooled_connection（RAII 包装器）

析构时调用 `reset()` → `pool_->recycle()`。`release()` 释放所有权不归还池。
移动语义转移所有权后源对象置空。内联存储 pool 指针、socket 指针、endpoint。

### address_racer（Happy Eyeballs）

`channel/eyeball/racer.hpp` + `.cpp`。命名空间 `psm::channel::eyeball`。

`race(span<endpoint>)` → `awaitable<pooled_connection>`：
单端点直接连接，多端点第一个立即启动，后续按 250ms 间隔依次启动。
`race_context` 共享状态（winner + 已完成标志），单线程无挂起点不需要互斥锁。
子协程直接捕获连接池引用（非 this），因 racer 可能是栈变量。

### 健康检测实现

`channel/health.hpp` + `.cpp`。命名空间 `psm::channel`。

`socket_state` 枚举：healthy/has_data/fin/error/invalid。
`health(s)` 完整检测（SO_ERROR → available → MSG_PEEK）；`healthy_fast(s)` 快速布尔版本。

## 文件清单

本模块所有源码头文件位于 `include/prism/channel/` 和 `src/prism/channel/`。

| 文件 | 说明 |
|------|------|
| `channel/transport/transmission.hpp` | 传输层纯虚接口 + Asio 适配函数 |
| `channel/transport/reliable.hpp` | TCP 实现（header-only） |
| `channel/transport/encrypted.hpp` | TLS 实现（header-only） |
| `channel/transport/unreliable.hpp` | UDP 实现（header-only） |
| `channel/transport/snapshot.hpp` | 可回滚装饰器（header-only） |
| `channel/connection/pool.hpp` | 连接池接口 |
| `channel/connection/pool.cpp` | 连接池实现 |
| `channel/eyeball/racer.hpp` | Happy Eyeballs 竞速接口 |
| `channel/eyeball/racer.cpp` | 竞速实现 |
| `channel/adapter/connector.hpp` | Boost.Asio 适配器（header-only） |
| `channel/health.hpp` | Socket 健康检测接口 |
| `channel/health.cpp` | 健康检测实现 |
| `channel.hpp` | 聚合头文件 |

## 与其他模块的关系

- **上游**：[[multiplex]] 的 `duct`/`parcel` 持有 `shared_transmission`
- **上游**：[[stealth]] 的方案通过 `snapshot` 包装传输层尝试多次握手
- **上游**：[[resolve]] 的 `router` 持有 `connection_pool` 引用并通过 `address_racer` 竞速连接
- **下游**：依赖 `fault` 模块的错误码和 `memory` 模块的 PMR 容器
