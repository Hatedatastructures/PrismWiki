---
title: UDP 协议实践指南
source:
  - I:/code/Prism/include/prism/channel/parcel.hpp
  - I:/code/Prism/include/prism/protocol/socks5/udp.hpp
  - I:/code/Prism/src/prism/channel/parcel/
  - I:/code/Prism/src/prism/protocol/socks5/udp.cpp
module: channel
type: reference
tags: [udp, transport, network, socks5, parcel, datagram, dns]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[modules/channel]]"
  - "[[modules/protocol]]"
  - "[[modules/multiplex]]"
  - "[[dev/tcp]]"
  - "[[configuration]]"
---

# UDP 协议实践指南

UDP（User Datagram Protocol）是无连接的传输层协议，提供简单、低开销的数据报传输。在代理领域，UDP 支持是完整代理功能的重要组成部分。

## 概述

### UDP 协议特性

UDP 与 TCP 的本质差异：

| 特性 | UDP | TCP |
|------|-----|-----|
| 连接模式 | 无连接 | 面向连接 |
| 可靠性 | 不保证送达 | 可靠传输（重传、确认） |
| 顺序性 | 不保证顺序 | 有序传输（序列号） |
| 头部大小 | 固定 8 字节 | 最少 20 字节（可变） |
| 流量控制 | 无 | 有（滑动窗口） |
| 拥塞控制 | 无 | 有（慢启动、拥塞避免） |
| 数据边界 | 保留（数据报） | 流式（无边界） |
| 广播/多播 | 支持 | 不支持 |

UDP 数据报头部格式：

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

字段说明：
- **Source Port（16 bit）**：源端口，可选（0 表示不指定）
- **Destination Port（16 bit）**：目标端口
- **Length（16 bit）**：整个数据报长度（头部 + 数据）
- **Checksum（16 bit）**：校验和，覆盖头部和数据

### UDP 的典型应用场景

| 应用 | 使用 UDP 的原因 |
|------|----------------|
| DNS 查询 | 低延迟，小数据包（传统 512B） |
| VoIP/视频流 | 实时性优先于可靠性 |
| 在线游戏 | 低延迟，可容忍丢包 |
| DHCP | 广播需求 |
| NTP | 时间同步，低延迟 |
| QUIC | 基于 UDP 的可靠传输 |
| WebRTC | P2P 通信，实时性 |

### UDP 在代理中的挑战

#### NAT 穿透问题

NAT（Network Address Translation）设备对 UDP 的处理与 TCP 不同：

```
客户端 (192.168.1.100:5000)
    │
    │ UDP 包 → 目标 8.8.8.8:53
    ▼
NAT 设备
    │ 创建映射: 192.168.1.100:5000 ↔ 公网IP:12345
    │ 
    │ 超时机制: 30-120秒无活动则删除映射
    ▼
公网 (公网IP:12345)
    │
    │ UDP 包 → 8.8.8.8:53
    ▼
DNS 服务器
```

NAT 映射类型：

| 类型 | 行为 | 穿透难度 |
|------|------|----------|
| Full Cone | 任何外部地址可发送到映射端口 | 容易 |
| Restricted Cone | 仅响应过的地址可发送 | 中等 |
| Port Restricted | 仅响应过的地址+端口可发送 | 较难 |
| Symmetric | 每个目标不同映射端口 | 最难 |

UDP NAT 超时通常 30-120 秒，需要保活机制维持映射。

#### 有状态防火墙

防火墙的 UDP 规则通常基于"已建立的会话"：

- 只有出站 UDP 请求对应的入站响应才被允许
- 代理服务器返回的 UDP 包必须匹配之前的出站请求
- 防火墙可能对 UDP 包大小有限制

#### 数据报边界保留

UDP 与 TCP 的根本差异：

```
TCP: 流式传输，无边界
  发送: send("Hello"); send("World");
  接收: recv() → "HelloWorld" 或 "Hel" + "loWorld"
  
UDP: 数据报传输，保留边界
  发送: sendto("Hello"); sendto("World");
  接收: recvfrom() → "Hello"; recvfrom() → "World"
```

代理必须正确处理：
- 每个数据报的目标地址信息
- 数据报边界不能合并或拆分
- UDP 会话管理（源地址 → 目标地址映射）

---

## 详解

