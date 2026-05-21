---
title: "config — DNS 解析器配置"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/config.hpp"
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

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/config.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]]
> 类型: header-only 配置定义

## 组件定位

该文件定义 DNS 解析器的全部配置类型，包括上游服务器、解析策略、域名规则和 IP 过滤等。所有容器类型均使用 PMR 多态内存资源分配器，支持运行时切换内存资源。

## 核心类型

### dns_protocol — 上游协议枚举

```cpp
enum class dns_protocol : std::uint8_t {
    udp,    // 纯 UDP 协议，如 1.2.3.4 或 udp://1.2.3.4
    tcp,    // TCP 协议，如 tcp://1.2.3.4
    tls,    // TLS 协议 (DoT)，如 tls://dns.example.com，默认端口 853
    https   // HTTPS 协议 (DoH)，如 https://dns.example.com/dns-query，默认端口 443
};
```

### dns_remote — 上游服务器配置

```cpp
struct dns_remote {
    memory::string address;                   // 原始地址字符串
    dns_protocol protocol{dns_protocol::udp}; // 通信协议
    memory::string hostname;                  // TLS SNI / HTTP Host
    std::uint16_t port{53};                   // 服务端口
    std::uint32_t timeout_ms{5000};           // 超时时间（毫秒）
    memory::string http_path;                 // DoH 路径，默认 /dns-query
    bool no_check_certificate{false};         // 跳过 TLS 证书验证
};
```

**地址解析规则**:
| 格式 | 协议 | 默认端口 |
|------|------|----------|
| `1.2.3.4` | UDP | 53 |
| `udp://1.2.3.4` | UDP | 53 |
| `tcp://1.2.3.4` | TCP | 53 |
| `tls://dns.example.com` | TLS (DoT) | 853 |
| `https://dns.example.com/dns-query` | HTTPS (DoH) | 443 |

### resolve_mode — 解析查询模式

```cpp
enum class resolve_mode : std::uint8_t {
    fastest,  // 并发查询所有上游，选择 RTT 最低的成功响应
    first,    // 并发查询所有上游，返回第一个成功响应
    fallback  // 按顺序尝试上游，前一个失败后尝试下一个
};
```

### address_rule — 地址映射规则

```cpp
struct address_rule {
    memory::string domain;                      // 匹配域名，支持通配符 *.xxx.com
    memory::vector<net::ip::address> addresses; // 映射的地址列表
    bool negative{false};                       // 否定应答（NXDOMAIN）
};
```

**用途**:
- `negative=false` — 静态地址映射，返回预定义 IP
- `negative=true` — 广告拦截，返回 NXDOMAIN

### cname_rule — CNAME 重定向规则

```cpp
struct cname_rule {
    memory::string domain;  // 源域名
    memory::string target;  // CNAME 目标域名
};
```

将特定域名的 A/AAAA 查询重定向到另一个域名。

### config — 主配置结构体

```cpp
struct config {
    // 上游服务器
    memory::vector<dns_remote> servers;       // 上游服务器列表
    resolve_mode mode{resolve_mode::fastest}; // 解析查询模式
    std::uint32_t timeout_ms{5000};           // 全局超时（毫秒）

    // 缓存配置
    bool cache_enabled{true};               // 是否启用 DNS 缓存
    std::size_t cache_size{10000};          // 缓存最大条目数
    std::chrono::seconds cache_ttl{120};    // 缓存默认 TTL
    bool serve_stale{true};                 // 过期后是否仍提供缓存数据
    std::chrono::seconds negative_ttl{300}; // 负缓存 TTL

    // TTL 钳制
    std::uint32_t ttl_min{60};    // 最小 TTL（秒）
    std::uint32_t ttl_max{86400}; // 最大 TTL（秒）

    // 域名规则
    memory::vector<address_rule> address_rules; // 地址映射规则
    memory::vector<cname_rule> cname_rules;     // CNAME 重定向规则

    // IPv6 控制
    bool disable_ipv6{false}; // 禁用 IPv6，跳过 AAAA 查询

    // IP 过滤
    memory::vector<net::ip::network_v4> blacklist_v4; // IPv4 黑名单
    memory::vector<net::ip::network_v6> blacklist_v6; // IPv6 黑名单
};
```

## Glaze JSON 序列化

文件末尾定义了所有配置类型的 Glaze 元数据，支持 JSON 序列化/反序列化：

