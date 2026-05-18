---
title: "DNS over TCP 协议"
category: "protocol"
type: ref
module: ref
source: "RFC 7766"
tags: [协议, dns, tcp, 解析, 查询, 响应, 大响应]
created: 2026-05-15
updated: 2026-05-17
related: [dns-over-udp, dns-over-tls, dns-over-https, dns-basics]
layer: ref
---

# DNS over TCP 协议

**类别**: 协议

## 概述

DNS over TCP 是 DNS 协议的 TCP 传输方式，由 RFC 7766（2016 年）规范化，是对 RFC 1035 中 TCP DNS 的扩展和更新。DNS over TCP 使用 TCP 协议作为传输层，默认端口为 53。相比 UDP，TCP 提供了可靠传输、流控制和更大的数据包容量，适用于大响应查询、DNSSEC 和需要可靠性的场景。

DNS over TCP 的主要用途：

- **大响应传输**：当 DNS 响应超过 UDP 限制时
- **DNSSEC 支持**：DNSSEC 记录通常较大
- **AXFR/IXFR**：DNS 区域传输
- **可靠性保证**：需要可靠传输的场景

在 Prism 项目中，DNS over TCP 是 Resolve 模块的备选传输方式，用于处理截断响应和需要大响应的查询。

### TCP DNS vs UDP DNS

TCP 与 UDP DNS 的详细对比：

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接建立 | 需要 TCP 三次握手 | 无连接 |
| 延迟 | 较高（握手开销） | 低 |
| 头部开销 | 20+ bytes | 8 bytes |
| 数据包大小 | 无限制（理论上） | 512/4096 bytes |
| 可靠性 | TCP 内置保证 | 应用层实现 |
| 流控制 | TCP 内置支持 | 无 |
| 拥塞控制 | TCP 内置支持 | 无 |
| 适用场景 | 大响应、可靠性 | 一般查询 |
| 连接复用 | 可以复用连接 | 无连接 |

### TCP DNS 的必要性

TCP DNS 必要的场景：

```
需要 TCP DNS 的场景：
┌────────────────────────────────────────────┐
│ 1. 响应超过 UDP 限制                        │
│    - TC=1 截断响应                          │
│    - EDNS0 超过协商大小                     │
│                                            │
│ 2. DNSSEC 记录                              │
│    - DNSKEY、RRSIG 等较大                   │
│    - 需要可靠传输                           │
│                                            │
│ 3. 区域传输 (AXFR/IXFR)                     │
│    - 需要传输整个区域                       │
│    - 仅支持 TCP                             │
│                                            │
│ 4. 需要可靠性保证                           │
│    - 网络不稳定                             │
│    - 重要查询                               │
└────────────────────────────────────────────┘
```

### TCP DNS 的历史

TCP DNS 的发展：

- **RFC 1035** (1987)：最初定义 TCP DNS
- **RFC 7766** (2016)：现代化 TCP DNS 规范
- **连接复用**：明确允许连接复用
- **流水线**：支持查询流水线

## 协议详解

### TCP DNS 报文格式

TCP DNS 报文在 UDP 格式基础上添加长度前缀：

```
TCP DNS Message:
┌────────────────────────────────────────────┐
│ Length: 2 bytes (消息长度)                  │
│ DNS Message: variable                       │
│   Header: 12 bytes                         │
│   Question: variable                       │
│   Answer: variable                         │
│   Authority: variable                      │
│   Additional: variable                     │
└────────────────────────────────────────────┘

长度字段：
┌────────────────────────────────────────────┐
│ 2 bytes，大端序                             │
│ 表示后续 DNS 消息的字节数                   │
│ 不包括这 2 字节本身                         │
│                                            │
│ 最大值: 65535 bytes                         │
│ 实际应用通常不超过几千字节                   │
└────────────────────────────────────────────┘

示例：
消息长度: 100 bytes
编码: 00 64 [100 bytes DNS message]
```

### TCP 连接流程

TCP DNS 的完整连接流程：

```
TCP DNS 流程：

Client                    DNS Server
  |                           |
  |--- TCP SYN -------------->|  (TCP 三次握手)
  |<-- TCP SYN+ACK -----------|
  |--- TCP ACK -------------->|
  |                           |
  |   (连接建立完成)           |
  |                           |
  |--- Query #1 ------------->|
  |   Length: 00 30          |
  |   [DNS Query Message]     |
  |                           |
  |--- Query #2 ------------->|  (流水线发送)
  |   Length: 00 30          |
  |   [DNS Query Message]     |
  |                           |
  |<-- Response #1 -----------|
  |   Length: 00 50          |
  |   [DNS Response Message]  |
  |                           |
  |<-- Response #2 -----------|
  |   Length: 00 50          |
  |   [DNS Response Message]  |
  |                           |
  |--- TCP FIN -------------->|  (连接关闭)
  |<-- TCP FIN+ACK -----------|
  |--- TCP ACK -------------->|
```

