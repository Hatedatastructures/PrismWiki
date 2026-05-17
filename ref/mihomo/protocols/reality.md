---
title: Reality
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, reality]
---
# Reality TLS 隐蔽技术

Reality 是一种 TLS 隐蔽技术，使代理流量伪装成正常 TLS 连接。

## 技术概述

Reality 特性：
- TLS 伪装技术
- 使用真实网站作为伪装目标
- 支持 X25519 密钥交换
- 支持 X25519MLKEM768（后量子安全）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/reality.go` | Reality 配置解析 |
| `component/tls/reality.go` | Reality TLS 实现 |

## YAML 配置示例

### VLESS + Reality

```yaml
proxies:
  - name: "vless-reality"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

### Trojan + Reality

```yaml
proxies:
  - name: "trojan-reality"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

### VMess + Reality

```yaml
proxies:
  - name: "vmess-reality"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

## Reality 配置字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `public-key` | string | 是 | X25519 公钥（Base64URL） |
| `short-id` | string | 否 | Short ID（Hex） |
| `support-x25519mlkem768` | bool | 否 | 支持后量子安全 |

## 客户端指纹

| Fingerprint | 描述 |
|-------------|------|
| `chrome` | Chrome TLS 指纹 |
| `firefox` | Firefox TLS 指纹 |
| `safari` | Safari TLS 指纹 |
| `edge` | Edge TLS 指纹 |
| `ios` | iOS TLS 指纹 |
| `android` | Android TLS 指纹 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Reality 实现 |
| Pipeline | 完全兼容 | TLS 流 |
| Stealth | 完全兼容 | TLS 隐蔽技术 |

## 相关文档

- [[ech]] - ECH 加密 Client Hello
- [[../../stealth/reality]] - Reality 隐蔽技术详解
- [[../../crypto/x25519]] - X25519 密钥交换