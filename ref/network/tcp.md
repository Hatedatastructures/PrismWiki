---
title: "TCP 协议"
category: "network"
type: ref
module: ref
source: "RFC 793, RFC 1122, RFC 2581, RFC 6298, RFC 7323"
tags: [网络, tcp, 传输, 可靠传输, 连接, 流量控制, 拥塞控制]
created: 2026-05-15
updated: 2026-05-17
---

# TCP 协议

**类别**: 网络

## 概述

TCP（Transmission Control Protocol，传输控制协议）是互联网协议族中最核心的传输层协议之一，由 RFC 793 定义。TCP 提供面向连接的、可靠的、基于字节流的传输服务，是互联网可靠数据传输的基石。在代理服务器领域，TCP 是绝大多数代理协议的底层传输载体，包括 HTTP、SOCKS5、Trojan、VLESS 等协议都运行在 TCP 之上。

TCP 的设计目标是解决不可靠网络层（IP）之上的可靠传输问题。IP 协议提供的是尽力而为的交付服务，数据包可能丢失、重复、乱序或损坏。TCP 通过一系列机制在不可靠的网络层之上构建可靠的传输服务，确保数据正确、有序、无损地送达接收端。

TCP 的核心特性包括：

**面向连接**：TCP 在数据传输前必须建立连接，传输结束后需要释放连接。连接是一个逻辑概念，双方维护连接状态，包括序列号、确认号、窗口大小等信息。三次握手建立连接，四次挥手断开连接，确保双方都准备好进行数据交换。

**可靠传输**：TCP 通过序列号、确认应答、超时重传等机制保证数据可靠送达。每个字节都有序列号，接收方通过 ACK 确认收到的数据，发送方超时未收到确认则重传。校验和用于检测数据损坏。

**流量控制**：TCP 使用滑动窗口机制实现流量控制，防止发送方发送过快导致接收方缓冲区溢出。接收方通过 TCP 头部的窗口字段告知发送方当前可接收的数据量，发送方据此调整发送速率。

**拥塞控制**：TCP 实现了完整的拥塞控制算法，包括慢启动、拥塞避免、快速重传和快速恢复。这些算法动态调整发送速率，避免网络拥塞，在拥塞发生时降低发送速率，在网络空闲时提高发送速率。

**全双工通信**：TCP 连接是全双工的，双方可以同时发送和接收数据。这通过 TCP 头部的序列号和确认号字段实现，每个方向的数据流都有独立的序列号空间。

**字节流服务**：TCP 提供字节流服务，没有消息边界的概念。应用层写入的数据可能被 TCP 分割或合并发送，接收方需要自己处理消息边界问题。这与 UDP 的数据报服务形成对比，UDP 保留消息边界。

在代理服务器场景中，TCP 的这些特性至关重要。代理服务器需要在客户端和目标服务器之间转发数据，TCP 的可靠传输保证数据不会丢失，流量控制防止缓冲区溢出，拥塞控制保证网络公平性。代理服务器通常需要管理大量并发 TCP 连接，理解 TCP 的工作原理对于性能优化和问题排查非常重要。

### TCP 在代理场景的特殊考量

代理服务器在使用 TCP 时有一些特殊的考量：

**连接复用**：为了减少连接建立开销，代理服务器通常使用连接池复用 TCP 连接。这要求理解 TCP 的 TIME_WAIT 状态和连接生命周期管理。

**多路复用**：在单个 TCP 连接上复用多个逻辑流（如 HTTP/2、SMUX、Yamux）是常见做法，这需要深入理解 TCP 的字节流特性和流控机制。

**代理协议**：各种代理协议（SOCKS5、HTTP、Trojan、VLESS 等）都运行在 TCP 之上，理解 TCP 对于实现和调试这些协议至关重要。

**性能优化**：TCP 缓冲区大小、Nagle 算法、延迟 ACK 等参数对代理性能有显著影响，需要根据场景调优。

**加密传输**：TLS/SSL 运行在 TCP 之上，代理服务器需要处理 TLS 握手和加密数据传输。

## 原理详解

### TCP 报文结构

TCP 报文由 TCP 头部和数据部分组成。TCP 头部通常为 20 字节（不含选项），格式如下：

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
|  Data Offset | Res |U|A|P|R|S|F|            Window           |
|              |     |R|C|S|S|Y|I|                              |
|              |     |G|K|H|T|N|N|                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**源端口和目的端口**：各 16 位，标识发送端和接收端的应用进程。端口号与 IP 地址组成套接字，唯一标识一个连接端点。