### 查询流水线

RFC 7766 允许 TCP DNS 查询流水线：

```
流水线机制：
┌────────────────────────────────────────────┐
│ 1. 客户端可以在收到响应前发送多个查询        │
│                                            │
│ 2. 服务器按照查询顺序返回响应               │
│                                            │
│ 3. 响应顺序与查询顺序匹配                   │
│                                            │
│ 4. 通过 ID 区分不同的查询                   │
│                                            │
│ 5. 实现要求：                               │
│    - 服务器必须按顺序响应                   │
│    - 客户端可以不等待响应                   │
└────────────────────────────────────────────┘

流水线优势：
- 减少往返延迟
- 提高查询效率
- 充分利用 TCP 连接
```

### 连接复用

TCP DNS 支持连接复用：

```
连接复用策略：
┌────────────────────────────────────────────┐
│ 1. 建立连接后可以发送多个查询                │
│                                            │
│ 2. 不必为每个查询建立新连接                  │
│                                            │
│ 3. 连接可以保持一段时间                      │
│                                            │
│ 4. 实现建议：                               │
│    - 客户端可保持连接池                     │
│    - 设置合理的空闲超时                     │
│    - 监控连接健康状态                       │
└────────────────────────────────────────────┘

复用优势：
- 减少握手开销
- 提高整体性能
- 降低服务器压力
```

### 区域传输 (AXFR/IXFR)

DNS 区域传输使用 TCP：

```
AXFR (完整区域传输):
┌────────────────────────────────────────────┐
│ Client                    Server           │
│   |                         |               │
│   |--- AXFR Query ---------->|              │
│   |   QTYPE: 252 (AXFR)     |               │
│   |                         |               │
│   |<-- Response #1 ----------|              │
│   |   SOA (起始记录)         |               │
│   |                         |               │
│   |<-- Response #2 ----------|              │
│   |   Resource Record        |               │
│   |                         |               │
│   |<-- Response #3 ----------|              │
│   |   Resource Record        |               │
│   |   ...                    |               │
│   |                         |               │
│   |<-- Response Final -------|              │
│   |   SOA (重复，表示结束)   |               │
└────────────────────────────────────────────┘

IXFR (增量区域传输):
┌────────────────────────────────────────────┐
│ Client                    Server           │
│   |                         |               │
│   |--- IXFR Query ---------->|              │
│   |   QTYPE: 251 (IXFR)     |               │
│   |   包含旧 SOA 序列号       |               │
│   |                         |               │
│   |<-- Response -------------|              │
│   |   增量变更记录           |               │
│   |   (仅变更部分)           |               │
└────────────────────────────────────────────┘
```

### 截断响应处理

UDP 截断响应后的 TCP fallback：

```
TCP Fallback 流程：

Client                    DNS Server
  |                           |
  |--- UDP Query ------------>|
  |                           |
  |<-- UDP Response ----------|
  |   TC: 1 (截断)            |
  |   (只有 Header + Question)|
  |                           |
  |   客户端检测到 TC=1       |
  |                           |
  |--- TCP SYN -------------->|
  |<-- TCP SYN+ACK -----------|
  |--- TCP ACK -------------->|
  |                           |
  |--- TCP Query ------------>|  (相同查询)
  |                           |
  |<-- TCP Response ----------|
  |   (完整响应，无截断)       |
  |                           |

注意：
┌────────────────────────────────────────────┐
│ - TCP 查询使用相同 ID                       │
│ - TCP 查询内容与 UDP 相同                   │
│ - TCP 响应包含完整数据                      │
└────────────────────────────────────────────┘
```

### TCP 连接管理

TCP DNS 连接管理的最佳实践：

```
连接管理建议：
┌────────────────────────────────────────────┐
│ 客户端：                                    │
│ - 使用连接池                                │
│ - 设置空闲超时（建议 10-30 秒）             │
│ - 监控连接状态                              │
│ - 处理连接错误                              │
│                                            │
│ 服务器：                                    │
│ - 设置最大连接数                            │
│ - 设置连接超时                              │
│ - 支持流水线                                │
│ - 响应顺序保证                              │
└────────────────────────────────────────────┘

超时配置：
┌────────────────────────────────────────────┐
│ 连接超时: 5-10 秒                           │
│ 查询超时: 10-30 秒                          │
│ 空闲超时: 10-30 秒                          │
│ 读超时: 根据响应大小调整                    │
└────────────────────────────────────────────┘
```

