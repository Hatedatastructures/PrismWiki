---
title: "Prism 协议公共组件"
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags: [protocol, common, address, form, udp-relay, shared, prism]
related: ["[[protocol]]", "[[protocol/http]]", "[[protocol/socks5]]", "[[protocol/trojan]]", "[[protocol/vless]]"]
---

> 相关：[[protocol]] | [[protocol/http]] | [[protocol/socks5]] | [[protocol/trojan]] | [[protocol/vless]] | [[protocol/shadowsocks]]

# 协议公共组件

## 1. 模块定位

`include/prism/protocol/common/` 定义了跨协议共享的基础类型和工具函数。这些组件被 SOCKS5、Trojan、VLESS、Shadowsocks 等多个协议子模块复用，消除重复定义，确保行为一致。

```
include/prism/protocol/common/
├── address.hpp       # 跨协议地址类型（IPv4/IPv6/域名）
├── form.hpp          # 传输形态枚举（stream/datagram）
├── read.hpp          # 通用 I/O 读取工具
└── udp_relay.hpp     # UDP 中继公共逻辑
```

## 2. 地址类型（address.hpp）

### 2.1 类型定义

三种地址结构均为 **POD 类型**，可直接从协议缓冲区拷贝填充，零解析开销：

```cpp
struct ipv4_address {
    std::array<std::uint8_t, 4> bytes;    // 网络字节序
};

struct ipv6_address {
    std::array<std::uint8_t, 16> bytes;   // 网络字节序
};

struct domain_address {
    std::uint8_t length;                   // 1-255
    std::array<char, 255> value;           // 域名内容

    auto to_string(memory::resource_pointer mr) -> memory::string;
};

// 类型安全的多态变体
using address = std::variant<ipv4_address, ipv6_address, domain_address>;
```

### 2.2 各协议的复用方式

各协议通过 `using` 声明引用共享类型，而非重复定义：

```cpp
// SOCKS5 message.hpp
namespace psm::protocol::socks5 {
    using protocol::common::address;
    using protocol::common::ipv4_address;
    using protocol::common::ipv6_address;
    using protocol::common::domain_address;
}

// Trojan message.hpp — 同样引用
// VLESS message.hpp  — 同样引用
```

### 2.3 地址转字符串

```cpp
auto address_to_string(const address &addr, memory::resource_pointer mr)
    -> memory::string;
```

内部使用 `std::visit` + 编译期 `if constexpr` 分发：

| 类型 | 转换方式 |
|------|----------|
| `ipv4_address` | `inet_ntop(AF_INET, ...)` |
| `ipv6_address` | `inet_ntop(AF_INET6, ...)` |
| `domain_address` | `domain_address::to_string()` |

支持自定义 PMR 内存资源，与帧竞技场兼容。

### 2.4 设计决策

为什么使用 `std::variant` 而非继承虚函数：

- **零堆分配**：variant 在栈上内联存储，无需 vtable 指针
- **编译期分发**：`std::visit` 在编译期展开所有分支，无运行时开销
- **POD 友好**：地址结构可 `memcpy`，直接从协议缓冲区填充
- **类型安全**：variant 的 `std::get` 在编译期检查类型合法性

## 3. 传输形态（form.hpp）

### 3.1 定义

```cpp
enum class form : std::uint8_t {
    stream,     // TCP 可靠流传输
    datagram    // UDP 数据报传输
};
```

### 3.2 使用场景

| 协议 | 命令 | 映射的 form |
|------|------|-------------|
| SOCKS5 | CONNECT (0x01) | `form::stream` |
| SOCKS5 | UDP_ASSOCIATE (0x03) | `form::datagram` |
| SOCKS5 | BIND (0x02) | `form::stream` |
| Trojan | CONNECT | `form::stream` |
| Trojan | UDP_ASSOCIATE | `form::datagram` |

form 在 pipeline 层用于路由决策：

```
request.transport == form::stream   → primitives::tunnel()（TCP 双向转发）
request.transport == form::datagram → UDP relay 路径
```

### 3.3 与 pipeline 的关系

form 是协议层到 pipeline 层的**语义桥梁**。协议 handler 解析命令后设置 form，pipeline 根据 form 选择不同的转发策略，无需了解具体协议细节。

## 4. 通用读取工具（read.hpp）

### 4.1 函数签名

```cpp
// 从传输层读取至少 min_size 字节
auto read_at_least(transmission &transport,
                   span<std::byte> buffer,
                   size_t min_size)
    -> awaitable<pair<fault::code, size_t>>;

// 从 current 位置继续读取到 target 字节
auto read_remaining(transmission &transport,
                    span<std::byte> buffer,
                    size_t current,
                    size_t target)
    -> awaitable<pair<fault::code, size_t>>;
```

