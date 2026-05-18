---
title: "DNS over HTTPS (DoH) 协议"
category: "protocol"
type: ref
module: ref
source: "RFC 8484"
tags: [协议, dns, https, doh, 加密, 隐私, http2]
created: 2026-05-15
updated: 2026-05-17
related: [dns-over-udp, dns-over-tcp, dns-over-tls, http-2]
layer: ref
---

# DNS over HTTPS (DoH) 协议

**类别**: 协议

## 概述

DNS over HTTPS (DoH) 是一种通过 HTTPS 协议传输 DNS 查询的方法，由 RFC 8484（2018 年）定义。DoH 将 DNS 查询封装在 HTTPS 请求中，使 DNS 流量与普通 Web 流量混合，难以被识别和封锁。DoH 使用标准 HTTPS 端口 443，并使用 HTTP/2 进行传输以提高效率。

DNS over HTTPS 的设计目标是：

- **流量隐藏**：DNS 查询与 HTTPS 流量混合，难以识别
- **隐私保护**：通过 HTTPS 加密保护 DNS 查询内容
- **广泛部署**：利用现有 HTTPS 基础设施
- **浏览器集成**：易于在浏览器中实现

DoH 在绕过 DNS 封锁和审查方面具有特殊价值，但也引发了网络管理争议。许多公共 DNS 服务提供商（如 Cloudflare、Google）提供 DoH 服务，主流浏览器（Chrome、Firefox）也支持 DoH。

在 Prism 项目中，DoH 是 Resolve 模块支持的加密 DNS 传输方式之一，用于向支持 DoH 的 DNS 服务器发送加密查询。

### DoH vs DoT vs 传统 DNS

三种 DNS 传输方式的对比：

| 特性 | 传统 DNS | DoT | DoH |
|------|----------|-----|-----|
| 协议 | UDP/TCP | TLS | HTTPS (HTTP/2) |
| 端口 | 53 | 853 | 443 |
| 加密 | 无 | TLS | TLS |
| 流量特征 | DNS | DoT (853) | HTTPS (443) |
| 识别难度 | 容易 | 中等 | 困难 |
| CDN 支持 | 难 | 难 | 可利用 CDN |
| 性能 | 最高 | 中等 | 较低（HTTP 开销） |
| 适用场景 | 基础设施 | DNS 客户端 | 浏览器集成 |

### DoH 的优势

DoH 的主要优势：

1. **流量隐藏**：与 HTTPS 流量混合，难以封锁
2. **CDN 支持**：可利用 CDN 加速和冗余
3. **标准化**：RFC 标准，广泛支持
4. **浏览器友好**：易于在浏览器中集成
5. **现有基础设施**：利用 HTTPS 生态

### DoH 的争议

DoH 引起的争议：

```
争议点：
┌────────────────────────────────────────────┐
│ 1. 网络管理影响                              │
│    - 企业无法监控 DNS 流量                   │
│    - 安全设备无法检测 DNS 攻击               │
│                                            │
│ 2. 本地 DNS 设置绕过                         │
│    - 可能绕过本地 DNS 策略                   │
│    - 影响企业 DNS 解析                       │
│                                            │
│ 3. 隐私与控制的平衡                           │
│    - 用户隐私保护 vs 网络管理                │
│    - 浏览器自动启用 DoH 的争议               │
└────────────────────────────────────────────┘
```

## 协议详解

### 协议流程

DNS over HTTPS 的完整流程：

```
DoH 协议流程：

Client                    DoH Server
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
  |--- HTTP/2 SETTINGS ----->|  (HTTP/2 初始化)
  |<-- HTTP/2 SETTINGS ------|
  |                           |
  |--- HTTP/2 HEADERS ------->|  (DNS 查询请求)
  |   :method: GET/POST       |
  |   :path: /dns-query?dns=..|
  |   :authority: dns.server  |
  |   content-type:           |
  |     application/dns-message|
  |                           |
  |--- HTTP/2 DATA ---------->|  (POST: DNS 消息体)
  |   [DNS Wire Format]       |
  |                           |
  |<-- HTTP/2 HEADERS --------|  (响应头)
  |   :status: 200            |
  |   content-type:           |
  |     application/dns-message|
  |                           |
  |<-- HTTP/2 DATA -----------|  (DNS 响应)
  |   [DNS Wire Format]       |
  |                           |
```

### HTTP 请求格式

DoH 使用两种 HTTP 方法：

#### GET 方法

