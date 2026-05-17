---
title: "mihomo GeoIP 规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, geoip, 地理位置规则]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo GeoIP 规则

GeoIP 规则根据 IP 归属地进行匹配。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/geoip.go` | GeoIP 规则实现 |
| `component/geodata/` | GeoData 加载模块 |
| `component/mmdb/` | MMDB 数据库模块 |

## GEOIP 结构

```go
type GEOIP struct {
    Base
    country     string
    adapter     string
    noResolveIP bool
    isSourceIP  bool
}

func (g *GEOIP) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    if !g.noResolveIP && !g.isSourceIP && helper.ResolveIP != nil {
        helper.ResolveIP()
    }
    
    ip := metadata.DstIP
    if g.isSourceIP {
        ip = metadata.SrcIP
    }
    if !ip.IsValid() {
        return false, ""
    }
    
    if g.country == "lan" {
        return g.isLan(ip), g.adapter
    }
    
    // MMDB lookup
    codes := mmdb.IPInstance().LookupCode(ip.AsSlice())
    return slices.Contains(codes, g.country), g.adapter
}

func NewGEOIP(country string, adapter string, isSrc, noResolveIP bool) (*GEOIP, error) {
    country = strings.ToLower(country)
    geoip := &GEOIP{
        country:     country,
        adapter:     adapter,
        noResolveIP: noResolveIP,
        isSourceIP:  isSrc,
    }
    if country == "lan" {
        return geoip, nil
    }
    if err := geodata.InitGeoIP(); err != nil {
        return nil, err
    }
    return geoip, nil
}
```

## YAML 配置

### 基本配置

```yaml
rules:
  - GEOIP,CN,DIRECT
  - GEOIP,US,Proxy
  - GEOIP,HK,Proxy
  - GEOIP,JP,Proxy
```

### 带 no-resolve 参数

```yaml
rules:
  - GEOIP,CN,DIRECT,no-resolve
```

### 源 IP 匹配

```yaml
rules:
  - SRC-GEOIP,CN,DIRECT
```

### LAN 匹配

```yaml
rules:
  - GEOIP,LAN,DIRECT
```

## 支持的国家代码

| 代码 | 国家/地区 |
|------|----------|
| `CN` | 中国大陆 |
| `US` | 美国 |
| `JP` | 日本 |
| `HK` | 香港 |
| `TW` | 台湾 |
| `SG` | 新加坡 |
| `KR` | 韩国 |
| `UK` | 英国 |
| `DE` | 德国 |
| `FR` | 法国 |
| `AU` | 澳大利亚 |
| `CA` | 加拿大 |
| `LAN` | 局域网 |

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `GEOIP` | `GEOIP,国家代码,策略,no-resolve` | 目标 IP GeoIP 匹配 |
| `SRC-GEOIP` | `SRC-GEOIP,国家代码,策略` | 源 IP GeoIP 匹配 |

## LAN 规则

`GEOIP,LAN` 匹配局域网 IP：

```go
func (g *GEOIP) isLan(ip netip.Addr) bool {
    return ip.IsPrivate() ||
        ip.IsUnspecified() ||
        ip.IsLoopback() ||
        ip.IsMulticast() ||
        ip.IsLinkLocalUnicast() ||
        resolver.IsFakeBroadcastIP(ip)
}
```

## MMDB 查询

```go
codes := mmdb.IPInstance().LookupCode(ip.AsSlice())
```

返回 IP 所属的国家代码列表。

## GeoData 配置

```yaml
geodata-mode: true
geox-url:
  geoip: "https://cdn.jsdelivr.net/gh/Loyalsoldier/geoip@release/geoip.dat"
  mmdb: "https://cdn.jsdelivr.net/gh/Loyalsoldier/geoip@release/Country.mmdb"
```

## 完整配置示例

```yaml
rules:
  # 局域网直连
  - GEOIP,LAN,DIRECT
  
  # 中国 IP 直连
  - GEOIP,CN,DIRECT,no-resolve
  
  # 其他 IP 走代理
  - MATCH,Proxy
```

## 相关文档

- [[overview]] - 规则系统概述
- [[geosite]] - GeoSite 规则
- [[ipcidr]] - IP CIDR 规则
- [[rule-set]] - 规则集