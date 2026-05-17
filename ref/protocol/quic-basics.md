---
title: "QUIC 基础"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 9000, RFC 9001, RFC 9002"
tags: [协议, quic, udp, tls, 多路复用, 拥塞控制]
created: 2026-05-17
updated: 2026-05-17
---

# QUIC 基础

**类别**: 协议

## 概述

QUIC（Quick UDP Internet Connections）是 Google 设计的新传输协议，基于 UDP 实现可靠传输，内置 TLS 1.3 加密。QUIC 解决了 TCP 的队头阻塞问题，支持连接迁移，提供更快的安全连接建立。

### 核心特性

QUIC 的核心特性：

| 特性 | 说明 |
|------|------|
| **基于 UDP** | 用户态实现，无内核限制 |
| **内置 TLS 1.3** | 加密握手集成 |
| **1-RTT/0-RTT** | 快速连接建立 |
| **多路复用** | 无队头阻塞 |
| **连接迁移** | Connection ID 标识连接 |
| **拥塞控制** | 用户态实现，可定制 |

### 与 TCP 的对比

QUIC 与 TCP+TLS 的对比：

```
QUIC vs TCP+TLS：

| 特性 | QUIC | TCP+TLS |
|------|------|---------|
| 传输层 | UDP | TCP |
| 握手 | 1-RTT (0-RTT) | 1-RTT TCP + 2-RTT TLS = 3-RTT |
| 加密 | 内置 TLS 1.3 | TLS 需额外握手 |
| 多路复用 | 无队头阻塞 | HTTP/2 有队头阻塞 |
| 连接迁移 | 支持 | 不支持 |
| 拥塞控制 | 用户态可定制 | 内核固定 |
| 部署 | 需要 UDP 支持 | 广泛支持 |

TCP+TLS 1.3 实际：1-RTT（合并握手）
QUIC 优势：多路复用无队头阻塞、连接迁移
```

## QUIC 连接

### 连接标识

QUIC 使用 Connection ID 标识连接：

```
Connection ID：

作用：
- 不依赖 IP/端口标识连接
- 支持 IP/端口变化后保持连接
- 移动网络切换无缝

格式：
- 长度可变（0-20 字节）
- 由接收方生成
- 可在握手过程中变化

Initial Connection ID：
- 客户端生成的初始 ID
- 包含在 Initial 包中
- 用于服务端响应

服务端 Connection ID：
- 服务端生成的 ID
- 包含在 Transport Parameters 中
- 用于后续通信
```

### 连接建立

QUIC 连接建立流程：

```
QUIC 1-RTT 握手：

客户端                                    服务端
  |                                          |
  |---------- Initial ---------------------->| (CRYPTO 帧: ClientHello)
  |          (包含 key_share, CID)           |
  |                                          |
  |<--------- Initial -----------------------| (CRYPTO 帧: ServerHello)
  |          (包含服务端 CID)                 |
  |<--------- Handshake ---------------------| (CRYPTO 帧: EncryptedExtensions,
  |          (加密握手消息)                    |  Certificate, Finished)
  |                                          | ← 握手密钥已建立
  |---------- Handshake --------------------->| (CRYPTO 帧: Finished)
  |          (加密握手完成)                    |
  |                                          | ← 应用密钥已建立
  |<========= 1-RTT Data =====================>| (加密应用数据)

特点：
- Initial 包未加密（仅 ClientHello）
- Handshake 包使用握手密钥加密
- 1-RTT 包使用应用密钥加密
- 类似 TLS 1.3 但集成在 QUIC 中
```

### 0-RTT 连接

QUIC 支持 0-RTT 数据：

```
QUIC 0-RTT：

前提：
- 之前有会话恢复信息
- 服务端接受 0-RTT

流程：
客户端                                    服务端
  |                                          |
  |---------- Initial ---------------------->| (CRYPTO 帧: ClientHello + PSK)
  |---------- 0-RTT ------------------------>| (加密早期数据)
  |                                          |
  |<--------- Initial -----------------------| (CRYPTO 帧: ServerHello)
  |<--------- Handshake ---------------------| (加密握手消息)
  |                                          |
  |---------- Handshake --------------------->| (Finished)
  |---------- 1-RTT ------------------------>| (后续数据)

安全限制：
- 无前向保密（使用 PSK）
- 可重放攻击
- 仅发送幂等数据
```

## QUIC 包格式

### 包类型

QUIC 包类型：

