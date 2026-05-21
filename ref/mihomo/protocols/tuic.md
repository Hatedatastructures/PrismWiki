---
title: TUIC
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, tuic]
---
# TUIC 协议

TUIC 是基于 QUIC 的 UDP 优化代理协议。

## 协议概述

TUIC 特性：
- 基于 QUIC 的 UDP 传输
- 支持多路复用
- 支持 UDP over Stream 模式
- 版本：v4（Token）和 v5（UUID+Password）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/tuic.go` | TUIC 适配器 |
| `transport/tuic/tuic.go` | TUIC 协议实现 |

## YAML 配置示例

### TUIC v4 配置

```yaml
proxies:
  - name: "tuic-v4"
    type: tuic
    server: server.example.com
    port: 443
    token: your-token
    congestion-controller: cubic
    udp-relay-mode: quic
    reduce-rtt: true
```

### TUIC v5 配置

```yaml
proxies:
  - name: "tuic-v5"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    congestion-controller: bbr
    udp-relay-mode: native
```

### UDP over Stream

```yaml
proxies:
  - name: "tuic-uos"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    udp-over-stream: true
    udp-over-stream-version: 2
```

### 完整配置

```yaml
proxies:
  - name: "tuic-full"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    congestion-controller: bbr
    udp-relay-mode: quic
    reduce-rtt: true
    request-timeout: 8000
    heartbeat-interval: 10000
    max-open-streams: 100
    max-udp-relay-packet-size: 1252
    fast-open: true
    skip-cert-verify: false
    alpn:
      - h3
    sni: server.example.com