### SOCKS5 UDP ASSOCIATE

SOCKS5 协议通过 UDP ASSOCIATE 命令支持 UDP 代理。

#### 工作流程

```
客户端                        SOCKS5 服务器
  │                               │
  │── TCP: UDP ASSOCIATE请求 ────→│  请求 UDP 代理能力
  │   (指定目标地址，通常为0.0.0.0)│
  │                               │
  │←─ TCP: BND.ADDR:BND.PORT ────│  服务器返回 UDP 监听地址
  │                               │
  │                               │
  │╒══ UDP 数据传输 ══════════════│  UDP 数据通过指定端口
  ││ [UDP请求头][实际数据]        │
  ││ → 目标地址                   │
  │╘══════════════════════════════│
  │                               │
  │←─ UDP: [UDP请求头][响应] ────│  响应数据
  │                               │
  │── TCP: 关闭控制连接 ─────────→│  结束 UDP 代理
  │                               │
```

关键点：
- UDP ASSOCIATE 需要 TCP 控制连接存在
- TCP 连接断开后 UDP 代理终止
- UDP 数据通过服务器返回的端口传输

#### UDP 请求头格式

```
+----+------+------+----------+----------+----------+
|RSV | FRAG | ATYP | DST.ADDR | DST.PORT |   DATA   |
+----+------+------+----------+----------+----------+
| 2  |  1   |  1   | Variable |    2     | Variable |
+----+------+------+----------+----------+----------+
```

字段说明：

| 字段 | 大小 | 说明 |
|------|------|------|
| RSV | 2 字节 | 保留，必须为 0 |
| FRAG | 1 字节 | 分片编号（0 = 未分片，实践中很少使用分片） |
| ATYP | 1 字节 | 地址类型（1=IPv4, 3=域名, 4=IPv6） |
| DST.ADDR | 可变 | 目标地址（IPv4=4B, 域名=1B长度+域名, IPv6=16B） |
| DST.PORT | 2 字节 | 目标端口 |
| DATA | 可变 | 实际数据 |

#### 地址类型编码

```
ATYP = 1 (IPv4):
  DST.ADDR = 4 字节 IPv4 地址
  例: 192.168.1.1 → 0xC0 0xA8 0x01 0x01

ATYP = 3 (域名):
  DST.ADDR = 1 字节长度 + 域名
  例: example.com → 0x0B "example.com"

ATYP = 4 (IPv6):
  DST.ADDR = 16 字节 IPv6 地址
  例: ::1 → 0x00...00 0x01 (16字节)
```

#### Prism SOCKS5 UDP 实现

```cpp
/**
 * @brief SOCKS5 UDP 数据报解析
 * @param buffer 原始 UDP 数据
 * @return 解析结果（地址、端口、数据）
 */
auto parse_udp_datagram(std::span<uint8_t> buffer) -> udp_datagram {
    // 1. 验证最小长度
    if (buffer.size() < 10) {
        throw protocol_error("UDP datagram too short");
    }
    
    // 2. 解析固定头部
    uint8_t atyp = buffer[3];
    
    // 3. 解析地址
    size_t addr_offset = 4;
    size_t addr_len = 0;
    std::string address;
    
    switch (atyp) {
        case 0x01:  // IPv4
            addr_len = 4;
            address = format_ipv4(buffer.subspan(addr_offset, 4));
            break;
        case 0x03:  // 域名
            addr_len = buffer[addr_offset] + 1;  // 长度字节 + 域名
            address = std::string(buffer.data() + addr_offset + 1, buffer[addr_offset]);
            break;
        case 0x04:  // IPv6
            addr_len = 16;
            address = format_ipv6(buffer.subspan(addr_offset, 16));
            break;
        default:
            throw protocol_error("Invalid address type");
    }
    
    // 4. 解析端口
    size_t port_offset = addr_offset + addr_len;
    uint16_t port = (buffer[port_offset] << 8) | buffer[port_offset + 1];
    
    // 5. 提取数据
    size_t data_offset = port_offset + 2;
    auto data = buffer.subspan(data_offset);
    
    return udp_datagram{address, port, data};
}

/**
 * @brief 构建 SOCKS5 UDP 数据报
 * @param address 目标地址
 * @param port 目标端口
 * @param data 实际数据
 * @return 编码后的 UDP 数据报
 */
auto build_udp_datagram(std::string address, uint16_t port, std::span<uint8_t> data) -> memory::vector<uint8_t> {
    memory::vector<uint8_t> buffer;
    
    // RSV (2 bytes) + FRAG (1 byte)
    buffer.append({0, 0, 0});
    
    // ATYP + 地址
    if (is_ipv4(address)) {
        buffer.push_back(0x01);
        buffer.append(parse_ipv4_bytes(address));
    } else if (is_ipv6(address)) {
        buffer.push_back(0x04);
        buffer.append(parse_ipv6_bytes(address));
    } else {
        buffer.push_back(0x03);
        buffer.push_back(address.size());
        buffer.append(address.begin(), address.end());
    }
    
    // 端口
    buffer.push_back(port >> 8);
    buffer.push_back(port & 0xFF);
    
    // 数据
    buffer.append(data.begin(), data.end());
    
    return buffer;
}
```

