---
title: SOCKS5
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, socks5]
---
# SOCKS5 协议

SOCKS5 是一种网络代理协议，支持 TCP 和 UDP 传输，提供认证机制。

## 协议概述

SOCKS5（Socket Secure 5）是最广泛使用的代理协议之一：
- 支持 TCP 和 UDP 代理
- 支持用户名/密码认证
- 支持 TLS 加密（SOCKS5 over TLS）
- 支持无认证模式

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/socks5.go` | SOCKS5 适配器实现 |
| `transport/socks5/socks5.go` | SOCKS5 协议传输层 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "socks5-proxy"
    type: socks5
    server: server.example.com
    port: 1080
```

### 认证配置

```yaml
proxies:
  - name: "socks5-auth"
    type: socks5
    server: server.example.com
    port: 1080
    username: user
    password: pass
```

### TLS 配置

```yaml
proxies:
  - name: "socks5-tls"
    type: socks5
    server: server.example.com
    port: 1080
    tls: true
    skip-cert-verify: false
    fingerprint: chrome
```

### 完整配置选项

```yaml
proxies:
  - name: "socks5-full"
    type: socks5
    server: server.example.com
    port: 1080
    username: user
    password: pass
    tls: true
    skip-cert-verify: false
    fingerprint: chrome
    udp: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `socks5` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `username` | string | 否 | 用户名 |
| `password` | string | 否 | 密码 |
| `tls` | bool | 否 | 启用 TLS |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `fingerprint` | string | 否 | TLS 指纹 |
| `udp` | bool | 否 | 启用 UDP |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SOCKS5 作为标准协议 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 部分 | UDP over TCP 需特殊处理 |

## 相关文档

- [[socks4]] - SOCKS4 协议
- [[../transport/socks5]] - SOCKS5 传输层
- [[core/crypto/aead|aead]] - AEAD 加密
- [[ref/protocol/socks5-spec|socks5-spec]] - SOCKS5 协议规范