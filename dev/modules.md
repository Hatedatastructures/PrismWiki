---
layer: dev
title: Prism 模块结构
created: 2026-05-17
updated: 2026-05-17
type: concept
tags: [architecture, modules, overview, design-pattern]
related: 
  - "[[core/instance/overview|agent-overview]]"
  - "[[core/architecture|agent-architecture]]"
  - "[[core/protocol/overview|protocol]]"
  - "[[core/pipeline/overview|pipeline]]"
  - "[[core/recognition/overview|recognition]]"
  - "[[dev/coding/coroutine|coroutines]]"
  - "[[dev/coding/pmr|pmr]]"
sources:
  - I:/code/Prism/CLAUDE.md
  - I:/code/Prism/include/prism/
confidence: high
layer: dev
---

# Prism 模块结构

Prism 采用模块化设计，每个模块职责清晰，通过聚合头文件组织。命名空间统一使用 `psm::` 前缀。这种设计确保了代码的可维护性、可测试性和可扩展性。

## 概述

Prism 是一个高性能协程代理服务器，采用 C++23 纯协程架构和 PMR（多态内存资源）实现热路径零堆分配。整个系统由17个核心模块组成，每个模块都有明确的职责边界和清晰的接口定义。

### 设计理念

Prism 的模块设计遵循以下核心原则：

1. **职责单一** — 每个模块只负责一个特定领域，避免功能混杂
2. **接口清晰** — 模块间通过明确定义的接口通信，降低耦合
3. **层次分明** — 模块按职责分层，上层依赖下层，下层不依赖上层
4. **可替换性** — 模块实现可以替换而不影响其他模块
5. **可测试性** — 每个模块都可以独立测试

### 模块分类

Prism 的模块可以分为以下几类：

| 分类 | 模块 | 特点 |
|------|------|------|
| 核心引擎 | agent | 会话管理、协议分发、负载均衡 |
| 协议层 | protocol、pipeline | 协议编解码、处理编排 |
| 识别层 | recognition | 智能协议识别、TLS 伪装方案 |
| 连接层 | channel、multiplex | 连接池、多路复用 |
| 伪装层 | stealth | TLS 伪装技术 |
| 解析层 | resolve | DNS 解析管道 |
| 基础设施 | memory、trace、fault、exception、crypto | 内存、日志、错误、加密 |
| 配置加载 | config、loader、transformer、outbound | 配置管理、数据转换 |

### 模块总览表

| 模块 | 头文件 | 源文件 | 职责 |
|------|--------|--------|------|
| agent（核心） | `include/prism/instance/` | `src/prism/instance/` | 核心会话链路：监听、负载均衡、会话管理、协议分发 |
| recognition | `include/prism/recognition/` | `src/prism/recognition/` | 协议智能识别：外层探测 + TLS 伪装方案识别 |
| pipeline | `include/prism/pipeline/` | `src/prism/pipeline/` | 协议处理管道：原语 + 各协议会话编排 |
| protocol | `include/prism/protocol/` | `src/prism/protocol/` | 协议编解码：消息结构、中继器、TLS 特征 |
| resolve | `include/prism/resolve/` | `src/prism/resolve/` | DNS 解析：七阶段查询管道 |
| connect | `include/prism/connect/` | `src/prism/connect/` | 连接管理：连接池、拨号、Happy Eyeballs |
| transport | `include/prism/transport/` | `src/prism/transport/` | 传输层 |
| multiplex | `include/prism/multiplex/` | `src/prism/multiplex/` | 多路复用：smux v1 + yamux |
| stealth | `include/prism/stealth/` | `src/prism/stealth/` | TLS 伪装：Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel |
| memory | `include/prism/memory/` | header-only | 内存管理：PMR 全局池 + 线程独占池 |
| trace | `include/prism/trace/` | `src/prism/trace/` | 日志系统 |
| fault | `include/prism/fault/` | header-only | 错误码定义 |
| crypto | `include/prism/crypto/` | `src/prism/crypto/` | 加密算法 |
| exception | `include/prism/exception/` | header-only | 异常层次结构 |
| config | `include/prism/config.hpp` | header-only | 全局配置管理 |
| transformer | `include/prism/transformer/` | header-only | 数据转换 |
| outbound | `include/prism/outbound/` | header-only | 出站连接抽象 |
| loader | `include/prism/loader/` | header-only | 配置加载 |

