---
title: ShadowsocksR
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, shadowsocksr]
---
# ShadowsocksR 协议

ShadowsocksR（SSR）是 Shadowsocks 的扩展版本，增加混淆和协议扩展。

## 协议概述

ShadowsocksR 特性：
- 扩展加密算法
- 支持混淆（obfs）
- 支持协议扩展
- 兼容原版 Shadowsocks

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/shadowsocksr.go` | SSR 适配器 |
| `transport/ssr/obfs/` | 混淆实现 |
| `transport/ssr/protocol/` | 协议实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "ssr-proxy"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: tls1.2_ticket_auth
    obfs-param: ""
    protocol: auth_aes128_md5
    protocol-param: ""
    udp: true
```

### HTTP Simple 混淆

```yaml
proxies:
  - name: "ssr-http"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: http_simple
    obfs-param: bing.com
    protocol: origin
```

### TLS 混淆

```yaml
proxies:
  - name: "ssr-tls"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: tls1.2_ticket_auth
    protocol: auth_aes128_sha1
```

## 支持的加密算法

| Cipher | 描述 |
|--------|------|
| `aes-128-cfb` | AES-128-CFB |
| `aes-192-cfb` | AES-192-CFB |
| `aes-256-cfb` | AES-256-CFB |
| `aes-128-ctr` | AES-128-CTR |
| `aes-192-ctr` | AES-192-CTR |
| `aes-256-ctr` | AES-256-CTR |
| `rc4-md5` | RC4-MD5（不推荐） |
| `chacha20` | ChaCha20 |
| `chacha20-ietf` | ChaCha20-IETF |
| `none` | 无加密 |

## 混淆类型

| Obfs | 描述 |
|------|------|
| `plain` | 无混淆 |
| `http_simple` | HTTP 混淆 |
| `http_post` | HTTP POST 混淆 |
| `tls1.2_ticket_auth` | TLS 混淆 |
| `random_head` | 随机头部 |

## 协议类型

| Protocol | 描述 |
|----------|------|
| `origin` | 原始协议 |
| `auth_sha1_v4` | SHA1 认证 |
| `auth_aes128_md5` | AES128-MD5 认证 |
| `auth_aes128_sha1` | AES128-SHA1 认证 |
| `auth_chain_a` | Chain A 认证 |
| `auth_chain_b` | Chain B 认证 |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `ssr` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `cipher` | string | 是 | 加密算法 |
| `password` | string | 是 | 密码 |
| `obfs` | string | 是 | 混淆类型 |
| `obfs-param` | string | 否 | 混淆参数 |
| `protocol` | string | 是 | 协议类型 |
| `protocol-param` | string | 否 | 协议参数 |
| `udp` | bool | 否 | 启用 UDP |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SSR 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Crypto | 部分 | 使用流加密（非 AEAD） |

## 相关文档

- [[shadowsocks]] - Shadowsocks 协议
- [[ref/protocol/shadowsocks-spec|shadowsocks-spec]] - Shadowsocks 协议规范
- [[../transport/ssr]] - SSR 传输层