### 4.2 设计特点

- **协程安全**：返回 `net::awaitable`，遵循项目纯协程设计
- **增量读取**：循环调用 `async_read_some` 直到满足目标，处理 TCP 流的部分读取
- **错误传播**：通过 `fault::code` 返回错误码，不抛异常
- **EOF 检测**：`n == 0` 时返回 `fault::code::eof`

### 4.3 使用场景

被 Trojan 和 VLESS 的 relay 实现共同使用，消除重复的读取循环代码：

```cpp
// Trojan relay: 读取请求头
auto [ec, n] = co_await common::read_at_least(transport, buffer, header_size);

// VLESS relay: 补读剩余数据
auto [ec, n] = co_await common::read_remaining(transport, buffer, current, target);
```

## 5. UDP 中继公共逻辑（udp_relay.hpp）

### 5.1 缓冲区管理

```cpp
struct udp_buffers {
    memory::vector<std::byte> recv;      // 接收缓冲区
    memory::vector<std::byte> send;      // 发送缓冲区
    memory::vector<std::byte> response;  // 响应缓冲区

    explicit udp_buffers(size_t max_datagram);
};
```

所有缓冲区使用 PMR 分配器，与帧竞技场兼容，热路径零堆分配。

### 5.2 数据报转发

```cpp
auto relay_udp_packet(net::ip::udp::socket &udp_socket,
                      const net::ip::udp::endpoint &target_ep,
                      span<const byte> payload,
                      udp_buffers &buf)
    -> awaitable<tuple<fault::code, size_t, net::ip::udp::endpoint>>;
```

处理流程：

```
1. 延迟打开 socket（首次调用按目标协议族打开，后续复用）
2. async_send_to → 发送 payload 到 target_ep
3. async_receive_from → 等待单个响应
4. 返回 {错误码, 响应长度, 发送者端点}
```

### 5.3 延迟打开设计

```cpp
if (!udp_socket.is_open()) {
    udp_socket.open(target_ep.protocol(), udp_ec);
}
```

- **首次调用**：根据目标端点的协议族（IPv4/IPv6）打开 socket
- **后续调用**：复用已打开的 socket，避免每包 open/close 的系统调用开销
- **协议族自适应**：同一会话中 IPv4/IPv6 目标自动切换

### 5.4 与 SOCKS5 UDP 的区别

| 特性 | SOCKS5 UDP (stream.hpp) | common::udp_relay |
|------|------------------------|-------------------|
| 报头 | SOCKS5 RSV+FRAG+ATYP+ADDR+PORT | 无（裸 UDP） |
| 控制面 | TCP 控制连接 + 空闲超时 | 无控制面 |
| 路由 | route_callback | 直接端点 |
| 使用者 | SOCKS5 UDP_ASSOCIATE | Trojan/VLESS UDP |

Trojan 和 VLESS 的 UDP 中继在 TLS 隧道内运行，不需要 SOCKS5 报头封装，直接使用裸 UDP 转发。

## 6. 组件关系图

```
protocol::common::address  ←──── socks5::message::request
  (ipv4/ipv6/domain)       ←──── trojan::message::request
                             ←──── vless::message::request

protocol::common::form     ←──── socks5::relay (handshake 设置)
  (stream/datagram)        ←──── trojan::relay (handshake 设置)
                             ←──── pipeline 路由决策

protocol::common::read     ←──── trojan::relay (读取请求)
  (read_at_least)          ←──── vless::relay (读取请求)

protocol::common::udp_relay ←─── trojan::relay (UDP 转发)
  (relay_udp_packet)       ←──── vless::relay (UDP 转发)
```

## 7. 设计原理

- **消除重复**：四个协议的 message.hpp 共享同一套地址定义，修改一处即全局生效
- **零拷贝友好**：地址 POD 结构可直接 `memcpy`，与协议缓冲区零转换
- **PMR 兼容**：所有容器使用 PMR 分配器，支持帧竞技场批量回收
- **协程原生**：所有 I/O 函数返回 `awaitable`，无阻塞调用
- **延迟初始化**：UDP socket 延迟打开，避免不必要的系统调用

## 相关页面

- [[protocol]] — 协议模块概览
- [[protocol/tls]] — TLS 特征分析
- [[protocol/http]] — HTTP 协议
- [[protocol/socks5]] — SOCKS5 协议
- [[protocol/trojan]] — Trojan 协议
- [[protocol/vless]] — VLESS 协议
- [[memory]] — Memory 模块（PMR 容器）