**序列号**：32 位，标识本报文段数据的第一个字节的序号。TCP 是字节流协议，每个字节都有序列号。序列号空间为 2^32，循环使用。

**确认号**：32 位，期望收到的下一个字节的序列号。确认号之前的所有字节都已正确接收。ACK 标志位为 1 时确认号才有效。

**数据偏移**：4 位，TCP 头部长度，单位是 32 位字。最小值为 5（20 字节），最大值为 15（60 字节），选项部分最多 40 字节。

**标志位**：6 位，控制 TCP 连接状态
- URG：紧急指针有效
- ACK：确认号有效，连接建立后所有报文的 ACK 都置 1
- PSH：接收方应尽快将数据推送给应用层
- RST：重置连接，用于异常情况
- SYN：同步序列号，用于建立连接
- FIN：结束连接，用于断开连接

**窗口大小**：16 位，接收窗口大小，用于流量控制。告诉发送方当前可接收的数据量（字节）。窗口缩放选项可扩展此字段。

**校验和**：16 位，覆盖 TCP 头部、数据和伪头部。伪头部包含源 IP、目的 IP、协议号和 TCP 长度，用于检测 IP 层路由错误。

**紧急指针**：16 位，URG 标志为 1 时有效，指向紧急数据末尾。

**选项**：可变长度，用于扩展功能。常见选项包括：

| 选项类型 | 长度 | 说明 |
|---------|------|------|
| 0 | 1 | 选项表结束 |
| 1 | 1 | 无操作（填充） |
| 2 | 4 | 最大段大小（MSS） |
| 3 | 3 | 窗口缩放因子 |
| 4 | 2 | 选择性确认允许（SACK Permitted） |
| 5 | 可变 | 选择性确认（SACK） |
| 8 | 10 | 时间戳 |

**MSS 选项**：通告本端可接收的最大 TCP 段大小。通常设置为 MTU - 40（IP 头 20 字节 + TCP 头 20 字节）。以太网 MTU 为 1500，故 MSS 通常为 1460。

**窗口缩放选项**：扩展窗口大小。窗口字段 16 位最大 65535 字节，通过窗口缩放因子可将窗口扩展到 1GB 以上。缩放因子在 SYN 握手时协商。

**时间戳选项**：包含发送时间戳（TSval）和回显时间戳（TSecr），用于精确计算 RTT 和防止序列号回绕（PAWS）。

**SACK 选项**：允许接收方告知发送方已收到的不连续数据块，发送方只需重传丢失的数据，提高重传效率。

### 三次握手

TCP 使用三次握手建立连接，确保双方都准备好进行数据传输。

```
      Client                                              Server
        |                                                    |
        |         1. SYN (seq=x)                             |
        |--------------------------------------------------->|
        |                                                    |
        |   2. SYN-ACK (seq=y, ack=x+1)                      |
        |<---------------------------------------------------|
        |                                                    |
        |         3. ACK (seq=x+1, ack=y+1)                  |
        |--------------------------------------------------->|
        |                                                    |
        |            连接建立完成                            |
        |<===================================================>|
```

**第一步（SYN）**：客户端发送 SYN 报文，携带初始序列号 ISN（Initial Sequence Number）x。客户端进入 SYN_SENT 状态。SYN 报文不携带数据，但消耗一个序列号。

**第二步（SYN-ACK）**：服务端收到 SYN，回复 SYN-ACK 报文。确认号为 x+1（确认客户端 SYN），同时携带自己的初始序列号 y。服务端进入 SYN_RCVD 状态。此报文同样消耗一个序列号。

**第三步（ACK）**：客户端收到 SYN-ACK，发送 ACK 报文确认服务端的 SYN（ack=y+1）。客户端进入 ESTABLISHED 状态。服务端收到 ACK 后也进入 ESTABLISHED 状态。此时连接建立完成，可以开始数据传输。

**为什么是三次握手**：
1. 防止历史连接：如果只有两次握手，服务端无法判断收到的 SYN 是否是历史延迟报文
2. 同步序列号：双方需要确认对方的初始序列号
3. 防止 SYN 洪水攻击：三次握手使服务端在收到 ACK 前不分配过多资源