### 错误处理

TCP DNS 的错误处理：

```
错误类型：
┌────────────────────────────────────────────┐
│ 连接错误：                                  │
│ - 连接失败（服务器不可达）                  │
│ - 连接超时                                  │
│ - 连接被拒绝                                │
│                                            │
│ 传输错误：                                  │
│ - 数据不完整                                │
│ - 长度字段错误                              │
│ - TCP 重置                                  │
│                                            │
│ 协议错误：                                  │
│ - ID 不匹配                                 │
│ - 格式错误                                  │
│ - 服务器错误                                │
└────────────────────────────────────────────┘

处理策略：
┌────────────────────────────────────────────┐
│ 连接失败：                                  │
│ - 尝试其他服务器                            │
│ - 回退到 UDP                                │
│                                            │
│ 传输错误：                                  │
│ - 关闭连接                                  │
│ - 建立新连接重试                            │
│                                            │
│ 协议错误：                                  │
│ - 记录错误                                  │
│ - 返回错误结果                              │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### TCP DNS 查询

Prism 的 TCP DNS 查询实现：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto tcp_query(const dns_config& config,
               const endpoint& server,
               std::span<const std::byte> query)
    -> net::awaitable<result<message>>
{
    auto executor = co_await net::this_coroutine::executor;
    
    // 从连接池获取或建立新连接
    auto connection = co_await get_or_create_connection(server);
    
    // 添加长度前缀
    std::vector<std::byte> tcp_query;
    append_u16(tcp_query, query.size());
    tcp_query.append(query.begin(), query.end());
    
    // 发送查询
    co_await net::async_write(connection.socket,
        net::buffer(tcp_query),
        net::use_awaitable);
    
    // 接收响应长度
    std::array<std::byte, 2> length_buf;
    co_await net::async_read(connection.socket,
        net::buffer(length_buf),
        net::use_awaitable);
    
    auto response_length = read_u16(length_buf, 0);
    
    // 接收响应消息
    std::vector<std::byte> response_buf(response_length);
    co_await net::async_read(connection.socket,
        net::buffer(response_buf),
        net::use_awaitable);
    
    // 解析响应
    auto response = parse_message(response_buf);
    
    // 验证 ID
    if (response.id != query_id) {
        co_return result::id_mismatch;
    }
    
    // 返回连接到池
    co_await return_connection(connection);
    
    co_return response.value();
}
```

### 连接池管理

Prism 的 TCP DNS 连接池：

```cpp
// 文件: src/prism/resolve/dns/pool.cpp
class tcp_connection_pool {
public:
    auto get_or_create(const endpoint& server)
        -> net::awaitable<tcp_connection>
    {
        // 查找现有连接
        auto& connections = pool_[server];
        
        for (auto it = connections.begin(); it != connections.end();) {
            if (it->is_valid() && !it->is_in_use()) {
                it->mark_in_use();
                co_return *it;
            }
            if (!it->is_valid()) {
                it = connections.erase(it);
            } else {
                ++it;
            }
        }
        
        // 创建新连接
        auto new_conn = co_await create_connection(server);
        new_conn.mark_in_use();
        connections.push_back(new_conn);
        
        co_return connections.back();
    }
    
    auto return_connection(tcp_connection& conn)
        -> net::awaitable<void>
    {
        conn.mark_not_in_use();
        conn.update_last_used();
        
        // 检查是否需要清理空闲连接
        cleanup_idle_connections();
    }
    
private:
    std::map<endpoint, std::vector<tcp_connection>> pool_;
};
```

### 截断响应处理

