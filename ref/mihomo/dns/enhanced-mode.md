---
title: "mihomo DNS enhanced-mode 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, enhanced-mode, fake-ip]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS enhanced-mode 配置

`enhanced-mode` 定义 DNS 增强模式，决定 DNS 处理策略。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/enhancer.go` | ResolverEnhancer 实现 |

## ResolverEnhancer 结构

```go
type ResolverEnhancer struct {
    mode          C.DNSMode
    fakeIPPool    *fakeip.Pool
    fakeIPPool6   *fakeip.Pool
    fakeIPSkipper *fakeip.Skipper
    fakeIPTTL     int
    mapping       *lru.LruCache[netip.Addr, string]
    useHosts      bool
}

func (h *ResolverEnhancer) FakeIPEnabled() bool {
    return h.mode == C.DNSFakeIP
}

func (h *ResolverEnhancer) MappingEnabled() bool {
    return h.mode == C.DNSFakeIP || h.mode == C.DNSMapping
}
```

## DNSMode 类型

```go
type DNSMode int

const (
    DNSNormal DNSMode = iota
    DNSFakeIP
    DNSMapping
)
```

## YAML 配置

### fake-ip 模式（推荐）

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
```

### redir-host 模式

```yaml
dns:
  enable: true
  enhanced-mode: redir-host
```

## 模式说明

| 模式 | 说明 | 特点 |
|------|------|------|
| `fake-ip` | 返回虚假 IP | 性能好，防泄露 |
| `redir-host` | 真实解析 | 需要真实 IP |

## fake-ip 模式原理

```
1. DNS 查询阶段
   应用查询 google.com
   -> mihomo 返回 198.18.x.x（fake-ip）
   
2. 连接阶段
   应用连接 198.18.x.x
   -> mihomo 还原为 google.com
   -> 通过代理建立连接
```

### fake-ip 优势

| 优势 | 说明 |
|------|------|
| 零 DNS 泄露 | 真实 DNS 通过代理发出 |
| 低延迟 | 本地立即返回结果 |
| 域名还原 | 代理收到正确 SNI |
| 防污染 | 不受本地污染影响 |

### fake-ip 局限

| 局限 | 解决方案 |
|------|----------|
| CDN 调度问题 | 使用 [[fake-ip-filter]] |
| NTP 时间同步 | 添加到 filter |
| 局域网域名 | 添加到 filter |

## fake-ip-range 配置

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
```

默认使用 `198.18.0.1/16`（IANA 保留地址段）。

## fake-ip-filter 配置

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "*.localhost"
    - "*.ntp.org"
    - "+.stun.*"
```

参见 [[fake-ip-filter]]。

## redir-host 模式

真实解析并缓存结果：

```
1. DNS 查询
   应用查询 google.com
   -> mihomo 通过代理查询真实 DNS
   
2. 连接
   应用连接真实 IP
   -> mihomo 根据规则分流
```

适用场景：
- 需要 IP 规则匹配
- 应用需要真实 IP

## 相关文档

- [[overview]] - DNS 配置概述
- [[fake-ip-filter]] - fake-ip 过滤
- [[servers]] - DNS 服务器配置