### 模块依赖关系图

```
main.cpp
    │
    ▼
agent ← loader（配置加载）← config（配置结构）
    │
    ├──→ recognition → stealth（TLS 伪装）
    │         │
    │         ▼
    │    protocol（编解码）
    │         │
    │         ▼
    │    pipeline（编排）
    │
    ├──→ channel（连接池）→ transport（传输层）
    │         │
    │         ▼
    │    multiplex（多路复用）
    │
    ├──→ resolve（DNS）→ dns（解析器）
    │         │
    │         ▼
    │    crypto（加密）
    │
    └──→ memory（内存池）← trace（日志）
              │
              ▼
         fault（错误码）← exception（异常）
```

### 启动流程与模块初始化

当 Prism 启动时，模块按以下顺序初始化：

1. **memory::system::enable_global_pooling()** — 初始化全局内存池
2. **stealth::register_all_schemes()** — 注册所有 TLS 伪装方案
3. **loader::load()** — 加载配置文件
4. **trace::init()** — 初始化日志系统
5. **loader::build_account_directory()** — 构建账户目录
6. **创建 worker 线程池** — CPU 核心数 - 1 个 worker
7. **构建 balancer** — 负载均衡器
8. **启动 listener** — 监听器
9. **启动所有线程** — worker 线程 + 监听线程

这个顺序确保了基础设施（内存、日志）先初始化，然后是配置加载，最后是业务模块启动。

## 详解

### Agent 模块

Agent 是 Prism 的核心引擎模块，负责整个代理服务器的运行生命周期。

#### Agent 子模块结构

`include/prism/instance/` 目录结构：

```
agent/
├── account/          # 账户管理
│   ├── entry.hpp     # 账户条目定义
│   └── directory.hpp # 账户目录（密码→SHA224 映射）
├── dispatch/         # 连接分发
│   └── table.hpp     # 处理器表（编译期常量数组）
├── front/            # 前端层
│   ├── listener.hpp  # TCP 监听器
│   └── balancer.hpp  # 负载均衡器
├── session/          # 会话管理
│   └── session.hpp   # 会话生命周期
├── worker/           # 工作线程
│   ├── worker.hpp    # worker 实例（持有 io_context）
│   ├── launch.hpp    # 会话启动
│   ├── stats.hpp     # 统计信息
│   └── tls.hpp       # TLS 配置
├── context.hpp       # 上下文
└── config.hpp        # 配置
```

#### Agent 核心职责

1. **监听连接** — `listener` 负责接受新连接，计算亲和性哈希
2. **负载均衡** — `balancer` 选择合适的 worker 处理连接
3. **会话管理** — `session` 管理连接的整个生命周期
4. **协议分发** — `dispatch` 根据协议类型选择处理器
5. **账户认证** — `account` 管理用户账户和认证

### Recognition 模块

Recognition 是协议智能识别层，采用三阶段流水线架构。

#### Recognition 流水线

```
probe::probe(transport, 24)     → detect() → protocol_type
       │ (仅当 TLS)
       ▼
identify():
  read_clienthello()            → raw_clienthello
       │
       ▼
  parse_clienthello()           → clienthello_features
       │
       ▼
  analyzer_registry::analyze()  → analysis_result{candidates, confidence}
       │
       ▼
  scheme_executor::execute()    → execution_result{transport, detected}
```

#### Recognition 子模块结构

```
recognition/
├── probe/            # 探测层
│   ├── probe.hpp     # 协议探测（预读 24 字节）
│   └── analyzer.hpp  # 特征分析器接口
├── recognition.hpp   # 识别入口
├── result.hpp        # 识别结果
└── confidence.hpp    # 置信度
```

#### 协议探测逻辑

Recognition 的第一阶段是探测，通过预读前 24 字节判断协议类型：

| 首字节特征 | 协议类型 |
|-----------|---------|
| `0x05` (SOCKS5 版本) | SOCKS5 |
| HTTP 方法（GET/POST/CONNECT 等） | HTTP |
| TLS Handshake Type `0x01` | TLS |
| SS2022 特征 | Shadowsocks 2022 |

