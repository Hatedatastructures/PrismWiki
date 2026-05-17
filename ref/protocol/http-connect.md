---
title: "HTTP CONNECT 代理方法"
category: "protocol"
type: ref
module: ref
source: "RFC 7231 Section 4.3.6"
tags: [协议, http, connect, 代理, 隧道, https]
created: 2026-05-15
updated: 2026-05-17
related: [socks5-rfc1928, trojan, http-1-1, proxy-basics]
---

# HTTP CONNECT 代理方法

**类别**: 协议

## 概述

HTTP CONNECT 是 HTTP 协议中定义的一种特殊请求方法，用于通过 HTTP 代理服务器建立 TCP 隧道。CONNECT 方法最初在 HTTP/1.1 (RFC 2616) 中定义，后来在 RFC 7231 Section 4.3.6 中重新规范。与普通的 HTTP 请求不同，CONNECT 方法不请求特定的资源，而是请求代理服务器建立一个到目标服务器和端口的 TCP 连接隧道。

HTTP CONNECT 代理是 HTTPS 代理的标准方式，也是 WebSocket 代理的常用方法。当客户端需要通过代理访问 HTTPS 网站时，必须使用 CONNECT 方法建立隧道，因为 HTTPS 流量需要端到端加密，代理服务器无法解密和转发 HTTP 请求。

HTTP CONNECT 在代理领域有着广泛的应用：

- **HTTPS 代理**：通过 HTTP 代理访问 HTTPS 网站
- **WebSocket 代理**：WebSocket 连接的代理方式
- **隧道代理**：代理任意 TCP 流量
- **企业代理**：企业网络的 HTTP 代理基础设施

在 Prism 项目中，HTTP CONNECT 是核心支持的代理协议之一，提供 HTTP 和 HTTPS 代理功能，支持 HTTP 基本认证和自定义头部。

### HTTP CONNECT 与普通 HTTP 请求的区别

CONNECT 方法与普通 HTTP 请求的关键区别：

| 特性 | CONNECT 方法 | 普通 HTTP 请求 |
|------|--------------|----------------|
| 目的 | 建立 TCP 隧道 | 获取资源 |
| 请求 URI | host:port | 完整 URL 或路径 |
| 响应 | 200 Connection Established | HTTP 内容 |
| 后续数据 | 原始 TCP 流 | 无后续数据 |
| 协议层级 | 隧道层 | 应用层 |
| 加密支持 | 支持 HTTPS 隧道 | 不涉及加密 |

### HTTPS 代理的工作原理

HTTPS 代理通过 CONNECT 方法实现：

```
HTTPS 代理原理：

Client                    HTTP Proxy                HTTPS Server
  |                            |                          |
  |--- CONNECT target:443 ---->|                          |
  |                            |                          |
  |                            |--- TCP 连接 ------------>|
  |                            |                          |
  |<-- 200 Connection Estab ---|                          |
  |                            |                          |
  |<======== TLS 握手隧道 ====================>|
  |                            |                          |
  |                            |   (代理服务器不解密)      |
  |                            |                          |
  |<======== HTTPS 数据隧道 ==================>|
```

代理服务器不知道 TLS 内部的 HTTP 请求内容，只负责转发加密数据。

### HTTP CONNECT 的优势

HTTP CONNECT 的主要优势：

1. **广泛支持**：所有现代浏览器和 HTTP 客户端都支持
2. **HTTPS 代理**：唯一标准的 HTTPS 代理方式
3. **协议透明**：可以代理任意 TCP 流量
4. **认证支持**：支持 HTTP 认证机制
5. **企业友好**：企业网络基础设施的标准组件

## 协议详解

### 请求格式

HTTP CONNECT 请求的标准格式：

```
CONNECT 请求格式：
┌────────────────────────────────────────────┐
│ CONNECT host:port HTTP/1.1                 │
│ Host: host:port                            │
│ [Proxy-Authorization: credentials]         │
│ [其他头部...]                               │
│                                            │
│ [空行结束请求头部]                           │
└────────────────────────────────────────────┘

请求行解析：
┌────────────────────────────────────────────┐
│ Method: "CONNECT" (必须大写)               │
│ Request-URI: "host:port"                   │
│   - host: 域名或 IP 地址                    │
│   - port: 目标端口                          │
│ HTTP-Version: "HTTP/1.1"                   │
└────────────────────────────────────────────┘
```

#### 请求示例

