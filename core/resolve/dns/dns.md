---
title: "resolver — DNS 解析器抽象接口"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/dns.hpp"
module: "resolve/dns"
type: interface
tags: [dns, resolver, async, interface]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/upstream
  - core/resolve/dns/config
  - core/resolve/dns/detail/cache
  - core/resolve/dns/detail/coalescer
  - core/resolve/dns/detail/rules
---

# resolver — DNS 解析器抽象接口

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/dns.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]]
> 类型: 抽象接口 + 聚合头文件

## 组件定位

该文件是 DNS 解析模块的顶层接口，定义了 `resolver` 纯虚接口，为上层模块提供统一的 DNS 解析抽象。同时作为 `dns` 模块的聚合头文件，引入子目录中所有公开头文件。

**设计意图**: 所有 DNS 解析功能均通过此接口访问，内部实现细节（缓存、规则、报文编码等）隐藏在 `detail/` 子目录中，外部模块不应直接 include 内部头文件。

## resolver 抽象接口

### 类定义

```cpp
class resolver {
public:
    virtual ~resolver() = default;

    // 异步解析域名到 IP 地址列表
    [[nodiscard]] virtual auto resolve(std::string_view host)
        -> net::awaitable<std::pair<fault::code, memory::vector<net::ip::address>>> = 0;

    // 异步解析到 TCP 端点列表
    [[nodiscard]] virtual auto resolve_tcp(std::string_view host, std::string_view port)
        -> net::awaitable<std::pair<fault::code, memory::vector<tcp::endpoint>>> = 0;

    // 异步解析到 UDP 端点
    [[nodiscard]] virtual auto resolve_udp(std::string_view host, std::string_view port)
        -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>> = 0;

    // 查询是否禁用了 IPv6
    [[nodiscard]] virtual auto ipv6_disabled() const noexcept -> bool = 0;
};
```

### 方法说明

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `resolve(host)` | `vector<ip::address>` | 同时查询 A 和 AAAA，返回合并的 IP 列表 |
| `resolve_tcp(host, port)` | `vector<tcp::endpoint>` | 解析域名后与端口组合为 TCP 端点 |
| `resolve_udp(host, port)` | `udp::endpoint` | 优先 A 记录，回退 AAAA，组合为 UDP 端点 |
| `ipv6_disabled()` | `bool` | 返回配置中的 `disable_ipv6` 标志 |

### 接口特性

1. **纯协程设计**: 所有异步操作返回 `net::awaitable<T>`
2. **错误码返回**: 使用 `fault::code` 而非异常
3. **非线程安全**: 应在单个 `io_context` 线程中使用
4. **域名规范化**: host 自动转小写并去末尾点号

## 工厂函数

```cpp
[[nodiscard]] auto make_resolver(net::io_context &ioc, config cfg,
                                 memory::resource_pointer mr = memory::current_resource())
    -> std::unique_ptr<resolver>;
```

创建 `resolver` 的具体实现，内部持有：

- `upstream` — DNS 查询客户端
- `cache` — DNS 结果缓存
- `rules_engine` — 域名规则引擎
- `coalescer` — 请求合并器

## 解析管道流程

```
resolver::resolve(host)
  │
  ├─→ 域名规范化（小写、去末尾点号）
  │
  ├─→ rules_engine::match(host)
  │     ├─→ 地址规则命中 → 返回静态 IP
  │     ├─→ 否定规则命中 → 返回 NXDOMAIN
  │     └─→ CNAME 规则命中 → 重定向到目标域名
  │
  ├─→ cache::get(host, A/AAAA)
  │     ├─→ 正向缓存命中 → 返回 IPs
  │     └─→ 负缓存命中 → 返回空列表
  │
  ├─→ coalescer::find_or_create(host, A/AAAA)
  │     ├─→ 已有请求进行中 → 等待结果复用
  │     └─→ 新请求 → 发起上游查询
  │
  ├─→ upstream::resolve(domain, A)
  ├─→ upstream::resolve(domain, AAAA)
  │     ├─→ query_udp/tcp/tls/https
  │     └→ message::unpack → extract_ips
  │
  ├─→ IP 过滤（黑名单网段）
  │
  ├─→ TTL 钳制（ttl_min ~ ttl_max）
  │
  └→ cache::put(host, qtype, ips, ttl)
```

## resolve_tcp 与 resolve_udp 区别

| 方法 | 查询策略 | 返回值 | 用途 |
|------|----------|--------|------|
| `resolve_tcp` | A + AAAA 并发 | 全部端点列表 | TCP 连接需要所有候选地址 |
| `resolve_udp` | A 优先，AAAA 回退 | 单个端点 | UDP 通常只需一个地址 |

