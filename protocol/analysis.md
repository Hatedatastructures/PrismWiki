---
title: "analysis — 协议分析与识别"
source: "include/prism/protocol/analysis.hpp"
module: "protocol"
type: api
tags: [protocol, analysis, 识别, 分析, target, protocol_type]
related:
  - "[[protocol/http/parser|parser]]"
  - "[[pipeline/pipeline|pipeline]]"
  - "[[outbound/proxy|proxy]]"
  - "[[protocol/common/address|address]]"
created: 2026-05-15
updated: 2026-05-15
---

# analysis.hpp

> 源码: `include/prism/protocol/analysis.hpp`
> 模块: [[protocol|protocol]]

## 概述

协议分析与识别。提供协议探测、目标地址解析等静态辅助方法，是代理系统协议栈的核心组件。该模块负责解析客户端请求、识别应用层协议类型、提取目标地址信息，为路由决策提供关键输入数据。核心功能包括协议探测、地址解析和协议转换。

设计原则：
- 无状态性：所有方法都是静态的，不维护内部状态
- 高性能：使用 `std::string_view` 避免数据拷贝
- 内存安全：使用 `memory::string` 管理内存
- 错误容忍：解析失败时返回合理默认值，不抛出异常

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory|memory]] | `memory::string`、`memory::resource_pointer` |
| 依赖 | [[protocol/http/parser|parser]] | HTTP 代理请求解析 |
| 被依赖 | [[outbound/proxy|proxy]] | 出站代理使用 `target` |
| 被依赖 | [[pipeline/pipeline|pipeline]] | 协议处理管道 |
| 被依赖 | [[recognition/probe/probe|probe]] | 协议探测 |

## 命名空间

`psm::protocol`

## 枚举: protocol_type

标识当前连接所使用的应用层协议类型，用于协议探测和路由决策。使用 `enum class` 提供类型安全，避免隐式转换。值顺序固定，可用于 `switch` 语句优化。

| 值 | 数值 | 说明 |
|----|------|------|
| `unknown` | `0` | 未知协议（探测失败时的默认值） |
| `http` | `1` | HTTP 协议 |
| `socks5` | `2` | SOCKS5 协议 |
| `trojan` | `3` | Trojan 协议 |
| `vless` | `4` | VLESS 协议 |
| `shadowsocks` | `5` | Shadowsocks 2022 协议 |
| `tls` | `6` | TLS 协议（通用类别，包含多种基于 TLS 的代理协议） |

## 函数: to_string_view()

### 功能说明

将 `protocol_type` 枚举值转换为可读的字符串表示，用于日志输出、调试和监控。提供编译时已知的字符串字面量，无运行时分配开销。设计为 `inline` 建议编译器内联展开，消除函数调用开销。

### 签名

```cpp
inline auto to_string_view(const protocol_type type) -> std::string_view;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `type` | `const protocol_type` | 协议类型枚举值 |

### 返回值

`std::string_view` — 协议类型的字符串表示，指向编译时常量。返回的字符串视图指向静态存储期的字符串字面量，生命周期与程序相同。

### 调用（向下）

无（纯函数，无外部依赖）

### 被调用（向上）

- `trace` 日志模块输出协议类型
- `session` 会话层记录协议类型
- `pipeline` 管道层日志

### 知识域

枚举序列化、日志输出

---

## 结构体: analysis::target

### 功能说明

封装了解析出的目标主机、端口以及是否需要正向代理，是路由决策的关键输入。使用 `memory::string` 管理内存，确保与线程局部内存池兼容。路由语义方面，当 `positive` 为 `true` 时表示客户端请求使用正向代理，当 `positive` 为 `false` 时表示普通请求或反向代理请求。`psm::resolve::router` 根据此标志选择正向或反向路由。

### 签名

```cpp
struct target {
    explicit target(memory::resource_pointer mr = memory::current_resource());
    memory::string host;
    memory::string port;
    bool positive{false};
};
```

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `host` | `memory::string` | 空 | 目标主机名或 IP 地址 |
| `port` | `memory::string` | `"80"` | 目标端口号（字符串形式） |
| `positive` | `bool` | `false` | 是否为正向代理请求 |

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mr` | `memory::resource_pointer` | `memory::current_resource()` | 内存资源指针，用于初始化 `host` 和 `port` 字符串的内存分配器 |

### 调用（向下）

- `memory::string` 构造函数（使用 PMR 分配器）

### 被调用（向上）

- [[protocol/http/parser|parser]] — HTTP 解析后构建 target
- [[outbound/proxy|proxy]] — 出站代理使用 target 建立连接
- [[pipeline/pipeline|pipeline]] — 管道层传递 target 给路由

### 知识域

- [[memory|memory]] PMR 内存管理
- [[resolve/router|router]] 路由决策
- [[protocol/common/address|address]] 地址解析

---