不同类型的 CONNECT 请求示例：

```
基本 CONNECT 请求（无认证）：
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
User-Agent: Mozilla/5.0
Proxy-Connection: Keep-Alive

带认证的 CONNECT 请求：
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Basic YWRtaW46cGFzc3dvcmQ=
User-Agent: Mozilla/5.0
Proxy-Connection: Keep-Alive

IPv4 地址 CONNECT 请求：
CONNECT 192.168.1.1:80 HTTP/1.1
Host: 192.168.1.1:80

IPv6 地址 CONNECT 请求：
CONNECT [2001:db8::1]:443 HTTP/1.1
Host: [2001:db8::1]:443

非标准端口 CONNECT 请求：
CONNECT example.com:8443 HTTP/1.1
Host: example.com:8443
```

### 响应格式

HTTP CONNECT 响应的标准格式：

```
CONNECT 成功响应：
┌────────────────────────────────────────────┐
│ HTTP/1.1 200 Connection Established       │
│ [响应头部...]                               │
│                                            │
│ [空行结束响应头部]                           │
│                                            │
│ [后续为隧道数据流]                           │
└────────────────────────────────────────────┘

成功响应示例：
HTTP/1.1 200 Connection Established
Proxy-Agent: Prism/1.0
Date: Sat, 17 May 2026 12:00:00 GMT
Connection: close

[隧道开始，后续为原始 TCP 数据]
```

#### 响应状态码

CONNECT 响应的常见状态码：

```
成功状态码：
┌────────────────────────────────────────────┐
│ 200: Connection Established                │
│   - 隧道建立成功                            │
│   - 可以开始传输数据                        │
└────────────────────────────────────────────┘

失败状态码：
┌────────────────────────────────────────────┐
│ 400: Bad Request                           │
│   - 请求格式错误                            │
│                                            │
│ 403: Forbidden                             │
│   - 目标地址被禁止                          │
│                                            │
│ 405: Method Not Allowed                    │
│   - 代理不支持 CONNECT 方法                 │
│                                            │
│ 407: Proxy Authentication Required        │
│   - 需要认证                                │
│                                            │
│ 408: Request Timeout                       │
│   - 请求超时                                │
│                                            │
│ 502: Bad Gateway                           │
│   - 无法连接目标服务器                      │
│                                            │
│ 504: Gateway Timeout                       │
│   - 连接目标服务器超时                      │
└────────────────────────────────────────────┘
```

#### 认证失败响应

当需要认证时，服务器返回 407 响应：

```
认证要求响应：
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Proxy"
Content-Type: text/html
Content-Length: 123

<html><body>
Proxy Authentication Required
</body></html>

客户端收到 407 后：
1. 提取认证方案和 realm
2. 使用用户名/密码计算 Authorization
3. 重发带 Proxy-Authorization 的请求
```

### HTTP 认证机制

HTTP CONNECT 支持多种认证机制：

#### HTTP Basic 认证

Basic 认证是最简单的认证方式：

```
Basic 认证流程：

请求（无认证）：
CONNECT example.com:443 HTTP/1.1
Host: example.com:443

响应（需要认证）：
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Proxy"

请求（带认证）：
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Basic YWRtaW46cGFzc3dvcmQ=

计算方式：
credentials = base64(username + ":" + password)
示例：
username: admin
password: password
credentials = base64("admin:password") = "YWRtaW46cGFzc3dvcmQ="
```

**安全性警告**：Basic 认证使用明文传输（Base64 可解码），必须配合 HTTPS 使用。

#### HTTP Digest 认证

Digest 认证提供更高的安全性：

```
Digest 认证流程：

响应（需要认证）：
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Digest
    realm="Proxy",
    nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
    algorithm=MD5,
    qop="auth"

请求（带认证）：
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Digest
    username="admin",
    realm="Proxy",
    nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093",
    uri="example.com:443",
    response="6629fae49393a05397450978507c4ef1",
    algorithm=MD5,
    qop="auth",
    nc=00000001,
    cnonce="0a4f113b"

计算方式：
HA1 = MD5(username:realm:password)
HA2 = MD5(method:uri)
response = MD5(HA1:nonce:nc:cnonce:qop:HA2)
```

#### NTLM 认证

NTLM 认证用于 Windows 环境：

