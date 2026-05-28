---
title: target.hpp — 目标地址解析
created: 2026-05-17
updated: 2026-05-27
layer: core
module: recognition
source: I:/code/Prism/include/prism/recognition/target.hpp
tags: [recognition, target, address, resolve, parse, host-port, uri-extraction]
---

# target.hpp — 目标地址解析

提供从 HTTP 请求或 host:port 字符串中解析目标地址的工具函数。该模块从 protocol/analysis 迁移而来，职责下沉到 recognition 模块。所有函数都是纯计算操作，无网络 I/O，线程安全。

## 设计决策

### 为什么 resolve() 使用 string_view 输入 + PMR 输出？

协议解析产生的地址信息是 `string_view`（零拷贝视图），但返回的 `protocol::target` 需要持有独立字符串。使用 `memory::resource_pointer` 参数让调用方控制输出内存来源：热路径传帧竞技场，冷路径传 nullptr 用默认分配器。

**后果**: 调用方必须确保输入 `string_view` 在返回值使用期间有效。

## 约束

### HTTP 请求必须包含有效数据

**类型**: 状态前置

**规则**: `resolve(req)` 的 `req` 必须包含已解析的 HTTP 请求数据（请求行和头部字段）。

**违反后果**: 返回空目标对象，后续连接失败。

**源码依据**: `target.hpp:36`

### IPv6 地址必须用方括号

**类型**: 数据格式

**规则**: `resolve(string_view)` 处理 IPv6 地址时，地址部分必须用 `[...]` 括起（如 `[2001:db8::1]:443`）。

**违反后果**: 冒号解析歧义，端口号被错误截取。

**源码依据**: `target.hpp:55`

## 使用场景

| 场景 | 调用 | 说明 |
|------|------|------|
| HTTP CONNECT 代理 | `resolve(req)` | 从 CONNECT 请求提取目标 |
| HTTP 正向代理绝对 URI | `resolve(req)` | 从绝对 URI 提取主机端口 |
| SOCKS5 目标地址 | `resolve(host_port)` | 从 host:port 字符串解析 |
| Trojan 目标地址 | `resolve(host_port)` | 同上 |

## 命名空间

`psm::recognition`

## 核心函数

### resolve (HTTP 请求)

从 HTTP 请求中解析目标地址。

```cpp
auto resolve(const protocol::http::proxy_request &req, memory::resource_pointer mr = nullptr)
    -> protocol::target;
```

**解析策略**: 检查绝对 URI -> 提取主机和端口 -> 否则从 Host 头字段提取

### resolve (字符串)

从字符串解析目标地址。

```cpp
auto resolve(std::string_view host_port, memory::resource_pointer mr = nullptr)
    -> protocol::target;
```

**支持格式**: `example.com:8080`, `192.168.1.1:80`, `[2001:db8::1]:443`, `example.com`

### parse

解析主机端口字符串为独立组件。

```cpp
void parse(std::string_view src, memory::string &host, memory::string &port);
```

## 调用链

- [[core/pipeline/protocols/http|http handler]] -> `recognition::resolve(req)`
- [[core/pipeline/protocols/socks5|socks5 handler]] -> `recognition::resolve(host_port)`
- [[core/protocol/common/target|protocol::target]] - 返回类型

## 依赖

- [[core/protocol/http/parser|http::proxy_request]] - HTTP 请求结构
- [[core/protocol/common/target|protocol::target]] - 目标地址结构
- [[core/memory/container|memory::string]] - PMR 字符串
