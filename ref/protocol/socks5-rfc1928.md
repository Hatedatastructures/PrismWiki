---
title: "SOCKS5 协议 RFC 1928"
category: "protocol"
type: ref
module: ref
source: "RFC 1928"
tags: [协议, socks5, 代理, 认证, tcp, udp, 隧道]
created: 2026-05-15
updated: 2026-05-17
related: [http-connect, trojan, vless, proxy-basics]
---

# SOCKS5 协议 RFC 1928

**类别**: 协议

## 概述

SOCKS5 是 SOCKS 协议的最新版本，由 IETF 发布为 RFC 1928（1996 年）。SOCKS5 是一种网络代理协议，允许客户端通过代理服务器建立到目标服务器的 TCP 或 UDP 连接。与 SOCKS4 相比，SOCKS5 增加了用户名/密码认证机制、UDP 代理支持和 IPv6 地址支持。

SOCKS5 协议在代理领域有着广泛的应用：

- **透明代理**：企业网络流量监控和控制
- **隐私保护**：隐藏客户端真实 IP 地址
- **网络穿透**：绕过防火墙和地理限制
- **负载均衡**：分布式代理服务器集群

在 Prism 项目中，SOCKS5 是核心支持的代理协议之一，提供 TCP 和 UDP 代理功能，支持无认证和用户名/密码认证两种模式。

### SOCKS 协议发展历史

SOCKS 协议的发展历程：

- **SOCKS4** (1992)：最初的 SOCKS 协议，仅支持 TCP 代理，无认证
- **SOCKS4a** (1998)：扩展支持域名解析（由代理服务器解析）
- **SOCKS5** (1996, RFC 1928)：添加认证、UDP、IPv6 支持
- **SOCKS5h**：域名解析由代理服务器执行

### SOCKS5 vs HTTP CONNECT

SOCKS5 与 HTTP CONNECT 代理的对比：

| 特性 | SOCKS5 | HTTP CONNECT |
|------|--------|--------------|
| 协议层级 | 会话层（第5层） | 应用层（第7层） |
| 认证支持 | 内置认证机制 | HTTP 认证（Basic/Digest） |
| UDP 支持 | 支持 | 不支持 |
| 目标地址 | TCP/UDP 连接 | 仅 TCP 隧道 |
| 协议开销 | 低（二进制） | 高（文本） |
| 适用场景 | 通用代理 | HTTP/Web 代理 |

### SOCKS5 的优势

SOCKS5 的主要优势：

1. **协议透明**：代理任意 TCP/UDP 流量
2. **认证机制**：内置用户名/密码认证
3. **UDP 支持**：支持 UDP 代理（适用于 DNS、游戏等）
4. **地址支持**：IPv4、IPv6、域名三种地址类型
5. **低开销**：二进制协议，效率高

## 协议详解

### 协议流程

完整的 SOCKS5 连接流程：

```
SOCKS5 TCP 代理流程：

Client                    SOCKS5 Server               Target
  |                             |                          |
  |--- 认证协商 --------------->|                          |
  |   VER: 0x05                 |                          |
  |   NMETHODS: 2               |                          |
  |   METHODS: [0x00, 0x02]     |                          |
  |                             |                          |
  |<-- 认证选择 ----------------|                          |
  |   VER: 0x05                 |                          |
  |   METHOD: 0x02 (用户名/密码) |                          |
  |                             |                          |
  |--- 认证数据 ---------------->|                          |
  |   VER: 0x01                 |                          |
  |   ULEN: 5                   |                          |
  |   UNAME: "admin"            |                          |
  |   PLEN: 8                   |                          |
  |   PASSWD: "password"        |                          |
  |                             |                          |
  |<-- 认证结果 ----------------|                          |
  |   VER: 0x01                 |                          |
  |   STATUS: 0x00 (成功)       |                          |
  |                             |                          |
  |--- 连接请求 ---------------->|                          |
  |   VER: 0x05                 |                          |
  |   CMD: 0x01 (CONNECT)       |                          |
  |   RSV: 0x00                 |                          |
  |   ATYP: 0x03 (域名)         |                          |
  |   DST.ADDR: "example.com"   |                          |
  |   DST.PORT: 443             |                          |
  |                             |                          |
  |                             |--- TCP 连接 ------------->|
  |                             |                          |
  |<-- 连接响应 ----------------|                          |
  |   VER: 0x05                 |                          |
  |   REP: 0x00 (成功)          |                          |
  |   RSV: 0x00                 |                          |
  |   ATYP: 0x01 (IPv4)         |                          |
  |   BND.ADDR: [服务器 IP]      |                          |
  |   BND.PORT: [服务器端口]     |                          |
  |                             |                          |
  |<========= TCP 数据隧道 ===================>|
  |                             |                          |
```

