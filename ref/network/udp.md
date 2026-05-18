---
title: "UDP 协议"
category: "network"
type: ref
module: ref
source: "RFC 768, RFC 8085"
tags: [网络, udp, 传输, 无连接, 低延迟, 数据报, dns]
created: 2026-05-15
updated: 2026-05-17
---

# UDP 协议

**类别**: 网络

## 概述

UDP（User Datagram Protocol，用户数据报协议）是互联网协议族中的无连接传输层协议，由 RFC 768 定义。与 TCP 相比，UDP 提供的是无连接的、不可靠的数据报服务，具有开销小、延迟低的特点。在代理服务器领域，UDP 主要用于 DNS 查询、QUIC 协议、实时音视频传输和游戏等对延迟敏感的场景。

UDP 的设计哲学是"简单即是美"。它不提供 TCP 那样复杂的可靠传输机制，而是将可靠性问题留给应用层处理。这种设计使得 UDP 成为一个轻量级的传输协议，头部仅 8 字节，相比 TCP 的 20+ 字节显著减少开销。UDP 不会因为重传、流量控制或拥塞控制而引入延迟，这使得它非常适合实时应用。

UDP 的核心特性包括：

**无连接**：UDP 不需要建立连接就可以发送数据。发送方只需知道接收方的 IP 地址和端口号即可发送数据报。没有三次握手、没有连接状态维护，这使得 UDP 的开销极低。每个数据报都是独立的，发送方不需要维护接收方的状态信息。

**不可靠传输**：UDP 不保证数据报的送达、顺序或唯一性。数据报可能丢失、重复、乱序到达，应用层需要自行处理这些问题。虽然这看起来是个缺点，但对于某些应用来说反而是优势：实时音视频可以容忍少量丢包，但无法容忍重传带来的延迟。

**消息边界保留**：与 TCP 的字节流不同，UDP 保留消息边界。应用层写入的数据报作为一个整体发送，接收方一次读取一个完整的数据报。这简化了应用层协议的设计，不需要处理粘包问题。

**低开销**：UDP 头部仅 8 字节，包含源端口、目的端口、长度和校验和四个字段。相比 TCP 的复杂状态机，UDP 的处理开销极低，适合高吞吐量场景。

**支持广播和组播**：UDP 支持一对多的通信模式，可以向广播地址或组播组发送数据。这使得 UDP 非常适合流媒体分发、服务发现等场景。

**无拥塞控制**：UDP 不实现拥塞控制，发送方可以以任意速率发送数据。这虽然可能导致网络拥塞，但对于可控环境下的应用提供了更大的灵活性。

在代理服务器场景中，UDP 有几个重要的应用：

**DNS 查询**：DNS 协议主要使用 UDP，单个查询响应通常在一个数据报内完成。UDP 的低延迟特性使得 DNS 查询快速完成。Prism 的 DNS 解析模块使用 UDP 作为主要传输协议。

**QUIC 协议**：QUIC 是基于 UDP 实现的可靠传输协议，结合了 TCP 的可靠性和 UDP 的灵活性。HTTP/3 基于 QUIC 实现，是现代 Web 的重要发展方向。

**UDP 代理**：SOCKS5 支持 UDP ASSOCIATE 命令，允许代理 UDP 流量。Trojan 协议也支持 UDP over TCP 的转发模式。

**实时通信**：WebRTC、VoIP 等实时通信应用使用 UDP 传输音视频数据，容忍少量丢包以换取低延迟。

### UDP 与 TCP 的对比

| 特性 | UDP | TCP |
|------|-----|-----|
| 连接 | 无连接 | 面向连接 |
| 可靠性 | 不保证 | 保证 |
| 顺序 | 不保证 | 保证 |
| 头部大小 | 8 字节 | 20+ 字节 |
| 流量控制 | 无 | 滑动窗口 |
| 拥塞控制 | 无 | 完整算法 |
| 消息边界 | 保留 | 不保留 |
| 广播/组播 | 支持 | 不支持 |
| 典型应用 | DNS、流媒体、游戏 | Web、文件传输、邮件 |

