---
title: HTTP/1.1 协议
created: 2026-05-17
updated: 2026-05-27
layer: ref
tags: [protocol, http, web, proxy]
---

# HTTP/1.1 协议

> HTTP/1.1（HyperText Transfer Protocol version 1.1）是 Web 的基础应用层协议，由 RFC 7230-7235 定义。代理服务器的 HTTP CONNECT 隧道、请求解析和协议识别均基于此协议。

---

## 消息格式

### 请求消息

HTTP/1.1 请求由请求行、头部字段和可选消息体组成：

```
METHOD SP Request-URI SP HTTP-Version CRLF
Header-Name: Header-Value CRLF
[...更多头部...]
CRLF
[消息体]
```

**请求行示例**：

```
GET /index.html HTTP/1.1
POST /api/data HTTP/1.1
CONNECT example.com:443 HTTP/1.1
```

**完整请求示例**：

```http
GET /index.html HTTP/1.1
Host: example.com
User-Agent: PrismClient/1.0
Accept: text/html
Connection: keep-alive

```

### 响应消息

HTTP/1.1 响应由状态行、头部字段和可选消息体组成：

```
HTTP-Version SP Status-Code SP Reason-Phrase CRLF
Header-Name: Header-Value CRLF
[...更多头部...]
CRLF
[消息体]
```

**完整响应示例**：

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234
Connection: keep-alive

<html>...
```

### 消息体传输方式

HTTP/1.1 支持三种消息体传输机制：

| 方式 | 头部 | 说明 |
|------|------|------|
| Content-Length | `Content-Length: N` | 明确指定字节数 |
| 分块传输 | `Transfer-Encoding: chunked` | 每个块前标明大小 |
| 连接关闭 | 无 | 读取直到连接断开（HTTP/1.0 风格） |

**分块传输编码格式**：

```
chunk-size CRLF
chunk-data CRLF
[...更多块...]
0 CRLF
[trailer-headers] CRLF
```

---

## HTTP 方法

### 常用方法

| 方法 | 用途 | 是否有消息体 |
|------|------|-------------|
| GET | 获取资源 | 否 |
| POST | 提交数据 | 是 |
| PUT | 替换资源 | 是 |
| DELETE | 删除资源 | 可选 |
| HEAD | 获取头部（无体） | 否 |
| OPTIONS | 查询支持的方法 | 否 |
| PATCH | 部分更新 | 是 |
| **CONNECT** | 建立隧道 | 否 |

### CONNECT 方法（代理隧道）

CONNECT 方法是 HTTP 代理的核心，用于建立端到端的 TCP 隧道：

```
客户端                         代理服务器                     目标服务器
  │                              │                              │
  │── CONNECT target:443 ───────→│                              │
  │   Host: target:443           │                              │
  │                              │                              │
  │                              │── TCP 连接 ──────────────────→│
  │                              │                              │
  │←── 200 Connection Established│←── 连接成功 ─────────────────│
  │                              │                              │
  │═══════ 透明双向数据流 ════════│══════════════════════════════│
  │         (通常是 TLS)         │                              │
```

**CONNECT 请求格式**：

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Connection: keep-alive
```

**成功响应**：

```http
HTTP/1.1 200 Connection Established
```

**失败响应**：

```http
HTTP/1.1 502 Bad Gateway
Content-Type: text/plain
Content-Length: 12

Bad Gateway
```

**代理认证**：

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Basic dXNlcjpwYXNz
```

---

## 状态码

### 状态码分类

| 范围 | 类别 | 说明 |
|------|------|------|
| 100-199 | 信息性 | 请求已接收，继续处理 |
| 200-299 | 成功 | 请求成功处理 |
| 300-399 | 重定向 | 需要进一步操作 |
| 400-499 | 客户端错误 | 请求有语法或逻辑问题 |
| 500-599 | 服务端错误 | 服务器处理失败 |

### 代理相关状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|----------|
| 200 | OK / Connection Established | CONNECT 隧道建立成功 |
| 400 | Bad Request | 请求格式错误 |
| 401 | Unauthorized | 需要代理认证 |
| 403 | Forbidden | 访问被拒绝 |
| 407 | Proxy Authentication Required | 代理要求认证 |
| 408 | Request Timeout | 请求超时 |
| 413 | Payload Too Large | 请求体过大 |
| 421 | Misdirected Request | HTTP/2 中连接复用错误 |
| 502 | Bad Gateway | 上游服务器不可达 |
| 503 | Service Unavailable | 服务暂时不可用 |
| 504 | Gateway Timeout | 上游服务器超时 |

---

## 持久连接（Keep-Alive）

### 连接管理

HTTP/1.1 默认使用持久连接（Connection: keep-alive），允许在同一 TCP 连接上发送多个请求：

```
连接 1:
  请求 A ──→ 响应 A
  请求 B ──→ 响应 B
  请求 C ──→ 响应 C
  ...
  直到一方发送 Connection: close