### Pipeline 模块

Pipeline 是协议处理管道层，负责协议的具体处理逻辑。

#### Pipeline 子模块结构

```
pipeline/
├── primitives.hpp    # 隧道原语（双向透明转发）
├── http.hpp          # HTTP 协议处理
├── socks5.hpp        # SOCKS5 协议处理
├── trojan.hpp        # Trojan 协议处理
├── vless.hpp         # VLESS 协议处理
├── shadowsocks.hpp   # SS2022 协议处理
└── pipeline.hpp      # 聚合头文件
```

#### Pipeline 处理流程

每个协议处理器遵循相同的处理模式：

1. **解析请求** — 解析协议头部，获取目标地址
2. **建立连接** — 通过 router 建立上游连接
3. **双向转发** — 使用 primitives::tunnel 进行透明转发
4. **清理资源** — 关闭连接，归还资源

### Channel 模块

Channel 是连接管理层，负责连接池、传输层和 Happy Eyeballs。

#### Channel 子模块结构

```
channel/
├── connection/       # 连接管理
│   ├── pool.hpp      # 连接池实现
│   └── pool.cpp      # LIFO 栈式缓存
├── transport/        # 传输层
│   ├── reliable.hpp  # 可靠传输（TCP）
│   ├── encrypted.hpp # 加密传输（TLS）
│   └── transmission.hpp # 传输抽象
├── eyeball.hpp       # Happy Eyeballs v2
└── channel.hpp       # 聚合头文件
```

#### 连接池特性

Channel 的连接池采用 LIFO 栈式缓存，具有以下特性：

1. **连接复用** — 空闲连接可被复用，减少握手开销
2. **超时清理** — 空闲连接在超时后自动清理
3. **僵尸检测** — 检测已关闭但未归还的连接
4. **线程隔离** — 每个 worker 有独立的连接池

### Multiplex 模块

Multiplex 是多路复用层，支持 smux 和 yamux 两种协议。

#### Multiplex 子模块结构

```
multiplex/
├── smux/             # SMUX 协议
│   ├── frame.hpp     # 帧定义
│   ├── session.hpp   # 会话管理
│   └── stream.hpp    # 流管理
├── yamux/            # Yamux 协议
│   ├── frame.hpp     # 帧定义
│   ├── session.hpp   # 会话管理
│   └── stream.hpp    # 流管理
├── duct.hpp          # TCP 流管道
├── parcel.hpp        # UDP 数据报管道
└── multiplex.hpp     # 聚合头文件
```

#### 多路复用对比

| 特性 | smux | yamux |
|------|------|-------|
| 设计来源 | 简化版 SSH mux | HashiCorp 参考 |
| 流控制 | 无窗口控制 | 滑动窗口 |
| 心跳机制 | NOP 帧 | Ping/Pong |
| 适用场景 | 低延迟链路 | 高延迟链路 |

### Stealth 模块

Stealth 是 TLS 伪装层，实现多种抗审查技术。

#### Stealth 子模块结构

```
stealth/
├── reality/          # Reality TLS
│   ├── handshake.hpp # 握手实现
│   ├── scheme.hpp    # 方案定义
│   └── config.hpp    # 配置
├── shadowtls/        # ShadowTLS
│   ├── handshake.hpp # 握手实现
│   ├── scheme.hpp    # 方案定义
│   └── config.hpp    # 配置
├── restls/           # Restls
│   ├── scheme.hpp    # 方案定义
│   └── config.hpp    # 配置
├── anytls/           # AnyTLS
│   └── scheme.hpp    # 方案定义
├── trusttunnel/      # TrustTunnel
│   └── scheme.hpp    # 方案定义
├── scheme.hpp        # 方案注册
└── stealth.hpp       # 聚合头文件
```

#### 伪装技术对比

| 技术 | 特点 | 适用场景 |
|------|------|---------|
| Reality | 无需证书，完美伪装 | 无域名服务器 |
| ShadowTLS | 前置 TLS 握手 | 已有证书 |
| Restls | 流量特征伪装 | 高审查环境 |
| AnyTLS | ECH 支持 | 增强隐蔽性 |
| TrustTunnel | 可靠隧道 | 稳定连接 |

