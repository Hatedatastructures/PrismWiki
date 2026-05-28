---
layer: dev
title: "TCP 协议实践"
source: "include/prism/transport/reliable.hpp"
module: "channel"
type: dev
tags: [tcp, transport, network, connection, boost-asio, socket]
created: 2026-05-13
updated: 2026-05-17
related:
  - "[[core/transport/reliable|reliable]]"
  - "[[core/connect/pool/pool|pool]]"
  - "[[core/connect/dial/racer|racer]]"
  - "[[dev/debugging/tls|tls]]"
  - "[[dev/debugging/udp|udp]]"
  - "[[core/multiplex/smux/craft|smux]]"
  - "[[core/multiplex/yamux/craft|yamux]]"
  - "[[core/instance/overview|agent]]"
layer: dev
---

# TCP 协议实践

TCP（Transmission Control Protocol）是互联网最核心的传输层协议，提供可靠的、有序的、面向连接的字节流传输。几乎所有代理协议都基于 TCP 构建，理解 TCP 的工作机制对于代理服务器开发至关重要。

## 概述

### TCP 的核心特性

TCP 是互联网协议栈中传输层的核心协议，由 RFC 793 定义并经过多次扩展（RFC 1122、RFC 1323、RFC 2018、RFC 6268 等）。相比 UDP，TCP 提供以下保证：

| 特性 | TCP | UDP | 对代理的影响 |
|------|-----|-----|-------------|
| 连接模式 | 面向连接 | 无连接 | 需要握手开销，但提供会话语义 |
| 可靠性 | 可靠传输（ACK + 重传） | 不保证 | 代理数据不会丢失 |
| 顺序性 | 有序（序列号） | 不保证 | 应用层无需处理乱序 |
| 流量控制 | 滑动窗口 | 无 | 防止发送方淹没接收方 |
| 拥塞控制 | 多种算法（Cubic、BBR） | 无 | 网络拥塞时自动降速 |
| 头部开销 | 20+ 字节 | 8 字节 | 每包有额外开销 |
| 数据边界 | 字节流（无边界） | 数据报保留边界 | 代理需要应用层协议定义边界 |

### TCP 头部结构

TCP 头部最小 20 字节，包含关键字段：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if Data Offset > 5)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

关键标志位：

| 标志 | 名称 | 含义 | 使用场景 |
|------|------|------|---------|
| SYN | Synchronize | 发起连接 | 三次握手第一步 |
| ACK | Acknowledge | 确认收到的数据 | 握手和数据传输 |
| FIN | Finish | 请求关闭连接 | 四次挥手 |
| RST | Reset | 强制重置连接 | 异常终止、拒绝连接 |
| PSH | Push | 推送数据到应用层 | 需要立即处理的数据 |
| URG | Urgent | 紧急指针有效 | 紧急数据传输（罕见） |

### TCP 与代理的关系

代理服务器的核心功能是在 TCP 连接之间转发数据：

```
[Client App] --TCP--> [Proxy Client] --TCP--> [Proxy Server] --TCP--> [Target Server]
```

代理对 TCP 的主要影响：

1. **额外延迟**：多一跳连接建立（两次握手 vs 一次）。典型的代理连接需要：
   - Client → Proxy：1-RTT 握手
   - Proxy → Target：1-RTT 握手
   - 总计 2-RTT，相比直连增加 1-RTT

2. **连接复用**：多路复用协议（[[core/multiplex/smux/craft|smux]]、[[core/multiplex/yamux/craft|yamux]]）可以在一条 TCP 上承载多个逻辑连接，消除部分握手开销。

3. **拥塞控制独立性**：代理两端分别进行拥塞控制：
   - Client-Proxy 段有自己的拥塞状态
   - Proxy-Target 段有独立的拥塞状态
   - 可能导致性能下降（队头阻塞）

4. **连接池复用**：[[core/connect/pool/pool|pool]] 可以复用空闲 TCP 连接，避免重复握手开销。

### TCP 数据流特性

TCP 是字节流协议，不保留消息边界：

```
发送方调用 3 次 write():
  write("AB", 2);
  write("CDE", 3);
  write("F", 1);

接收方可能收到的数据组合（任意）：
  1. "ABCDEF"（一次 read）
  2. "AB" + "CDE" + "F"（三次 read）
  3. "ABC" + "DEF"（两次 read）
  ...
```

这对代理协议的影响：

- HTTP、SOCKS5 等协议需要明确定义消息边界（如 Content-Length、固定头部长度）
- 多路复用协议需要帧头标记数据边界
- 需要缓冲区管理处理不完整的数据包

### TCP 可靠性机制

TCP 通过以下机制保证可靠性：

**ACK 确认机制**：

- 每个 TCP 段都有序列号（Sequence Number）
- 接收方通过 ACK 确认已收到的最大连续序列号
- 累积确认：ACK N 表示所有小于 N 的数据都已收到

**超时重传**：

- 发送方启动重传定时器（RTO，Retransmission Timeout）
- 未收到 ACK 时重传数据
- RTO 动态计算：基于测量 RTT 和波动

```
RTO = SRTT + max(G, 4*RTTVAR)
其中：
  SRTT = smoothed RTT（平滑 RTT）
  RTTVAR = RTT variance（RTT 方差）
  G = clock granularity（时钟粒度）
```

**快速重传**：

- 收到 3 个重复 ACK 时立即重传，无需等待 RTO
- 表明中间数据段丢失，后续数据段已到达