## 原理详解

### UDP 报文结构

UDP 报文由 UDP 头部和数据部分组成。UDP 头部固定 8 字节，格式如下：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**源端口（Source Port）**：16 位，发送方的端口号。可选字段，如果不需要回复可以置 0。在请求-响应模式中，源端口用于接收方回复。

**目的端口（Destination Port）**：16 位，接收方的端口号。必需字段，用于将数据报分发给正确的应用进程。

**长度（Length）**：16 位，UDP 报文的总长度，包括头部和数据。最小值为 8（仅头部，无数据），最大值为 65535。受限于 IP 层的 MTU，实际有效载荷通常小于 MTU - 28（IP 头 20 字节 + UDP 头 8 字节）。

**校验和（Checksum）**：16 位，覆盖 UDP 头部、数据和伪头部。校验和是可选的，IPv4 中可以置 0 表示不使用校验和，但 IPv6 强制要求使用校验和。伪头部包含源 IP、目的 IP、协议号和 UDP 长度，用于检测 IP 层路由错误。

**伪头部结构**：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Zero  | Protocol  |          UDP Length                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

伪头部不实际传输，仅用于校验和计算。它确保数据报到达正确的目的地，即使 IP 头部被篡改，校验和也会失败。

### UDP 数据报大小

理论最大 UDP 数据报大小为 65535 字节（IP 层限制），但实际有效载荷受限于 MTU。

**MTU 限制**：
以太网 MTU 通常为 1500 字节，减去 IP 头 20 字节和 UDP 头 8 字节，UDP 有效载荷最大约 1472 字节。超过 MTU 的 UDP 数据报会被 IP 层分片。

**IP 分片问题**：
- 分片增加延迟和开销
- 任一分片丢失，整个数据报丢弃
- 某些网络设备禁止分片或丢弃分片
- Path MTU Discovery 可以发现路径最小 MTU

**推荐做法**：
- 应用层限制 UDP 数据报大小在 512-1400 字节
- DNS 传统限制为 512 字节
- 使用 Path MTU Discovery 发现 MTU
- 实现分片重组或避免分片

### UDP 端口与多路复用

UDP 使用端口号实现应用层多路复用。端口号范围：

| 端口范围 | 用途 |
|---------|------|
| 0-1023 | 知名端口，需要特权 |
| 1024-49151 | 注册端口，IANA 分配 |
| 49152-65535 | 动态/私有端口，临时使用 |

**知名 UDP 端口**：
- 53：DNS（Domain Name System）
- 67/68：DHCP（Dynamic Host Configuration Protocol）
- 69：TFTP（Trivial File Transfer Protocol）
- 123：NTP（Network Time Protocol）
- 161：SNMP（Simple Network Management Protocol）
- 443：QUIC/HTTP/3

与 TCP 不同，UDP 端口与 TCP 端口独立。同一个端口号可以同时被 UDP 和 TCP 使用，例如 DNS 同时监听 UDP 和 TCP 的 53 端口。

### UDP 的可靠性考量

虽然 UDP 本身不提供可靠性，但应用层可以实现各种可靠性机制：

**确认与重传**：
应用层实现 ACK 机制，发送方超时重传。TFTP 使用简单的停等协议（Stop-and-Wait），每个数据包等待确认。

**序列号**：
为每个数据报分配序列号，接收方检测丢失和乱序。RTP（Real-time Transport Protocol）使用序列号检测丢包。

**前向纠错（FEC）**：
添加冗余数据，接收方可以恢复一定数量的丢失数据。适用于实时流媒体，不能容忍重传延迟。

**拥塞控制**：
应用层实现拥塞控制，避免淹没网络。QUIC 在 UDP 之上实现了完整的 TCP 拥塞控制算法。

**应用层示例**：

DNS 使用简单的请求-响应模式：
- 客户端发送查询，等待响应
- 超时重试（通常 2-3 次）
- 响应匹配查询 ID

