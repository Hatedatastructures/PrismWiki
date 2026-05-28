---
layer: core
source: I:/code/Prism/include/prism/protocol/http/relay.hpp
title: HTTP 代理中继器
tags: [protocol, http, relay, connect, forward]
---

# HTTP 代理中继器

HTTP 代理协议层的核心处理类，封装请求头读取、解析、认证和响应写入。

## 源码位置

`I:/code/Prism/include/prism/protocol/http/relay.hpp`

## 类定义

```cpp
class relay
{
public:
    explicit relay(transport::shared_transmission transport,
                   agent::account::directory *account_directory = nullptr);

    auto handshake() -> net::awaitable<std::pair<fault::code, proxy_request>>;
    auto write_connect_success() -> net::awaitable<fault::code>;
    auto write_bad_gateway() -> net::awaitable<fault::code>;
    auto forward(const proxy_request &req,
                 transport::shared_transmission outbound,
                 std::pmr::memory_resource *mr) -> net::awaitable<void>;
    auto release() -> transport::shared_transmission;
};
```

## 方法说明

### handshake

执行 HTTP 代理握手。

**流程**：
1. 读取完整 HTTP 请求头
2. 解析请求行和头字段
3. 若配置账户目录，执行 Basic 认证
4. 认证失败自动发送 407/403 响应

### write_connect_success

发送 `200 Connection Established` 响应，用于 CONNECT 方法成功建连后通知客户端。

### write_bad_gateway

发送 `502 Bad Gateway` 响应，用于上游连接失败。

### forward

转发普通 HTTP 请求到上游。

**操作**：
- 将绝对 URI 重写为相对路径
- 构建新请求行写入上游
- 写入剩余数据（headers + body）

### release

释放底层传输层，供 tunnel() 进行双向转发。

## 生命周期

```
make_relay 创建 -> handshake 完成协议协商 ->
write_connect_success/forward 执行响应 -> release 释放传输层 -> 析构
```

## 设计特点

- **不继承 transmission**：握手完成后传输层被释放给 tunnel
- **账户租约管理**：relay 析构时自动释放 `account::lease`
- **缓冲区管理**：内部维护 `std::vector<char> buffer_` 用于读取

## 工厂函数

```cpp
auto make_relay(transport::shared_transmission transport,
                agent::account::directory *account_directory = nullptr)
    -> std::shared_ptr<relay>;
```

## 调用链

```
agent/worker/worker -> protocol/http::make_relay -> relay
relay::handshake -> protocol/http::parse_proxy_request -> proxy_request
relay::handshake -> protocol/http::authenticate_proxy_request -> auth_result
relay::forward -> protocol/http::build_forward_request_line -> memory::string
relay::release -> transport::shared_transmission
```

## 依赖

- [[core/protocol/http/parser]] - 请求解析器
- [[core/transport/transmission]] - 传输层接口
- [[core/account/directory|directory]] - 账户目录
- [[core/account/entry|entry]] - 连接租约