### TCP 流量控制

滑动窗口机制防止发送方淹没接收方：

```
发送方                              接收方
  |                                   |
  |  Window = 65535 字节              |
  |  已发送 30000 字节                |
  |  已确认 20000 字节                |
  |  可发送 35535 字节                |
  |                                   |
  |  ← ACK=25000, Win=50000           |  接收方处理慢，窗口缩小
  |                                   |
  |  可发送调整为 25000 字节          |
```

零窗口探测：

- 当接收方窗口为零时，发送方定时发送 1 字节探测
- 接收方回复当前窗口大小
- 避免死锁：接收方处理完毕后无法通知发送方

### TCP 拥塞控制

拥塞控制是 TCP 的核心算法，防止网络过载：

**四个阶段**：

1. **慢启动（Slow Start）**：初始 cwnd = 1 MSS，每收到一个 ACK 加倍，指数增长
2. **拥塞避免（Congestion Avoidance）**：cwnd 超过阈值后线性增长，每个 RTT 增加 1 MSS
3. **快速重传（Fast Retransmit）**：收到 3 个重复 ACK，立即重传丢失段
4. **快速恢复（Fast Recovery）**：快速重传后，cwnd 减半，进入拥塞避免

**常见拥塞控制算法**：

| 算法 | 特点 | 适用场景 | 代理推荐 |
|------|------|----------|----------|
| Cubic | Linux 默认，基于丢包 | 通用场景 | 默认可用 |
| BBR | 基于带宽估计，不依赖丢包 | 高延迟/高丢包链路 | 推荐 |
| BBRv2 | 改进公平性，混合丢包+带宽 | 与 Cubic 混合环境 | 推荐 |
| Reno | 传统 AIMD | 低带宽网络 | 不推荐 |
| Vegas | 基于延迟估计 | 低延迟网络 | 不推荐 |

在代理场景中推荐 BBR 或 BBRv2，原因：

- 代理链路通常较长，RTT 较大
- 国际链路可能丢包率较高
- BBR 不将丢包视为拥塞信号，更稳定

## TCP 连接流程

### 三次握手（连接建立）

TCP 使用三次握手建立连接，确保双方都准备好接收数据：

```
Client (主动发起方)                    Server (被动监听方)
  |                                       |
  |  CLOSED                               |  LISTEN
  |                                       |
  |------- SYN (seq=x) --------------->   |  Step 1: Client 发起
  |       Flags: SYN=1, ACK=0             |  Server 收到 → SYN_RCVD
  |                                       |
  |<------ SYN-ACK (seq=y, ack=x+1) ----   |  Step 2: Server 确认并发起
  |       Flags: SYN=1, ACK=1             |  Client 收到 → ESTABLISHED
  |                                       |
  |------- ACK (ack=y+1) -------------->   |  Step 3: Client 确认
  |       Flags: SYN=0, ACK=1             |  Server 收到 → ESTABLISHED
  |                                       |
  |  ESTABLISHED                          |  ESTABLISHED
  |                                       |
  |       双向数据传输开始                 |
```

握手过程详解：

**Step 1 - SYN（发起）**：

- 客户端选择初始序列号（ISN）x
- 发送 SYN 段：`SEQ=x, SYN=1, ACK=0`
- 状态：`CLOSED → SYN_SENT`
- 客户端等待 SYN-ACK，超时后重传 SYN

**Step 2 - SYN-ACK（确认并发起）**：

- 服务端收到 SYN，验证合法性（未超限）
- 选择自己的 ISN y
- 发送 SYN-ACK 段：`SEQ=y, ACK=x+1, SYN=1, ACK=1`
- 状态：`LISTEN → SYN_RCVD`
- 服务端等待 ACK，超限后丢弃半连接

**Step 3 - ACK（确认）**：

- 客户端收到 SYN-ACK，验证 ACK 字段
- 发送 ACK 段：`SEQ=x+1, ACK=y+1, ACK=1`
- 状态：`SYN_SENT → ESTABLISHED`
- 可以携带数据（TCP 快速打开）

### 为什么是三次握手

三次握手而非两次的原因：

1. **防止历史连接**：假设旧 SYN 延迟到达，两次握手会导致服务端误建立连接
2. **同步序列号**：双方都需要确认对方的 ISN
3. **双向确认**：确认双方都准备好发送和接收

```
假设两次握手：

Client 发送 SYN(旧,seq=x) -- 延迟到达 Server
Client 发送 SYN(新,seq=x+1000) -- 正常到达 Server，建立连接
Client 完成通信后关闭连接

此时旧的 SYN(x) 到达 Server：
  Server 误以为是新连接请求
  Server 发送 SYN-ACK
  Server 等待数据（但 Client 已经不存在）
  
结果：Server 资源浪费，半连接状态
```

### TCP 状态变迁

完整的 TCP 状态变迁图：

