---
title: "DNS over UDP 协议"
category: "protocol"
type: ref
module: ref
source: "RFC 1035"
tags: [协议, dns, udp, 解析, 查询, 响应]
created: 2026-05-15
updated: 2026-05-17
related: [dns-over-tcp, dns-over-tls, dns-over-https, dns-basics]
---

# DNS over UDP 协议

**类别**: 协议

## 概述

DNS over UDP 是传统的 DNS 查询传输方式，由 RFC 1035（1987 年）定义。DNS over UDP 使用 UDP 协议作为传输层，默认端口为 53。UDP 作为无连接协议，提供了低延迟、低开销的 DNS 查询服务，是 DNS 协议最广泛使用的传输方式。

DNS over UDP 的特点：

- **低延迟**：无需建立连接，直接发送查询
- **低开销**：UDP 头部仅 8 字节，数据包小
- **单次查询**：每个 UDP 数据包承载一个查询
- **无序保证**：依赖应用层匹配查询和响应（通过 ID）
- **重传机制**：应用层实现超时重传

DNS over UDP 在域名解析基础设施中占据核心地位：

- **递归解析器**：向权威服务器查询使用 UDP
- **客户端解析**：操作系统 DNS 客户端默认使用 UDP
- **缓存服务器**：DNS 缓存服务主要使用 UDP

在 Prism 项目中，DNS over UDP 是 Resolve 模块的核心传输方式，用于向 DNS 上游服务器发送查询请求。

### DNS 协议发展

DNS 协议的发展历程：

- **RFC 882/883** (1983)：最初的 DNS 规范
- **RFC 1034/1035** (1987)：DNS 标准规范
- **RFC 6891** (2013)：EDNS0 扩展
- **RFC 7858** (2016)：DNS over TLS
- **RFC 8484** (2018)：DNS over HTTPS

### UDP vs TCP DNS

UDP 与 TCP DNS 的对比：

| 特性 | UDP | TCP |
|------|-----|-----|
| 连接 | 无连接 | 需要建立连接 |
| 头部开销 | 8 bytes | 20+ bytes |
| 延迟 | 低 | 较高（握手开销） |
| 数据包大小 | 传统 512，EDNS0 4096 | 无限制 |
| 可靠性 | 应用层保证 | TCP 保证 |
| 适用场景 | 大多数查询 | 大响应、DNSSEC |

### UDP DNS 的局限性

UDP DNS 的主要限制：

1. **数据包大小**：传统限制 512 字节
2. **无可靠性**：可能丢包，需要重传
3. **无加密**：查询和响应明文传输
4. **易伪造**：可能遭受 DNS 欺骗攻击

## 协议详解

### DNS 报文格式

DNS 报文的完整二进制结构：

```
DNS Message:
┌────────────────────────────────────────────┐
│ Header: 12 bytes                           │
│   ID: 2 bytes                              │
│   Flags: 2 bytes                           │
│   QDCOUNT: 2 bytes                         │
│   ANCOUNT: 2 bytes                         │
│   NSCOUNT: 2 bytes                         │
│   ARCOUNT: 2 bytes                         │
├────────────────────────────────────────────┤
│ Question Section: variable                 │
├────────────────────────────────────────────┤
│ Answer Section: variable                   │
├────────────────────────────────────────────┤
│ Authority Section: variable                │
├────────────────────────────────────────────┤
│ Additional Section: variable               │
└────────────────────────────────────────────┘
```

#### Header 部分

DNS Header 的详细结构：

```
Header (12 bytes):
┌────────────────────────────────────────────┐
│ ID: 16 bits                                │
│   - 查询标识符                              │
│   - 客户端随机生成                          │
│   - 响应必须匹配                            │
├────────────────────────────────────────────┤
│ Flags: 16 bits                             │
│   QR: 1 bit (0=Query, 1=Response)         │
│   OPCODE: 4 bits (操作码)                  │
│   AA: 1 bit (权威回答)                      │
│   TC: 1 bit (截断标志)                      │
│   RD: 1 bit (期望递归)                      │
│   RA: 1 bit (递归可用)                      │
│   Z: 3 bits (保留)                          │
│   RCODE: 4 bits (响应码)                   │
├────────────────────────────────────────────┤
│ QDCOUNT: 16 bits (问题数)                  │
├────────────────────────────────────────────┤
│ ANCOUNT: 16 bits (回答数)                  │
├────────────────────────────────────────────┤
│ NSCOUNT: 16 bits (权威记录数)               │
├────────────────────────────────────────────┤
│ ARCOUNT: 16 bits (附加记录数)               │
└────────────────────────────────────────────┘
```

