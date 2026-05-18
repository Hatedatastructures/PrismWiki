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