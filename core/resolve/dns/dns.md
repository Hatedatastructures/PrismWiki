---
title: resolver — DNS 解析器抽象接口
layer: core
source: include/prism/resolve/dns/dns.hpp
module: resolve/dns
type: interface
tags: [dns, resolver, async, interface]
updated: 2026-05-27
---

# resolver — DNS 解析器抽象接口

DNS 解析模块的顶层接口，定义 `resolver` 纯虚基类。上层模块通过此接口进行域名解析，内部实现（缓存、规则、报文编码等）隐藏在 `detail/` 子目录中。

## 核心接口

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `resolve(host)` | `pair<fault::code, vector<ip::address>>` | 查询 A + AAAA，返回合并 IP 列表 |
| `resolve_tcp(host, port)` | `pair<fault::code, vector<tcp::endpoint>>` | 解析后与端口组合为 TCP 端点 |
| `resolve_udp(host, port)` | `pair<fault::code, udp::endpoint>` | A 优先，回退 AAAA，返回单个 UDP 端点 |
| `ipv6_disabled()` | `bool` | 配置中的 `disable_ipv6` 标志 |

### 工厂函数

`make_resolver(ioc, cfg, mr)` → `unique_ptr<resolver>`，内部持有 upstream、cache、rules_engine、coalescer。

## 设计决策

### 为什么 resolve_udp 只返回单个端点且优先 A 记录？

UDP 数据报只需一个目标地址。优先 A 记录是因为 IPv4 在传统网络中兼容性更好，许多防火墙/路由器不转发 IPv4 协议类型的 UDP。`resolve_tcp` 则返回所有候选端点（A + AAAA 并发查询），因为 Happy Eyeballs 需要全部地址参与竞速。

**后果**: UDP 路径不参与 IPv6 竞速，只在 A 记录不存在时才回退 AAAA。

### 为什么 make_resolver 返回 unique_ptr 而非 shared_ptr？

resolver 被单一线程独占使用（每个 worker 各持一个），不需要共享所有权。`unique_ptr` 明确表达独占语义，避免无意义的引用计数开销。

**后果**: resolver 不可跨 worker 共享。如需多线程访问，每个线程各创建一个实例。

### 为什么是七阶段管道而不是简单查询？

实际 DNS 解析需要处理：域名规范化、IP 字面量短路、静态规则（广告屏蔽/CNAME 重定向）、缓存（含 serve-stale）、请求合并（防惊群）、多上游策略、TTL 钳制。每个阶段都可以短路退出，避免不必要的上游查询。

### 七阶段管道

| 阶段 | 操作 | 短路条件 |
|------|------|----------|
| 1 | 域名规范化（小写、去末尾点号） | — |
| 2 | IP 字面量检测 | 命中 → 跳过 3-7 |
| 3 | rules_engine 匹配 | 地址/否定规则命中 → 跳过 4-7；CNAME → 递归从 1 开始 |
| 4 | cache 查找 (A + AAAA) | 正向/负缓存命中 → 跳过 5-7 |
| 5 | coalescer 请求合并 | 已有请求 → 等待复用，跳过 6 |
| 6 | upstream 查询 (fastest/first/fallback 模式) + IP 黑名单过滤 | — |
| 7 | TTL 钳制 → cache 写入 → coalescer 通知等待者 | — |

## 约束

### 单线程使用

**类型**: 线程安全

**规则**: resolver 实例仅在创建它的 io_context 线程中使用。cache、coalescer 等内部组件均非线程安全。

**违反后果**: 数据竞争，缓存损坏，请求合并器状态不一致。

**源码依据**: `dns.hpp:40` `@note`

### 域名必须先规范化再查缓存

**类型**: 调用顺序

**规则**: host 参数传入后自动规范化（小写 + 去末尾点号），缓存键使用规范化后的域名。

**违反后果**: 如果绕过 resolver 直接操作 cache，必须自行规范化，否则缓存未命中。

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/resolve/dns/config|config]] | DNS 配置（上游服务器、TTL 范围、黑名单等） |
| [[core/resolve/dns/upstream|upstream]] | DNS 查询客户端（UDP/TCP/TLS/HTTPS） |

### 内部组件（detail/，不对外暴露）

| 组件 | 用途 |
|------|------|
| [[core/resolve/dns/detail/cache|cache]] | DNS 结果缓存 + serve-stale |
| [[core/resolve/dns/detail/coalescer|coalescer]] | 请求合并（防惊群） |
| [[core/resolve/dns/detail/rules|rules_engine]] | 域名规则匹配（静态 IP、广告屏蔽、CNAME） |

### 被引用

| 模块 | 调用方式 |
|------|----------|
| [[core/resolve/router|router]] | `resolve_tcp()` / `resolve_udp()` |
