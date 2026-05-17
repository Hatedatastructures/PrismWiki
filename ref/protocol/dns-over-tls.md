---
title: "DNS over TLS (DoT) 协议"
category: "protocol"
type: ref
module: ref
source: "RFC 7858"
tags: [协议, dns, tls, dot, 加密, 隐私]
created: 2026-05-15
updated: 2026-05-17
related: [dns-over-udp, dns-over-tcp, dns-over-https, tls-1-3]
---

# DNS over TLS (DoT) 协议

**类别**: 协议

## 概述

DNS over TLS (DoT) 是一种加密 DNS 查询的协议，由 RFC 7858（2016 年）定义。DoT 使用 TLS 协议加密 DNS 查询和响应，保护 DNS 流量的隐私性和完整性。DoT 使用专用端口 853，与普通 DNS 的端口 53 区分开，使网络管理员可以识别和管理 DoT 流量。

DNS over TLS 解决了传统 DNS 的安全问题：

- **隐私保护**：防止 DNS 查询被窃听
- **完整性保护**：防止 DNS 响应被篡改
- **身份验证**：验证 DNS 服务器身份
- **防欺骗攻击**：防止 DNS 欺骗攻击

在 Prism 项目中，DoT 是 Resolve 模块支持的加密 DNS 传输方式之一，用于向支持 DoT 的 DNS 服务器发送加密查询。

### DNS 安全问题

传统 DNS 的安全缺陷：

```
传统 DNS 安全问题：
┌────────────────────────────────────────────┐
│ 1. 明文传输                                  │
│    - 查询内容可被窃听                        │
│    - 暴露访问的域名                          │
│    - 泄露用户行为                            │
│                                            │
│ 2. 无完整性保护                              │
│    - 响应可被篡改                            │
│    - DNS 欺骗攻击                            │
│    - 中间人攻击                              │
│                                            │
│ 3. 无身份验证                                │
│    - 无法验证服务器身份                      │
│    - 可能连接假冒服务器                      │
│                                            │
│ 4. 易被分析                                  │
│    - 流量可被监控                            │
│    - 行为可被分析                            │
└────────────────────────────────────────────┘
```

### DoT vs 传统 DNS

DoT 与传统 DNS 的对比：

| 特性 | 传统 DNS | DoT |
|------|----------|-----|
| 端口 | 53 | 853 |
| 加密 | 无 | TLS |
| 完整性 | 无 | TLS MAC |
| 身份验证 | 无 | TLS 证书 |
| 隐私性 | 无 | 加密保护 |
| 性能 | 低延迟 | 较高（TLS 开销） |
| 流量识别 | 容易 | 需要解密 |

### DoT vs DoH

DoT 与 DNS over HTTPS 的对比：

| 特性 | DoT | DoH |
|------|-----|-----|
| 协议 | TLS over TCP | HTTPS (HTTP/2) |
| 端口 | 853 | 443 |
| 流量特征 | 可识别（853） | 与 HTTPS 混合 |
| 性能 | 较高 | 较高（HTTP 开销） |
| 实现复杂度 | 较低 | 较高 |
| 适用场景 | DNS 客户端 | 浏览器集成 |
| CDN 支持 | 难 | 可利用 CDN |

### DoT 的优势

DoT 的主要优势：

1. **隐私保护**：加密查询内容，防止窃听
2. **完整性保证**：TLS 提供消息完整性保护
3. **服务器验证**：证书验证防止假冒服务器
4. **标准化**：RFC 标准，广泛支持
5. **易于部署**：相比 DoH 更简单

## 协议详解

### 协议流程

DNS over TLS 的完整流程：

```
DoT 协议流程：

Client                    DoT Server
  |                           |
  |--- TCP SYN -------------->|  (TCP 连接)
  |<-- TCP SYN+ACK -----------|
  |--- TCP ACK -------------->|
  |                           |
  |--- TLS ClientHello ------>|  (TLS 握手)
  |<-- TLS ServerHello -------|
  |<-- TLS Certificate -------|
  |<-- TLS Finished ----------|
  |--- TLS Finished --------->|
  |                           |
  |   (TLS 连接建立完成)       |
  |                           |
  |--- DNS Query (加密) ----->|  (加密的 DNS 查询)
  |   Length: 00 30          |
  |   [DNS Query Message]     |
  |                           |
  |<-- DNS Response (加密) ---|  (加密的 DNS 响应)
  |   Length: 00 50          |
  |   [DNS Response Message]  |
  |                           |
  |--- DNS Query #2 (加密) -->|  (可以发送多个查询)
  |                           |
  |<-- DNS Response #2 (加密)-|
  |                           |
  |--- TLS CloseNotify ----->|  (关闭连接)
  |<-- TLS CloseNotify ------|
  |                           |
```

