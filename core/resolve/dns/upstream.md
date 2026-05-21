---
title: "upstream — DNS 查询客户端"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/upstream.hpp"
module: "resolve/dns"
type: component
tags: [dns, upstream, udp, tcp, dot, doh]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/dns
  - core/resolve/dns/config
  - core/resolve/dns/detail/format
---

# upstream — DNS 查询客户端

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/upstream.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]]

## 组件定位

`upstream` 是 DNS 查询的核心客户端，提供基于 Boost.Asio 协程的异步 DNS 查询能力，支持 UDP、TCP、DoT (DNS over TLS) 和 DoH (DNS over HTTPS) 四种传输协议。

## 支持的传输协议

| 协议 | 方法 | 默认端口 | 特点 |
|------|------|----------|------|
| UDP | `query_udp()` | 53 | 标准查询，小型响应 |
| TCP | `query_tcp()` | 53 | 2 字节长度前缀帧格式，大型响应 |
| DoT | `query_tls()` | 853 | TLS 加密，证书验证可选 |
| DoH | `query_https()` | 443 | HTTPS POST，`/dns-query` 路径 |

## 核心类型

### query_result

```cpp
struct query_result {
    message response;                        // DNS 响应报文
    memory::vector<net::ip::address> ips;    // 提取的 IP 地址列表
    uint64_t rtt_ms{0};                      // 往返时间（毫秒）
    memory::string server_addr;              // 响应来源上游服务器
    fault::code error{fault::code::success}; // 错误码
};
```

封装单次 DNS 查询的完整输出，用于解析策略比较和结果聚合。

### message 与 qtype

从 `detail/format.hpp` 引入：

```cpp
using detail::message;  // DNS 报文类
using detail::qtype;    // 查询类型枚举（A=1, AAAA=28）
```

## upstream 类

### 构造与配置

```cpp
explicit upstream(net::io_context &ioc, 
                  memory::resource_pointer mr = memory::current_resource());

void set_servers(memory::vector<dns_remote> servers);  // 设置上游服务器列表
void set_mode(resolve_mode mode);                       // 设置解析策略
void set_timeout(uint32_t ms);                          // 设置默认超时
```

### 核心查询方法

```cpp
[[nodiscard]] auto resolve(std::string_view domain, qtype qt)
    -> net::awaitable<query_result>;
```

根据当前解析策略向配置的上游服务器发起查询。

## 三种解析策略

### fastest（默认）

```
并发查询所有上游 → 选择 RTT 最低的成功响应
```

- 适用于追求最快响应的场景
- 所有上游同时发起查询
- 比较各上游的 RTT，返回最优结果

### first

```
并发查询所有上游 → 返回第一个成功响应
```

- 适用于追求可靠性的场景
- 不等待所有响应，首个成功即返回
- 降低等待延迟

### fallback

```
按顺序逐一尝试上游 → 前一个失败后尝试下一个
```

- 适用于单上游或高容错场景
- 串行尝试，避免无效并发
- 上游列表按优先级排序

## 私有查询方法实现

### query_udp

```cpp
auto query_udp(const dns_remote &server, const message &query)
    -> net::awaitable<query_result>;
```

流程：
1. 创建 UDP socket
2. 序列化查询报文 `query.pack()`
3. 发送到 `server.address:server.port`
4. 等待响应（超时 `server.timeout_ms`）
5. 解析响应 `message::unpack()`
6. 提取 IP 地址 `response.extract_ips()`

### query_tcp

```cpp
auto query_tcp(const dns_remote &server, const message &query)
    -> net::awaitable<query_result>;
```

流程：
1. 创建 TCP socket
2. 连接到上游服务器
3. 发送 2 字节大端长度前缀 + DNS 报文
4. 接收响应（先读 2 字节长度，再读报文）
5. 解析响应

**TCP 帧格式**:
```
[2 bytes length (big-endian)] [DNS message bytes]
```

### query_tls

```cpp
auto query_tls(const dns_remote &server, const message &query)
    -> net::awaitable<query_result>;
```

流程：
1. 获取或创建 SSL 上下文 `get_ssl_context(server)`
2. 创建 TLS socket
3. 设置 SNI 为 `server.hostname`
4. TLS 握手
5. 使用 TCP 帧格式发送/接收

**配置参数**:
- `hostname` — TLS SNI 主机名
- `no_check_certificate` — 是否跳过证书验证

### query_https

```cpp
auto query_https(const dns_remote &server, const message &query)
    -> net::awaitable<query_result>;
```

