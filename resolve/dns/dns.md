---
title: "dns.hpp — DNS 解析器抽象接口"
source: "include/prism/resolve/dns/dns.hpp"
module: "resolve"
type: api
tags: [resolve, dns, DNS, 解析器, 接口]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/router
  - resolve/dns/upstream
  - resolve/dns/config
  - resolve/dns/detail/cache
  - resolve/dns/detail/rules
  - resolve/dns/detail/coalescer
---

# dns.hpp

> 源码: `include/prism/resolve/dns/dns.hpp` + `src/prism/resolve/dns/resolver.cpp`
> 模块: [[resolve|Resolve]] / dns

## 概述

定义 `resolver` 纯虚接口，为上层模块提供统一的 DNS 解析抽象。所有 DNS 解析功能均通过此接口访问，内部实现细节（[[resolve/dns/detail/cache|缓存]]、[[resolve/dns/detail/rules|规则]]、[[resolve/dns/detail/format|报文编码]]等）隐藏在 `detail/` 子目录中。该文件同时作为 dns 模块的聚合头文件。

具体实现类 `resolver_impl`（位于 `resolver.cpp`）内部持有 [[resolve/dns/upstream|upstream]]、[[resolve/dns/detail/cache|cache]]、[[resolve/dns/detail/rules|rules_engine]]、[[resolve/dns/detail/coalescer|coalescer]] 等组件，构成完整的七阶段 DNS 查询管道。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[resolve/dns/config|config]] | DNS 配置 |
| 依赖 | [[resolve/dns/upstream|upstream]] | 上游查询客户端 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/router|router]] | 路由器持有 DNS 解析器 |

## 命名空间

`psm::resolve::dns`

---

## 类: resolver（抽象接口）

### 概述

DNS 解析器抽象接口。上层模块通过此接口进行 DNS 解析，不依赖具体实现。支持域名到 IP 地址的异步解析，以及 TCP/UDP 端点解析。

### 类层次