### 认证协商阶段

#### 客户端认证请求

客户端发送支持的认证方法：

```
认证协商请求格式：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x05)                  │
│ NMETHODS: 1 byte (方法数量)                 │
│ METHODS: NMETHODS bytes                    │
└────────────────────────────────────────────┘

认证方法列表：
┌────────────────────────────────────────────┐
│ 0x00: NO AUTHENTICATION (无认证)          │
│ 0x01: GSSAPI                               │
│ 0x02: USERNAME/PASSWORD (用户名/密码)      │
│ 0x03-0x7F: IANA 分配                       │
│ 0x80-0xFE: 私有方法                        │
│ 0xFF: NO ACCEPTABLE METHODS (无可用方法)  │
└────────────────────────────────────────────┘
```

示例请求：

```
无认证和用户名/密码请求：
05 02 00 02
│  │  │  │
│  │  │  └── 方法2: 用户名/密码
│  │  └── 方法1: 无认证
│  └── 方法数量: 2
└── SOCKS5 版本: 5
```

#### 服务器认证响应

服务器选择认证方法：

```
认证协商响应格式：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x05)                  │
│ METHOD: 1 byte (选定的方法)                 │
└────────────────────────────────────────────┘

响应示例：
┌────────────────────────────────────────────┐
│ 05 02 → 选择用户名/密码认证                 │
│ 05 00 → 选择无认证                          │
│ 05 FF → 无可用方法，拒绝连接                │
└────────────────────────────────────────────┘
```

### 用户名/密码认证（RFC 1929）

如果服务器选择了用户名/密码认证（METHOD = 0x02），客户端发送认证数据：

```
用户名/密码认证请求：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x01)                  │
│ ULEN: 1 byte (用户名长度)                   │
│ UNAME: ULEN bytes (用户名)                  │
│ PLEN: 1 byte (密码长度)                     │
│ PASSWD: PLEN bytes (密码)                   │
└────────────────────────────────────────────┘

用户名/密码认证响应：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x01)                  │
│ STATUS: 1 byte                             │
│   0x00: 成功                                │
│   其他: 失败                                │
└────────────────────────────────────────────┘

示例请求（用户名: admin, 密码: pass）：
01 05 61 64 6D 69 6E 04 70 61 73 73
│  │  └─────────────  └──────────
│  │  用户名: "admin"  密码: "pass"
│  └── 用户名长度: 5
└── 认证版本: 1
```

### 连接请求阶段

#### TCP 连接请求（CONNECT）

客户端请求建立 TCP 连接：

```
CONNECT 请求格式：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x05)                  │
│ CMD: 1 byte                                │
│   0x01: CONNECT (建立 TCP 连接)            │
│   0x02: BIND (绑定端口，用于 FTP 等)        │
│   0x03: UDP ASSOCIATE (UDP 代理)           │
│ RSV: 1 byte (必须为 0x00)                  │
│ ATYP: 1 byte (地址类型)                     │
│ DST.ADDR: variable (目标地址)               │
│ DST.PORT: 2 bytes (目标端口)                │
└────────────────────────────────────────────┘

地址类型：
┌────────────────────────────────────────────┐
│ ATYP = 0x01: IPv4                          │
│   DST.ADDR: 4 bytes                        │
│                                            │
│ ATYP = 0x03: DOMAINNAME                    │
│   DST.ADDR: 1 byte (长度) + 域名            │
│                                            │
│ ATYP = 0x04: IPv6                          │
│   DST.ADDR: 16 bytes                       │
└────────────────────────────────────────────┘
```

#### 请求示例

不同地址类型的请求示例：