```
GET 请求格式：
┌────────────────────────────────────────────┐
│ GET /dns-query?dns=BASE64URL HTTP/2       │
│ Host: dns.example.com                      │
│ Accept: application/dns-message            │
│                                            │
│ URL 参数：                                  │
│   dns: Base64URL 编码的 DNS 查询消息       │
│                                            │
│ Base64URL 编码：                            │
│   - 标准 Base64 编码                        │
│   - 替换 + 为 -                             │
│   - 替换 / 为 _                             │
│   - 去除 = 填充                             │
└────────────────────────────────────────────┘

示例：
GET /dns-query?dns=AAABAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQ HTTP/2
Host: dns.cloudflare.com
Accept: application/dns-message
```

#### POST 方法

```
POST 请求格式：
┌────────────────────────────────────────────┐
│ POST /dns-query HTTP/2                    │
│ Host: dns.example.com                      │
│ Content-Type: application/dns-message     │
│ Content-Length: N                          │
│                                            │
│ [DNS Wire Format 消息体]                    │
└────────────────────────────────────────────┘

示例：
POST /dns-query HTTP/2
Host: dns.cloudflare.com
Content-Type: application/dns-message
Content-Length: 33

[33 bytes DNS query message]
```

### DNS 消息格式

DoH 使用标准 DNS Wire Format：

```
DNS Wire Format (与 UDP DNS 相同)：
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

注意：
┌────────────────────────────────────────────┐
│ - POST 方法：直接发送 DNS Wire Format       │
│ - GET 方法：Base64URL 编码后放入 URL        │
│ - 响应：直接返回 DNS Wire Format            │
│ - 不使用 TCP DNS 的长度前缀                 │
└────────────────────────────────────────────┘
```

### HTTP 响应格式

DoH 的 HTTP 响应：

```
成功响应：
┌────────────────────────────────────────────┐
│ HTTP/2 200 OK                              │
│ Content-Type: application/dns-message     │
│ Content-Length: N                          │
│ Cache-Control: max-age=3600                │
│                                            │
│ [DNS Wire Format 响应]                      │
└────────────────────────────────────────────┘

错误响应：
┌────────────────────────────────────────────┐
│ HTTP/2 400 Bad Request                     │
│ Content-Type: application/dns-message     │
│                                            │
│ [可能包含 DNS 错误响应]                      │
│                                            │
│ HTTP/2 404 Not Found                       │
│ (URI 不正确)                                │
│                                            │
│ HTTP/2 415 Unsupported Media Type         │
│ (Content-Type 不正确)                       │
│                                            │
│ HTTP/2 500 Internal Server Error          │
│ (服务器处理错误)                             │
└────────────────────────────────────────────┘
```

### HTTP/2 要求

DoH 对 HTTP/2 的要求：

```
HTTP/2 要求：
┌────────────────────────────────────────────┐
│ 必须使用 HTTP/2：                            │
│   - RFC 8484 要求                           │
│   - 支持多路复用                             │
│   - 头部压缩                                 │
│                                            │
│ HTTP/2 特性利用：                            │
│   - Stream 复用：多个 DNS 查询              │
│   - Header Compression: 减少 overhead      │
│   - Server Push: 可推送 DNS 响应            │
│                                            │
│ 注意：                                       │
│   - DoH 实现必须支持 HTTP/2                 │
│   - HTTP/1.1 不符合 RFC 要求                │
└────────────────────────────────────────────┘
```

### URI 模板

DoH 的 URI 格式：

```
URI 模板：
┌────────────────────────────────────────────┐
│ 标准 URI: https://server/dns-query{?dns}  │
│                                            │
│ 示例：                                       │
│   Cloudflare: https://cloudflare-dns.com/dns-query│
│   Google: https://dns.google/dns-query     │
│   Quad9: https://dns.quad9.net/dns-query   │
│                                            │
│ GET 请求 URI：                               │
│   https://server/dns-query?dns=BASE64URL  │
│                                            │
│ POST 请求 URI：                              │
│   https://server/dns-query                 │
└────────────────────────────────────────────┘
```

### Content-Type

DoH 的内容类型：

```
Content-Type:
┌────────────────────────────────────────────┐
│ application/dns-message                    │
│                                            │
│ 用途：                                       │
│   - 请求 Accept 头                          │
│   - 请求 Content-Type (POST)               │
│   - 响应 Content-Type                       │
│                                            │
│ 表示 DNS Wire Format 消息                   │
└────────────────────────────────────────────┘
```

### GET vs POST