流程：
1. 创建 HTTPS 连接
2. 构造 HTTP POST 请求
3. Content-Type: `application/dns-message`
4. Body: DNS 报文 wire format
5. 解析 HTTP 响应体中的 DNS 报文

**HTTP 请求格式**:
```
POST /dns-query HTTP/1.1
Host: {hostname}
Content-Type: application/dns-message
Content-Length: {length}

{DNS wire format bytes}
```

## SSL 上下文缓存

### 缓存键结构

```cpp
struct ssl_cache_key {
    memory::string hostname;     // TLS 主机名
    bool verify_peer;            // 是否验证对端证书
};

// 缓存表：避免重复创建相同配置的 SSL 上下文
memory::unordered_map<ssl_cache_key, std::shared_ptr<ssl::context>, ssl_cache_key_hash> ssl_cache_;
```

### get_ssl_context

```cpp
auto get_ssl_context(const dns_remote &server) -> std::shared_ptr<ssl::context>;
```

- 查找缓存：`ssl_cache_.find({hostname, !no_check_certificate})`
- 未命中则创建新 SSL 上下文并加入缓存
- 复用相同配置的 SSL 上下文，避免重复开销

## 成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `ioc_` | `net::io_context &` | IO 上下文引用 |
| `mr_` | `memory::resource_pointer` | 内存资源 |
| `servers_` | `vector<dns_remote>` | 上游服务器列表 |
| `mode_` | `resolve_mode` | 解析策略（默认 fastest） |
| `timeout_ms_` | `uint32_t` | 默认超时（默认 4000ms） |
| `ssl_cache_` | `unordered_map` | SSL 上下文缓存 |

## 调用链

```
upstream::resolve(domain, qtype)
  │
  ├─→ fastest 模式
  │     → 并发调用 query_udp/tcp/tls/https
  │     → 等待所有结果或超时
  │     → 比较 RTT，返回最低的成功响应
  │
  ├─→ first 模式
  │     → 并发调用 query_udp/tcp/tls/https
  │     → 第一个成功响应立即返回
  │
  └─→ fallback 模式
        → 按 servers_ 顺序逐一调用
        → 失败后尝试下一个上游
```

## 内部依赖

```
upstream
  ├── → detail::format (message, qtype)
  ├── → config (dns_remote, resolve_mode)
  ├── → memory::container
  └── → fault::code
```

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/resolve/dns/config|dns_remote]] — 上游服务器配置
- [[core/resolve/dns/detail/format|message]] — DNS 报文编解码

---

## 上游服务器选择逻辑

### 选择决策树

```
upstream::resolve(domain, qtype)
    │
    ▼
resolve_mode 是什么?
    │
    ├── fastest (默认)
    │     │
    │     ├── 遍历所有 servers_
    │     ├── 对每个 server 创建查询任务
    │     ├── 并发执行所有查询
    │     ├── 等待所有结果或超时
    │     ├── 过滤失败的查询
    │     ├── 在成功结果中选择 RTT 最低的
    │     └── 无成功结果 → 返回最后一个错误
    │
    ├── first
    │     │
    │     ├── 遍历所有 servers_
    │     ├── 对每个 server 创建查询任务
    │     ├── 并发执行所有查询
    │     ├── 等待首个成功结果
    │     ├── 返回首个成功
    │     └── 全部失败 → 返回首个错误
    │
    └── fallback
          │
          ├── 按 servers_ 顺序逐一执行
          ├── server[0]: 查询
          │     ├── 成功 → 立即返回
          │     └── 失败 → 继续下一个
          ├── server[1]: 查询
          │     └── ...
          └── 全部失败 → 返回最后一个错误
```

### fastest 模式详解

```cpp
auto resolve_fastest(domain, qt) -> awaitable<query_result> {
    std::vector<awaitable<query_result>> futures;

    // 为每个上游创建并发查询
    for (const auto& server : servers_) {
        futures.push_back(query_server(server, make_query(domain, qt)));
    }

    // 等待所有查询
    auto results = co_await when_all(std::move(futures));

    // 选择最佳结果 (RTT 最低的成功响应)
    query_result* best = nullptr;
    for (auto& r : results) {
        if (r.error == fault::code::success) {
            if (!best || r.rtt_ms < best->rtt_ms) {
                best = &r;
            }
        }
    }

    if (best) co_return std::move(*best);
    co_return results.back();  // 返回最后的结果（含错误信息）
}
```

