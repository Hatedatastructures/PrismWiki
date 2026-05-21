---
title: Mieru
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, mieru]
---
# Mieru 协议

Mieru 是一种新兴的代理协议，设计目标是在高丢包、高延迟的恶劣网络环境下仍然保持稳定的代理性能。它通过独特的多路复用、端口范围利用和流量模式控制等技术实现了这一目标。

## 协议概述

Mieru 的核心特性：

- **双传输模式**：支持 TCP 和 UDP 两种传输方式
- **端口范围利用**：在 UDP 模式下可使用端口范围分散流量
- **多路复用**：内置高级多路复用机制，单连接承载多条代理流
- **Traffic Pattern 控制**：可配置流量模式以规避基于流量特征的检测
- **加密认证**：使用用户名/密码进行认证，数据传输加密
- **握手模式可选**：支持不同的握手模式以平衡安全性和性能

Mieru 的设计理念是"弹性传输"——即使在网络条件极差的情况下（如高丢包率的移动网络），也能维持可用的代理连接。

## 多路复用原理

### 设计动机

传统的代理协议在单条 TCP 连接上通常只能传输一条代理流。如果需要同时访问多个网站，就需要建立多条 TCP 连接，这带来了以下问题：

1. **连接开销**：每条连接都需要独立的 TLS 握手（如果使用 TLS）
2. **端口消耗**：每条连接占用一个本地端口
3. **检测风险**：大量并发连接容易引起检测

Mieru 通过在单条传输连接上复用多条代理流来解决这些问题。

### 多路复用架构

```
┌───────────────────────────────────────────┐
│  传输层 (TCP / UDP)                        │
│  ┌─────────────────────────────────────┐  │
│  │  Mieru 多路复用层                     │  │
│  │  ┌─────────┐ ┌─────────┐ ┌────────┐ │  │
│  │  │ Stream 1│ │ Stream 2│ │Stream N│ │  │
│  │  │ (HTTP)  │ │ (DNS)   │ │(TCP)   │ │  │
│  │  └─────────┘ └─────────┘ └────────┘ │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

### 流标识和帧格式

Mieru 使用流 ID（Stream ID）来区分不同的复用流：

```
+-----------------+-----------------+-----------------+-----------------+
| Frame Header    | Stream ID       | Payload Length  | Payload         |
+-----------------+-----------------+-----------------+-----------------+
| 变长            | 2 字节          | 2 字节          | 变长            |
+-----------------+-----------------+-----------------+-----------------+

Frame Header 结构:
+----------+----------+----------+
| Magic    | Version  | Flags    |
+----------+----------+----------+
| 2 字节   | 1 字节   | 1 字节   |
+----------+----------+----------+

Flags 位掩码:
Bit 0: FIN（流结束标志）
Bit 1: SYN（流建立标志）
Bit 2: RST（流重置标志）
Bit 3: ACK（确认标志）
Bit 4-7: 保留
```

| 字段 | 说明 |
|------|------|
| `Magic` | 协议魔数，用于识别 Mieru 流量 |
| `Version` | 协议版本号 |
| `Flags` | 帧控制标志位 |
| `Stream ID` | 流标识符，0 通常保留给控制流 |
| `Payload Length` | 载荷长度（不包含头部） |
| `Payload` | 加密后的代理数据 |

### 流生命周期

```
建立:
Client                    Server
  |  SYN (Stream ID=1)      |
  |------------------------>|
  |  ACK                    |
  |<------------------------|
  |  数据传输 (Stream ID=1)  |

传输:
  |  DATA (Stream ID=1)     |
  |------------------------>|
  |  DATA (Stream ID=1)     |
  |<------------------------|

关闭:
  |  FIN (Stream ID=1)      |
  |------------------------>|
  |  ACK                    |
  |<------------------------|
```

### 多路复用级别

```yaml
proxies:
  - name: "mieru-mux"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    multiplexing: HIGH
```

| 级别 | 描述 | 适用场景 |
|------|------|----------|
| `OFF` | 不启用多路复用，每条连接只承载一条流 | 最低延迟 |
| `LOW` | 少量流复用（2-4 条） | 平衡性能和资源 |
| `MEDIUM` | 中等流复用（4-16 条） | 日常使用 |
| `HIGH` | 大量流复用（16+ 条） | 多任务并发 |

更高的复用级别意味着：
- **更少的连接数**：单条传输连接承载更多代理流
- **更低的连接开销**：减少握手次数
- **更高的队头阻塞风险**：TCP 模式下一条流的问题可能影响其他流
- **UDP 模式优势**：在 UDP 模式下，多路复用无队头阻塞问题

## 协议头部格式

### 握手阶段

Mieru 的握手阶段用于认证和协商参数：

```
TCP 模式:
Client                    Server
  |  Handshake Request      |
  |  (认证信息 + 参数)       |
  |------------------------>|
  |  Handshake Response     |
  |  (确认 + 参数)           |
  |<------------------------|
  |  数据传输开始             |