GET 与 POST 方法的对比：

```
GET 方法：
┌────────────────────────────────────────────┐
│ 优点：                                       │
│   - 可缓存（利用 HTTP 缓存）                 │
│   - URL 可预构建                             │
│   - 无消息体开销                             │
│                                            │
│ 缺点：                                       │
│   - URL 长度限制                             │
│   - Base64URL 编码开销                      │
│   - 查询内容出现在 URL 中                    │
└────────────────────────────────────────────┘

POST 方法：
┌────────────────────────────────────────────┐
│ 优点：                                       │
│   - 无长度限制                               │
│   - 无编码开销                               │
│   - 更适合大查询                             │
│                                            │
│ 缺点：                                       │
│   - 默认不可缓存                             │
│   - 需要消息体                               │
│   - 需要设置 Content-Type                   │
└────────────────────────────────────────────┘
```

### 缓存机制

DoH 的 HTTP 缓存：

```
HTTP 缓存利用：
┌────────────────────────────────────────────┐
│ 服务器响应 Cache-Control:                   │
│   Cache-Control: max-age=3600              │
│   (基于 DNS TTL)                             │
│                                            │
│ 缓存机制：                                    │
│   - HTTP 层缓存                              │
│   - CDN 缓存                                 │
│   - 客户端缓存                               │
│                                            │
│ 缓存键：                                      │
│   - GET: URL (包含 dns 参数)                │
│   - POST: 需要特殊配置                       │
│                                            │
│ 注意：                                        │
│   - DNS TTL 与 HTTP Cache 应一致            │
│   - 负缓存也需要处理                          │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### DoH 查询实现

Prism 的 DoH 查询实现：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto doh_query(const dns_config& config,
               const std::string& doh_uri,
               std::span<const std::byte> query)
    -> net::awaitable<result<message>>
{
    auto executor = co_await net::this_coroutine::executor;
    
    // 创建 HTTPS 连接
    auto https_stream = co_await get_or_create_https_connection(doh_uri);
    
    // 构造 HTTP/2 请求
    http::request<http::vector_body<std::byte>> request;
    request.method(http::verb::post);
    request.target("/dns-query");
    request.set(http::field::host, extract_host(doh_uri));
    request.set(http::field::content_type, "application/dns-message");
    request.set(http::field::accept, "application/dns-message");
    request.body() = std::vector<std::byte>(query.begin(), query.end());
    request.prepare_payload();
    
    // 发送请求
    co_await http::async_write(https_stream, request, net::use_awaitable);
    
    // 接收响应
    http::response<http::vector_body<std::byte>> response;
    co_await http::async_read(https_stream, buffer, response,
                              net::use_awaitable);
    
    // 检查 HTTP 状态
    if (response.result() != http::status::ok) {
        co_return result::http_error;
    }
    
    // 解析 DNS 响应
    auto dns_response = parse_message(response.body());
    
    co_return dns_response.value();
}
```

### GET 方法实现

Prism 的 GET 方法：

```cpp
// 文件: src/prism/resolve/dns/upstream.cpp
auto doh_get_query(const dns_config& config,
                   const std::string& doh_uri,
                   std::span<const std::byte> query)
    -> net::awaitable<result<message>>
{
    // Base64URL 编码
    auto encoded_query = base64url_encode(query);
    
    // 构造 URI
    auto target = "/dns-query?dns=" + encoded_query;
    
    // 构造 HTTP/2 请求
    http::request<http::empty_body> request;
    request.method(http::verb::get);
    request.target(target);
    request.set(http::field::host, extract_host(doh_uri));
    request.set(http::field::accept, "application/dns-message");
    
    // 发送请求并接收响应...
}
```

### HTTPS 连接池

Prism 的 DoH 连接池：

```cpp
// 文件: src/prism/resolve/dns/pool.cpp
class https_connection_pool {
public:
    auto get_or_create(const std::string& uri)
        -> net::awaitable<https_stream>
    {
        auto host = extract_host(uri);
        
        // 查找现有连接
        auto& connections = pool_[host];
        
        for (auto& conn : connections) {
            if (conn.is_valid() && !conn.is_in_use()) {
                conn.mark_in_use();
                co_return conn.stream();
            }
        }
        
        // 创建新连接
        auto new_stream = co_await create_https_connection(uri);
        
        // HTTP/2 初始化
        co_await init_http2(new_stream);
        
        connections.push_back({new_stream, true});
        
        co_return new_stream;
    }
    
private:
    auto create_https_connection(const std::string& uri)
        -> net::awaitable<https_stream>
    {
        // TCP 连接
        // TLS 握手
        // HTTP/2 协商
        // ...
    }
    
    std::map<std::string, std::vector<https_connection>> pool_;
};
```

