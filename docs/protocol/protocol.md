---
title: Protocol 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [protocol, codec, overview, design]
related: [[pipeline], [recognition], [memory/overview], [fault/overview]]
---

# Protocol 模块

Protocol 模块是 Prism 的**协议编解码层**，定义各代理协议的消息结构、编解码逻辑和数据中继器。该模块不处理会话编排（由 pipeline 负责），也不处理协议识别（由 recognition 负责），专注于协议数据的解析和构造。

## 概述

### 核心职责

- **协议分析**：通过预读数据识别协议类型，解析目标地址
- **消息编解码**：定义各协议的请求/响应结构，提供序列化/反序列化
- **TLS 特征提取**：解析 ClientHello 提取 SNI、session_id、key_share 等特征
- **数据中继**：HTTP、Trojan、VLESS、Shadowsocks 的握手和双向转发
- **共享地址类型**：跨协议通用的 IPv4/IPv6/域名地址变体

### 模块结构

```
include/prism/protocol/
├── analysis.hpp              # 协议分析器（核心入口）
├── common/                   # 跨协议共享类型
│   ├── address.hpp           # 地址类型（ipv4/ipv6/domain variant）
│   ├── form.hpp              # 传输形式（stream/datagram）
│   ├── read.hpp              # 通用读取工具
│   └── udp_relay.hpp         # UDP 中继通用逻辑
├── http/                     # HTTP 代理协议
├── socks5/                   # SOCKS5 代理协议
├── trojan/                   # Trojan 协议
├── vless/                    # VLESS 协议
├── shadowsocks/              # Shadowsocks 2022 协议
└── tls/                      # TLS 特征分析
```

### 关键组件

#### analysis（协议分析器）

无状态静态工具类，提供协议探测和地址解析。

```cpp
enum class protocol_type { unknown, http, socks5, trojan, vless, shadowsocks, tls };

struct analysis {
    struct target {
        memory::string host;    // 目标主机
        memory::string port;    // 目标端口（默认 "80"）
        bool positive{false};   // 是否为正向代理请求
    };

    static auto resolve(const http::proxy_request &req, ...) -> target;
    static auto resolve(std::string_view host_port, ...) -> target;
    static auto detect_tls(std::string_view peek_data) -> protocol_type;
};
```

target.positive 标志用于路由决策：true 表示正向代理，false 表示普通或反向代理。

#### TLS 特征分析

从 ClientHello 提取特征，用于 recognition 模块的伪装方案识别：

- `tls/types.hpp` — client_hello_features 结构体
- `tls/feature_bitmap.hpp` — 32 位特征位图，支持 Tier 0 快速检测
- `tls/signal.hpp` — ClientHello 解析入口

#### 协议子模块

每个协议子模块包含标准组件集：

| 组件 | 用途 | 示例 |
|------|------|------|
| message.hpp | 消息结构定义 | 请求/响应字段 |
| config.hpp | 协议配置 | 能力开关、参数 |
| constants.hpp | 协议常量 | 命令类型、版本号 |
| relay.hpp | 数据中继器 | 握手 + 转发 |
| format.hpp | 编解码 | 序列化/反序列化 |

### 数据流

```
recognition::probe::detect(peek_data)  ← 使用 analysis 进行协议探测
         │
         ▼
pipeline::http/socks5/trojan/...       ← pipeline 调用 protocol 的 relay 和 format
         │
         ▼
protocol::*::relay                     ← 握手、认证、编解码
protocol::*::format                    ← 消息序列化/反序列化
```

## 详细设计

### 模块职责与边界

#### 职责

- **协议分析与探测**：`analysis` 结构体提供静态方法，从预读数据识别协议类型，解析目标地址
- **协议消息编解码**：定义各协议的请求/响应消息结构，提供序列化/反序列化
- **TLS ClientHello 解析**：提取 SNI、key_share、session_id 等特征字段，构建特征位图
- **数据中继**：各协议的 relay 实现握手和双向数据转发

#### 边界

- **不负责**：会话生命周期管理（由 agent/session 处理）
- **不负责**：协议识别的编排（由 recognition 模块处理，但提供 analysis 和 tls 原语）
- **不负责**：路由决策和 DNS 解析（由 resolve 模块处理）
- **不负责**：连接池管理（由 channel 模块处理）

### 关键类和接口

#### `protocol::analysis` — 协议分析器

**文件**: `analysis.hpp`

```cpp
namespace psm::protocol {
    enum class protocol_type {
        unknown, http, socks5, trojan, vless, shadowsocks, tls
    };

    inline auto to_string_view(protocol_type type) -> std::string_view;

    struct analysis {
        struct target {
            explicit target(memory::resource_pointer mr = memory::current_resource());
            memory::string host;
            memory::string port;      // 默认 "80"
            bool positive{false};     // 正向代理标志
        };

        static auto resolve(const http::proxy_request &req, memory::resource_pointer mr = nullptr) -> target;
        static auto resolve(std::string_view host_port, memory::resource_pointer mr = nullptr) -> target;
        static auto detect_tls(std::string_view peek_data) -> protocol_type;
    };
}
```

