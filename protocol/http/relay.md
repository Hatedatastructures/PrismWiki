---
title: "relay — HTTP 代理中继器"
source: "include/prism/protocol/http/relay.hpp"
module: "protocol"
type: api
tags: [protocol, http, relay, 代理, 握手, Basic认证, CONNECT]
related:
  - "[[protocol/http/parser|parser]]"
  - "[[protocol/analysis|analysis]]"
  - "[[agent/account/directory|directory]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# relay.hpp

> 源码: `include/prism/protocol/http/relay.hpp`
> 实现: `src/prism/protocol/http/relay.cpp`
> 模块: [[protocol|protocol]] > http

## 概述

HTTP 代理中继器。HTTP 代理协议层的核心处理类，封装请求头读取、解析、认证和响应写入。设计参照 `socks5::relay` 模式，将协议级逻辑从 pipeline 编排层分离。relay 持有入站传输层的所有权，完成握手后通过 `release()` 释放传输层供隧道使用。

生命周期：`make_relay` 创建 -> `handshake` 完成协议协商 -> `write_connect_success`/`forward` 执行响应 -> `release` 释放传输层 -> 析构

> **注意**: relay 不继承 `transmission`，因为它不是传输装饰器 -- 握手完成后传输层被释放给 `tunnel()`，relay 本身仅作为握手阶段的状态持有者。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/http/parser|parser]] | HTTP 请求解析 |
| 依赖 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[agent/account/directory|directory]] | 账户认证 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[pipeline/protocols/http|pipeline]] | HTTP 协议处理管道 |

## 命名空间

`psm::protocol::http`

---

## 类: relay

> 源码: `include/prism/protocol/http/relay.hpp:35`

### 概述

HTTP 代理中继器。管理 HTTP 代理请求的完整握手流程。relay 持有的 `account::lease` 在 relay 析构时自动释放，确保连接计数正确。

### 构造函数

```cpp
explicit relay(transport::shared_transmission transport,
               agent::account::directory *account_directory = nullptr);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `transport::shared_transmission` | 入站传输层（通常经 preview 包装） |
| `account_directory` | `agent::account::directory *` | 账户目录指针，为空时跳过认证 |

---

### 函数: handshake()

#### 功能说明

执行 HTTP 代理握手。读取完整 HTTP 请求头、解析请求行和头字段、若配置了账户目录则执行 Basic 认证。认证失败时自动发送 407/403 响应。

#### 签名

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, proxy_request>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, proxy_request>>` — 错误码和解析后的代理请求。

#### 调用（向下）

- `read_until_header_end()` — 循环读取直到找到 `\r\n\r\n`
- [[protocol/http/parser|parser]] `parse_proxy_request()` — 解析请求头
- [[protocol/http/parser|parser]] `authenticate_proxy_request()` — Basic 认证

#### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] 协议处理管道

#### 知识域

HTTP 代理握手、Basic 认证

---

### 函数: write_connect_success()

#### 功能说明

发送 `200 Connection Established` 响应。用于 CONNECT 方法成功建连后通知客户端隧道已建立。

#### 签名

```cpp
auto write_connect_success() -> net::awaitable<fault::code>;
```

#### 参数

无

#### 返回值

`net::awaitable<fault::code>` — 写入结果。

#### 调用（向下）

- `write_bytes()` — 写入响应字符串

#### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] CONNECT 成功后

#### 知识域

HTTP CONNECT 响应

---

### 函数: write_bad_gateway()

#### 功能说明

发送 `502 Bad Gateway` 响应。用于上游连接失败时通知客户端。

#### 签名

```cpp
auto write_bad_gateway() -> net::awaitable<fault::code>;
```

#### 参数

无

#### 返回值

`net::awaitable<fault::code>` — 写入结果。

#### 调用（向下）

- `write_bytes()` — 写入响应字符串

#### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] 上游连接失败时

#### 知识域

HTTP 错误响应

---

### 函数: forward()

#### 功能说明

将普通 HTTP 请求转发到上游。将绝对 URI 重写为相对路径，构建新请求行写入上游，随后写入请求行之后的剩余数据（headers + body）。

#### 签名

```cpp
auto forward(const proxy_request &req, transport::shared_transmission outbound,
             std::pmr::memory_resource *mr) -> net::awaitable<void>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `req` | `const proxy_request &` | 已解析的代理请求 |
| `outbound` | `transport::shared_transmission` | 上游传输层 |
| `mr` | `std::pmr::memory_resource *` | PMR 内存资源 |

#### 返回值

`net::awaitable<void>`

#### 调用（向下）

- [[protocol/http/parser|parser]] `build_forward_request_line()` — 构建请求行
- `transport_->async_write_some()` — 写入上游
- `outbound->async_write_some()` — 写入上游

#### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] 普通 HTTP 请求转发

#### 知识域

HTTP 正向代理转发、请求行重写

---

### 函数: release()

#### 功能说明

释放底层传输层。握手完成后调用，将传输层交给 `tunnel()` 进行双向转发。

#### 签名

```cpp
auto release() -> transport::shared_transmission;
```

#### 参数

无

#### 返回值

`transport::shared_transmission` — 入站传输层的共享指针。

#### 调用（向下）

`std::move(transport_)`

#### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] 握手完成后转移所有权

#### 知识域

所有权转移

---

## 工厂函数: make_relay()

### 功能说明

工厂函数，创建 HTTP 代理中继器实例。封装 `std::make_shared` 调用，返回 `shared_ptr<relay>`。

### 签名

```cpp
inline auto make_relay(transport::shared_transmission transport,
                       agent::account::directory *account_directory = nullptr)
    -> std::shared_ptr<relay>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `transport::shared_transmission` | 入站传输层（通常经 preview 包装） |
| `account_directory` | `agent::account::directory *` | 账户目录指针，为空时跳过认证 |

### 返回值

`std::shared_ptr<relay>` — HTTP 代理中继器共享指针。

### 调用（向下）

- `std::make_shared<relay>(transport, account_directory)`

### 被调用（向上）

- [[pipeline/protocols/http|pipeline]] 创建 HTTP 中继器

### 知识域

- [[protocol/http/relay|relay]] HTTP 代理中继
- [[pipeline/protocols/http|pipeline]] 协议处理管道

## 相关页面

- [[protocol/http/parser|parser]] — HTTP 请求解析器
- [[protocol/analysis|analysis]] — 协议分析与识别
- [[agent/account/directory|directory]] — 账户目录