```cpp
// dns_protocol 映射
template <>
struct glz::meta<psm::resolve::dns::dns_protocol> {
    static constexpr auto value = glz::enumerate(
        "udp", psm::resolve::dns::dns_protocol::udp,
        "tcp", psm::resolve::dns::dns_protocol::tcp,
        "tls", psm::resolve::dns::dns_protocol::tls,
        "https", psm::resolve::dns::dns_protocol::https);
};

// resolve_mode 映射
template <>
struct glz::meta<psm::resolve::dns::resolve_mode> {
    static constexpr auto value = glz::enumerate(
        "fastest", psm::resolve::dns::resolve_mode::fastest,
        "first", psm::resolve::dns::resolve_mode::first,
        "fallback", psm::resolve::dns::resolve_mode::fallback);
};

// config 字段映射
template <>
struct glz::meta<psm::resolve::dns::config> {
    using T = psm::resolve::dns::config;
    static constexpr auto value = glz::object(
        "servers", &T::servers,
        "mode", &T::mode,
        "timeout_ms", &T::timeout_ms,
        "cache_enabled", &T::cache_enabled,
        "cache_size", &T::cache_size,
        "cache_ttl", &T::cache_ttl,
        "serve_stale", &T::serve_stale,
        "negative_ttl", &T::negative_ttl,
        "ttl_min", &T::ttl_min,
        "ttl_max", &T::ttl_max,
        "address_rules", &T::address_rules,
        "cname_rules", &T::cname_rules,
        "disable_ipv6", &T::disable_ipv6);
};
```

## 配置项详解

### 上游服务器 (servers)

每个 `dns_remote` 可独立配置：
- 协议类型
- 地址与端口
- 超时时间
- TLS 选项（hostname, no_check_certificate）
- DoH 选项（http_path）

### 缓存参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `cache_enabled` | `true` | 是否启用缓存 |
| `cache_size` | `10000` | 最大缓存条目数 |
| `cache_ttl` | `120s` | 默认缓存 TTL |
| `serve_stale` | `true` | 过期数据仍可返回 |
| `negative_ttl` | `300s` | 失败域名缓存时间 |

**serve-stale 模式**: 过期数据在后台刷新时可继续使用，避免解析中断。

### TTL 钳制

```
min_ttl = 60s   → 过短 TTL 放大至 60s
max_ttl = 86400 → 过长 TTL 缩减至 1 天
```

防止 TTL=0 导致无缓存，或 TTL 过长导致数据陈旧。

### IP 黑名单

使用 `net::ip::network_v4/v6` 表示 CIDR 网段：

```cpp
blacklist_v4 = {
    net::ip::make_network_v4("10.0.0.0/8"),
    net::ip::make_network_v4("192.168.0.0/16")
};
```

解析结果中的 IP 若命中黑名单网段则被过滤。

## JSON 配置示例

```json
{
  "servers": [
    {"address": "8.8.8.8", "protocol": "udp"},
    {"address": "dns.google", "protocol": "tls", "hostname": "dns.google"},
    {"address": "https://dns.google/dns-query", "protocol": "https"}
  ],
  "mode": "fastest",
  "timeout_ms": 5000,
  "cache_enabled": true,
  "cache_size": 10000,
  "cache_ttl": 120,
  "serve_stale": true,
  "negative_ttl": 300,
  "ttl_min": 60,
  "ttl_max": 86400,
  "disable_ipv6": false,
  "address_rules": [
    {"domain": "*.ad.com", "negative": true}
  ],
  "cname_rules": []
}
```

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/resolve/dns/upstream|upstream]] — DNS 查询客户端
- [[core/resolve/dns/detail/rules|rules_engine]] — 域名规则引擎
- [[core/resolve/dns/detail/cache|cache]] — DNS 结果缓存

---

## 各配置字段详解

### 上游服务器 (servers)

每个 `dns_remote` 实例的完整配置字段：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `address` | string | (必填) | 服务器地址（IP 或域名） |
| `protocol` | dns_protocol | `udp` | 传输协议类型 |
| `hostname` | string | 从 address 提取 | TLS SNI / HTTPS Host 头 |
| `port` | uint16 | 按协议默认 | 服务端口 |
| `timeout_ms` | uint32 | 5000 | 单次查询超时时间（毫秒） |
| `http_path` | string | `/dns-query` | DoH 路径 |
| `no_check_certificate` | bool | `false` | 跳过 TLS 证书验证 |

**协议默认端口映射**:

