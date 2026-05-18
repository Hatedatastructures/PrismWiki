---
layer: core
source: "I:/code/Prism/include/prism/protocol/analysis.hpp"
---

# 协议分析与识别

> 源码位置: `I:/code/Prism/include/prism/protocol/analysis.hpp`

## 概述

协议分析与识别模块，提供协议探测、目标地址解析等静态辅助方法，是代理系统协议栈的核心组件。负责解析客户端请求、识别应用层协议类型、提取目标地址信息，为路由决策提供关键输入数据。

## 命名空间

```cpp
namespace psm::protocol
```

## 协议类型枚举

### protocol_type

标识当前连接所使用的应用层协议类型。

```cpp
enum class protocol_type
{
    unknown,       // 未知协议
    http,          // HTTP 协议
    socks5,        // SOCKS5 协议
    trojan,        // Trojan 协议
    vless,         // VLESS 协议
    shadowsocks,   // Shadowsocks 2022 协议
    tls            // TLS 协议
};
```

### to_string_view

将协议类型转换为字符串视图。

```cpp
inline auto to_string_view(const protocol_type type) -> std::string_view;
```

**返回值**:
| 类型 | 字符串 |
|------|--------|
| `unknown` | "unknown" |
| `http` | "http" |
| `socks5` | "socks5" |
| `trojan` | "trojan" |
| `vless` | "vless" |
| `shadowsocks` | "shadowsocks" |
| `tls` | "tls" |

## 协议分析器

### analysis

协议分析器结构体，提供静态方法。

```cpp
struct analysis
```

**设计特性**:
- 无状态：所有方法都是静态的
- 线程安全：纯函数操作，可安全并发调用
- 内存高效：使用 `memory::string` 和 `std::string_view`
- 错误容忍：解析失败时返回合理默认值，不抛出异常

## 目标地址结构

### target

目标地址信息结构，封装解析出的目标主机、端口以及是否需要正向代理。

```cpp
struct target
{
    explicit target(memory::resource_pointer mr = memory::current_resource())
        : host(mr), port(mr)
    {
        port.assign("80");
    }

    memory::string host;    // 目标主机名或 IP 地址
    memory::string port;    // 目标端口号，字符串形式
    bool positive{false};   // 是否为正向代理请求
};
```

**路由语义**:
- `positive` = `true`: 客户端请求使用正向代理
- `positive` = `false`: 普通请求或反向代理请求

**默认值**: 端口默认为 "80"（HTTP 默认端口）

## 核心方法

### resolve (HTTP 请求)

从 HTTP 请求中解析目标地址。

```cpp
static auto resolve(const http::proxy_request &req, memory::resource_pointer mr = nullptr)
    -> target;
```

**解析策略**:
1. 检查请求行是否包含绝对 URI
2. 如有绝对 URI，从中提取主机和端口
3. 否则从 Host 头字段提取
4. Host 头缺少端口时使用协议默认端口

**支持**:
- HTTP 代理请求的 CONNECT 方法
- 带端口的 Host 头格式
- 自动识别是否为正向代理请求

### resolve (字符串)

从字符串解析目标地址。

```cpp
static auto resolve(std::string_view host_port, memory::resource_pointer mr = nullptr)
    -> target;
```

**支持格式**:
- `example.com:8080` - 基本格式
- `192.168.1.1:80` - IPv4 地址
- `[2001:db8::1]:443` - IPv6 地址
- `example.com` - 省略端口（使用默认值 "80"）

**解析规则**:
- 查找最后一个冒号作为端口分隔符
- 处理 IPv6 地址的方括号语法
- 验证端口号有效性
- 主机名转换为小写

### detect_tls

探测 TLS 内部协议类型。

```cpp
static auto detect_tls(std::string_view peek_data)
    -> protocol_type;
```

**用途**: 在 TLS 握手完成后，探测内部承载的应用层协议类型（HTTPS、Trojan over TLS、VLESS 等）

**建议**: peek_data 至少 60 字节，数据不足时返回 `protocol_type::unknown`

## 调用链

- [[core/protocol/http/parser|HTTP Parser]] - HTTP 请求解析
- [[core/protocol/trojan/relay|Trojan Relay]] - Trojan 协议处理
- [[core/protocol/vless/relay|VLESS Relay]] - VLESS 协议处理
- [[core/protocol/shadowsocks/relay|SS2022 Relay]] - SS2022 协议处理
- [[core/memory/container|Memory Container]] - memory::string 定义
- [[core/agent/resolve/router|Router]] - 路由决策使用目标信息
