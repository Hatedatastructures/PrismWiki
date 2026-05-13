---
title: "UDP 协议基础"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [udp, transport, network]
related:
  - "[[channel]]"
  - "[[protocol/socks5]]"
  - "[[multiplex]]"
  - "[[dev/tcp]]"
  - "[[client/mihomo-dns]]"
  - "[[agent]]"
---

# UDP 协议基础

UDP（User Datagram Protocol）是无连接的传输层协议，提供简单、低开销的数据报传输。在代理领域，UDP 的支持程度是衡量代理工具能力的重要指标。

## UDP 基本特性

| 特性 | UDP | TCP |
|------|-----|-----|
| 连接模式 | 无连接 | 面向连接 |
| 可靠性 | 不保证 | 可靠传输 |
| 顺序性 | 不保证 | 有序 |
| 头部大小 | 8 字节 | 20+ 字节 |
| 流量控制 | 无 | 有 |
| 拥塞控制 | 无 | 有 |

### UDP 数据报格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Data ...                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

头部仅 8 字节，远小于 TCP。

## UDP 的典型应用

- **DNS 查询**：最常见的 UDP 应用（见 [[client/mihomo-dns]]）
- **视频/音频流**：实时性要求高于可靠性
- **在线游戏**：低延迟优先
- **WebRTC**：P2P 音视频通信
- **QUIC**：基于 UDP 的新一代传输协议

## UDP 在代理中的挑战

### NAT 穿透

NAT 设备为 UDP 维护的映射表有超时机制（通常 30-120 秒）：

1. 客户端发送 UDP 包到 NAT 外部
2. NAT 创建映射：内部地址:端口 ↔ 外部地址:端口
3. 映射在空闲超时后删除
4. 代理需要定期发送保活包维持映射

### 有状态防火墙

大多数防火墙只允许"已建立"的 UDP 流量（即有对应的出站请求）。代理服务器返回的 UDP 包必须匹配之前的出站请求，否则会被丢弃。

### 与 TCP 的根本差异

代理 UDP 不能简单地像 TCP 那样建立连接然后双向流式传输：

- UDP 是数据报边界保留的：一次 `sendto` 对应一次 `recvfrom`
- 每个数据报都携带目标地址信息
- 代理需要为每个 UDP 会话维护状态

## SOCKS5 UDP ASSOCIATE

SOCKS5 协议通过 UDP ASSOCIATE 命令支持 UDP 代理（见 [[protocol/socks5]]）：

### 工作流程

```
Client                        SOCKS5 Server
  |                               |
  |--- TCP: UDP ASSOCIATE ------->|  请求 UDP 代理
  |<-- TCP: BND.ADDR:BND.PORT ---|  服务器 UDP 地址
  |                               |
  |=== UDP: [Header][Data] =====>|  UDP 数据通过指定端口
  |<== UDP: [Header][Data] =====|  响应数据
  |                               |
  |--- TCP: DISCONNECT --------->|  结束 UDP 代理
```

### UDP 请求头格式

```
+----+------+------+----------+----------+----------+
|RSV | FRAG | ATYP | DST.ADDR | DST.PORT |   DATA   |
+----+------+------+----------+----------+----------+
| 2  |  1   |  1   | Variable |    2     | Variable |
+----+------+------+----------+----------+----------+
```

- RSV：保留字段
- FRAG：分片编号（0 = 未分片）
- ATYP/ADDR/PORT：目标地址

### 局限

- UDP ASSOCIATE 依赖 TCP 控制连接存在
- 控制连接断开后 UDP 代理终止
- FRAG 分片支持在实践中很少实现

## Prism 中的 UDP 处理

### 数据报中继

Prism 的 UDP 中继模块处理无连接数据报的转发：

```
[App] --UDP--> [Prism Client] --TCP/UDP--> [Prism Server] --UDP--> [Target]
```

核心设计：

1. **会话管理**：为每个 UDP 源（src_addr:src_port）维护会话
2. **超时清理**：空闲会话在超时后释放
3. **地址映射**：记录每个会话的源地址，用于回传响应

### Parcel 模块

在多路复用场景中（见 [[multiplex]]），UDP 数据报通过 Parcel 模块封装：

```
+--------+--------+------------------+
| Type   | Length | Datagram Data    |
| 1 byte | 2 byte | Variable        |
+--------+--------+------------------+
```

Parcel 在 TCP 复用连接上模拟 UDP 语义：

- 保留数据报边界
- 携带目标地址信息
- 支持双向传输

### 性能考虑

UDP 代理的性能瓶颈：

- 每个数据报都需要完整的地址解析
- 小包（如 DNS 查询）的 per-packet overhead 大
- 高频 UDP 应用（如游戏）需要高效的会话管理

Prism 使用以下优化：

- 哈希表存储会话，O(1) 查找
- 批量处理数据报减少系统调用
- 零拷贝路径减少内存分配

## DNS over UDP vs TCP

DNS 传统上使用 UDP，但也有 TCP 模式：

| 特性 | DNS over UDP | DNS over TCP |
|------|-------------|-------------|
| 端口 | 53 | 53 |
| 数据包大小 | 512 字节（传统）/ 4096（EDNS） | 无限制 |
| 可靠性 | 不保证 | 可靠 |
| 延迟 | 低 | 略高（握手） |
| 使用场景 | 常规查询 | 大响应、DNSSEC |

现代 DNS 还支持：
- **DoH**（DNS over HTTPS）：TCP 端口 443
- **DoT**（DNS over TLS）：TCP 端口 853
- **DoQ**（DNS over QUIC）：UDP 端口 853

在代理场景中，DNS 查询通常通过代理发出以防止泄露，详见 [[client/mihomo-dns]]。

## QUIC：下一代 UDP 协议

QUIC 基于 UDP 构建，融合了 TCP 的可靠性与 UDP 的灵活性：

- 0-RTT 连接建立
- 内置 TLS 1.3
- 多路复用无队头阻塞
- 连接迁移（Connection ID）

部分代理协议（如 Hysteria）基于 QUIC 构建，利用 UDP 的特性实现高性能代理。

## 相关页面

- [[channel]] — 通道模块与连接管理
- [[protocol/socks5]] — SOCKS5 协议详解
- [[multiplex]] — 多路复用概览
- [[dev/tcp]] — TCP 协议基础
- [[client/mihomo-dns]] — mihomo DNS 处理
- [[agent]] — Prism Agent 概览
