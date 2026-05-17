---
title: "mihomo IP CIDR 规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, ipcidr, IP规则]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo IP CIDR 规则

IP CIDR 规则根据目标 IP 地址段进行匹配。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/ipcidr.go` | IP-CIDR 规则实现 |
| `rules/common/ipsuffix.go` | IP-SUFFIX 规则实现 |
| `rules/common/ipasn.go` | IP-ASN 规则实现 |

## IPCIDR 结构

```go
type IPCIDR struct {
    Base
    ipnet       netip.Prefix
    adapter     string
    isSourceIP  bool
    noResolveIP bool
}

func (i *IPCIDR) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    if !i.noResolveIP && !i.isSourceIP && helper.ResolveIP != nil {
        helper.ResolveIP()
    }
    
    ip := metadata.DstIP
    if i.isSourceIP {
        ip = metadata.SrcIP
    }
    return ip.IsValid() && i.ipnet.Contains(ip.WithZone("")), i.adapter
}
```

## YAML 配置

### 基本配置

```yaml
rules:
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
```

### 带 no-resolve 参数

```yaml
rules:
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - IP-CIDR,91.108.4.0/22,Telegram,no-resolve
```

### IPv6 配置

```yaml
rules:
  - IP-CIDR6,2001:db8::/32,DIRECT,no-resolve
```

### 源 IP 匹配

```yaml
rules:
  - SRC-IP-CIDR,192.168.1.100/32,Proxy
```

## 常用私有 IP 段

```yaml
rules:
  # IPv4 私有地址
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve      # 本地回环
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve       # A类私有
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve    # B类私有
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve   # C类私有
  - IP-CIDR,100.64.0.0/10,DIRECT,no-resolve    # Carrier-grade NAT
```

## Telegram IP 段

```yaml
rules:
  - IP-CIDR,91.108.4.0/22,Telegram,no-resolve
  - IP-CIDR,91.108.8.0/22,Telegram,no-resolve
  - IP-CIDR,91.108.12.0/22,Telegram,no-resolve
  - IP-CIDR,91.108.16.0/22,Telegram,no-resolve
  - IP-CIDR,91.108.56.0/22,Telegram,no-resolve
  - IP-CIDR,149.154.160.0/20,Telegram,no-resolve
```

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `IP-CIDR` | `IP-CIDR,CIDR,策略,no-resolve` | IPv4 CIDR 匹配 |
| `IP-CIDR6` | `IP-CIDR6,CIDR,策略,no-resolve` | IPv6 CIDR 匹配 |
| `SRC-IP-CIDR` | `SRC-IP-CIDR,CIDR,策略` | 源 IP CIDR 匹配 |

## 参数说明

| 参数 | 说明 |
|------|------|
| `no-resolve` | 不进行 DNS 解析，仅匹配已解析的 IP |

## no-resolve 使用场景

1. **私有 IP 段**：域名不可能解析为私有 IP
2. **已知 IP 段**：如 Telegram 固定 IP 段
3. **性能优化**：减少不必要的 DNS 查询

## 选项函数

```go
type IPCIDROption func(*IPCIDR)

func WithIPCIDRSourceIP(b bool) IPCIDROption {
    return func(i *IPCIDR) {
        i.isSourceIP = b
    }
}

func WithIPCIDRNoResolve(noResolve bool) IPCIDROption {
    return func(i *IPCIDR) {
        i.noResolveIP = noResolve
    }
}
```

## IP-SUFFIX 规则

```yaml
rules:
  - IP-SUFFIX,8.8.8.8/24,Proxy
  - SRC-IP-SUFFIX,192.168.1.0/24,Proxy
```

## IP-ASN 规则

```yaml
rules:
  - IP-ASN,13335,Proxy        # Cloudflare ASN
  - SRC-IP-ASN,4134,DIRECT    # 中国电信 ASN
```

## 相关文档

- [[overview]] - 规则系统概述
- [[geoip]] - GeoIP 规则
- [[rule-set]] - 规则集