```
                              +-----------------+
                              |     CLOSED      |
                              +-----------------+
                                    |  passive
                                    |  open
                                    v
                              +-----------------+
                              |     LISTEN      |
                              +-----------------+
                                    |  SYN
                                    v
                              +-----------------+
                              |    SYN_RCVD     |
                              +-----------------+
                                    |  ACK
                                    v
    active open          +-----------------+         passive close
    SYN -------------->  |   ESTABLISHED   |  <---------- FIN
                          +-----------------+
                                |       |
                                |       | active close
                                |       v
                                | +-----------------+
                                | |   FIN_WAIT_1    |
                                | +-----------------+
                                |       | ACK
                                |       v
                                | +-----------------+
                                | |   FIN_WAIT_2    |
                                | +-----------------+
                                |       | FIN
                                v       v
                          +-----------------+
                          |    TIME_WAIT    |  <-- 2MSL 等待
                          +-----------------+
                                | timeout
                                v
                          +-----------------+
                          |     CLOSED      |
                          +-----------------+

被动关闭方：

ESTABLISHED <-- FIN --> CLOSE_WAIT
CLOSE_WAIT -- FIN --> LAST_ACK
LAST_ACK <-- ACK --> CLOSED
```

关键状态说明：

| 状态 | 含义 | 常见问题 |
|------|------|---------|
| LISTEN | 服务端等待连接 | 无 |
| SYN_SENT | 客户端等待 SYN-ACK | SYN 被丢弃、超时 |
| SYN_RCVD | 服务端等待 ACK | 半连接队列满、SYN Flood |
| ESTABLISHED | 连接已建立，可传输数据 | 正常状态 |
| FIN_WAIT_1 | 主动关闭方，等待 ACK | ACK 丢失 |
| FIN_WAIT_2 | 主动关闭方，等待 FIN | 被动方不发送 FIN |
| TIME_WAIT | 主动关闭方，等待 2MSL | 端口耗尽 |
| CLOSE_WAIT | 被动关闭方，等待应用关闭 | 应用未调用 close |
| LAST_ACK | 被动关闭方，等待最终 ACK | ACK 丢失 |

### 四次挥手（连接关闭）

TCP 使用四次挥手关闭连接，确保双方都完成数据传输：

```
主动关闭方 (Client)                    被动关闭方 (Server)
  |                                       |
  |  ESTABLISHED                          |  ESTABLISHED
  |                                       |
  |------- FIN (seq=m) --------------->   |  Step 1: 请求关闭
  |       Flags: FIN=1, ACK=1             |  Server 收到 → CLOSE_WAIT
  |                                       |
  |<------ ACK (ack=m+1) --------------   |  Step 2: 确认关闭请求
  |       Flags: FIN=0, ACK=1             |  Client 收到 → FIN_WAIT_2
  |                                       |
  |         Server 继续发送剩余数据        |
  |                                       |
  |<------ FIN (seq=n) ---------------    |  Step 3: Server 请求关闭
  |       Flags: FIN=1, ACK=1             |  Client 收到 → TIME_WAIT
  |                                       |
  |------- ACK (ack=n+1) -------------->   |  Step 4: 确认关闭
  |       Flags: FIN=0, ACK=1             |  Server 收到 → CLOSED
  |                                       |
  |  TIME_WAIT (2MSL)                     |  CLOSED
  |       等待网络残留报文消散             |
  |                                       |
  |  CLOSED                               |
```

挥手过程详解：

**Step 1 - FIN（请求关闭）**：

- 主动关闭方调用 `close()`，发送 FIN
- 表示"我不再发送数据"
- 状态：`ESTABLISHED → FIN_WAIT_1`

**Step 2 - ACK（确认）**：

- 被动关闭方收到 FIN，回复 ACK
- 表示"我收到了你的关闭请求"
- 被动方仍可以发送剩余数据
- 状态：主动方 `FIN_WAIT_1 → FIN_WAIT_2`，被动方 `ESTABLISHED → CLOSE_WAIT`

**Step 3 - FIN（被动方关闭）**：

- 被动关闭方发送完毕后调用 `close()`
- 发送自己的 FIN
- 状态：`CLOSE_WAIT → LAST_ACK`

**Step 4 - ACK（最终确认）**：

- 主动关闭方收到 FIN，回复 ACK
- 状态：`FIN_WAIT_2 → TIME_WAIT`
- 被动方状态：`LAST_ACK → CLOSED`

### TIME_WAIT 状态详解

TIME_WAIT 是主动关闭方进入的特殊状态，持续 2MSL（Maximum Segment Lifetime）：

**MSL 定义**：

- MSL 是 TCP 段在网络中存在的最大时间
- RFC 793 定义 MSL = 2 分钟（120 秒）
- Linux 默认 MSL = 30 秒，TIME_WAIT = 60 秒
- Windows 默认 TIME_WAIT = 4 分钟（240 秒）

**TIME_WAIT 的作用**：

1. **确保最后一个 ACK 到达**：
   - 如果 ACK 丢失，被动方会重发 FIN
   - TIME_WAIT 状态可以响应重发的 FIN

2. **等待网络残留报文消散**：
   - 防止旧连接的报文干扰新连接
   - 2MSL 保证往返时间内的报文都消散

```
假设没有 TIME_WAIT：

Old Connection: Client:portA → Server:portB
New Connection: Client:portA → Server:portB (立即复用)

如果 Old Connection 的延迟报文到达 Server：
  Server 无法区分是 Old 还是 New
  可能导致数据混乱
```

**TIME_WAIT 问题**：

高并发代理服务器可能出现大量 TIME_WAIT 连接：

```
$ netstat -an | grep TIME_WAIT | wc -l
28672  // 可能导致端口耗尽
```

端口范围限制（Linux）：

```
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999  // 约 28K 个可用端口
```

**解决方案**：

1. **SO_REUSEADDR / SO_REUSEPORT**：

   ```cpp
   int opt = 1;
   setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
   setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
   ```

   允许新连接绑定 TIME_WAIT 状态的端口。

