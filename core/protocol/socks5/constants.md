---
layer: core
source: I:/code/Prism/include/prism/protocol/socks5/constants.hpp
---

# SOCKS5 协议常量

定义 SOCKS5 协议的命令字、地址类型、认证方法和响应码。

## 源码位置

`I:/code/Prism/include/prism/protocol/socks5/constants.hpp`

## 命令类型

```cpp
enum class command : std::uint8_t
{
    connect = 0x01,        // 建立 TCP 连接
    bind = 0x02,           // 绑定端口等待反向连接
    udp_associate = 0x03   // 建立 UDP 关联
};
```

## 地址类型

```cpp
enum class address_type : std::uint8_t
{
    ipv4 = 0x01,           // IPv4 地址（4 字节）
    domain = 0x03,         // 域名地址（1 字节长度 + 域名）
    ipv6 = 0x04            // IPv6 地址（16 字节）
};
```

## 认证方法

```cpp
enum class auth_method : std::uint8_t
{
    no_auth = 0x00,        // 无需认证
    gssapi = 0x01,         // GSSAPI 认证
    password = 0x02,       // 用户名/密码认证
    no_acceptable_methods = 0xFF  // 无可接受的认证方法
};
```

## 响应码

```cpp
enum class reply_code : std::uint8_t
{
    succeeded = 0x00,              // 成功
    server_failure = 0x01,         // 服务器内部错误
    connection_not_allowed = 0x02, // 连接被策略拒绝
    network_unreachable = 0x03,    // 网络不可达
    host_unreachable = 0x04,       // 主机不可达
    connection_refused = 0x05,     // 连接被目标拒绝
    ttl_expired = 0x06,            // TTL 过期
    command_not_supported = 0x07,  // 不支持的命令
    address_type_not_supported = 0x08  // 不支持的地址类型
};
```

## 设计特点

- **单字节值**：直接映射 RFC 1928 规范
- **无需转换**：可直接用于网络字节序读写
- **类型安全**：使用枚举类避免魔数

## 调用链

```
protocol/socks5::wire::parse_header -> command, address_type
protocol/socks5::relay::handshake -> command::connect/udp_associate/bind
protocol/socks5::relay::async_write_error -> reply_code
```

## 依赖

- [[core/protocol/socks5/wire]] - 线级解析
- [[core/protocol/socks5/stream]] - SOCKS5 中继器