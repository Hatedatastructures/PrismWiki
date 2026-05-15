---
title: "config.hpp — DNS 解析器配置"
source: "include/prism/resolve/dns/config.hpp"
module: "resolve"
type: api
tags: [resolve, dns, config, DNS配置]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[resolve/dns/dns|DNS 解析器]]"
  - "[[resolve/dns/upstream|上游查询]]"
  - "[[resolve/router|路由器]]"
  - "[[memory/container|PMR 容器]]"
---

# config.hpp

> 源码: `include/prism/resolve/dns/config.hpp`
> 模块: [[resolve|Resolve]] / dns

## 概述

定义 DNS 解析器的全部配置类型，包括上游服务器、解析模式、域名规则和 IP 过滤等。所有容器类型均使用 PMR 多态内存资源分配器。该文件为 header-only 实现，通过 `glaze` 库实现 JSON 序列化/反序列化。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver 接口使用配置 |
| 被依赖 | [[resolve/dns/upstream|upstream]] | 上游客户端使用配置 |
| 被依赖 | [[resolve/router|router]] | 路由器传递 DNS 配置 |

## 命名空间

`psm::resolve::dns`

---

## 枚举: dns_protocol

```cpp
enum class dns_protocol : std::uint8_t { udp, tcp, tls, https };
```

### 概述
DNS 上游服务器协议类型。

| 值 | 说明 |
|----|------|
| `udp` | 纯 UDP 协议 |
| `tcp` | TCP 协议 |
| `tls` | TLS 协议 (DoT)，默认端口 853 |
| `https` | HTTPS 协议 (DoH)，默认端口 443 |

---

## 结构体: dns_remote

### 概述
DNS 上游服务器配置，描述一个上游 DNS 服务器的连接参数。

### 成员变量
| 类型 | 名称 | 默认值 | 说明 |
|------|------|--------|------|
| `memory::string` | `address` | | 原始地址字符串 |
| `dns_protocol` | `protocol` | `udp` | 通信协议 |
| `memory::string` | `hostname` | | TLS SNI / HTTP Host |
| `std::uint16_t` | `port` | 53 | 服务端口 |
| `std::uint32_t` | `timeout_ms` | 5000 | 超时时间（毫秒） |
| `memory::string` | `http_path` | "/dns-query" | DoH 路径 |
| `bool` | `no_check_certificate` | false | 跳过 TLS 证书验证 |

### 地址解析规则
- 无 scheme 前缀：默认 UDP 协议
- `tcp://`：TCP 协议
- `tls://`：TLS (DoT)，端口 853
- `https://`：HTTPS (DoH)，端口 443

---

## 枚举: resolve_mode

```cpp
enum class resolve_mode : std::uint8_t { fastest, first, fallback };
```

### 概述
DNS 解析查询模式。

| 值 | 说明 |
|----|------|
| `fastest` | 并发查询所有上游，选择 RTT 最低的成功响应 |
| `first` | 并发查询所有上游，返回第一个成功响应 |
| `fallback` | 按顺序尝试上游，前一个失败后尝试下一个 |

---

## 结构体: address_rule

### 概述
DNS 地址映射规则，将特定域名映射到预定义的 IP 地址列表。支持通配符匹配。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `domain` | 匹配域名，支持通配符 `*.xxx.com` |
| `memory::vector<net::ip::address>` | `addresses` | 映射的地址列表 |
| `bool` | `negative` | 否定应答（NXDOMAIN） |

---

## 结构体: cname_rule

### 概述
DNS CNAME 重定向规则。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `domain` | 源域名 |
| `memory::string` | `target` | CNAME 目标域名 |

---

## 结构体: config

### 概述
DNS 解析器主配置，聚合所有配置项。

### 成员变量
| 类型 | 名称 | 默认值 | 说明 |
|------|------|--------|------|
| `memory::vector<dns_remote>` | `servers` | | 上游服务器列表 |
| `resolve_mode` | `mode` | `fastest` | 解析查询模式 |
| `std::uint32_t` | `timeout_ms` | 5000 | 全局超时（毫秒） |
| `bool` | `cache_enabled` | true | 是否启用 DNS 缓存 |
| `std::size_t` | `cache_size` | 10000 | 缓存最大条目数 |
| `std::chrono::seconds` | `cache_ttl` | 120 | 缓存默认 TTL |
| `bool` | `serve_stale` | true | 过期后是否仍提供缓存数据 |
| `std::chrono::seconds` | `negative_ttl` | 300 | 负缓存 TTL |
| `std::uint32_t` | `ttl_min` | 60 | 最小 TTL（秒） |
| `std::uint32_t` | `ttl_max` | 86400 | 最大 TTL（秒） |
| `memory::vector<address_rule>` | `address_rules` | | 地址映射规则列表 |
| `memory::vector<cname_rule>` | `cname_rules` | | CNAME 重定向规则列表 |
| `bool` | `disable_ipv6` | false | 是否禁用 IPv6 |
| `memory::vector<net::ip::network_v4>` | `blacklist_v4` | | IPv4 黑名单 |
| `memory::vector<net::ip::network_v6>` | `blacklist_v6` | | IPv6 黑名单 |

---

## 知识域

- [[ref/protocol/dns-over-udp|DNS 解析]]
- [[loader/load|配置系统]]