---

### UDP 中继实现

#### Prism UDP 中继架构

```
+------------------------+
|    UDP 中继模块         |
+------------------------+
          │
    +-----+-----+
    │           │
+--------+  +--------+
|会话管理|  |地址映射|
+--------+  +--------+
    │           │
    +-----+-----+
          │
+------------------------+
|    Parcel 封装层        |  ← 多路复用时使用
+------------------------+
```

#### 会话管理

UDP 中继为每个源地址（src_addr:src_port）维护会话：

```cpp
/**
 * @brief UDP 会话管理器
 * @details 维护源地址到目标地址的映射
 */
class udp_session_manager {
public:
    /**
     * @brief 获取或创建会话
     * @param src_addr 源地址
     * @param src_port 源端口
     * @return 会话对象
     */
    auto get_or_create(std::string src_addr, uint16_t src_port) -> udp_session& {
        auto key = make_key(src_addr, src_port);
        
        auto it = sessions_.find(key);
        if (it != sessions_.end()) {
            // 更新活跃时间
            it->second.last_active = steady_clock::now();
            return it->second;
        }
        
        // 创建新会话
        auto& session = sessions_[key];
        session.src_addr = src_addr;
        session.src_port = src_port;
        session.last_active = steady_clock::now();
        
        return session;
    }
    
    /**
     * @brief 清理过期会话
     * @param timeout 超时时间（秒）
     */
    void cleanup_expired(uint32_t timeout) {
        auto deadline = steady_clock::now() - seconds(timeout);
        
        for (auto it = sessions_.begin(); it != sessions_.end();) {
            if (it->second.last_active < deadline) {
                it = sessions_.erase(it);
            } else {
                ++it;
            }
        }
    }
    
private:
    std::unordered_map<std::string, udp_session> sessions_;
};
```

#### 地址映射

UDP 中继需要维护双向映射：

```
客户端源: 192.168.1.100:5000
    │
    │ UDP → 目标: 8.8.8.8:53
    ▼
代理服务器
    │ 记录映射: 客户端源 ↔ 目标地址
    │ 
    │ 转发 → 目标
    ▼
目标服务器
    │
    │ 响应 → 代理服务器
    ▼
代理服务器
    │ 查找映射: 目标响应 → 客户端源
    │ 
    │ 转发 → 客户端
    ▼
客户端
```

---

### Parcel 模块

在多路复用场景中，UDP 数据报通过 Parcel 模块封装在 TCP 连接上传输。

#### Parcel 封装格式

```
+--------+--------+------------------+--------+--------+
| Type   | Length | Address + Port   | Data   | Padding|
| 1 byte | 2 byte | Variable         | Variable| Variable|
+--------+--------+------------------+--------+--------+
```

Type 字段：

| 值 | 含义 |
|----|------|
| 0x01 | IPv4 数据报 |
| 0x02 | 域名数据报 |
| 0x03 | IPv6 数据报 |

#### Parcel 与 TCP 多路复用

```
[UDP 应用] ─UDP─→ [Prism 客户端] ─TCP─Parcel─→ [Prism 服务器] ─UDP─→ [目标]
                 │                              │
                 │ UDP → Parcel 封装            │ Parcel → UDP 解包
                 │                              │
                 └──────────────────────────────┘
                     TCP 连接上模拟 UDP 语义
```

