---
title: Hysteria
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, hysteria]
---
# Hysteria 协议

Hysteria 是基于 QUIC 的高性能代理协议。

## 协议概述

Hysteria v1 特性：
- 基于 QUIC 协议
- Brutal 拥塞控制算法
- 支持端口跳跃
- 支持混淆
- 高性能 UDP 传输

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/hysteria.go` | Hysteria 适配器 |
| `transport/hysteria/core/client.go` | Hysteria 核心客户端 |
| `transport/hysteria/congestion/brutal.go` | Brutal 拥塞控制 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "hysteria-proxy"
    type: hysteria
    server: server.example.com
    port: 443
    auth: your-auth-string
    up: "100 Mbps"
    down: "200 Mbps"
    obfs: obfs-string
```

### 端口跳跃

```yaml
proxies:
  - name: "hysteria-hop"
    type: hysteria
    server: server.example.com
    ports: 10000-50000
    protocol: udp
    auth: your-auth-string
    up: "100 Mbps"
    down: "200 Mbps"
    hop-interval: 10
```

### 完整配置

```yaml
proxies:
  - name: "hysteria-full"
    type: hysteria
    server: server.example.com
    port: 443
    auth-str: your-auth-string
    obfs: obfs-password
    up: "100 Mbps"
    down: "200 Mbps"
    sni: server.example.com
    skip-cert-verify: false
    alpn:
      - hysteria
    recv-window-conn: 15728640
    recv-window: 67108864
    disable-mtu-discovery: false
    fast-open: true
```

## 带宽配置

带宽可使用以下格式：
- `100 Mbps` - Mbps 单位
- `1 Gbps` - Gbps 单位
- `12500000` - Bps 单位（字节/秒）

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `hysteria` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口（ports 时可选） |
| `ports` | string | 否 | 端口范围（跳跃） |
| `auth` | string | 否 | Base64 认证 |
| `auth-str` | string | 否 | 字符串认证 |
| `up` | string | 是 | 上传带宽 |
| `down` | string | 是 | 下载带宽 |
| `obfs` | string | 否 | 混淆密码 |
| `sni` | string | 否 | TLS SNI |
| `fast-open` | bool | 否 | 快速打开 |

## 协议背景

Hysteria 是由 `apernet` 开发的基于 QUIC 的高性能代理协议。它专为高延迟、高丢包率的网络环境设计，通过自定义的 Brutal 拥塞控制算法，在恶劣网络条件下依然能保持高吞吐量。

### 设计理念

Hysteria 的核心设计理念是"**带宽优先**"——不同于传统的基于延迟或丢包率的拥塞控制，Hysteria 允许用户手动指定上行/下行带宽，拥塞控制算法直接基于这些值进行速率控制。

### v1 vs v2 对比

| 特性 | Hysteria v1 | Hysteria v2 |
|------|-------------|-------------|
| 认证方式 | auth / auth-str | auth-str + 用户系统 |
| 协议版本 | QUIC v1 (IETF) | QUIC v2 (RFC 9369) |
| 传输模式 | UDP 流 | 统一流 + 可选 UDP |
| 混淆 | 固定 obfs | 可配置 obfs |
| 端口跳跃 | 支持 | 支持（增强） |
| 带宽配置 | up/down | up/down |
| 拥塞控制 | Brutal | Brutal（增强） |
| NAT 类型检测 | 不支持 | 支持 |
| 服务质量 (QoS) | 不支持 | 支持 |

## Hysteria v1 架构

### QUIC 之上的传输

```
┌─────────────────────────────────────────┐
│           Hysteria v1 协议栈              │
├─────────────────────────────────────────┤
│  代理数据                                  │
│       │                                   │
│       ▼                                   │
│  Hysteria 协议层                           │
│  ├─ TCP 流代理（QUIC Stream）              │
│  └─ UDP 代理（QUIC Datagram）              │
│       │                                   │
│       ▼                                   │
│  QUIC 传输层                              │
│  ├─ Brutal 拥塞控制                        │
│  └─ TLS 1.3 加密                          │
│       │                                   │
│       ▼                                   │
│  UDP（底层传输）                           │
└─────────────────────────────────────────┘
```

### Brutal 拥塞控制

Brutal 是 Hysteria 的核心拥塞控制算法，工作原理如下：

```
┌──────────────────────────────────────────┐
│          Brutal CC 工作流程                 │
├──────────────────────────────────────────┤
│                                           │
│  用户配置：up = 100 Mbps                   │
│              down = 200 Mbps               │
│                                           │
│  发送端：                                   │
│  ├─ 按 up 带宽持续发送                     │
│  ├─ 不检测网络拥塞信号                     │
│  └─ 由接收端控制速率                        │
│                                           │
│  接收端：                                   │
│  ├─ 测量实际接收速率                        │
│  ├─ 如果 < down → 不干预                   │
│  └─ 如果 > down → 丢弃多余包               │
│                                           │
│  反馈回路：                                  │
│  └─ 接收端通过 ACK 反馈实际接收量            │
│     发送端据此调整发送速率                   │
└──────────────────────────────────────────┘
```

Brutal 与传统拥塞控制的区别：
- **传统**：通过 RTT 增长或丢包推断拥塞，被动减速
- **Brutal**：用户指定带宽，主动按速率发送，由接收端丢包控制

### 认证机制

v1 支持两种认证方式：

| 方式 | 配置字段 | 说明 |
|------|---------|------|
| 字符串认证 | `auth-str` | 直接使用字符串 |
| Base64 认证 | `auth` | Base64 编码的认证字符串 |

认证流程：
1. 客户端在 QUIC 连接建立时发送认证信息
2. 服务器验证认证信息
3. 验证失败则关闭连接

### 端口跳跃（Port Hopping）

用于对抗基于端口的流量检测：

```yaml
ports: 10000-50000   # 端口范围
hop-interval: 10     # 切换间隔（秒）
```

客户端定期在端口范围内随机选择端口发送探测包，保持多个 UDP 端口活跃。

## Hysteria v2 改进

### 协议变更

v2 相比 v1 的主要改进：

1. **QUIC v2**：使用 RFC 9369 定义的 QUIC v2，提供更好的头部保护
2. **统一认证**：合并 `auth` 和 `auth-str`，统一使用 `auth-str`
3. **连接复用优化**：改进流复用效率
4. **NAT 类型检测**：自动检测 NAT 类型并优化传输

### 配置差异

```yaml
# Hysteria v1
type: hysteria
auth-str: "password"
up: "100 Mbps"
down: "200 Mbps"

# Hysteria v2
type: hysteria2
auth: "password"          # 字段名简化
up: "100 Mbps"
down: "200 Mbps"
```

## 错误处理

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| QUIC 握手失败 | TLS 协商错误 | 连接断开 |
| 认证失败 | auth-str 不匹配 | 服务器拒绝连接 |
| 带宽超限 | 超过 up/down 配置 | Brutal 丢包控制 |
| 端口跳跃失效 | 所有端口不可达 | 标记连接断开 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Hysteria 协议实现 |
| Pipeline | 支持 | QUIC 流 |
| Multiplex | 完全兼容 | QUIC 内置多路复用 |

## 相关文档

- [[hysteria2]] - Hysteria v2 协议
- [[tuic]] - TUIC 协议
- [[dev/debugging/udp|udp]] - UDP 传输