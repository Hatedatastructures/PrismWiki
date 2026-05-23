---
title: "TCP 基础"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 793"
tags: [协议, tcp, 传输, 连接, 握手, 流量控制]
created: 2026-05-17
updated: 2026-05-17
---

# TCP 基础

**类别**: 协议

## 概述

TCP（Transmission Control Protocol）是互联网核心传输协议，提供可靠的、面向连接的字节流服务。TCP 是绝大多数代理协议的底层传输载体。

### 核心特性

TCP 的核心特性：

| 特性 | 说明 |
|------|------|
| **面向连接** | 三次握手建立，四次挥手断开 |
| **可靠传输** | 序列号、确认应答、超时重传 |
| **流量控制** | 滑动窗口防止发送过快 |
| **拥塞控制** | 动态调整避免网络拥塞 |
| **全双工** | 双向同时传输 |
| **字节流** | 无消息边界 |

### TCP 报文结构

TCP 报文的基本结构：

```
TCP 报文头部（20 字节基本）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data Offset | Res |U|A|P|R|S|F|            Window           |
|              |     |R|C|S|S|Y|I|                              |
|              |     |G|K|H|T|N|N|                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- Source Port (16 bit): 源端口
- Destination Port (16 bit): 目标端口
- Sequence Number (32 bit): 数据序列号
- Acknowledgment Number (32 bit): 确认号
- Data Offset (4 bit): 头部长度（32 bit 单位）
- Flags (6 bit): URG, ACK, PSH, RST, SYN, FIN
- Window (16 bit): 接收窗口大小
- Checksum (16 bit): 校验和
- Urgent Pointer (16 bit): 紧急指针
```

## 连接建立

### 三次握手

TCP 三次握手建立连接：

```
TCP 三次握手：

客户端                                    服务端
  |                                          |
  |---------- SYN (seq=x) ------------------>|
  |          状态：SYN_SENT                  |
  |                                          | 状态：LISTEN
  |                                          |
  |<--------- SYN-ACK (seq=y, ack=x+1) ------|
  |                                          | 状态：SYN_RECEIVED
  |                                          |
  |---------- ACK (seq=x+1, ack=y+1) -------->|
  |          状态：ESTABLISHED               |
  |                                          | 状态：ESTABLISHED
  |                                          |
  |<========= 双向数据传输 ==================>|

握手目的：
- SYN: 客户端请求建立连接，发送初始序列号
- SYN-ACK: 服务端同意连接，发送自己的初始序列号
- ACK: 客户端确认服务端的序列号

三次握手的必要性：
- 确保双方都能收发
- 防止旧的重复连接请求
- 同步双方序列号
```

### SYN 包格式

SYN 包的特殊格式：

```
SYN 包结构：

- Flags: SYN=1, ACK=0
- Sequence Number: 客户端初始序列号 (ISN)
- Acknowledgment Number: 无效（ACK=0）
- Options: 通常包含 MSS、Window Scale、SACK 等

常见选项：
- MSS (Maximum Segment Size): 最大报文段大小
- Window Scale: 窗口缩放因子
- SACK Permitted: 选择性确认支持
- Timestamps: 时间戳选项

ISN 生成：
- 不是从 0 开始
- 使用随机值防止序列号预测攻击
- 通常基于时钟和连接信息生成
```

## 连接断开

### 四次挥手

TCP 四次挥手断开连接：

```
TCP 四次挥手：

主动关闭方                                  被动关闭方
  |                                          |
  |---------- FIN (seq=m) ------------------>|
  |          状态：FIN_WAIT_1                | 状态：ESTABLISHED
  |                                          |
  |<--------- ACK (ack=m+1) -----------------|
  |          状态：FIN_WAIT_2                | 状态：CLOSE_WAIT
  |                                          |
  |                                          |---------- FIN (seq=n) -->|
  |                                          | 状态：LAST_ACK           |
  |                                          |
  |<--------- FIN (seq=n) -------------------|
  |                                          |
  |---------- ACK (ack=n+1) ----------------->|
  |          状态：TIME_WAIT                 | 状态：CLOSED
  |                                          |
  |  (等待 2MSL 后进入 CLOSED)               |

挥手目的：
- FIN: 请求关闭连接
- ACK: 确认 FIN
- 四次挥手因为 TCP 是全双工
- 每个方向的关闭是独立的
```

### TIME_WAIT 状态

TIME_WAIT 的作用：

```
TIME_WAIT 状态：

持续时间：2MSL (Maximum Segment Lifetime)
通常：60 秒（MSL = 30 秒）

作用：
1. 确保最后的 ACK 到达对方
   - 如果 ACK 丢失，对方会重发 FIN
   - TIME_WAIT 状态可以响应重发的 FIN

2. 等待旧连接的数据包消失
   - 防止旧数据包干扰新连接
   - 2MSL 确保所有数据包消失

问题：
- 大量短连接导致 TIME_WAIT 积累
- 占用端口资源
- 可能耗尽端口

解决方案：
- 使用连接池复用连接
- 开启 SO_REUSEADDR
- 快速回收 TIME_WAIT（谨慎）
```

## 可靠传输

### 序列号机制

TCP 序列号的工作方式：

```
序列号机制：

每个字节有序列号：
- 序列号是字节流的位置索引
- 不是报文序号，而是字节序号

发送方：
- 发送数据带序列号 seq
- seq = 本段第一个字节的序号

接收方：
- 确认号 ack = 期望收到的下一个字节序号
- ack 表示 ack 之前的所有字节都已收到

示例：
发送方发送 seq=1000, len=500 的数据
接收方收到后发送 ack=1500
表示 1000-1499 的字节已收到，期望 1500
```