```
IPv4 地址请求 (连接 192.168.1.1:443):
05 01 00 01 C0 A8 01 01 01 BB
│  │  │  │  └─────────  └──
│  │  │  │  IPv4 地址    端口: 443
│  │  │  └── ATYP: IPv4
│  │  └── RSV
│  └── CMD: CONNECT
└── VER: 5

域名请求 (连接 example.com:443):
05 01 00 03 0B 65 78 61 6D 70 6C 65 2E 63 6F 6D 01 BB
│  │  │  │  │  └─────────────────────  └──
│  │  │  │  │  域名: "example.com"      端口: 443
│  │  │  │  └── 域名长度: 11
│  │  │  └── ATYP: DOMAIN
│  │  └── RSV
│  └── CMD: CONNECT
└── VER: 5

IPv6 地址请求:
05 01 00 04 [16 bytes IPv6] [2 bytes port]
```

#### 服务器响应

服务器返回连接结果：

```
连接响应格式：
┌────────────────────────────────────────────┐
│ VER: 1 byte (必须为 0x05)                  │
│ REP: 1 byte (响应码)                        │
│ RSV: 1 byte (必须为 0x00)                  │
│ ATYP: 1 byte (地址类型)                     │
│ BND.ADDR: variable (绑定地址)               │
│ BND.PORT: 2 bytes (绑定端口)                │
└────────────────────────────────────────────┘

响应码：
┌────────────────────────────────────────────┐
│ 0x00: 成功                                  │
│ 0x01: 一般 SOCKS 服务器失败                 │
│ 0x02: 不允许的连接                          │
│ 0x03: 网络不可达                            │
│ 0x04: 主机不可达                            │
│ 0x05: 连接被拒绝                            │
│ 0x06: TTL 超时                              │
│ 0x07: 不支持的命令                          │
│ 0x08: 不支持的地址类型                      │
│ 0x09-0xFF: 未分配                           │
└────────────────────────────────────────────┘
```

成功响应示例：

```
成功响应:
05 00 00 01 [服务器 IPv4] [服务器端口]
│  │  │  │  └──────────  └──
│  │  │  │  绑定地址     绑定端口
│  │  │  └── ATYP: IPv4
│  │  └── RSV
│  └── REP: 成功
└── VER: 5
```

### UDP 代理

SOCKS5 支持 UDP 代理，使用 UDP ASSOCIATE 命令：

```
UDP 代理流程：

Client                    SOCKS5 Server
  |                             |
  |--- UDP ASSOCIATE ----------->|
  |   CMD: 0x03                 |
  |   DST.ADDR: [客户端地址]     |
  |   DST.PORT: [客户端端口]     |
  |                             |
  |<-- UDP ASSOCIATE 响应 -------|
  |   BND.ADDR: [UDP 接收地址]   |
  |   BND.PORT: [UDP 接收端口]   |
  |                             |
  |<====== UDP 数据包 =========>|
  |   (通过 UDP 接收端口收发)     |

UDP 数据包格式：
┌────────────────────────────────────────────┐
│ RSV: 2 bytes (必须为 0x0000)               │
│ FRAG: 1 byte (片段号)                       │
│ ATYP: 1 byte (地址类型)                     │
│ DST.ADDR: variable (目标地址)               │
│ DST.PORT: 2 bytes (目标端口)                │
│ DATA: variable (UDP 数据)                   │
└────────────────────────────────────────────┘

片段号：
┌────────────────────────────────────────────┐
│ 0x00: 独立数据包（不分片）                   │
│ 0x01-0xFF: 分片数据包的片段号                │
└────────────────────────────────────────────┘
```

### BIND 命令

BIND 命令用于需要服务器主动连接客户端的场景（如 FTP 数据连接）：

```
BIND 命令流程：

Client                    SOCKS5 Server               Target
  |                             |                          |
  |--- BIND 请求 -------------->|                          |
  |                             |                          |
  |<-- BIND 响应 1 -------------|                          |
  |   BND.PORT: [监听端口]       |                          |
  |                             |                          |
  |--- 通知 Target 连接 --------|                          |
  |                             |                          |
  |                             |<--- Target 连接 ---------|
  |                             |                          |
  |<-- BIND 响应 2 -------------|                          |
  |   BND.ADDR: [Target 地址]    |                          |
  |   BND.PORT: [Target 端口]    |                          |
  |                             |                          |
  |<========= 数据传输 =========>|
```

### SOCKS5 域名解析

SOCKS5 支持两种域名解析方式：