```

## 拥塞控制

| Controller | 描述 |
|------------|------|
| `cubic` | CUBIC 拥塞控制 |
| `bbr` | BBR 拥塞控制 |
| `new_reno` | New Reno |

## UDP Relay 模式

| Mode | 描述 |
|------|------|
| `quic` | QUIC Datagram |
| `native` | 原生 UDP |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `tuic` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `token` | string | 否 | v4 Token |
| `uuid` | string | 否 | v5 UUID |
| `password` | string | 否 | v5 Password |
| `congestion-controller` | string | 否 | 拥塞控制 |
| `udp-relay-mode` | string | 否 | UDP 模式 |
| `reduce-rtt` | bool | 否 | 0-RTT |
| `fast-open` | bool | 否 | 快速打开 |
| `ip` | string | 否 | 直接指定服务器 IP（绕过 DNS） |
| `disable-sni` | bool | 否 | 禁用 SNI（需配合 skip-cert-verify） |
| `recv-window-conn` | int | 否 | QUIC 流接收窗口 |
| `recv-window` | int | 否 | QUIC 连接接收窗口 |
| `disable-mtu-discovery` | bool | 否 | 禁用 MTU 发现 |
| `max-datagram-frame-size` | int | 否 | 最大数据帧大小（默认 1252） |

## 协议背景

TUIC（Total UDP-based Internet Acceleration Cache）是由 `EAimTY` 开发的基于 QUIC 的代理协议。它专门针对高延迟、高丢包率的网络环境进行了优化，是目前最流行的基于 QUIC 的代理协议之一。

### 设计目标

- **极致的 UDP 支持**：原生基于 QUIC Datagram 传输 UDP 流量
- **低延迟**：利用 QUIC 的 0-RTT 握手快速建立连接
- **高吞吐量**：支持 BBR/CUBIC 等现代拥塞控制
- **多路复用**：QUIC 内置的多流复用，避免队头阻塞

### 协议架构

```
┌─────────────────────────────────────────────┐
│              TUIC 协议栈                       │
├─────────────────────────────────────────────┤
│  代理层                                      │
│  ├─ TCP 代理（QUIC Stream）                   │
│  ├─ UDP 代理（QUIC Datagram）                 │
│  └─ UDP over Stream（可选）                   │
│       │                                       │
│       ▼                                       │
│  QUIC 传输层                                 │
│  ├─ 多路复用（Stream）                        │
│  ├─ 拥塞控制（BBR/CUBIC/New Reno）            │
│  └─ TLS 1.3 加密                             │
│       │                                       │
│       ▼                                       │
│  UDP（底层传输）                              │
└─────────────────────────────────────────────┘
```

## 版本对比：v4 vs v5

| 特性 | TUIC v4 | TUIC v5 |
|------|---------|---------|
| 认证方式 | Token | UUID + Password |
| 认证流程 | 单令牌验证 | HMAC-SHA256 认证 |
| 协议设计 | 简单令牌匹配 | 完整的认证握手 |
| 向后兼容 | 不兼容 | 不兼容 |
| 推荐场景 | 简单部署 | 多用户系统 |

### v4 Token 认证

客户端在 QUIC 连接建立后发送一个固定长度的 Token：

```
Token 格式：
+----------------------------------+
|          Token (固定长度)          |
|   由服务器预生成并分发给客户端      |
+----------------------------------+
```

### v5 UUID + Password 认证

使用更完善的认证流程：

```
┌──────────────────────────────────────────┐
│           TUIC v5 认证流程                 │
├──────────────────────────────────────────┤
│                                          │
│  1. 客户端发送认证请求                     │
│     [Command: Authenticate]               │
│     [UUID + HMAC 签名]                    │
│                                          │
│  2. 服务器验证                             │
│     查找 UUID → 获取对应 Password         │
│     计算 HMAC 并验证                      │
│                                          │
│  3. 服务器响应                             │
│     [Command: Authenticate Response]      │
│     [Status: OK / FAIL]                   │
│                                          │
└──────────────────────────────────────────┘
```

认证请求使用 UUID 标识用户，使用 Password 计算 HMAC-SHA256 签名，防止令牌重放。

## 帧格式详解

### TUIC 命令帧

所有 TUIC 命令帧以 Type 字段开头：

```
+--------+--------+==================+
| Type   | Length |     Payload      |
| (1B)   | (2B)   |    (Variable)    |
+--------+--------+==================+
```

### 命令类型

| Command Type | 值 | 方向 | 说明 |
|-------------|-----|------|------|
| Authenticate | `0x00` | C→S | 认证请求（v5） |
| Authenticate Response | `0x01` | S→C | 认证响应（v5） |
| Connect | `0x02` | C→S | 建立 TCP 连接 |
| Connect Response | `0x03` | S→C | TCP 连接响应 |
| Packet | `0x04` | C→S/S→C | UDP 数据报（QUIC Datagram） |
| Packet Fragment | `0x05` | C→S/S→C | UDP 分片数据报 |
| Heartbeat | `0x06` | C→S/S→C | 心跳包 |
| Dissociate | `0x07` | C→S | UDP 解关联（结束 UDP 会话） |
| Disconnect | `0x08` | C→S | 断开连接通知 |

### Connect 命令帧（TCP 连接请求）

```
+--------+--------+--------+-------+-----------+----------+
| Type=2 | Length | ATYP   | ADDR  | Port (2B) |  Padding  |
| (1B)   | (2B)   | (1B)   | (Var) |           |  (Var)    |
+--------+--------+--------+-------+-----------+----------+
```

- **ATYP**（1 字节）：地址类型
  - `0x00` — 保留
  - `0x01` — IPv4
  - `0x03` — 域名
  - `0x04` — IPv6
- **ADDR**（可变）：目标地址
  - IPv4：4 字节
  - IPv6：16 字节
  - 域名：1 字节长度 + N 字节域名

### Connect Response 帧

```
+--------+--------+--------+
| Type=3 | Length | Status |
| (1B)   | (2B)   | (1B)   |
+--------+--------+--------+
```

| Status | 含义 |
|--------|------|
| `0x00` | 成功 |
| `0x01` | 认证失败 |
| `0x02` | 目标不可达 |
| `0x03` | 连接被拒绝 |
| `0x04` | 内部错误 |

### UDP 数据报帧（Packet）

```
+--------+--------+--------+-------+-----------+----------+======+
| Type=4 | Length | ATYP   | ADDR  | Port (2B) | Assoc ID | DATA |
| (1B)   | (2B)   | (1B)   | (Var) |           |  (2B)    |      |
+--------+--------+--------+-------+-----------+----------+======+
```

- **Assoc ID**（2 字节）：UDP 会话标识符，用于区分不同的 UDP 流

### UDP over Stream 模式

TUIC v5 支持将 UDP 数据报通过 QUIC Stream 传输（而非 Datagram），解决某些网络环境下 Datagram 被限制的问题：

```
UDP over Stream v1:
[ATYP][ADDR][PORT][DATA]