## 函数: analysis::resolve()（HTTP 请求）

### 功能说明

解析 HTTP 请求，提取目标主机和端口信息。支持 HTTP/1.1 的绝对 URI 格式和 Host 头字段。解析策略为首先检查请求行是否包含绝对 URI，如果存在则从中提取主机和端口，否则从 Host 头字段提取。支持 HTTP 代理请求的 CONNECT 方法，处理带端口的 Host 头格式，自动识别是否为正向代理请求。

### 签名

```cpp
static auto resolve(const http::proxy_request &req, memory::resource_pointer mr = nullptr)
    -> target;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `req` | `const http::proxy_request &` | HTTP 请求对象，包含请求行和头部字段 |
| `mr` | `memory::resource_pointer` | 内存资源指针，为空时使用默认资源 |

### 返回值

`target` — 解析出的目标信息，包含主机、端口和正向代理标志。如果解析失败，返回的目标对象可能包含空字符串。

### 调用（向下）

- [[protocol/http/parser|parser]] — 访问 `proxy_request` 字段
- `analysis::parse()` — 内部解析主机端口字符串

### 被调用（向上）

- [[pipeline/pipeline|pipeline]] — HTTP 协议处理管道

### 知识域

HTTP 代理协议、URI 解析

---

## 函数: analysis::resolve()（字符串）

### 功能说明

解析 `"host:port"` 格式的字符串，提取主机和端口信息。用于解析 SOCKS5、TLS 等协议中的目标地址字段。支持基本格式、IPv4 地址、IPv6 地址（方括号语法）和省略端口的默认值。解析规则为查找最后一个冒号作为端口分隔符，处理 IPv6 地址的方括号语法，主机名转换为小写。

### 签名

```cpp
static auto resolve(std::string_view host_port, memory::resource_pointer mr = nullptr)
    -> target;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `host_port` | `std::string_view` | `"host:port"` 格式的字符串，可能包含 IPv6 地址 |
| `mr` | `memory::resource_pointer` | 内存资源指针，为空时使用默认资源 |

### 返回值

`target` — 解析出的目标信息。对于非 HTTP 协议，`positive` 标志通常为 `false`。格式错误返回空主机和默认端口 `"80"`。

### 调用（向下）

- `analysis::parse()` — 内部解析主机端口字符串

### 被调用（向上）

- [[pipeline/pipeline|pipeline]] — SOCKS5/Trojan/VLESS 协议处理
- [[protocol/socks5/stream|socks5::relay]] — SOCKS5 握手后解析目标
- [[protocol/trojan/relay|trojan::relay]] — Trojan 握手后解析目标

### 知识域

URI 解析、地址格式化

---

## 函数: analysis::detect_tls()

### 功能说明

在 TLS 握手完成后，探测内部承载的应用层协议类型。用于区分 HTTPS、Trojan over TLS、VLESS 等协议。

### 签名

```cpp
static auto detect_tls(std::string_view peek_data) -> protocol_type;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `peek_data` | `std::string_view` | TLS 握手后读取的数据，建议至少 60 字节 |

### 返回值

`protocol_type` — 检测到的内部协议类型。数据不足 60 字节时返回 `protocol_type::unknown`。

### 调用（向下）

- 内部字节模式匹配

### 被调用（向上）

- [[recognition/probe/probe|probe]] — TLS 内层协议探测
- [[pipeline/pipeline|pipeline]] — TLS 后协议分发

### 知识域

TLS 协议分析、流量指纹识别

---

## 函数: analysis::parse()（私有）

### 功能说明

将 `"host:port"` 格式的字符串解析为独立的主机和端口组件。该方法是 `resolve(std::string_view)` 的内部实现，处理 IPv6 地址等复杂情况。解析步骤包括处理 IPv6 地址的方括号语法、查找端口分隔符、提取主机部分、提取端口部分并验证有效性。解析失败时不抛出异常。

### 签名

```cpp
static auto parse(std::string_view src, memory::string &host, memory::string &port) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `src` | `std::string_view` | 源字符串，格式为 `"host:port"` 或 `"[ipv6]:port"` |
| `host` | `memory::string &` | 输出参数，存储解析出的主机名或 IP 地址 |
| `port` | `memory::string &` | 输出参数，存储解析出的端口号 |

### 返回值

无（`void`）

### 调用（向下）

- `memory::string` 操作（赋值、清空）

### 被调用（向上）

- `analysis::resolve(std::string_view)` — 字符串解析重载
- `analysis::resolve(const http::proxy_request &)` — HTTP 请求解析重载

### 知识域

字符串解析、IPv6 地址格式

## 相关页面

- [[protocol/http/parser|parser]] — HTTP 代理请求解析
- [[protocol/common/address|address]] — 共享地址类型
- [[pipeline/pipeline|pipeline]] — 协议处理管道
- [[outbound/proxy|proxy]] — 出站代理