**并发模型**:
```
Server A ──▶ query ──▶ [RTT=15ms] ──▶ 成功 ──┐
Server B ──▶ query ──▶ [RTT=25ms] ──▶ 成功 ──┤── 选 A (RTT 最低)
Server C ──▶ query ──▶ [timeout] ──▶ 失败 ────┘
```

### first 模式详解

```cpp
auto resolve_first(domain, qt) -> awaitable<query_result> {
    std::vector<awaitable<query_result>> futures;

    for (const auto& server : servers_) {
        futures.push_back(query_server(server, make_query(domain, qt)));
    }

    // 等待首个成功结果
    auto results = co_await when_any(std::move(futures));
    if (results.value.error == fault::code::success)
        co_return std::move(results.value);

    // 首个失败了，继续等待其他
    // ... 或者返回错误
}
```

**与 fastest 的区别**:
- `fastest` 等待所有响应完成后比较 RTT
- `first` 在首个成功响应到达时立即返回
- `first` 的延迟可能更低，但不保证是最优结果

### fallback 模式详解

```cpp
auto resolve_fallback(domain, qt) -> awaitable<query_result> {
    fault::code last_error;

    for (const auto& server : servers_) {
        auto result = co_await query_server(server, make_query(domain, qt));
        if (result.error == fault::code::success) {
            co_return result;
        }
        last_error = result.error;
    }

    co_return query_result{.error = last_error};
}
```

**串行模型**:
```
Server A ──▶ query ──▶ 失败 ──▶ 继续
Server B ──▶ query ──▶ 失败 ──▶ 继续
Server C ──▶ query ──▶ 成功 ──▶ 返回

总延迟 = RTT_A + RTT_B + RTT_C
```

## Fallback 机制详解

### 查询失败场景

| 失败原因 | 错误码 | fallback 行为 |
|----------|--------|---------------|
| 上游不可达 | `host_unreachable` | 立即尝试下一上游 |
| 查询超时 | `timed_out` | 超时后尝试下一上游 |
| DNS 错误 (SERVFAIL) | `dns_server_failure` | 尝试下一上游 |
| 空响应 (NOERROR, 0 answers) | `success` (但 ips 为空) | 视情况继续 |
| 网络断开 | `network_error` | 立即尝试下一上游 |

### 超时传递

每个 `dns_remote` 可独立配置 `timeout_ms`:

```json
{
  "servers": [
    {"address": "8.8.8.8", "timeout_ms": 3000},
    {"address": "1.1.1.1", "timeout_ms": 5000},
    {"address": "9.9.9.9", "timeout_ms": 10000}
  ]
}
```

fallback 模式下，上游按顺序尝试，每个有自己的超时时间。

### 智能回退策略

```
主上游 (最快但可能不稳定)
    ↓ 失败
备用上游 (次快但更稳定)
    ↓ 失败
兜底上游 (最慢但最可靠)
    ↓ 失败
返回错误
```

### 各传输协议的 fallback 行为

```
query_udp(server, query):
    ├── 成功 → 返回结果
    ├── 超时 → 尝试 query_tcp (UDP fallback to TCP)
    └── 截断 (TC=1) → 自动重试 query_tcp

query_tcp(server, query):
    ├── 成功 → 返回结果
    ├── 连接失败 → 返回错误
    └── 超时 → 返回错误

query_tls(server, query):
    ├── TLS 握手失败 → 返回错误
    │     (证书无效、SNI 不匹配等)
    ├── 成功 → 返回结果
    └── 超时 → 返回错误

query_https(server, query):
    ├── HTTPS 连接失败 → 返回错误
    ├── HTTP 错误状态 → 返回错误
    ├── 成功 → 返回结果
    └── 超时 → 返回错误
```

### UDP 截断自动重试

DNS over UDP 响应有大小限制（通常 512 字节）。当响应超过限制：

```
query_udp → 收到响应
    │
    检查 TC (Truncated) 标志
    │
    ├── TC=0 (完整) → 正常处理
    └── TC=1 (截断)
          │
          自动重试 query_tcp
          │
          返回 TCP 完整响应
```

这是 DNS 协议的标准行为，确保大响应（如包含大量记录的域）不被截断。

## 上游选择策略推荐

| 场景 | 推荐模式 | 上游数量 | 说明 |
|------|----------|----------|------|
| 标准场景 | fastest | 2-3 | 自动选择最优响应 |
| 低延迟优先 | first | 2-3 | 首个响应即返回 |
| 高可靠优先 | fallback | 2+ | 保证有序尝试 |
| 单上游 | 任意 | 1 | 模式无区别 |
| DoT/DoH 混合 | fastest | 2-4 | 利用不同协议优势 |