### Resolve 模块

Resolve 是 DNS 解析层，实现七阶段查询管道。

#### Resolve 子模块结构

```
resolve/
├── router.hpp        # 路由器（连接池管理）
├── dns/              # DNS 解析
│   ├── dns.hpp       # DNS 解析入口
│   ├── config.hpp    # DNS 配置
│   ├── upstream.hpp  # 上游查询
│   └── detail/       # 内部实现
│       ├── cache.hpp       # 缓存
│       ├── coalescer.hpp   # 请求合并
│       ├── rules.hpp       # 规则匹配
│       ├── transparent.hpp # 透明代理
│       ├── format.hpp      # 格式化
│       └── utility.hpp     # 工具函数
└── resolve.hpp       # 聚合头文件
```

#### DNS 七阶段管道

1. **域名规范化** — 处理域名大小写、尾随点
2. **规则匹配** — blocked/静态 IP/CNAME 重定向
3. **缓存查找** — 正向/负向/serve-stale 缓存
4. **请求合并** — 合并并发查询，减少重复请求
5. **上游查询** — UDP/TCP/DoT/DoH 多协议支持
6. **IP 黑名单过滤** — 过滤恶意 IP
7. **TTL 钳制 + 缓存存储** — 控制缓存时间

### Memory 模块

Memory 是内存管理层，实现 PMR 内存池。

#### Memory 子模块结构

```
memory/
├── container.hpp     # PMR 容器定义
├── pool.hpp          # 内存池实现
└── memory.hpp        # 聚合头文件
```

#### PMR 内存策略

Prism 使用 PMR（多态内存资源）实现热路径零堆分配：

1. **全局池** — `memory::string` 使用全局池
2. **帧竞技场** — `memory::vector<T>` 使用帧竞技场
3. **启动初始化** — `memory::system::enable_global_pooling()` 必须调用

### Trace 模块

Trace 是日志系统层，基于 spdlog 实现。

#### Trace 子模块结构

```
trace/
├── config.hpp        # 日志配置
├── trace.hpp         # 日志入口
└── trace.cpp         # 日志实现
```

#### 日志特性

1. **异步日志** — 异步队列避免阻塞主线程
2. **文件轮转** — 日志文件自动轮转
3. **多级别** — trace/debug/info/warn/error/critical
4. **格式自定义** — 可自定义日志格式

### Fault 模块

Fault 是错误码定义层，header-only 实现。

#### Fault 子模块结构

```
fault/
├── code.hpp          # 错误码枚举
├── handling.hpp      # 错误处理策略
├── compatible.hpp    # 兼容性处理
└── fault.hpp         # 聚合头文件
```

#### 双轨错误处理

Prism 采用双轨错误处理策略：

1. **热路径** — 使用 `fault::code` 枚举返回错误码，不抛异常
2. **启动/致命错误** — 使用异常层次结构

### Crypto 模块

Crypto 是加密算法层，实现各种加密操作。

#### Crypto 子模块结构

```
crypto/
├── aead.hpp          # AEAD 加密
├── blake3.hpp        # BLAKE3 哈希
├── hkdf.hpp          # HKDF 密钥派生
├── x25519.hpp        # X25519 密钥交换
├── block.hpp         # 分组加密
└── crypto.hpp        # 聚合头文件
```

#### 加密算法支持

| 算法 | 用途 | 来源 |
|------|------|------|
| AES-128/256-GCM | AEAD 加密 | BoringSSL |
| BLAKE3 | 密钥派生 | BLAKE3 库 |
| HKDF | TLS 密钥派生 | BoringSSL |
| X25519 | 密钥交换 | BoringSSL |

### Exception 模块

Exception 是异常层次结构层，header-only 实现。

#### Exception 子模块结构

```
exception/
├── deviant.hpp       # 基类异常
├── network.hpp       # 网络异常
├── protocol.hpp      # 协议异常
├── security.hpp      # 安全异常
└── exception.hpp     # 聚合头文件
```

#### 异常层次结构