QUIC 实现了类似 TCP 的可靠性：
- 序列号和确认机制
- 快速重传和选择性确认
- 拥塞控制（CUBIC、BBR）
- 连接迁移支持

### UDP 在 NAT 环境

NAT（网络地址转换）对 UDP 有特殊影响：

**NAT 映射**：
NAT 设备为 UDP 创建映射表，记录内部地址端口与外部地址端口的对应关系。与 TCP 不同，UDP 没有连接状态，NAT 映射基于超时过期。

**NAT 超时**：
UDP 映射超时通常较短（30-120 秒），需要定期发送保活包维持映射。某些 NAT 设备对 UDP 支持不完善，可能导致映射提前过期。

**NAT 穿透**：
- STUN：获取 NAT 映射后的公网地址
- TURN：通过中继服务器转发
- ICE：综合使用多种穿透技术
- UDP Hole Punching：利用 NAT 的映射机制实现 P2P

**对称 NAT 问题**：
对称 NAT 为每个目的地址创建不同的映射，导致 UDP Hole Punching 失效。需要使用 TURN 中继。

### UDP 缓冲区

UDP 使用接收缓冲区存储到达的数据报：

**接收缓冲区**：
内核为每个 UDP socket 维护接收缓冲区。数据报到达时放入缓冲区，应用层读取时从缓冲区取出。缓冲区满时，新数据报被丢弃，不发送任何通知。

**缓冲区溢出**：
UDP 没有流量控制，发送方可能淹没接收方。症状包括：
- 数据包丢失
- 高丢包率
- 性能下降

**调优方法**：
```bash
# 查看当前缓冲区大小
sysctl net.core.rmem_max net.core.rmem_default

# 调整缓冲区大小
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.rmem_default=26214400
```

**监控丢包**：
```bash
# 查看 UDP 统计
netstat -su | grep -i udp
cat /proc/net/snmp | grep Udp
```

### UDP 与安全性

UDP 的无连接特性带来安全挑战：

**反射放大攻击**：
攻击者伪造源 IP 发送 UDP 请求，服务器响应发送到受害者。NTP、DNS、SSDP 等服务可能被滥用。

**UDP 洪水**：
大量 UDP 数据报淹没目标，消耗带宽和处理能力。

**安全实践**：
- 限制响应速率
- 验证请求源
- 使用较小的响应
- 实现 BCP 38 入口过滤

**DTLS**：
UDP 版本的 TLS，提供加密和认证。QUIC 内置加密，不依赖 DTLS。

## 在 Prism 中的应用

Prism 在多个场景使用 UDP，主要涉及 DNS 查询、UDP 代理和多路复用协议。

### DNS 解析

Prism 的 DNS 解析模块主要使用 UDP：

```cpp
// DNS 查询流程
auto query = build_dns_query(domain, type);
co_await udp_socket.async_send_to(query, resolver_endpoint);
co_await udp_socket.async_receive_from(response, sender_endpoint);
```

**UDP DNS 配置**：
```json
{
    "dns": {
        "servers": [
            {
                "address": "8.8.8.8",
                "port": 53,
                "protocol": "udp"
            }
        ],
        "timeout_ms": 2000
    }
}
```

**DNS over UDP 特点**：
- 快速响应，通常一个 RTT 完成
- 限制响应大小 512 字节（传统）
- 超过限制时回退到 TCP
- EDNS0 扩展支持更大响应

### UDP 代理

Prism 支持 SOCKS5 UDP ASSOCIATE 和 Trojan UDP 转发：

**SOCKS5 UDP 关联**：
```
客户端 <---> 代理服务器 <---> 目标服务器
  UDP            UDP
```

SOCKS5 UDP 代理流程：
1. 客户端发送 UDP ASSOCIATE 请求
2. 服务器返回 UDP 中继地址
3. 客户端通过中继地址发送 UDP 数据报
4. 服务器转发到目标地址