**设计原理**：
- 所有方法均为 `static`，无状态设计，线程安全
- `target` 使用 PMR 分配器，与帧竞技场兼容
- `detect_tls` 用于区分 TLS 内部承载的协议（HTTPS/Trojan/VLESS 等）
- 解析失败时返回合理默认值，不抛异常

#### `protocol::common::address` — 共享地址类型

**文件**: `common/address.hpp`

```cpp
namespace psm::protocol::common {
    struct ipv4_address { std::array<std::uint8_t, 4> bytes; };
    struct ipv6_address { std::array<std::uint8_t, 16> bytes; };
    struct domain_address {
        std::uint8_t length;
        std::array<char, 255> value;
    };
    using address = std::variant<ipv4_address, ipv6_address, domain_address>;
}
```

POD 类型设计，可直接从协议缓冲区拷贝填充。各协议通过 `using` 声明引用，消除重复定义。

#### TLS ClientHello 特征结构

**文件**: `tls/types.hpp`, `tls/feature_bitmap.hpp`

```cpp
namespace psm::protocol::tls {
    struct client_hello_features {
        memory::string server_name;           // SNI
        memory::vector<std::uint8_t> session_id;
        bool has_x25519{false};
        std::array<std::uint8_t, 32> x25519_key{};
        memory::vector<std::uint16_t> versions;
        std::array<std::uint8_t, 32> random{};
        memory::vector<std::uint8_t> raw_hs_msg;
        memory::vector<std::byte> raw_record;
    };

    enum feature_bit : std::uint32_t {
        has_sni = 1 << 0,
        sni_matched_config = 1 << 1,
        has_x25519 = 1 << 2,
        has_full_session_id = 1 << 3,
        reality_marker_01_08_02 = 1 << 4,
    };
}
```

特征位图将 ClientHello 特征编码为 32 位整数，Tier 0 检测仅需一次位运算。

#### HTTP 解析器

**文件**: `http/parser.hpp`

```cpp
namespace psm::protocol::http {
    struct proxy_request {
        std::string_view method;
        std::string_view target;
        std::string_view host;
        std::string_view authorization;
        std::string_view version;
        std::size_t req_line_end{0};
        std::size_t header_end{0};
    };

    auto parse_proxy_request(std::string_view raw_data, proxy_request &out) -> fault::code;
}
```

零堆分配设计，所有结果以 `string_view` 指向原始缓冲区。仅提取代理决策所需字段，不处理 body。

### 文件结构

#### 源文件 (`src/prism/protocol/`)

```
protocol/
├── analysis.cpp              # 协议分析器实现
├── http/parser.cpp           # HTTP 解析器实现
├── http/relay.cpp            # HTTP 中继器实现
├── trojan/format.cpp         # Trojan 编解码实现
├── trojan/relay.cpp          # Trojan 中继器实现
├── vless/format.cpp          # VLESS 编解码实现
├── vless/relay.cpp           # VLESS 中继器实现
├── shadowsocks/format.cpp    # SS2022 编解码实现
├── shadowsocks/relay.cpp     # SS2022 中继器实现
├── shadowsocks/datagram.cpp  # SS2022 UDP 数据报实现
└── tls/signal.cpp            # TLS ClientHello 解析实现
```

## 模块依赖

### 依赖的模块

| 模块 | 用途 |
|------|------|
| `memory` | PMR 容器（memory::string, memory::vector） |
| `fault` | 错误码定义 |
| `agent/account` | 账户目录（HTTP/Trojan 认证） |
| `channel/transport` | 传输层抽象（relay 使用） |

### 被依赖的模块

| 模块 | 用途 |
|------|------|
| `recognition` | 使用 `analysis` 和 `tls` 进行协议探测和特征分析 |
| `pipeline` | 使用各协议的 `relay` 和 `format` 进行协议处理 |

### 文件依赖关系

```
analysis.hpp
├── memory/container.hpp
└── http/parser.hpp

tls/types.hpp
└── memory/container.hpp

tls/signal.hpp
├── tls/types.hpp
├── memory/container.hpp
└── fault/code.hpp

tls/feature_bitmap.hpp
└── tls/types.hpp
```

## 设计原理

- **无状态设计**：`analysis` 所有方法均为 `static`，不维护内部状态：线程安全、可测试、无实例开销
- **零拷贝设计**：HTTP 解析器使用 `string_view` 指向原始缓冲区；地址类型使用 POD 结构，可直接从协议缓冲区拷贝；TLS 特征结构使用 PMR 容器，与帧竞技场兼容
- **共享类型复用**：`common/address.hpp` 定义跨协议通用的地址类型，消除四个 `message.hpp` 中的重复定义
- **特征位图优化**：TLS 特征使用 32 位位图压缩，Tier 0 检测仅需一次位运算，预留 16 位供未来扩展

## 相关链接

- [[protocol/http]] — HTTP 协议详解
- [[protocol/socks5]] — SOCKS5 协议详解
- [[protocol/trojan]] — Trojan 协议详解
- [[protocol/vless]] — VLESS 协议详解
- [[protocol/shadowsocks]] — Shadowsocks 协议详解
- [[pipeline]] — Pipeline 模块
- [[recognition]] — Recognition 模块
- [[memory]] — Memory 模块
