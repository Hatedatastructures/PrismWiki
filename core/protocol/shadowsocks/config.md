---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/config.hpp"
---

# SS2022 协议配置

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/config.hpp`

## 概述

SS2022 协议配置结构，定义 PSK、加密方法选择、TCP/UDP 开关和时间戳窗口参数。

## 命名空间

```cpp
namespace psm::protocol::shadowsocks
```

## 配置结构

### config

SS2022 协议配置结构体。

```cpp
struct config
{
    memory::string psk;                    // Base64 编码的 PSK
    memory::string method;                 // 加密方法名（可选）
    bool enable_tcp = true;                // 是否启用 TCP 代理
    bool enable_udp = false;               // 是否启用 UDP 代理
    std::int64_t timestamp_window = 30;    // 时间戳重放窗口（秒）
    std::int64_t salt_pool_ttl = 60;       // Salt 池 TTL（秒）
    std::uint32_t udp_idle_timeout = 60;   // UDP 会话空闲超时（秒）
};
```

## 配置项说明

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `psk` | `memory::string` | - | Base64 编码的 PSK |
| `method` | `memory::string` | - | 加密方法名（可选，自动推断） |
| `enable_tcp` | `bool` | `true` | 是否启用 TCP 代理 |
| `enable_udp` | `bool` | `false` | 是否启用 UDP 代理 |
| `timestamp_window` | `std::int64_t` | `30` | 时间戳重放窗口（秒） |
| `salt_pool_ttl` | `std::int64_t` | `60` | Salt 池 TTL（秒） |
| `udp_idle_timeout` | `std::uint32_t` | `60` | UDP 会话空闲超时（秒） |

## PSK 规范

**解码后长度**:
- 16 字节 → AES-128-GCM
- 32 字节 → AES-256-GCM 或 ChaCha20-Poly1305

**自动推断规则**:
- 16B → aes-128
- 32B → aes-256（默认）
- 显式设置 `method` 支持 ChaCha20

## 调用链

- [[core/protocol/shadowsocks/relay|Relay]] - 中继器使用配置
- [[core/protocol/shadowsocks/constants|Constants]] - 方法名常量
- [[core/protocol/shadowsocks/format|Format]] - PSK 解码
- [[core/protocol/shadowsocks/salts|Salts]] - 使用 salt_pool_ttl