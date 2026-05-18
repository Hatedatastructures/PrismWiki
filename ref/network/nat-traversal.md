---
title: "NAT 穿透"
category: "network"
type: ref
module: ref
source: "RFC 8489, RFC 8656, RFC 8445"
tags: [网络, nat, 穿透, stun, turn, ice, p2p, udp]
created: 2026-05-15
updated: 2026-05-17
---

# NAT 穿透

**类别**: 网络

## 概述

NAT 穿透（NAT Traversal）是一系列使位于 NAT（Network Address Translation，网络地址转换）后面的设备能够与其他设备建立直接通信的技术。NAT 广泛用于家庭和企业网络，允许多个设备共享单一公网 IP 地址，但同时也造成了端到端通信的障碍。NAT 穿透技术通过探测 NAT 类型、获取映射地址、协调连接等方式解决这一问题。

在代理服务器和 P2P 通信场景中，NAT 穿透是一个重要课题。理解 NAT 穿透有助于：
- 实现 P2P 代理协议
- 优化 UDP 代理实现
- 理解网络通信障碍
- 设计穿透性更好的协议

NAT 的基本原理是将私有 IP 地址转换为公网 IP 地址，使得内网设备可以访问外网资源。然而，NAT 的映射机制也阻止了外网设备主动连接内网设备，除非 NAT 有相应规则允许。这就造成了端到端通信的问题，尤其是在 P2P 场景中。

NAT 穿透的核心技术栈包括：

**STUN（Session Traversal Utilities for NAT）**：用于探测 NAT 映射后的公网地址和端口。客户端向 STUN 服务器发送请求，服务器返回客户端的公网地址，客户端据此知晓自己的映射地址。STUN 是 NAT 穿透的基础组件。

**TURN（Traversal Using Relays around NAT）**：当直接 P2P 连接无法建立时（如对称 NAT），使用 TURN 服务器作为中继转发数据。TURN 是最后的兜底方案，消耗服务器资源但保证连接性。

**ICE（Interactive Connectivity Establishment）**：综合框架，协调使用 STUN 和 TURN 尝试所有可能的连接路径，选择最优方案。ICE 是 WebRTC 等应用的 NAT 穿透标准方案。

这些技术通过 RFC 标准化：
- RFC 8489：STUN 协议
- RFC 8656：TURN 协议
- RFC 8445：ICE 框架

### NAT 穿透的应用场景

NAT 穿透技术广泛应用于以下场景：

**WebRTC 实时通信**：浏览器之间的音视频通信需要建立 P2P 连接，ICE 是 WebRTC 的核心穿透组件。

**VoIP 和视频会议**：Zoom、Skype 等应用需要穿透 NAT 建立直接媒体通道。

**P2P 文件共享**：BitTorrent 等协议需要 P2P 连接，NAT 穿透是关键。

**在线游戏**：玩家之间的直接通信需要 NAT 穿透。

**物联网设备**：IoT 设备通常在 NAT 后面，需要穿透实现远程访问。

**代理服务器**：理解 NAT 穿透有助于优化代理协议设计，特别是 UDP 代理。

## 原理详解

### NAT 类型分类

NAT 有多种类型，不同类型的穿透难度不同。了解 NAT 类型是穿透的基础。

**完全锥形 NAT（Full Cone NAT）**：
最宽松的 NAT 类型。一旦内网设备创建了映射（内网地址:端口 → 公网地址:端口），任何外网设备都可以通过该公网地址端口向内网设备发送数据。

```
内网设备(10.0.0.1:5000) → NAT → 公网(203.0.113.1:12345)

任何外网设备都可以向 203.0.113.1:12345 发送数据，
NAT 会将数据转发给 10.0.0.1:5000
```

特点：
- 映射建立后不限制来源
- 最容易穿透
- 安全性最低

**受限锥形 NAT（Restricted Cone NAT）**：
只有曾经接收过数据的内网设备才能接收外网数据。映射建立后，只有内网设备曾经发送过数据的外网 IP 地址可以发送数据到该映射端口。