**Trojan UDP**：
Trojan 协议将 UDP 数据封装在 TCP 流中传输：
```
UDP 数据 → 封装 → TCP 流 → 解封装 → UDP 数据
```

### 多路复用

Prism 的多路复用协议支持 UDP 流：

**Parcel（UDP 数据包）**：
`multiplex/parcel` 模块处理多路复用中的 UDP 数据：
- UDP 数据包封装
- 会话管理
- 超时处理

```cpp
// UDP 数据包封装
struct parcel_frame {
    uint32_t session_id;
    uint16_t length;
    std::vector<uint8_t> data;
};
```

### 相关模块链接

| 模块 | 功能 | 文档 |
|------|------|------|
| parcel | UDP 数据包多路复用 | [[core/multiplex/parcel|parcel]] |
| udp_relay | UDP 中继 | [[core/protocol/common/udp_relay|udp_relay]] |
| upstream | DNS 上游查询 | [[core/resolve/dns/upstream|upstream]] |
| duct | TCP 流多路复用 | [[core/multiplex/duct|duct]] |

## 最佳实践

### 数据报大小

**推荐大小**：
- 默认限制 512 字节（DNS 传统限制）
- 局域网可用 1400 字节
- 跨公网建议 512-1024 字节
- 避免分片

**MTU 发现**：
```cpp
// 实现 Path MTU Discovery
// 使用 IP_MTU_DISCOVER 选项
int val = IP_PMTUDISC_DO;
setsockopt(sock, IPPROTO_IP, IP_MTU_DISCOVER, &val, sizeof(val));
```

### 超时与重传

**推荐超时**：
- DNS 查询：2-5 秒
- 实时应用：根据 RTT 动态调整
- 重传次数：2-3 次

**指数退避**：
```cpp
// 重传超时指数退避
int timeout = base_timeout;
for (int retry = 0; retry < max_retries; ++retry) {
    co_await send_and_wait(timeout);
    timeout *= 2;  // 指数退避
}
```

### 缓冲区管理

**调整接收缓冲区**：
```cpp
// 设置接收缓冲区大小
int rcvbuf = 256 * 1024;  // 256 KB
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
```

**避免缓冲区溢出**：
- 监控丢包统计
- 根据流量调整缓冲区
- 实现背压机制

### NAT 穿透

**保活机制**：
```cpp
// 定期发送保活包维持 NAT 映射
net::steady_timer timer(io_context);
while (running) {
    timer.expires_after(std::chrono::seconds(15));
    co_await timer.async_wait(net::use_awaitable);
    co_await send_keepalive();
}
```

**推荐保活间隔**：
- 一般 NAT：30-60 秒
- 对称 NAT：15-30 秒
- 考虑最保守的超时时间

## 常见问题

### Q1: 为什么 UDP 数据报会丢失？

**原因**：
1. 网络拥塞导致路由器丢包
2. 接收缓冲区溢出
3. NAT 映射超时
4. MTU 问题导致分片丢失
5. 校验和错误

**排查方法**：
```bash
# 查看丢包统计
netstat -su | grep "packet receive errors"
cat /proc/net/snmp | grep Udp

# 抓包分析
tcpdump -i eth0 udp and port <port>
```

### Q2: UDP 和 TCP 可以绑定同一个端口吗？

**答案**：可以。UDP 和 TCP 端口空间独立，同一端口可以同时被 UDP 和 TCP socket 使用。例如 DNS 服务通常同时监听 UDP 和 TCP 的 53 端口。

```cpp
// TCP socket
net::ip::tcp::acceptor tcp_acceptor(io_context, {net::ip::tcp::v4(), 53});

// UDP socket
net::ip::udp::socket udp_socket(io_context, {net::ip::udp::v4(), 53});
```

### Q3: 如何实现 UDP 可靠传输？

**方案**：
1. 使用 QUIC：在 UDP 之上实现完整可靠性
2. 使用 RUDP：可靠 UDP 实现
3. 应用层实现：添加序列号、确认、重传
4. 使用 KCP：快速可靠的 ARQ 协议

