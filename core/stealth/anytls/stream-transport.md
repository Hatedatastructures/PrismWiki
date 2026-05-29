---
layer: core
source: include/prism/stealth/anytls/mux/transport.hpp
title: AnyTLS Stream Transport
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, anytls, transport, stream, transmission]
---
# AnyTLS Stream Transport

## 设计决策（WHY）

### 为什么 stream_transport 持有 shared_ptr<session>

stream_transport 是单个流的双向传输层，但实际的帧读写由 session 的 recv_loop 完成。shared_ptr 确保只要还有活跃的流，session 就不会被销毁。

### 为什么 async_write 需要封装为帧

上层协议（如 Shadowsocks）调用 async_write 时只提供明文数据。stream_transport 需要将数据封装为 AnyTLS 帧（添加 cmd + stream_id + length 头部），然后交给 session 发送。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 每个 stream 对应一个 stream_transport | 一对一 | 不可共享 |
| 帧封装开销 7 字节/帧 | 帧头 | 带宽开销 |
| 数据通过 session 的 channel 传递 | 并发安全 | 不直接写 socket |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| session 已关闭 | 上层 session 析构 | 写入失败 |
| stream_id 无效 | session 不认识此流 | 帧被丢弃 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| stream_transport -> session | 依赖 | 通过 session 发送帧 |
| stream_transport -> channel::transport::transmission | 继承 | 实现标准传输接口 |
| protocol handlers -> stream_transport | 使用 | 协议处理器读写数据 |