#### Flags 详细解释

```
Flags 字段详解：

QR (Query/Response):
┌────────────────────────────────────────────┐
│ 0: 查询消息                                 │
│ 1: 响应消息                                 │
└────────────────────────────────────────────┘

OPCODE:
┌────────────────────────────────────────────┐
│ 0: Standard Query (QUERY)                  │
│ 1: Inverse Query (IQUERY) - 已废弃         │
│ 2: Server Status Request (STATUS)          │
│ 4: Notify                                  │
│ 5: Update                                  │
└────────────────────────────────────────────┘

AA (Authoritative Answer):
┌────────────────────────────────────────────┐
│ 0: 非权威回答                               │
│ 1: 权威回答（来自权威服务器）                │
└────────────────────────────────────────────┘

TC (Truncation):
┌────────────────────────────────────────────┐
│ 0: 未截断                                   │
│ 1: 截断（响应超过 UDP 限制）                │
│    → 客户端应使用 TCP 重试                  │
└────────────────────────────────────────────┘

RD (Recursion Desired):
┌────────────────────────────────────────────┐
│ 0: 不期望递归                               │
│ 1: 期望递归解析                             │
│    （客户端请求服务器递归查询）              │
└────────────────────────────────────────────┘

RA (Recursion Available):
┌────────────────────────────────────────────┐
│ 0: 服务器不支持递归                         │
│ 1: 服务器支持递归                           │
│    （响应中设置）                            │
└────────────────────────────────────────────┘

RCODE (Response Code):
┌────────────────────────────────────────────┐
│ 0: NoError (成功)                           │
│ 1: FormErr (格式错误)                       │
│ 2: ServFail (服务器失败)                    │
│ 3: NXDomain (域名不存在)                    │
│ 4: NotImp (未实现)                          │
│ 5: Refused (拒绝)                           │
│ 6-15: 其他错误                              │
└────────────────────────────────────────────┘
```

#### Question 部分

Question Section 的结构：

```
Question Entry:
┌────────────────────────────────────────────┐
│ QNAME: variable (域名)                      │
│   - 使用 DNS 标签格式                        │
│   - 以 0x00 结束                            │
├────────────────────────────────────────────┤
│ QTYPE: 2 bytes (查询类型)                  │
├────────────────────────────────────────────┤
│ QCLASS: 2 bytes (查询类)                   │
└────────────────────────────────────────────┘

QTYPE 常用值：
┌────────────────────────────────────────────┐
│ 1: A (IPv4 地址)                            │
│ 2: NS (名称服务器)                          │
│ 5: CNAME (别名)                             │
│ 6: SOA (起始授权)                           │
│ 12: PTR (指针)                              │
│ 15: MX (邮件交换)                           │
│ 16: TXT (文本)                              │
│ 28: AAAA (IPv6 地址)                        │
│ 33: SRV (服务)                              │
│ 41: OPT (EDNS0)                             │
│ 255: ANY (任意)                             │
└────────────────────────────────────────────┘

QCLASS 常用值：
┌────────────────────────────────────────────┐
│ 1: IN (Internet)                            │
│ 2: CS (CSNET) - 已废弃                      │
│ 3: CH (CHAOS)                               │
│ 4: HS (Hesiod)                              │
│ 255: ANY (任意)                             │
└────────────────────────────────────────────┘
```

#### DNS 标签格式

DNS 域名使用特殊的标签格式编码：

```
DNS Label Format:

域名: www.example.com
编码: 03 77 77 77 07 65 78 61 6D 70 6C 65 03 63 6F 6D 00
       │  └─────────  └─────────────────  └───────  │
       │  "www"        "example"          "com"     │
       │  (3 bytes)    (7 bytes)         (3 bytes) │
       │                                            │
       └── 长度字节                              结束符 0x00

压缩格式 (Message Compression):
- 如果域名在消息中已出现，可使用指针引用
- 指针格式: 2 bytes，最高两位为 11
- 例如: C0 0C = 指向偏移 12 的位置
```

#### Resource Record 部分

Answer、Authority、Additional 使用 Resource Record 格式：

