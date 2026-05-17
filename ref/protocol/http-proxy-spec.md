---
title: "HTTP 代理规范"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 7230, RFC 7231"
tags: [协议, http, 代理, connect, 隧道]
created: 2026-05-17
updated: 2026-05-17
---

# HTTP 代理规范

**类别**: 协议

## 概述

HTTP 代理有两种工作模式：正向代理（Forward Proxy）和反向代理（Reverse Proxy）。在代理服务器场景中，HTTP CONNECT 方法用于建立 TCP 隧道，是 HTTP 代理的核心功能。

### 代理模式

HTTP 代理的两种模式：

| 模式 | 说明 | 用途 |
|------|------|------|
| **正向代理** | 代理客户端请求 | 企业代理、隐私保护 |
| **反向代理** | 代理服务端响应 | 负载均衡、缓存 |
| **CONNECT 隧道** | 建立 TCP 隧道 | HTTPS 代理、WebSocket |

### CONNECT 方法

HTTP CONNECT 是代理隧道的核心：

```
HTTP CONNECT 流程：

客户端 → 代理服务器:
CONNECT target.example.com:443 HTTP/1.1
Host: target.example.com
Proxy-Authorization: Basic <credentials>

代理服务器 → 客户端:
HTTP/1.1 200 Connection Established

后续：
客户端 ↔ 代理 ↔ 目标服务器
- TLS 握手（HTTPS）
- 加密数据传输
```

## CONNECT 方法

### 请求格式

CONNECT 请求的标准格式：

```
CONNECT 请求格式：

CONNECT <host>:<port> HTTP/1.1
Host: <host>:<port>
[Proxy-Authorization: Basic <credentials>]
[Proxy-Connection: Keep-Alive]
[其他头部...]

示例：
CONNECT www.example.com:443 HTTP/1.1
Host: www.example.com:443
Proxy-Authorization: Basic dXNlcjpwYXNz
User-Agent: curl/7.68.0

关键说明：
- CONNECT 后的 URI 格式为 host:port
- Host 头部必须与 CONNECT 目标一致
- 请求路径是固定格式，不是 URL
```

### 响应格式

CONNECT 响应的标准格式：

```
CONNECT 响应格式：

成功响应：
HTTP/1.1 200 Connection Established
[Date: ...]
[其他头部...]

失败响应：
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="proxy"
Content-Length: 0

常见状态码：
- 200: 连接成功，隧道建立
- 400: 请求格式错误
- 403: 禁止访问
- 407: 需要代理认证
- 502: 目标服务器错误
- 504: 连接超时
```

### 隧道建立

CONNECT 成功后的隧道行为：

```
隧道建立后：

1. 代理服务器停止解析 HTTP
   - 不处理后续 HTTP 请求
   - 只转发原始 TCP 数据

2. 数据双向转发
   - 客户端 → 代理 → 目标服务器
   - 目标服务器 → 代理 → 客户端

3. 典型应用：HTTPS 隧道
   客户端在隧道上完成 TLS 握手
   TLS 连接：客户端 ↔ 目标服务器
   代理只转发加密数据，无法解密

4. 隧道关闭
   - 任一端关闭 TCP 连接
   - 代理关闭另一端连接
```

## 代理认证

### Basic 认证

HTTP Basic 代理认证：

```
Proxy-Authorization 格式：

Proxy-Authorization: Basic <base64-encoded>

编码内容：
username:password → Base64 编码

示例：
username = "user", password = "pass"
原始字符串 = "user:pass"
Base64 编码 = "dXNlcjpwYXNz"

Proxy-Authorization: Basic dXNlcjpwYXNz

安全警告：
- Base64 不是加密，只是编码
- 凭据明文可见（解码后）
- 必须在 TLS 连接上使用
```

### 407 响应

代理认证失败时的响应：

```
407 Proxy Authentication Required：

HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="proxy"
Content-Type: text/plain
Content-Length: 0

客户端收到 407 后：
1. 获取认证方案（Basic）
2. 提供 Proxy-Authorization
3. 重发 CONNECT 请求

认证头部对比：
- Authorization: 目标服务器认证
- Proxy-Authorization: 代理服务器认证
```

## HTTP 请求转发

### 正向代理

HTTP GET/POST 通过正向代理：

```
HTTP 请求转发（非 CONNECT）：

客户端 → 代理：
GET http://www.example.com/path HTTP/1.1
Host: www.example.com
Proxy-Authorization: Basic <credentials>

代理处理：
1. 解析完整 URL
2. 解析 Host 头部
3. 建立到目标服务器的连接
4. 转发请求（去除代理相关头部）

代理 → 目标服务器：
GET /path HTTP/1.1
Host: www.example.com
[其他头部...]

目标服务器 → 代理 → 客户端：
HTTP/1.1 200 OK
Content-Type: text/html
[响应内容...]

注意：
- URL 使用完整格式 http://host/path
- 代理需要解析 URL 提取主机和路径
- 转发时将完整 URL 改为相对路径
```

### 头部处理