```
内网设备(10.0.0.1:5000) → NAT → 公网(203.0.113.1:12345)
内网设备向 外网IP A(198.51.100.1) 发送过数据

只有 198.51.100.1:* 可以向 203.0.113.1:12345 发送数据
其他 IP 发送的数据会被 NAT 丢弃
```

特点：
- 限制发送方 IP 地址
- 不限制端口
- 穿透难度中等

**端口受限锥形 NAT（Port Restricted Cone NAT）**：
最严格的锥形 NAT。只有内网设备曾经发送过数据的确切外网地址（IP:端口）才能发送数据到该映射端口。

```
内网设备(10.0.0.1:5000) → NAT → 公网(203.0.113.1:12345)
内网设备向 外网(198.51.100.1:8000) 发送过数据

只有 198.51.100.1:8000 可以向 203.0.113.1:12345 发送数据
其他地址发送的数据会被 NAT 丢弃
```

特点：
- 限制发送方 IP 和端口
- 穿透难度较高
- 需要精确 Hole Punching

**对称 NAT（Symmetric NAT）**：
最严格的 NAT 类型。每个不同的外网目标地址使用不同的映射。内网设备向不同的外网地址发送数据时，NAT 会创建不同的映射。

```
内网设备(10.0.0.1:5000) → NAT → 目标A → 公网(203.0.113.1:12345)
内网设备(10.0.0.1:5000) → NAT → 目标B → 公网(203.0.113.1:12346)

向不同目标发送数据使用不同的公网端口
每个映射只允许对应的外网地址发送数据
```

特点：
- 每目标独立映射
- 无法直接 Hole Punching
- 通常需要 TURN 中继

**NAT 类型对比表**：

| 类型 | 映射规则 | 入站规则 | 穿透难度 | 常见场景 |
|------|---------|---------|---------|---------|
| 完全锥形 | 固定映射 | 任何来源 | 易 | 企业 NAT |
| 受限锥形 | 固定映射 | 同 IP 来源 | 中 | 部分家庭 NAT |
| 端口受限锥形 | 固定映射 | 同 IP:端口来源 | 较难 | 大部分家庭 NAT |
| 对称 NAT | 每目标不同映射 | 仅对应目标 | 难 | 移动网络、严格 NAT |

### NAT 类型检测

STUN 协议用于检测 NAT 类型，RFC 8489 定义了检测流程：

**检测步骤**：

**步骤 1：测试 I**：
向 STUN 服务器发送 Binding Request，观察响应：
- 如果收到的地址与本地地址相同 → 无 NAT（公网直连）
- 如果收到的地址与本地地址不同 → 有 NAT，继续检测

**步骤 2：测试 II**：
向 STUN 服务器发送请求，请求服务器从不同 IP 地址响应：
- 如果收到响应 → 完全锥形 NAT（允许任何来源）
- 如果未收到响应 → 受限类型，继续检测

**步骤 3：测试 III**：
向 STUN 服务器发送请求，请求服务器从相同 IP 但不同端口响应：
- 如果收到响应 → 受限锥形 NAT（允许同 IP 不同端口）
- 如果未收到响应 → 端口受限锥形或对称 NAT

**步骤 4：测试 IV**：
向 STUN 服务器不同 IP 地址发送请求，比较映射地址：
- 如果映射地址相同 → 端口受限锥形 NAT
- 如果映射地址不同 → 对称 NAT

**检测流程图**：
```
                    +---------+
                    | 开始    |
                    +----+----+
                         |
                    发送 Test I
                         |
              +----------+----------+
              |                     |
         相同地址              不同地址
              |                     |
    +---------v---------+     +-----v-----+
    | 无 NAT（公网）     |     | 有 NAT    |
    +-------------------+     +-----+-----+
                                  |
                             发送 Test II
                                  |
                        +---------+---------+
                        |                   |
                    收到响应            未收到响应
                        |                   |
            +-----------v-----------+       |
            | 完全锥形 NAT          |       |
            +-----------------------+       |
                                            |
                                       发送 Test III
                                            |
                              +-------------+-------------+
                              |                           |
                          收到响应                  未收到响应
                              |                           |
                  +-----------v-----------+               |
                  | 受限锥形 NAT          |               |
                  +-----------------------+               |
                                                            |
                                                       发送 Test IV
                                                            |
                                                  +---------+---------+
                                                  |                   |
                                              相同地址            不同地址
                                                  |                   |
                                      +-----------v-----------v-----------+
                                      | 端口受限锥形 NAT    对称 NAT     |
                                      +-----------------------------------+
```

