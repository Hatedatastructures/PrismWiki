---
layer: core
source: I:/code/Prism/include/prism/multiplex/core.hpp
title: multiplex::core — 多路复用核心抽象基类
module: multiplex
tags: [multiplex, core, abstract, stream-lifecycle]
updated: 2026-05-27
---

# multiplex::core — 多路复用核心抽象基类

多路复用协议的抽象基类，提供会话生命周期管理、流状态跟踪和发送串行化。协议特定的帧编解码由子类实现（smux::craft、yamux::craft、h2mux::craft）。

## 流状态管理

| 状态 | 类型 | 说明 |
|------|------|------|
| pending | `pending_entry` | SYN 后累积地址数据，地址完整后连接目标 |
| duct | [[core/multiplex/duct\|duct]] | TCP 流双向转发 |
| parcel | [[core/multiplex/parcel\|parcel]] | UDP 数据报中继 |

流状态转换：SYN → pending_entry（累积地址）→ activate_stream() → duct/parcel → FIN → 关闭

## 核心接口

### 公开

| 方法 | 说明 |
|------|------|
| `core(core_options)` | 构造，聚合传输层/路由器/配置/PMR |
| `start()` | co_spawn 启动 run() 协程 |
| `close()` | 幂等关闭，清空所有映射 |
| `is_active()` | 原子读取活跃标志 |
| `set_traffic(t, p)` | 设置流量统计和归属协议 |
| `accumulate_traffic(up, down)` | 原子累加子流流量 |

### 纯虚（子类必须实现）

| 方法 | 说明 |
|------|------|
| `send_data(stream_id, payload)` | 发送数据帧（零拷贝 move） |
| `send_fin(stream_id)` | 发送半关闭帧 |
| `executor()` | 返回执行器 |
| `run()` | 协议主循环（帧读取/解析/分发） |

### 受保护虚函数

| 方法 | 说明 |
|------|------|
| `remove_duct(stream_id)` | duct::close() 回调，子类可 override |
| `remove_parcel(stream_id)` | parcel::close() 回调，子类可 override |

## 设计决策

### 为什么 close() 中使用 std::move(ducts_)？

遍历 `ducts_` 时调用 `duct->close()`，而 duct::close() 会回调 `core::remove_duct()` 修改 ducts_，导致迭代器失效。先 std::move 将映射移到局部变量，ducts_ 变空，后续 remove_duct 对空 map 的 erase 是空操作。

**后果**: close() 只能执行一次有效清理（由 `active_.exchange(false)` 保证幂等）。

**源码依据**: `core.cpp:101`

### 为什么 pending/ducts/parcels 三态分离？

流在不同阶段有不同行为：pending 需累积地址数据，duct 做 TCP 双向转发，parcel 做 UDP 中继。三个独立的 unordered_map 按流 ID 查找，dispatch 时按优先级依次检查（pending → ducts → parcels），每步 O(1)。

**后果**: 查找需检查三个映射，但每步都是哈希查找，开销可忽略。

### 为什么 core 继承 enable_shared_from_this？

`start()` 通过 co_spawn 启动 run() 协程，协程可能在 core 对象析构后恢复。捕获 `shared_from_this()` 到 lambda 中保证协程运行期间 core 存活。

**后果**: core 必须通过 shared_ptr 管理。

### 为什么 set_traffic 是可选的？

bootstrap 创建 core 后调用 set_traffic()。如果跳过此调用，core::close() 不会刷入流量统计（traffic_ 为 nullptr）。这不是错误——某些场景（如内部测试）不需要流量统计。

**后果**: 忘记调用 set_traffic 不会崩溃，但子流流量不计入统计。

**源码依据**: `core.cpp:90-95`

## 约束

### 流 ID 唯一性

**类型**: 状态前置

**规则**: 客户端分配的流 ID 在同一 mux 会话内必须唯一。

**违反后果**: 重复 SYN 导致 emplace 失败（yamux 会 trace::warn）或覆盖（smux），数据路由到错误管道。

### pending_entry 地址累积阈值

**类型**: 资源上限

**规则**: buffer >= 7 字节时尝试解析地址；>= 21 字节仍无法解析时判定地址格式错误。

**违反后果**: 无法连接目标，发送错误状态后关闭流。

### 单线程使用

**类型**: 线程安全

**规则**: 单个 core 实例在 transport executor 上串行使用。accumulate_traffic 和 active_ 是原子的，但 pending_/ducts_/parcels_ 非线程安全。

**违反后果**: 数据竞争。

## 故障场景

### 所有活跃流同时失效

**触发条件**: 底层传输连接断开
**传播路径**: transport->cancel() → 各协程 operation_canceled → close() 清理所有映射
**恢复机制**: 客户端重新建立传输连接

### pending 流永不激活

**触发条件**: 恶意客户端发送 SYN 后不发送后续数据
**传播路径**: pending_entry 永远留在 pending_ 中
**外部表现**: 占用流配额，max_streams 限制下可能耗尽
**恢复机制**: max_streams 兜底限制（默认 32）

### 流量统计丢失

**触发条件**: 未调用 set_traffic()
**传播路径**: close() 中 traffic_ 为空则跳过 flush
**外部表现**: 子流流量不计入统计
**源码依据**: `core.cpp:90-95`

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/channel/transport/transmission\|transmission]] | 底层传输 |
| [[core/connect/dial/router\|router]] | 地址解析和目标连接 |
| [[core/stats/traffic\|traffic_state]] | 流量统计 |

### 子类

- [[core/multiplex/smux/craft|smux::craft]] — smux v1 协议
- [[core/multiplex/yamux/craft|yamux::craft]] — yamux 协议
- [[core/multiplex/h2mux/craft|h2mux::craft]] — h2mux 协议

### 被引用

- [[core/multiplex/bootstrap|bootstrap]] — 创建 core 实例
- [[core/multiplex/duct|duct]] — friend，直接访问 pending_/ducts_
- [[core/multiplex/parcel|parcel]] — friend，直接访问 parcels_

### 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| duct | core | duct::close() 回调 core::remove_duct()，形成重入。core 用 std::move(map) 规避迭代器失效 |
| parcel | core | 同 duct，close() 回调 remove_parcel() |
| bootstrap | core | bootstrap 创建 core 后调用 set_traffic()，否则流量统计丢失 |
