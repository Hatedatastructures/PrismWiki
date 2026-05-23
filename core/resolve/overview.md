---
title: "resolve — 解析阶段概述"
layer: core
source: "I:/code/Prism/include/prism/resolve/"
module: "resolve"
type: overview
tags: [resolve, dns, router, cache, rules]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/router
  - core/resolve/dns/dns
  - core/resolve/dns/upstream
  - core/resolve/dns/config
---

# resolve — 解析阶段概述

> 源码目录: `I:/code/Prism/include/prism/resolve/`
> 模块: [[resolve|resolve]]
> 定位: 请求处理流程的地址解析阶段

## 模块职责

解析阶段（Resolve Layer）负责将域名转换为 IP 地址，并为上层提供路由决策。该阶段整合了 DNS 解析器、反向路由表和连接池，为分发层提供统一的路由接口。

核心职责：

| 职责 | 说明 | 关键组件 |
|------|------|----------|
| 路由分发 | 反向代理/直连/正向代理/数据报路由 | [[core/resolve/router|router]] |
| DNS 解析 | 异步域名解析，支持多种上游协议 | [[core/resolve/dns/dns|resolver]] |
| DNS 查询客户端 | UDP/TCP/DoT/DoH 四种传输协议 | [[core/resolve/dns/upstream|upstream]] |
| 配置管理 | 上游服务器、解析策略、规则配置 | [[core/resolve/dns/config|config]] |
| 结果缓存 | 正向缓存 + 负缓存，TTL 管理 | [[core/resolve/dns/detail/cache|cache]] |
| 请求合并 | 同目标并发请求合并为单次操作 | [[core/resolve/dns/detail/coalescer|coalescer]] |
| 域名规则 | 地址映射、广告屏蔽、CNAME 重定向 | [[core/resolve/dns/detail/rules|rules_engine]] |
| 报文编解码 | DNS wire format 构造与解析 | [[core/resolve/dns/detail/format|message]] |

## 源码结构

```
resolve/
├── router.hpp                  # 分发层路由器（顶层门面）
│
└── dns/                        # DNS 解析模块
    ├── dns.hpp                 # 解析器抽象接口 + 聚合头文件
    ├── upstream.hpp            # DNS 查询客户端（UDP/TCP/DoT/DoH）
    ├── config.hpp              # DNS 配置结构体
    │
    └── detail/                 # 内部实现细节
        ├── cache.hpp           # DNS 结果缓存
        ├── coalescer.hpp       # 请求合并器
        ├── rules.hpp           # 域名规则引擎（基数树）
        ├── format.hpp          # DNS 报文编解码
        ├── utility.hpp         # 工具函数（零分配解析）
        └── transparent.hpp     # 透明哈希/相等比较器
```

## 调用链

### 上游调用（入）

解析阶段接收以下上游模块的请求：

| 上游模块 | 调用入口 | 说明 |
|----------|----------|------|
| Agent 层 | `router::async_forward()` | 正向代理域名解析 |
| Agent 层 | `router::async_reverse()` | 反向代理路由查找 |
| Agent 层 | `router::async_datagram()` | UDP 数据报地址解析 |
| Pipeline 层 | `router::async_direct()` | 直连端点连接 |

### 下游调用（出）

解析阶段调用以下下游模块：

| 下游模块 | 调用出口 | 说明 |
|----------|----------|------|
| [[core/connect/pool/pool|connection_pool]] | `async_acquire()` | 建立 TCP 连接 |
| [[core/connect/dial/racer|address_racer]] | `race()` | Happy Eyeballs 竞速 |
| Boost.Asio | `udp::socket::open()` | 创建 UDP 套接字 |

### 内部调用链

```
router::async_forward(host, port)
  ├─→ IP 字面量 → pool::async_acquire(endpoint)
  │
  └─→ 域名 → dns::resolver::resolve_tcp(host, port)
      ├─→ rules_engine::match(host) → 静态规则匹配
      ├─→ cache::get(host, qtype) → 缓存查找
      ├─→ coalescer::find_or_create() → 请求合并
      └─→ upstream::resolve(domain, qtype)
          ├─→ query_udp() / query_tcp() / query_tls() / query_https()
          └─→ message::unpack() → 解析响应
      → cache::put() → 写入缓存
      → address_racer::race(endpoints) → 端点竞速连接
```

## 模块依赖

### 依赖关系图

```
resolve/
├── router
│   ├── → dns::resolver
│   ├── → connection_pool
│   └── → address_racer
│
└── dns/
    ├── dns (接口)
    │   └── → config, upstream
    │
    ├── upstream (客户端)
    │   └── → format (message, qtype)
    │   └── → config (dns_remote, resolve_mode)
    │
    ├── config (配置)
    │   └── → memory::container
    │
    └── detail/
        ├── cache → format, transparent
        ├── coalescer → transparent
        ├── rules → domain_trie, rule_result
        ├── format → message, qtype, record
        ├── utility → parse_port (header-only)
        └── transparent → transparent_hash, transparent_equal
```

## 关键设计

### router 四种路由模式

[[core/resolve/router|router]] 支持四种路由模式：

| 模式 | 方法 | 说明 |
|------|------|------|
| 反向代理 | `async_reverse()` | 查找反向路由表，获取预配置端点 |
| 直连 | `async_direct()` | 直接连接指定端点 |
| 正向代理 | `async_forward()` | DNS 解析域名后连接 |
| 数据报 | `async_datagram()` | DNS 解析后创建 UDP socket |

### DNS 解析管道

[[core/resolve/dns/dns|resolver]] 内部实现完整的 DNS 解析管道：

1. **规则匹配** — 静态地址规则/CNAME 重定向/广告屏蔽
2. **缓存查找** — 正向缓存命中/负缓存命中
3. **请求合并** — 同目标并发请求合并
4. **上游查询** — 按策略调度上游服务器
5. **IP 过滤** — 黑名单网段过滤
6. **缓存写入** — 写入正向或负缓存

### 三种解析策略

[[core/resolve/dns/config|resolve_mode]] 定义三种查询调度策略：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `fastest` | 并发查询所有上游，选择 RTT 最低的 | 默认，追求最快响应 |
| `first` | 并发查询所有上游，返回首个成功 | 追求可靠性 |
| `fallback` | 逐一尝试上游，失败后尝试下一个 | 单上游或高容错场景 |

## 故障模式

### DNS 负缓存 TTL 300s

DNS 解析失败结果被缓存 300 秒（硬编码）。临时 DNS 故障 → 写入负缓存 → 5 分钟内该域名所有查询直接失败，即使 DNS 已恢复。

与 blacklist 的交互：所有解析 IP 被 blacklist 过滤也写入负缓存（`fault::code::blocked`），导致 300s 内该域名不可达。

**排障**：检查 DNS 缓存状态，评估是否需要重启清除。考虑减小 `negative_ttl` 配置。

详见 [[dev/debugging/deep-dive/system-risks|系统级风险与资源耗尽分析]]

## 参见

- [[core/resolve/router|router]] — 分发层路由器
- [[core/resolve/dns/dns|resolver]] — DNS 解析器抽象接口
- [[core/resolve/dns/upstream|upstream]] — DNS 查询客户端
- [[core/resolve/dns/config|config]] — DNS 配置结构体
- [[core/resolve/dns/detail/cache|cache]] — DNS 结果缓存
- [[core/resolve/dns/detail/coalescer|coalescer]] — 请求合并器
- [[core/resolve/dns/detail/rules|rules_engine]] — 域名规则引擎
- [[core/resolve/dns/detail/format|message]] — DNS 报文编解码