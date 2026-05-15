---
title: "upstream.hpp — DNS 查询客户端"
source: "include/prism/resolve/dns/upstream.hpp"
module: "resolve"
type: api
tags: [resolve, dns, upstream, DNS查询, UDP, TCP, DoT, DoH]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - resolve/dns/config
  - resolve/dns/detail/format
  - fault/code
---

# upstream.hpp

> 源码: `include/prism/resolve/dns/upstream.hpp` + `src/prism/resolve/dns/upstream.cpp`
> 模块: [[resolve|Resolve]] / dns

## 概述

提供基于 Boost.Asio 协程的异步 DNS 查询能力，支持 UDP、TCP、DoT (DNS over TLS) 和 DoH (DNS over HTTPS) 四种传输协议。客户端可配置多上游服务器，并根据指定策略（first / fallback / fastest）选择最优响应。所有内部缓冲区使用 PMR 多态内存资源分配，与项目内存管理基础设施无缝集成。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[resolve/dns/detail/format|format]] | DNS 报文编解码（message、qtype） |
| 依赖 | [[resolve/dns/config|config]] | DNS 配置类型（dns_remote、resolve_mode） |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 依赖 | [[fault/code|code]] | 错误码 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver_impl 使用 upstream 发起查询 |

## 命名空间

`psm::resolve::dns`

---

## 结构体: query_result

### 概述

DNS 查询结果，封装一次 DNS 查询的完整输出，包括原始响应报文、提取的 IP 地址列表、往返时间（毫秒）、响应来源上游标识以及错误码。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `message` | `response` | DNS 响应报文 |
| `memory::vector<net::ip::address>` | `ips` | 从响应中提取的 IP 地址 |
| `uint64_t` | `rtt_ms` | 往返时间（毫秒） |
| `memory::string` | `server_addr` | 响应来自哪个上游服务器 |
| `fault::code` | `error` | 错误码（默认 `success`） |

---

### 函数: query_result::query_result()

- **功能说明**: 构造查询结果，使用指定内存资源初始化所有 PMR 容器成员。
- **签名**:
  ```cpp
  explicit query_result(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源指针，用于内部容器分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `upstream::resolve()` 及各 `query_*` 方法内部构造结果对象
- **知识域**: [[memory/container|PMR 容器]]

---

## 类: upstream

### 概述

异步 DNS 查询客户端，管理一组上游 DNS 服务器，根据配置的解析策略发起异步查询。

### 设计意图

支持四种传输协议：
- **UDP**：标准 DNS 查询，小型响应截断回退至 TCP
- **TCP**：带 2 字节长度前缀的 DNS 帧格式
- **DoT**：在 TLS 连接上承载 TCP 帧格式，端口 853
- **DoH**：在 HTTPS 上承载 HTTP POST 请求（RFC 8484），端口 443

解析策略（`resolve_mode`）：
- **first**：并发查询所有上游，返回第一个成功响应
- **fallback**：按顺序尝试上游，前一个失败后才尝试下一个
- **fastest**：并发查询所有上游，选择 RTT 最低的成功响应

### 内部传输结构体

实现文件中定义了四个传输结构体（`udp_transport`、`tcp_transport`、`tls_transport`、https_transport`），每个实现 `connect()` / `send()` / `recv()` / `close()` 接口，由模板函数 `query_via()` 统一编排查询流程。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `net::io_context &` | `ioc_` | IO 上下文 |
| `memory::resource_pointer` | `mr_` | 内存资源 |
| `memory::vector<dns_remote>` | `servers_` | 上游服务器列表 |
| `resolve_mode` | `mode_` | 解析策略（默认 fastest） |
| `uint32_t` | `timeout_ms_` | 默认超时（毫秒，默认 4000） |
| `memory::unordered_map<ssl_cache_key, ...>` | `ssl_cache_` | SSL 上下文缓存 |

---

### 函数: upstream::upstream()

