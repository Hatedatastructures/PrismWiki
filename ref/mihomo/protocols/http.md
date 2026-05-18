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

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | 标准 HTTP CONNECT |
| Pipeline | 完全兼容 | TCP 隧道流 |
| UDP 支持 | 不兼容 | HTTP 代理不支持 UDP |

## 相关文档

- [[ref/protocol/http-spec|http-spec]] - HTTP 协议规范
- [[dev/debugging/tcp|tcp]] - TCP 连接