```
Resource Record:
┌────────────────────────────────────────────┐
│ NAME: variable (域名)                       │
│   - 可使用压缩指针                           │
├────────────────────────────────────────────┤
│ TYPE: 2 bytes (记录类型)                    │
├────────────────────────────────────────────┤
│ CLASS: 2 bytes (记录类)                     │
├────────────────────────────────────────────┤
│ TTL: 4 bytes (生存时间)                     │
├────────────────────────────────────────────┤
│ RDLENGTH: 2 bytes (数据长度)                │
├────────────────────────────────────────────┤
│ RDATA: variable (记录数据)                  │
└────────────────────────────────────────────┘

A 记录 RDATA:
┌────────────────────────────────────────────┐
│ IPv4 Address: 4 bytes                      │
│ 例: 192.168.1.1 → C0 A8 01 01             │
└────────────────────────────────────────────┘

AAAA 记录 RDATA:
┌────────────────────────────────────────────┐
│ IPv6 Address: 16 bytes                     │
└────────────────────────────────────────────┘

CNAME 记录 RDATA:
┌────────────────────────────────────────────┐
│ Target Domain: variable (DNS 标签格式)      │
└────────────────────────────────────────────┘

MX 记录 RDATA:
┌────────────────────────────────────────────┐
│ Preference: 2 bytes                        │
│ Exchange: variable (邮件服务器域名)          │
└────────────────────────────────────────────┘
```

### EDNS0 扩展

EDNS0 (Extension Mechanisms for DNS) 扩展 UDP 数据包大小：

```
EDNS0 OPT Record (TYPE = 41):
┌────────────────────────────────────────────┐
│ NAME: 0x00 (必须为根域名)                    │
│ TYPE: 0x0029 (41 = OPT)                    │
│ CLASS: UDP Payload Size (最大接收大小)      │
│   - 传统: 512                              │
│   - EDNS0: 通常 4096                        │
│ TTL: Extended RCODE + Flags                │
│   - 高 8 bits: Extended RCODE              │
│   - 中 8 bits: Version (0)                 │
│   - 低 16 bits: Flags                      │
│     - DO bit: DNSSEC OK                    │
│ RDLENGTH: variable                         │
│ RDATA: Options                             │
└────────────────────────────────────────────┘

客户端 EDNS0 请求示例：
- CLASS: 4096 (最大可接收 4096 字节)
- DO bit: 1 (请求 DNSSEC 数据)
```

### UDP 数据包大小限制

UDP DNS 数据包大小：

```
大小限制：
┌────────────────────────────────────────────┐
│ 传统限制: 512 bytes                         │
│   - RFC 1035 定义                           │
│   - 防止 IP 分片                            │
│                                            │
│ EDNS0 扩展: 最大 4096 bytes                │
│   - RFC 6891 定义                           │
│   - 通过 OPT 记录协商                       │
│                                            │
│ 实际限制:                                   │
│   - 考虑 IP/UDP 头部 (20+8=28 bytes)        │
│   - IPv6 头部更大 (40+8=48 bytes)          │
│   - 避免超过 MTU                            │
└────────────────────────────────────────────┘

截断处理：
┌────────────────────────────────────────────┐
│ 响应超过限制时:                              │
│ 1. 设置 TC = 1                             │
│ 2. 返回部分数据（Header + Question）        │
│ 3. 客户端收到 TC=1 后:                      │
│    - 使用 TCP 重试                          │
│    - 或接受部分数据                          │
└────────────────────────────────────────────┘
```

### 查询响应流程

UDP DNS 查询响应流程：

```
UDP DNS 流程：

Client                    DNS Server
  |                           |
  |--- UDP Query ----------->|
  |   ID: 12345              |
  |   QR: 0 (Query)          |
  |   RD: 1 (期望递归)        |
  |   QNAME: example.com      |
  |   QTYPE: A                |
  |   QCLASS: IN              |
  |                           |
  |<-- UDP Response ----------|
  |   ID: 12345 (必须匹配)    |
  |   QR: 1 (Response)        |
  |   AA: 0 (非权威)          |
  |   RA: 1 (支持递归)        |
  |   RCODE: 0 (成功)         |
  |   ANCOUNT: 1              |
  |   Answer:                 |
  |     example.com → A       |
  |     TTL: 3600             |
  |     93.184.216.34         |
  |                           |

超时重传：
┌────────────────────────────────────────────┐
│ 默认超时: 2-5 秒                            │
│ 重传次数: 通常 2-3 次                       │
│ 重传间隔: 递增（指数退避）                   │
└────────────────────────────────────────────┘
```

### UDP 端口和传输

