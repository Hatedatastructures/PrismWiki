---
title: "TCP 协议基础"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [tcp, transport, network, connection]
related:
  - "[[channel]]"
  - "[[dev/tls]]"
  - "[[dev/udp]]"
  - "[[multiplex/smux]]"
  - "[[multiplex/yamux]]"
  - "[[agent]]"
---

# TCP 协议基础

TCP（Transmission Control Protocol）是互联网最核心的传输层协议，提供可靠的、有序的、面向连接的字节流传输。几乎所有代理协议都基于 TCP 构建。

## TCP 三次握手

建立连接需要三个步骤：

```
Client                  Server
  |                       |
  |--- SYN (seq=x) ------>|  1. 客户端发起连接
  |<-- SYN-ACK (seq=y,    |  2. 服务端确认并发起
  |    ack=x+1) ----------|
  |--- ACK (ack=y+1) ---->|  3. 客户端确认
  |                       |
  |   连接建立，开始传输    |
```

### 关键状态

- 客户端：`SYN_SENT` → `ESTABLISHED`
- 服务端：`SYN_RCVD` → `ESTABLISHED`

### SYN Flood 防护

SYN Flood 攻击利用三次握手的半连接状态消耗服务器资源。防护手段：

- SYN Cookie：不分配资源，将状态编码在序列号中
- SYN Cache：限制半连接队列大小
- 防火墙限速：限制 SYN 包速率

## TCP 四次挥手

关闭连接需要四次交互：

```
Client                  Server
  |                       |
  |--- FIN -------------->|  1. 客户端请求关闭
  |<-- ACK ---------------|  2. 服务端确认
  |<-- FIN ---------------|  3. 服务端请求关闭
  |--- ACK -------------->|  4. 客户端确认
  |                       |
```

### TIME_WAIT 状态

主动关闭方进入 `TIME_WAIT` 状态，持续 2MSL（通常 60 秒）。目的：

1. 确保最后一个 ACK 到达对方
2. 等待网络中残留的报文消散

在高并发代理服务器上，大量 `TIME_WAIT` 连接会耗尽端口资源。解决方案：

- 启用 `SO_REUSEADDR` / `SO_REUSEPORT`
- 调整内核参数 `net.ipv4.tcp_tw_reuse`
- 使用连接池复用连接（见 [[channel]]）

## TCP 保活（Keep-Alive）

TCP Keep-Alive 机制用于检测死连接：

```yaml
# Linux 内核参数
net.ipv4.tcp_keepalive_time = 7200   # 空闲 2 小时后开始探测
net.ipv4.tcp_keepalive_intvl = 75    # 探测间隔 75 秒
net.ipv4.tcp_keepalive_probes = 9    # 探测 9 次无响应则断开
```

在代理场景中，通常使用应用层心跳而非 TCP Keep-Alive，因为：

- 内核级 Keep-Alive 间隔太长
- 代理可能需要更频繁的存活检测
- 部分 NAT 设备会丢弃纯 ACK 包

## TCP 在代理中的角色

代理协议本质上是在 TCP 连接上封装用户数据：

```
[Client] --TCP--> [Proxy Client] --TCP--> [Proxy Server] --TCP--> [Target]
```

### 代理对 TCP 的影响

1. **额外延迟**：多一跳连接建立（两次握手 vs 一次）
2. **连接复用**：代理可以在一条 TCP 上复用多个逻辑连接（见 [[multiplex/smux]]、[[multiplex/yamux]]）
3. **拥塞控制**：代理的拥塞控制与端到端不同，可能导致性能下降

### Prism 中的 TCP 处理

Prism 使用 Boost.Asio 进行异步 TCP 操作：

```cpp
// 简化的连接建立流程
tcp::resolver resolver(io_context);
auto results = resolver.resolve(host, port);
tcp::socket socket(io_context);
boost::asio::connect(socket, results);
```

关键设计决策：

- **非阻塞 I/O**：所有 socket 操作都是异步的，避免线程阻塞
- **SO_REUSEADDR**：允许快速重启绑定相同端口
- **TCP_NODELAY**：禁用 Nagle 算法，降低交互延迟
- **自定义缓冲区**：使用零拷贝缓冲区减少内存分配

## Happy Eyeballs（RFC 8305）

Happy Eyeballs 算法解决双栈（IPv4/IPv6）连接选择问题：

1. 同时解析 A 和 AAAA 记录
2. 优先尝试 IPv6（或按配置优先级）
3. 若首选地址在 250ms 内无响应，同时尝试另一个地址族
4. 使用先成功连接的那个

Prism 实现了 Happy Eyeballs v2，支持在连接代理节点时同时尝试多个地址，提升连接成功率。

## TCP 拥塞控制

现代拥塞控制算法对代理性能有显著影响：

| 算法 | 特点 | 适用场景 |
|------|------|----------|
| Cubic | Linux 默认，基于丢包 | 通用 |
| BBR | 基于带宽估计，不依赖丢包 | 高延迟/高丢包链路 |
| BBRv2 | 改进公平性 | 与 Cubic 混合环境 |

代理服务器建议使用 BBR 以优化高延迟链路的吞吐量。

## Socket 选项

Prism 设置的关键 socket 选项：

```cpp
// 禁用 Nagle 算法，降低延迟
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one));

// 允许地址复用
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));

// 设置 keep-alive
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &one, sizeof(one));

// 设置发送/接收缓冲区大小
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));
```

## 相关页面

- [[channel]] — 通道模块与连接管理
- [[dev/tls]] — TLS 协议基础
- [[dev/udp]] — UDP 协议基础
- [[multiplex/smux]] — SMux 多路复用
- [[multiplex/yamux]] — Yamux 多路复用
- [[agent]] — Prism Agent 概览