- **功能说明**: 构造 DNS 查询客户端，初始化内部上游服务器列表和 SSL 上下文缓存。
- **签名**:
  ```cpp
  explicit upstream(net::io_context &ioc, memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `ioc` | `net::io_context &` | IO 上下文引用，用于创建套接字和定时器 |
  | `mr` | `memory::resource_pointer` | 内存资源指针，用于内部容器分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `resolver_impl` 构造函数中初始化
- **知识域**: [[resolve/dns/upstream|DNS 查询]]

---

### 函数: set_servers()

- **功能说明**: 替换当前所有上游服务器配置。每个 `dns_remote` 可独立配置协议、地址、端口和超时时间。
- **签名**:
  ```cpp
  void set_servers(memory::vector<dns_remote> servers);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `servers` | `memory::vector<dns_remote>` | 上游服务器配置向量 |

- **返回值**: 无
- **调用（向下）**: 无（赋值 `servers_` 成员）
- **被调用（向上）**: `resolver_impl` 构造函数中调用
- **知识域**: [[resolve/dns/config|DNS 配置]]

---

### 函数: set_mode()

- **功能说明**: 更改查询调度策略，影响后续 `resolve()` 调用的行为。
- **签名**:
  ```cpp
  void set_mode(resolve_mode mode);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mode` | `resolve_mode` | 解析策略（first / fallback / fastest） |

- **返回值**: 无
- **调用（向下）**: 无（赋值 `mode_` 成员）
- **被调用（向上）**: `resolver_impl` 构造函数中调用
- **知识域**: [[resolve/dns/config|DNS 配置]]

---

### 函数: set_timeout()

- **功能说明**: 设置默认超时时间。当上游服务器未配置独立超时时间时使用此默认值。
- **签名**:
  ```cpp
  void set_timeout(uint32_t ms);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `ms` | `uint32_t` | 超时时间（毫秒） |

- **返回值**: 无
- **调用（向下）**: 无（赋值 `timeout_ms_` 成员）
- **被调用（向上）**: `resolver_impl` 构造函数中调用
- **知识域**: [[resolve/dns/config|DNS 配置]]

---

### 函数: resolve()

- **功能说明**: 异步查询域名，根据当前解析策略向配置的上游服务器发起查询。构造 DNS 查询报文后，按模式分发：`fallback` 模式顺序尝试；`first` / `fastest` 模式并发查询所有上游，使用信号驱动机制（`steady_timer` 取消唤醒）等待结果，避免忙等待轮询。`first` 模式收到首个成功响应即返回；`fastest` 模式等待全部完成后选择 RTT 最低的结果。
- **签名**:
  ```cpp
  [[nodiscard]] auto resolve(std::string_view domain, qtype qt) -> net::awaitable<query_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 待查询的域名 |
  | `qt` | `qtype` | 查询类型（A / AAAA 等） |

- **返回值**: `net::awaitable<query_result>` — 包含响应报文、IP 列表、RTT 和错误码
- **调用（向下）**: `message::make_query()` → 按协议分发到 `query_udp()` / `query_tcp()` / `query_tls()` / `query_https()`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在上游查询阶段调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]、[[resolve/dns/config|DNS 配置]]

---

### 函数: query_udp() [private]

- **功能说明**: 通过 UDP 发送 DNS 查询。构造 UDP 报文发送到上游服务器，等待响应并解析结果。若响应 TC（截断）标志置位，自动回退到 TCP 重试。
- **签名**:
  ```cpp
  [[nodiscard]] auto query_udp(const dns_remote &server, const message &query)
      -> net::awaitable<query_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `server` | `const dns_remote &` | 目标上游服务器配置 |
  | `query` | `const message &` | DNS 查询报文 |

- **返回值**: `net::awaitable<query_result>` — 查询结果
- **调用（向下）**: `query_via(udp_transport{}, ...)` → TC 检测 → `query_tcp()` 回退
- **被调用（向上）**: `upstream::resolve()` 在 `dns_protocol::udp` 时调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]、[[resolve/dns/detail/format|DNS 报文]]

---

### 函数: query_tcp() [private]

- **功能说明**: 通过 TCP 发送 DNS 查询，使用 2 字节大端长度前缀的 TCP 帧格式收发 DNS 报文。
- **签名**:
  ```cpp
  [[nodiscard]] auto query_tcp(const dns_remote &server, const message &query)
      -> net::awaitable<query_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `server` | `const dns_remote &` | 目标上游服务器配置 |
  | `query` | `const message &` | DNS 查询报文 |

