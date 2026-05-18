---
title: "Mihomo Mux 概览"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/adapter/outbound, mihomo-Meta/listener/inbound"
tags: [mihomo, mux, smux, yamux, singmux, 多路复用]
created: 2026-05-17
updated: 2026-05-17
related: [trojan, vless, vmess, shadowsocks]
layer: ref
---

# Mihomo Mux 概览

**类别**: Mihomo 多路复用参考

## 概述

Mihomo 的 Mux（多路复用）模块提供了多种多路复用协议实现，允许在单个 TCP 连接上运行多个独立的逻辑流。多路复用可以减少连接数量、降低连接开销、提高资源效率。

Mihomo 支持的多路复用协议：

| 协议 | 来源 | 特点 |
|------|------|------|
| [[ref/mihomo/mux/smux|Smux]] | xtaci | 轻量级，简单高效 |
| [[ref/mihomo/mux/yamux|Yamux]] | HashiCorp | 功能丰富，窗口控制 |
| [[ref/mihomo/mux/singmux|Sing-Mux]] | sing-box | 现代实现，支持 Brutal |

## 配置位置

Mux 配置有两种方式：

### 1. 代理节点级 Mux（Sing-Mux）

```yaml
proxies:
  - name: "mux-node"
    type: trojan
    server: example.com
    port: 443
    password: "trojan-password"
    # Sing-Mux 配置
    smux:
      enabled: true
      protocol: smux    # smux/yamux/h2mux
      max-connections: 4
      min-streams: 4
      max-streams: 0
      padding: false
      # TCP Brutal（可选）
      brutal-opts:
        enabled: true
        up: "100 Mbps"
        down: "100 Mbps"
      statistic: false
      only-tcp: false
```

### 2. 服务端 Mux

```yaml
# Trojan 服务端 Mux
listeners:
  - name: trojan-in
    type: trojan
    listen: :443
    users:
      - password: "password"
    # Mux 配置
    mux:
      padding: false
      brutal:
        enabled: false
```

## 多路复用原理

```
多路复用原理：
┌─────────────────────────────────────────────┐
│                                             │
│  Client                    Server            │
│    |                         |              │
│    |--- TCP/TLS 连接 --------|              │
│    |   (一条物理连接)         |              │
│    |                         |              │
│    |======== Mux 帧 ========|               │
│    |                         |              │
│    |  帧1: Stream ID=1       |              │
│    |  帧2: Stream ID=2       |              │
│    |  帧3: Stream ID=1       |              │
│    |  ...                    |              │
│    |                         |              │
│  Stream 1: 代理请求A          │
│  Stream 2: 代理请求B          │
│  Stream 3: UDP 请求           │
│                                             │
└─────────────────────────────────────────────┘
```

## 协议对比

### Smux vs Yamux vs Sing-Mux

| 特性 | Smux | Yamux | Sing-Mux |
|------|------|-------|----------|
| 帧头大小 | 12 bytes | 12 bytes | 协议决定 |
| 流控制 | 简单窗口 | 窗口机制 | 窗口机制 |
| 心跳 | Keepalive | Ping/Pong | 心跳 |
| Brutal | 无 | 无 | 支持 |
| Padding | 无 | 无 | 支持 |
| H2Mux | 无 | 无 | 支持 |
| 性能 | 高 | 中 | 高 |

## 与 Prism 兼容性

### Prism Mux 支持

Prism 支持 Smux 和 Yamux 多路复用协议。

```cpp
// Prism Mux 抽象接口
// 文件: src/prism/multiplex/interface.hpp
namespace psm::multiplex {

enum class mux_protocol {
    smux,
    yamux,
    h2mux
};

class mux_session {
public:
    virtual auto open_stream()
        -> net::awaitable<mux_stream> = 0;
    
    virtual auto accept_stream()
        -> net::awaitable<mux_stream> = 0;
    
    virtual auto close()
        -> net::awaitable<void> = 0;
    
    virtual auto is_closed() const
        -> bool = 0;
};

class mux_stream : public net::stream {
public:
    virtual auto stream_id() const
        -> uint32_t = 0;
    
    virtual auto close()
        -> net::awaitable<void> = 0;
};

} // namespace psm::multiplex
```

### Smux 帧

```cpp
// Smux 帧构造
// 文件: src/prism/multiplex/smux/craft.hpp
namespace psm::multiplex::smux::craft {

// SYN 帧：创建新流
auto syn(uint32_t stream_id) -> std::vector<std::byte>;

// PSH 帧：数据推送
auto psh(uint32_t stream_id, std::span<const std::byte> data)
    -> std::vector<std::byte>;

// FIN 帧：关闭流
auto fin(uint32_t stream_id) -> std::vector<std::byte>;

// UPD 帧：窗口更新
auto upd(uint32_t stream_id, uint32_t window_delta)
    -> std::vector<std::byte>;

// KPA 帧：Keepalive
auto keepalive() -> std::vector<std::byte>;

} // namespace psm::multiplex::smux::craft
```

## 使用建议

### 协议选择

根据场景选择多路复用协议：

| 场景 | 推荐协议 |
|------|----------|
| 简单代理 | Smux |
| 高并发 | Sing-Mux (h2mux) |
| 需要 Brutal | Sing-Mux |
| 与其他客户端兼容 | Yamux |

### 性能优化

- **max-connections**：控制最大连接数（4-16）
- **max-streams**：每连接最大流数（0=无限制）
- **padding**：随机填充（增加隐蔽性）
- **brutal**：TCP Brutal 加速（需要服务端支持）

## 相关链接

- [[ref/mihomo/mux/smux|Smux]] — Smux 协议详解
- [[ref/mihomo/mux/yamux|Yamux]] — Yamux 协议详解
- [[ref/mihomo/mux/singmux|Sing-Mux]] — Sing-Mux 协议详解
- [[ref/mihomo/mux/config|Mux 配置]] — 配置参数详解
- [[ref/protocol/smux-yamux|Smux 与 Yamux]] — 协议规范参考