### STUN 协议详解

STUN（Session Traversal Utilities for NAT）是一个轻量级协议，用于探测 NAT 映射地址。

**STUN 报文结构**：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0 0|     STUN Message Type     |         Message Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Magic Cookie                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Transaction ID (96 bits)                  |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Attributes                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**消息类型**：
- Binding Request（0x0001）：请求映射地址
- Binding Response（0x0101）：成功响应
- Binding Error Response（0x0111）：错误响应

**Magic Cookie**：
固定值 0x21C3A42A，用于区分 STUN 报文和其他协议。

**Transaction ID**：
96 位随机 ID，用于匹配请求和响应。

**关键属性**：

| 属性类型 | 值 | 说明 |
|---------|-----|------|
| MAPPED-ADDRESS | 0x0001 | 映射后的地址和端口 |
| XOR-MAPPED-ADDRESS | 0x0020 | XOR 编码的映射地址（推荐） |
| CHANGE-REQUEST | 0x0003 | 请求服务器改变响应地址 |
| CHANGED-ADDRESS | 0x0005 | 服务器用于 CHANGE-REQUEST 的新地址 |
| ERROR-CODE | 0x0009 | 错误码 |

**XOR-MAPPED-ADDRESS 编码**：
使用 Magic Cookie XOR 地址，防止 NAT 修改地址信息：
```
原始地址：192.0.2.1:12345
XOR 后地址：192.0.2.1 XOR 0x21C3A42A
```

**STUN 请求示例**：
```cpp
// 构建 STUN Binding Request
std::vector<uint8_t> build_stun_request() {
    std::vector<uint8_t> request(20);  // 最小 STUN 头部

    // 消息类型：Binding Request
    request[0] = 0x00;
    request[1] = 0x01;

    // 消息长度：无属性
    request[2] = 0x00;
    request[3] = 0x00;

    // Magic Cookie
    request[4] = 0x21;
    request[5] = 0xC3;
    request[6] = 0xA4;
    request[7] = 0x2A;

    // Transaction ID（随机）
    std::random_device rd;
    std::mt19937 gen(rd());
    for (int i = 8; i < 20; ++i) {
        request[i] = gen() & 0xFF;
    }

    return request;
}
```

### UDP Hole Punching

UDP Hole Punching 是锥形 NAT 的主要穿透技术，利用 NAT 的映射机制实现 P2P 连接。

**基本原理**：
1. 双方分别向 STUN 服务器获取自己的公网地址
2. 双方通过信令服务器交换地址信息
3. 双方同时向对方的公网地址发送 UDP 数据报
4. NAT 收到对方的数据报时已有对应映射，允许通过

**详细流程**：
```
        Peer A (内网)             STUN Server            Peer B (内网)
        10.0.0.1:5000            198.51.100.1           10.0.0.2:6000
             |                        |                      |
             |--- STUN Request ------>|                      |
             |                        |                      |
             |<-- STUN Response ------|                      |
             |   (映射: 203.0.113.1:12345)                   |
             |                        |                      |
             |                        |<----- STUN Request --|
             |                        |                      |
             |                        |----- STUN Response >|
             |                        |   (映射: 203.0.113.2:23456)
             |                        |                      |
             |==== 通过信令服务器交换地址信息 ====
             |                        |                      |
             |--- UDP to 203.0.113.2:23456 --> X (先发)      |
             |                        |                      |
             |                        |      X <-- UDP to 203.0.113.1:12345 ---|
             |                        |                      |
             |<--- UDP from B --------|----------------------|
             |   (映射已建立，允许通过)                        |
             |                        |                      |
             |======== P2P 连接建立 ========|
```

