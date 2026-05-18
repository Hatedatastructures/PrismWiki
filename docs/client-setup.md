---
title: 客户端配置
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 客户端配置

Mihomo/Clash 客户端连接 Prism 的配置示例。

## 功能简介

提供各协议客户端配置示例，包括 Trojan、VLESS、SS2022、Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel。

## 如何启用

在 Mihomo/Clash 的 `proxies` 部分添加节点配置，确保与服务端参数匹配。

## 协议示例

### Trojan

```yaml
- name: "Trojan"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  skip-cert-verify: true
```

### VLESS Reality

```yaml
- name: "VLESS Reality"
  type: vless
  server: your.server.com
  port: 8081
  uuid: "uuid-here"
  tls: true
  servername: www.microsoft.com
  client-fingerprint: chrome
  reality-opts:
    public-key: "public-key-base64"
    short-id: "short-id"
```

### SS2022 ShadowTLS

```yaml
- name: "SS2022 ShadowTLS"
  type: ss
  server: your.server.com
  port: 8081
  cipher: 2022-blake3-aes-128-gcm
  password: "base64-key"
  plugin: shadow-tls
  plugin-opts:
    host: www.apple.com
    password: shadowtls_password
    version: 3
```

## 常见问题

**Q: SNI 不匹配怎么办？**
A: 客户端 `servername` 或 `sni` 必须与服务端 `server_names` 一致。

**Q: 如何启用多路复用？**
A: 添加 `smux` 配置块并设置 `enabled: true`。

## 相关页面

- [[ref/mihomo/index|mihomo]]
- [[ref/mihomo/proxy-groups/overview|proxy-groups]]
- [[core/protocol/trojan|Trojan 协议]]
- [[core/stealth/reality/handshake|Reality 伪装]]
- [[core/stealth/shadowtls|ShadowTLS 伪装]]