**初始序列号（ISN）的选择**：
ISN 不是从 0 或 1 开始，而是基于时钟的伪随机数，每 4 微秒增加 1。这样做是为了：
1. 防止新连接的序列号与旧连接重叠
2. 增加安全性，防止序列号预测攻击
3. RFC 793 建议时钟驱动 ISN，现代实现还加入随机因素

**SYN 队列与 Accept 队列**：
服务端维护两个队列处理连接请求：
1. SYN 队列（半连接队列）：存放收到 SYN 但未完成三次握手的连接
2. Accept 队列（全连接队列）：存放已完成三次握手等待应用 accept 的连接

队列满时行为：
- SYN 队列满：丢弃新 SYN 或不发送 SYN-ACK
- Accept 队列满：取决于 tcp_abort_on_overflow 参数

### 四次挥手

TCP 使用四次挥手断开连接，因为 TCP 是全双工协议，每个方向需要单独关闭。

```
      Client                                              Server
        |                                                    |
        |         1. FIN (seq=x)                             |
        |--------------------------------------------------->|
        |                                                    |
        |   2. ACK (ack=x+1)                                 |
        |<---------------------------------------------------|
        |                                                    |
        |            客户端 -> 服务端方向关闭                |
        |                                                    |
        |   3. FIN (seq=y)                                   |
        |<---------------------------------------------------|
        |                                                    |
        |         4. ACK (ack=y+1)                           |
        |--------------------------------------------------->|
        |                                                    |
        |            服务端 -> 客户端方向关闭                |
        |                                                    |
```

**第一步（FIN）**：主动关闭方发送 FIN 报文，表示不再发送数据。进入 FIN_WAIT_1 状态。FIN 报文消耗一个序列号。

**第二步（ACK）**：被动关闭方收到 FIN，回复 ACK。被动关闭方进入 CLOSE_WAIT 状态，主动关闭方进入 FIN_WAIT_2 状态。此时被动关闭方仍可发送数据。

**第三步（FIN）**：被动关闭方发送完剩余数据后，发送自己的 FIN 报文，进入 LAST_ACK 状态。

**第四步（ACK）**：主动关闭方收到 FIN，回复 ACK，进入 TIME_WAIT 状态。被动关闭方收到 ACK 后进入 CLOSED 状态。

**TIME_WAIT 状态**：
主动关闭方在发送最后一个 ACK 后进入 TIME_WAIT 状态，等待 2MSL（Maximum Segment Lifetime，报文最大生存时间，通常 60 秒）。TIME_WAIT 存在的原因：

1. 确保最后的 ACK 到达：如果 ACK 丢失，被动关闭方会重发 FIN，TIME_WAIT 允许重发 ACK
2. 让旧连接的报文消亡：防止新连接收到旧连接的延迟报文

TIME_WAIT 过多会导致端口耗尽，解决方案：
- 使用连接池复用连接
- 设置 SO_REUSEADDR 选项
- 调整 tcp_tw_reuse 参数（Linux）

### 可靠传输机制

TCP 通过多种机制实现可靠传输：

**序列号与确认应答**：
每个字节都有序列号，接收方通过 ACK 确认连续收到的最高序列号（累积确认）。例如 ACK=1001 表示序列号 1000 及之前的所有字节都已收到。

**超时重传（RTO）**：
发送方启动定时器，超时未收到 ACK 则重传。RTO（Retransmission Timeout）基于 RTT（Round-Trip Time）动态计算。

RTT 估计算法：
```
SRTT = (1 - α) * SRTT + α * RTT_sample    // α = 1/8
RTTVAR = (1 - β) * RTTVAR + β * |RTT_sample - SRTT|    // β = 1/4
RTO = SRTT + 4 * RTTVAR
```

Karn 算法：重传报文不用于 RTT 估计，因为无法区分 ACK 是针对原报文还是重传报文。

**快速重传**：
收到 3 个重复 ACK（共 4 个相同 ACK）时，不必等待超时就立即重传。这比超时重传更快，因为重复 ACK 说明后续数据已到达，只是中间有数据丢失。

**选择性重传（SACK）**：
传统 TCP 使用 Go-Back-N，丢失一个包后重传该包及之后所有包。SACK 允许接收方告知已收到的非连续数据块，发送方只重传真正丢失的数据。

### 流量控制

流量控制防止发送方淹没接收方，基于滑动窗口实现。