Parcel 的作用：
- 保留数据报边界（TCP 是流式的，需要边界标记）
- 携带目标地址信息（每次数据报可能有不同目标）
- 支持双向传输（上行和下行各用不同 Parcel）

#### Prism Parcel 实现

```cpp
/**
 * @brief Parcel 数据报构建
 * @param address 目标地址
 * @param port 目标端口
 * @param data 实际数据
 * @return Parcel 编码字节流
 */
auto parcel::build(std::string address, uint16_t port, std::span<uint8_t> data) -> memory::vector<uint8_t> {
    memory::vector<uint8_t> buffer;
    
    // 1. Type（根据地址类型）
    uint8_t type = determine_type(address);
    buffer.push_back(type);
    
    // 2. 计算总长度（地址 + 端口 + 数据）
    size_t addr_len = address.size() + (type == 0x03 ? 1 : 0);  // 域名额外有长度字节
    size_t total_len = addr_len + 2 + data.size();
    
    // 3. Length（2 字节，大端）
    buffer.push_back(total_len >> 8);
    buffer.push_back(total_len & 0xFF);
    
    // 4. Address + Port
    encode_address(buffer, address);
    buffer.push_back(port >> 8);
    buffer.push_back(port & 0xFF);
    
    // 5. Data
    buffer.append(data.begin(), data.end());
    
    return buffer;
}

/**
 * @brief Parcel 数据报解析
 * @param buffer Parcel 字节流
 * @return 解析结果
 */
auto parcel::parse(std::span<uint8_t> buffer) -> parcel_result {
    // 1. Type
    uint8_t type = buffer[0];
    
    // 2. Length
    uint16_t length = (buffer[1] << 8) | buffer[2];
    
    // 3. Address
    size_t offset = 3;
    std::string address = decode_address(buffer, type, offset);
    
    // 4. Port
    uint16_t port = (buffer[offset] << 8) | buffer[offset + 1];
    offset += 2;
    
    // 5. Data
    auto data = buffer.subspan(offset, length - (offset - 3));
    
    return parcel_result{address, port, data};
}
```

---

### DNS over UDP vs TCP

DNS 是 UDP 的典型应用，但也可使用 TCP。

#### DNS 协议对比

| 特性 | DNS over UDP | DNS over TCP |
|------|-------------|-------------|
| 端口 | 53 | 53 |
| 数据包大小 | 传统 512B（EDNS0 可扩展） | 无限制 |
| 可靠性 | 无保证，依赖重试 | 可靠传输 |
| 延迟 | 1-RTT | 2-RTT（需要建立连接） |
| 使用场景 | 常规查询 | 大响应（>512B）、DNSSEC、AXFR |

#### EDNS0 扩展

EDNS0（Extension Mechanisms for DNS）扩展 UDP 包大小：

```
传统 DNS: 最大 512 字节
EDNS0: 最大 4096 字节（通常配置）
```

OPT RR（伪资源记录）携带扩展信息：

```
OPT RR 格式:
  NAME: 0 (根域名)
  TYPE: OPT (41)
  UDP Payload Size: 最大 UDP 包大小
  Extended RCODE: 扩展错误码
  VERSION: EDNS 版本 (0)
  ...
```

#### DNS 加密协议

现代 DNS 加密方案：

| 协议 | 端口 | 传输层 | 特点 |
|------|------|--------|------|
| Do53 (传统) | 53 | UDP/TCP | 无加密 |
| DoT | 853 | TCP + TLS | 加密，需建立 TLS |
| DoH | 443 | TCP + HTTPS | 加密，伪装成 HTTPS |
| DoQ | 853 | UDP + QUIC | 加密，低延迟 |

代理场景中推荐使用加密 DNS，防止 DNS 泄露和污染。

---

### QUIC 协议

QUIC（Quick UDP Internet Connections）是基于 UDP 的新一代传输协议。

#### QUIC 特性

```
UDP 基础
    │
    +─── 连接可靠性（类似 TCP）
    │    ├── 连接 ID（支持连接迁移）
    │    ├── 重传和确认
    │    ├── 流量控制
    │    └── 拥塞控制
    │
    +─── TLS 1.3 内置
    │    ├── 加密握手集成
    │    ├── 0-RTT 快速连接
    │    └── 无需额外 TLS 层
    │
    +─── 多路复用
    │    ├── 多个流共享连接
    │    ├── 无队头阻塞（流独立）
    │    └── 流量隔离
    │
    ▼
完整的可靠传输协议
```