2. **内核参数调整**：

   ```bash
   # Linux
   net.ipv4.tcp_tw_reuse = 1    # 允许复用 TIME_WAIT 连接（仅出站）
   net.ipv4.tcp_tw_recycle = 0  # 已废弃，存在安全问题
   net.ipv4.tcp_fin_timeout = 30  # 缩短 FIN_WAIT_2 超时
   ```

3. **连接池复用**：

   Prism 使用 [[core/connect/pool/pool|pool]] 复用空闲连接，避免频繁关闭。

4. **被动关闭策略**：

   让服务端主动关闭连接，客户端进入 CLOSE_WAIT 而非 TIME_WAIT。

### SYN Flood 攻击防护

SYN Flood 利用三次握手的半连接状态：

```
攻击者：
  发送大量伪造源 IP 的 SYN
  不响应 SYN-ACK
  
服务端：
  半连接队列（SYN_RCVD 状态）被填满
  正常连接无法建立
```

防护手段：

**1. SYN Cookie**：

- 不分配资源存储半连接状态
- 将状态编码在 SYN-ACK 的序列号中
- 客户端返回 ACK 时恢复状态

```
SYN Cookie 序列号计算：
  seq = encode(source_ip, dest_ip, source_port, dest_port, timestamp, MSS)

服务端收到 ACK：
  ack - 1 即为之前发送的 SYN-ACK 序列号
  解码验证，成功则分配资源
```

**2. SYN Cache**：

- 限制半连接队列大小
- 使用更紧凑的数据结构
- 超限时丢弃最早或随机 SYN

**3. 防火墙限速**：

- 限制 SYN 包速率
- 异常流量触发告警

### TCP 半打开与半关闭

**半打开（Half-Open）**：

一方已关闭或崩溃，另一方不知道：

```
Client 发送数据 → Server 已崩溃
  Client 等待 ACK
  Server 无响应
  Client 重传多次后触发 RST
```

检测机制：TCP Keep-Alive（见下节）。

**半关闭（Half-Close）**：

一方停止发送但仍接收：

```
Client: "我发完了（FIN），但仍可接收你的数据"
Server: "收到（ACK），我还有数据要发"
Server: 发送剩余数据
Server: "我也发完了（FIN）"
Client: "收到（ACK），关闭"
```

应用场景：HTTP 请求-响应模式中，客户端发完请求后关闭写端，等待响应。

TCP 支持 `shutdown()` 实现半关闭：

```cpp
// 关闭写端，仍可接收数据
shutdown(fd, SHUT_WR);  // 发送 FIN

// 关闭读端，仍可发送数据
shutdown(fd, SHUT_RD);

// 全关闭（等同于 close）
shutdown(fd, SHUT_RDWR);
```

Prism 中 [[core/transport/reliable|reliable]] 实现了 `shutdown_write()` 方法。

### TCP 保活（Keep-Alive）

TCP Keep-Alive 用于检测死连接：

**内核参数**：

```bash
# Linux 默认值
net.ipv4.tcp_keepalive_time = 7200   # 空闲 2 小时后开始探测
net.ipv4.tcp_keepalive_intvl = 75    # 探测间隔 75 秒
net.ipv4.tcp_keepalive_probes = 9    # 探测 9 次无响应则断开

# Windows 默认值
KeepAliveTime = 7200000 (2 小时)
KeepAliveInterval = 1000 (1 秒)
```

**工作流程**：

```
Connection idle for keepalive_time
  |--- Keep-Alive Probe (seq=last_seq-1, ACK=1) --->|
  |<-- ACK ----------------------------------------|
  |   Connection alive                             |
  |                                                |
  |  若无响应：等待 keepalive_intvl               |
  |--- Keep-Alive Probe --------------------------->|
  |                                                |
  |  重复 keepalive_probes 次                     |
  |  全部无响应 → RST → Connection closed          |
```

**代理场景的问题**：

- 内核级 Keep-Alive 间隔太长（2 小时）
- 代理需要更频繁的存活检测（秒级）
- 部分 NAT 设备会丢弃纯 ACK 包（视为无数据）
- GFW 可能识别 Keep-Alive 包特征

**替代方案**：

1. 应用层心跳：
   - 定期发送带数据的探测包
   - 如 SOCKS5 的心跳、多路复用的 ping 帧

2. 缩短内核参数：
   ```bash
   net.ipv4.tcp_keepalive_time = 60   # 1 分钟
   net.ipv4.tcp_keepalive_intvl = 10  # 10 秒
   net.ipv4.tcp_keepalive_probes = 6  # 6 次
   ```

Prism 默认启用 `SO_KEEPALIVE`，参数可在配置中调整。

## Prism TCP 实现

### Boost.Asio TCP 封装

Prism 使用 Boost.Asio 进行异步 TCP 操作，核心类：

**tcp::socket**：TCP 连接端点

```cpp
namespace net = boost::asio;
using tcp = net::ip::tcp;

// 创建 socket
tcp::socket socket(io_context);

// 异步连接
socket.async_connect(endpoint, [](std::error_code ec) {
    if (!ec) {
        // 连接成功
    }
});

// 异步读取
socket.async_read_some(net::buffer(data, size), [](std::error_code ec, size_t bytes) {
    if (!ec) {
        // 读取成功
    }
});

// 异步写入
socket.async_write_some(net::buffer(data, size), [](std::error_code ec, size_t bytes) {
    if (!ec) {
        // 写入成功
    }
});
```