```
域名解析方式：

SOCKS5 (本地解析):
┌────────────────────────────────────────────┐
│ 客户端解析域名 → 发送 IP 地址               │
│ ATYP: IPv4/IPv6                            │
│ 代理服务器不知道原始域名                    │
└────────────────────────────────────────────┘

SOCKS5h (远程解析):
┌────────────────────────────────────────────┐
│ 客户端发送域名 → 代理服务器解析              │
│ ATYP: DOMAINNAME                           │
│ 适用于：                                    │
│   - 客户端无法解析域名                      │
│   - 需要隐藏域名查询                        │
│   - DNS 分流（使用代理服务器 DNS）          │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### 协议实现

Prism 实现完整的 SOCKS5 协议：

```cpp
// 文件: src/prism/protocol/socks5/wire.hpp
namespace psm::protocol::socks5::wire {

// 认证协商请求解析
auto parse_auth_request(std::span<const std::byte> data)
    -> outcome::result<auth_request>
{
    if (data.size() < 2) {
        return fault::code::insufficient_data;
    }
    
    auto version = static_cast<uint8_t>(data[0]);
    if (version != 0x05) {
        return fault::code::invalid_version;
    }
    
    auto num_methods = static_cast<uint8_t>(data[1]);
    if (data.size() < 2 + num_methods) {
        return fault::code::insufficient_data;
    }
    
    std::vector<uint8_t> methods;
    for (size_t i = 0; i < num_methods; ++i) {
        methods.push_back(static_cast<uint8_t>(data[2 + i]));
    }
    
    return auth_request{.methods = std::move(methods)};
}

// 认证协商响应构造
auto craft_auth_response(uint8_t method)
    -> std::vector<std::byte>
{
    return {std::byte{0x05}, std::byte{method}};
}

// 连接请求解析
auto parse_connect_request(std::span<const std::byte> data)
    -> outcome::result<connect_request>
{
    // 解析 VER, CMD, RSV, ATYP
    // 解析地址和端口
    // ...
}

// 连接响应构造
auto craft_connect_response(uint8_t rep,
                             const address& bind_addr)
    -> std::vector<std::byte>
{
    // 构造响应
    // ...
}

} // namespace psm::protocol::socks5::wire
```

### 流处理

Prism 使用状态机处理 SOCKS5 流：

```cpp
// 文件: src/prism/protocol/socks5/stream.hpp
class stream {
public:
    enum class state {
        waiting_for_auth,
        waiting_for_auth_data,
        waiting_for_request,
        tunneling
    };
    
    auto process(std::span<const std::byte> data)
        -> outcome::result<process_result>
    {
        switch (current_state_) {
            case state::waiting_for_auth:
                return handle_auth_request(data);
            case state::waiting_for_auth_data:
                return handle_auth_data(data);
            case state::waiting_for_request:
                return handle_connect_request(data);
            case state::tunneling:
                return process_result{.data = data};
        }
    }
    
private:
    state current_state_;
    // ...
};
```

### 认证处理

Prism 的 SOCKS5 认证：

```cpp
// 文件: src/prism/protocol/socks5/auth.cpp
auto select_method(const std::vector<uint8_t>& client_methods,
                   const socks5_config& config)
    -> uint8_t
{
    // 优先选择用户名/密码认证
    if (config.enable_auth) {
        for (auto method : client_methods) {
            if (method == 0x02) {
                return 0x02;
            }
        }
    }
    
    // 其次选择无认证
    if (config.enable_no_auth) {
        for (auto method : client_methods) {
            if (method == 0x00) {
                return 0x00;
            }
        }
    }
    
    // 无可用方法
    return 0xFF;
}

