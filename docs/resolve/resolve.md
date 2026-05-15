---
title: Resolve 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [resolve, dns, routing, cache, rules, router, overview]
related: [[multiplex], [channel], [stealth/overview], [client/mihomo-dns], [agent/overview], [memory/overview], [fault/overview]]
---

# Resolve 模块

## 模块定位

Resolve 是 Prism 的路由与 DNS 解析层。向上为 [[multiplex]] 的 `core`
和各协议处理器提供统一的路由接口，向下使用 [[channel]] 的
`connection_pool` 建立和管理连接。[[stealth]] 的伪装方案也通过
`router` 建立 fallback 连接。

Router 是该模块的顶层门面，整合 DNS 解析器、反向路由表和连接池，
为上层提供多种路由模式的统一入口。

## 路由器（router）

`router` 是分发层的顶层协调器，支持四种路由模式：

| 方法 | 路由模式 | 说明 |
|------|----------|------|
| `async_reverse(host)` | 反向代理 | 查找预配置的路由表获取目标端点 |
| `async_direct(endpoint)` | 直连 | 直接通过连接池获取到指定端点的连接 |
| `async_forward(host, port)` | 正向代理 | 通过 DNS 解析域名后建立连接 |
| `async_datagram(host, port)` | 数据报 | DNS 解析后创建 UDP socket |

`connect_with_retry` 内部按顺序尝试候选端点列表，最多尝试 3 个端点。
`set_positive_endpoint` 配置正向代理端点后，`async_forward` 转发到该端点而非直连。
`add_reverse_route` 添加反向路由规则，将指定主机名路由到给定端点。

路由器内部使用透明哈希函数（`string_hash` + `string_equal`）支持
`string_view` 和 `memory::string` 异构查找，避免临时键对象分配。

## DNS 解析器

`dns::resolver` 是纯虚接口，上层通过此接口进行 DNS 解析，不依赖具体实现：

- `resolve(host)` — 异步解析域名到 IP 地址列表（同时查询 A 和 AAAA）
- `resolve_tcp(host, port)` — 异步解析到 TCP 端点列表
- `resolve_udp(host, port)` — 异步解析到 UDP 端点（优先 A，回退 AAAA）
- `ipv6_disabled()` — 查询 IPv6 是否禁用

`make_resolver()` 工厂函数创建具体实现，内部持有 upstream、cache、rules_engine、
coalescer 等组件，构成完整的 DNS 查询管道。

## DNS 查询管道

DNS 解析器内部由多个组件协作形成查询管道：

1. **规则匹配**（`rules_engine`）— 基于反转域名基数树的高效匹配
2. **缓存查找**（`cache`）— 内存缓存，支持正向缓存和负缓存
3. **请求合并**（`coalescer`）— 并发请求同一域名时合并为单次查询
4. **上游查询**（`upstream`）— 向配置的 DNS 服务器发起异步查询
5. **IP 过滤** — 基于黑名单过滤解析结果

### 上游客户端（upstream）

`dns::upstream` 管理一组上游 DNS 服务器，支持四种传输协议：

- **UDP**：标准 DNS 查询，适用于小型响应，截断时回退至 TCP
- **TCP**：带 2 字节长度前缀的 DNS 帧格式，适用于大型响应
- **DoT**：DNS over TLS，端口 853
- **DoH**：DNS over HTTPS，HTTP POST，端口 443

解析策略（`resolve_mode`）：
- **fastest**：并发查询所有上游，返回 RTT 最低的成功响应
- **first**：并发查询所有上游，返回第一个成功响应
- **fallback**：按顺序尝试上游，前一个失败后尝试下一个

### 缓存（cache）

支持正向缓存和负缓存。缓存策略包括 TTL 过期、serve-stale 模式
（过期后仍返回旧数据，允许后台刷新）和 FIFO 淘汰。
键格式为 `"domain:qtype_number"`，通过 `make_key_view` 实现零分配查找。

### 请求合并（coalescer）

当多个协程同时请求同一域名时，仅执行一次实际查询，
其他协程通过定时器挂起等待结果复用。`find_or_create` 查找或创建
flight 记录，`flush_cleanup` 延迟删除避免迭代器失效。

### 规则引擎（rules_engine）

基于反转域名基数树（domain_trie）实现高效域名匹配：
将 `"www.example.com"` 存储为 `["com", "example", "www"]`，
使后缀匹配等价于前缀遍历。支持精确匹配和通配符匹配
（`*.example.com` 在 `example` 节点标记 wildcard）。
规则类型包括静态地址映射、否定规则（广告屏蔽，返回 NXDOMAIN）和 CNAME 重定向。

## 设计原则

- **协程架构**：所有异步操作返回 `net::awaitable<T>`，使用 `co_await` 执行
- **错误码返回**：热路径使用 `fault::code`，不抛异常
- **PMR 内存**：所有容器使用 PMR 分配器，支持外部注入内存资源
- **请求合并**：并发请求同一域名时合并为单次查询，降低上游压力
- **serve-stale**：缓存过期后仍可返回旧数据，允许异步后台刷新

## 核心类实现细节

### router 实现细节

`resolve/router.hpp` + `.cpp`。

```
class router:
  router(connection_pool&, io_context&, dns::config, resource_pointer)

  set_positive_endpoint(host, port)
  add_reverse_route(host, endpoint)

  async_reverse(host) -> awaitable<pair<fault::code, pooled_connection>>
  async_direct(endpoint) -> awaitable<pair<fault::code, pooled_connection>>
  async_forward(host, port) -> awaitable<pair<fault::code, pooled_connection>>
  async_datagram(host, port) -> awaitable<pair<fault::code, udp::socket>>
  resolve_datagram_target(host, port) -> awaitable<pair<fault::code, udp::endpoint>>
  ipv6_disabled() -> bool

private:
  async_positive(host, port)
  connect_with_retry(endpoints)         // 最多尝试 3 个端点

  connection_pool& pool_
  unique_ptr<dns::resolver> dns_
  reverse_map reverse_map_              // hash_map<string, endpoint>
  optional<string> positive_host_
  uint16 positive_port_
```

