---
title: DNS 解析器实现
module: "resolve"
namespace: psm::resolve::dns
files:
  - include/prism/resolve/dns/dns.hpp
  - include/prism/resolve/dns/upstream.hpp
  - include/prism/resolve/dns/config.hpp
  - include/prism/resolve/dns/detail/cache.hpp
  - include/prism/resolve/dns/detail/coalescer.hpp
  - include/prism/resolve/dns/detail/rules.hpp
  - include/prism/resolve/dns/detail/format.hpp
see_also:
  - "[[resolve]]"
  - "[[memory/pool]]"
  - "[[channel/transport]]"
---

# DNS 解析器实现

DNS 模块为 [[resolve]] 提供完整的域名解析能力，内部包含规则匹配、缓存、
请求合并、上游查询等完整查询管道。所有异步接口返回 `net::awaitable<T>`，
不依赖系统 resolver，完全自行编解码 DNS 报文。

## 查询管道

```
  resolve(host)
      │
      ▼
  rules_engine.match()   ← 域名规则（地址映射/广告屏蔽/CNAME）
      │  命中 → 直接返回
      ▼
  cache.get()            ← 内存缓存（正向/负缓存/serve-stale）
      │  命中 → 直接返回
      ▼
  coalescer.find_or_create()  ← 请求合并（防雪崩）
      │  已有请求 → 等待复用
      ▼
  upstream.resolve()     ← 上游查询（UDP/TCP/DoT/DoH）
      │
      ▼
  cache.put() / put_negative()  ← 写入缓存
```

## resolver 接口

`resolver` 是纯虚接口，`resolve()` 同时查询 A/AAAA 记录返回合并 IP 列表，
`resolve_tcp()` 返回端点列表，`resolve_udp()` 优先 A 回退 AAAA 返回单端点。

工厂函数：`make_resolver(io_context&, config, resource_pointer)`

## 配置体系

### dns_protocol — 上游协议

| 枚举值 | 协议 | 默认端口 | 说明 |
|--------|------|----------|------|
| `udp` | UDP | 53 | 标准 DNS，最小开销 |
| `tcp` | TCP | 53 | 可靠传输，2 字节长度前缀 |
| `tls` | DoT | 853 | DNS over TLS |
| `https` | DoH | 443 | DNS over HTTPS，POST 方法 |

地址解析规则：无 scheme 默认 UDP，`tcp://`、`tls://`、`https://` 前缀选择协议。

### dns_remote — 上游服务器

包含 address、protocol、hostname（TLS SNI/HTTP Host）、port、timeout_ms、
http_path（DoH 路径，默认 `/dns-query`）、no_check_certificate 字段。

### resolve_mode — 解析策略

| 模式 | 行为 |
|------|------|
| `fastest` | 并发查询所有上游，返回 RTT 最低的成功响应 |
| `first` | 并发查询所有上游，返回首个成功响应 |
| `fallback` | 按顺序尝试，前一个失败后才尝试下一个 |

### config — 主配置

聚合所有配置项：servers（上游列表）、cache_enabled/size/ttl、serve_stale、
negative_ttl（默认 300 秒）、ttl_min/ttl_max（60-86400 秒）、
address_rules/cname_rules、blacklist_v4/v6。所有容器使用 PMR 分配器，
支持 glaze JSON 序列化。

## 缓存（detail::cache）

缓存键格式 `"domain:qtype_number"`（如 `"www.example.com:1"`），
通过 `transparent_hash` 的 `string_view` 重载实现零分配查找。

**缓存策略：**

- **TTL 过期：** 过期后若 `serve_stale=true` 返回旧数据（调用方触发后台刷新）
- **FIFO 淘汰：** 超过 `max_entries` 时按插入时间淘汰最旧条目
- **负缓存：** 解析失败的域名以 `negative_ttl` 缓存，防止重复查询上游
- **LRU 链表：** 双向链表记录访问顺序

**查找返回值语义：** `nullopt` = 未命中，空 vector = 负缓存命中，
非空 vector = 正向缓存命中。

## 请求合并（detail::coalescer）

当多个协程同时请求同一域名时，coalescer 确保仅执行一次上游查询，
其他协程等待结果复用，防止雪崩效应。

**核心机制：**

- **flight 结构体：** 跟踪进行中的请求，包含等待定时器、等待者计数
- **find_or_create：** 已有请求则返回现有 flight 并递增 `waiters`
- **定时器挂起：** 等待协程通过 `timer.async_wait()` 挂起，请求完成后 `cancel()` 唤醒
- **延迟清理：** `pending_cleanup` 标记避免迭代器失效

键格式为 `"host:port"`，由 `make_key(host, port)` 生成。

## 规则引擎（detail::rules_engine）

### 反转域名基数树（domain_trie）

域名标签按反转顺序存储：`www.example.com` → `["com", "example", "www"]`，
后缀匹配等价于前缀遍历。通配符 `*.example.com` 在 `"example"` 节点标记
`wildcard=true`，要求至少匹配一级子域。

### 规则类型

| 规则 | 方法 | 效果 |
|------|------|------|
| 地址映射 | `add_address_rule(domain, ips)` | 命中返回静态 IP |
| 否定规则 | `add_negative_rule(domain)` | 命中返回 NXDOMAIN |
| CNAME | `add_cname_rule(domain, target)` | 重定向到目标域名 |

匹配优先级：地址规则 > CNAME 规则。两棵独立基数树，结果合并返回。

## 上游查询（detail::upstream）

管理一组上游服务器，根据解析策略发起异步查询。

| 协议 | 方法 | 说明 |
|------|------|------|
| UDP | `query_udp()` | `async_send_to` / `async_receive_from` |
| TCP | `query_tcp()` | 2 字节大端长度前缀帧格式 |
| DoT | `query_tls()` | TLS 连接上承载 TCP 帧格式，SNI 支持 |
| DoH | `query_https()` | HTTP POST + `application/dns-message` |

DoT/DoH 复用 SSL 上下文，缓存键为 `{hostname, verify_peer}` 组合。

**查询结果（query_result）：** 包含原始响应报文 message、提取的 IP 列表、
RTT（毫秒）、上游服务器标识和错误码。

## 报文编解码（detail::format）

DNS 报文（RFC 1035）的完整表示，包含 Header、Question、Answer、
Authority、Additional 四个段。

| 方法 | 说明 |
|------|------|
| `pack()` | 序列化为 DNS wire format |
| `unpack(data)` | 从二进制反序列化，支持域名压缩指针 |
| `make_query(domain, qtype)` | 创建标准递归查询报文 |
| `extract_ips()` | 提取所有 A/AAAA 记录的 IP |
| `min_ttl()` | 计算所有记录的最小 TTL |

支持 A、AAAA、CNAME、NS、MX、TXT、SOA、PTR、OPT 等记录类型。
`pack_tcp()` / `unpack_tcp()` 处理 TCP 传输的 2 字节大端长度前缀。