**关键技术点**：
- 同时发送：双方几乎同时发送，确保双方 NAT 都建立映射
- 重复发送：穿透数据报可能被 NAT 丢弃，需要重复发送直到连接建立
- 保活维护：持续发送保活包维持 NAT 映射

**端口受限锥形 NAT 的处理**：
端口受限锥形 NAT 需要精确预测对方的映射端口：
1. 使用 STUN 获取对方的确切映射地址
2. 向该确切地址发送数据报
3. NAT 检测到地址匹配，允许通过

**对称 NAT 的挑战**：
对称 NAT 无法使用 Hole Punching：
- Peer A 向 STUN 的映射 != Peer A 向 Peer B 的映射
- Peer B 无法知道 Peer A 向自己发送数据时使用的端口
- 需要使用 TURN 中继

### TURN 协议详解

TURN（Traversal Using Relays around NAT）是最后的穿透方案，通过中继服务器转发数据。

**适用场景**：
- 对称 NAT 无法 Hole Punching
- 双方都是严格 NAT
- 防火墙完全阻止 UDP
- 需要 guaranteed connectivity

**TURN 架构**：
```
        Peer A                 TURN Server              Peer B
        (内网)                  (公网)                   (内网)
             |                      |                      |
             |--- Allocate Request >|                      |
             |<-- Allocate Response-|                      |
             |   (relay地址: port)   |                      |
             |                      |                      |
             |==== 通过信令交换 relay 地址 ====
             |                      |                      |
             |--- Data to B -------->|--- Data to B -------->|
             |                      |                      |
             |<-- Data from B -------|<-- Data from B -------|
             |                      |                      |
             |======== 通过 TURN 中继通信 ========|
```

**TURN 报文类型**：

| 类型 | 说明 |
|------|------|
| ALLOCATE | 申请中继端口 |
| REFRESH | 刷新分配 |
| SEND | 发送数据 |
| DATA | 接收数据 |
| CREATE_PERMISSION | 创建转发权限 |
| ChannelBind | 创建通道（优化传输） |

**ALLOCATE 流程**：
```cpp
// TURN Allocate Request
auto allocate_request = build_turn_allocate();

// 发送请求
co_await turn_socket.async_send_to(allocate_request, turn_server);

// 接收响应
auto response = co_await turn_socket.async_receive_from(buffer, sender);

// 解析 relay 地址
auto relay_address = parse_turn_allocate_response(response);
// relay_address: 例如 198.51.100.1:50000
```

**数据转发**：
```cpp
// 使用 SEND 指令发送数据
auto send_request = build_turn_send(peer_address, data);
co_await turn_socket.async_send_to(send_request, turn_server);

// 接收 DATA 指令转发来的数据
auto data_indication = co_await turn_socket.async_receive_from(buffer, sender);
auto forwarded_data = parse_turn_data(data_indication);
```

**ChannelBind 优化**：
创建通道后，数据转发使用简化的通道格式：
```cpp
// 创建通道
auto channel_bind = build_turn_channel_bind(peer_address, channel_number);
co_await turn_socket.async_send_to(channel_bind, turn_server);

// 通道数据格式更简洁
auto channel_data = build_channel_data(channel_number, data);
// 通道数据头部仅 4 字节，比 SEND 指令更高效
```

**TURN vs STUN**：

| 特性 | STUN | TURN |
|------|------|------|
| 功能 | 地址探测 | 中继转发 |
| 服务器负载 | 低 | 高 |
| 带宽消耗 | 无 | 双倍（进出） |
| 延迟 | 低 | 高（额外中继） |
| 适用场景 | 锥形 NAT | 对称 NAT |

### ICE 框架详解

ICE（Interactive Connectivity Establishment）是综合 NAT 穿透框架，协调使用 STUN 和 TURN。

**ICE 核心概念**：

**候选地址（Candidate）**：
可能的连接地址，分为：
- Host Candidate：本地接口地址
- Server Reflexive Candidate：通过 STUN 获取的公网地址
- Peer Reflexive Candidate：对方 Hole Punching 时发现的地址
- Relayed Candidate：TURN 中继地址

