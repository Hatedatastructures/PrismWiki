---
title: "Sniffing 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/component/sniffer"
tags: [mihomo, sniffing, 嗅探, 协议检测, 流量识别]
created: 2026-05-17
updated: 2026-05-17
related: [dns, http, tls, quic]
---

# Sniffing 概览

**类别**: Mihomo 嗅探参考

## 概述

Sniffing（嗅探）是 Mihomo 的流量识别功能，通过分析连接初始数据来识别实际协议类型。嗅探功能允许 Mihomo 在代理流量时自动识别协议，从而实现精确的路由规则匹配。

### 嗅探原理

```
嗅探流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端 ──→ Mihomo ──→ 目标服务器           │
│              │                              │
│              ├── 1. 接收连接                │
│              ├── 2. 读取首包数据            │
│              ├── 3. 协议特征匹配            │
│              ├── 4. 识别协议类型            │
│              ├── 5. 应用路由规则            │
│              └── 6. 转发流量                 │
│                                             │
└─────────────────────────────────────────────┘
```

### 支持的嗅探类型

| 类型 | 协议 | 描述 |
|------|------|------|
| HTTP | HTTP/1.x | 识别 Host 头 |
| TLS | TLS 1.0-1.3 | 提取 SNI |
| QUIC | QUIC | 提取 SNI |
| DNS | DNS | DNS 请求识别 |
| STUN | STUN | NAT 穿透检测 |

## 配置位置

嗅探功能在 `sniffer` 配置块中启用：

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
  force-domain:
    - "+.google.com"
  skip-domain:
    - "miejue.org"
```

## 核心配置项

| 配置项 | 类型 | 说明 |
|--------|------|------|
| enable | bool | 全局启用开关 |
| parse-pure-ip | bool | 是否嗅探纯 IP 连接 |
| sniff | map | 协议嗅探配置 |
| force-domain | []string | 强制嗅探域名列表 |
| skip-domain | []string | 跳过嗅探域名列表 |
| force-ip | []string | 强制嗅探 IP 列表 |
| skip-ip | []string | 跳过嗅探 IP 列表 |

## 嗅探与路由

嗅探结果影响路由决策：

```yaml
rules:
  # 基于嗅探域名的规则
  - DOMAIN-SUFFIX,google.com,PROXY
  - DOMAIN-SUFFIX,github.com,PROXY

  # 基于嗅探协议的规则
  - GEOSITE,category-ads-all,REJECT

  # 嗅探结果用于匹配
  - PROCESS-NAME,chrome.exe,PROXY
```

## 性能考量

嗅探功能需要读取连接初始数据：

| 因素 | 影响 | 建议 |
|------|------|------|
| 首包大小 | 小于首包可能导致误判 | 部分协议需要更多数据 |
| 延迟 | 增加路由决策延迟 | 通常可忽略 |
| 内存 | 缓存嗅探结果 | 影响较小 |
| CPU | 协议匹配计算 | 影响较小 |

## 相关链接

- [[ref/mihomo/sniffing/enable|启用嗅探]] — 启用配置详解
- [[ref/mihomo/sniffing/sniffing-types|嗅探类型]] — 协议嗅探详解
- [[ref/mihomo/sniffing/ports|端口配置]] — 嗅探端口配置
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — TLS 协议分析
- [[ref/protocol/dns-over-udp|DNS over UDP]] — DNS 协议参考