#### QUIC vs TCP+TLS

| 特性 | TCP + TLS 1.3 | QUIC |
|------|---------------|------|
| 连接建立 | 2-RTT（首次） | 1-RTT（首次） |
| 恢复连接 | 1-RTT | 0-RTT |
| 队头阻塞 | 有（TCP 层） | 无（流独立） |
| 连接迁移 | 不支持（IP绑定） | 支持（Connection ID） |
| 加密 | 外层 TLS | 内置 TLS 1.3 |
| 拥塞控制 | 内核实现 | 用户态实现（可定制） |

#### QUIC 在代理中的应用

部分代理协议基于 QUIC：

- **Hysteria**: 基于 QUIC 的高性能代理
- **NaiveProxy**: 使用 QUIC 作为传输层
- **TUIC**: 专为代理设计的 QUIC 协议

优势：
- 低延迟连接建立
- 无队头阻塞（多请求并行）
- 连接迁移（网络切换无缝）

---

## 使用示例

### SOCKS5 UDP 代理配置

```json
// configuration.json 中启用 UDP
{
    "protocol": {
        "socks5": {
            "enable_tcp": true,
            "enable_udp": true,
            "udp_bind_port": 0,         // 自动分配
            "udp_idle_timeout": 60,     // 60秒超时
            "udp_max_datagram": 65535   // 最大数据报
        }
    }
}
```

### Trojan UDP over TLS 配置

```json
{
    "protocol": {
        "trojan": {
            "enable_tcp": true,
            "enable_udp": true,
            "udp_idle_timeout": 60
        }
    }
}
```

### UDP 会话超时处理

```cpp
// 协程中处理 UDP 会话超时
net::awaitable<void> udp_session_timeout_handler() {
    auto timer = net::steady_timer(co_await net::this_coro::executor);
    
    while (true) {
        timer.expires_after(std::chrono::seconds(30));
        co_await timer.async_wait(net::use_awaitable);
        
        // 清理过期会话
        session_manager.cleanup_expired(60);
    }
}
```

### UDP 数据报转发

```cpp
/**
 * @brief UDP 数据报转发协程
 * @param client_endpoint 客户端地址
 * @param datagram UDP 数据报
 */
net::awaitable<void> forward_udp_datagram(udp::endpoint client_endpoint, udp_datagram datagram) {
    // 1. 解析目标地址
    auto target = co_await resolve(datagram.address, datagram.port);
    
    // 2. 建立上游 UDP socket
    auto upstream_socket = udp::socket(co_await net::this_coro::executor);
    upstream_socket.open(target.protocol());
    
    // 3. 发送到目标
    co_await upstream_socket.async_send_to(
        net::buffer(datagram.data),
        target,
        net::use_awaitable
    );
    
    // 4. 接收响应
    std::array<uint8_t, 65535> buffer;
    auto [response, source] = co_await upstream_socket.async_receive_from(
        net::buffer(buffer),
        net::use_awaitable
    );
    
    // 5. 构建响应数据报
    auto response_datagram = build_udp_datagram(datagram.address, datagram.port, response);
    
    // 6. 发回客户端
    co_await client_socket.async_send_to(
        net::buffer(response_datagram),
        client_endpoint,
        net::use_awaitable
    );
}
```

---

## 最佳实践

### UDP 代理配置建议

| 参数 | 建议值 | 原因 |
|------|--------|------|
| `udp_idle_timeout` | 60-120 秒 | 平衡 NAT 保活和资源释放 |
| `udp_max_datagram` | 65535 | UDP 最大值，适应大包 |
| `udp_bind_port` | 0（自动） | 避免端口冲突 |

### UDP 保活策略

```cpp
// DNS 查询保活（每 30 秒）
net::awaitable<void> dns_keepalive() {
    auto timer = net::steady_timer(co_await net::this_coro::executor);
    
    while (true) {
        timer.expires_after(std::chrono::seconds(30));
        co_await timer.async_wait(net::use_awaitable);
        
        // 发送保活包（空 DNS 查询或 NOP）
        co_await send_keepalive_packet();
    }
}
```

