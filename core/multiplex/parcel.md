---
layer: core
source: include/prism/multiplex/parcel.hpp
title: multiplex::parcel — UDP 数据报中继管道
module: multiplex
tags: [multiplex, parcel, udp, datagram, relay]
updated: 2026-05-27
---

# multiplex::parcel — UDP 数据报中继管道

协议无关的 UDP 数据报中继管道。每个 mux 流中的 UDP 流对应一个 parcel 实例。与 [[core/multiplex/duct|duct]] 的面向连接模型不同，parcel 是无连接的请求-响应模型。

## 核心接口

| 方法 | 说明 |
|------|------|
| `parcel(config, owner, router)` | 从配置构造，owner 为 core 的 shared_ptr |
| `start()` | co_spawn 启动空闲超时监控 |
| `on_data(data)` | 接收 mux 数据报并转发到目标 |
| `close()` | 幂等关闭，从 core 的 parcels_ 移除 |
| `stream_id()` | 流标识符 |
| `set_destination(host, port)` | 设置 length_prefixed 模式的固定目标 |

### parcel_config

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `stream_id` | 0 | SYN 帧分配的流 ID |
| `idle_timeout` | 30000 | 空闲超时（毫秒） |
| `max_dgram` | 4096 | 最大数据报长度（字节） |
| `mode` | length_prefixed | 地址编码模式 |

### addr_mode

| 模式 | 帧格式 | 说明 |
|------|--------|------|
| `length_prefixed` | `[Length 2B BE][Payload]` | 目标地址在 SYN 时确定 |
| `packet_addr` | `[ATYP 1B][Addr(var)][Port 2B][Data]` | 每帧携带 SOCKS5 地址 |

## 设计决策

### 为什么 mux_buffer_ 需要累积缓冲？

sing-mux 客户端可能将一个 UDP 数据报拆成多个 PSH 帧发送（地址、Length、Payload 各一帧）。`on_data` 每次收到一帧，不能假设一帧就是一个完整数据报。`mux_buffer_` 累积数据后循环解析完整数据报。`processing_` 原子标志防止并发处理。

**后果**: 增加缓冲区管理复杂度，但保证数据报完整性。

### 为什么 egress_socket_ 延迟创建？

UDP socket 需要按目标地址的协议族（IPv4/IPv6）创建。SYN 阶段只知道主机名，不知道解析后的协议族。首次 `do_send` 时通过 `ensure_socket()` 按解析结果创建。目标协议族变化时自动重建 socket。

**后果**: 首次发送有额外 socket 创建开销。

### 为什么 owner_ 用 weak_ptr 而非 shared_ptr？

core 的 `parcels_` 持有 parcel 的 shared_ptr。如果 parcel 再持有 core 的 shared_ptr，形成循环引用，导致两者永不释放。weak_ptr 打破循环：core 析构后 `lock()` 返回空，parcel 安全退出。

**后果**: 每次 `send_data` 前需要 `owner_.lock()` 检查 core 是否存活。

### 为什么 downlink_loop 在首次发送时才启动？

UDP 响应需要目标先发数据后才产生。首次 `do_send` 时启动 `downlink_loop` 异步读取响应。如果目标不响应，不会有下行数据，不需要启动接收循环。

**后果**: downlink_loop 的生命周期与 egress_socket_ 绑定。

## 约束

### 单处理循环保证

**类型**: 线程安全

**规则**: `processing_` 原子标志保证同一时刻只有一个 `process_buffer` 协程运行。

**违反后果**: 同一 socket 上并发 `async_send_to` 可能交错。

**源码依据**: `parcel.cpp:153` `if (!processing_.exchange(true))`

### UDP 数据报最大长度

**类型**: 资源上限

**规则**: `mux_buffer_.size() > max_dgram_` 时调用 `close()` 关闭管道。

**违反后果**: 管道被关闭，后续数据报丢失。

### 不可路由地址过滤

**类型**: 状态前置

**规则**: 目标地址以 0 开头的 IPv4 地址（如 sing-mux 占位地址 0.0.0.1）被跳过。

**违反后果**: 尝试连接不可路由地址导致 DNS 解析失败。

**源码依据**: `parcel.cpp:263-265`

## 故障场景

| 场景 | 触发条件 | 处理 | 恢复 |
|------|----------|------|------|
| Socket 创建失败 | fd 耗尽/权限不足 | `ensure_socket` 返回 false，静默丢弃 | 下次 do_send 重试 |
| 空闲超时 | 无数据活动超过 idle_timeout | uplink_loop 超时 → close() | 客户端新建流 |
| DNS 解析失败 | 主机名无法解析 | 静默丢弃数据报 | 客户端重试 |

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/multiplex/core\|core]] | owner_ 弱引用，send_data 发帧 |
| [[core/connect/dial/router\|router]] | DNS 解析目标主机名 |

### 被引用

| 模块 | 使用方式 |
|------|----------|
| [[core/multiplex/core\|core]] | parcels_ 映射管理 parcel 生命周期 |
| smux::craft / yamux::craft | dispatch_push → co_spawn on_data |
