---
title: HTTP
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, http]
---
# HTTP 代理协议

HTTP CONNECT 代理通过 HTTP 协议建立隧道连接。

## 协议概述

HTTP 代理特性：
- 使用 HTTP CONNECT 方法建立隧道
- 支持 Basic 认证
- 支持 HTTPS（TLS）
- 仅支持 TCP 代理

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/http.go` | HTTP 代理适配器 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "http-proxy"
    type: http
    server: server.example.com
    port: 8080
```

### 认证配置

```yaml
proxies:
  - name: "http-auth"
    type: http
    server: server.example.com
    port: 8080
    username: user
    password: pass
```

### TLS 配置

```yaml
proxies:
  - name: "http-tls"
    type: http
    server: server.example.com
    port: 8080
    tls: true
    sni: custom.sni.com
    skip-cert-verify: false
```

### 自定义 Headers

```yaml
proxies:
  - name: "http-headers"
    type: http
    server: server.example.com
    port: 8080
    headers:
      User-Agent: "Custom-Agent"
      X-Forwarded-For: "1.2.3.4"
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `http` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `username` | string | 否 | 用户名 |
| `password` | string | 否 | 密码 |
| `tls` | bool | 否 | 启用 TLS |
| `sni` | string | 否 | TLS SNI |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `headers` | map | 否 | 自定义 HTTP 头 |

## HTTP CONNECT 隧道原理

HTTP CONNECT 方法是 HTTP/1.1 协议扩展（RFC 7231 Section 4.3.6），允许客户端通过 HTTP 代理建立 TCP 隧道。与普通的 HTTP 请求不同，CONNECT 方法不请求资源，而是请求代理服务器建立一个到目标服务器的透明 TCP 连接。

### CONNECT 方法工作流程

```
┌──────────────────────────────────────────────────────┐
│               HTTP CONNECT 完整流程                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Client                          HTTP Proxy          │
│    |                                 |                │
│    |  --- CONNECT target:443 ------> |                │
│    |       HTTP/1.1                  |                │
│    |       Host: target:443         |                │
│    |       Proxy-Authorization: ...  |                │
│    |                                 |                │
│    |  <--- HTTP/1.1 200 ----------  |                │
│    |       Connection Established    |                │
│    |                                 |                │
│    |  === 透明 TCP 隧道 ============ |                │
│    |  ←── 双向二进制流 ──→           |                │
│    |  (通常用于 TLS 流量)             |                │
│    |                                 |                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### CONNECT 请求帧格式

```
CONNECT target.example.com:443 HTTP/1.1\r\n
Host: target.example.com:443\r\n
Proxy-Authorization: Basic base64(user:pass)\r\n
User-Agent: Custom-Agent/1.0\r\n
Proxy-Connection: keep-alive\r\n
\r\n
```

| 行 | 内容 | 说明 |
|----|------|------|
| 请求行 | `CONNECT host:port HTTP/1.1` | 目标地址和协议版本 |
| Host | `Host: host:port` | 目标主机（必需） |
| Proxy-Authorization | `Basic base64(user:pass)` | 代理认证（可选） |
| 其他头部 | 自定义 HTTP 头 | mihomo 支持 `headers` 配置 |
| 空行 | `\r\n` | 头部结束标记 |

### CONNECT 响应格式

成功响应：
```
HTTP/1.1 200 Connection Established\r\n
\r\n
```

失败响应：
```
HTTP/1.1 407 Proxy Authentication Required\r\n
Proxy-Authenticate: Basic realm="proxy"\r\n
\r\n
```

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | Connection Established | 隧道建立成功 |
| 403 | Forbidden | 代理拒绝连接 |
| 405 | Method Not Allowed | 不支持 CONNECT 方法 |
| 407 | Proxy Authentication Required | 需要代理认证 |
| 502 | Bad Gateway | 目标服务器不可达 |
| 503 | Service Unavailable | 代理服务不可用 |

## 认证机制

### Basic 认证

HTTP 代理使用 RFC 7617 定义的 Basic 认证方案：

```
Proxy-Authorization: Basic dXNlcjpwYXNz
```

其中 `dXNlcjpwYXNz` 是 `user:pass` 的 Base64 编码。

**安全注意事项**：
- Basic 认证以明文传输凭据（Base64 不是加密）
- 应配合 TLS（HTTPS 代理）使用
- mihomo 支持 `tls: true` 对代理连接进行加密

### TLS 加密代理（HTTPS Proxy）

```yaml
proxies:
  - name: "https-proxy"
    type: http
    server: proxy.example.com
    port: 443
    tls: true
    sni: proxy.example.com
    username: user
    password: pass
```

连接流程：
1. 与代理服务器建立 TCP 连接
2. 进行 TLS 握手（验证代理服务器证书）
3. 在加密通道上发送 CONNECT 请求
4. 建立隧道后，客户端可在隧道内再次进行 TLS 握手（双重 TLS）

```
┌───────────────────────────────────────────────────┐
│              双重 TLS 架构                          │
├───────────────────────────────────────────────────┤
│                                                   │
│  Client ──[TLS1]──→ HTTP Proxy ──[TLS2]──→ Server │
│         │              │              │           │
│         │              │              │           │
│   TLS1: 客户端与       │        TLS2: 代理与       │
│         代理之间的      │        目标服务器之间的    │
│         加密通道        │        加密通道            │
│                                                   │
│  隧道内部（TLS1 之上）：                             │
│  [CONNECT request] → 代理处理                        │
│  [200 OK] ← 代理响应                                │
│  [TLS2 握手] → 通过隧道传输到目标服务器               │
│                                                   │
└───────────────────────────────────────────────────┘
```

### 自定义 Headers

mihomo 支持在 CONNECT 请求中添加自定义 HTTP 头部：

```yaml
headers:
  User-Agent: "Custom-Agent"
  X-Forwarded-For: "1.2.3.4"
  X-Custom-Header: "value"
```

常见用途：
- `User-Agent`：伪装客户端标识
- `X-Forwarded-For`：传递客户端 IP（部分代理需要）
- 自定义头部：用于代理服务器的特殊处理逻辑

## 错误处理

| 错误场景 | 状态码 | 处理 |
|---------|--------|------|
| 认证失败 | 407 | 检查用户名密码 |
| 目标拒绝 | 403/502 | 检查目标地址可达性 |
| 代理不可用 | 503 | 检查代理服务状态 |
| TLS 握手失败 | N/A | 检查证书和 SNI 配置 |
| 连接超时 | N/A | 检查网络连接 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | 标准 HTTP CONNECT |
| Pipeline | 完全兼容 | TCP 隧道流 |
| UDP 支持 | 不兼容 | HTTP 代理不支持 UDP |

## 相关文档

- [[ref/protocol/http-spec|http-spec]] - HTTP 协议规范
- [[dev/debugging/tcp|tcp]] - TCP 连接