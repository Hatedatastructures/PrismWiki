---
layer: core
source: include/prism/stealth/anytls/mux/frame.hpp
title: AnyTLS Frame Protocol
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, anytls, frame, protocol, mux]
---
# AnyTLS Frame Protocol

## 设计决策（WHY）

### 为什么帧头是 7 字节而非更短

帧头 = cmd(1B) + stream_id(4B) + length(2B) = 7 字节。stream_id 4 字节支持约 40 亿个并发流，length 2 字节支持最大 65535 字节载荷。这些字段大小是权衡：更短的头会限制并发数或帧大小，更长的头浪费带宽。

### 为什么有 11 种帧类型

AnyTLS 的多路复用需要多种操作：创建/关闭流、数据传输、窗口控制、keep-alive、认证等。11 种帧类型覆盖了这些操作，类似 HTTP/2 的帧类型设计。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 帧头固定 7 字节 | 协议定义 | 不可变 |
| length 最大 65535 | uint16_t | 超过需分帧 |
| stream_id 由发送方分配 | 协议设计 | 接收方不能重用 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 未知帧类型 | 协议版本不匹配 | 连接终止 |
| length 与实际载荷不符 | 数据截断 | 解析错误 |
| stream_id 冲突 | 实现错误 | 流混淆 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| frame -> session | 被使用 | session 解析/构造帧 |
| frame -> transport | 被使用 | transport 读写帧数据 |
