---
title: "Mihomo Transport 概览"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport"
tags: [mihomo, transport, 传输层, 插件, 混淆]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, trojan, vmess, vless]
layer: ref
---

# Mihomo Transport 概览

**类别**: Mihomo 配置参考

## 概述

Mihomo（原 Clash.Meta）的 Transport 模块提供了多种传输层协议和混淆插件，用于在代理协议之上建立额外的传输层。这些传输层可以实现流量混淆、连接复用、协议伪装等功能，增强代理的隐蔽性和稳定性。

Transport 模块位于 `transport/` 目录下，包含以下主要组件：

| 插件 | 功能 | 适用协议 |
|------|------|----------|
| [[ref/mihomo/transport/shadowtls|ShadowTLS]] | TLS 伪装 | Shadowsocks |
| [[ref/mihomo/transport/restls|RestLS]] | TLS 指纹伪装 | Shadowsocks |
| [[ref/mihomo/transport/v2ray-plugin|V2ray-Plugin]] | WebSocket + HTTP Upgrade | Shadowsocks, VMess |
| [[ref/mihomo/transport/simple-obfs|Simple-Obfs]] | HTTP/TLS 混淆 | Shadowsocks |
| [[ref/mihomo/transport/gost-plugin|Gost-Plugin]] | WebSocket + Mux | Shadowsocks |
| [[ref/mihomo/transport/kcptun|Kcptun]] | KCP + UDP 传输 | Shadowsocks |
| [[ref/mihomo/transport/gun|Gun (gRPC)]] | HTTP/2 gRPC 传输 | VLESS, Trojan |

## Transport 层次结构

```
代理协议层次：
┌─────────────────────────────────────────────┐
│ 应用层：代理协议（Shadowsocks/Trojan/VLESS） │
├─────────────────────────────────────────────┤
│ 传输层：Transport 插件（可选）               │
│   - WebSocket                               │
│   - gRPC (Gun)                              │
│   - HTTP/TLS Obfuscation                    │
│   - KCP (UDP)                               │
├─────────────────────────────────────────────┤
│ 加密层：TLS/Reality/ECH（可选）              │
├─────────────────────────────────────────────┤
│ 网络层：TCP/UDP                             │
└─────────────────────────────────────────────┘
```

## 配置结构

Transport 配置通常作为代理节点的子配置：

```yaml
proxies:
  - name: "node-name"
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    # Transport 插件配置
    plugin: v2ray-plugin
    plugin-opts:
      mode: websocket
      tls: true
      path: /path
      host: server.com
```

## 与 Prism 兼容性

Prism 目前支持的 Transport：

| Transport | Prism 支持 | 实现状态 |
|-----------|-----------|----------|
| WebSocket | 支持 | 完整实现 |
| gRPC (Gun) | 支持 | 完整实现 |
| HTTP Obfs | 支持 | 完整实现 |
| TLS Obfs | 支持 | 完整实现 |
| ShadowTLS | 支持 | 完整实现 |
| RestLS | 部分 | 基础实现 |
| Kcptun | 计划中 | 待实现 |

### Prism Transport 实现

```cpp
// Prism Transport 抽象接口
// 文件: src/prism/transport/interface.hpp
namespace psm::transport {

class transport_layer {
public:
    virtual auto connect(
        net::tcp::socket& underlying,
        const config& cfg
    ) -> net::awaitable<void> = 0;
    
    virtual auto read(
        std::span<std::byte> buffer
    ) -> net::awaitable<size_t> = 0;
    
    virtual auto write(
        std::span<const std::byte> data
    ) -> net::awaitable<size_t> = 0;
    
    virtual auto close() -> net::awaitable<void> = 0;
};

} // namespace psm::transport
```

## Transport 选择指南

### 根据场景选择

| 场景 | 推荐 Transport | 原因 |
|------|---------------|------|
| 基础伪装 | simple-obfs (HTTP) | 简单有效 |
| TLS 伪装 | v2ray-plugin (ws+tls) | 伪装为 HTTPS |
| 高隐蔽性 | ShadowTLS | TLS 指纹伪装 |
| 网络不稳定 | Kcptun | UDP + KCP |
| HTTP/2 环境 | Gun (gRPC) | 原生 HTTP/2 |
| 连接复用 | gost-plugin (ws+mux) | 多路复用 |

### 性能对比

| Transport | 连接开销 | 延迟 | 吞吐量 | 稳定性 |
|-----------|---------|------|--------|--------|
| WebSocket | 低 | 中 | 高 | 高 |
| gRPC | 中 | 低 | 高 | 高 |
| HTTP Obfs | 低 | 中 | 中 | 高 |
| TLS Obfs | 中 | 中 | 中 | 高 |
| ShadowTLS | 高 | 高 | 中 | 高 |
| Kcptun | 低 | 低 | 高 | 中 |

## 相关链接

- [[ref/mihomo/transport/shadowtls|ShadowTLS]] — TLS 伪装插件
- [[ref/mihomo/transport/restls|RestLS]] — TLS 指纹伪装
- [[ref/mihomo/transport/v2ray-plugin|V2ray-Plugin]] — WebSocket 插件
- [[ref/mihomo/transport/simple-obfs|Simple-Obfs]] — HTTP/TLS 混淆
- [[ref/mihomo/transport/gost-plugin|Gost-Plugin]] — WebSocket + Mux
- [[ref/mihomo/transport/kcptun|Kcptun]] — KCP UDP 传输
- [[ref/mihomo/transport/gun|Gun (gRPC)]] — HTTP/2 gRPC
- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用