### HTTP/2 支持

Prism 的 HTTP/2 处理：

```cpp
// 文件: src/prism/resolve/dns/http2.cpp
class http2_client {
public:
    auto send_request(https_stream& stream,
                      const http_request& request)
        -> net::awaitable<http_response>
    {
        // 创建 HTTP/2 stream
        auto stream_id = create_stream();
        
        // 发送 HEADERS frame
        co_await send_headers_frame(stream, stream_id, request.headers());
        
        // 发送 DATA frame (POST)
        if (request.has_body()) {
            co_await send_data_frame(stream, stream_id, request.body());
        }
        
        // 接收响应 HEADERS
        auto response_headers = co_await receive_headers_frame(stream, stream_id);
        
        // 接收响应 DATA
        auto response_data = co_await receive_data_frame(stream, stream_id);
        
        co_return build_response(response_headers, response_data);
    }
};
```

## 最佳实践

### 连接管理

1. **连接复用**：复用 HTTPS 连接减少开销
2. **HTTP/2 多路复用**：在单一连接发送多个查询
3. **空闲超时**：设置合理的空闲超时
4. **连接预热**：提前建立连接

### 缓存利用

1. **GET 方法**：使用 GET 方法便于缓存
2. **Cache-Control**：遵循服务器缓存指示
3. **本地缓存**：结合本地 DNS 缓存
4. **CDN 利用**：选择有 CDN 支持的 DoH 服务

### 性能优化

1. **HTTP/2**：使用 HTTP/2 多路复用
2. **头部压缩**：利用 HPACK 头部压缩
3. **并行查询**：在多个 stream 上并行发送
4. **预构建请求**：提前构建请求减少延迟

### 安全考虑

1. **证书验证**：验证服务器证书
2. **HTTPS 安全**：确保 TLS 配置安全
3. **DoH 服务器选择**：选择可信的 DoH 服务
4. **日志隐私**：选择无日志的 DoH 服务

## 常见问题

### Q1: DoH 为什么必须使用 HTTP/2？

HTTP/2 的必要性：

- RFC 8484 明确要求
- 支持多路复用（多个查询）
- 头部压缩减少开销
- 更高效的连接利用

### Q2: DoH 与 DoT 哪个更好？

各有优势：

- **DoH**：流量隐藏、CDN 支持、浏览器友好
- **DoT**：实现简单、专用协议、明确用途

选择取决于应用场景。

### Q3: GET 还是 POST 更好？

选择建议：

- **GET**：可缓存、适合重复查询
- **POST**：无限制、适合大查询

一般推荐 GET 以利用缓存。

### Q4: DoH 如何处理缓存？

缓存策略：

- 服务器设置 Cache-Control（基于 DNS TTL）
- HTTP 缓存层缓存响应
- CDN 可缓存 GET 请求
- 客户端本地缓存

### Q5: DoH 能绕过所有封锁吗？

限制因素：

- 端口 443 通常不被封锁
- 但可能封锁特定 DoH 服务器域名
- 需要选择合适的 DoH 服务
- 或使用 IP 直连

### Q6: Prism 如何选择 DoH 服务器？

Prism 的选择策略：

- 配置指定的 DoH URI
- 支持多个服务器轮询
- 验证服务器响应
- 自动 fallback

### Q7: DoH 的性能如何？

性能因素：

- HTTPS 连接开销较高
- HTTP/2 多路复用缓解
- 后续查询延迟较低
- CDN 可改善性能

### Q8: 如何验证 DoH 服务器？

验证方法：

- 检查 HTTPS 证书
- 发送测试查询
- 验证 DNS 响应格式
- 检查 HTTP 状态码

## 参考资料

- [RFC 8484 - DNS Queries over HTTPS](https://tools.ietf.org/html/rfc8484)
- [RFC 7540 - HTTP/2](https://tools.ietf.org/html/rfc7540)
- [DNS over HTTPS - Wikipedia](https://en.wikipedia.org/wiki/DNS_over_HTTPS)

## 相关知识

- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
- [[ref/protocol/dns-over-tcp|DNS over TCP]] — TCP DNS 查询
- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT 加密 DNS
- [[ref/protocol/http-2|HTTP/2]] — HTTP/2 协议