---
title: "UDP 基础"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 768"
tags: [协议, udp, 传输, 数据报, 无连接]
created: 2026-05-17
updated: 2026-05-17
---

# UDP 基础

**类别**: 协议

## 概述

UDP（User Datagram Protocol）是无连接的数据报协议，由 RFC 768 定义。UDP 提供简单、快速的传输服务，不保证可靠性，但保留消息边界。

### 核心特性

UDP 的核心特性：

| 特性 | 说明 |
|------|------|
| **无连接** | 无需握手，直接发送 |
| **不可靠** | 可能丢失、乱序、重复 |
| **数据报** | 保留消息边界 |
| **快速** | 无连接开销 |
| **轻量** | 8 字节头部 |
| **全双工** | 双向同时传输 |

### 与 TCP 的对比

UDP 与 TCP 的核心差异：

```
UDP vs TCP：

| 特性 | UDP | TCP |
|------|-----|-----|
| 连接 | 无 | 三次握手 |
| 可靠性 | 不保证 | 保证 |
| 流量控制 | 无 | 滑动窗口 |
| 拥塞控制 | 无 | 慢启动/避免 |
| 消息边界 | 保留 | 字节流 |
| 头部大小 | 8 字节 | 20+ 字节 |
| 适用场景 | DNS、QUIC、实时 | 文件、网页、代理 |

选择原则：
- 需要可靠性：TCP
- 需要速度、消息边界：UDP
- 应用层实现可靠性：UDP + 应用层协议（如 QUIC）
```

## UDP 报文结构

### 头部格式

UDP 头部极其简单：

```
UDP 头部（8 字节）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- Source Port (16 bit): 源端口
- Destination Port (16 bit): 目标端口
- Length (16 bit): UDP 报文总长度（头部 + 数据）
- Checksum (16 bit): 校验和

注意：
- Length 最小为 8（仅头部，无数据）
- Checksum 可选（IPv4 中），IPv6 中强制
- Checksum 包含伪头部
```

### 校验和计算

UDP 校验和的计算方法：

```
校验和计算：

伪头部（不传输，仅用于计算）：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Zero  | Proto |        UDP Length                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

校验和覆盖：
1. 伪头部（源/目的 IP、协议号、UDP 长度）
2. UDP 头部
3. UDP 数据

计算方法：
- 16 位对齐，补零
- 所有 16 位字求和
- 进位加回结果
- 取反得到校验和

IPv4: 校验和可选（可设为 0 表示不计算）
IPv6: 校验和强制（不能为 0）
```

## UDP 数据报特性

### 消息边界

UDP 保留消息边界：

```
消息边界特性：

发送：
send(socket, "message1", 7);
send(socket, "message2", 7);

接收：
recv(socket, buffer1, 100); → 收到 "message1" (7 字节)
recv(socket, buffer2, 100); → 收到 "message2" (7 字节)

对比 TCP 字节流：
TCP 可能合并：收到 "message1message2" (14 字节)
TCP 可能分割：收到 "mes" (3 字节), "sage1mess" (8 字节), "age2" (5 字节)

UDP 应用：
- DNS 查询/响应：一个请求一个数据报
- QUIC：每个 UDP 包是完整或部分 QUIC 包
- 实时音视频：每帧一个或多个数据报
```

### 最大长度

UDP 数据报的最大长度：

```
UDP 数据报长度限制：

理论最大：65507 字节
- IP 最大 65535 字节
- IP 头部 20 字节
- UDP 头部 8 字节
- 65535 - 20 - 8 = 65507

实际限制：
- MTU（Maximum Transmission Unit）
- 以太网 MTU = 1500 字节
- UDP 数据建议 ≤ 1472 字节（1500 - 20 - 8）

超出 MTU：
- IP 分片（Fragmentation）
- 分片可能丢失（任一片丢失整个包丢失）
- 建议避免分片

最佳实践：
- 数据大小 ≤ MTU - 28（IP + UDP 头部）
- 应用层分片比 IP 分片更可控
- Path MTU Discovery 可确定实际 MTU
```

## UDP 在代理中的应用

### SOCKS5 UDP

SOCKS5 UDP ASSOCIATE：

