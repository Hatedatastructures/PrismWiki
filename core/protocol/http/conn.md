---
title: HTTP 代理中继器
layer: core
source:
  - I:/code/Prism/include/prism/protocol/http/conn.hpp
  - I:/code/Prism/src/prism/protocol/http/conn.cpp
tags: [protocol, http, conn]
---

# HTTP 代理中继器

HTTP 代理协议层的核心处理类，封装请求头读取、解析、Basic 认证和响应写入。管理 HTTP 代理请求的完整握手生命周期。

## 模块概述

`conn` **不继承** `transport::transmission`——它与 Trojan/VLESS/SS2022 的中继器有本质区别。HTTP 代理握手完成后，底层传输层通过 `release()` 被释放给 `tunnel()` 进行双向转发，`conn` 本身仅作为握手阶段的状态持有者。

**生命周期**：

```
make_conn 创建 → handshake（读取请求头 + 解析 + 认证）
  → send_ok / send_gateway_err / forward（响应客户端）
  → release（释放传输层给 tunnel）
  → conn 析构（account::lease 自动释放）
```

`conn` 持有的 `account::lease` 在析构时自动释放，确保账户连接计数正确。

## 核心函数

### handshake()

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, proxy_request>>;
```

执行 HTTP 代理握手的完整流程：

1. **读取请求头**：循环调用 `async_read_some` 直到在缓冲区中找到 `\r\n\r\n` 结束标记。缓冲区从 4096 字节起步，动态倍增，最大 65536 字节（`max_hdr_size`），防止慢速 OOM 攻击
2. **解析请求**：调用 `parse_req` 提取 method、target、host、authorization 等字段
3. **Basic 认证**（可选）：若配置了账户目录（`acct_dir_`），调用 `authenticate_proxy` 验证凭据。认证失败时自动发送 407/403 响应

### send_ok()

```cpp
auto send_ok() -> net::awaitable<fault::code>;
```

发送 `HTTP/1.1 200 Connection Established\r\n\r\n`，用于 CONNECT 方法成功建连后通知客户端隧道已建立。

### send_gateway_err()

```cpp
auto send_gateway_err() -> net::awaitable<fault::code>;
```

发送 `HTTP/1.1 502 Bad Gateway\r\nContent-Length: 0\r\n\r\n`，用于上游连接失败时通知客户端。

### forward()

```cpp
auto forward(const proxy_request &req, transport::shared_transmission outbound,
             std::pmr::memory_resource *mr) -> net::awaitable<void>;
```

转发普通 HTTP 请求到上游：

1. 调用 `build_fwd` 将绝对 URI（如 `http://example.com/path?q=1`）重写为相对路径（如 `/path?q=1`），构建新请求行
2. 写入新请求行到上游传输层
3. 写入请求行之后的剩余数据（headers + body）

### release()

```cpp
auto release() -> transport::shared_transmission;
```

释放底层传输层的所有权。握手完成后调用，将传输层交给 `tunnel()` 进行双向转发。此后 `conn` 不再持有传输层。

### read_hdr()

```cpp
auto read_hdr() -> net::awaitable<bool>;
```

私有方法，循环读取直到在缓冲区中找到 `\r\n\r\n`。缓冲区满时动态倍增，达到 `max_hdr_size`（65536 字节）上限后返回 false。

## 设计决策

### WHY: 不继承 transmission

- **Problem**：HTTP 代理在握手完成后，后续数据是明文直通的隧道流量，不需要任何协议层处理。
- **Choice**：`conn` 不继承 `transport::transmission`，握手完成后通过 `release()` 释放传输层，让 tunnel 直接操作原始传输层。
- **Consequence**：tunnel 层获得零开销的明文转发路径；但 `conn` 与传输层的生命周期绑定关系需要调用者正确管理（先 release 再析构）。

### WHY: 慢速攻击防御（max_hdr_size）