- **返回值**: `net::awaitable<query_result>` — 查询结果
- **调用（向下）**: `query_via(tcp_transport{}, ...)`
- **被调用（向上）**: `upstream::resolve()` 在 `dns_protocol::tcp` 时调用；`query_udp()` TC 回退时调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]

---

### 函数: query_tls() [private]

- **功能说明**: 通过 DoT (DNS over TLS) 发送 DNS 查询。在 TLS 连接上承载 TCP 帧格式，默认端口 853。支持 SNI 主机名设置和证书验证控制。
- **签名**:
  ```cpp
  [[nodiscard]] auto query_tls(const dns_remote &server, const message &query)
      -> net::awaitable<query_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `server` | `const dns_remote &` | 目标上游服务器配置 |
  | `query` | `const message &` | DNS 查询报文 |

- **返回值**: `net::awaitable<query_result>` — 查询结果
- **调用（向下）**: `get_ssl_context(server)` → `query_via(tls_transport{...}, ...)`
- **被调用（向上）**: `upstream::resolve()` 在 `dns_protocol::tls` 时调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]

---

### 函数: query_https() [private]

- **功能说明**: 通过 DoH (DNS over HTTPS) 发送 DNS 查询。在 HTTPS 连接上使用 HTTP POST 方法发送 DNS 报文，Content-Type 为 `application/dns-message`，默认端口 443。按 RFC 8484 规范构造 HTTP 请求。
- **签名**:
  ```cpp
  [[nodiscard]] auto query_https(const dns_remote &server, const message &query)
      -> net::awaitable<query_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `server` | `const dns_remote &` | 目标上游服务器配置 |
  | `query` | `const message &` | DNS 查询报文 |

- **返回值**: `net::awaitable<query_result>` — 查询结果
- **调用（向下）**: `get_ssl_context(server)` → `query_via(https_transport{...}, ...)`
- **被调用（向上）**: `upstream::resolve()` 在 `dns_protocol::https` 时调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]

---

### 函数: get_ssl_context() [private]

- **功能说明**: 获取或创建 SSL 上下文。根据上游服务器配置（hostname + verify_peer）查找缓存的 SSL 上下文，若未命中则创建新的 TLS 上下文并配置 SNI 回调和证书验证模式，然后加入缓存复用。
- **签名**:
  ```cpp
  [[nodiscard]] auto get_ssl_context(const dns_remote &server) -> std::shared_ptr<ssl::context>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `server` | `const dns_remote &` | 上游服务器配置 |

- **返回值**: `std::shared_ptr<ssl::context>` — SSL 上下文的共享指针
- **调用（向下）**: `ssl_cache_.find()` → 创建 `ssl::context` → 设置 SNI 回调 → `ssl_cache_[key] = ctx`
- **被调用（向上）**: `query_tls()` 和 `query_https()` 在建立 TLS 连接前调用
- **知识域**: [[resolve/dns/upstream|DNS 查询]]

---

## 调用链总览

```
[[resolve/dns/dns|resolver_impl]] → upstream::resolve()
  ├── fallback 模式: 顺序 query_udp/tcp/tls/https()
  └── first/fastest 模式: 并发 co_spawn → 信号驱动等待

query_udp()  → query_via(udp_transport) → TC 检测 → query_tcp() 回退
query_tcp()  → query_via(tcp_transport)
query_tls()  → get_ssl_context() → query_via(tls_transport)
query_https()→ get_ssl_context() → query_via(https_transport)

query_via<T>(): connect → pack → send → recv → unpack → validate → extract_ips
```

---

## 知识域

- [[ref/protocol/dns-over-udp|DNS 解析]]
- [[resolve/dns/upstream|DNS 传输协议]]
- [[resolve/dns/config|DNS 配置]]