**候选地址优先级**：
优先级计算公式：
```
priority = (type_preference << 24) |
           (local_preference << 8) |
           (component_id)
```

类型优先级：
| 类型 | 偏好值 |
|------|--------|
| Host | 126 |
| Peer Reflexive | 110 |
| Server Reflexive | 100 |
| Relayed | 0 |

**ICE 流程**：

**步骤 1：收集候选地址**：
```cpp
// 收集本地地址
auto host_candidates = get_local_addresses();

// 通过 STUN 获取公网地址
auto srflx_candidates = stun_get_mapped_addresses();

// 通过 TURN 获取中继地址
auto relay_candidates = turn_allocate();
```

**步骤 2：交换候选地址**：
通过信令协议（如 SDP）交换候选地址：
```
a=candidate:1 1 UDP 2130706431 10.0.0.1 5000 typ host
a=candidate:2 1 UDP 1694498815 203.0.113.1 12345 typ srflx raddr 10.0.0.1 rport 5000
a=candidate:3 1 UDP 16777215 198.51.100.1 50000 typ relay raddr 10.0.0.1 rport 5000
```

**步骤 3：连通性检查**：
```cpp
// ICE 连通性检查
for (auto& local_candidate : local_candidates) {
    for (auto& remote_candidate : remote_candidates) {
        // 按优先级排序的检查对
        auto check_pair = make_check_pair(local_candidate, remote_candidate);

        // 发送 STUN Binding Request 检查连通性
        auto result = co_await connectivity_check(check_pair);

        if (result.success) {
            // 记录成功的候选对
            valid_pairs.push_back(check_pair);
        }
    }
}
```

**步骤 4：选择最佳候选对**：
选择优先级最高的成功候选对：
- Host-Host：最优（直连）
- Host-Srflx：良好（单向 NAT）
- Srflx-Srflx：P2P Hole Punching
- Relay-Relay：最差（双向中继）

**步骤 5：冻结和激活**：
ICE 使用"冻结"机制避免不必要的检查：
- 按组件分组
- 同组高优先级对成功后，低优先级对冻结
- 失败后激活冻结的对

**ICE 完整流程图**：
```
                    +---------+
                    | 开始    |
                    +----+----+
                         |
              +----------v-----------+
              |  收集候选地址        |
              |  - host              |
              |  - srflx (STUN)      |
              |  - relay (TURN)      |
              +----------+-----------+
                         |
              +----------v-----------+
              |  通过信令交换        |
              |  候选地址            |
              +----------+-----------+
                         |
              +----------v-----------+
              |  排序候选对          |
              |  按优先级排序        |
              +----------+-----------+
                         |
              +----------v-----------+
              |  连通性检查          |
              |  发送 STUN check     |
              |  按顺序检查          |
              +----------+-----------+
                         |
              +----------+-----------+
              |  选择最佳候选对      |
              |  最高优先级成功对    |
              +----------+-----------+
                         |
              +----------v-----------+
              |  保持连接            |
              |  定期 STUN keepalive |
              +----------+-----------+
                         |
                    +----v----+
                    | 完成    |
                    +---------+
```

### NAT 穿透最佳实践

**保活机制**：
NAT 映射超时通常 30-120 秒，需要定期发送保活包：
```cpp
// 保活协程
auto keepalive_loop(net::udp::socket& sock, endpoint target)
    -> net::awaitable<void> {
    net::steady_timer timer(sock.get_executor());
    while (running) {
        timer.expires_after(std::chrono::seconds(15));
        co_await timer.async_wait(net::use_awaitable);
        // 发送保活包
        co_await sock.async_send_to(empty_packet, target);
    }
}
```

**重试策略**：
穿透数据报可能丢失，需要重复发送：
```cpp
// 重试穿透
for (int retry = 0; retry < max_retries; ++retry) {
    co_await send_hole_punch_packet(peer_address);
    // 短暂等待响应
    auto result = co_await wait_for_response(100ms);
    if (result) break;
}
```

