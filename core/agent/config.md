---
title: "config.hpp — Agent 配置类型定义（Core 层）"
layer: core
source: "I:/code/Prism/include/prism/instance/config.hpp"
created: 2026-05-17
tags: [agent, config, endpoint, authentication, certificate, PMR, glaze]
---

# config.hpp — Agent 配置类型定义

> 源码: `include/prism/instance/config.hpp`
> 模块: [[overview|Agent]]

## 文件结构

| 类型 | 文件路径 |
|------|----------|
| 头文件 | `include/prism/instance/config.hpp` |
| 源文件 | header-only |

## 模块职责

声明 Agent 运行时配置类型，涵盖：

- 网络端点定义（监听、正向代理、反向代理）
- 连接限制配置（黑名单策略）
- SSL/TLS 证书配置
- 身份认证配置（统一用户表）
- 缓冲区配置
- 协议配置聚合
- 伪装配置聚合

所有配置结构采用 PMR 内存分配器，支持高性能零分配内存管理。

## 依赖关系

```
config.hpp
    |
    +-- prism/memory/container.hpp     -> memory::string, memory::vector, memory::map
    +-- prism/protocol/socks5/config.hpp
    +-- prism/protocol/trojan/config.hpp
    +-- prism/protocol/vless/config.hpp
    +-- prism/protocol/shadowsocks/config.hpp
    +-- prism/stealth/reality/config.hpp
    +-- prism/stealth/shadowtls/config.hpp
    +-- prism/stealth/restls/config.hpp
    +-- prism/stealth/anytls/config.hpp
    +-- prism/stealth/trusttunnel/config.hpp
    +-- glaze/glaze.hpp                -> JSON 序列化元数据
```

### 调用者

| 调用者 | 用途 |
|--------|------|
| [[core/agent/context|context.hpp]] | 持有配置引用 |
| `psm::config` | 聚合 agent::config 到顶层 |
| [[core/loader/load|loader::load]] | 解析 JSON 填充配置 |
| [[core/loader/load|loader::build_account_directory]] | 从 authentication 构建账户目录 |
| [[core/agent/worker/worker|worker.cpp]] | 构造时读取配置初始化资源 |

### 被调用者

- [[account/directory|account::directory]] — 使用 authentication.users
- [[core/agent/front/listener|listener]] — 使用 addressable 端点
- [[core/agent/worker/tls|tls]] — 使用 certificate 配置
- [[core/resolve/router|router]] — 使用 positive/reverse_map 路由

---

## 结构体详解

### endpoint

**源码位置**: `config.hpp:39-43`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `host` | memory::string | 空 | 主机名（域名或 IP 地址） |
| `port` | uint16 | 0 | 端口号，0 表示未设置 |

用于：监听端点 (`addressable`)、正向代理目标 (`positive`)、反向代理后端 (`reverse_map`)。

---

### limit

**源码位置**: `config.hpp:57-60`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `blacklist` | bool | `true` | 是否启用黑名单 |

黑名单默认启用，由 Agent 内部维护可疑 IP。并发限制作用于全局所有工作线程。

---

### certificate

**源码位置**: `config.hpp:74-78`

| 字段 | 类型 | 说明 |
|------|------|------|
| `key` | memory::string | 私钥文件路径（PEM 格式） |
| `cert` | memory::string | 证书文件路径（PEM 格式） |

**调用链**: certificate 配置 -> `worker::worker()` 构造 -> `tls::make(cfg.agent)` -> 读取 key/cert 文件 -> `ssl::context` 配置（`use_private_key_file` + `use_certificate_chain_file`）

**约束**: 文件路径必须可读，否则 SSL 上下文初始化失败。

---

### authentication

**源码位置**: `config.hpp:92-115`

**authentication::user** 子结构:

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `password` | memory::string | 空 | 密码认证（Trojan/HTTP/SOCKS5），启动时转为 SHA224 哈希 |
| `uuid` | memory::string | 空 | VLESS UUID 字符串，直接注册 |
| `max_connections` | uint32 | 0 | 最大并发连接数，0 表示无限制 |

| 字段 | 类型 | 说明 |
|------|------|------|
| `users` | memory::vector\<user\> | 统一用户列表 |

**设计要点**: password 和 uuid 共享同一个 entry，共享连接数配额。两个字段均为可选，但至少一个非空才有效。

**调用链**: `loader::build_account_directory(auth)` -> 创建 `account::directory` -> 遍历 users -> password 经 SHA224 哈希后注册，uuid 直接注册。

---

### buffer

