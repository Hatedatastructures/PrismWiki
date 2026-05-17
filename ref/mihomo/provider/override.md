---
title: "Override"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/provider"
tags: [mihomo, provider, override, 覆盖, 节点配置]
created: 2026-05-17
updated: 2026-05-17
related: [overview, proxy-provider]
---

# Override

**类别**: Mihomo Provider 覆盖配置

## 概述

Override（覆盖）允许修改 Provider 提供的节点配置。通过覆盖配置，可以修改节点名称、添加跳板、调整 TLS 配置等，无需修改原始订阅。

## 基础配置

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      additional-prefix: "[Provider]"
      additional-suffix: ""
```

## 配置参数

### additional-prefix

```yaml
additional-prefix: "[Provider]"
```

在节点名称前添加前缀。

用途：标识节点来源。

### additional-suffix

```yaml
additional-suffix: " - Provider1"
```

在节点名称后添加后缀。

用途：区分不同 Provider 的节点。

### dialer-proxy

```yaml
dialer-proxy: "前置代理"
```

指定节点使用的前置代理（跳板）。

用途：链式代理、通过前置节点连接。

### skip-cert-verify

```yaml
skip-cert-verify: true
```

跳过证书验证。

| 值 | 说明 |
|-----|------|
| true | 跳过证书验证 |
| false | 验证证书 |

用途：解决证书问题，但降低安全性。

### udp-over-tcp

```yaml
udp-over-tcp: true
```

UDP over TCP 模式。

| 值 | 说明 |
|-----|------|
| true | UDP 通过 TCP 转发 |
| false | 直接转发 UDP |

用途：UDP 不稳定时使用。

### udp-over-tcp-version

```yaml
udp-over-tcp-version: 2
```

UDP over TCP 版本。

| 值 | 说明 |
|-----|------|
| 1 | 版本 1 |
| 2 | 版本 2（推荐） |

## 配置示例

### 添加名称标识

```yaml
proxy-providers:
  provider-1:
    type: http
    url: "https://provider1.com/proxies.yaml"
    override:
      additional-prefix: "[P1] "
      additional-suffix: ""

  provider-2:
    type: http
    url: "https://provider2.com/proxies.yaml"
    override:
      additional-prefix: "[P2] "
      additional-suffix: ""

proxy-groups:
  - name: "all"
    type: select
    use:
      - provider-1
      - provider-2
```

结果：节点名称显示为 `[P1] 香港节点`、`[P2] 美国节点`。

### 链式代理

```yaml
proxies:
  - name: "前置节点"
    type: trojan
    server: front-server.com
    port: 443
    password: "password"

proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      dialer-proxy: "前置节点"

proxy-groups:
  - name: "chain"
    type: select
    use:
      - my-provider
```

效果：通过 "前置节点" 连接 Provider 中的所有节点。

### 跳过证书验证

```yaml
proxy-providers:
  insecure-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      skip-cert-verify: true
```

用途：解决证书过期、自签名证书等问题。

### UDP over TCP

```yaml
proxy-providers:
  udp-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      udp-over-tcp: true
      udp-over-tcp-version: 2
```

用途：UDP 不稳定场景。

### 完整覆盖配置

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    path: ./provider/my-provider.yaml
    override:
      additional-prefix: "[订阅] "
      additional-suffix: ""
      dialer-proxy: ""
      skip-cert-verify: false
      udp-over-tcp: false
```

## 链式代理原理

```
链式代理流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端                                      │
│      │                                      │
│      │ 连接请求                              │
│      ▼                                      │
│  Mihomo                                     │
│      │                                      │
│      │ 使用 dialer-proxy                    │
│      ▼                                      │
│  前置节点 (跳板)                             │
│      │                                      │
│      │ 转发连接                              │
│      ▼                                      │
│  Provider 节点                              │
│      │                                      │
│      │ 最终连接                              │
│      ▼                                      │
│  目标服务器                                  │
│                                             │
└─────────────────────────────────────────────┘
```

## 多 Provider 区分

```yaml
proxy-providers:
  hk-provider:
    type: http
    url: "https://hk.example.com/proxies.yaml"
    override:
      additional-prefix: "[香港] "

  us-provider:
    type: http
    url: "https://us.example.com/proxies.yaml"
    override:
      additional-prefix: "[美国] "

  jp-provider:
    type: http
    url: "https://jp.example.com/proxies.yaml"
    override:
      additional-prefix: "[日本] "
```

## 注意事项

### dialer-proxy

- 前置代理必须是已定义的节点
- 式代理会增加延迟
- 部分协议可能不支持链式代理

### skip-cert-verify

- 降低安全性
- 仅在必要时使用
- 建议联系订阅提供商解决证书问题

### udp-over-tcp

- 增加 TCP 转发开销
- UDP 延迟可能增加
- 适用场景：UDP 不稳定或被阻断

## 相关链接

- [[ref/mihomo/provider/overview|Provider 概览]] — 提供者总览
- [[ref/mihomo/provider/proxy-provider|Proxy Provider]] — 代理提供者
- [[ref/mihomo/provider/health-check|Health Check]] — 健康检查配置