**多地址尝试**：
同时尝试多个地址提高成功率：
```cpp
// 并行尝试多个候选地址
auto checks = std::vector<net::awaitable<bool>>();
for (auto& addr : candidates) {
    checks.push_back(connectivity_check(addr));
}
auto results = co_await parallel_wait(checks);
```

## 在 Prism 中的应用

Prism 作为代理服务器，虽然不直接实现 P2P NAT 穿透，但理解 NAT 穿透对于 UDP 代理和协议设计有重要意义。

### UDP 代理与 NAT

**SOCKS5 UDP ASSOCIATE**：
SOCKS5 的 UDP 代理涉及 NAT 映射：
```
客户端(内网) → SOCKS5代理 → 目标服务器(公网)

客户端向代理的 UDP 中继端口发送数据
代理转发到目标服务器
目标响应返回代理
代理转发给客户端
```

**NAT 影响**：
- 客户端在 NAT 后面，需要知道自己的映射地址
- UDP 关联建立时，客户端告知代理自己的接收地址
- 代理需要正确处理 NAT 映射

**Trojan UDP 转发**：
Trojan 协议将 UDP 数据封装在 TCP 流中：
```
UDP 数据 → Trojan 封装 → TCP 流 → Trojan 解封装 → UDP 数据
```
这种方式避免了 NAT 映射问题，但增加了 TCP 开销。

### DNS 与 NAT

Prism 的 DNS 解析模块需要考虑 NAT：
- DNS 查询使用 UDP，需要 NAT 映射
- DNS over TLS/TCP 避免 NAT 映射超时问题
- 缓存 DNS 结果减少查询次数

### 相关概念应用

**连接竞争与 NAT**：
Happy Eyeballs 在 NAT 环境下同样有效：
- IPv4 和 IPv6 可能经过不同 NAT
- 选择最优路径
- NAT 映射保活

**连接池与 NAT**：
NAT 映射超时影响连接池：
- 长时间空闲连接可能失效
- 需要定期保活或重新建立
- 空闲超时参数与 NAT 超时配合

### 相关模块链接

| 模块 | 功能 | 文档 |
|------|------|------|
| parcel | UDP 数据包多路复用 | [[core/multiplex/parcel|parcel]] |
| udp_relay | UDP 中继 | [[core/protocol/common/udp_relay|udp_relay]] |
| upstream | DNS 上游查询 | [[core/resolve/dns/upstream|upstream]] |
| eyeball | Happy Eyeballs | [[core/channel/eyeball|eyeball]] |

## 最佳实践

### NAT 穿透设计原则

**优先使用直连**：
- 优先尝试 Host 候选地址
- STUN 探测后尝试 Hole Punching
- TURN 作为最后方案

**并行尝试**：
- 多候选地址同时尝试
- 减少穿透时间
- 提高成功率

**保活策略**：
- 定期发送保活包
- 保活间隔 < NAT 超时/2
- 使用 STUN Binding Request 作为保活

### TURN 服务器部署

**容量规划**：
- 带宽：每用户约 2 倍媒体带宽
- CPU：加密处理开销
- 内存：连接状态存储

**地理分布**：
- 多区域部署减少延迟
- 用户就近选择服务器
- 冗余保证可用性

**安全考虑**：
- 认证机制防止滥用
- 速率限制防止攻击
- 日志审计追踪使用

### UDP 代理 NAT 处理

**SOCKS5 UDP 最佳实践**：
- 正确处理 UDP ASSOCIATE 请求
- 验证客户端地址有效性
- 定期检查 NAT 映射状态

**Trojan UDP 建议**：
- 使用 TCP 封装避免 NAT 问题
- 适用于长连接场景
- 短消息场景开销较大

## 常见问题

### Q1: 为什么 Hole Punching 有时会失败？

**原因**：
1. 对称 NAT：无法预测对方映射端口
2. 双向严格 NAT：双方都无法接收对方数据
3. 防火墙阻止 UDP
4. 保活不及时导致映射过期

