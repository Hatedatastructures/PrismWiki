---
title: "tls.hpp — TLS 上下文初始化模块"
source: "include/prism/agent/worker/tls.hpp"
module: "agent"
type: api
tags: [agent, worker, tls, TLS, 证书, ALPN]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/worker/worker|Worker 核心]]"
  - "[[ref/protocol/tls-1.3|TLS 1.3 协议]]"
  - "[[ref/protocol/tls-extensions|TLS 扩展]]"
  - "[[ref/crypto/ecdhe|ECDHE 密钥交换]]"
---

# tls.hpp

> 源码: `include/prism/agent/worker/tls.hpp`
> 实现: `src/prism/agent/worker/tls.cpp`
> 模块: [[agent|Agent]] / worker / tls

## 概述

提供 TLS/SSL 上下文的创建和配置功能。根据服务配置加载证书链和私钥，设置 GREASE 扩展和 ALPN 协议协商参数。如果未提供证书或私钥，则返回空指针表示运行明文模式。该模块在 Worker 初始化阶段调用，每个 Worker 创建一次 TLS 上下文。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/config\|config]] | 证书和私钥路径配置 |
| 被依赖 | [[agent/worker/worker\|worker]] | worker 构造函数调用 tls::make() |

## 命名空间

`psm::agent::worker::tls`

---

## 类型别名

```cpp
using shared_context = std::shared_ptr<ssl::context>;
```
TLS 上下文共享指针类型别名。

---

## 函数: configure()

**功能说明**: 对给定的 TLS 上下文进行初始化配置，包括加载证书链、私钥、设置 ALPN、协议版本、会话缓存、加密套件和曲线参数。

**签名**:
```cpp
void configure(ssl::context &ctx, std::string_view cert, std::string_view key);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `ctx` | `ssl::context &` | 待配置的 TLS 上下文引用 |
| `cert` | `std::string_view` | 证书链文件路径 |
| `key` | `std::string_view` | 私钥文件路径 |

**返回值**: 无

**异常**: 证书或私钥文件加载失败时抛出 `exception::protocol`

**调用（向下）**:
- `ssl::context::use_certificate_chain_file()` 加载证书链
- `ssl::context::use_private_key_file()` 加载私钥
- `SSL_CTX_set_alpn_protos()` 设置 ALPN 协议列表 (`h2`, `http/1.1`)
- `SSL_CTX_set_min_proto_version()` / `SSL_CTX_set_max_proto_version()` 限定 TLS 1.2 ~ TLS 1.3
- `SSL_CTX_set_session_cache_mode()` 启用服务端会话缓存（2048 条目，5 分钟超时）
- `SSL_CTX_set_cipher_list()` 设置加密套件优先级
- `SSL_CTX_set1_curves_list()` 设置 ECDHE 曲线 (`X25519`, `P-256`, `P-384`)
- `SSL_CTX_set_options(SSL_OP_NO_COMPRESSION)` 禁用压缩（防御 CRIME 攻击）
- `trace::error()` 记录加载失败日志
- `exception::protocol()` 抛出异常

**被调用（向上）**: `tls::make()` 内部调用

**涉及的知识域**:
- [[ref/protocol/tls-1.3\|TLS 1.3 协议]]
- [[ref/protocol/tls-extensions\|ALPN 协议协商]]
- [[ref/crypto/ecdhe\|ECDHE 密钥交换]]

---

## 函数: make()

**功能说明**: 根据服务配置创建 TLS 上下文，如果证书或私钥未配置则返回空指针表示运行明文 HTTP 模式。

**签名**:
```cpp
[[nodiscard]] auto make(const agent::config &cfg)
    -> shared_context;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `cfg` | `const agent::config &` | 代理服务配置，包含证书和私钥路径 |

**返回值**: `shared_context`（`std::shared_ptr<ssl::context>`），如果未配置证书则返回空指针

**异常**:
- `exception::protocol` — 证书或私钥加载失败
- `std::exception` — 其他初始化异常

**调用（向下）**:
- 读取 `cfg.cert.cert` 和 `cfg.cert.key` 判断是否配置
- `std::make_shared<ssl::context>(ssl::context::tls)` 创建 TLS 上下文
- `configure()` 配置上下文参数
- `trace::warn()` 记录无证书时的警告日志
- `trace::error()` 记录初始化失败日志

**被调用（向上）**: [[agent/worker/worker\|worker 构造函数]] 初始化 `ssl_ctx_` 成员时调用

**涉及的知识域**:
- [[ref/protocol/tls-1.3\|TLS 1.3 协议]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹伪装]]

---

## 知识域

- [[ref/protocol/tls-1.3\|TLS 1.3 协议]]
- [[ref/protocol/tls-extensions\|TLS 扩展与 ALPN]]
- [[ref/crypto/ecdhe\|ECDHE 密钥交换]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹伪装]]
