---
title: ECH
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, ech]
---
# ECH（Encrypted Client Hello）

ECH 是 TLS 扩展，加密 Client Hello 以防止域名泄露。

## 技术概述

ECH 特性：
- 加密 TLS Client Hello
- 防止 SNI 泄露
- 支持 DNS HTTPS 记录查询
- 支持 Base64 配置或动态查询

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/ech.go` | ECH 配置解析 |
| `component/ech/ech.go` | ECH 实现 |

## YAML 配置示例

### 静态 ECH 配置

```yaml
proxies:
  - name: "proxy-ech"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      config: base64-encoded-ech-config-list
```

### 动态 DNS 查询

```yaml
proxies:
  - name: "proxy-ech-dns"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      query-server-name: ech.example.com
```

## ECH 配置字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enable` | bool | 是 | 启用 ECH |
| `config` | string | 否 | Base64 ECH 配置 |
| `query-server-name` | string | 否 | DNS HTTPS 查询域名 |

## 支持的协议

ECH 可用于以下协议：
- [[vless]]
- [[vmess]]
- [[trojan]]
- [[anytls]]
- [[hysteria]]
- [[hysteria2]]
- [[tuic]]
- [[trusttunnel]]

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | ECH 实现 |
| Pipeline | 完全兼容 | TLS 扩展 |
| Stealth | 完全兼容 | 防止 SNI 泄露 |

## 相关文档

- [[reality]] - Reality TLS 隐蔽
- [[core/stealth/ech|ech]] - ECH 技术详解