**解决**：
- 使用 TURN 中继
- 尝试 TCP 穿透
- 优化保活策略

### Q2: STUN 和 TURN 服务器有什么区别？

**STUN**：
- 轻量级，仅地址探测
- 低服务器负载
- 不转发数据

**TURN**：
- 中继转发数据
- 高服务器负载（带宽）
- Guaranteed connectivity

### Q3: 如何判断 NAT 类型？

**方法**：
- 使用 STUN 协议检测
- RFC 8489 定义的标准检测流程
- 多种开源工具可用（如 stunclient）

**工具**：
```bash
# 使用 stunclient 检测
stunclient stun.l.google.com 19302
```

### Q4: ICE 检查为什么这么慢？

**原因**：
- 多候选地址需要逐一检查
- 网络延迟影响检查时间
- 串行检查效率低

**优化**：
- 使用并行检查
- 冻结低优先级对
- 快速触发策略

### Q5: NAT 穿透有安全风险吗？

**风险**：
- Hole Punching 可能被滥用
- TURN 中继可能泄露数据
- DoS 攻击可能性

**防护**：
- 认证机制
- 速率限制
- 加密传输

## 排障指南

### NAT 类型诊断

**步骤 1：使用 STUN 工具检测**
```bash
# Linux
stunclient stun.l.google.com 19302

# Windows (使用 pystun)
pip install pystun
pystun
```

**步骤 2：分析检测结果**
- No NAT：公网直连
- Full Cone：容易穿透
- Restricted：需要 Hole Punching
- Symmetric：需要 TURN

**步骤 3：验证 NAT 行为**
```bash
# 发送 UDP 到不同目标，观察映射变化
# 对称 NAT：不同目标不同映射
nc -u stun1.l.google.com 19302
nc -u stun2.l.google.com 19302
```

### Hole Punching 问题排查

**步骤 1：检查候选地址**
- 确认获取了正确的公网地址
- 验证地址交换成功
- 检查地址优先级排序

**步骤 2：检查时序**
- 双方是否同时发送穿透包
- 发送间隔是否合适
- 是否有足够的重试

**步骤 3：检查防火墙**
```bash
# 检查防火墙规则
iptables -L -n | grep udp

# 检查是否有 UDP 阻止规则
```

### TURN 连接问题排查

**步骤 1：检查 TURN 服务器**
```bash
# 测试 TURN 服务器连通性
ping turn.example.com

# 测试 TURN 服务
turnutils_uclient -u user -p pass turn.example.com
```

**步骤 2：检查认证**
- 验证用户名密码
- 检查认证时效
- 确认权限配置

**步骤 3：检查资源**
- TURN 服务器是否有足够带宽
- 是否达到用户限制
- 是否有资源耗尽

### Prism UDP 代理排障

**步骤 1：检查 UDP 关联**
```cpp
// 日志记录 UDP 关联
spdlog::debug("UDP ASSOCIATE: client_addr={}, relay_addr={}",
              client_address, relay_address);
```

**步骤 2：检查数据转发**
```cpp
// 记录 UDP 数据转发
spdlog::debug("UDP forward: src={}, dst={}, size={}",
              source, destination, data_size);
```

**步骤 3：检查 NAT 映射状态**
- 客户端映射是否有效
- 保活是否正常发送
- 超时配置是否合理

## 参考资料

- [RFC 8489 - STUN](https://tools.ietf.org/html/rfc8489)
- [RFC 8656 - TURN](https://tools.ietf.org/html/rfc8656)
- [RFC 8445 - ICE](https://tools.ietf.org/html/rfc8445)
- [RFC 4787 - NAT UDP Behavioral Requirements](https://tools.ietf.org/html/rfc4787)

## 相关知识

- [[ref/network/tcp|TCP]] — 传输控制协议
- [[ref/network/udp|UDP]] — 用户数据报协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞速算法
- [[ref/network/connection-pool|连接池]] — 连接复用技术
- [[core/multiplex/parcel|Parcel]] — UDP 数据包多路复用
- [[core/protocol/common/udp_relay|UDP Relay]] — UDP 中继实现