**tcp::acceptor**：TCP 监听器

```cpp
tcp::acceptor acceptor(io_context, tcp::endpoint(tcp::v4(), port));

// 异步接受连接
acceptor.async_accept([](std::error_code ec, tcp::socket socket) {
    if (!ec) {
        // 新连接到达
    }
});
```

**tcp::resolver**：地址解析器

```cpp
tcp::resolver resolver(io_context);

// 异步解析
resolver.async_resolve(host, port, [](std::error_code ec, tcp::resolver::results_type results) {
    if (!ec) {
        for (auto& entry : results) {
            // entry.endpoint() 是解析结果
        }
    }
});
```

### 异步模型与协程

Prism 使用 C++23 协程封装 Boost.Asio，所有 TCP 操作返回 `net::awaitable<T>`：

```cpp
// 同步风格（阻塞，Prism 禁止）
tcp::socket socket(io_context);
socket.connect(endpoint);  // 阻塞当前线程

// 异步回调风格（传统）
socket.async_connect(endpoint, [](std::error_code ec) { ... });

// 协程风格（Prism 采用）
auto connect_and_read() -> net::awaitable<std::string> {
    tcp::socket socket(co_await net::this_coro::executor);
    
    // 异步连接，等待完成
    co_await socket.async_connect(endpoint, net::use_awaitable);
    
    // 异步读取，等待完成
    std::string buffer(1024, '\0');
    auto bytes = co_await socket.async_read_some(
        net::buffer(buffer), net::use_awaitable);
    
    co_return buffer.substr(0, bytes);
}
```

**协程优势**：

- 顺序写异步代码，无回调嵌套
- 自动管理协程生命周期
- 错误处理统一（try-catch 或 redirect_error）

### reliable 传输层

[[core/transport/reliable|reliable]] 是 Prism 的 TCP 传输层实现，继承 `transmission` 接口：

```cpp
class reliable : public transmission {
public:
    using socket_type = net::ip::tcp::socket;
    
    // 构造函数
    explicit reliable(net::any_io_executor executor);
    explicit reliable(socket_type socket);
    explicit reliable(psm::connect::pooled_connection pooled);
    
    // 异步读取（返回实际读取字节数）
    auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;
    
    // 异步写入
    auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;
    
    // Scatter-gather 写入优化
    auto async_write_scatter(const std::span<const std::byte> *buffers,
                             std::size_t count, std::error_code &ec)
        -> net::awaitable<std::size_t> override;
    
    // 关闭（连接池复用时归还而非关闭）
    void close() override;
    
    // 取消异步操作
    void cancel() override;
    
    // 半关闭写端
    void shutdown_write() override;
    
    // 是否可靠传输
    [[nodiscard]] bool is_reliable() const noexcept override;
    
    // 获取底层 socket
    socket_type &native_socket() noexcept;
};
```

**工厂函数**：

```cpp
// 创建未打开的传输层
auto make_reliable(net::any_io_executor executor) -> shared_transmission;

// 从已连接 socket 创建
auto make_reliable(net::ip::tcp::socket socket) -> shared_transmission;

// 从连接池连接创建（析构时归还）
auto make_reliable(psm::connect::pooled_connection pooled) -> shared_transmission;
```

### Scatter-Gather 写入优化

TCP 字节流特性导致协议需要帧头标记边界：

```
[Frame Header] [Payload Data]
     4 bytes      N bytes
```

传统方式需要两次 write：

```cpp
// 传统方式（两次系统调用）
socket.async_write_some(buffer(header, 4));
socket.async_write_some(buffer(payload, N));
```

Scatter-Gather 将多个缓冲区合并为单次系统调用：

```cpp
// Scatter-Gather（单次系统调用）
std::span<const std::byte> buffers[2] = {
    std::span<const std::byte>(header, 4),
    std::span<const std::byte>(payload, N)
};
transport->async_write_scatter(buffers, 2, ec);
```

底层实现：

- Linux：`writev()` 系统调用
- Windows：`WSASend()` 系统调用
- 减少系统调用开销，提升吞吐量

### 连接池复用

[[core/connect/pool/pool|pool]] 提供 TCP 连接池，避免重复握手：

**核心流程**：

```
                    +------------------+
                    | connection_pool  |
                    +------------------+
                           | cache_
                           | unordered_map<endpoint_key, stack<socket*>>
                           |
    async_acquire(ep) ---->|  1. 查找缓存
                           |     ├─ 命中 → 弹出栈顶 → 健康检测 → 返回
                           |     └─ 未命中 → 创建新连接
                           |
    pooled_connection      |  2. 使用连接
    析构 ----------------->|  3. recycle 归还
                           |     ├─ 健康检测通过 → 入栈等待复用
                           |     └─ 检测失败 → 关闭销毁
                           |
    后台定时器 ----------->|  4. cleanup 清理过期连接
```

**配置参数**：

```cpp
struct config {
    uint32_t max_cache_per_endpoint = 32;  // 单端点最大缓存数
    uint64_t connect_timeout_ms = 300;     // 连接超时
    uint64_t max_idle_seconds = 30;        // 空闲超时
    uint64_t cleanup_interval_sec = 10;    // 清理间隔
    uint32_t recv_buffer_size = 65536;     // 接收缓冲区
    uint32_t send_buffer_size = 65536;     // 发送缓冲区
    bool tcp_nodelay = true;               // 禁用 Nagle
    bool keep_alive = true;                // 启用 Keep-Alive
};
```