**滑动窗口**：
接收方在 ACK 中通告窗口大小（rwnd），表示接收缓冲区剩余空间。发送方维护发送窗口，保证未确认数据量不超过 rwnd。

窗口关闭：当 rwnd=0 时，发送方停止发送，但发送零窗口探测（Zero Window Probe），检测窗口何时打开。

窗口更新：接收方在缓冲区有空间时发送窗口更新（Window Update），不含数据的 ACK，仅更新窗口值。

### 拥塞控制

拥塞控制防止发送方淹没网络，独立于流量控制。

**慢启动**：
连接初始时，cwnd（拥塞窗口）从 1 MSS 开始，每收到一个 ACK，cwnd 加倍（指数增长）。直到 cwnd 达到 ssthresh（慢启动阈值），进入拥塞避免。

**拥塞避免**：
每 RTT 增加 1 MSS（线性增长），更温和地探测可用带宽。

**快速重传与快速恢复**：
收到 3 个重复 ACK 时，认为发生丢包：
1. ssthresh = cwnd / 2
2. 快速重传丢失的段
3. cwnd = ssthresh + 3 MSS（进入快速恢复）
4. 收到新 ACK 时，cwnd = ssthresh（退出快速恢复）

现代拥塞控制算法（CUBIC、BBR）有更复杂的实现，但基本原理相同。

### TCP 状态机

TCP 连接有完整的状态机：

| 状态 | 说明 |
|------|------|
| CLOSED | 连接关闭 |
| LISTEN | 服务端监听状态 |
| SYN_SENT | 已发送 SYN，等待 SYN-ACK |
| SYN_RCVD | 已收到 SYN，已发送 SYN-ACK，等待 ACK |
| ESTABLISHED | 连接建立，可传输数据 |
| FIN_WAIT_1 | 已发送 FIN，等待 ACK |
| FIN_WAIT_2 | 已收到 FIN 的 ACK，等待对方 FIN |
| CLOSE_WAIT | 收到 FIN，已发送 ACK，等待应用关闭 |
| LAST_ACK | 已发送 FIN，等待 ACK |
| TIME_WAIT | 主动关闭方等待 2MSL |

状态转换图：
```
                              +---------+
                              | CLOSED  |
                              +---------+
                                   |
                                   | passive open / SYN
                                   v
                              +---------+
                              | LISTEN  |
                              +---------+
                                   |
                                   | SYN / ACK
                                   v
                              +---------+
                              | SYN_RCVD|
                              +---------+
                                   |
                                   | ACK
                                   v
+----------+                  +----------+
|SYN_SENT  |----------------->|ESTABLISHED|
+----------+                  +----------+
     |                              |
     | SYN                          | FIN
     v                              v
+----------+                  +----------+
|ESTABLISHED|<----------------|FIN_WAIT_1|
+----------+                  +----------+
     |                              |
     | FIN                          | ACK
     v                              v
+----------+                  +----------+
|CLOSE_WAIT|                  |FIN_WAIT_2|
+----------+                  +----------+
     |                              |
     | close                        | FIN
     v                              v
+----------+                  +----------+
|LAST_ACK  |<----------------|TIME_WAIT |
+----------+                  +----------+
     |                              |
     | ACK                          | 2MSL
     v                              v
+----------+                  +---------+
| CLOSED   |<-----------------| CLOSED  |
+----------+                  +---------+
```

### TCP 选项与扩展

**最大段大小（MSS）**：
TCP 载荷的最大大小。TCP 层分段，IP 层分片。建议 MSS 设置为 MTU - 40，避免 IP 分片。路径 MTU 发现（PMTUD）可动态确定路径最小 MTU。

**窗口缩放（Window Scaling）**：
将 16 位窗口扩展到 30 位（1GB）。缩放因子在 SYN 握手时协商，左移窗口值。高延迟网络需要大窗口才能充分利用带宽。

**选择性确认（SACK）**：
允许接收方告知已收到的非连续数据块。发送方 SACK 选项中包含已收到的数据块范围，避免不必要的重传。

**时间戳选项**：
用于 RTT 测量和 PAWS（Protection Against Wrapped Sequence Numbers）。时间戳在 SYN 握手时协商。

**TCP Fast Open（TFO）**：
允许在 SYN 中携带数据，减少 RTT。需要双方支持且之前建立过连接（使用 Cookie）。

## 在 Prism 中的应用