UDP DNS 的端口和传输：

```
端口：
┌────────────────────────────────────────────┐
│ 标准端口: 53                                │
│ 客户端: 随机端口                            │
│ 服务器: 53                                  │
└────────────────────────────────────────────┘

传输特性：
┌────────────────────────────────────────────┐
│ 无连接: 直接发送数据包                      │
│ 单次查询: 每个包一个查询                    │
│ ID 匹配: 通过 ID 区分不同查询               │
│ 超时重传: 应用层实现                        │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### DNS 报文解析

Prism 解析 DNS 报文：

```cpp
// 文件: src/prism/resolve/dns/format.hpp
namespace psm::resolve::dns {

auto parse_message(std::span<const std::byte> data)
    -> outcome::result<message>
{
    if (data.size() < 12) {
        return fault::code::insufficient_data;
    }
    
    message msg;
    
    // 解析 Header
    msg.id = read_u16(data, 0);
    msg.flags = parse_flags(read_u16(data, 2));
    msg.qdcount = read_u16(data, 4);
    msg.ancount = read_u16(data, 6);
    msg.nscount = read_u16(data, 8);
    msg.arcount = read_u16(data, 10);
    
    size_t offset = 12;
    
    // 解析 Question
    for (uint16_t i = 0; i < msg.qdcount; ++i) {
        auto [name, new_offset] = parse_name(data, offset);
        auto qtype = read_u16(data, new_offset);
        auto qclass = read_u16(data, new_offset + 2);
        
        msg.questions.push_back({
            .name = name,
            .type = qtype,
            .class = qclass
        });
        
        offset = new_offset + 4;
    }
    
    // 解析 Answer
    for (uint16_t i = 0; i < msg.ancount; ++i) {
        auto rr = parse_resource_record(data, offset);
        msg.answers.push_back(rr.value());
        offset = rr.offset;
    }
    
    return msg;
}

} // namespace psm::resolve::dns
```

### DNS 报文构造

Prism 构造 DNS 查询报文：

```cpp
// 文件: src/prism/resolve/dns/format.hpp
auto craft_query(std::string_view domain,
                 uint16_t qtype,
                 uint16_t qclass,
                 uint16_t id,
                 bool recursion_desired)
    -> std::vector<std::byte>
{
    std::vector<std::byte> buffer;
    
    // Header
    append_u16(buffer, id);
    
    uint16_t flags = 0;
    if (recursion_desired) {
        flags |= 0x0100; // RD bit
    }
    append_u16(buffer, flags);
    
    append_u16(buffer, 1);    // QDCOUNT
    append_u16(buffer, 0);    // ANCOUNT
    append_u16(buffer, 0);    // NSCOUNT
    append_u16(buffer, 0);    // ARCOUNT
    
    // Question
    append_name(buffer, domain);
    append_u16(buffer, qtype);
    append_u16(buffer, qclass);
    
    return buffer;
}
```

### UDP DNS 查询

Prism 的 UDP DNS 查询实现：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto udp_query(const dns_config& config,
               const endpoint& server,
               std::span<const std::byte> query)
    -> net::awaitable<result<message>>
{
    auto executor = co_await net::this_coroutine::executor;
    
    // 创建 UDP socket
    net::udp::socket socket(executor);
    socket.open(net::udp::v4());
    
    // 设置超时
    net::steady_timer timer(executor);
    timer.expires_after(std::chrono::milliseconds(config.timeout_ms));
    
    // 发送查询
    co_await socket.async_send_to(
        net::buffer(query),
        server,
        net::use_awaitable);
    
    // 接收响应（带超时）
    std::array<std::byte, 4096> buffer;
    net::udp::endpoint sender;
    
    auto result = co_await (
        socket.async_receive_from(
            net::buffer(buffer),
            sender,
            net::use_awaitable)
        || timer.async_wait(net::use_awaitable)
    );
    
    if (result.index() == 1) {
        // 超时
        co_return result::timeout;
    }
    
    auto bytes_received = std::get<0>(result);
    auto response = parse_message(buffer.data(), bytes_received);
    
    // 验证 ID 匹配
    if (response.id != query_id) {
        co_return result::id_mismatch;
    }
    
    // 检查截断
    if (response.flags.tc) {
        // 响应截断，需要 TCP 重试
        co_return result::truncated;
    }
    
    co_return response.value();
}
```

### EDNS0 支持

Prism 的 EDNS0 实现：