**使用示例**：

```cpp
auto acquire_and_use(connection_pool &pool, tcp::endpoint ep)
    -> net::awaitable<void> {
    // 获取连接（复用或新建）
    auto [code, conn] = co_await pool.async_acquire(ep);
    
    if (code != fault::code::success) {
        // 连接失败
        co_return;
    }
    
    // 创建传输层（析构时归还到池）
    auto transport = make_reliable(std::move(conn));
    
    // 使用连接
    co_await transport->async_write(...);
    co_await transport->async_read(...);
    
    // transport 析构 → pooled_connection 析构 → 归还到池
}
```

### Happy Eyeballs（RFC 8305）

[[core/connect/dial/racer|racer]] 实现 Happy Eyeballs 算法，解决双栈连接选择问题：

**问题背景**：

DNS 返回多个地址（IPv4 + IPv6），传统方式按顺序尝试：

```
传统方式：
  Try IPv6 → 失败（超时 30s） → Try IPv4 → 成功
  总耗时：30s + RTT
```

**Happy Eyeballs 算法**：

```
Happy Eyeballs：
  IPv6_1 立即尝试（0ms）
  IPv6_2 延迟 250ms
  IPv4_1 延迟 500ms
  IPv4_2 延迟 750ms
  ...
  第一个成功 → 其他取消
  总耗时：RTT（最佳情况）
```

**Prism 实现**：

```cpp
class address_racer {
public:
    explicit address_racer(connection_pool &pool);
    
    // 并发竞速连接
    [[nodiscard]] auto race(std::span<const tcp::endpoint> endpoints)
        -> net::awaitable<pooled_connection>;
};
```

流程：

1. 端点列表排序（IPv6 优先）
2. 第一个端点立即尝试
3. 后续端点按 250ms 延迟依次启动
4. 第一个成功 → `exchange` 竞争 winner → 返回
5. 落败者 → 归还连接到池

### Socket 选项配置

Prism 设置的关键 socket 选项：

**TCP_NODELAY**：

禁用 Nagle 算法，降低交互延迟：

```cpp
// Nagle 算法原理
// 小数据包会被缓存，等待更多数据或 ACK
// 延迟最多 200ms

// TCP_NODELAY = true
// 立即发送，不等待
// 适合交互式应用（代理）
```

**SO_REUSEADDR**：

允许快速重启绑定相同端口：

```cpp
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

**SO_KEEPALIVE**：

启用 TCP Keep-Alive：

```cpp
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
```

**缓冲区大小**：

```cpp
// 接收缓冲区
uint32_t recv_buf = 65536;
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &recv_buf, sizeof(recv_buf));

// 发送缓冲区
uint32_t send_buf = 65536;
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &send_buf, sizeof(send_buf));
```

## 使用示例

### 基本连接建立

```cpp
#include <prism/channel/transport/reliable.hpp>
#include <prism/channel/connection/pool.hpp>

using namespace psm::channel;

// 创建连接池
net::io_context ioc;
connection_pool::config pool_cfg;
connection_pool pool(ioc, pool_cfg);

