---
layer: core
source: "I:/code/Prism/include/prism/protocol/vless/config.hpp"
---

# VLESS 协议配置

> 源码位置: `I:/code/Prism/include/prism/protocol/vless/config.hpp`

## 概述

VLESS 协议配置结构，定义 UDP 能力开关和参数。用户认证已统一到 agent::authentication 中，VLESS 通过 account::directory 查询 UUID 凭证。

## 命名空间

```cpp
namespace psm::protocol::vless
```

## 配置结构

### config

VLESS 协议配置结构体。

```cpp
struct config
{
    bool enable_udp = false;                // 是否允许 UDP 命令
    std::uint32_t udp_idle_timeout = 60;    // UDP 会话空闲超时（秒）
    std::uint32_t udp_max_datagram = 65535; // UDP 数据报最大长度
};
```

## 配置项说明

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enable_udp` | `bool` | `false` | 是否允许 UDP 命令（UDP over TLS） |
| `udp_idle_timeout` | `uint32_t` | `60` | UDP 会话空闲超时时间（秒） |
| `udp_max_datagram` | `uint32_t` | `65535` | UDP 数据报最大长度（字节） |

## 配置特性

**生命周期**:
- 由 `agent::config` 持有
- 服务启动时初始化
- 传递给 relay 构造函数
- 生命周期内保持不变

**线程安全**:
- 配置结构本身是只读的
- 可安全多线程读取

## 认证机制

VLESS 协议的 UUID 认证不在此配置中处理，而是通过:
- `agent::authentication` - 统一认证模块
- `account::directory` - UUID 凭证查询

## 调用链

- [[core/protocol/vless/relay|Relay]] - 中继器使用配置控制 UDP 权限
- [[core/protocol/vless/constants|Constants]] - 常量配合配置使用
- [[agent/config|Agent Config]] - 服务级配置持有此结构
- [[agent/account/directory|Account Directory]] - UUID 凭证查询