### Q4: UDP 适合什么场景？

**适合 UDP**：
- 实时音视频（容忍丢包，要求低延迟）
- 游戏（实时同步）
- DNS 查询（快速响应）
- 服务发现（广播/组播）
- 物联网（资源受限设备）

**不适合 UDP**：
- 文件传输（需要可靠性）
- Web 浏览（传统 HTTP）
- 邮件传输（需要可靠性）
- 数据库同步（数据完整性）

### Q5: 如何监控 UDP 性能？

**关键指标**：
- 丢包率
- 延迟
- 抖动（延迟变化）
- 吞吐量

**监控命令**：
```bash
# 查看 UDP 统计
cat /proc/net/snmp | grep Udp

# 实时监控
watch -n 1 'cat /proc/net/snmp | grep Udp'

# 使用 ss 命令
ss -ua  # UDP sockets
ss -um  # UDP 内存使用
```

## 排障指南

### 连接问题诊断

**步骤 1：检查端口监听**
```bash
# 查看 UDP 端口监听
netstat -lun
ss -ul

# 检查特定端口
netstat -lun | grep <port>
```

**步骤 2：测试连通性**
```bash
# 使用 nc 发送 UDP 数据
echo "test" | nc -u <host> <port>

# 使用 dig 测试 DNS
dig @<server> <domain>
```

**步骤 3：抓包分析**
```bash
# 抓取 UDP 流量
tcpdump -i any udp and port <port>

# 保存到文件
tcpdump -i any udp and port <port> -w udp.pcap
```

### 性能问题诊断

**步骤 1：检查丢包**
```bash
# 查看丢包统计
netstat -su | grep -E "packet receive errors|buffer errors"

# 持续监控
while true; do
    cat /proc/net/snmp | grep Udp
    sleep 1
done
```

**步骤 2：检查缓冲区**
```bash
# 查看缓冲区大小
sysctl net.core.rmem_max net.core.rmem_default
sysctl net.core.wmem_max net.core.wmem_default

# 查看当前使用
ss -um
```

**步骤 3：检查 MTU 问题**
```bash
# 查看 MTU
ip link show

# 测试 MTU
ping -M do -s 1472 <host>  # 1472 + 28 = 1500 (MTU)
```

### NAT 问题诊断

**步骤 1：检测 NAT 类型**
```bash
# 使用 STUN 服务器检测 NAT 类型
stun stun.l.google.com
```

**步骤 2：检查映射超时**
- 发送 UDP 数据报
- 等待不同时间后尝试接收响应
- 确定超时时间

**步骤 3：排查穿透失败**
- 检查是否是对称 NAT
- 验证 STUN/TURN 服务器可达
- 检查防火墙规则

### Prism 特定排障

**DNS 解析问题**：
```json
// 启用详细日志
{
    "trace": {
        "log_level": "debug"
    }
}
```

**UDP 代理问题**：
1. 检查 UDP 关联是否正确建立
2. 验证数据封装格式
3. 确认超时配置合理

**多路复用问题**：
1. 检查会话 ID 管理
2. 验证数据包序号
3. 确认超时和重传机制

## 参考资料

- [RFC 768 - User Datagram Protocol](https://tools.ietf.org/html/rfc768)
- [RFC 8085 - UDP Usage Guidelines](https://tools.ietf.org/html/rfc8085)
- [RFC 8446 - The Transport Layer Security (TLS) Protocol](https://tools.ietf.org/html/rfc8446)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://tools.ietf.org/html/rfc9000)

## 相关知识

- [[ref/network/tcp|TCP]] — 传输控制协议
- [[ref/network/nat-traversal|NAT 穿透]] — NAT 穿透技术
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞争算法
- [[core/resolve/dns/upstream|DNS 上游]] — DNS 解析实现
- [[core/multiplex/parcel|Parcel]] — UDP 数据包多路复用