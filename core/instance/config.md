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
    │
    ├── prism/memory/container.hpp     → memory::string, memory::vector, memory::map
    ├── prism/protocol/socks5/config.hpp
    ├── prism/protocol/trojan/config.hpp
    ├── prism/protocol/vless/config.hpp
    ├── prism/protocol/shadowsocks/config.hpp
    ├── prism/stealth/reality/config.hpp
    ├── prism/stealth/shadowtls/config.hpp
    ├── prism/stealth/restls/config.hpp
    ├── prism/stealth/anytls/config.hpp
    ├── prism/stealth/trusttunnel/config.hpp
    │
    └── glaze/glaze.hpp                → JSON 序列化元数据
```

### 调用者

| 调用者 | 用途 |
|--------|------|
| [[context|context.hpp]] | 持有配置引用 |
| `psm::config` | 聚合 agent::config 到顶层 |
| [[../loader/load|loader::load]] | 解析 JSON 填充配置 |
| [[../loader/load|loader::build_account_directory]] | 从 authentication 构建账户目录 |
| [[worker/worker|worker.cpp]] | 构造时读取配置初始化资源 |

### 被调用者

配置结构被以下模块使用：

- [[account/directory|account::directory]] — 使用 authentication.users
- [[front/listener|listener]] — 使用 addressable 端点
- [[worker/tls|tls]] — 使用 certificate 配置
- [[core/resolve/router|router]] — 使用 positive/reverse_map 路由

---

## 结构体详解

### endpoint

**源码位置**: `config.hpp:36-40`

```cpp
struct endpoint
{
    memory::string host;    // 主机名，可以是域名或 IP 地址
    std::uint16_t port = 0; // 端口号，范围 1-65535，0 表示未设置
};
```

#### 职责

表示网络端点，包含主机名和端口号。用于：

- 监听端点 (`addressable`)
- 正向代理目标 (`positive`)
- 反向代理后端 (`reverse_map`)

#### 设计要点

- 主机名支持域名和 IP 地址两种格式
- 端口号 0 表示未设置，配置验证时应检查有效性
- 使用 `memory::string` 存储，支持 PMR 内存分配

#### Glaze 元数据

```cpp
// 源码: config.hpp:188-192
template <>
struct glz::meta<psm::instance::endpoint>
{
    using T = psm::instance::endpoint;
    static constexpr auto value = glz::object("host", &T::host, "port", &T::port);
};
```

#### 使用示例

```cpp
// 监听端点配置
agent::config cfg;
cfg.addressable.host = "0.0.0.0";
cfg.addressable.port = 443;

// 正向代理配置
cfg.positive.host = "proxy.example.com";
cfg.positive.port = 1080;
```

---

### limit

**源码位置**: `config.hpp:54-57`

```cpp
struct limit
{
    bool blacklist = true; // 是否启用黑名单，默认启用
};
```

#### 职责

控制并发连接数和黑名单策略，防止资源耗尽和恶意攻击。

#### 设计要点

- 黑名单默认启用
- 并发限制作用于全局所有工作线程
- 黑名单由 Agent 内部维护，记录可疑 IP

#### Glaze 元数据

```cpp
// 源码: config.hpp:194-199
template <>
struct glz::meta<psm::instance::limit>
{
    using T = psm::instance::limit;
    static constexpr auto value = glz::object("blacklist", &T::blacklist);
};
```

---

### certificate

**源码位置**: `config.hpp:71-75`

```cpp
struct certificate
{
    memory::string key;  // 私钥文件路径，PEM 格式
    memory::string cert; // 证书文件路径，PEM 格式
};
```

#### 职责

包含 TLS 加密所需的私钥和证书文件路径。

#### 设计要点

- 文件格式要求 PEM
- 私钥用于 TLS 握手签名
- 证书包含公钥和证书链信息
- 文件路径必须可读，否则 SSL 上下文初始化失败

#### Glaze 元数据

```cpp
// 源码: config.hpp:200-206
template <>
struct glz::meta<psm::instance::certificate>
{
    using T = psm::instance::certificate;
    static constexpr auto value = glz::object("key", &T::key, "cert", &T::cert);
};
```

#### 调用链

```
certificate 配置
    │
    ▼