**设计决策**: UDP 场景通常只需要一个目标地址，优先返回 IPv4 地址更兼容传统网络环境；若 A 记录不存在才回退到 AAAA。

## 模块聚合

该文件作为聚合头文件，引入以下公开头文件：

```cpp
#include <prism/resolve/dns/config.hpp>    // 配置结构体
#include <prism/resolve/dns/upstream.hpp>  // 上游客户端
```

**注意**: `detail/` 下的内部头文件不在此暴露：
- `cache.hpp`
- `coalescer.hpp`
- `rules.hpp`
- `format.hpp`
- `utility.hpp`
- `transparent.hpp`

## 调用链

### 上游调用

| 上游模块 | 调用入口 | 说明 |
|----------|----------|------|
| [[core/resolve/router|router]] | `resolve_tcp()` | 正向代理 TCP 连接 |
| [[core/resolve/router|router]] | `resolve_udp()` | 数据报地址解析 |

### 下游调用

| 下游模块 | 调用出口 | 说明 |
|----------|----------|------|
| [[core/resolve/dns/detail/rules|rules_engine]] | `match()` | 静态规则匹配 |
| [[core/resolve/dns/detail/cache|cache]] | `get()` / `put()` | 缓存操作 |
| [[core/resolve/dns/detail/coalescer|coalescer]] | `find_or_create()` | 请求合并 |
| [[core/resolve/dns/upstream|upstream]] | `resolve()` | 上游查询 |

## 参见

- [[core/resolve/dns/upstream|upstream]] — DNS 查询客户端
- [[core/resolve/dns/config|config]] — DNS 配置结构体
- [[core/resolve/dns/detail/cache|cache]] — DNS 结果缓存
- [[core/resolve/dns/detail/coalescer|coalescer]] — 请求合并器
- [[core/resolve/dns/detail/rules|rules_engine]] — 域名规则引擎
- [[core/resolve/dns/detail/format|message]] — DNS 报文编解码

---

## 完整查询链路图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        resolver::resolve(host)                       │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ Stage 1: 域名规范化                                              │ │
│  │   - host 转小写                                                  │ │
│  │   - 去除末尾点号 (trailing dot)                                 │ │
│  │   - 验证字符合法性                                               │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │ normalized_host                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 2: IP 字面量检测                                           │ │
│  │   - 尝试解析为 IPv4/IPv6                                        │ │
│  │   - 成功 → 直接返回，跳过所有后续阶段                            │ │
│  │   - 失败 → 继续 Stage 3                                          │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 3: 规则引擎匹配                                            │ │
│  │   rules_engine::match(normalized_host)                           │ │
│  │   ├── 地址规则命中 → 返回静态 IPs (跳过缓存和上游查询)            │ │
│  │   ├── 否定规则命中 → 返回空列表 (跳过后续)                       │ │
│  │   ├── CNAME 规则命中 → 重定向，递归调用 resolve(target)          │ │
│  │   └── 无规则命中 → 继续 Stage 4                                  │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 4: 缓存查找                                                │ │
│  │   cache::get(host, A) + cache::get(host, AAAA)                  │ │
│  │   ├── 正向命中 → 返回 IPs                                        │ │
│  │   ├── 负缓存命中 → 返回空列表                                    │ │
│  │   └── 未命中 → 继续 Stage 5                                      │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 5: 请求合并                                                │ │
│  │   coalescer::find_or_create(key, executor)                       │ │
│  │   ├── 新请求 → 继续 Stage 6                                      │ │
│  │   └── 已有请求 → co_await 等待，复用结果                         │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 6: 上游查询                                                │ │
│  │   upstream::resolve(domain, A)  ← 并发                           │ │
│  │   upstream::resolve(domain, AAAA) ← 并发                         │ │
│  │   ├── 根据 resolve_mode 选择策略                                  │ │
│  │   │   ├── fastest: 全部并发，选 RTT 最低                         │ │
│  │   │   ├── first: 全部并发，首个成功即返回                        │ │
│  │   │   └── fallback: 逐一尝试                                     │ │
│  │   ├── 报文解包 message::unpack                                   │ │
│  │   ├── IP 提取 extract_ips                                       │ │
│  │   └── IP 黑名单过滤                                             │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│  ┌───────────────────────▼─────────────────────────────────────────┐ │
│  │ Stage 7: 缓存写入                                                │ │
│  │   TTL 钳制 (ttl_min ≤ TTL ≤ ttl_max)                            │ │
│  │   cache::put(host, qtype, ips, clamped_ttl)                     │ │
│  │   coalescer: timer.cancel() 通知等待者                           │ │
│  │   coalescer: cleanup + flush_cleanup                             │ │
│  └───────────────────────┬─────────────────────────────────────────┘ │
│                          │                                            │
│                          ▼                                            │
│                  返回 IPs 列表                                        │
└──────────────────────────────────────────────────────────────────────┘
```

## 七阶段查询管道详解

### Stage 1: 域名规范化

```cpp
// 规范化操作
auto normalize(std::string_view host) -> memory::string {
    memory::string result(host);
    // 1. 转小写
    std::transform(result.begin(), result.end(), result.begin(), ::tolower);
    // 2. 去末尾点号
    while (!result.empty() && result.back() == '.')
        result.pop_back();
    return result;
}
```

**为什么需要规范化**:
- DNS 域名不区分大小写，`EXAMPLE.COM` 和 `example.com` 等价
- 末尾点号表示根域，但实际查询时可省略
- 规范化后确保缓存键和规则匹配的一致性

### Stage 2: IP 字面量检测

```cpp
if (auto addr = net::ip::make_address(host, ec); !ec) {
    // 是 IP 地址字面量
    if (addr.is_v6() && cfg.disable_ipv6) {
        co_return {fault::code::host_unreachable, {}};
    }
    co_return {fault::code::success, {addr}};
}
```

**短路优化**: IP 字面量无需 DNS 查询，直接返回。

### Stage 3: 规则引擎匹配

详细算法见 [[core/resolve/dns/detail/rules|rules_engine]]。

```
规则匹配优先级:
  1. 精确地址规则 (domain_trie 精确匹配)
  2. 通配符地址规则 (domain_trie 通配符匹配)
  3. 否定规则 (广告屏蔽)
  4. CNAME 重定向

