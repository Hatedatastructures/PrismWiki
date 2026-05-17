---
title: "Shortcuts"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/script"
tags: [mihomo, script, shortcuts, 快捷配置, 节点配置]
created: 2026-05-17
updated: 2026-05-17
related: [overview, rule-script]
---

# Shortcuts

**类别**: Mihomo 脚本功能

## 概述

Shortcuts（快捷配置）允许预设节点配置修改项，通过标签快速应用到节点。简化批量节点配置修改。

## 基础配置

### 定义 Shortcuts

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"
    dns: "skip-cert-verify=true"
```

### 应用 Shortcuts

```yaml
proxies:
  - name: ss-node
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: quic
```

## 配置参数

### shortcuts 定义

```yaml
shortcuts:
  name: "key1=value1&key2=value2&key3=value3"
```

格式：`key=value` 组合，用 `&` 分隔。

### shortcuts 应用

```yaml
proxies:
  - name: node
    # ...
    shortcuts: shortcut-name
```

节点中使用 `shortcuts` 字段引用预设。

## 支持的配置项

Shortcuts 支持修改以下配置：

| 配置项 | 说明 |
|--------|------|
| udp-over-tcp | UDP over TCP |
| udp-over-tcp-version | UDP over TCP 版本 |
| skip-cert-verify | 跳过证书验证 |
| additional-prefix | 名称前缀 |
| additional-suffix | 名称后缀 |

## 配置示例

### 单配置项

```yaml
script:
  shortcuts:
    udp: "udp-over-tcp=true"

proxies:
  - name: node-1
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: udp
```

### 多配置项

```yaml
script:
  shortcuts:
    full: "udp-over-tcp=true&udp-over-tcp-version=2&skip-cert-verify=true"

proxies:
  - name: node-1
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: full
```

### 多 Shortcuts 定义

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"
    dns: "skip-cert-verify=true"
    name: "additional-prefix=[快捷]"

proxies:
  - name: node-1
    type: ss
    server: server1.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: quic

  - name: node-2
    type: ss
    server: server2.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: dns
```

### Provider 中使用

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"

proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      shortcuts: quic
```

## UDP over TCP 示例

### 启用 UDP over TCP

```yaml
script:
  shortcuts:
    uot: "udp-over-tcp=true&udp-over-tcp-version=2"

proxies:
  - name: ss-uot
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: uot
```

### 批量应用

```yaml
script:
  shortcuts:
    uot: "udp-over-tcp=true&udp-over-tcp-version=2"

proxies:
  - name: node-1
    type: ss
    server: server1.com
    shortcuts: uot
  - name: node-2
    type: ss
    server: server2.com
    shortcuts: uot
  - name: node-3
    type: ss
    server: server3.com
    shortcuts: uot
```

## 证书验证示例

### 跳过证书验证

```yaml
script:
  shortcuts:
    insecure: "skip-cert-verify=true"

proxies:
  - name: node-insecure
    type: trojan
    server: server.com
    port: 443
    password: "password"
    shortcuts: insecure
```

## 名称修改示例

### 添加前缀

```yaml
script:
  shortcuts:
    label: "additional-prefix=[订阅A] "

proxies:
  - name: 香港
    type: ss
    server: server.com
    shortcuts: label
    # 实际名称: [订阅A] 香港
```

### 添加后缀

```yaml
script:
  shortcuts:
    suffix: "additional-suffix=-Premium"

proxies:
  - name: 美国
    type: ss
    server: server.com
    shortcuts: suffix
    # 实际名称: 美国-Premium
```

## Shortcuts vs Override

| 方式 | 配置位置 | 适用场景 |
|------|----------|----------|
| Shortcuts | 节点级 | 单个节点配置 |
| Override | Provider级 | Provider 批量配置 |

Shortcuts 更灵活，Override 更适合批量处理。

## 与 Override 配合

```yaml
script:
  shortcuts:
    uot: "udp-over-tcp=true&udp-over-tcp-version=2"

proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    override:
      additional-prefix: "[订阅] "
      shortcuts: uot
```

Override 先应用，Shortcuts 后应用。

## 配置生效顺序

```
配置生效顺序：
┌─────────────────────────────────────────────┐
│                                             │
│  1. Provider 原始配置                        │
│      │                                      │
│      ▼                                      │
│  2. Override 配置                            │
│      │                                      │
│      ▼                                      │
│  3. Shortcuts 配置                           │
│      │                                      │
│      ▼                                      │
│  4. 最终节点配置                             │
│                                             │
└─────────────────────────────────────────────┘
```

## 相关链接

- [[ref/mihomo/script/overview|Script 概览]] — 脚本功能总览
- [[ref/mihomo/script/rule-script|Rule Script]] — 规则脚本详解
- [[ref/mihomo/provider/override|Override]] — Provider 覆盖配置