// 建立连接
auto connect_and_transfer() -> net::awaitable<void> {
    // 解析目标地址
    tcp::resolver resolver(ioc);
    auto results = co_await resolver.async_resolve(
        "example.com", "80", net::use_awaitable);
    
    // 获取连接
    auto endpoint = results.begin()->endpoint();
    auto [code, conn] = co_await pool.async_acquire(endpoint);
    
    if (code != fault::code::success) {
        // 错误处理
        co_return;
    }
    
    // 创建传输层
    auto transport = make_reliable(std::move(conn));
    
    // 发送 HTTP 请求
    std::string request = "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
    co_await transport->async_write(
        std::span<const std::byte>(
            reinterpret_cast<const std::byte*>(request.data()),
            request.size()));
    
    // 接收响应
    std::vector<std::byte> buffer(4096);
    auto bytes = co_await transport->async_read(std::span(buffer));
    
    // 处理响应...
}
```

### 带超时的连接

```cpp
auto connect_with_timeout(connection_pool &pool, tcp::endpoint ep)
    -> net::awaitable<pooled_connection> {
    net::steady_timer timer(pool.get_executor());
    timer.expires_after(std::chrono::seconds(5));
    
    // 启动连接协程
    auto connect_coro = pool.async_acquire(ep);
    
    // 等待连接或超时
    auto result = co_await net::experimental::make_parallel_group(
        net::co_spawn(pool.get_executor(), std::move(connect_coro), net::use_awaitable),
        timer.async_wait(net::use_awaitable)
    ).async_wait(net::experimental::wait_for_one());
    
    if (result.index() == 1) {  // 超时
        co_return pooled_connection{};
    }
    
    auto [code, conn] = std::get<0>(result);
    if (code != fault::code::success) {
        co_return pooled_connection{};
    }
    
    co_return std::move(conn);
}
```

### 连接池统计监控

```cpp
void monitor_pool(connection_pool &pool) {
    auto stats = pool.stats();
    
    std::cout << "连接池统计:\n";
    std::cout << "  空闲连接数: " << stats.idle_count << "\n";
    std::cout << "  缓存端点数: " << stats.endpoint_count << "\n";
    std::cout << "  总获取次数: " << stats.total_acquires << "\n";
    std::cout << "  缓存命中数: " << stats.total_hits << "\n";
    std::cout << "  新建连接数: " << stats.total_creates << "\n";
    std::cout << "  归还次数:   " << stats.total_recycles << "\n";
    std::cout << "  驱逐次数:   " << stats.total_evictions << "\n";
    
    double hit_rate = static_cast<double>(stats.total_hits) / stats.total_acquires;
    std::cout << "  命中率:     " << (hit_rate * 100) << "%\n";
}
```

### Happy Eyeballs 使用

```cpp
auto connect_happy_eyeballs(connection_pool &pool)
    -> net::awaitable<void> {
    // DNS 解析返回多个地址
    tcp::resolver resolver(ioc);
    auto results = co_await resolver.async_resolve(
        "www.example.com", "443", net::use_awaitable);
    
    // 收集端点
    std::vector<tcp::endpoint> endpoints;
    for (auto it = results.begin(); it != results.end(); ++it) {
        endpoints.push_back(it->endpoint());
    }
    
    // 端点排序（IPv6 优先）
    std::sort(endpoints.begin(), endpoints.end(),
        [](const tcp::endpoint& a, const tcp::endpoint& b) {
            return a.address().is_v6() && !b.address().is_v6();
        });
    
    // Happy Eyeballs 竞速
    address_racer racer(pool);
    auto conn = co_await racer.race(std::span(endpoints));
    
    if (!conn.valid()) {
        // 所有地址都失败
        co_return;
    }
    
    // 使用连接...
}
```

### 半关闭示例

```cpp
auto half_close_example(shared_transmission transport)
    -> net::awaitable<void> {
    // 发送完毕，关闭写端
    transport->shutdown_write();
    
    // 仍可接收数据
    std::vector<std::byte> buffer(1024);
    auto bytes = co_await transport->async_read(std::span(buffer));
    
    // 处理接收的数据...
    
    // 完全关闭
    transport->close();
}
```

## 最佳实践

### 连接复用优先

避免频繁建立和关闭连接：

- 使用连接池复用空闲连接
- 多路复用协议在单连接上承载多流
- 主动关闭方会产生 TIME_WAIT，考虑让服务端主动关闭

### 正确处理 TCP 字节流

TCP 不保留消息边界，应用层需定义：

- 使用固定长度头部标记消息大小
- 使用分隔符（如 HTTP 的 `\r\n\r\n`）
- 使用应用层帧结构（如多路复用帧头）

### 异步操作超时设置

所有异步操作都应设置超时：

```cpp
// 连接超时
pool.config().connect_timeout_ms = 300;  // 300ms

// 读/写超时需要应用层定时器
auto read_with_timeout(shared_transmission transport)
    -> net::awaitable<std::vector<std::byte>> {
    net::steady_timer timer(transport->executor());
    timer.expires_after(std::chrono::seconds(10));
    
    // 并行等待读或超时
    auto result = co_await net::experimental::make_parallel_group(
        transport->async_read(...),
        timer.async_wait(net::use_awaitable)
    ).async_wait(net::experimental::wait_for_one());
    
    if (result.index() == 1) {  // 超时
        transport->close();
        throw std::runtime_error("read timeout");
    }
    
    co_return std::get<0>(result);
}
```

### 错误处理

TCP 连接可能随时断开，正确处理错误：

```cpp
auto safe_transfer(shared_transmission transport)
    -> net::awaitable<void> {
    std::error_code ec;
    
    while (true) {
        auto bytes = co_await transport->async_read_some(buffer, ec);
        
        if (ec) {
            if (ec == net::error::eof) {
                // 对端正常关闭
                break;
            } else if (ec == net::error::operation_aborted) {
                // 操作被取消（如超时）
                break;
            } else {
                // 其他错误
                break;
            }
        }
        
        // 处理数据...
    }
}
```

### 拥塞控制选择

根据场景选择拥塞控制算法：

- 国际代理链路：推荐 BBR 或 BBRv2
- 国内低延迟链路：Cubic 可用
- 高丢包率环境：BBR 不依赖丢包

Linux 配置：

```bash
# 查看当前算法
sysctl net.ipv4.tcp_congestion_control

# 设置 BBR
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

### 避免阻塞操作

在协程中禁止阻塞操作：

- 禁止 `socket.connect()` 同步调用
- 禁止 `recv()` / `send()` 同步调用
- 禁止 `sleep()` 或条件变量等待
- 使用异步 API + `co_await`

## 常见问题

### 连接超时频繁

**现象**：大量 `fault::code::timeout` 错误

**原因**：

1. 网络延迟过高
2. 目标服务器响应慢
3. 超时设置过短

**解决方案**：

```cpp
// 调整超时配置
pool.config().connect_timeout_ms = 1000;  // 增加到 1 秒

// 或使用 Happy Eyeballs 降低等待时间
address_racer racer(pool);
auto conn = co_await racer.race(endpoints);
```

### TIME_WAIT 端口耗尽

**现象**：无法建立新连接，`Cannot assign requested address`

**原因**：大量 TIME_WAIT 状态占用端口

**诊断**：

```bash
netstat -an | grep TIME_WAIT | wc -l
```

**解决方案**：

```bash
# 内核参数
sysctl -w net.ipv4.tcp_tw_reuse=1

# 或使用连接池减少关闭次数
```

### 连接池命中率低

**现象**：统计显示命中率 < 10%

**原因**：

1. 目标端点分散（代理访问大量不同目标）
2. 空闲超时过短，连接被清理
3. 单端点缓存容量过小

