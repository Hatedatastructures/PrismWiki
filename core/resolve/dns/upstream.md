---
title: "upstream -- DNS 查询客户端"
layer: core
source: "include/prism/resolve/dns/upstream.hpp"
module: "resolve/dns"
type: component
tags: [dns, upstream, udp, tcp, dot, doh]
created: 2026-05-17
updated: 2026-05-28
related:
  - core/resolve/dns/dns
  - core/resolve/dns/config
  - core/resolve/dns/detail/format
---

# upstream -- DNS 查询客户端

> 源码: `include/prism/resolve/dns/upstream.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]]

## 组件定位

DNS 查询核心客户端，提供基于 Boost.Asio 协程的异步 DNS 查询能力。所有异步接口返回 `net::awaitable<T>`，错误通过 `fault::code` 表达，不抛出异常。所有内部缓冲区使用 PMR 分配。

## 支持的传输协议

| 协议 | 方法 | 默认端口 | 特点 |
|------|------|----------|------|
| UDP | `query_udp` | 53 | 标准查询，小型响应，TC=1 截断时自动回退 TCP |
| TCP | `query_tcp` | 53 | 2 字节大端长度前缀帧格式，大型响应 |
| DoT | `query_tls` | 853 | TLS 加密 + TCP 帧格式，证书验证可选 |
| DoH | `query_https` | 443 | HTTP POST，`/dns-query`，`application/dns-message` |

## 核心类型

### query_result

| 成员 | 类型 | 说明 |
|------|------|------|
| `response` | `message` | DNS 响应报文 |
| `ips` | `vector<ip::address>` | 提取的 IP 地址列表 |
| `rtt_ms` | `uint64_t` | 往返时间（毫秒） |
| `server_addr` | `string` | 响应来源上游服务器 |
| `error` | `fault::code` | 错误码（默认 success，但 ips 为空应视为逻辑失败） |

### 类型导入

`message` 和 `qtype` 从 `detail/format.hpp` 引入：message 为 DNS 报文类，qtype 为查询类型枚举 (A=1, AAAA=28)。

---

## upstream 类接口

### 配置方法

| 方法 | 说明 |
|------|------|
| `upstream(io_context &, mr)` | 构造，初始化服务器列表和 SSL 缓存 |
| `set_servers(vector<dns_remote>)` | 设置上游服务器列表 |
| `set_mode(resolve_mode)` | 设置解析策略 (fastest/first/fallback) |
| `set_timeout(uint32_t ms)` | 设置默认超时（默认 4000ms） |

### 核心查询

| 方法 | 说明 |
|------|------|
| `resolve(domain, qtype) -> awaitable<query_result>` | 根据当前策略查询域名 |

### 私有成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `ioc_` | `io_context &` | IO 上下文 |
| `mr_` | `resource_pointer` | 内存资源 |
| `servers_` | `vector<dns_remote>` | 上游服务器列表 |
| `mode_` | `resolve_mode` | 解析策略（默认 fastest） |
| `timeout_ms_` | `uint32_t` | 默认超时 |
| `ssl_cache_` | `unordered_map<ssl_key, shared_ptr<ssl::context>>` | SSL 上下文缓存 |

---

## 三种解析策略

### fastest (默认)

并发查询所有上游，等待全部完成后选择 RTT 最低的成功响应。适用于追求最优结果的场景。

**并发模型**: 所有上游同时发起查询，比较 RTT 选最优。

### first

并发查询所有上游，首个成功响应立即返回。适用于追求低延迟的场景。

**与 fastest 的区别**: first 不等待所有响应完成，延迟可能更低但不保证最优结果。

### fallback

按 servers_ 顺序逐一尝试，前一个失败后尝试下一个。总延迟为各上游 RTT 之和。适用于高容错场景。

### 策略选择建议

| 场景 | 推荐模式 | 上游数量 |
|------|----------|----------|
| 标准场景 | fastest | 2-3 |
| 低延迟优先 | first | 2-3 |
| 高可靠优先 | fallback | 2+ |
| 单上游 | 任意（无区别） | 1 |
| DoT/DoH 混合 | fastest | 2-4 |

---

## 传输协议实现要点

### query_udp

序列化查询 -> 发送到上游 -> 等待响应（超时 `server.timeout_ms`）-> 解析响应 -> 提取 IP。

**UDP 截断自动重试**: 当响应 TC (Truncated) 标志为 1 时，自动通过 `query_tcp` 重试，确保大响应不被截断。

### query_tcp

2 字节大端长度前缀 + DNS 报文。先读 2 字节长度，再读报文。

### query_tls

获取或创建 SSL 上下文（缓存） -> TLS socket -> SNI 设置 -> 握手 -> TCP 帧格式收发。

配置参数: `hostname` (TLS SNI)、`no_check_certificate` (是否跳过验证)。

### query_https

HTTP POST `/dns-query`，Content-Type `application/dns-message`，Body 为 DNS wire format。

---

## SSL 上下文缓存

缓存键为 `(hostname, verify_peer)` 组合，避免重复创建相同配置的 SSL 上下文。哈希函数组合主机名哈希和验证标志哈希。

### 为什么缓存 SSL 上下文

SSL 上下文创建涉及证书加载等昂贵操作。相同配置的上游共享上下文可减少开销。`ssl::context` 是只读配置对象，握手后每个连接持有独立 `ssl::stream`，缓存共享是安全的。

---

## Fallback 行为

| 失败原因 | 错误码 | fallback 行为 |
|----------|--------|---------------|
| 上游不可达 | `host_unreachable` | 立即尝试下一上游 |
| 查询超时 | `timed_out` | 超时后尝试下一上游 |
| DNS SERVFAIL | `dns_server_failure` | 尝试下一上游 |
| 网络断开 | `network_error` | 立即尝试下一上游 |

每个 `dns_remote` 可独立配置 `timeout_ms`，fallback 模式下每个上游使用各自的超时时间。

## 内部依赖

| 依赖 | 用途 |
|------|------|
| `detail::format` | message, qtype |
| `config` | dns_remote, resolve_mode |
| `memory::container` | PMR 容器 |
| `fault::code` | 错误码 |

## 参见

- [[core/resolve/dns/dns|resolver]] -- DNS 解析器接口
- [[core/resolve/dns/config|dns_remote]] -- 上游服务器配置
- [[core/resolve/dns/detail/format|message]] -- DNS 报文编解码