```cpp
auto default_port(dns_protocol proto) -> uint16_t {
    switch (proto) {
        case dns_protocol::udp:     return 53;
        case dns_protocol::tcp:     return 53;
        case dns_protocol::tls:     return 853;
        case dns_protocol::https:   return 443;
    }
}
```

### 解析模式 (mode)

```cpp
enum class resolve_mode : std::uint8_t {
    fastest,   // 并发全部，选择 RTT 最低的
    first,     // 并发全部，首个成功即返回
    fallback   // 逐一尝试，前一个失败后尝试下一个
};
```

| 模式 | 查询策略 | 适用场景 | 延迟特征 |
|------|----------|----------|----------|
| `fastest` | 全部上游并发 | 追求最低延迟 | 等于最快上游的 RTT |
| `first` | 全部上游并发 | 追求可靠性 | 等于首个成功上游的 RTT |
| `fallback` | 逐一串行尝试 | 上游有优先级 | 等于失败上游数 × RTT + 成功上游 RTT |

### 缓存参数

| 参数 | 默认值 | 推荐范围 | 说明 |
|------|--------|----------|------|
| `cache_enabled` | `true` | - | 全局缓存开关 |
| `cache_size` | 10000 | 1000-100000 | 最大缓存条目数 |
| `cache_ttl` | 120s | 30s-600s | 默认 TTL（当 DNS 响应无 TTL 时使用） |
| `serve_stale` | `true` | - | 是否提供过期数据 |
| `negative_ttl` | 300s | 60s-600s | 负缓存 TTL（失败域名缓存时间） |

### TTL 钳制

```
实际 TTL = clamp(响应中的 TTL, ttl_min, ttl_max)

默认:
  ttl_min = 60s    → 防止 TTL=0 导致无缓存
  ttl_max = 86400s → 防止 TTL 过长导致数据陈旧

建议:
  低延迟场景: ttl_min=30, ttl_max=3600
  标准场景:   ttl_min=60, ttl_max=86400
  高可用场景: ttl_min=120, ttl_max=43200
```

### IPv6 控制

| 值 | 行为 |
|----|------|
| `false` (默认) | 正常查询 A 和 AAAA 记录 |
| `true` | 跳过 AAAA 查询，仅返回 A 记录 |

**使用场景**: IPv6 网络不可用或不稳定时禁用。

### IP 黑名单

```cpp
// 黑名单配置示例
blacklist_v4 = {
    "10.0.0.0/8",        // 内网 A 类
    "172.16.0.0/12",     // 内网 B 类
    "192.168.0.0/16",    // 内网 C 类
    "127.0.0.0/8",       // 回环地址
    "0.0.0.0/32"         // 无效地址
};
blacklist_v6 = {
    "::1/128",           // 回环地址
    "fe80::/10"          // 链路本地地址
};
```

**过滤逻辑**: 解析结果中的 IP 若命中黑名单网段则被移除，不返回给调用方。

## 配置字段默认值汇总

```cpp
config default_config() {
    return {
        .servers = {},
        .mode = resolve_mode::fastest,
        .timeout_ms = 5000,

        .cache_enabled = true,
        .cache_size = 10000,
        .cache_ttl = std::chrono::seconds(120),
        .serve_stale = true,
        .negative_ttl = std::chrono::seconds(300),

        .ttl_min = 60,
        .ttl_max = 86400,

        .address_rules = {},
        .cname_rules = {},

        .disable_ipv6 = false,

        .blacklist_v4 = {},
        .blacklist_v6 = {}
    };
}
```

## 性能影响分析

### servers 数量

| 上游数 | fastest 模式 | fallback 模式 |
|--------|-------------|---------------|
| 1 | 单次查询，延迟 = RTT | 同 fastest |
| 2 | 2 次并发，延迟 = min(RTT1, RTT2) | 最多 2 次串行 |
| 3+ | N 次并发，延迟 = min(RTTi) | 最多 N 次串行 |

**建议**: fastest 模式 2-3 个上游为最优，过多上游增加并发开销但收益递减。

### cache_size

```
cache_size = 10000  →  约 500KB - 2MB 内存（取决于平均 IP 数）
cache_size = 100000 →  约 5MB - 20MB 内存
```

缓存命中率随 size 增大而提高，但边际收益递减。

### serve_stale 的影响

```
无 serve_stale:  过期 → 重新查询 → 查询期间阻塞
有 serve_stale:  过期 → 立即返回旧数据 + 后台刷新 → 无阻塞
```

在网络不稳定的场景下，`serve_stale=true` 显著提升可用性。