worker::worker() 构造
    │
    ▼
tls::make(cfg.agent)  // worker/tls.hpp
    │
    ├── 读取 key 文件
    ├── 读取 cert 文件
    │
    ▼
ssl::context 配置
    │
    ├── ssl_ctx_->use_private_key_file()
    └── ssl_ctx_->use_certificate_chain_file()
```

---

### authentication

**源码位置**: `config.hpp:89-112`

```cpp
struct authentication
{
    struct user
    {
        memory::string password;           // 密码认证，用于 Trojan/HTTP/SOCKS5
        memory::string uuid;               // VLESS UUID 字符串
        std::uint32_t max_connections = 0; // 最大并发连接数，0 表示无限制
    };

    memory::vector<user> users; // 统一用户列表
};
```

#### 职责

管理客户端身份验证的凭据和用户限制。

#### 设计要点

- password 用于 Trojan/HTTP/SOCKS5 协议
- uuid 用于 VLESS 协议
- 两种凭证共享同一个 entry，共享连接数配额
- max_connections = 0 表示无限制

#### Glaze 元数据

```cpp
// 源码: config.hpp:209-221
template <>
struct glz::meta<psm::instance::authentication::user>
{
    using T = psm::instance::authentication::user;
    static constexpr auto value = glz::object(
        "password", &T::password, "uuid", &T::uuid, "max_connections", &T::max_connections);
};

template <>
struct glz::meta<psm::instance::authentication>
{
    using T = psm::instance::authentication;
    static constexpr auto value = glz::object("users", &T::users);
};
```

#### 调用链详解

```
loader::build_account_directory(auth)
    │
    │  // 源码: loader/load.hpp:75-114
    │
    ├── 创建 account::directory
    │       const auto dir = std::make_shared<agent::account::directory>(...);
    │
    ├── 预估条目数并预留容量
    │       for (const auto &user : auth.users)
    │           if (!user.password.empty()) ++entry_count;
    │           if (!user.uuid.empty()) ++entry_count;
    │       dir->reserve(entry_count);
    │
    ├── 遍历用户列表
    │       for (const auto &user : auth.users)
    │       {
    │           auto shared_entry = std::make_shared<agent::account::entry>();
    │           shared_entry->max_connections = user.max_connections;
    │
    │           // password → SHA224 哈希 → 注册
    │           if (!user.password.empty())
    │           {
    │               const auto normalized = crypto::normalize_credential(user.password);
    │               dir->insert(normalized, shared_entry);
    │           }
    │
    │           // uuid → 直接注册
    │           if (!user.uuid.empty())
    │           {
    │               dir->insert(std::string_view(user.uuid), shared_entry);
    │           }
    │       }
    │
    ▼