auto verify_credentials(std::string_view username,
                        std::string_view password,
                        const account_directory& accounts)
    -> bool
{
    auto entry = accounts.lookup(username);
    if (!entry) {
        return false;
    }
    
    return entry->password == password;
}
```

### UDP 支持

Prism 的 SOCKS5 UDP 代理：

```cpp
// 文件: src/prism/protocol/socks5/udp.cpp
auto parse_udp_packet(std::span<const std::byte> data)
    -> outcome::result<udp_packet>
{
    if (data.size() < 10) {
        return fault::code::insufficient_data;
    }
    
    auto rsv = read_u16(data, 0);
    if (rsv != 0x0000) {
        return fault::code::invalid_reserved;
    }
    
    auto frag = static_cast<uint8_t>(data[2]);
    auto atyp = static_cast<uint8_t>(data[3]);
    
    // 解析地址和端口
    auto [addr, offset] = parse_address(data, atyp, 4);
    auto port = read_u16(data, offset);
    
    auto payload = data.subspan(offset + 2);
    
    return udp_packet{
        .fragment = frag,
        .destination = addr,
        .port = port,
        .data = payload
    };
}
```

## 最佳实践

### 认证配置

1. **生产环境**：启用用户名/密码认证
2. **密码安全**：使用强密码，考虑哈希存储
3. **连接限制**：限制每用户的最大连接数
4. **日志记录**：记录认证失败事件

### 地址处理

1. **域名优先**：使用域名地址，让代理服务器解析
2. **IPv6 支持**：支持 IPv6 地址类型
3. **地址验证**：验证地址格式的有效性
4. **黑名单**：检查目标地址是否在黑名单中

### UDP 代理

1. **MTU 考虑**：UDP 数据包大小限制
2. **超时设置**：设置合理的 UDP 关联超时
3. **分片处理**：正确处理 UDP 分片（或不分片）
4. **性能优化**：使用零拷贝处理 UDP 数据

### 错误处理

1. **响应码正确**：使用正确的响应码告知客户端
2. **连接超时**：设置合理的连接超时
3. **资源清理**：正确清理失败的连接
4. **日志记录**：记录连接失败的原因

## 常见问题

### Q1: SOCKS5 与 SOCKS5h 有什么区别？

主要区别在于域名解析：

- **SOCKS5**：客户端解析域名，发送 IP 地址给代理服务器
- **SOCKS5h**：客户端发送域名给代理服务器，由代理服务器解析

SOCKS5h 的优势：
- 隐藏客户端的 DNS 查询
- 使用代理服务器的 DNS（可能更快或有不同结果）
- 绕过客户端的 DNS 限制

### Q2: UDP ASSOCIATE 的 DST.ADDR/PORT 有什么用？

UDP ASSOCIATE 请求中的 DST.ADDR 和 DST.PORT 用于：

- 原始设计：指定预期的 UDP 数据来源
- 实际应用：通常设为 0.0.0.0:0（接收任意来源）
- 某些实现：用于过滤 UDP 数据包来源

### Q3: BIND 命令什么时候使用？

BIND 命令主要用于：

- **FTP 数据连接**：FTP 主动模式需要服务器连接客户端
- **其他协议**：需要反向连接的场景

现代应用很少使用 BIND 命令。

### Q4: SOCKS5 认证安全吗？

用户名/密码认证的安全性：

- **传输安全**：明文传输，需要 TLS 加密
- **建议**：在 TLS 连接上使用 SOCKS5
- **替代方案**：使用 GSSAPI 认证或外部加密

### Q5: 如何处理 SOCKS5 请求超时？

超时处理策略：

- **认证超时**：设置认证阶段超时（如 30 秒）
- **连接超时**：设置目标连接超时（如 10 秒）
- **隧道超时**：设置数据传输超时（如 5 分钟无数据）
- **UDP 超时**：设置 UDP 关联超时（如 1 分钟）

### Q6: SOCKS5 支持多路复用吗？

标准 SOCKS5 不支持多路复用：

- 每个请求需要独立的 TCP 连接
- 解决方案：在 SOCKS5 上叠加 Smux/Yamux
- Prism 支持：SOCKS5 + Smux/Yamux 组合

### Q7: 如何检测 SOCKS5 流量？

SOCKS5 流量特征：

- **端口特征**：常用端口 1080
- **协议特征**：首字节为 0x05
- **认证特征**：认证协商阶段可识别
- **指纹识别**：通过协议解析识别

### Q8: Prism 如何处理 SOCKS5 错误？

Prism 的错误处理：

- **解析错误**：断开连接，记录日志
- **认证失败**：返回认证失败响应
- **连接失败**：返回相应的响应码
- **超时处理**：发送超时响应，清理资源

## 参考资料

- [RFC 1928 - SOCKS Protocol Version 5](https://tools.ietf.org/html/rfc1928)
- [RFC 1929 - Username/Password Authentication for SOCKS V5](https://tools.ietf.org/html/rfc1929)
- [Wikipedia - SOCKS](https://en.wikipedia.org/wiki/SOCKS)

## 相关知识

- [[ref/protocol/http-connect|HTTP CONNECT]] — HTTP CONNECT 代理
- [[ref/protocol/trojan|Trojan 协议]] — Trojan 代理
- [[ref/protocol/vless|VLESS 协议]] — VLESS 代理
- [[ref/network/tcp|TCP]] — TCP 协议基础
- [[ref/network/udp|UDP]] — UDP 协议基础