```
NTLM 认证流程（多轮）：

1. 发送 Type 1 消息（协商）
Proxy-Authorization: NTLM TlRMTVNTUAABAAAA...

2. 服务器返回 Type 2 消息（挑战）
Proxy-Authenticate: NTLM TlRMTVNTUAACAAAA...

3. 发送 Type 3 消息（认证）
Proxy-Authorization: NTLM TlRMTVNTUAADAAAA...
```

### 隧道建立流程

完整的隧道建立流程：

```
完整流程：

Client                    HTTP Proxy               Target Server
  |                            |                          |
  |                            |                          |
  |--- CONNECT Request ------>|                          |
  |   CONNECT target:443       |                          |
  |   Host: target:443         |                          |
  |   [认证头部]                |                          |
  |                            |                          |
  |                            |--- 检查认证 ------------>|
  |                            |   (可选)                  |
  |                            |                          |
  |                            |--- 检查目标 ------------>|
  |                            |   (黑名单等)              |
  |                            |                          |
  |                            |--- TCP Connect --------->|
  |                            |   connect(target:443)    |
  |                            |                          |
  |                            |<-- TCP Connected -------|
  |                            |                          |
  |<-- 200 Response -----------|                          |
  |   Connection Established   |                          |
  |                            |                          |
  |                            |                          |
  |<======== 隧道数据流 =====================>|
  |                            |                          |
  |                            |                          |
  |                            |   (双向透明转发)          |
  |                            |                          |
```

### CONNECT 请求解析规则

CONNECT 请求的解析规则：

```
请求行解析：
┌────────────────────────────────────────────┐
│ 1. 方法必须是 "CONNECT"                     │
│                                            │
│ 2. Request-URI 必须是 host:port 格式        │
│   - host: 域名或 IP 地址                    │
│   - IPv6 地址必须用方括号：[IPv6]:port      │
│   - port: 十进制端口数字                    │
│                                            │
│ 3. HTTP 版本通常是 HTTP/1.1                │
│                                            │
│ 4. Host 头部应与 Request-URI 匹配          │
└────────────────────────────────────────────┘

常见解析错误：
┌────────────────────────────────────────────┐
│ ❌ CONNECT https://example.com HTTP/1.1    │
│    (不应包含 URL)                           │
│                                            │
│ ❌ CONNECT example.com HTTP/1.1            │
│    (缺少端口)                               │
│                                            │
│ ❌ connect example.com:443 HTTP/1.1        │
│    (方法必须大写)                           │
│                                            │
│ ❌ CONNECT 2001:db8::1:443 HTTP/1.1        │
│    (IPv6 缺少方括号)                        │
└────────────────────────────────────────────┘
```

### 隧道数据传输

隧道建立后的数据传输：