UDP over Stream v2:
[Packet Type=0x04][Length][ATYP][ADDR][PORT][Assoc ID][DATA]
```

v2 版本增加了明确的类型标识和长度字段，提供更可靠的解析。

## QUIC 连接多路复用

### Stream 复用

QUIC 协议原生支持在同一连接上创建多个独立的 Stream：

```
┌──────────────────────────────────────────────────┐
│              QUIC 连接多路复用                      │
├──────────────────────────────────────────────────┤
│                                                   │
│  QUIC Connection                                  │
│  ├─ Stream 0 (控制流)                              │
│  │   └─ 认证、心跳等                               │
│  ├─ Stream 1 (TCP 代理流 #1)                       │
│  │   └─ 代理到 example.com:443                    │
│  ├─ Stream 2 (TCP 代理流 #2)                       │
│  │   └─ 代理到 api.example.com:80                  │
│  ├─ Stream 3 (TCP 代理流 #3)                       │
│  │   └─ 代理到 cdn.example.com:443                │
│  └─ Datagram (UDP 代理)                            │
│      ├─ Assoc ID 0: DNS 查询                       │
│      └─ Assoc ID 1: 游戏流量                        │
│                                                   │
│  优势：                                             │
│  - 无队头阻塞（每个 Stream 独立）                    │
│  - 单连接多路复用，减少握手开销                       │
│  - 0-RTT 快速重连                                  │
│                                                   │
└──────────────────────────────────────────────────┘
```

### 拥塞控制算法

| 算法 | 特点 | 适用场景 |
|------|------|---------|
| BBR | 基于带宽和 RTT，主动探测管道容量 | 高带宽、低延迟 |
| CUBIC | 基于丢包的三次函数增长 | 标准互联网环境 |
| New Reno | 经典 AIMD 算法 | 兼容性和简单场景 |

### 窗口配置

```yaml
recv-window-conn: 8388608   # QUIC 连接接收窗口（8 MB）
recv-window: 4194304        # QUIC 流接收窗口（4 MB）
```

- **recv-window-conn**：整个 QUIC 连接的接收缓冲区大小
- **recv-window**：单个 QUIC Stream 的接收缓冲区大小
- 较大的窗口在高延迟网络中能更好地利用带宽

## 心跳机制

```yaml
heartbeat-interval: 10000   # 10 秒
```

心跳用于保持 NAT 映射活跃和检测连接状态：

```
Client → Server: [Heartbeat]
Server → Client: [Heartbeat]
```

如果超过 3 个心跳间隔未收到响应，连接被视为断开。

## 错误处理

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| 认证失败 | Token/UUID+Password 不匹配 | 连接被拒绝 |
| QUIC 握手失败 | TLS 协商错误或 ALPN 不匹配 | 连接断开 |
| 目标不可达 | Connect 响应 Status != 0 | 返回错误给客户端 |
| Stream 被拒绝 | max-open-streams 超限 | 等待空闲 Stream |
| Datagram 丢失 | 网络拥塞或 MTU 限制 | 上层协议处理重传 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | TUIC 协议实现 |
| Pipeline | 支持 | QUIC 流 |
| Multiplex | 完全兼容 | QUIC 多路复用 |

## 相关文档

- [[hysteria]] - Hysteria 协议
- [[hysteria2]] - Hysteria2 协议
- [[dev/debugging/udp|udp]] - UDP 传输