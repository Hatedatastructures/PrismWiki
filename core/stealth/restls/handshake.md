---
layer: core
source: include/prism/stealth/restls/handshake.hpp
title: Restls Handshake
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, restls, handshake, blake3, backend-relay]
---
# Restls Handshake

> 源码位置: include/prism/stealth/restls/handshake.hpp + src/prism/stealth/restls/handshake.cpp

## 设计决策（WHY）

### 为什么 Restls 使用 Path C proxy 模式

Restls 的握手不解析或修改 ClientHello 内容。它直接将客户端的 ClientHello 转发给后端 TLS 服务器，然后在后端的 ServerHello 中提取 server_random。这种设计比 Reality 简单得多，代价是无法在握手阶段进行认证。

### 为什么 Step 4 设置 polluted=true

Step 4 将后端的 ServerHello 转发给客户端。此时已经向客户端 socket 写入了数据，如果后续认证失败，rewind 无法撤回已发送的 TCP 数据。因此 polluted=true 告知执行器不可 rewind。

### 为什么 relay 使用 co_spawn

后端到客户端的 relay 需要与客户端认证读取并发执行。relay 持续转发后端数据给客户端，同时主协程处理客户端发送的认证信息。两者通过 atomic bool 的 cancellation_signal 协调终止。

### 为什么 TLS 1.2 和 TLS 1.3 的 XOR 偏移不同

TLS 1.2 使用 legacy explicit IV（8 字节），TLS 1.3 使用 XNonce（无额外字节）。Restls 的 XOR mask 作用位置取决于 TLS 版本。version_hint 控制这个偏移。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 握手 Step 4 后不可 rewind | polluted=true | 已向客户端发送 ServerHello |
| relay 协程可能比握手活得更长 | 500ms 超时 | socket 可能被关闭后 relay 仍在写入 |
| server_random 提取位置固定 | 偏移 43 | TLS header + handshake header + version + random |
| client_finished 必须从 relay 获取 | relay 转发客户端 Finished | 主协程无法直接读取 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 后端不可达 | host 配置错误 | connection_refused |
| ServerHello 解析失败 | 后端返回非 TLS 响应 | protocol_error |
| client_finished 未收到 | relay 提前终止 | protocol_error |
| relay socket 写入失败 | 客户端断开 | connection_refused + polluted=true |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| handshake -> stealth::common | 调用 | read_raw_tls_frame 读取后端 TLS 帧 |
| handshake -> connect | 调用 | TCP resolver 建立后端连接 |
| handshake -> scheme | 返回 | handshake_detail 传给 restls_handover |
| handshake -> script | 依赖 | restls_script 构造 script_engine |