```
QUIC 包类型：

| 类型 | 长度 | 说明 | 加密 |
|------|------|------|------|
| Initial | Long | 初始握手 | 未加密或初始密钥 |
| 0-RTT | Long | 早期数据 | 0-RTT 密钥 |
| Handshake | Long | 握手消息 | 握手密钥 |
| Retry | Long | 重试请求 | 未加密 |
| Version Negotiation | Long | 版本协商 | 未加密 |
| 1-RTT | Short | 应用数据 | 应用密钥 |

Long Header 格式：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1| Form | Type(7)|       Reserved        |    Version          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Destination Connection ID Length (8)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Destination Connection ID (0..160)        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Connection ID Length (8)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Source Connection ID (0..160)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Length (i)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Packet Number (i)         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Packet Payload ...        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Short Header 格式：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|1|S|K| Key Phase | Packet Number (8/16/24/32)              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Destination Connection ID (0..160)        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Packet Payload ...        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 包编号

QUIC 包编号机制：

```
Packet Number：

特点：
- 每个包有唯一编号
- 严格递增
- 独立于加密级别

编码：
- 可变长度（1-4 字节）
- 根据范围动态选择
- 编码足够表示预期最大编号

用途：
- ACK 确认
- 丢包检测
- 重传标识

与 TCP 序列号区别：
- TCP 序列号是字节序号
- QUIC 包编号是包序号
- QUIC 重传使用新编号
```

## QUIC 帧

### 帧类型

QUIC 使用帧承载数据：

```
QUIC 帧类型：

| 类型 | 名称 | 说明 |
|------|------|------|
| 0x00 | PAD | 填充 |
| 0x01 | PING | 保活探测 |
| 0x02-0x03 | ACK | 确认 |
| 0x04 | RESET_STREAM | 重置流 |
| 0x05 | STOP_SENDING | 停止发送 |
| 0x06 | CRYPTO | 加密握手数据 |
| 0x07 | NEW_TOKEN | 新令牌 |
| 0x08-0x0f | STREAM | 流数据 |
| 0x10 | MAX_DATA | 最大数据 |
| 0x11 | MAX_STREAM_DATA | 最大流数据 |
| 0x12-0x13 | MAX_STREAMS | 最大流数 |
| 0x14 | DATA_BLOCKED | 数据阻塞 |
| 0x15-0x17 | STREAM_DATA_BLOCKED | 流数据阻塞 |
| 0x18-0x19 | STREAMS_BLOCKED | 流数阻塞 |
| 0x1a | NEW_CONNECTION_ID | 新连接 ID |
| 0x1b | RETIRE_CONNECTION_ID | 退役连接 ID |
| 0x1c | PATH_CHALLENGE | 路径挑战 |
| 0x1d | PATH_RESPONSE | 路径响应 |
| 0x1e | CONNECTION_CLOSE | 连接关闭 |
| 0x1f | CONNECTION_CLOSE (app) | 应用关闭 |
| 0x30 | HANDSHAKE_DONE | 握手完成 |
```

### STREAM 帧

STREAM 帧承载流数据：

```
STREAM 帧格式：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|1|F|S| Len(2)|        Stream ID (i)        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
[ Offset (i) ... ]
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
[ Length (i) ... ]
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Stream Data ...           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段：
- F (Fin): 流结束标志
- S (Offset): 是否有偏移
- Len: 长度字段
- Stream ID: 流标识
- Offset: 数据偏移
- Length: 数据长度
- Data: 流数据

流特性：
- 每个 STREAM 帧可能包含部分数据
- Offset 标识数据位置
- Fin 标识流结束
```

### CRYPTO 帧

CRYPTO 帧承载握手数据：

```
CRYPTO 帧：

格式：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Offset (i) ...            |   Length (i)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Crypto Data ...          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

用途：
- Initial 包：ClientHello, ServerHello
- Handshake 包：EncryptedExtensions, Certificate, Finished
- 1-RTT 包：后续握手消息（如 NEW_TOKEN）

特点：
- 类似 STREAM 帧，但用于握手
- 没有 Stream ID（只有一个加密流）
- 保证可靠传递
```

## 多路复用

### 流机制

QUIC 的流机制：

```
QUIC 流：

流类型：
| 首位 | 类型 | 说明 |
|------|------|------|
| 0 | 客户端发起双向流 | 客户端创建，双方发送 |
| 1 | 服务端发起双向流 | 服务端创建，双方发送 |
| 2 | 客户端发起单向流 | 客户端发送 |
| 3 | 服务端发起单向流 | 服务端发送 |

