---
layer: core
source: include/prism/stealth/anytls/mux/session.hpp
title: AnyTLS Session
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, anytls, session, mux, concurrent-channel]
---
# AnyTLS Session

## 设计决策（WHY）

### 为什么使用 concurrent_channel 而非 mutex

AnyTLS session 需要在 recv_loop 和用户读写操作之间共享状态。concurrent_channel 是 Boost.Asio 的无锁并发通道，协程友好，不会阻塞 io_context 线程。使用 mutex 会违反 Prism 的协程纯度要求。

### 为什么 recv_loop 是独立协程

recv_loop 持续从传输层读取帧并分发到对应的 stream。作为独立协程（co_spawn），它与用户的读写操作并发执行。recv_loop 通过 steady_timer 实现 keep-alive 超时检测。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| session 生命周期由 shared_ptr 管理 | 多方引用 | transport 和 stream 都持有 session |
| recv_loop 异常终止会导致所有流失败 | 单点 | session 级别的故障 |
| steady_timer 用于超时检测 | keep-alive | 超时关闭连接 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| recv_loop 崩溃 | 传输层错误 | 所有流断开 |
| keep-alive 超时 | 长时间无数据 | 连接关闭 |
| stream_id 耗尽 | 40 亿流后溢出 | 理论上不可能 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| session -> frame | 调用 | 解析/构造帧 |
| session -> transport | 依赖 | 读写帧数据 |
| session -> stream_transport | 创建 | 为每个流创建传输层 |