路由逻辑：reverse 查表 → direct 直连 → forward DNS → positive 正向代理。
透明哈希：`string_hash` + `string_equal`，`is_transparent` 异构查找。
`open_udp_socket(executor, target)` 自动选择 IPv4/IPv6。

### dns::upstream 实现细节

`resolve/dns/upstream.hpp` + `.cpp`。UDP/TCP/DoT/DoH 四种协议。

```
class upstream:
  set_servers(vector<dns_remote>)
  set_mode(resolve_mode)        // fastest/first/fallback
  resolve(domain, qtype) -> awaitable<query_result>
private:
  query_udp/tcp/tls/https(server, query) -> awaitable<query_result>
  get_ssl_context(server) -> shared_ptr<ssl::context>  // 按 (hostname,verify_peer) 缓存
```

`query_result`：response 报文、ips、rtt_ms、server_addr、error。

### dns::config

`resolve/dns/config.hpp`。header-only + glaze JSON。

```
enum dns_protocol: udp, tcp, tls, https
enum resolve_mode: fastest, first, fallback
struct dns_remote { address, protocol, hostname, port, timeout_ms, http_path, no_check_certificate }
struct address_rule { domain, addresses, negative }
struct cname_rule { domain, target }
struct config { servers, mode, timeout_ms, cache_*, ttl_*, rules, blacklist, disable_ipv6 }
```

### dns::detail::cache 实现细节

`resolve/dns/detail/cache.hpp` + `.cpp`。

```
class cache:
  cache(mr, ttl=120s, max_entries=10000, serve_stale=true)
  get(domain, qtype) -> optional<vector<ip::address>>
  put(domain, qtype, ips, ttl) / put_negative(domain, qtype, negative_ttl=30s) / evict_expired()
```

键 `"domain:qtype_number"`，`make_key_view` 零分配查找。
`cache_entry`：ips + ttl + expire + inserted + failed。
`list<string>` LRU + `unordered_map` 索引。serve-stale + FIFO 淘汰。

### dns::detail::coalescer 实现细节

`resolve/dns/detail/coalescer.hpp`。header-only。

`find_or_create(key, executor)` → flight 记录，首个协程 `is_new=true`，
其他通过 `timer` 等待。`cleanup_flight`/`flush_cleanup` 延迟删除。
`list<flight>` + `unordered_map<string_view, iterator>` 索引。

### dns::detail::rules_engine 实现细节

`resolve/dns/detail/rules.hpp` + `.cpp`。

```
struct rule_result { addresses, cname, negative, blocked }
class rules_engine:
  add_address_rule(domain, ips) / add_negative_rule(domain) / add_cname_rule(domain, target)
  match(domain) -> optional<rule_result>
```

反转基数树 `domain_trie`：`"www.example.com"` → `["com","example","www"]`。
精确 + 通配符匹配（`*.example.com` 在 `example` 节点标记 wildcard）。

### dns::detail::format（报文编解码）

`resolve/dns/detail/format.hpp` + `.cpp`。

```
enum qtype: a=1, ns=2, cname=5, soa=6, mx=15, txt=16, aaaa=28, opt=41
struct question { name, query_type, qclass }
struct record { name, type, rclass, ttl, rdata }
class message: id, qr, opcode, flags, rcode, questions, answers, authority, additional
  pack() / unpack() / make_query() / extract_ips() / min_ttl()
```

`pack_tcp/unpack_tcp`：2 字节大端 TCP 帧。压缩指针循环检测（>255 次返回 nullopt）。

### 辅助组件

`dns/detail/transparent.hpp`：`transparent_hash`（FNV-1a）+ `transparent_equal`。
`dns/detail/utility.hpp`：工具函数。

## 文件清单

| 文件 | 说明 |
|------|------|
| `resolve/router.hpp` / `.cpp` | 路由器（四种路由模式 + 透明哈希） |
| `resolve/dns/dns.hpp` | DNS 解析器接口 + 聚合头文件 |
| `resolve/dns/resolver.cpp` | DNS 解析器实现（查询管道） |
| `resolve/dns/upstream.hpp` / `.cpp` | 上游客户端（UDP/TCP/DoT/DoH） |
| `resolve/dns/config.hpp` | DNS 配置（header-only + glaze JSON 元数据） |
| `resolve/dns/detail/cache.hpp` / `.cpp` | DNS 结果缓存 |
| `resolve/dns/detail/coalescer.hpp` | 请求合并器（header-only） |
| `resolve/dns/detail/rules.hpp` / `.cpp` | 域名规则引擎（反转基数树） |
| `resolve/dns/detail/format.hpp` / `.cpp` | DNS 报文编解码 |
| `resolve/dns/detail/transparent.hpp` | 透明哈希/比较（FNV-1a） |
| `resolve/dns/detail/utility.hpp` | 工具函数 |
| `resolve.hpp` | 聚合头文件 |

## 与其他模块的关系

- **上游**：[[multiplex]] 的 `core` 持有 `router` 引用发起路由请求
- **上游**：[[stealth]] 的方案通过 `router` 建立 fallback 连接
- **上游**：各协议处理器通过 `router` 建立到目标服务器的连接
- **下游**：使用 [[channel]] 的 `connection_pool` 管理 TCP 连接
- **配置**：`dns::config` 支持 glaze JSON 序列化
