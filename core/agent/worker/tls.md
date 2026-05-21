---
layer: core
source: I:/code/Prism/include/prism/agent/worker/tls.hpp
title: tls 模块
---

# tls 模块

## 源码位置

`I:/code/Prism/include/prism/agent/worker/tls.hpp`

## 模块职责

TLS 上下文初始化模块，提供 TLS/SSL 上下文的创建和配置功能。根据服务配置加载证书链和私钥，设置 GREASE 扩展和 ALPN 协议协商参数。如果未提供证书或私钥，则返回空指针表示运行明文模式。

**调用时机**: Worker 初始化阶段，每个 Worker 创建一次 TLS 上下文。

## 类型定义

```cpp
using shared_context = std::shared_ptr<ssl::context>;
```

TLS 上下文共享指针类型别名。

## 主要函数

### configure

```cpp
void configure(
    ssl::context &ctx,
    std::string_view cert,
    std::string_view key
);
```

对给定的 TLS 上下文进行初始化配置。

**配置项**:

| 配置项 | 说明 |
|--------|------|
| 证书链 | 加载证书链文件，用于服务端身份验证 |
| 私钥 | 加载私钥文件，用于密钥交换 |
| GREASE 扩展 | 增加 TLS 指纹随机性，提升安全性 |
| ALPN 协议 | 支持 HTTP/2 和 HTTP/1.1 协议协商 |

**参数**:
| 参数 | 说明 |
|------|------|
| `ctx` | 待配置的 TLS 上下文引用 |
| `cert` | 证书链文件路径 |
| `key` | 私钥文件路径 |

**异常**: 证书或私钥文件加载失败时抛出 `exception::protocol`

### make

```cpp
[[nodiscard]] auto make(const agent::config &cfg)
    -> shared_context;
```

根据服务配置创建 TLS 上下文。

**处理流程**:

```mermaid
graph TD
    A[make] --> B{证书路径为空?}
    B -->|是| C[返回空指针]
    B -->|否| D{私钥路径为空?}
    D -->|是| C
    D -->|否| E[创建 ssl::context]
    E --> F[configure 配置]
    F --> G[返回 TLS 上下文]
```

**参数**:
| 参数 | 说明 |
|------|------|
| `cfg` | 代理服务配置，包含证书和私钥路径 |

**返回值**: TLS 上下文共享指针，如果未配置证书则返回空指针

**异常**:
- `exception::protocol`: 证书或私钥加载失败
- `std::exception`: 其他初始化异常

## 配置详情

### GREASE 扩展

GREASE (Generate Random Extensions And Sustain Extensibility) 是一种防止 TLS 指纹识别的技术，通过在扩展中填充随机值来增加指纹的随机性。

### ALPN 协议协商

支持以下协议优先级:
1. `h2` - HTTP/2
2. `http/1.1` - HTTP/1.1

## 调用链

```mermaid
graph TD
    A[worker 构造] --> B[tls::make]
    B --> C{证书配置存在?}
    C -->|是| D[创建 ssl::context]
    D --> E[tls::configure]
    E --> F[加载证书链]
    F --> G[加载私钥]
    G --> H[设置 GREASE]
    H --> I[设置 ALPN]
    I --> J[返回 TLS 上下文]
    C -->|否| K[返回空指针]

    L[session 启动] --> M{TLS 上下文存在?}
    M -->|是| N[TLS 握手]
    M -->|否| O[明文模式]
```

## 明文模式

当 TLS 上下文为空指针时，worker 仅处理明文 HTTP 流量:

```cpp
if (ssl_ctx_) {
    // TLS 模式
} else {
    // 明文模式
}
```

## 相关文档

- [[core/agent/worker/worker|Worker 模块]]
- [[core/agent/config|配置模块]]
- [[core/crypto/tls|TLS 加密]]

---

## TLS 配置详解

### configure 函数逐项解析

```cpp
void configure(
    ssl::context &ctx,
    std::string_view cert,
    std::string_view key
);
```

完整配置流程：

```cpp
void configure(ssl::context &ctx, std::string_view cert, std::string_view key) {
    // ===== 阶段 1: TLS 协议版本 =====

    // 设置最低 TLS 版本为 TLS 1.2（拒绝 SSLv3/SSLv2/TLS 1.0/1.1）
    ctx.set_options(ssl::context::default_workarounds);
    ctx.set_verify_mode(ssl::context::verify_none);
    // verify_none: 服务端模式不验证客户端证书

    // ===== 阶段 2: 加载证书链 =====

    ctx.use_certificate_chain_file(std::string{cert}, ec);
    if (ec) {
        throw exception::protocol{"Failed to load certificate chain: " + ec.message()};
    }
    // use_certificate_chain_file 加载完整的证书链（服务器证书 + 中间 CA）
    // 证书文件必须是 PEM 格式

    // ===== 阶段 3: 加载私钥 =====

    ctx.use_private_key_file(std::string{key}, ssl::context::pem, ec);
    if (ec) {
        throw exception::protocol{"Failed to load private key: " + ec.message()};
    }
    // 支持 RSA 和 EC 私钥
    // 私钥文件必须是 PEM 格式
    // 加密的私钥需要先解密再加载

    // ===== 阶段 4: GREASE 扩展 =====

    ctx.set_options(ssl::context::enable_grease);
    // GREASE (Generate Random Extensions And Sustain Extensibility)
    // 在 ClientHello/ServerHello 中注入随机扩展值
    // 防止中间件通过固定 TLS 指纹识别代理流量

    // ===== 阶段 5: ALPN 协议协商 =====

    ctx.set_alpn_protocols({"h2", "http/1.1"});
    // 优先级顺序: HTTP/2 > HTTP/1.1
    // 客户端在 TLS 握手中声明支持的 ALPN
    // 服务端选择最高优先级的共同协议
```