### UDP 会话管理优化

```cpp
// 使用哈希表快速查找
class udp_session_table {
    // O(1) 查找
    std::unordered_map<std::string, udp_session> table_;
    
    // 批量清理减少遍历
    void cleanup_expired(uint32_t timeout) {
        auto deadline = steady_clock::now() - seconds(timeout);
        
        // 使用 erase_if（C++20）
        std::erase_if(table_, [&](const auto& pair) {
            return pair.second.last_active < deadline;
        });
    }
};
```

### 避免 UDP 分片

SOCKS5 UDP 支持 FRAG 分片，但实践中不建议使用：

- 大多数实现不支持分片重组
- 分片增加复杂性
- 大数据报可使用 TCP 替代

```cpp
// FRAG 必须为 0
void validate_udp_header(udp_header header) {
    if (header.frag != 0) {
        throw protocol_error("UDP fragmentation not supported");
    }
}
```

---

## 常见问题

### Q1: UDP 代理为什么需要 TCP 控制连接？

**A**: SOCKS5 UDP ASSOCIATE 设计如此：
- TCP 控制连接用于认证和建立 UDP 代理能力
- UDP 数据传输不需要认证（依赖已建立的信任）
- TCP 连接断开时 UDP 代理自动终止，防止滥用

### Q2: UDP NAT 映射多久过期？

**A**: 不同 NAT 设备超时不同：
- 家用路由器：通常 30-60 秒
- 企业防火墙：可能 120-300 秒
- 建议：每 30 秒发送保活包维持映射

### Q3: 如何处理 UDP 包大小超过限制？

**A**: 
- 使用 EDNS0 扩展 DNS 包大小（最大 4096B）
- 对于更大的数据，切换到 TCP
- 代理协议通常有 `udp_max_datagram` 配置限制

### Q4: UDP 数据丢失怎么办？

**A**: UDP 本身不保证可靠性：
- 应用层需要重试机制
- DNS 查询失败自动重试 2-3 次
- 关键数据使用 TCP 或 QUIC

### Q5: UDP 代理性能如何优化？

**A**: 
- 使用会话哈希表 O(1) 查找
- 批量处理数据报减少系统调用
- 零拷贝路径减少内存分配
- 多路复用时使用 Parcel 封装

---

## 排障指南

### 问题：UDP 数据无法发送

**症状**: UDP ASSOCIATE 成功但数据无法到达目标

**排查步骤**:

1. 检查 UDP 端口绑定
   ```bash
   netstat -an | grep <port>
   ```

2. 检查防火墙规则
   ```bash
   iptables -L -n | grep udp
   ```

3. 检查 NAT 映射
   - 是否有保活包维持映射
   - NAT 超时是否过短

4. 使用抓包工具验证
   ```bash
   tcpdump -i any udp port <port>
   ```

---

### 问题：UDP 响应无法接收

**症状**: 发送成功但收不到响应

**排查步骤**:

1. 检查目标是否可达
   ```bash
   ping <target>
   ```

2. 检查会话映射
   - 是否正确记录源地址
   - 响应是否能匹配映射

3. 检查 NAT 类型
   - Symmetric NAT 需要特殊处理

---

### 问题：UDP 数据报解析错误

**症状**: 解析失败，地址/端口错误

**排查步骤**:

1. 检查数据报格式
   - 是否符合 SOCKS5 UDP 头规范
   - ATYP 字段是否正确

2. 检查数据完整性
   - 数据报是否被截断
   - 长度字段是否匹配

3. 检查编码实现
   - IPv4/IPv6/域名编码是否正确

---

### 问题：多路复用 UDP 性能差

**症状**: Parcel 封装后吞吐量下降

**排查步骤**:

1. 检查 TCP 连接性能
   - TCP 是否有拥塞
   - 窗口大小是否足够

2. 检查 Parcel 封装开销
   - 封装层是否正确解析
   - 是否有不必要的拷贝

3. 优化建议：
   - 增大 yamux 窗口
   - 批量发送多个 Parcel

---

## 相关链接

- [[modules/channel]] — 通道模块与连接管理
- [[modules/protocol]] — 协议模块概览
- [[modules/multiplex]] — 多路复用模块
- [[dev/tcp]] — TCP 协议实践
- [[configuration]] — UDP 配置参数