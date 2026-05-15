---
title: "parser — HTTP 代理请求轻量解析器"
source: "include/prism/protocol/http/parser.hpp"
module: "protocol"
type: api
tags: [protocol, http, parser, 代理, 解析, Basic认证, 零分配]
related:
  - "[[protocol/http/relay|relay]]"
  - "[[protocol/analysis|analysis]]"
  - "[[agent/account/directory|directory]]"
created: 2026-05-15
updated: 2026-05-15
---

# parser.hpp

> 源码: `include/prism/protocol/http/parser.hpp`
> 实现: `src/prism/protocol/http/parser.cpp`
> 模块: [[protocol|protocol]] > http

## 概述

HTTP 代理请求轻量解析器。专为代理场景设计的 HTTP 请求头解析模块，仅提取代理转发所需的最少信息：请求方法、目标地址、Host 头字段和 Proxy-Authorization 头字段。不构建完整的 HTTP 消息对象，直接在原始字节上操作。

设计目标：
- 零堆分配：所有结果以 `string_view` 指向原始缓冲区
- 最小化解析：仅提取代理决策所需字段
- 内存高效：解析过程不分配任何内存

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[fault|fault]] | 错误码 |
| 依赖 | [[agent/account/entry|entry]] | 账户租约类型 |
| 依赖 | [[memory|memory]] | `memory::string` |
| 被依赖 | [[protocol/analysis|analysis]] | `analysis::resolve()` 使用 `proxy_request` |
| 被依赖 | [[protocol/http/relay|relay]] | HTTP 中继器使用解析结果 |

## 命名空间

`psm::protocol::http`

---

## 结构体: proxy_request

### 功能说明

HTTP 代理请求解析结果。存储从原始 HTTP 请求头中提取的代理转发所需信息。所有字符串字段以 `string_view` 形式指向原始缓冲区，不持有数据所有权。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `method` | `std::string_view` | 请求方法，如 `"CONNECT"`、`"GET"`、`"POST"` |
| `target` | `std::string_view` | 请求目标，绝对 URI 或 `host:port`（CONNECT） |
| `host` | `std::string_view` | Host 头字段值 |
| `authorization` | `std::string_view` | Proxy-Authorization 头字段值 |
| `version` | `std::string_view` | HTTP 版本字符串，如 `"HTTP/1.1"` |
| `req_line_end` | `std::size_t` | 请求行末尾 `\r\n` 之后的偏移量 |
| `header_end` | `std::size_t` | 完整头部 `\r\n\r\n` 之后的偏移量 |

> **注意**: 字段生命周期与原始缓冲区绑定，原始缓冲区必须在使用期间保持有效。

---

## 函数: parse_proxy_request()

### 功能说明

从原始字节中提取请求方法、目标地址、Host 和 Proxy-Authorization 头字段。解析过程不分配内存，所有 `string_view` 直接指向输入缓冲区。请求行格式为 `"METHOD TARGET HTTP/version\r\n"`，头字段以 `"\r\n\r\n"` 结束。

### 签名

```cpp
[[nodiscard]] auto parse_proxy_request(std::string_view raw_data, proxy_request &out)
    -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `raw_data` | `std::string_view` | 包含完整 HTTP 请求头的原始数据（须包含 `\r\n\r\n`） |
| `out` | `proxy_request &` | 接收解析结果的代理请求结构体 |

### 返回值

`fault::code` — 解析状态码。

### 调用（向下）

- 字符串扫描（`\r\n` 分隔）

### 被调用（向上）

- [[protocol/http/relay|relay]] `handshake()` 解析请求头

### 知识域

HTTP/1.1 请求格式、代理协议

---

## 函数: extract_relative_path()

### 功能说明

将代理场景中的绝对 URI（如 `"http://example.com/path?q=1"`）转换为源站所需的相对路径（如 `"/path?q=1"`）。若 target 不是绝对 URI 则原样返回。若绝对 URI 不含路径，返回 `"/"`。

### 签名

```cpp
[[nodiscard]] auto extract_relative_path(std::string_view target) -> std::string_view;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `target` | `std::string_view` | 请求目标，可能是绝对 URI 或相对路径 |

### 返回值

`std::string_view` — 相对路径部分。

### 调用（向下）

- 字符串前缀检查

### 被调用（向上）

- `build_forward_request_line()` 构建转发请求行

### 知识域

HTTP 正向代理、URI 格式

---

## 结构体: auth_result

### 功能说明

HTTP 代理认证结果。封装 Basic 代理认证的验证结果。认证成功时持有连接租约，失败时包含待发送的错误响应。

### 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `authenticated` | `bool` | 认证是否通过 |
| `error_response` | `std::string_view` | 失败时待发送的 HTTP 错误响应（指向静态常量） |
| `lease` | `agent::account::lease` | 认证通过时获取的连接租约 |

---

## 函数: authenticate_proxy_request()

### 功能说明

验证 HTTP 代理 Basic 认证。解码 Base64 凭据，提取密码并计算 SHA224 哈希后查询账户目录。认证流程：验证 Basic 方案前缀 -> Base64 解码 -> 提取密码 -> SHA224 哈希 -> 查询账户目录获取租约。任一步骤失败均返回对应的错误响应。

### 签名

```cpp
[[nodiscard]] auto authenticate_proxy_request(
    std::string_view authorization,
    agent::account::directory &directory) -> auth_result;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `authorization` | `std::string_view` | Proxy-Authorization 头字段值 |
| `directory` | `agent::account::directory &` | 账户目录引用 |

### 返回值

`auth_result` — 认证结果，包含是否通过、错误响应和连接租约。

### 调用（向下）

- Base64 解码
- [[crypto/sha224|sha224]] — 密码哈希
- [[agent/account/directory|directory]] `try_acquire()` — 查询账户

### 被调用（向上）

- [[protocol/http/relay|relay]] `handshake()` 执行认证

### 知识域

HTTP Basic 认证、Base64 编码、SHA224 哈希

---

## 函数: build_forward_request_line()

### 功能说明

构建正向代理转发请求行。将绝对 URI 转换为相对路径，拼接方法、路径和版本号构成新的请求行。仅用于普通 HTTP 请求转发，CONNECT 方法无需调用此函数。

### 签名

```cpp
[[nodiscard]] auto build_forward_request_line(const proxy_request &req, std::pmr::memory_resource *mr)
    -> memory::string;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `req` | `const proxy_request &` | 已解析的代理请求 |
| `mr` | `std::pmr::memory_resource *` | PMR 内存资源指针 |

### 返回值

`memory::string` — 重写后的请求行，格式为 `"METHOD relative-path HTTP/version\r\n"`。

### 调用（向下）

- `extract_relative_path()` — 提取相对路径
- `memory::string` 拼接

### 被调用（向上）

- [[protocol/http/relay|relay]] `forward()` 转发请求

### 知识域

HTTP 正向代理、请求行格式

## 相关页面

- [[protocol/http/relay|relay]] — HTTP 代理中继器
- [[protocol/analysis|analysis]] — 协议分析与识别
- [[agent/account/directory|directory]] — 账户目录