- **Problem**：攻击者可以缓慢发送 HTTP 头部数据，使服务器长时间占用缓冲区内存。
- **Choice**：缓冲区从 4096 字节起步倍增，上限 `max_hdr_size = 65536` 字节。超过上限时返回 false，触发连接关闭。
- **Consequence**：限制了单个 HTTP 代理连接的最大头部内存为 64KB；超大的合法请求头（罕见）会被拒绝。

### WHY: account::lease 自动释放

- **Problem**：连接计数必须在连接结束时正确递减，否则会导致账户配额泄漏。
- **Choice**：`conn` 持有 `account::lease` 成员，析构函数自动释放。无论连接正常结束还是异常退出，lease 都会正确归还。
- **Consequence**：调用者无需手动管理连接计数；但需确保 `conn` 的生命周期不短于 tunnel 阶段（通过 shared_ptr 保证）。

## 约束

| 约束 | 值 | 来源 |
|------|----|------|
| 初始缓冲区大小 | 4096 字节 | 构造函数初始化 |
| 最大头部大小 | 65536 字节 | `max_hdr_size` 常量 |
| 响应版本 | HTTP/1.1 | 硬编码常量 |
| 认证方案 | Basic | `authenticate_proxy` 实现 |

## 失败场景

### 1. 头部解析失败

客户端发送格式错误的 HTTP 请求（缺少 `\r\n`、请求行格式不正确）。`parse_req` 返回 `fault::code::parse_error`，`handshake()` 返回 `(io_error, {})` 或 `(parse_error, {})`。连接被直接关闭，不发送任何错误响应（因为无法确定客户端期望的响应格式）。

### 2. Basic 认证失败

客户端未提供 Proxy-Authorization 头或凭据无效。`authenticate_proxy` 返回 `authenticated=false` 及对应的错误响应（407 或 403），`handshake()` 自动发送错误响应后返回 `fault::code::auth_failed`。

### 3. 头部过大 / 慢速攻击

客户端发送超过 65536 字节的 HTTP 头部数据。`read_hdr()` 检测到 `buffer_.size() >= max_hdr_size`，返回 false，`handshake()` 返回 `(io_error, {})`。

### 4. 传输层读取错误

底层连接在握手过程中断开。`async_read_some` 返回错误码或 0 字节，`read_hdr()` 返回 false。

## 跨模块契约

| 上游模块 | 契约 |
|----------|------|
| [[core/protocol/http/process]] | 调用 `make_conn`、`handshake`、`send_ok`/`send_gateway_err`/`forward`、`release` |
| [[core/protocol/http/parser]] | 提供 `parse_req`、`authenticate_proxy`、`build_fwd` |
| [[core/account/directory]] | 提供账户目录，用于 Basic 认证 |
| [[core/account/entry]] | 提供 `account::lease`，conn 析构时自动释放 |
| [[core/transport/transmission]] | 入站传输层接口，`release()` 后交给 tunnel |

## 变更敏感度

| 修改点 | 影响范围 |
|--------|----------|
| `max_hdr_size` | 所有 HTTP 代理连接的最大头部限制 |
| 响应字符串（resp200/resp502） | 客户端对隧道建立/失败的判定逻辑 |
| `read_hdr` 的 `\r\n\r\n` 检测 | HTTP 头部边界识别，修改会导致解析失败 |
| `buffer_` 初始大小 | 内存使用和读取次数的权衡 |
| `release()` 调用时机 | 必须在最后一次写入操作之后调用，否则传输层已被释放 |

## 相关文档

- [[core/protocol/http/process]] — 入口函数，编排 handshake → send_ok/forward → release → tunnel
- [[core/protocol/http/parser]] — 请求解析、认证、URI 重写
- [[core/protocol/http/relay]] — 旧版 relay 文档
- [[core/account/directory]] — 账户目录
- [[core/transport/transmission]] — 传输层接口