代理的头部处理规则：

```
代理头部处理：

移除的头部：
- Proxy-Authorization: 已完成认证
- Proxy-Connection: 转换为 Connection
- 其他 Proxy- 前缀头部

添加/修改的头部：
- Via: 记录代理路径
- X-Forwarded-For: 记录客户端 IP
- X-Forwarded-Proto: 记录原始协议

Via 头部示例：
Via: 1.1 proxy.example.com (squid/3.5.20)

X-Forwarded-For 示例：
X-Forwarded-For: 192.168.1.100, 10.0.0.1
```

## HTTPS 隧道详解

### 隧道上的 TLS

HTTPS 通过 CONNECT 隧道：

```
HTTPS 隧道流程：

1. 建立 CONNECT 隧道
   客户端 → 代理:
   CONNECT www.example.com:443 HTTP/1.1
   
   代理 → 客户端:
   HTTP/1.1 200 Connection Established

2. TLS 握手（在隧道上）
   客户端 → 目标服务器（通过代理）:
   ClientHello
   
   目标服务器 → 客户端（通过代理）:
   ServerHello, Certificate, Finished

3. 加密数据传输
   客户端 ↔ 目标服务器: TLS 记录层加密
   代理只转发加密数据

代理视角：
- 无法看到 TLS 内容
- 无法解密 HTTPS 数据
- 只能看到隧道流量
```

### 隧道安全

CONNECT 隧道的安全特性：

```
隧道安全分析：

安全特性：
- 代理无法解密 HTTPS 内容
- TLS 连接端点是目标服务器
- 证书验证由客户端完成

潜在风险：
- 代理可以看到目标主机名（CONNECT 请求）
- 代理可以拒绝建立隧道
- MITM 代理可以拦截（需要客户端信任其证书）

最佳实践：
- 使用可信代理
- 不信任未知代理的证书
- 在隧道上验证服务器证书
```

## WebSocket 隧道

### WebSocket 代理

WebSocket 通过 HTTP CONNECT 隧道：

```
WebSocket 隧道流程：

1. 建立 CONNECT 隧道
   CONNECT ws.example.com:443 HTTP/1.1
   
2. WebSocket 握手（在隧道上）
   客户端 → 目标服务器:
   GET /ws HTTP/1.1
   Host: ws.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: <key>
   Sec-WebSocket-Version: 13
   
   目标服务器 → 客户端:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: <accept>

3. WebSocket 数据帧传输
   客户端 ↔ 目标服务器: WebSocket 帧
   代理转发原始数据

注意：
- WebSocket 需要先建立隧道
- WebSocket 握手在隧道上完成
- 代理不解析 WebSocket 帧
```

## 在 Prism 中的应用

### HTTP 入站

Prism 支持 HTTP 作为入站协议：

```
Prism HTTP 入站配置：

{
    "type": "http",
    "listen": "0.0.0.0:8080",
    "users": [
        {
            "username": "user",
            "password": "pass"
        }
    ]
}

处理流程：
1. 接收 HTTP 请求
2. 检查 CONNECT 方法
3. 认证（如果配置了用户）
4. 建立 CONNECT 隧道或转发请求
5. 双向转发数据
```

详见 [[protocol/http|http]]。

### HTTP 流处理器

Prism 的 HTTP 流处理：

```
HTTP 流处理器：

class http_stream_handler {
    // 解析 HTTP 请求
    auto parse_request(buffer) → request;
    
    // 处理 CONNECT 方法
    auto handle_connect(target_address);
    
    // 处理认证
    auto check_auth(Proxy-Authorization) → auth_result;
    
    // 发送响应
    auto send_response(status, headers);
};
```

## 与 SOCKS5 的对比

### 功能对比

| 特性 | HTTP CONNECT | SOCKS5 |
|------|--------------|--------|
| 协议类型 | HTTP 子方法 | 独立协议 |
| 认证 | HTTP Basic | 自定义认证子协议 |
| 地址类型 | 仅 host:port | IPv4/域名/IPv6 |
| UDP 支持 | 无 | UDP ASSOCIATE |
| 普通请求 | 支持 GET/POST | 仅连接请求 |
| 隧道建立 | HTTP 请求/响应 | 二进制协议 |

### 性能对比

```
性能对比：

HTTP CONNECT：
- 请求/响应文本解析开销
- 头部处理开销
- 适合 HTTP 客户端直接使用

SOCKS5：
- 二进制协议，解析更快
- 更少的数据开销
- 需要专用客户端支持

选择建议：
- 浏览器环境：HTTP CONNECT 更方便
- 专用代理场景：SOCKS5 更高效
```

## 参见

- [[ref/protocol/socks5-spec|SOCKS5 规范]] — SOCKS5 协议
- [[ref/protocol/http-1-1|HTTP/1.1]] — HTTP 协议详解
- [[protocol/http|http]] — HTTP 实现接口
- [[dispatch/handlers/http|HTTP 处理器]] — Prism 实现
- [[ref/protocol/tcp-basics|TCP 基础]] — TCP 协议原理