```cpp
// 文件: src/prism/resolve/dns/format.hpp
auto append_edns0(std::vector<std::byte>& buffer,
                  uint16_t payload_size,
                  bool dnssec_ok)
    -> void
{
    // OPT 记录
    append_u8(buffer, 0);       // NAME: root
    append_u16(buffer, 41);     // TYPE: OPT
    append_u16(buffer, payload_size); // CLASS: payload size
    
    uint32_t ttl = 0;
    if (dnssec_ok) {
        ttl |= 0x8000; // DO bit
    }
    append_u32(buffer, ttl);
    append_u16(buffer, 0);      // RDLENGTH: no options
}
```

## 最佳实践

### 超时配置

1. **初始超时**：2-5 秒
2. **重传次数**：2-3 次
3. **超时递增**：指数退避（如 2s → 4s → 8s）
4. **总超时限制**：30 秒

### 数据包大小

1. **默认发送**：512 字节以下
2. **EDNS0 接收**：支持 4096 字节
3. **检测截断**：处理 TC 标志
4. **TCP fallback**：截断时自动切换 TCP

### 安全考虑

1. **随机 ID**：使用随机 ID 防止欺骗
2. **源端口随机**：随机源端口增加安全性
3. **DNSSEC 支持**：支持 DNSSEC 验证
4. **加密方案**：考虑 DoT/DoH 加密

### 缓存优化

1. **TTL 遵守**：正确处理 TTL
2. **负缓存**：缓存 NXDomain 等错误响应
3. **缓存预取**：接近过期时预取
4. **缓存大小**：设置合理的缓存大小

## 常见问题

### Q1: UDP DNS 为什么有 512 字节限制？

512 字节限制的原因：

- **避免分片**：超过可能触发 IP 分片
- **历史原因**：1987 年网络环境保守
- **兼容性**：确保所有 DNS 实现兼容

### Q2: 如何处理截断响应（TC=1）？

截断响应的处理：

1. 检查响应中的 TC 标志
2. 如果 TC=1，使用 TCP 重试
3. TCP 查询使用相同 ID 和内容
4. TCP 响应不会有截断问题

### Q3: EDNS0 如何协商更大的数据包？

EDNS0 协商过程：

1. 客户端在 Additional 中添加 OPT 记录
2. OPT CLASS 设置最大接收大小
3. 服务器收到后可发送更大响应
4. 服务器也可能在响应中包含 OPT 记录

### Q4: UDP DNS 容易遭受什么攻击？

UDP DNS 的安全风险：

- **DNS 欺骗**：伪造响应（通过随机 ID 和端口缓解）
- **DNS 放大攻击**：利用 DNS 服务器反射流量
- **查询泄露**：查询内容被窃听（使用 DoT/DoH）
- **中间人攻击**：修改响应内容

### Q5: Prism 如何处理 UDP DNS 超时？

Prism 的超时处理：

1. 发送查询后启动定时器
2. 定时器到期时检查响应
3. 未收到响应则重传或返回超时
4. 使用协程实现异步超时

### Q6: DNS ID 为什么需要随机？

随机 ID 的安全意义：

- 防止攻击者伪造匹配 ID 的响应
- 增加欺骗攻击的难度
- 16 位 ID + 随机端口 = 更高安全性

### Q7: UDP 和 TCP DNS 如何选择？

选择原则：

- **小查询/响应**：UDP（大多数情况）
- **大响应**：TCP（截断后）
- **DNSSEC**：可能需要 TCP
- **安全要求高**：考虑 DoT/DoH

### Q8: 如何优化 UDP DNS 性能？

性能优化建议：

- **并行查询**：同时查询多个服务器
- **缓存命中**：优先检查缓存
- **预取机制**：接近过期时预取
- **EDNS0 支持**：减少 TCP fallback

## 参考资料

- [RFC 1035 - Domain Names: Implementation and Specification](https://tools.ietf.org/html/rfc1035)
- [RFC 6891 - Extension Mechanisms for DNS (EDNS0)](https://tools.ietf.org/html/rfc6891)
- [RFC 7766 - DNS Transport over TCP](https://tools.ietf.org/html/rfc7766)

## 相关知识

- [[ref/protocol/dns-over-tcp|DNS over TCP]] — TCP DNS 查询
- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT 加密 DNS
- [[ref/protocol/dns-over-https|DNS over HTTPS]] — DoH 加密 DNS
- [[ref/resolve/dns-basics|DNS 基础]] — DNS 协议基础