```

**关键头部**：

| 头部 | 值 | 说明 |
|------|------|------|
| `Connection` | `keep-alive` | HTTP/1.1 默认行为 |
| `Connection` | `close` | 响应后关闭连接 |
| `Keep-Alive` | `timeout=30, max=100` | 保持参数（非标准） |

### 管道化（Pipelining）

HTTP/1.1 支持请求管道化，客户端可以连续发送多个请求而不必等待响应：

```
客户端                         服务器
  │── 请求 1 ──────────────────→│
  │── 请求 2 ──────────────────→│
  │── 请求 3 ──────────────────→│
  │←── 响应 1 ─────────────────│
  │←── 响应 2 ─────────────────│
  │←── 响应 3 ─────────────────│
```

**问题**：
- 服务器必须按序返回响应（队头阻塞）
- 代理服务器不总是正确支持
- 实践中使用较少，HTTP/2 通过多路复用解决

---

## 安全相关头部

### 传输安全

| 头部 | 说明 | 示例 |
|------|------|------|
| `Strict-Transport-Security` | 强制 HTTPS | `max-age=31536000; includeSubDomains` |
| `Upgrade-Insecure-Requests` | 请求 HTTPS 版本 | `1` |
| `Content-Security-Policy` | 内容安全策略 | `default-src 'self'` |

### 认证头部

| 头部 | 方向 | 说明 |
|------|------|------|
| `Authorization` | 客户端 → 服务器 | 资源认证凭据 |
| `Proxy-Authorization` | 客户端 → 代理 | 代理认证凭据 |
| `Proxy-Authenticate` | 代理 → 客户端 | 代理认证挑战 |
| `WWW-Authenticate` | 服务器 → 客户端 | 资源认证挑战 |

**Basic 认证示例**：

```http
# 代理发送挑战
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Proxy"

# 客户端提供凭据（Base64 编码）
CONNECT example.com:443 HTTP/1.1
Proxy-Authorization: Basic dXNlcjpwYXNz
```

### 隐私与追踪

| 头部 | 说明 |
|------|------|
| `X-Forwarded-For` | 客户端原始 IP 链 |
| `X-Forwarded-Proto` | 原始协议（http/https） |
| `Forwarded` | 标准化的转发信息（RFC 7239） |
| `Via` | 经过的代理/网关 |

---

## 代理模式

### 正向代理（Forward Proxy）

代理客户端请求，客户端知道代理的存在：

```http
# 客户端发送完整 URI
GET http://example.com/index.html HTTP/1.1
Host: example.com
```

### 反向代理（Reverse Proxy）

代理服务器响应，客户端不知道后端服务器：

```http
# 客户端发送相对 URI
GET /index.html HTTP/1.1
Host: www.example.com
```

### 透明代理

客户端不知道代理存在，代理通过拦截流量工作：

- 网络层拦截（iptables 重定向）
- 不需要客户端配置
- 常用于企业网络和防火墙

---

## 在 Prism 中的应用

### HTTP 代理处理流程

1. 读取请求行，判断方法类型
2. 若为 CONNECT → 建立隧道
3. 若为普通方法 → 正向代理转发
4. 解析 `Host` 头部确定目标
5. 检查 `Proxy-Authorization` 进行认证
6. 建立到目标的上游连接
7. 双向转发数据

详见 [[ref/protocol/http-proxy-spec|HTTP 代理规范]]。

---

## 相关参考

- RFC 7230: HTTP/1.1 Message Syntax and Routing
- RFC 7231: HTTP/1.1 Semantics and Content
- RFC 7235: HTTP/1.1 Authentication
- [[ref/protocol/http-proxy-spec|HTTP CONNECT 规范]]
- [[ref/protocol/http-connect|HTTP CONNECT 详解]]