### 支持的密钥格式

| 密钥类型 | 文件格式 | 支持情况 | 说明 |
|----------|----------|----------|------|
| RSA | PEM | 支持 | `-----BEGIN RSA PRIVATE KEY-----` |
| RSA | PKCS#8 PEM | 支持 | `-----BEGIN PRIVATE KEY-----` |
| EC (P-256) | PEM | 支持 | `-----BEGIN EC PRIVATE KEY-----` |
| EC (P-384) | PEM | 支持 | 同上 |
| EC (P-521) | PEM | 支持 | 同上 |
| X25519 | PEM | 取决于 OpenSSL 版本 | 需要 OpenSSL 1.1.1+ |
| 加密私钥 | PEM + 密码 | 不支持 | 需要预解密 |

### 密码套件选择

Prism 依赖 Boost.Asio 的默认密码套件配置，该配置基于 OpenSSL 的默认安全策略：

| 密码套件类别 | 是否启用 | 说明 |
|-------------|----------|------|
| AES-256-GCM | 是 | 首选，256 位安全性 |
| AES-128-GCM | 是 | 次选，128 位安全性 |
| ChaCha20-Poly1305 | 是 | 移动设备优化 |
| AES-CBC | 取决于 OpenSSL 版本 | 可能被禁用 |
| RC4/DES | 否 | 已被现代 OpenSSL 禁用 |
| NULL 加密 | 否 | 安全性为零 |

---

## 证书管理

### 证书加载时机

```
进程启动
    │
    ▼
读取配置文件 (config)
    │
    ├── agent.tls_cert = "/path/to/cert.pem"
    └── agent.tls_key = "/path/to/key.pem"
    │
    ▼
每个 worker 构造时调用 tls::make(cfg)
    │
    ├── 读取 cert.pem（文件系统 I/O）
    ├── 读取 key.pem（文件系统 I/O）
    ├── 解析 PEM 格式（CPU 密集型）
    ├── 验证证书有效性（日期检查）
    └── 返回 shared_ptr<ssl::context>
    │
    ▼
所有 worker 共享同一个 ssl::context
```

### 证书热更新（当前与未来）

**当前状态**：TLS 上下文在 worker 启动时创建，之后不可变更。

**未来扩展方案**：

```cpp
// 利用 server_context::swap_config 的热加载机制
// 未来可实现类似方案：

void reload_tls(server_context &srv, std::string_view new_cert, std::string_view new_key) {
    auto new_ctx = std::make_shared<ssl::context>(ssl::context::tlsv12_server);
    configure(*new_ctx, new_cert, new_key);
    srv.ssl_ctx.store(new_ctx);  // 原子交换
    // 新连接使用新上下文，已有连接不受影响
}
```

**热更新约束**：
- 已建立的 TLS 连接不受影响（每个连接持有独立的 `ssl::stream`）
- 新连接使用新的 `ssl::context`
- 需要确保旧证书文件在切换期间仍可用

### SNI (Server Name Indication) 路由

**当前状态**：Prism 的 TLS 模块使用单一证书上下文，不支持 SNI 多证书。

**SNI 扩展方案**：

```cpp
// 通过 set_servername_callback 实现 SNI 路由
ctx.set_servername_callback([&certificates](ssl::stream<tcp::socket&> &stream,
                                             const char *server_name) {
    auto it = certificates.find(server_name);
    if (it != certificates.end()) {
        stream.use_certificate(it->second.cert);
        stream.use_private_key(it->second.key);
    }
});
```

**适用场景**：
- 一个 Prism 实例服务多个域名
- 每个域名使用不同的 TLS 证书
- 反向代理场景下需要匹配后端 SNI

---

## 与伪装方案的交互

### TLS 上下文在协议栈中的位置