```
exception::deviant（基类）
    ├── exception::network（网络异常）
    ├── exception::protocol（协议异常）
    └ exception::security（安全异常）
```

## 使用示例

### 引入模块头文件

```cpp
// 引入单个模块
#include <prism/agent.hpp>       // Agent 模块
#include <prism/memory.hpp>      // Memory 模块
#include <prism/protocol.hpp>    // Protocol 模块
#include <prism/stealth.hpp>     // Stealth 模块
#include <prism/channel.hpp>     // Channel 模块
#include <prism/multiplex.hpp>   // Multiplex 模块
#include <prism/resolve.hpp>     // Resolve 模块
#include <prism/pipeline.hpp>    // Pipeline 模块
#include <prism/recognition.hpp> // Recognition 模块

// 引入单个子模块
#include <prism/agent/session/session.hpp>
#include <prism/channel/connection/pool.hpp>
#include <prism/resolve/dns/dns.hpp>
```

### 使用 PMR 内存池

```cpp
#include <prism/memory.hpp>

// 初始化全局内存池（启动时必须调用）
psm::memory::system::enable_global_pooling();

// 使用全局池的字符串
psm::memory::string str = "hello world";

// 使用帧竞技场的向量
psm::memory::vector<int> vec;
vec.push_back(1);
vec.push_back(2);
```

### 使用连接池

```cpp
#include <prism/channel.hpp>

// 创建连接池配置
psm::connect::config pool_config;
pool_config.max_cache_per_endpoint = 32;
pool_config.max_idle_seconds = 30;
pool_config.connect_timeout_ms = 300;

// 使用连接池
auto pool = psm::connect::pool::pool(pool_config);
auto conn = pool.acquire("example.com", 443);
// 使用连接...
pool.recycle(conn);
```

### 使用 DNS 解析器

```cpp
#include <prism/resolve.hpp>

// 创建 DNS 配置
psm::resolve::dns::config dns_config;
dns_config.servers.push_back({
    .address = "8.8.8.8",
    .port = 53,
    .protocol = "udp"
});
dns_config.cache_enabled = true;

// 使用 DNS 解析器
auto resolver = psm::resolve::dns::resolver(dns_config);
auto result = resolver.resolve("example.com");
```

### 使用协议识别

```cpp
#include <prism/recognition.hpp>

// 创建识别上下文
auto context = psm::recognition::context(transport);

// 执行协议识别
auto result = psm::recognition::recognize(context);

// 根据结果分发
switch (result.detected) {
    case psm::protocol_type::socks5:
        // 处理 SOCKS5
        break;
    case psm::protocol_type::http:
        // 处理 HTTP
        break;
    case psm::protocol_type::trojan:
        // 处理 Trojan
        break;
}
```

### 使用 TLS 伪装方案

```cpp
#include <prism/stealth.hpp>

// 注册所有伪装方案（启动时必须调用）
psm::stealth::register_all_schemes();

// 创建 Reality 配置
psm::stealth::reality::config reality_config;
reality_config.dest = "www.microsoft.com:443";
reality_config.server_names = {"www.microsoft.com"};
reality_config.private_key = "base64_encoded_key";
reality_config.short_ids = {"abc123"};

// 执行 Reality 握手
auto executor = psm::stealth::scheme_executor();
auto result = executor.execute(reality_config, transport);
```

### 使用多路复用

```cpp
#include <prism/multiplex.hpp>

// 创建 SMUX 配置
psm::multiplex::smux::config smux_config;
smux_config.max_streams = 32;
smux_config.buffer_size = 4096;
smux_config.keepalive_interval_ms = 30000;

// 创建 SMUX 会话
auto session = psm::multiplex::smux::session(smux_config, transport);

// 打开新流
auto stream = session.open_stream();

// 发送数据
stream.write(data);

// 接收数据
auto received = stream.read();
```

### 使用隧道原语

```cpp
#include <prism/pipeline.hpp>

// 双向透明转发
auto tunnel_result = psm::pipeline::primitives::tunnel(
    client_transport,
    server_transport
);

// 等待隧道完成
co_await tunnel_result;
```

## 最佳实践

### 模块引入最佳实践