**解决方案**：

```cpp
// 增加空闲超时和缓存容量
pool.config().max_idle_seconds = 60;
pool.config().max_cache_per_endpoint = 64;
```

### 连接被意外关闭

**现象**：数据传输中突然收到 EOF

**原因**：

1. Keep-Alive 超时（长时间空闲）
2. 服务端超时关闭
3. 中间设备（NAT/防火墙）超时

**解决方案**：

```cpp
// 缩短 Keep-Alive 间隔
pool.config().keep_alive = true;

// 或使用应用层心跳
```

### 读操作返回 0 字节

**现象**：`async_read_some` 返回 0 字节且无错误

**原因**：

- 对端关闭写端（发送 FIN）
- 本端读端仍打开，但无数据可读

**处理**：

```cpp
auto bytes = co_await transport->async_read_some(buffer, ec);
if (bytes == 0 && !ec) {
    // 对端关闭写端
    transport->close();  // 完全关闭
}
```

## 排障指南

### 连接建立问题

**诊断步骤**：

1. 检查网络连通性：

   ```bash
   ping target_server
   traceroute target_server
   ```

2. 检查端口可达性：

   ```bash
   nc -zv target_server port
   telnet target_server port
   ```

3. 检查防火墙规则：

   ```bash
   iptables -L -n
   netstat -an | grep :port
   ```

4. 检查连接池日志：

   ```cpp
   auto stats = pool.stats();
   // 查看 total_creates vs total_acquires 比例
   ```

**常见错误码**：

| 错误码 | 含义 | 排查方向 |
|--------|------|---------|
| `connection_refused` | 目标拒绝连接 | 目标未监听、防火墙阻断 |
| `host_unreachable` | 主机不可达 | 网络、路由问题 |
| `network_unreachable` | 网络不可达 | 路由、网关问题 |
| `timeout` | 超时 | 网络延迟、超时设置 |
| `operation_aborted` | 操作取消 | 应用取消、超时 |

### 数据传输问题

**诊断步骤**：

1. 检查 socket 状态：

   ```bash
   ss -tan state established '( dport = :port or sport = :port )'
   ```

2. 检查缓冲区：

   ```bash
   ss -tan '( dport = :port )' | grep -iRecv-Q -iSend-Q
   ```

3. 抓包分析：

   ```bash
   tcpdump -i any port target_port -w capture.pcap
   ```

4. 检查流量控制：

   - 观察 `Recv-Q` 和 `Send-Q` 是否堆积
   - 检查窗口大小变化

**缓冲区堆积分析**：

| 状态 | Recv-Q | Send-Q | 可能原因 |
|------|--------|--------|---------|
| 正常 | 0 | 0 | 无堆积 |
| 接收慢 | >0 | 0 | 应用处理慢 |
| 发送慢 | 0 | >0 | 网络拥塞、对端窗口小 |
| 双向慢 | >0 | >0 | 双向拥塞 |

### 连接关闭问题

**诊断步骤**：

1. 检查状态分布：

   ```bash
   netstat -an | awk '/tcp/ {print $6}' | sort | uniq -c
   ```

2. 检查 TIME_WAIT 数量：

   ```bash
   netstat -an | grep TIME_WAIT | wc -l
   ```

3. 检查 CLOSE_WAIT 数量：

   ```bash
   netstat -an | grep CLOSE_WAIT | wc -l
   ```

**异常状态分析**：

| 状态异常 | 可能原因 | 解决方案 |
|----------|----------|---------|
| TIME_WAIT 过多 | 频繁短连接 | 连接池、SO_REUSEADDR |
| CLOSE_WAIT 过多 | 应用未 close | 检查资源泄漏 |
| FIN_WAIT_2 过多 | 对端不 FIN | 检查对端状态 |
| SYN_SENT 过多 | 连接超时 | 检查网络、超时设置 |

### 性能问题

**诊断步骤**：

1. 测量 RTT：

   ```bash
   ping -c 10 target_server | tail -1
   ```

2. 测量吞吐：

   ```bash
   iperf3 -c target_server -p port -t 60
   ```

3. 检查拥塞控制：

   ```bash
   ss -ti '( dport = :port )'
   # 查看 cwnd、rtt、rttvar
   ```

4. 连接池效率：

   ```cpp
   auto stats = pool.stats();
   double hit_rate = stats.total_hits / stats.total_acquires;
   // hit_rate 应 > 50%（代理场景可能更低）
   ```

**性能优化方向**：

| 问题 | 优化方案 |
|------|---------|
| 连接延迟高 | Happy Eyeballs、连接池 |
| 吞吐低 | BBR、增大缓冲区 |
| CPU 占用高 | 减少拷贝、scatter-gather |
| 内存占用高 | 缩小缓冲区、限制连接池 |

## 相关页面

- [[core/transport/reliable|reliable]] — TCP 可靠传输层实现
- [[core/connect/pool/pool|pool]] — TCP 连接池实现
- [[core/connect/dial/racer|racer]] — Happy Eyeballs 竞速器
- [[dev/debugging/tls|tls]] — TLS 协议基础
- [[dev/debugging/udp|udp]] — UDP 协议基础
- [[core/multiplex/smux/craft|smux]] — SMux 多路复用
- [[core/multiplex/yamux/craft|yamux]] — Yamux 多路复用
- [[core/instance/overview|agent]] — Prism Agent 概览