```
客户端连接
    │
    ▼
listener.accept() ──► balancer.dispatch() ──► worker.dispatch_socket()
    │
    ▼
session.start() ──► diversion() 协议检测
    │
    ├── 检测到 HTTP ──► HTTP Pipeline（明文）
    ├── 检测到 TLS ──► TLS Pipeline ──► SSL Handshake ──► 内部协议检测
    │       │                              │
    │       │                              ▼
    │       │                    使用 ssl_ctx_ 进行握手
    │       │                              │
    │       │                              ▼
    │       │                    解密后检测: VLESS / Trojan / 其他
    │       │
    │       └── ssl_ctx_ 来源: worker.ssl_ctx_ (tls::make)
    │
    └── 检测到 SOCKS5 ──► SOCKS5 Pipeline（明文认证）
```

### Reality 方案交互

Reality 是 Xray-core 提出的一种 TLS 伪装方案，通过在 TLS 握手中使用真实网站的证书来规避检测：

```
Reality 与 Prism TLS 的关系:

Prism 原生 TLS:
    ClientHello ──► ServerHello (自有证书)
                         │
                         ▼
                   正常的 TLS 握手

Reality 模式 (未来扩展):
    ClientHello ──► 检测是否为 Reality 客户端
                         │
                    ┌────┴────┐
                    ▼         ▼
              Reality 客户端    正常 TLS 客户端
                    │              │
                    ▼              ▼
              Reality 握手       正常握手 (使用 ssl_ctx_)
              (自有密钥)
```

在 Reality 模式下：
- 非 Reality 客户端看到的是一个正常的 TLS 握手（使用真实网站证书）
- Reality 客户端通过特定 SNI 或 TLS 扩展触发 Reality 握手
- Prism 的 `ssl_ctx_` 需要支持双模式：Reality 自有证书 + 正常自有证书

### ShadowTLS 方案交互

ShadowTLS 是一种将代理协议流量伪装为正常 TLS 流量的方案：

```
ShadowTLS 与 Prism TLS 的关系:

ShadowTLS V3 流程:
    Client ──► TLS ClientHello (正常)
                 │
                 ▼
    Server ──► TLS ServerHello (正常证书)
                 │
                 ▼
    双方进入 TLS 应用数据流
                 │
                 ▼
    ShadowTLS 在 TLS 记录层之上复用底层 TLS 流
                 │
                 ▼
    第一条应用数据是 ShadowTLS 握手（非标准 TLS 协议）

Prism 集成点:
    diversion() 检测到 TLS (0x16 0x03 ...)
        │
        ▼
    TLS Pipeline: 完成 TLS 握手
        │
        ▼
    解密后的第一条数据
        │
        ├── 标准 TLS 应用数据 (HTTP/2, etc.) ──► 正常处理
        └── ShadowTLS 握手 ──► ShadowTLS Pipeline
```

Prism 与 ShadowTLS 的集成需要在 TLS Pipeline 内部增加协议检测：

```
TLS Pipeline 内部流程:
    SSL Handshake ──► TLS 加密通道建立
        │
        ▼
    解密第一条应用数据
        │
        ├── HTTP ──► HTTP Handler
        ├── SOCKS5 ──► SOCKS5 Handler
        ├── ShadowTLS ──► ShadowTLS Handler
        └── VLESS/Trojan ──► 对应 Handler
```

### 与 TLS Pipeline 的分工

`tls::make/configure` 创建的上下文是底层 TLS 握手的基础设施，上层伪装协议（Reality/ShadowTLS）在其之上运行：

```
层级关系:
    ┌─────────────────────────────────┐
    │  应用层 (HTTP/SOCKS5/VLESS)      │
    ├─────────────────────────────────┤
    │  伪装层 (Reality/ShadowTLS)     │ ← 可选
    ├─────────────────────────────────┤
    │  TLS 加密层 (ssl::stream)       │ ← ssl_ctx_ 创建
    ├─────────────────────────────────┤
    │  传输层 (tcp::socket)           │ ← reliable transmission
    └─────────────────────────────────┘
```

TLS 模块仅负责第三层（TLS 加密层），上层的伪装逻辑由各自的 pipeline 处理。

---

## 明文模式详解

当 `tls::make(cfg)` 返回空指针时，worker 进入明文模式：

```cpp
// worker 构造
ssl_ctx_ = tls::make(cfg);  // cfg.tls_cert 为空 → 返回 nullptr

// session 创建时
if (ssl_ctx_) {
    // 创建 ssl::stream<tcp::socket>
} else {
    // 直接使用 tcp::socket
    auto transmission = channel::transport::make_reliable(
        std::move(socket), pool
    );
}
```

**明文模式下的协议检测**：

```
diversion() 在明文模式下:
    预读 24 字节
        │
        ├── HTTP ──► HTTP Pipeline
        ├── SOCKS5 ──► SOCKS5 Pipeline
        ├── TLS (0x16 0x03 ...) ──► 警告: 客户端尝试 TLS 连接但服务端为明文
        │                              可以选择关闭连接或记录日志
        └── 未知 ──► 原始透传
```

**安全建议**：生产环境不建议使用明文模式，除非 Prism 运行在内网受信任环境中。