```
隧道数据传输规则：
┌────────────────────────────────────────────┐
│ 1. 响应后，代理停止解析 HTTP                │
│                                            │
│ 2. 后续数据作为原始 TCP 流转发              │
│                                            │
│ 3. 双向透明转发                             │
│   - 客户端→代理→目标                        │
│   - 目标→代理→客户端                        │
│                                            │
│ 4. 连接关闭规则                             │
│   - 任一端关闭，关闭另一端                  │
│   - 连接超时，关闭隧道                      │
└────────────────────────────────────────────┘

数据流示意：
┌────────────────────────────────────────────┐
│ Client → Proxy → Target                   │
│   [加密的 TLS 数据]                         │
│   [WebSocket 帧]                           │
│   [任意 TCP 数据]                           │
│                                            │
│ Target → Proxy → Client                   │
│   [加密的 TLS 数据]                         │
│   [WebSocket 帧]                           │
│   [任意 TCP 数据]                           │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### HTTP 解析器

Prism 解析 HTTP CONNECT 请求：

```cpp
// 文件: src/prism/protocol/http/parser.hpp
namespace psm::protocol::http {

auto parse_connect_request(std::string_view request)
    -> outcome::result<connect_request>
{
    // 解析请求行
    auto request_line_end = request.find('\r');
    if (request_line_end == std::string_view::npos) {
        return fault::code::invalid_request;
    }
    
    auto request_line = request.substr(0, request_line_end);
    auto parts = split(request_line, ' ');
    
    if (parts.size() != 3) {
        return fault::code::invalid_request;
    }
    
    // 检查方法
    if (parts[0] != "CONNECT") {
        return fault::code::invalid_method;
    }
    
    // 解析 host:port
    auto target = parts[1];
    auto colon_pos = target.find_last_of(':');
    if (colon_pos == std::string_view::npos) {
        return fault::code::invalid_target;
    }
    
    auto host = target.substr(0, colon_pos);
    auto port_str = target.substr(colon_pos + 1);
    
    // 处理 IPv6 地址
    if (host.starts_with('[') && host.ends_with(']')) {
        host = host.substr(1, host.size() - 2);
    }
    
    auto port = parse_port(port_str);
    if (!port) {
        return fault::code::invalid_port;
    }
    
    // 解析头部
    auto headers_start = request_line_end + 2;
    auto headers_end = request.find("\r\n\r\n", headers_start);
    if (headers_end == std::string_view::npos) {
        return fault::code::invalid_request;
    }
    
    auto headers = parse_headers(request.substr(headers_start, 
                                                  headers_end - headers_start));
    
    return connect_request{
        .host = std::string(host),
        .port = port.value(),
        .headers = headers.value(),
        .total_length = headers_end + 4
    };
}

} // namespace psm::protocol::http
```

### 认证处理

Prism 的 HTTP 认证：

```cpp
// 文件: src/prism/protocol/http/auth.cpp
auto verify_proxy_authorization(const http_headers& headers,
                                 const account_directory& accounts)
    -> outcome::result<std::string>
{
    auto auth_header = headers.find("Proxy-Authorization");
    if (!auth_header) {
        return fault::code::authentication_required;
    }
    
    auto [scheme, credentials] = parse_auth_header(*auth_header);
    
    if (scheme == "Basic") {
        auto decoded = base64_decode(credentials);
        auto colon_pos = decoded.find(':');
        if (colon_pos == std::string_view::npos) {
            return fault::code::invalid_credentials;
        }
        
        auto username = decoded.substr(0, colon_pos);
        auto password = decoded.substr(colon_pos + 1);
        
        auto entry = accounts.lookup(username);
        if (!entry || entry->password != password) {
            return fault::code::authentication_failed;
        }
        
        return username;
    }
    
    // 其他认证方案...
    return fault::code::unsupported_auth_scheme;
}

auto craft_auth_challenge(std::string_view realm)
    -> std::string
{
    return "HTTP/1.1 407 Proxy Authentication Required\r\n"
           "Proxy-Authenticate: Basic realm=\"" + std::string(realm) + "\"\r\n"
           "Content-Length: 0\r\n"
           "\r\n";
}
```

### 响应构造

Prism 构造 CONNECT 响应：

```cpp
// 文件: src/prism/protocol/http/response.cpp
auto craft_connect_response(uint16_t status_code,
                             std::string_view status_text,
                             const http_headers& extra_headers)
    -> std::string
{
    std::string response;
    
    // 状态行
    response = "HTTP/1.1 " + std::to_string(status_code) + " " + 
               std::string(status_text) + "\r\n";
    
    // 响应头部
    for (const auto& [name, value] : extra_headers) {
        response += name + ": " + value + "\r\n";
    }
    
    // 空行结束
    response += "\r\n";
    
    return response;
}

auto craft_success_response()
    -> std::string
{
    return "HTTP/1.1 200 Connection Established\r\n"
           "Proxy-Agent: Prism/1.0\r\n"
           "\r\n";
}

auto craft_failure_response(uint16_t status_code, std::string_view message)
    -> std::string
{
    auto body = "<html><body><h1>" + std::to_string(status_code) + 
                " " + std::string(message) + "</h1></body></html>";
    
    return "HTTP/1.1 " + std::to_string(status_code) + " " + 
           std::string(message) + "\r\n"
           "Content-Type: text/html\r\n"
           "Content-Length: " + std::to_string(body.size()) + "\r\n"
           "\r\n" + body;
}
```

### 隧道转发

Prism 的隧道数据转发：

```cpp
// 文件: src/prism/protocol/http/relay.cpp
auto relay_tunnel(net::socket client_socket,
                  net::socket target_socket)
    -> net::awaitable<void>
{
    // 双向转发协程
    auto self = co_await net::this_coroutine::executor;
    
    co_await net::co_spawn(self,
        tunnel_forward(client_socket, target_socket),
        net::deferred);
    
    co_await net::co_spawn(self,
        tunnel_forward(target_socket, client_socket),
        net::deferred);
}