Prism 作为代理服务器，TCP 是其核心传输协议。以下是 TCP 相关的关键应用点：

### 连接管理

**监听器（Listener）**：
`agent/front/listener` 负责监听 TCP 端口，接受入站连接。使用 `net::ip::tcp::acceptor` 进行异步接受连接，通过负载均衡器分发到 worker 线程。

```cpp
// 接受连接的核心流程
auto socket = co_await acceptor.async_accept(net::use_awaitable);
co_await balancer.dispatch(std::move(socket));
```

**连接池（Connection Pool）**：
`channel/connection/pool` 实现 TCP 连接池，复用上游连接。连接池管理策略：
- 每端点最大缓存连接数
- 空闲连接超时清理
- 连接健康检查

```cpp
// 连接池配置
struct pool_config {
    std::size_t max_cache_per_endpoint{32};  // 每端点最大缓存
    std::size_t max_idle_seconds{60};         // 空闲超时
    std::size_t connect_timeout_ms{5000};     // 连接超时
};
```

**多路复用（Multiplex）**：
`multiplex/duct` 在单个 TCP 连接上复用多个流。支持 SMUX 和 Yamux 协议：
- SMUX：轻量级，适合低延迟场景
- Yamux：功能完整，支持流控和 ping

### 协议实现

**SOCKS5**：
`protocol/socks5` 在 TCP 上实现 SOCKS5 协议：
- 握手阶段：认证协商
- 请求阶段：CONNECT/UDP ASSOCIATE
- 数据阶段：透明转发

**HTTP 代理**：
`protocol/http` 实现 HTTP CONNECT 隧道和普通代理：
- CONNECT 方法建立 TCP 隧道
- 普通代理解析 HTTP 请求，建立上游连接

**Trojan**：
`protocol/trojan` 实现 Trojan 协议：
- TLS + 自定义协议
- 密码认证
- TCP/UDP 支持

### 性能优化

**缓冲区配置**：
```json
{
    "pool": {
        "recv_buffer_size": 65536,
        "send_buffer_size": 65536
    }
}
```

**TCP_NODELAY**：
禁用 Nagle 算法，减少小包延迟。代理场景建议开启。

**SO_REUSEADDR/SO_REUSEPORT**：
允许端口复用，支持多进程监听同一端口。

**TCP 快速打开**：
减少握手 RTT，适用于频繁建立连接的场景。

### 相关模块链接

| 模块 | 功能 | 文档 |
|------|------|------|
| listener | TCP 监听 | [[core/agent/front/listener|listener]] |
| connection_pool | 连接池 | [[core/channel/connection/pool|connection-pool]] |
| duct | TCP 流多路复用 | [[core/multiplex/duct|duct]] |
| parcel | UDP 数据包 | [[core/multiplex/parcel|parcel]] |
| tunnel | 双向转发 | [[core/pipeline/tunnel|tunnel]] |

## 最佳实践

### 连接管理

**连接池配置**：
- 根据并发量设置合适的池大小
- 空闲超时设置为 30-120 秒，平衡资源占用和复用效率
- 连接超时设置 3-10 秒，避免长时间等待

**避免连接泄漏**：
- 使用 RAII 管理 socket 生命周期
- 异常路径正确关闭连接
- 定期检查连接池健康状态

**优雅关闭**：
- 先发送缓冲区数据再关闭
- 使用 shutdown(SHUT_WR) 半关闭
- 设置合理的 linger 时间

### 性能调优

**缓冲区大小**：
- 根据带宽延迟积（BDP）调整缓冲区
- 高延迟高带宽链路需要大缓冲区
- 使用动态缓冲区适应网络变化

**禁用 Nagle 算法**：
- 代理场景通常禁用 Nagle（TCP_NODELAY）
- 延迟敏感应用必须禁用
- 批量传输场景可保持启用

**调整 keepalive**：
- 检测死连接，释放资源
- 默认值通常太长，建议调整
- 应用层心跳更可靠

### 安全考虑

**序列号随机化**：
- 使用安全的 ISN 生成算法
- 防止序列号预测攻击

**连接限制**：
- 限制每个客户端的连接数
- 防止资源耗尽攻击
- 使用 SYN Cookie 防御 SYN 洪水

**加密传输**：
- 使用 TLS 保护数据
- 代理协议认证
- 防止中间人攻击

## 常见问题

### Q1: 为什么连接会卡在 SYN_SENT 状态？