### 超时重传

TCP 超时重传机制：

```
超时重传：

RTO (Retransmission Timeout)：
- 动态计算的超时时间
- 基于 RTT (Round-Trip Time) 估计

RTT 测量：
- 记录发送时间和 ACK 到达时间
- 计算样本 RTT
- 使用加权平均平滑 RTT

RTO 计算（经典）：
SRTT = (1-α) × SRTT + α × RTT_sample  (α = 0.125)
RTTVAR = (1-β) × RTTVAR + β × |RTT_sample - SRTT|  (β = 0.25)
RTO = SRTT + 4 × RTTVAR

重传策略：
- RTO 到达后重传
- 每次重传 RTO 加倍（指数退避）
- 达到限制后放弃
```

## 流量控制

### 滑动窗口

TCP 滑动窗口机制：

```
滑动窗口：

窗口字段：16 bit，最大 65535 字节
Window Scale 选项：可扩展到 1 GB

窗口含义：
- 接收方当前的接收缓冲区大小
- 发送方不能发送超过窗口的数据

工作方式：
发送方                      接收方
  |                            |
  |--- 发送窗口内数据 --------->|
  |                            | 缓冲区填充
  |                            |
  |<------ ACK + 新窗口 -------|
  |                            |
  |--- 继续发送 -------------->|

窗口关闭：
- 接收方缓冲区满，窗口=0
- 发送方停止发送
- 接收方处理后发送窗口更新
```

### 零窗口探测

窗口为零时的处理：

```
零窗口探测：

接收方窗口=0 时：
- 发送方停止发送
- 发送方定期发送 1 字节探测
- 接收方响应当前窗口大小

目的：
- 防止窗口更新包丢失导致死锁
- 接收方恢复后发送窗口更新
- 探测间隔逐渐增加

Persist Timer：
- 控制零窗口探测间隔
- 类似超时重传的指数退避
```

## 拥塞控制

### 拥塞控制算法

TCP 拥塞控制的核心算法：

```
拥塞控制算法：

1. 慢启动 (Slow Start)
   - 初始 cwnd = 1 MSS
   - 每个 ACK cwnd 加倍
   - 达到 ssthresh 后进入拥塞避免

2. 拥塞避免 (Congestion Avoidance)
   - 每个 RTT cwnd 增加 1 MSS
   - 线性增长
   - 检测到拥塞后降低

3. 快速重传 (Fast Retransmit)
   - 收到 3 个重复 ACK
   - 立即重传，不等 RTO

4. 快速恢复 (Fast Recovery)
   - 快速重传后
   - cwnd = ssthresh + 3
   - 每个 dup ACK cwnd 增加 1

参数：
- cwnd: 拥塞窗口（发送方限制）
- ssthresh: 慢启动阈值
- rwnd: 接收窗口（接收方限制）

发送窗口 = min(cwnd, rwnd)
```

###拥塞信号

TCP 检测拥塞的信号：

```
拥塞信号：

1. 超时 (Timeout)
   - 最强的拥塞信号
   - cwnd = 1, ssthresh = cwnd/2
   - 进入慢启动

2. 重复 ACK (Duplicate ACK)
   - 较弱的拥塞信号
   - 可能是丢包或乱序
   - 快速重传/恢复

3. ECN (Explicit Congestion Notification)
   - IP/TCP 头部标记
   - 显式拥塞通知
   - 不依赖丢包检测
```

## TCP 选项

### 常见选项

TCP 常见选项：

```
TCP 选项：

| 选项 | 类型 | 说明 |
|------|------|------|
| MSS | 2 | 最大报文段大小 |
| Window Scale | 3 | 窗口缩放因子 |
| SACK Permitted | 4 | 选择性确认支持 |
| SACK | 5 | 选择性确认块 |
| Timestamps | 8 | 时间戳（RTT 测量） |
| NOP | 1 | 填充 |

MSS 选项：
- 格式：类型=2, 长度=4, MSS 值(2 字节)
- 通常：1460（以太网 MTU 1500 - 40 字节头部）

Window Scale 选项：
- 格式：类型=3, 长度=3, shift count(1 字节)
- 窗口 = Window × 2^shift_count
- 最大 shift = 14（窗口最大 1 GB）
```

## 在 Prism 中的应用

### TCP 传输

Prism 使用 TCP 作为底层传输：

```
Prism TCP 使用：

入站协议：
- SOCKS5 over TCP
- HTTP CONNECT over TCP
- Trojan over TLS over TCP

出站协议：
- 直连 TCP
- TLS 加密 TCP
- Reality over TLS over TCP

连接管理：
- 连接池复用
- 多路复用（smux/yamux）
- Happy Eyeballs 双栈连接
```

详见 [[core/transport|传输层]]。

### TCP 配置优化

代理服务器的 TCP 优化：

```
TCP 优化建议：

1. 增大缓冲区
   SO_SNDBUF, SO_RCVBUF
   根据带宽延迟乘积调整

2. 禁用 Nagle 算法
   TCP_NODELAY
   减少小包延迟

3. 开启快速回收
   tcp_tw_reuse
   解决 TIME_WAIT 资源问题

4. 调整拥塞控制
   选择合适的拥塞控制算法
   BBR、CUBIC 等

5. 保持连接
   TCP Keepalive
   检测死连接
```

## 参见

- [[ref/network/tcp|TCP 协议]] — TCP 详细原理
- [[ref/protocol/udp-basics|UDP 基础]] — UDP 协议对比
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 双栈连接
- [[ref/network/connection-pool|连接池]] — 连接复用
- [[core/transport|传输层]] — Prism 实现