UDP 模式:
Client                    Server
  |  Handshake Datagram    |
  |------------------------>|
  |  Handshake Response    |
  |<------------------------|
  |  Data Datagram         |
```

握手请求格式：

```
+-----------------+-----------------+-----------------+
| Magic           | Version         | Auth Length     |
+-----------------+-----------------+-----------------+
| 2 字节          | 1 字节          | 2 字节          |
+-----------------+-----------------+-----------------+
| Auth Data       | Parameters      | Padding         |
| (变长)          | (变长)          | (变长)          |
+-----------------+-----------------+-----------------+
```

### 认证机制

认证数据经过加密处理：

```
Auth Data = Encrypt(
    key = derived_from(username, password),
    data = {
        username: "...",
        timestamp: 1234567890,
        nonce: random_bytes
    }
)
```

### 数据帧格式

正常数据传输阶段的帧格式：

```
+----------+----------+----------+----------+----------+
| Magic    | StreamID | Length   | Flags    | Payload  |
+----------+----------+----------+----------+----------+
| 2 B      | 2 B      | 2 B      | 1 B      | 变长     |
+----------+----------+----------+----------+----------+
```

整个帧经过 AEAD 加密，外部观察者无法得知帧的边界和内容。

## TCP 模式

TCP 模式下，Mieru 使用标准的 TCP 连接进行数据传输：

```yaml
proxies:
  - name: "mieru-tcp"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
```

TCP 模式的优势：
- 可靠性高，数据不会丢失
- 与所有网络环境兼容
- 多路复用后队头阻塞影响可控

## UDP 模式与端口范围

UDP 模式下，Mieru 可以使用端口范围来分散流量：

```yaml
proxies:
  - name: "mieru-udp"
    type: mieru
    server: server.example.com
    port-range: 10000-20000
    transport: UDP
    username: user
    password: pass
```

端口范围的优势：
1. **流量分散**：数据报分散到多个端口，降低单端口流量特征
2. **负载均衡**：多个端口可以分担负载
3. **抗封锁**：封锁单个端口不影响整体连接
4. **NAT 穿透**：在某些 NAT 环境下表现更好

## Traffic Pattern 流量模式控制

Mieru 支持配置流量模式以规避基于流量特征的检测：

```yaml
proxies:
  - name: "mieru-pattern"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    traffic-pattern: "pattern-string"
```

Traffic Pattern 控制以下方面：
- **包大小分布**：调整输出包的大小分布
- **发送间隔**：控制数据包之间的时间间隔
- **突发模式**：模拟正常浏览的突发流量模式
- **空闲填充**：在空闲期间发送填充数据，维持连接活跃度

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/mieru.go` | Mieru 适配器，配置解析和协议集成 |
| `transport/mieru/` | Mieru 传输层实现，包含多路复用和加密 |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `mieru` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口（TCP 模式必填） |
| `port-range` | string | 否 | 端口范围（UDP 模式，如 `10000-20000`） |
| `transport` | string | 是 | `TCP` 或 `UDP` |
| `username` | string | 是 | 用户名 |
| `password` | string | 是 | 密码 |
| `multiplexing` | string | 否 | 多路复用级别（OFF/LOW/MEDIUM/HIGH） |
| `handshake-mode` | string | 否 | 握手模式 |
| `traffic-pattern` | string | 否 | 流量模式配置 |

## YAML 配置示例

### TCP 模式

```yaml
proxies:
  - name: "mieru-tcp"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
```

### UDP 模式

```yaml
proxies:
  - name: "mieru-udp"
    type: mieru
    server: server.example.com
    port-range: 10000-20000
    transport: UDP
    username: user
    password: pass
```

### 多路复用

```yaml
proxies:
  - name: "mieru-mux"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    multiplexing: HIGH
```

### Traffic Pattern

```yaml
proxies:
  - name: "mieru-pattern"
    type: mieru
    server: server.example.com
    port: 443
    transport: TCP
    username: user
    password: pass
    traffic-pattern: "pattern-string"
```

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Mieru 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 内置多路复用 |

## 相关文档

- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
- [[ref/mihomo/multiplex|multiplex]] - 多路复用配置