Prism 处理 UDP 截断响应：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto resolve(const dns_config& config,
             const endpoint& server,
             std::string_view domain,
             uint16_t qtype)
    -> net::awaitable<result<message>>
{
    // 首先尝试 UDP
    auto udp_result = co_await udp_query(config, server, domain, qtype);
    
    if (udp_result.status == result::truncated) {
        // UDP 响应截断，切换到 TCP
        auto tcp_result = co_await tcp_query(config, server, domain, qtype);
        co_return tcp_result;
    }
    
    co_return udp_result;
}
```

### 长度解析

Prism 解析 TCP DNS 长度：

```cpp
// 文件: src/prism/resolve/dns/format.hpp
auto parse_tcp_message(net::socket& socket)
    -> net::awaitable<message>
{
    // 读取长度
    std::array<std::byte, 2> length_buf;
    co_await net::async_read(socket,
        net::buffer(length_buf),
        net::transfer_exactly(2),
        net::use_awaitable);
    
    auto length = static_cast<size_t>(
        (static_cast<uint16_t>(length_buf[0]) << 8) |
        static_cast<uint16_t>(length_buf[1])
    );
    
    // 检查长度合理性
    if (length < 12) {
        throw dns_error("invalid message length");
    }
    if (length > 65535) {
        throw dns_error("message too large");
    }
    
    // 读取消息
    std::vector<std::byte> message_buf(length);
    co_await net::async_read(socket,
        net::buffer(message_buf),
        net::transfer_exactly(length),
        net::use_awaitable);
    
    return parse_message(message_buf);
}
```

## 最佳实践

### 连接管理

1. **连接池**：使用连接池复用 TCP 连接
2. **空闲超时**：设置合理的空闲超时（10-30 秒）
3. **最大连接数**：限制每个服务器的连接数
4. **健康检查**：定期检查连接健康状态

### 流水线使用

1. **顺序保证**：确保响应顺序与查询匹配
2. **ID 管理**：正确管理查询 ID
3. **错误隔离**：一个查询错误不影响其他
4. **实现复杂度**：根据需要选择是否支持

### 截断处理

1. **及时检测**：立即检测 TC 标志
2. **快速切换**：快速切换到 TCP
3. **相同查询**：TCP 查询使用相同内容
4. **结果验证**：验证 TCP 响应完整性

### 性能优化

1. **延迟 TCP**：优先使用 UDP，必要时才用 TCP
2. **预判截断**：已知大响应时直接用 TCP
3. **并行查询**：流水线并行发送查询
4. **缓存利用**：减少需要 TCP 的情况

## 常见问题

### Q1: 什么时候应该使用 TCP DNS？

使用 TCP DNS 的时机：

- UDP 响应截断（TC=1）
- 需要大响应（如 DNSSEC）
- 区域传输（AXFR/IXFR）
- 网络不稳定需要可靠传输

### Q2: TCP DNS 的 2 字节长度字段有什么作用？

长度字段的作用：

- 标识消息边界
- 支持多个消息在同一连接
- 区分不同的查询/响应
- 消息完整性检查

### Q3: 如何处理 TCP DNS 连接超时？

连接超时处理：

- 设置合理的连接超时（5-10 秒）
- 超时后尝试其他服务器
- 或回退到 UDP 查询
- 记录超时事件用于分析

### Q4: TCP DNS 可以和 UDP DNS 使用相同端口吗？

端口使用：

- 标准 DNS 端口：53（UDP 和 TCP）
- 服务器通常同时监听 UDP 和 TCP
- 客户端连接 TCP 时使用端口 53
- DoT 使用不同端口（853）

### Q5: 查询流水线如何保证响应顺序？

顺序保证机制：

- 服务器必须按查询顺序响应
- 通过 ID 区分不同的查询
- 客户端匹配响应 ID 和查询 ID
- RFC 7766 明确要求顺序响应

### Q6: Prism 如何管理 TCP DNS 连接池？

Prism 的连接池管理：

- 每个上游服务器独立的连接池
- 最大连接数限制
- 空闲超时清理
- 自动重连机制

### Q7: TCP DNS 和 DoT 有什么区别？

主要区别：

| 特性 | TCP DNS | DoT |
|------|---------|-----|
| 端口 | 53 | 853 |
| 加密 | 无 | TLS 加密 |
| 连接 | TCP | TLS over TCP |
| 认证 | 无 | 证书认证 |

### Q8: 如何优化 TCP DNS 性能？

性能优化建议：

- 使用连接池避免握手开销
- 支持查询流水线
- 设置合理的超时
- 优先使用 UDP，必要时才用 TCP
- 预判大响应直接用 TCP

## 参考资料

- [RFC 7766 - DNS Transport over TCP](https://tools.ietf.org/html/rfc7766)
- [RFC 1035 - Domain Names: Implementation](https://tools.ietf.org/html/rfc1035)
- [RFC 5936 - DNS Zone Transfer Protocol (AXFR)](https://tools.ietf.org/html/rfc5936)

## 相关知识

- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT 加密 DNS
- [[ref/protocol/dns-over-https|DNS over HTTPS]] — DoH 加密 DNS
- [[ref/resolve/dns-basics|DNS 基础]] — DNS 协议基础