---
title: "config — DNS 解析器配置"
layer: core
source: "include/prism/resolve/dns/config.hpp"
module: "resolve/dns"
type: config
tags: [dns, config, upstream, rules, cache]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/dns
  - core/resolve/dns/upstream
  - core/resolve/dns/detail/rules
  - core/resolve/dns/detail/cache
---

# config — DNS 解析器配置

> 源码位置: `include/prism/resolve/dns/config.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]]
> 类型: header-only 配置定义

## 组件定位

该文件定义 DNS 解析器的全部配置类型，包括上游服务器、解析策略、域名规则和 IP 过滤等。所有容器类型均使用 PMR 多态内存资源分配器，支持运行时切换内存资源。

## 核心类型

### dns_protocol — 上游协议枚举

| 值 | 说明 | 默认端口 |
|----|------|----------|
| `udp` | 纯 UDP 协议，如 `1.2.3.4` 或 `udp://1.2.3.4` | 53 |
| `tcp` | TCP 协议，如 `tcp://1.2.3.4` | 53 |
| `tls` | TLS 协议 (DoT)，如 `tls://dns.example.com` | 853 |
| `https` | HTTPS 协议 (DoH)，如 `https://dns.example.com/dns-query` | 443 |

地址解析规则：无 scheme 前缀默认为 UDP；`tcp://` 使用 TCP；`tls://` 使用 TLS (DoT)；`https://` 使用 HTTPS (DoH)。

### dns_remote — 上游服务器配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `address` | memory::string | (必填) | 服务器地址（IP 或域名） |
| `protocol` | dns_protocol | `udp` | 传输协议类型 |
| `hostname` | memory::string | 从 address 提取 | TLS SNI / HTTPS Host 头 |
| `port` | uint16 | 按协议默认 | 服务端口 |
| `timeout_ms` | uint32 | 5000 | 单次查询超时（毫秒） |
| `http_path` | memory::string | `/dns-query` | DoH 路径 |
| `skip_cert_check` | bool | `false` | 跳过 TLS 证书验证 |

### resolve_mode — 解析查询模式

| 模式 | 查询策略 | 适用场景 | 延迟特征 |
|------|----------|----------|----------|
| `fastest` | 全部上游并发，选择 RTT 最低的 | 追求最低延迟 | 等于最快上游的 RTT |
| `first` | 全部上游并发，首个成功即返回 | 追求可靠性 | 等于首个成功上游的 RTT |
| `fallback` | 逐一串行尝试，前一个失败后才尝试下一个 | 上游有优先级 | 等于失败上游数 x RTT + 成功上游 RTT |

**建议**: `fastest` 模式 2-3 个上游为最优，过多上游增加并发开销但收益递减。

### address_rule — 地址映射规则

| 字段 | 类型 | 说明 |
|------|------|------|
| `domain` | memory::string | 匹配域名，支持通配符 `*.xxx.com` |
| `addresses` | memory::vector<ip::address> | 映射的地址列表 |
| `negative` | bool | 否定应答（NXDOMAIN），用于广告拦截 |

- `negative=false` — 静态地址映射，返回预定义 IP
- `negative=true` — 广告拦截，返回 NXDOMAIN

### cname_rule — CNAME 重定向规则

| 字段 | 类型 | 说明 |
|------|------|------|
| `domain` | memory::string | 源域名 |
| `target` | memory::string | CNAME 目标域名 |

将特定域名的 A/AAAA 查询重定向到另一个域名。

### config — 主配置结构体

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `servers` | vector\<dns_remote\> | 空 | 上游服务器列表 |
| `mode` | resolve_mode | `fastest` | 解析查询模式 |
| `timeout_ms` | uint32 | 5000 | 全局超时（毫秒） |
| `cache_enabled` | bool | `true` | 是否启用 DNS 缓存 |
| `cache_size` | size_t | 10000 | 缓存最大条目数 |
| `cache_ttl` | chrono::seconds | 120s | 缓存默认 TTL |
| `serve_stale` | bool | `true` | 过期后是否仍提供缓存数据 |
| `negative_ttl` | chrono::seconds | 300s | 负缓存 TTL |
| `ttl_min` | uint32 | 60 | 最小 TTL（秒） |
| `ttl_max` | uint32 | 86400 | 最大 TTL（秒） |
| `address_rules` | vector\<address_rule\> | 空 | 地址映射规则 |
| `cname_rules` | vector\<cname_rule\> | 空 | CNAME 重定向规则 |
| `disable_ipv6` | bool | `false` | 禁用 IPv6，跳过 AAAA 查询 |
| `blacklist_v4` | vector\<network_v4\> | 空 | IPv4 黑名单（CIDR） |
| `blacklist_v6` | vector\<network_v6\> | 空 | IPv6 黑名单（CIDR） |