纯虚基类，具体实现由 `resolver_impl`（内部类）提供，通过 [[#函数-make_resolver|make_resolver()]] 工厂函数创建。

---

### 函数: resolve()

- **功能说明**: 异步解析域名到 IP 地址列表。同时并发查询 A 和 AAAA 记录（若禁用 IPv6 则仅查询 A），合并结果后返回。内部经过七阶段管道：域名规范化 → 规则匹配 → 缓存查找 → 请求合并 → 上游查询 → IP 黑名单过滤 → TTL 钳制+缓存存储。
- **签名**:
  ```cpp
  [[nodiscard]] virtual auto resolve(std::string_view host)
      -> net::awaitable<std::pair<fault::code, memory::vector<net::ip::address>>> = 0;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 主机名（自动规范化为小写并去末尾点号） |

- **返回值**: `net::awaitable<std::pair<fault::code, memory::vector<net::ip::address>>>` — 错误码和 IP 地址列表的配对
- **调用（向下）**: `query_pipeline(host, qtype::a)` + `query_pipeline(host, qtype::aaaa)`（并发）
- **被调用（向上）**: 当前无直接外部调用方（`resolve_tcp` / `resolve_udp` 内部使用 `query_pipeline`）
- **知识域**: [[resolve/dns/dns|DNS 查询管道]]

---

### 函数: resolve_tcp()

- **功能说明**: 异步解析到 TCP 端点列表。先通过 `query_pipeline` 解析域名获取 IP 地址（A+AAAA 并发），再与端口组合为 TCP 端点列表。端口号通过 [[resolve/dns/detail/utility|parse_port()]] 解析。
- **签名**:
  ```cpp
  [[nodiscard]] virtual auto resolve_tcp(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, memory::vector<tcp::endpoint>>> = 0;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 主机名 |
  | `port` | `std::string_view` | 服务端口字符串 |

- **返回值**: `net::awaitable<std::pair<fault::code, memory::vector<tcp::endpoint>>>` — 错误码和 TCP 端点列表的配对
- **调用（向下）**: `query_pipeline(host, qtype::a)` + `query_pipeline(host, qtype::aaaa)` → 组合端点
- **被调用（向上）**: [[resolve/router|router::async_forward()]] 在非字面量场景下调用
- **知识域**: [[resolve/dns/dns|DNS 查询管道]]

---

### 函数: resolve_udp()

- **功能说明**: 异步解析到 UDP 端点。优先查询 A 记录，若无结果且未禁用 IPv6 则回退查询 AAAA 记录。返回单个端点（取首个 IP）。
- **签名**:
  ```cpp
  [[nodiscard]] virtual auto resolve_udp(std::string_view host, std::string_view port)
      -> net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>> = 0;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 主机名 |
  | `port` | `std::string_view` | 服务端口字符串 |

- **返回值**: `net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>` — 错误码和 UDP 端点的配对
- **调用（向下）**: `query_pipeline(host, qtype::a)` → 回退 `query_pipeline(host, qtype::aaaa)`
- **被调用（向上）**: [[resolve/router|router::async_datagram()]]、[[resolve/router|router::resolve_datagram_target()]] 调用
- **知识域**: [[resolve/dns/dns|DNS 查询管道]]

---

### 函数: ipv6_disabled()

- **功能说明**: 查询是否禁用了 IPv6。返回配置中的 `disable_ipv6` 标志。
- **签名**:
  ```cpp
  [[nodiscard]] virtual auto ipv6_disabled() const noexcept -> bool = 0;
  ```
- **参数**: 无
- **返回值**: `bool` — 禁用 IPv6 返回 `true`，否则返回 `false`
- **调用（向下）**: 无（读取配置标志）
- **被调用（向上）**: [[resolve/router|router::ipv6_disabled()]] 委托调用
- **知识域**: [[resolve/dns/config|DNS 配置]]

---

## 工厂函数: make_resolver()

- **功能说明**: 创建 DNS 解析器实例。内部构造 `resolver_impl`，该实现持有 [[resolve/dns/upstream|upstream]]、[[resolve/dns/detail/cache|cache]]、[[resolve/dns/detail/rules|rules_engine]]、[[resolve/dns/detail/coalescer|coalescer]] 等组件，并启动定时清理协程（每 30 秒清理过期缓存条目）。
- **签名**:
  ```cpp
  [[nodiscard]] auto make_resolver(net::io_context &ioc, config cfg,
                                   memory::resource_pointer mr = memory::current_resource())
      -> std::unique_ptr<resolver>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `ioc` | `net::io_context &` | IO 上下文引用 |
  | `cfg` | `config` | DNS 配置 |
  | `mr` | `memory::resource_pointer` | 内存资源指针 |

- **返回值**: `std::unique_ptr<resolver>` — 唯一所有权指向 resolver 实例
- **调用（向下）**: 构造 `resolver_impl` → 初始化 upstream/cache/rules/coalescer → 启动清理协程
- **被调用（向上）**: [[resolve/router|router::router()]] 构造函数中调用
- **知识域**: [[resolve/dns/dns|DNS 查询管道]]、[[resolve/dns/config|DNS 配置]]

---

## 内部实现: resolver_impl 查询管道

`resolver_impl::query_pipeline()` 是 DNS 解析的核心私有方法，实现七阶段查询管道：

```
1. 域名规范化     normalize() → 小写 + 去末尾点号
2. 规则匹配       rules_.match() → 静态地址/否定规则/CNAME
3. 缓存查找       cache_.get() → 正向缓存/负缓存
4. 请求合并       coalescer_.find_or_create() → 合并并发查询
5. 上游查询       upstream_.resolve() → UDP/TCP/DoT/DoH
6. IP 过滤        is_blacklisted() → 黑名单匹配
7. TTL 钳制       std::clamp(ttl, min, max) → cache_.put()
```

---

## 调用链总览

```
[[resolve/router|router]] → make_resolver() → resolver_impl
resolver_impl::resolve()      → query_pipeline(a) + query_pipeline(aaaa) [并发]
resolver_impl::resolve_tcp()  → query_pipeline(a) + query_pipeline(aaaa) → 组合端点
resolver_impl::resolve_udp()  → query_pipeline(a) → 回退 query_pipeline(aaaa)

query_pipeline():
  rules_.match()  → 命中 → 返回静态地址/拦截
  cache_.get()    → 命中 → 返回缓存结果
  coalescer_      → 已有请求 → 等待 + 读缓存
  upstream_.resolve() → 上游查询
  is_blacklisted()    → IP 过滤
  cache_.put()        → TTL 钳制 + 缓存存储
```

---

## 知识域

- [[ref/protocol/dns-over-udp|DNS 解析]]
- [[resolve/dns/dns|DNS 查询管道]]
- [[resolve/dns/config|DNS 配置]]
