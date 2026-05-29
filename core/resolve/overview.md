---
title: "resolve — 解析阶段概述"
layer: core
source: "include/prism/resolve/"
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

> 源码目录: `include/prism/resolve/`
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
## 设计决策（WHY）

### 为什么采用七阶段解析管道

DNS 解析需要经过规则匹配、缓存查找、请求合并、上游查询、IP 过滤、缓存写入等多个步骤。将解析过程拆分为管道阶段，每个阶段职责单一且可独立测试。阶段顺序经过精心设计：规则匹配在最前（零网络 IO），缓存查找次之（零上游负载），请求合并确保同目标不重复查询，IP 过滤在缓存写入前执行。

### 为什么提供三种解析策略

不同部署环境对 DNS 查询有不同需求：fastest 适合多上游竞速场景，返回 RTT 最低的响应（默认）；first 适合追求可靠性，任一上游响应即可；fallback 适合单上游或高容错场景，逐一尝试减少无效并发。三种策略覆盖了主流 DNS 调度需求，通过 config 中的 resolve_mode 切换。

### 为什么负缓存 TTL 设为 300 秒

DNS 解析失败时缓存错误结果可避免反复查询不可达域名。300 秒（5 分钟）是权衡值：过短则频繁重试浪费资源，过长则 DNS 恢复后域名仍不可达。该值硬编码在 config 默认值中，可通过配置覆盖。但需注意负缓存与 IP 黑名单的交互：被过滤的 IP 也触发负缓存，可能导致域名误伤。

### 为什么使用 coalescer 合并并发请求

多连接同时查询同一域名时，不合并意味着每个连接独立发起上游查询，浪费网络资源和上游配额。coalescer 使用透明哈希表以域名为 key，首个请求创建条目并发起查询，后续请求挂起等待同一结果。查询完成后所有等待者同时获得响应。

### 为什么 resolver 使用抽象接口 + 工厂函数

resolver 定义为纯虚接口，上层模块（router）仅依赖抽象。具体实现通过 make_resolver() 工厂函数创建，内部组合 upstream、cache、rules_engine、coalescer 等组件。这使得测试可用 mock 替换，未来可切换实现（如集成系统 DNS）而不影响 router。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 单线程使用 | resolver 在单个 io_context 线程中使用 | 数据竞争、缓存不一致 | upstream.hpp 注释 |
| PMR 内存 | 所有容器使用 memory::* PMR 类型 | 堆分配开销破坏延迟 | 项目规范 |
| fault::code 错误 | 热路径不抛异常 | 异常栈展开影响协程 | 双轨错误处理 |
| TTL 钳制 | 响应 TTL 限制在 ttl_min(60s) 到 ttl_max(86400s) | 过短 TTL 频繁重查，过长 TTL 数据过时 | config 定义 |
| 负缓存 300s | 解析失败结果缓存 300 秒 | 临时故障被放大 | config 默认值 |
| 缓存容量 | 最大 10000 条目 | 超出后 LRU 淘汰 | config.cache_size |
| 上游列表非空 | servers 为空时 resolve() 直接返回失败 | 所有域名不可达 | upstream.hpp 注释 |

## 故障场景

### 1. DNS 负缓存放大

DNS 解析失败写入负缓存（TTL 300s）。期间该域名所有查询直接返回失败，即使 DNS 已恢复。与 IP 黑名单交互：所有解析 IP 被过滤也写入负缓存（fault::code::blocked），导致 300s 内域名不可达。排障需检查 DNS 缓存状态，评估重启清除或减小 negative_ttl。

### 2. 所有上游 DNS 服务器不可达

当所有配置的上游服务器均无响应时，resolve() 返回错误。coalescer 确保并发查询只产生一次上游请求。若 serve_stale 启用（默认 true），可返回过期的缓存数据作为兜底。若缓存也为空，则所有依赖该域名的连接均失败。

### 3. 缓存容量耗尽

DNS 缓存达到 10000 条目上限后，新条目需要淘汰旧条目。高频查询的热门域名可能被冷门查询挤出缓存，导致缓存命中率下降。可通过 cache_size 调整容量，或缩短 cache_ttl 加速淘汰。

### 4. IP 黑名单过度过滤

blacklist_v4/blacklist_v6 配置过大网段时，解析出的 IP 全部被过滤。过滤结果写入负缓存，域名在 300s 内不可达。排障需检查黑名单范围是否过于宽泛。

### 5. 域名规则 CNAME 循环

CNAME 规则形成循环（A -> B -> A）时，递归解析无终止条件。实现中需限制 CNAME 跟随深度或检测循环，否则协程无限递归。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| router 聚合 | resolve -> connect | router 持有 dns::resolver + connection_pool，对外提供四种路由方法 |
| dns::resolver 接口 | resolve <- connect::router | router 通过抽象接口调用 resolve_tcp/resolve_udp，不依赖实现 |
| make_resolver 工厂 | resolve 内部 | 创建具体 resolver 实现，封装 upstream/cache/rules/coalescer 组合 |
| config 驱动 | 配置 -> resolve | config 结构体控制所有 DNS 行为，从配置文件加载 |
| fault::code 错误 | resolve -> 上层 | 解析结果通过 fault::code 传递，router 转换为连接错误 |
| message 格式 | resolve 内部共享 | DNS wire format 的 message/qtype/record 类型在 detail 中定义，upstream 和 cache 共用 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| resolver 接口变更 | connect::router | router 的 async_forward/async_datagram 依赖此接口 |
| config 结构字段增删 | 配置加载逻辑、所有 DNS 子组件 | 配置驱动的组件均需适配 |
| router 四种路由方法签名 | protocol 各 handler | handle() 内部通过 ctx 调用 router 方法 |

### 对内影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| message 报文格式变更 | upstream、cache、format | wire format 是 DNS 模块内部通用结构 |
| cache 淘汰策略变更 | 缓存命中率、负缓存行为 | 影响 DNS 解析性能和可靠性 |
| coalescer 合并逻辑变更 | 并发查询行为 | 影响上游负载和响应延迟 |
| rules 匹配算法变更 | 域名规则命中率 | 基数树实现变更影响规则匹配性能 |
| upstream 传输协议变更 | query_udp/tcp/tls/https | 新增或修改传输协议需回归测试所有模式 |
