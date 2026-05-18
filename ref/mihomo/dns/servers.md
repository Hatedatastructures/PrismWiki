---
title: "mihomo DNS servers 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, servers, nameserver]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS servers 配置

DNS servers 定义上游 DNS 服务器，包括 nameserver、fallback 等。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/resolver.go` | Resolver 结构定义 |
| `dns/doh.go` | DoH 客户端 |
| `dns/dot.go` | DoT 客户端 |
| `dns/doq.go` | DoQ 客户端 |

## NameServer 结构

```go
type NameServer struct {
    Net          string            // 协议类型
    Addr         string            // 服务器地址
    ProxyAdapter C.ProxyAdapter    // 代理适配器
    ProxyName    string            // 代理名称
    Params       map[string]string // 参数
    PreferH3     bool              // 优先 HTTP/3
}
```

## YAML 配置

### nameserver（默认 DNS）

```yaml
dns:
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
    - tls://dns.alidns.com:853
    - 223.5.5.5
```

### fallback（备用 DNS）

```yaml
dns:
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
    - tls://dns.google:853
```

### proxy-server-nameserver（代理节点 DNS）

```yaml
dns:
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
```

### direct-nameserver（直连 DNS）

```yaml
dns:
  direct-nameserver:
    - https://dns.alidns.com/dns-query
```

## DNS 协议类型

| 协议 | 格式 | 说明 |
|------|------|------|
| UDP | `ip:port` | 传统 UDP DNS |
| TCP | `tcp://ip:port` | TCP DNS |
| DoH | `https://url` | DNS over HTTPS |
| DoT | `tls://host:port` | DNS over TLS |
| DoQ | `quic://host:port` | DNS over QUIC |

## DoH 配置参数

```yaml
dns:
  nameserver:
    - https://dns.google/dns-query#h3=true&skip-cert-verify=true
```

| 参数 | 说明 |
|------|------|
| `h3=true` | 强制使用 HTTP/3 |
| `skip-cert-verify=true` | 跳过证书验证 |

## DoT 配置参数

```yaml
dns:
  nameserver:
    - tls://dns.google:853#skip-cert-verify=true&disable-reuse=true
```

| 参数 | 说明 |
|------|------|
| `skip-cert-verify=true` | 跳过证书验证 |
| `disable-reuse=true` | 禁用连接复用 |

## DoQ 配置

```yaml
dns:
  nameserver:
    - quic://dns.nextdns.io:853#skip-cert-verify=true
```

## 常用 DNS 服务器

### 国内 DNS

| 服务商 | DoH | DoT |
|--------|-----|-----|
| 阿里 DNS | `https://dns.alidns.com/dns-query` | `tls://dns.alidns.com:853` |
| 腾讯 DNS | `https://doh.pub/dns-query` | `tls://doh.pub:853` |
| 114 DNS | `https://dns.114.cn/dns-query` | - |

### 海外 DNS

| 服务商 | DoH | DoT |
|--------|-----|-----|
| Google | `https://dns.google/dns-query` | `tls://dns.google:853` |
| Cloudflare | `https://cloudflare-dns.com/dns-query` | `tls://1.1.1.1:853` |
| NextDNS | `https://dns.nextdns.io/dns-query` | `tls://dns.nextdns.io:853` |

## 完整配置示例

```yaml
dns:
  enable: true
  
  # 默认 nameserver（国内 DNS）
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
    
  # fallback DNS（海外 DNS，通过代理）
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
    
  # 代理节点域名解析
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query
    
  # 直连 DNS
  direct-nameserver:
    - https://dns.alidns.com/dns-query
```

## 相关文档

- [[overview]] - DNS 配置概述
- [[ref/mihomo/dns/fallback|fallback]] - fallback DNS
- [[ref/mihomo/dns/nameserver-policy|nameserver-policy]] - DNS 策略