```
SOCKS5 UDP ASSOCIATE：

流程：
1. 建立 TCP 控制连接（SOCKS5 握手）
2. 发送 UDP ASSOCIATE 请求
3. 服务端返回 UDP 中继地址
4. 客户端发送 UDP 数据报到中继
5. 中继转发到目标

UDP 数据帧封装：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  RSV  |  FRAG |  ATYP   |    DST.ADDR                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    DST.PORT                   |    DATA                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

注意：TCP 控制连接必须保持，否则 UDP 关联终止
```

详见 [[ref/protocol/socks5-spec|SOCKS5 规范]]。

### SS2022 UDP

Shadowsocks 2022 UDP 加密：

```
SS2022 UDP 加密流程：

1. 接收客户端 UDP 包
2. 解析 SeparateHeader
3. 验证 session_id
4. 解密 payload（XChaCha20-Poly1305）
5. 转发到目标

SeparateHeader 结构：
- type (1 字节)
- session_id (7 字节)
- padding (可选)

加密：
- 使用 XChaCha20-Poly1305（24 字节 nonce）
- 随机 nonce，无需状态管理
- AAD = SeparateHeader（加密后）

详见 [[core/protocol/shadowsocks|SS2022]]。
```

## QUIC 协议

### QUIC 基础

QUIC 基于 UDP 的可靠传输：

```
QUIC 特性：

基于 UDP：
- UDP 封装，避免 TCP 握手开销
- 可穿越 NAT 和防火墙
- 用户态实现拥塞控制

内置 TLS 1.3：
- 加密内置，握手集成
- 1-RTT（或 0-RTT）建立加密连接

连接迁移：
- Connection ID 标识连接
- IP/端口变化不影响连接
- 移动网络切换无缝

多路复用：
- 内置流（Stream）概念
- 无队头阻塞（HoL Blocking）

应用：
- HTTP/3（HTTP over QUIC）
- 快速连接恢复
```

详见 [[ref/protocol/quic-basics|QUIC 基础]]。

## UDP 可靠化

### 应用层可靠性

在 UDP 上实现可靠性：

```
UDP 可靠化方案：

1. 重传机制
   - 序列号标识包
   - ACK 确认接收
   - 超时重传丢失的包

2. 顺序控制
   - 包序号
   - 接收方排序
   - 丢弃重复包

3. 流量控制
   - 接收窗口
   - 发送速率限制

4. 拥塞控制
   - RTT 测量
   - 动态调整发送速率

QUIC 实现了完整的可靠性：
- 类 TCP 的可靠性
- 基于 UDP
- 更快的握手
- 更好的多路复用
```

## 在 Prism 中的应用

### UDP 转发

Prism 的 UDP 转发处理：

```
Prism UDP 转发：

入站：
- SOCKS5 UDP ASSOCIATE
- SS2022 UDP
- Transparent UDP

出站：
- 直连 UDP
- SS2022 UDP 出站

处理流程：
1. 接收 UDP 数据报
2. 解析协议封装
3. 解密（如果需要）
4. 转发到目标
5. 接收响应
6. 封装返回客户端
```

详见 [[core/channel/transport|传输层]]。

### NAT 处理

UDP NAT 行为：

```
UDP NAT 行为：

NAT 类型：
- Full Cone: 最宽松，任一外部可访问
- Restricted Cone: 仅允许已发送目标的响应
- Port Restricted: 更严格的端口限制
- Symmetric: 最严格，每个目标不同映射

影响：
- UDP 打洞（NAT 穿透）
- P2P 应用
- UDP 代理的可达性

Symmetric NAT 问题：
- 无法使用打洞技术
- 需要中继服务器
- 代理可能无法访问某些客户端
```

## 参见

- [[ref/network/udp|UDP 协议]] — UDP 详细原理
- [[ref/protocol/tcp-basics|TCP 基础]] — TCP 协议对比
- [[ref/protocol/quic-basics|QUIC 基础]] — QUIC 协议
- [[ref/protocol/socks5-spec|SOCKS5 规范]] — SOCKS5 UDP
- [[core/protocol/shadowsocks|SS2022]] — SS2022 UDP
- [[core/channel/transport|传输层]] — Prism 实现