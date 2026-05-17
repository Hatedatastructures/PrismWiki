---
title: Shadowsocks
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, shadowsocks]
---
# Shadowsocks 协议

Shadowsocks 是一种轻量级代理协议，使用 AEAD 加密算法。

## 协议概述

Shadowsocks 特性：
- 基于 AEAD 的加密
- 支持多种加密算法
- 支持 UDP
- 支持多种插件（obfs、v2ray-plugin等）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/shadowsocks.go` | Shadowsocks 适配器 |
| `transport/shadowsocks/core/cipher.go` | 加密核心 |
| `transport/shadowsocks/shadowaead/` | AEAD 实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "ss-proxy"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    udp: true
```

### 支持的加密算法

| Cipher | 描述 |
|--------|------|
| `aes-128-gcm` | AES-128-GCM |
| `aes-256-gcm` | AES-256-GCM |
| `chacha20-ietf-poly1305` | ChaCha20-Poly1305 |
| `2022-blake3-aes-128-gcm` | Shadowsocks 2022 |
| `2022-blake3-aes-256-gcm` | Shadowsocks 2022 |
| `2022-blake3-chacha20-poly1305` | Shadowsocks 2022 |

### obfs 插件

```yaml
proxies:
  - name: "ss-obfs"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: obfs
    plugin-opts:
      mode: tls
      host: bing.com
```

### v2ray-plugin 插件

```yaml
proxies:
  - name: "ss-v2ray"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: v2ray-plugin
    plugin-opts:
      mode: websocket
      tls: true
      host: server.example.com
      path: /ss
```

### ShadowTLS 插件

```yaml
proxies:
  - name: "ss-shadowtls"
    type: ss
    server: server.example.com
    port: 443
    cipher: aes-128-gcm
    password: your-password
    plugin: shadowtls
    plugin-opts:
      password: shadowtls-password
      host: bing.com
      version: 2
```

### Restls 插件

```yaml
proxies:
  - name: "ss-restls"
    type: ss
    server: server.example.com
    port: 443
    cipher: aes-128-gcm
    password: your-password
    plugin: restls
    plugin-opts:
      password: restls-password
      host: bing.com
      version-hint: tls12
```

### kcptun 插件

```yaml
proxies:
  - name: "ss-kcptun"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: kcptun
    plugin-opts:
      key: kcptun-key
      crypt: aes
      mode: fast
```

### UDP over TCP

```yaml
proxies:
  - name: "ss-uot"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    udp-over-tcp: true
    udp-over-tcp-version: 2
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `ss` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `cipher` | string | 是 | 加密算法 |
| `password` | string | 是 | 密码 |
| `udp` | bool | 否 | 启用 UDP |
| `plugin` | string | 否 | 插件名称 |
| `plugin-opts` | map | 否 | 插件配置 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Shadowsocks 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Crypto | 完全兼容 | 使用 AEAD 加密 |

## 相关文档

- [[shadowsocksr]] - ShadowsocksR 协议
- [[../../protocol/shadowsocks]] - Shadowsocks 协议规范
- [[../../crypto/aead]] - AEAD 加密
- [[../transport/shadowsocks]] - Shadowsocks 传输层