1. **优先使用聚合头文件** — `#include <prism/agent.hpp>` 而非逐个子头文件
2. **按需引入** — 只引入实际使用的模块，减少编译依赖
3. **更新聚合头文件** — 新增子头文件时同步更新聚合头文件

### 模块依赖最佳实践

1. **遵循层次依赖** — 上层模块依赖下层，下层不依赖上层
2. **避免循环依赖** — 模块间不应有循环依赖关系
3. **使用接口隔离** — 通过接口而非具体实现进行依赖

### 内存管理最佳实践

1. **启动时初始化** — 必须调用 `memory::system::enable_global_pooling()`
2. **使用 PMR 容器** — 热路径使用 `memory::string` 和 `memory::vector`
3. **及时归还资源** — 使用完毕后及时归还连接、内存等资源

### 错误处理最佳实践

1. **热路径用错误码** — 使用 `fault::code` 返回错误，不抛异常
2. **启动用异常** — 配置错误、初始化失败使用异常
3. **统一错误类型** — 使用统一的错误码和异常类型

### 协程使用最佳实践

1. **禁止阻塞操作** — 协程中不允许使用阻塞调用
2. **按值捕获 self** — `co_spawn` 的 lambda 按值捕获 `shared_ptr`
3. **使用 strand** — 并发访问使用 strand 保证线程安全

### 配置管理最佳实践

1. **使用强类型配置** — 配置使用 C++ 结构体而非松散 JSON
2. **合理设置默认值** — 为配置项设置合理的默认值
3. **验证配置有效性** — 加载配置后验证其有效性

### 日志使用最佳实践

1. **合理选择级别** — 根据信息重要性选择日志级别
2. **避免敏感信息** — 日志不应包含密码、密钥等敏感信息
3. **使用异步日志** — 确保日志不阻塞主线程

## 常见问题

### Q1: 如何选择引入哪些模块？

根据功能需求选择：
- 构建代理服务器：引入 `agent.hpp`
- 协议处理：引入 `protocol.hpp` + `pipeline.hpp`
- DNS 解析：引入 `resolve.hpp`
- 连接管理：引入 `channel.hpp`
- 内存管理：引入 `memory.hpp`（几乎所有模块都需要）

### Q2: 模块间有循环依赖怎么办？

循环依赖通常表明设计问题。解决方案：
1. 重新审视模块职责，拆分或合并模块
2. 使用接口隔离，将共同依赖提取为独立模块
3. 使用依赖注入而非直接依赖

### Q3: 为什么有些模块是 header-only？

header-only 模块的原因：
1. **模板类** — 模板类必须 header-only
2. **轻量功能** — 小型功能无需单独源文件
3. **配置结构** — 配置结构体定义不需要编译
4. **减少编译依赖** — header-only 减少链接依赖

### Q4: 如何扩展新模块？

扩展新模块的步骤：
1. 在 `include/prism/` 下创建新目录
2. 实现模块头文件和源文件
3. 创建聚合头文件
4. 更新 `src/CMakeLists.txt` 添加源文件
5. 更新 CLAUDE.md 文档

### Q5: 如何理解模块的层次关系？

模块层次从上到下：
- Front → Worker → Session → Recognition → Dispatch → Pipeline → Resolve → Multiplex → Channel → Stealth → Crypto → Memory → Fault/Exception

### Q6: memory 模块为什么需要启动初始化？

PMR 内存池需要初始化全局资源：
1. 分配全局内存池资源
2. 设置线程本地池
3. 注册内存统计回调
不初始化会导致 PMR 容器无法正常工作。

### Q7: 如何处理模块间的错误传递？

错误传递策略：
1. 同模块内使用错误码
2. 跨模块使用错误码或异常（根据场景）
3. 最终处理层决定是恢复还是终止

### Q8: 如何测试单个模块？

每个模块都有独立测试：
- 测试文件位于 `tests/` 目录
- 测试命名与模块对应（如 `MemoryArena.cpp`）
- 使用 `TestRunner` 框架运行测试

### Q9: 如何理解 recognition 的三阶段流水线？

三阶段流水线：
1. **Probe** — 预读 24 字节，初步判断协议类型
2. **Identify** — 深入分析 TLS ClientHello 特征
3. **Execute** — 执行识别出的伪装方案

