---
layer: core
source: "I:/code/Prism/include/prism/protocol/trojan/config.hpp"
title: Trojan 协议配置
tags: [protocol, trojan, config, tcp, udp]
---

# Trojan 协议配置

> 源码位置: `I:/code/Prism/include/prism/protocol/trojan/config.hpp`

## 概述

Trojan 协议配置结构，定义能力开关和 UDP 配置参数。配置用于控制协议层支持的命令类型和 UDP 行为。

## 命名空间

```cpp
namespace psm::protocol::trojan
```

## 配置结构

### config

Trojan 协议配置结构体。

```cpp
struct config
{
    bool enable_tcp = true;                 // 是否允许 CONNECT 命令
    bool enable_udp = false;                // 是否允许 UDP_ASSOCIATE 命令
    std::uint32_t udp_idle_timeout = 60;    // UDP 会话空闲超时（秒）
    std::uint32_t udp_max_datagram = 65535; // UDP 数据报最大长度
};
```

## 配置项说明

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enable_tcp` | `bool` | `true` | 是否允许 CONNECT 命令（TCP 隧道） |
| `enable_udp` | `bool` | `false` | 是否允许 UDP_ASSOCIATE 命令（UDP over TLS） |
| `udp_idle_timeout` | `uint32_t` | `60` | UDP 会话空闲超时时间（秒） |
| `udp_max_datagram` | `uint32_t` | `65535` | UDP 数据报最大长度（字节） |

## 配置特性

**生命周期**:
- 由 `agent::config` 持有
- 服务启动时初始化
- 传递给 relay 构造函数
- stream 生命周期内保持不变

**线程安全**:
- 配置结构本身是只读的
- 可安全多线程读取
- 修改配置需要重启服务

## 调用链

- [[core/protocol/trojan/relay|Relay]] - 中继器使用配置检查命令权限
- [[core/protocol/trojan/constants|Constants]] - 常量配合配置使用
- [[core/instance/config|Agent Config]] - 服务级配置持有此结构