返回 account::directory 共享指针
```

---

### buffer

**源码位置**: `config.hpp:122-125`

```cpp
struct buffer
{
    std::uint32_t size = 262144U; // 传输缓冲区大小（字节），默认 256KB
};
```

#### 职责

控制数据传输时的缓冲区大小。

#### 设计要点

- 默认值 256KB 用于抵消高延迟网络影响
- 高延迟高带宽环境建议增大
- 内存受限环境可减小

---

### namespace protocol

**源码位置**: `config.hpp:134-143`

```cpp
namespace protocol
{
    struct config
    {
        psm::protocol::socks5::config socks5;
        psm::protocol::trojan::config trojan;
        psm::protocol::vless::config vless;
        psm::protocol::shadowsocks::config shadowsocks;
    };
}
```

#### 职责

聚合所有代理协议的配置项。

#### Glaze 元数据

```cpp
// 源码: config.hpp:230-239
template <>
struct glz::meta<psm::instance::protocol::config>
{
    using T = psm::instance::protocol::config;
    static constexpr auto value = glz::object(
        "socks5",       &T::socks5,
        "trojan",       &T::trojan,
        "vless",        &T::vless,
        "shadowsocks",  &T::shadowsocks);
};
```

---

### namespace stealth

**源码位置**: `config.hpp:151-161`

```cpp
namespace stealth
{
    struct config
    {
        psm::stealth::reality::config reality;
        psm::stealth::shadowtls::config shadowtls;
        psm::stealth::restls::config restls;
        psm::stealth::anytls::config anytls;
        psm::stealth::trusttunnel::config trusttunnel;
    };
}
```

#### 职责

聚合所有 TLS 伪装方案的配置项。

#### Glaze 元数据

```cpp
// 源码: config.hpp:241-251
template <>
struct glz::meta<psm::instance::stealth::config>
{
    using T = psm::instance::stealth::config;
    static constexpr auto value = glz::object(
        "reality",      &T::reality,
        "shadowtls",    &T::shadowtls,
        "restls",       &T::restls,
        "anytls",       &T::anytls,
        "trusttunnel",  &T::trusttunnel);
};
```

---

### config

**源码位置**: `config.hpp:173-182`

```cpp
struct config
{
    limit limits;                                      // 连接限制配置
    endpoint positive;                                 // 正向代理端点
    endpoint addressable;                              // 监听端点
    certificate cert;                                  // SSL/TLS 证书配置
    authentication auth;                               // 身份认证配置
    memory::string camouflage;                         // 伪装路径，用于抗探测
    memory::map<memory::string, endpoint> reverse_map; // 反向代理路由表
};
```

#### 职责

Agent 核心配置，仅包含 agent 模块专属配置项。

#### 设计要点

- 配置应在服务启动前完成初始化
- 运行时修改可能不会立即生效
- 其他模块配置已移至顶层 `psm::config`

#### Glaze 元数据

```cpp
// 源码: config.hpp:253-265
template <>
struct glz::meta<psm::instance::config>
{
    using T = psm::instance::config;
    static constexpr auto value = glz::object(
        "limit",           &T::limits,
        "positive",        &T::positive,
        "addressable",     &T::addressable,
        "certificate",     &T::cert,
        "authentication",  &T::auth,
        "camouflage",      &T::camouflage,
        "reverse_map",     &T::reverse_map);
};
```

#### 调用链详解

```
loader::load(path)  // loader/load.hpp
    │
    ├── 打开文件
    │       std::ifstream file(path.data(), std::ios::binary);
    │
    ├── 读取全部内容
    │       memory::string content(size, '\0');
    │       file.read(content.data(), size);
    │
    ├── glaze 反序列化
    │       transformer::json::deserialize(content, cfg);
    │       → glaze 内部使用 meta<> 映射 JSON 字段到结构体成员
    │
    ▼
返回 psm::config
    │
    ├── cfg.agent.addressable → listener 监听端点
    ├── cfg.agent.auth → build_account_directory()
    ├── cfg.agent.cert → tls::make()
    ├── cfg.agent.positive → router.set_positive_endpoint()
    ├── cfg.agent.reverse_map → router.add_reverse_route()
```

---

## 设计理念

### 配置分离原则

Agent 配置与全局配置分离：

```
psm::config (顶层)
    ├── agent::config      ← Agent 专属
    ├── connect::config    ← 连接池
    ├── agent::buffer      ← 缓冲区
    ├── agent::protocol::config ← 协议聚合
    ├── multiplex::config  ← 多路复用
    ├── agent::stealth::config ← 伪装聚合
    ├── resolve::dns::config ← DNS
    ├── trace::config      ← 日志
```

### PMR 内存策略

所有字符串和容器使用 PMR：

- 启动时在全局池分配一次
- 运行时零堆分配开销

### 运行时不可变性

配置对象在服务启动前完成初始化：

- 确保配置状态一致性
- 避免运行时竞态条件

---

## 知识域

- [[context|上下文类型定义]] — server_context 持有配置引用
- [[../loader/load|配置加载]] — loader::load() 填充配置
- [[account/directory|账户目录]] — 从 authentication 构建
- [[worker/tls|TLS 配置]] — 从 certificate 构建 SSL 上下文
- [[front/listener|监听器]] — 使用 addressable 端点
- [[core/resolve/router|DNS 路由器]] — 使用 positive/reverse_map