### Q10: 如何选择 smux 还是 yamux？

选择依据：
- **smux** — 低延迟链路、简单场景
- **yamux** — 高延迟链路、需要流控制
- 默认使用 smux，复杂场景使用 yamux

## 排障指南

### 问题：模块头文件找不到

**症状**：编译时报 `#include <prism/xxx.hpp>` 找不到

**原因**：
1. 头文件路径未正确配置
2. 模块名拼写错误

**解决方案**：
1. 检查 CMake 的 include 目录配置
2. 确认模块名正确（参考模块总览表）
3. 检查聚合头文件是否存在

### 问题：内存池初始化失败

**症状**：启动时报内存池初始化错误

**原因**：
1. 未调用 `enable_global_pooling()`
2. 系统内存不足
3. 内存池配置错误

**解决方案**：
1. 确保启动时调用 `memory::system::enable_global_pooling()`
2. 检查系统可用内存
3. 检查内存池配置参数

### 问题：协议识别失败

**症状**：连接无法正确识别协议类型

**原因**：
1. 首包数据不足 24 字节
2. 协议特征未知
3. TLS ClientHello 解析失败

**解决方案**：
1. 检查首包是否完整
2. 查看日志中的识别过程
3. 检查 ClientHello 是否标准格式

### 问题：连接池连接泄露

**症状**：连接池连接数持续增长

**原因**：
1. 连接使用后未归还
2. 连接超时配置过长
3. 僵尸连接检测失效

**解决方案**：
1. 确保所有连接使用后调用 `recycle()`
2. 调整 `max_idle_seconds` 配置
3. 检查僵尸连接检测逻辑

### 问题：DNS 解析超时

**症状**：DNS 解析频繁超时

**原因**：
1. 上游 DNS 服务器不可用
2. 超时配置过短
3. 网络问题

**解决方案**：
1. 检查上游 DNS 服务器可用性
2. 增大 `timeout_ms` 配置
3. 使用多个 DNS 服务器作为备份

### 问题：多路复用连接断开

**症状**：smux/yamux 连接意外断开

**原因**：
1. 心跳超时
2. 流数量超限
3. 窗口耗尽

**解决方案**：
1. 调整心跳间隔配置
2. 增大 `max_streams` 配置
3. 增大窗口大小

### 问题：TLS 伪装失败

**症状**：Reality/ShadowTLS 握手失败

**原因**：
1. 配置参数错误
2. 伪装目标不可用
3. 密钥不匹配

**解决方案**：
1. 检查配置参数完整性
2. 确认伪装目标服务器可用
3. 检查客户端服务端密钥匹配

### 问题：协程阻塞导致性能下降

**症状**：处理延迟突然增大

**原因**：
1. 协程中使用了阻塞操作
2. strand 串行化导致等待
3. 资源竞争

**解决方案**：
1. 使用协程纯度审计技能检查
2. 减少不必要的 strand 使用
3. 优化资源访问模式

### 问题：日志输出过多影响性能

**症状**：日志占用大量 CPU/IO

**原因**：
1. 日志级别设置过低
2. 日志格式过于复杂
3. 文件输出配置不当

**解决方案**：
1. 调整 `log_level` 为 info 或更高
2. 简化日志格式
3. 减少文件输出频率

### 问题：模块编译依赖过重

**症状**：编译时间过长

**原因**：
1. 引入了过多头文件
2. 头文件包含链过长
3. 模块耦合度高

**解决方案**：
1. 只引入需要的模块头文件
2. 优化头文件包含关系
3. 使用前向声明减少依赖

## 相关页面

- [[core/instance/overview|overview]] — Agent 模块概述
- [[core/instance/architecture|architecture]] — 架构设计
- [[core/protocol/overview]] — Protocol 模块
- [[core/pipeline/overview]] — Pipeline 模块
- [[core/recognition/overview]] — Recognition 模块
- [[dev/coding/coroutine|coroutine]] — 协程技术
- [[dev/coding/pmr|pmr]] — PMR 内存池
- [[dev/testing/testing|testing]] — 测试体系
- [[dev/building/configuration|configuration]] — 配置参考