### TLS 层配置

DoT 的 TLS 配置要求：

```
TLS 配置要求 (RFC 7858):
┌────────────────────────────────────────────┐
│ 1. TLS 版本                                 │
│    - 必须支持 TLS 1.2                       │
│    - 推荐支持 TLS 1.3                       │
│    - 不应使用 TLS 1.0/1.1                   │
│                                            │
│ 2. 密码套件                                  │
│    - 必须使用 AEAD 套件                     │
│    - 推荐使用 TLS_AES_128_GCM_SHA256       │
│                                            │
│ 3. 证书要求                                  │
│    - 服务器必须提供有效证书                  │
│    - 客户端应验证证书                        │
│    - 支持证书透明度                          │
│                                            │
│ 4. 扩展                                     │
│    - 支持 ALPN (可选)                       │
│    - 支持 SNI                               │
└────────────────────────────────────────────┘
```

### DNS 消息格式

DoT 使用 TCP DNS 格式（带长度前缀）：

```
DoT 消息格式：
┌────────────────────────────────────────────┐
│ TLS Record Layer:                          │
│   ContentType: Application Data            │
│   Version: TLS 1.2/1.3                     │
│   Length: N                                │
│                                            │
│   Encrypted Data:                          │
│     DNS Message Length: 2 bytes           │
│     DNS Message: variable                  │
│                                            │
│   TLS MAC/AEAD Tag                         │
└────────────────────────────────────────────┘

解密后的 DNS 消息：
┌────────────────────────────────────────────┐
│ Length: 2 bytes (大端序)                    │
│ DNS Message:                               │
│   Header: 12 bytes                         │
│   Question: variable                       │
│   Answer: variable                         │
│   Authority: variable                      │
│   Additional: variable                     │
└────────────────────────────────────────────┘
```

### 连接复用

DoT 支持连接复用：

```
连接复用策略：
┌────────────────────────────────────────────┐
│ 1. 建立 TLS 连接                            │
│    - TLS 握手开销较高                       │
│    - 应复用连接                              │
│                                            │
│ 2. 多查询共享连接                            │
│    - 在同一 TLS 连接上发送多个查询           │
│    - 按顺序响应                              │
│                                            │
│ 3. 流水线支持                                │
│    - 可以连续发送多个查询                    │
│    - 不必等待每个响应                        │
│                                            │
│ 4. 空闲超时                                  │
│    - 设置合理的空闲超时                      │
│    - 关闭空闲连接                            │
└────────────────────────────────────────────┘
```

### 端口和连接

DoT 的端口配置：

```
端口配置：
┌────────────────────────────────────────────┐
│ 标准端口: 853                                │
│                                            │
│ 客户端行为:                                  │
│   - 连接到服务器端口 853                     │
│   - 进行 TLS 握手                            │
│   - 发送加密 DNS 查询                        │
│                                            │
│ 服务器行为:                                  │
│   - 监听端口 853                             │
│   - 提供 TLS 证书                            │
│   - 处理加密 DNS 查询                        │
│                                            │
│ 注意事项:                                    │
│   - 端口 853 与普通 DNS (53) 不同           │
│   - 可以被防火墙识别为 DoT 流量              │
└────────────────────────────────────────────┘
```

### 证书验证

DoT 的证书验证要求：

```
证书验证流程：
┌────────────────────────────────────────────┐
│ 1. 服务器提供证书                            │
│                                            │
│ 2. 客户端验证：                              │
│    a) 证书链完整性                           │
│    b) 证书有效期                             │
│    c) 证书用途（服务器认证）                 │
│    d) 域名匹配                               │
│    e) 签名验证                               │
│    f) 信任链                                 │
│                                            │
│ 3. 可选验证：                                │
│    - OCSP 检查                               │
│    - 证书透明度                              │
│    - DANE (DNSSEC)                          │
│                                            │
│ 4. 验证失败处理：                            │
│    - 不建立连接                              │
│    - 不发送查询                              │
│    - 记录错误                                │
└────────────────────────────────────────────┘
```

### 性能考虑

DoT 的性能因素：