流 ID 编号：
- 双向流：0, 1, 4, 5, 8, 9...（+4）
- 单向流：2, 3, 6, 7, 10, 11...（+4）
- 客户端流：偶数
- 服务端流：奇数

无队头阻塞：
- 每个流独立序列
- 一个流的丢包不影响其他流
- HTTP/3 可以并行传输多个请求
```

### 流控制

QUIC 流量控制：

```
流控制：

层级：
- 连接级：总数据量限制
- 流级：单流数据量限制

机制：
- MAX_DATA: 连接级最大数据偏移
- MAX_STREAM_DATA: 流级最大数据偏移
- DATA_BLOCKED: 连接级阻塞通知
- STREAM_DATA_BLOCKED: 流级阻塞通知

工作方式：
1. 发送方发送数据
2. 接收方更新 MAX_DATA/MAX_STREAM_DATA
3. 发送方根据限制调整发送
4. 阻塞时发送 BLOCKED 帧

自动调整：
- 类似 TCP 滑动窗口
- 动态调整限制
- 基于接收缓冲区
```

## 拥塞控制

### QUIC 拥塞控制

QUIC 实现用户态拥塞控制：

```
拥塞控制特点：

用户态实现：
- 不依赖内核 TCP 拥塞控制
- 可定制算法
- 更快迭代

支持算法：
- Cubic: 默认，类似 TCP Cubic
- BBR: 带宽估计，适合高延迟
- Reno: 经典 TCP Reno

检测信号：
- ACK: 数据成功传递
- 丢包: 通过 ACK 缺失检测
- ECN: 显式拥塞通知（如果支持）

与 TCP 的区别：
- 更准确的 RTT 测量（包编号）
- 更快的丢包检测
- 可定制算法
```

## 连接迁移

### IP/端口变化

QUIC 支持连接迁移：

```
连接迁移场景：

1. 移动网络切换
   - WiFi → 4G → WiFi
   - Connection ID 保持不变
   - 连接无缝继续

2. 地址重绑定
   - DHCP 重新分配 IP
   - NAT 重新映射
   - 连接保持

实现机制：
1. Connection ID 标识连接
   - 不依赖 IP/端口
   
2. PATH_CHALLENGE/PATH_RESPONSE
   - 验证新路径可达
   - 防止攻击
   
3. NEW_CONNECTION_ID
   - 服务端提供新 CID
   - 防止链接跟踪

流程：
客户端 (IP1)                          服务端
  |                                    |
  |------- QUIC 数据 ----------------->|
  |                                    |
  [客户端 IP 变化为 IP2]
  |                                    |
  |------- PATH_CHALLENGE ------------>| (新 IP)
  |                                    |
  |<------ PATH_RESPONSE --------------|
  |                                    |
  |------- QUIC 数据 (继续) ----------->|
```

## HTTP/3

### HTTP over QUIC

HTTP/3 是 HTTP over QUIC：

```
HTTP/3 特性：

基于 QUIC：
- 使用 QUIC 流承载 HTTP 请求/响应
- 无队头阻塞
- 快速握手

请求映射：
- 每个请求使用一个双向流
- 流 ID = 0, 4, 8, 12...
- 请求头使用 QPACK 编码

帧类型：
| 类型 | 名称 |
|------|------|
| 0x00 | DATA |
| 0x01 | HEADERS |
| 0x03 | CANCEL_PUSH |
| 0x04 | SETTINGS |
| 0x05 | PUSH_PROMISE |
| 0x07 | GOAWAY |
| 0x08 | MAX_PUSH_ID |
| 0x09 | DUP_PUSH_ID |

QPACK：
- 类似 HTTP/2 的 HPACK
- 静态表 + 动态表
- 动态表更新使用单独流
```

## 在 Prism 中的应用

### QUIC 支持

Prism 对 QUIC 的潜在支持：

```
Prism QUIC 支持：

入站：
- HTTP/3 入站（未来）
- QUIC 直接入站

出站：
- HTTP/3 出站
- QUIC 出站

优势：
- 快速连接建立
- 无队头阻塞
- 连接迁移支持

挑战：
- UDP 防火墙限制
- NAT 行为差异
- 部署成熟度
```

## 参见

- [[ref/protocol/tcp-basics|TCP 基础]] — TCP 协议对比
- [[ref/protocol/udp-basics|UDP 基础]] — UDP 协议基础
- [[ref/crypto/tls-crypto|TLS 加密]] — TLS 1.3 加密
- [[ref/protocol/tls-handshake|TLS 握手]] — TLS 握手流程
- [[core/multiplex/overview|multiplex]] — 多路复用协议