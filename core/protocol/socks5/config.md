---
layer: core
source: include/prism/protocol/socks5/config.hpp
title: SOCKS5 协议配置
tags: [protocol, socks5, config, tcp, udp, auth]
---

# SOCKS5 协议配置

定义 SOCKS5 协议的能力开关和 UDP 中继参数。

## 源码位置

`include/prism/protocol/socks5/config.hpp`

## 结构定义

```cpp
struct config
{
    bool enable_tcp = true;              // 允许 CONNECT 命令
    bool enable_udp = true;              // 允许 UDP_ASSOCIATE 命令
    bool enable_bind = false;            // 允许 BIND 命令（通常禁用）
    std::uint16_t udp_bind_port = 0;     // UDP relay 绑定端口（0 自动分配）
    std::uint32_t udp_idle_timeout = 60; // UDP 会话空闲超时（秒）
    std::uint32_t udp_max_datagram = 65535; // UDP 数据报最大长度
    bool enable_auth = false;            // 启用用户名/密码认证
};
```

## 字段说明

| 字段 | 默认值 | 说明 |
|---|---|---|
| `enable_tcp` | true | TCP 隧道开关，最常用场景默认启用 |
| `enable_udp` | true | UDP 中继开关，代理转发默认启用 |
| `enable_bind` | false | BIND 命令开关，降低安全风险默认禁用 |
| `udp_bind_port` | 0 | UDP 端口，0 由系统自动分配 |
| `udp_idle_timeout` | 60 | 空闲超时秒数，防止资源泄漏 |
| `udp_max_datagram` | 65535 | 最大数据报长度 |
| `enable_auth` | false | RFC 1929 认证开关 |

## 设计原则

- **最小权限**：BIND 默认禁用降低安全风险
- **默认可用**：TCP/UDP 默认启用满足常用场景
- **只读配置**：运行时不可修改，需重启生效

## 配置管理

- 由 [[core/instance/config|config]] 持有
- 随服务启动时初始化
- 传递给 `socks5::relay` 构造函数

## 调用链

```
agent/config -> protocol/socks5::config
agent/worker/worker -> protocol/socks5::make_relay(config)
protocol/socks5::relay::handshake -> config.enable_tcp/enable_udp
protocol/socks5::relay::async_associate -> config.udp_bind_port/udp_idle_timeout
```

## 依赖

- [[core/instance/config|config]] - Agent 全局配置
- [[core/protocol/socks5/stream]] - SOCKS5 中继器

## 注意事项

- 配置在 relay 构造时传入
- 运行时不可修改
- 修改后需重启服务生效