```
性能影响因素：
┌────────────────────────────────────────────┐
│ TLS 握手开销：                               │
│   - 完整握手：1-2 RTT                        │
│   - TLS 1.3：1 RTT                          │
│   - 会话恢复：0-1 RTT                        │
│                                            │
│ 数据开销：                                    │
│   - TLS 记录层开销                          │
│   - AEAD 标签（16 bytes）                   │
│   - TLS 握手数据                            │
│                                            │
│ 缓解措施：                                    │
│   - 连接复用                                 │
│   - TLS 1.3 快速握手                        │
│   - 会话票证                                 │
│   - 查询流水线                               │
└────────────────────────────────────────────┘
```

### 关闭连接

DoT 连接的正确关闭：

```
连接关闭流程：
┌────────────────────────────────────────────┐
│ TLS 层关闭：                                 │
│   - 发送 CloseNotify                        │
│   - 接收 CloseNotify                        │
│   - 等待数据读取完成                         │
│                                            │
│ TCP 层关闭：                                 │
│   - 发送 FIN                                 │
│   - 接收 FIN+ACK                            │
│   - 发送 ACK                                 │
│                                            │
│ 注意事项：                                    │
│   - 先完成 TLS 关闭再关闭 TCP               │
│   - 不应直接 TCP reset                       │
│   - 确保所有数据已传输                       │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### DoT 查询实现

Prism 的 DoT 查询实现：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto dot_query(const dns_config& config,
               const endpoint& server,
               std::span<const std::byte> query)
    -> net::awaitable<result<message>>
{
    auto executor = co_await net::this_coroutine::executor;
    
    // 从 TLS 连接池获取连接
    auto tls_stream = co_await get_or_create_tls_connection(server);
    
    // 构造 TCP DNS 格式（带长度前缀）
    std::vector<std::byte> tcp_query;
    append_u16(tcp_query, query.size());
    tcp_query.append(query.begin(), query.end());
    
    // 通过 TLS 连接发送
    co_await net::async_write(tls_stream,
        net::buffer(tcp_query),
        net::use_awaitable);
    
    // 接收响应长度
    std::array<std::byte, 2> length_buf;
    co_await net::async_read(tls_stream,
        net::buffer(length_buf),
        net::transfer_exactly(2),
        net::use_awaitable);
    
    auto response_length = read_u16(length_buf, 0);
    
    // 接收响应消息
    std::vector<std::byte> response_buf(response_length);
    co_await net::async_read(tls_stream,
        net::buffer(response_buf),
        net::transfer_exactly(response_length),
        net::use_awaitable);
    
    // 解析响应
    auto response = parse_message(response_buf);
    
    // 返回连接到池
    co_await return_tls_connection(tls_stream);
    
    co_return response.value();
}
```

### TLS 连接池

Prism 的 DoT 连接池：

```cpp
// 文件: src/prism/resolve/dns/pool.cpp
class tls_connection_pool {
public:
    auto get_or_create(const endpoint& server)
        -> net::awaitable<tls_stream>
    {
        // 查找现有 TLS 连接
        auto& connections = pool_[server];
        
        for (auto& conn : connections) {
            if (conn.is_valid() && !conn.is_in_use()) {
                // 验证 TLS 连接状态
                if (conn.tls_session_active()) {
                    conn.mark_in_use();
                    co_return conn.stream();
                }
            }
        }
        
        // 创建新的 TLS 连接
        auto new_stream = co_await create_tls_connection(server);
        
        // 保存到池
        connections.push_back({new_stream, true});
        
        co_return new_stream;
    }
    
private:
    auto create_tls_connection(const endpoint& server)
        -> net::awaitable<tls_stream>
    {
        // TCP 连接
        net::tcp::socket socket(executor_);
        co_await socket.async_connect(server, net::use_awaitable);
        
        // TLS 配置
        ssl::context ctx(ssl::context::tls_client);
        ctx.set_verify_mode(ssl::verify_peer);
        ctx.set_default_verify_paths();
        
        // TLS 握手
        ssl::stream<net::tcp::socket> tls_stream(socket, ctx);
        co_await tls_stream.async_handshake(ssl::stream_base::client,
                                             net::use_awaitable);
        
        co_return tls_stream;
    }
    
    std::map<endpoint, std::vector<tls_connection>> pool_;
};
```

### 证书验证

Prism 的 DoT 证书验证：