地址规则命中 → 立即返回，不查缓存、不查上游
CNAME 规则命中 → 递归调用 resolve(target_domain)
```

### Stage 4: 缓存查找

详细机制见 [[core/resolve/dns/detail/cache|cache]]。

```cpp
// 并发查询 A 和 AAAA 缓存
auto a_result = cache_.get(host, qtype::a);
auto aaaa_result = cache_.get(host, qtype::aaaa);

// 合并结果
if (a_result && aaaa_result) {
    co_return merge(a_result.value(), aaaa_result.value());
}
// 任一未命中 → 需要上游查询
```

**serve-stale 模式**: 过期数据仍可返回，但标记需后台刷新。

### Stage 5: 请求合并

详细机制见 [[core/resolve/dns/detail/coalescer|coalescer]]。

```
场景: 10 个协程同时请求 resolve("www.example.com")

无合并: 10 次独立上游查询
有合并: 1 次上游查询 + 9 个协程等待结果

合并键: "host:qtype" (如 "www.example.com:1")
```

### Stage 6: 上游查询

详细实现见 [[core/resolve/dns/upstream|upstream]]。

```cpp
// fastest 模式: 并发查询所有上游
std::vector<awaitable<query_result>> futures;
for (const auto& server : servers_) {
    futures.push_back(query_server(server, query_msg));
}

// 等待所有结果
auto results = co_await when_all(futures.begin(), futures.end());

// 选择 RTT 最低的成功响应
auto best = std::ranges::min_element(results,
    [](const auto& a, const auto& b) {
        if (a.error != success) return false;
        if (b.error != success) return true;
        return a.rtt_ms < b.rtt_ms;
    });
```

**IP 黑名单过滤**:
```cpp
auto filter_blacklisted(const vector<ip::address>& ips) {
    vector<ip::address> result;
    for (const auto& ip : ips) {
        if (ip.is_v4() && in_blacklist_v4(ip.to_v4())) continue;
        if (ip.is_v6() && in_blacklist_v6(ip.to_v6())) continue;
        result.push_back(ip);
    }
    return result;
}
```

### Stage 7: 缓存写入

```cpp
// TTL 钳制
auto clamped_ttl = std::clamp(ttl, cfg.ttl_min, cfg.ttl_max);

// 写入缓存
cache_.put(host, qtype::a, a_ips, clamped_ttl);
cache_.put(host, qtype::aaaa, aaaa_ips, clamped_ttl);

// 通知等待的协程
coalescer_.notify(key);  // timer.cancel()
coalescer_.flush_cleanup();
```

### 管道短路场景总结

| 阶段 | 短路条件 | 跳过阶段 |
|------|----------|----------|
| Stage 2 | IP 字面量 | 3-7 |
| Stage 3 | 地址规则命中 | 4-7 |
| Stage 3 | 否定规则命中 | 4-7 |
| Stage 3 | CNAME 重定向 | 4-6 (递归从 Stage 1 开始) |
| Stage 4 | 缓存命中 | 5-7 |
| Stage 5 | 请求已存在 | 6 (等待已有请求) |