所有 PMR 容器通过 `config(memory::resource_pointer mr)` 构造函数统一分配。

## 设计决策

### 为什么使用 PMR 容器存储配置？

**问题**: DNS 配置可能包含大量规则（地址映射、CNAME、黑名单），使用默认分配器会在运行时产生碎片化的堆分配。

**选择**: 所有容器使用 PMR 多态内存资源，构造时统一指定内存资源。

**后果**: 可以将配置分配到自定义内存池或竞技场上，运行时完全可控。

### 为什么 serve_stale 默认为 true？

**问题**: DNS 缓存过期后，如果所有上游都不可达，解析会完全阻塞。

**选择**: `serve_stale=true` 在过期数据后台刷新时继续返回旧数据。

**后果**: 网络不稳定场景下显著提升可用性，代价是可能返回略微过期的记录。

### 为什么需要 TTL 钳制？

**问题**: 某些 DNS 响应 TTL=0（禁止缓存）或 TTL 极长（864000s），两者都有问题。

**选择**: `ttl_min`/`ttl_max` 将实际 TTL 钳制到合理范围。

**后果**: `ttl_min=60` 防止无缓存放大查询量；`ttl_max=86400` 防止数据陈旧。

## 约束

### 所有容器使用 PMR 分配器

**类型**: 内存管理
**规则**: 所有 `memory::vector` 和 `memory::string` 成员必须通过 PMR 内存资源分配。构造函数接受 `memory::resource_pointer` 参数。
**违反后果**: 使用默认 PMR 资源（通常等价于标准分配器）

### Glaze 序列化字段映射

**类型**: 序列化一致性
**规则**: `glz::meta<config>` 中映射的字段必须与 struct 成员完全对应。新增字段需同步更新 Glaze 映射。
**违反后果**: JSON 反序列化时字段被静默忽略

## 缓存参数详解

| 参数 | 默认值 | 推荐范围 | 说明 |
|------|--------|----------|------|
| `cache_enabled` | `true` | - | 全局缓存开关 |
| `cache_size` | 10000 | 1000-100000 | 最大缓存条目数（约 500KB-2MB） |
| `cache_ttl` | 120s | 30s-600s | 默认 TTL（DNS 响应无 TTL 时使用） |
| `serve_stale` | `true` | - | 过期数据在后台刷新时仍可使用 |
| `negative_ttl` | 300s | 60s-600s | 失败域名缓存时间 |

### TTL 钳制范围

```
实际 TTL = clamp(响应中的 TTL, ttl_min, ttl_max)

建议场景:
  低延迟: ttl_min=30, ttl_max=3600
  标准:   ttl_min=60, ttl_max=86400
  高可用: ttl_min=120, ttl_max=43200
```

### IP 黑名单

使用 `net::ip::network_v4/v6` 表示 CIDR 网段。解析结果中的 IP 若命中黑名单网段则被移除，不返回给调用方。

典型配置：
- IPv4: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `0.0.0.0/32`
- IPv6: `::1/128`, `fe80::/10`

## 性能影响

| 上游数 | fastest 模式 | fallback 模式 |
|--------|-------------|---------------|
| 1 | 单次查询，延迟 = RTT | 同 fastest |
| 2 | 2 次并发，延迟 = min(RTT1, RTT2) | 最多 2 次串行 |
| 3+ | N 次并发，延迟 = min(RTTi) | 最多 N 次串行 |

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/resolve/dns/upstream|upstream]] — DNS 查询客户端
- [[core/resolve/dns/detail/rules|rules_engine]] — 域名规则引擎
- [[core/resolve/dns/detail/cache|cache]] — DNS 结果缓存