```cpp
// 文件: src/prism/resolve/dns/verify.cpp
auto verify_dot_certificate(const ssl::stream<net::tcp::socket>& stream,
                             const std::string& expected_hostname)
    -> bool
{
    // 获取证书
    auto cert = stream.peer_certificate();
    if (!cert) {
        return false;
    }
    
    // 验证证书链
    auto verify_result = cert.verify();
    if (!verify_result) {
        return false;
    }
    
    // 验证域名匹配
    auto subject_name = cert.subject_name();
    if (!subject_name.matches(expected_hostname)) {
        return false;
    }
    
    // 验证有效期
    auto now = std::chrono::system_clock::now();
    if (cert.not_after() < now || cert.not_before() > now) {
        return false;
    }
    
    return true;
}
```

### 协议检测

Prism 检测 DoT 支持：

```cpp
// 文件: src/prism/resolve/dns/discovery.cpp
auto probe_dot_support(const endpoint& server)
    -> net::awaitable<bool>
{
    try {
        // 尝试建立 TLS 连接
        auto tls_stream = co_await create_tls_connection(server);
        
        // 发送简单查询测试
        auto test_query = craft_query("test.local", QTYPE::A);
        auto response = co_await dot_query(test_query);
        
        co_return true;
    } catch (...) {
        co_return false;
    }
}
```

## 最佳实践

### 连接管理

1. **连接复用**：复用 TLS 连接减少握手开销
2. **空闲超时**：设置合理的空闲超时（30-60 秒）
3. **最大连接数**：限制每个服务器的连接数
4. **健康检查**：定期检查连接健康状态

### 证书验证

1. **严格验证**：验证证书链、有效期、域名
2. **信任锚**：使用系统信任锚或自定义
3. **错误处理**：证书验证失败应拒绝连接
4. **日志记录**：记录验证失败事件

### 性能优化

1. **TLS 1.3**：使用 TLS 1.3 减少握手开销
2. **会话恢复**：使用会话票证恢复会话
3. **查询流水线**：在同一连接发送多个查询
4. **预建立连接**：提前建立连接备用

### 隐私增强

1. **严格证书验证**：防止假冒服务器
2. **无日志服务器**：选择无日志的 DoT 服务
3. **分散查询**：使用多个 DoT 服务器
4. **本地缓存**：减少对外部查询的需求

## 常见问题

### Q1: DoT 使用什么端口？

DoT 使用端口 853：

- 与普通 DNS 端口 53 不同
- 可以被防火墙识别
- 某些网络可能封锁 853 端口

### Q2: DoT 比 DoH 更好吗？

各有优势：

- **DoT 优势**：实现简单、专用端口、标准化
- **DoH 优势**：与 HTTPS 混合、可利用 CDN、浏览器友好

选择取决于应用场景。

### Q3: DoT 能防止 DNS 劫持吗？

DoT 提供防护：

- 加密防止查询泄露
- TLS 保护响应完整性
- 证书验证防止假冒
- 但无法防止服务器本身的劫持

### Q4: DoT 连接可以复用吗？

可以复用：

- 同一 TLS 连接可发送多个查询
- 减少握手开销
- 按顺序响应
- 应设置合理的空闲超时

### Q5: Prism 如何选择 DoT 服务器？

Prism 的选择策略：

- 配置指定的 DoT 服务器
- 验证服务器证书
- 检测连接可用性
- 按优先级或轮询选择

### Q6: DoT 会影响 DNS 查询速度吗？

影响因素：

- TLS 握手增加首次延迟
- 连接复用可缓解
- TLS 1.3 减少握手 RTT
- 后续查询延迟接近普通 DNS

### Q7: 如何验证 DoT 服务器证书？

验证步骤：

1. 检查证书链完整性
2. 验证证书签名
3. 检查有效期
4. 验证域名匹配
5. 检查信任锚

### Q8: DoT 可以使用 TLS 1.3 吗？

可以使用 TLS 1.3：

- TLS 1.3 提供更好的性能
- 1-RTT 握手
- 更强的安全性
- 推荐 DoT 服务器支持 TLS 1.3

## 参考资料

- [RFC 7858 - Specification for DNS over TLS](https://tools.ietf.org/html/rfc7858)
- [RFC 8310 - Usage Profiles for DNS over TLS](https://tools.ietf.org/html/rfc8310)
- [RFC 8446 - TLS 1.3](https://tools.ietf.org/html/rfc8446)

## 相关知识

- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
- [[ref/protocol/dns-over-tcp|DNS over TCP]] — TCP DNS 查询
- [[ref/protocol/dns-over-https|DNS over HTTPS]] — DoH 加密 DNS
- [[ref/protocol/tls-1-3|TLS 1.3]] — TLS 1.3 协议