**源码位置**: `config.hpp:125-128`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | uint32 | 262144 (256KB) | 传输缓冲区大小（字节） |

默认值 256KB 用于抵消高延迟网络影响。高延迟高带宽环境建议增大，内存受限环境可减小。

---

### protocol::config

**源码位置**: `config.hpp:140-147`

| 字段 | 类型 | 说明 |
|------|------|------|
| `socks5` | protocol::socks5::config | SOCKS5 协议配置 |
| `trojan` | protocol::trojan::config | Trojan 协议配置 |
| `vless` | protocol::vless::config | VLESS 协议配置 |
| `shadowsocks` | protocol::shadowsocks::config | Shadowsocks 协议配置 |

聚合所有代理协议的配置项。

---

### stealth::config

**源码位置**: `config.hpp:158-167`

| 字段 | 类型 | 说明 |
|------|------|------|
| `reality` | stealth::reality::config | Reality TLS 伪装配置 |
| `shadowtls` | stealth::shadowtls::config | ShadowTLS 伪装配置 |
| `restls` | stealth::restls::config | Restls 伪装配置 |
| `anytls` | stealth::anytls::config | AnyTLS 伪装配置 |
| `trusttunnel` | stealth::trusttunnel::config | TrustTunnel 伪装配置 |
| `native_tls` | stealth::native::config | 原生 TLS 兜底配置 |

聚合所有 TLS 伪装方案的配置项。

---

### config（主配置）

**源码位置**: `config.hpp:179-188`

| 字段 | 类型 | 说明 |
|------|------|------|
| `limits` | limit | 连接限制配置 |
| `positive` | endpoint | 正向代理端点 |
| `addressable` | endpoint | 监听端点 |
| `cert` | certificate | SSL/TLS 证书配置 |
| `auth` | authentication | 身份认证配置 |
| `camouflage` | memory::string | 伪装路径，用于抗探测 |
| `reverse_map` | memory::map\<string, endpoint\> | 反向代理路由表 |

仅包含 agent 模块专属配置项。其他模块配置已移至顶层 `psm::config`。

**调用链**: `loader::load(path)` -> 读取文件 -> Glaze JSON 反序列化（通过 `glz::meta<>` 映射字段）-> 返回 `psm::config` -> 各字段分发到对应模块。

---

## 设计决策

### 配置分离原则

Agent 配置与全局配置分离：

```
psm::config (顶层)
    +-- agent::config           <- Agent 专属
    +-- connect::config         <- 连接池
    +-- agent::buffer           <- 缓冲区
    +-- agent::protocol::config <- 协议聚合
    +-- multiplex::config       <- 多路复用
    +-- agent::stealth::config  <- 伪装聚合
    +-- resolve::dns::config    <- DNS
    +-- trace::config           <- 日志
```

### 为什么所有容器使用 PMR？

**问题**: 配置可能包含大量条目（用户列表、反向路由、规则），使用默认分配器产生碎片化的堆分配。

**选择**: 所有 `memory::string` 和 `memory::vector` 使用 PMR 多态内存资源。

**后果**: 启动时在全局池分配一次，运行时零堆分配开销。

### 为什么配置在启动后不可变？

**问题**: 运行时修改配置可能导致竞态条件。

**选择**: 配置对象在服务启动前完成初始化，之后被视为不可变。

**后果**: 确保配置状态一致性，但热重载需要重新构造整个配置对象。

## 约束

### 配置必须在启动前初始化

**类型**: 生命周期
**规则**: 配置对象必须在 `listener::start()` 之前完成初始化。
**违反后果**: 运行时空指针或默认值行为异常

### Glaze 字段映射需与 struct 同步

**类型**: 序列化一致性
**规则**: `glz::meta<T>` 中映射的字段必须与 struct 成员完全对应。新增字段需同步更新 Glaze 映射。
**违反后果**: JSON 反序列化时字段被静默忽略

### certificate 文件路径必须可读

**类型**: 运行时前提条件
**规则**: `key` 和 `cert` 指向的文件必须在 worker 启动时可读。
**违反后果**: SSL 上下文初始化失败，进程退出

## 知识域

- [[core/agent/context|上下文类型定义]] — server_context 持有配置引用
- [[core/loader/load|配置加载]] — loader::load() 填充配置
- [[account/directory|账户目录]] — 从 authentication 构建
- [[core/agent/worker/tls|TLS 配置]] — 从 certificate 构建 SSL 上下文
- [[core/agent/front/listener|监听器]] — 使用 addressable 端点
- [[core/resolve/router|DNS 路由器]] — 使用 positive/reverse_map