**原因**：
1. 服务端未运行或防火墙丢弃 SYN
2. 网络不通或路由问题
3. 服务端 SYN 队列满

**排查**：
```bash
# 查看连接状态
netstat -an | grep SYN_SENT

# 检查防火墙规则
iptables -L -n

# 抓包分析
tcpdump -i eth0 tcp and host <server_ip>
```

### Q2: 为什么会有大量 TIME_WAIT 连接？

**原因**：
1. 短连接频繁创建销毁
2. 主动关闭方会进入 TIME_WAIT
3. 高并发场景端口复用压力大

**解决**：
- 使用连接池复用连接
- 开启 tcp_tw_reuse（Linux）
- 调整 tcp_max_tw_buckets

### Q3: 为什么数据传输速度上不去？

**可能原因**：
1. 滑动窗口过小，限制吞吐量
2. 拥塞控制过于保守
3. 缓冲区设置不当
4. Nagle 算法与延迟 ACK 交互问题

**排查**：
```bash
# 查看连接参数
ss -ti | grep -A 10 <port>

# 检查窗口大小
netstat -an | grep <port>
```

### Q4: 什么是粘包问题？

**现象**：多次 send 的数据被合并接收，或一次 send 的数据被分多次接收。

**原因**：TCP 是字节流协议，不保留消息边界。

**解决**：
- 固定长度消息
- 长度前缀协议
- 分隔符协议
- 应用层协议（如 HTTP、WebSocket）

### Q5: 如何处理 RST 报文？

**常见原因**：
1. 连接不存在（端口未监听）
2. 异常终止连接
3. 接收缓冲区溢出
4. 保活超时

**处理方式**：
- 正确处理连接关闭事件
- 重启连接或重试
- 记录日志便于排查

## 排障指南

### 连接问题诊断

**步骤 1：网络连通性检查**
```bash
# Ping 检查基本连通性
ping <host>

# 检查端口可达性
nc -zv <host> <port>

# 路由追踪
traceroute <host>
```

**步骤 2：抓包分析**
```bash
# 抓取 TCP 流量
tcpdump -i any tcp and host <host> and port <port> -w capture.pcap

# 使用 Wireshark 分析
wireshark capture.pcap
```

**步骤 3：连接状态检查**
```bash
# 查看所有 TCP 连接状态
netstat -ant

# 使用 ss 命令（更高效）
ss -ant

# 查看特定端口的连接
ss -ant 'sport = :<port> or dport = :<port>'
```

### 性能问题诊断

**步骤 1：检查连接统计**
```bash
# 查看连接状态分布
netstat -ant | awk '{print $6}' | sort | uniq -c

# 查看 TIME_WAIT 数量
netstat -ant | grep TIME_WAIT | wc -l
```

**步骤 2：检查缓冲区使用**
```bash
# 查看缓冲区配置
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem

# 查看当前缓冲区使用
ss -tm
```

**步骤 3：检查拥塞控制**
```bash
# 查看拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

# 查看连接的拥塞参数
ss -ti
```

### Prism 特定排障

**日志分析**：
查看 Prism 日志中的连接相关错误：
```bash
# 查找连接错误
grep -i "connection" /var/log/prism/trace.log
grep -i "timeout" /var/log/prism/trace.log
```

**配置检查**：
1. 检查监听端口是否正确配置
2. 检查上游连接超时设置
3. 检查连接池参数是否合理

**监控指标**：
- 活跃连接数
- 连接建立成功率
- 平均连接延迟
- 连接池命中率

## 参考资料

- [RFC 793 - Transmission Control Protocol](https://tools.ietf.org/html/rfc793)
- [RFC 1122 - Requirements for Internet Hosts](https://tools.ietf.org/html/rfc1122)
- [RFC 2581 - TCP Congestion Control](https://tools.ietf.org/html/rfc2581)
- [RFC 6298 - Computing TCP's Retransmission Timer](https://tools.ietf.org/html/rfc6298)
- [RFC 7323 - TCP Extensions for High Performance](https://tools.ietf.org/html/rfc7323)

## 相关知识

- [[ref/network/udp|UDP]] — 用户数据报协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞争算法
- [[ref/network/connection-pool|连接池]] — 连接复用技术
- [[core/channel/connection/pool|连接池实现]] — Prism 连接池模块
- [[core/multiplex/duct|Duct]] — TCP 流多路复用