auto tunnel_forward(net::socket source,
                    net::socket destination)
    -> net::awaitable<void>
{
    std::array<std::byte, 8192> buffer;
    
    while (true) {
        auto bytes_read = co_await source.async_read_some(
            net::buffer(buffer), net::use_awaitable);
        
        if (bytes_read == 0) {
            break; // 连接关闭
        }
        
        co_await net::async_write(destination,
            net::buffer(buffer.data(), bytes_read),
            net::use_awaitable);
    }
}
```

## 最佳实践

### 安全配置

1. **认证必须**：生产环境启用认证
2. **HTTPS 前置**：使用 HTTPS 保护认证数据
3. **黑名单机制**：限制可访问的目标地址
4. **日志记录**：记录 CONNECT 请求和结果

### 性能优化

1. **连接池**：复用到目标的 TCP 连接
2. **零拷贝**：使用零拷贝技术转发数据
3. **缓冲区**：使用合适的缓冲区大小
4. **协程架构**：使用异步非阻塞处理

### 错误处理

1. **超时设置**：设置连接和隧道超时
2. **优雅关闭**：正确处理连接关闭
3. **错误响应**：返回正确的 HTTP 状态码
4. **资源清理**：清理失败的连接资源

### 兼容性

1. **HTTP/1.1**：支持 HTTP/1.1 标准
2. **IPv6 支持**：正确处理 IPv6 地址格式
3. **头部处理**：正确处理 Host 头部
4. **代理头部**：理解 Proxy-Connection 等头部

## 常见问题

### Q1: CONNECT 方法只能用于 HTTPS 吗？

CONNECT 方法可以用于任何 TCP 流量：

- HTTPS 网站（最常见）
- WebSocket 连接
- SSH 隧道
- 其他 TCP 协议

但大多数浏览器只用它访问 HTTPS。

### Q2: 为什么 HTTPS 必须用 CONNECT？

HTTPS 需要端到端加密：

- 普通 HTTP 代理会解析请求
- 代理服务器无法解密 HTTPS
- CONNECT 建立隧道，保持加密完整性
- 代理只转发加密数据，不解析

### Q3: HTTP Basic 认证安全吗？

Basic 认证的安全性：

- 明文传输（Base64 可解码）
- 必须配合 HTTPS 使用
- 或使用 Digest 认证
- 或使用更安全的方案（如 OAuth）

### Q4: CONNECT 隧道如何关闭？

隧道关闭规则：

- 任一端关闭连接 → 代理关闭另一端
- 连接超时 → 代理关闭隧道
- 代理主动关闭 → 关闭两端
- 目标服务器关闭 → 代理关闭客户端连接

### Q5: 可以在 CONNECT 隧道上使用多路复用吗？

可以，但需要叠加协议：

- CONNECT + Smux/Yamux
- 需要目标服务器支持
- 或使用代理服务器的多路复用

### Q6: 如何检测 HTTP CONNECT 流量？

检测特征：

- 首行以 "CONNECT " 开始
- Request-URI 是 host:port 格式
- 常用端口 8080, 3128 等
- HTTP 头部结构

### Q7: Prism 如何处理 CONNECT 超时？

Prism 的超时处理：

- 请求解析超时（30 秒）
- 认证超时（30 秒）
- 目标连接超时（10 秒）
- 隧道空闲超时（5 分钟）

### Q8: CONNECT 与 SOCKS5 哪个更好？

各有优势：

| 场景 | 推荐 |
|------|------|
| Web 浏览器 | CONNECT（原生支持） |
| 任意应用 | SOCKS5（更通用） |
| UDP 流量 | SOCKS5 |
| 企业环境 | CONNECT（常见） |
| 性能敏感 | SOCKS5（更低开销） |

## 参考资料

- [RFC 7231 Section 4.3.6 - CONNECT](https://tools.ietf.org/html/rfc7231#section-4.3.6)
- [RFC 2616 Section 9.9 - CONNECT](https://tools.ietf.org/html/rfc2616#section-9.9)
- [RFC 7235 - HTTP Authentication](https://tools.ietf.org/html/rfc7235)
- [RFC 7617 - The 'Basic' HTTP Authentication Scheme](https://tools.ietf.org/html/rfc7617)

## 相关知识

- [[ref/protocol/socks5-rfc1928|SOCKS5]] — SOCKS5 代理协议
- [[ref/protocol/trojan|Trojan 协议]] — Trojan 代理
- [[ref/protocol/http-1-1|HTTP/1.1]] — HTTP/1.1 协议
- [[ref/network/tcp|TCP]] — TCP 协议基础