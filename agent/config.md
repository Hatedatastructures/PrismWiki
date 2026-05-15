---
title: "config.hpp — Agent 运行时配置类型定义"
source: "include/prism/agent/config.hpp"
module: "agent"
type: api
tags: [agent, config, 配置]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[memory/container|PMR 容器]]"
  - "[[agent/context|运行时上下文]]"
  - "[[stealth/reality/config|Reality 配置]]"
  - "[[stealth/shadowtls/config|ShadowTLS 配置]]"
  - "[[stealth/restls/config|Restls 配置]]"
  - "[[stealth/anytls/config|AnyTLS 配置]]"
  - "[[stealth/trusttunnel/config|TrustTunnel 配置]]"
  - "[[protocol/socks5/config|SOCKS5 配置]]"
  - "[[protocol/trojan/config|Trojan 配置]]"
  - "[[protocol/vless/config|VLESS 配置]]"
  - "[[protocol/shadowsocks/config|SS2022 配置]]"
---

# config.hpp

> 源码: `include/prism/agent/config.hpp`
> 模块: [[agent|Agent]]

## 概述

声明代理服务的配置模型，涵盖工作线程启动、账户状态管理、路由策略、监听端点、TLS 证书以及协议管道行为等核心配置项。所有配置结构均采用 PMR 内存分配器，支持高性能内存管理。配置结构体通过 `glaze` 库实现 JSON 序列化/反序列化。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器类型 (memory::string, memory::vector, memory::map) |
| 依赖 | [[protocol/socks5/config|socks5::config]] | SOCKS5 协议配置 |
| 依赖 | [[protocol/trojan/config|trojan::config]] | Trojan 协议配置 |
| 依赖 | [[protocol/vless/config|vless::config]] | VLESS 协议配置 |
| 依赖 | [[protocol/shadowsocks/config|shadowsocks::config]] | SS2022 协议配置 |
| 依赖 | [[stealth/reality/config|reality::config]] | Reality 伪装配置 |
| 依赖 | [[stealth/shadowtls/config|shadowtls::config]] | ShadowTLS 伪装配置 |
| 依赖 | [[stealth/restls/config|restls::config]] | Restls 伪装配置 |
| 依赖 | [[stealth/anytls/config|anytls::config]] | AnyTLS 伪装配置 |
| 依赖 | [[stealth/trusttunnel/config|trusttunnel::config]] | TrustTunnel 伪装配置 |
| 被依赖 | [[agent/context|context]] | 会话上下文引用配置 |

## 命名空间

`psm::agent`、`psm::agent::protocol`、`psm::agent::stealth`

---

## 结构体: endpoint

### 概述
网络端点配置，表示一个包含主机名和端口号的网络地址。用于定义监听地址、正向代理目标、反向代理后端等场景。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `host` | 主机名，可以是域名或 IP 地址 |
| `std::uint16_t` | `port` | 端口号，范围 1-65535，0 表示未设置 |

---

## 结构体: limit

### 概述
连接限制配置，控制代理服务的并发连接数和黑名单策略。并发限制作用于全局所有工作线程。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `bool` | `blacklist` | 是否启用黑名单，默认 true |

---

## 结构体: certificate

### 概述
SSL/TLS 证书配置，包含 TLS 加密所需的私钥和证书文件路径。文件格式要求为 PEM。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `key` | 私钥文件路径，PEM 格式 |
| `memory::string` | `cert` | 证书文件路径，PEM 格式 |

---

## 结构体: authentication

### 概述
身份认证配置，管理客户端身份验证的凭据和用户限制。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::vector<user>` | `users` | 统一用户列表 |

### 内部结构体: authentication::user

统一用户配置，可同时配置密码和 UUID 两种认证方式。`password` 用于 Trojan/HTTP/SOCKS5 协议，启动时自动转换为 SHA224 哈希注册到 `account::directory`。`uuid` 用于 VLESS 协议，直接注册到 `account::directory`。两种凭证共享同一个 `entry`，从而共享连接数配额。

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `password` | 密码认证，用于 Trojan/HTTP/SOCKS5 |
| `memory::string` | `uuid` | VLESS UUID 字符串 |
| `std::uint32_t` | `max_connections` | 最大并发连接数，0 表示无限制 |

---

## 结构体: buffer

### 概述
缓冲区配置，控制数据传输时的缓冲区大小。默认值 256KB 用于抵消高延迟网络的影响。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint32_t` | `size` | 传输缓冲区大小（字节），默认 262144 |

---

## 命名空间: protocol

### 结构体: protocol::config

聚合所有代理协议的配置项。

| 类型 | 名称 | 说明 |
|------|------|------|
| `psm::protocol::socks5::config` | `socks5` | SOCKS5 协议配置 |
| `psm::protocol::trojan::config` | `trojan` | Trojan 协议配置 |
| `psm::protocol::vless::config` | `vless` | VLESS 协议配置 |
| `psm::protocol::shadowsocks::config` | `shadowsocks` | SS2022 协议配置 |

---

## 命名空间: stealth

### 结构体: stealth::config

聚合所有 TLS 伪装方案的配置项。

| 类型 | 名称 | 说明 |
|------|------|------|
| `psm::stealth::reality::config` | `reality` | Reality 伪装配置 |
| `psm::stealth::shadowtls::config` | `shadowtls` | ShadowTLS 伪装配置 |
| `psm::stealth::restls::config` | `restls` | Restls 伪装配置 |
| `psm::stealth::anytls::config` | `anytls` | AnyTLS 伪装配置 |
| `psm::stealth::trusttunnel::config` | `trusttunnel` | TrustTunnel 伪装配置 |

---

## 结构体: config

### 概述
代理服务核心配置，仅包含 agent 模块专属的配置项。其他模块配置（pool、buffer、protocol、mux、dns、stealth）已移至顶层 `psm::config`。

### 设计意图
将 agent 专属配置与全局配置分离，避免循环依赖。配置对象应在服务启动前完成初始化，运行时修改可能不会立即生效。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `limit` | `limits` | 连接限制配置 |
| `endpoint` | `positive` | 正向代理端点 |
| `endpoint` | `addressable` | 监听端点 |
| `certificate` | `cert` | SSL/TLS 证书配置 |
| `authentication` | `auth` | 身份认证配置 |
| `memory::string` | `camouflage` | 伪装路径，用于抗探测 |
| `memory::map<memory::string, endpoint>` | `reverse_map` | 反向代理路由表 |

---

## 知识域

- [[loader/load|配置系统]]
- [[ref/memory/pmr|PMR 内存管理]]
