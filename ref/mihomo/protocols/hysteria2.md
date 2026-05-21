---
title: Hysteria2
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, hysteria2]
---
# Hysteria2 协议

Hysteria2 是 Hysteria 协议的第二代实现，基于 QUIC 传输协议构建，专为高丢包、高延迟的恶劣网络环境设计。它利用 QUIC 的内建多路复用、拥塞控制和快速握手特性，结合自研的拥塞控制算法，在弱网环境下仍能保持出色的吞吐性能。mihomo 在 `adapter/outbound/hysteria2.go` 中实现了完整的 Hysteria2 客户端。

## 协议背景

### 为什么基于 QUIC

Hysteria2 选择 QUIC（RFC 9000）作为传输层，主要原因如下：

1. **UDP 基础**：QUIC 基于 UDP，绕过了 TCP 的中间设备干预和连接劫持问题。
2. **0-RTT 连接恢复**：之前连接过的服务端可以 0-RTT 恢复会话，减少握手延迟。
3. **内置多路复用**：QUIC Stream 天然支持多路复用，无需额外的复用层。
4. **连接迁移**：QUID 使用 Connection ID 而非五元组标识连接，支持网络切换时不中断。
5. **更好的拥塞控制**：QUIC 在用户态实现拥塞控制，可以灵活替换算法（如 BBR）。

### Hysteria v1 vs v2 对比

| 特性 | Hysteria v1 | Hysteria v2 |
|------|-------------|-------------|
| 传输协议 | 自研 UDP 协议 | QUIC (RFC 9000) |
| 认证方式 | Auth String | Password + TLS |
| 拥塞控制 | 自定义 | BBR v1（可调参） |
| 加密 | 可选混淆 | 强制 TLS + Salamander 可选 |
| 端口跳跃 | 不支持 | 原生支持 |
| 多用户 | 不支持 | 原生支持（Token） |
| 配置复杂度 | 高（带宽、协议参数） | 低（简化配置） |

### Salamander 混淆

Salamander 是 Hysteria2 引入的可选混淆层，位于 QUIC 数据报之上：

```
原始 QUIC Payload → XOR 混淆 → 发送
接收 → XOR 混淆 → 原始 QUIC Payload
```

混淆原理：

1. 使用一个预共享密钥（`obfs-password`）初始化伪随机数生成器。
2. 对每个字节与 PRNG 输出进行 XOR 操作。
3. 接收端使用相同密钥逆向 XOR 恢复原始数据。

Salamander 的目的不是提供加密（QUIC 已经加密），而是改变流量统计特征，抵抗基于流量模式的 DPI 检测。

## QUIC 帧结构

### QUIC 短头部（Short Header）

用于初始握手完成后的数据传输：

```
+------+-------+-------+
| 1    |  00   | F   R |  ← 固定位(1) + 保留位(00) + Flags
+------+-------+-------+
| Connection ID (变长)  |  ← 连接标识
+-----------------------+
| Packet Number (1-4B)  |  ← 包编号
+-----------------------+
| Protected Payload     |  ← AEAD 加密的载荷
+-----------------------+
```

### QUIC 长头部（Long Header）

用于握手阶段：

```
+------+-------+-------+-------+
| 1    |  01   | Type  |  RR   |  ← 固定位 + 版本 + 类型
+------+-------+-------+-------+
| Version (4 bytes)            |  ← QUIC 版本
+------------------------------+
| DCIL | SCIL | DCID | SCID    |  ← 目标/源连接 ID 长度及值
+------------------------------+
| Length (变长)                |  ← 剩余长度
+------------------------------+
| Packet Number (变长)          |
+------------------------------+
| Protected Payload            |
+------------------------------+
```

长头部类型：

| Type | 名称 | 用途 |
|------|------|------|
| 0x00 | Initial | 握手初始包，携带 TLS ClientHello |
| 0x01 | 0-RTT | 0-RTT 数据 |
| 0x02 | Handshake | TLS 握手数据 |
| 0x03 | Retry | 服务端重试（Anti-DoS） |

### QUIC Frame 类型

在 Hysteria2 中，主要使用以下 QUIC Frame：

| Frame Type | 用途 |
|------------|------|
| CRYPTO (0x06) | 传输 TLS 握手数据 |
| STREAM (0x08-0x0F) | 传输应用数据（Hysteria2 业务数据） |
| ACK (0x02-0x03) | 确认收到数据包 |
| PING (0x01) | 保活探测 |
| PADDING (0x00) | 填充，满足 Initial 包最小长度要求 |
| CONNECTION_CLOSE (0x1C-0x1D) | 关闭连接 |
| NEW_CONNECTION_ID (0x19) | 分配新连接 ID |

### Hysteria2 应用层帧

Hysteria2 在 QUIC Stream 之上定义自己的应用层协议：

```
+----------+----------+------------------+
| Type (1) | Length   | Payload          |
|          | (变长)   |                  |
+----------+----------+------------------+
```

| Type | 含义 |
|------|------|
| 0x01 | TCP 请求（携带目标地址） |
| 0x02 | TCP 数据 |
| 0x03 | UDP 会话请求 |
| 0x04 | UDP 数据 |
| 0x05 | 控制消息（带宽通告等） |

## 拥塞控制

### BBR 在 Hysteria2 中的应用

Hysteria2 使用 Google BBR（Bottleneck Bandwidth and Round-trip propagation time）拥塞控制算法，这是其高性能的核心。

#### BBR 工作原理

BBR 不依赖传统的丢包检测，而是通过建模网络的带宽和 RTT 来优化发送速率：

```
         ┌───────────────┐
         │  带宽采样      │  ← 测量实际发送速率（delivery rate）
         └───────┬───────┘
                 ▼
         ┌───────────────┐
         │  RTT 采样      │  ← 测量最小 RTT（RTprop）
         └───────┬───────┘
                 ▼
         ┌───────────────┐
         │  BDP 估算      │  ← BDP = 带宽 × RTT
         └───────┬───────┘
                 ▼
    ┌────────────┴────────────┐
    ▼                         ▼
┌────────┐              ┌────────┐
|Pacing  │              | cwnd   │
|Rate    │              | 窗口    │
└────────┘              └────────┘
```

BBR 的四个状态：

1. **STARTUP**：指数增长发送速率，探测最大带宽。
2. **DRAIN**：减少排队的包，清空管道。
3. **PROBE_BW**：周期性探测更高带宽（Gain Cycle）。
4. **PROBE_RTT**：定期降低发送速率，测量最小 RTT。

#### Hysteria2 的 BBR 调参

通过 `cwnd` 和相关配置，可以调整 BBR 行为：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `cwnd` | 32 | 初始拥塞窗口（BBR 单位） |
| `up` | 自动 | 上行带宽上限，影响 BBR 探测范围 |
| `down` | 自动 | 下行带宽上限 |

BBR 的优势在高丢包环境（如跨国链路、移动网络）中尤为明显：

- **传统 TCP**：1% 丢包率下吞吐量下降约 50%。
- **BBR**：1% 丢包率下吞吐量几乎不受影响。
- **传统 TCP**：5% 丢包率下连接几乎不可用。
- **BBR**：5% 丢包率下仍可维持可用吞吐量。

### 流控制

QUIC 提供两级流控制：

1. **Stream 级别**：单个流的数据量限制。
2. **Connection 级别**：整个连接的数据量限制。

```
Sender ──[STREAM Frame]──► Receiver
             │
             │ 接收窗口 = min(StreamWindow, ConnWindow)
             ▼
         如果窗口耗尽 → 发送 STOP_SENDING
         接收方处理后 → 发送 MAX_STREAM_DATA / MAX_DATA 扩大窗口
```

Hysteria2 配置字段：

| 字段 | 作用域 | 说明 |
|------|--------|------|
| `initial-stream-receive-window` | Stream | QUIC 流初始接收窗口 |
| `max-stream-receive-window` | Stream | QUIC 流最大接收窗口 |
| `initial-connection-receive-window` | Connection | QUIC 连接初始接收窗口 |
| `max-connection-receive-window` | Connection | QUIC 连接最大接收窗口 |

窗口大小的合理配置对性能影响显著：

- 窗口过小：发送方被限速，无法充分利用带宽。
- 窗口过大：内存消耗增加，缓冲区膨胀（Bufferbloat）。
- 经验值：高带宽链路建议设置为 `带宽 × RTT × 2`（BDP 的 2 倍）。

## 通信流程

### QUIC 连接建立

```
客户端                                                    服务端
   │                                                         │
   │  1. DNS 解析，获取服务器 IP                              │
   │  2. 生成 QUIC Connection ID                             │
   │  3. TLS ClientHello（在 Initial 包中）                   │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │
   │                           4. TLS ServerHello + 证书     │
   │                           5. 验证密码（Password 认证）  │
   │ ◄────────────────────────────────────────────────────── │
   │                                                         │
   │  6. 验证服务端证书（SNI 匹配）                           │
   │  7. TLS 握手完成                                         │
   │  8. 1-RTT 密钥就绪                                      │
   │                                                         │
   │  9. 开始 QUIC Stream 传输                               │
   │ ◄────────────────────────────────────────────────────── │
   │                                                         │
```

### TCP 代理时序

```
客户端                                                    服务端
   │                                                         │
   │  1. 打开新 QUIC Stream                                  │
   │  2. 发送 Type=0x01 帧（目标地址）                        │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  3. 解析目标地址
   │                                                         │  4. 建立到目标的 TCP 连接
   │                                                         │
   │  5. 发送 Type=0x02 帧（TCP 数据）                        │
   │ ◄────────────────────────────────────────────────────── │  6. 转发目标响应
   │                                                         │
   │  7. 关闭 Stream                                         │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │
```

### UDP 代理时序

```
客户端                                                    服务端
   │                                                         │
   │  1. 打开新 QUIC Stream（UDP 会话）                       │
   │  2. 发送 Type=0x03 帧（UDP 会话请求 + 目标地址）         │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  3. 创建 UDP Socket
   │                                                         │
   │  4. 发送 Type=0x04 帧（UDP 数据）                        │
   │ ◄────────────────────────────────────────────────────── │  5. 转发到目标 + 返回响应
   │                                                         │
   │  6. 继续收发 UDP 数据                                    │
   │ ◄────────────────────────────────────────────────────── │
   │                                                         │
   │  7. 关闭 QUIC Stream（结束 UDP 会话）                    │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  8. 关闭 UDP Socket
   │                                                         │
```

### 端口跳跃（Port Hopping）

Hysteria2 的端口跳跃允许在多个 UDP 端口之间切换连接：

1. 客户端配置 `ports: 10000-50000`。
2. 初始连接随机选择一个端口。
3. 如果连接失败或质量下降，尝试切换到同一范围内的另一个端口。
4. `hop-interval` 控制切换的最小时间间隔，避免频繁震荡。

端口跳跃的优势：

- **绕过端口级封锁**：某些防火墙封锁特定端口。
- **负载均衡**：多端口分散连接。
- **自动恢复**：主端口不可用时自动切换。

## 错误处理

### 连接超时

| 超时类型 | 触发条件 | 处理方式 |
|---------|---------|---------|
| 握手超时 | TLS 握手超过 10 秒未完成 | 关闭连接，尝试重连（端口跳跃） |
| 空闲超时 | 15 秒无数据传输 | 发送 PING，若仍无响应则关闭 |
| 路径验证超时 | 网络切换后路径不可达 | 触发连接迁移，使用新 Connection ID |

### 流控制错误

| 错误类型 | 触发条件 | 处理方式 |
|---------|---------|---------|
| 流窗口耗尽 | 接收方窗口满，无法接收更多数据 | 发送 STOP_SENDING，等待 MAX_STREAM_DATA |
| 连接窗口耗尽 | 整个连接窗口满 | 暂停发送所有流，等待 MAX_DATA |
| 最终超时 | 窗口未扩大且超时 | 关闭受影响的 Stream |

### QUIC 连接错误

| 错误类型 | QUIC Error Code | 处理方式 |
|---------|----------------|---------|
| 认证失败 | AuthenticationError | 连接关闭，上报错误 |
| 证书验证失败 | CertificateError | 检查 `skip-cert-verify` 配置 |
| 版本不协商 | VersionNegotiationError | 尝试其他 QUIC 版本 |
| 内部错误 | InternalError | 记录日志，重连 |

### Salamander 混淆错误

| 错误类型 | 触发条件 | 处理方式 |
|---------|---------|---------|
| 混淆密钥不匹配 | 客户端/服务端 obfs-password 不一致 | QUIC 解密失败，连接建立失败 |
| PRNG 状态不一致 | 混淆数据偏移错位 | 所有后续数据均无法解密，连接断开 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Hysteria2 协议实现 |
| Pipeline | 支持 | QUIC 流需要适配 Pipeline 抽象 |
| Multiplex | 完全兼容 | QUIC 内置多路复用，无需外部 mux |
| Crypto | 部分兼容 | QUIC 使用自己的 AEAD（AES-GCM/ChaCha20） |
| Front | 支持 | 需要 UDP socket 支持 |

Prism 的 Hysteria2 实现注意点：

- QUIC 协议栈需要完整的 RFC 9000 实现或使用现有库（如 quinn、msquic）。
- BBR 拥塞控制需要在用户态实现，不依赖内核 TCP 拥塞控制。
- Salamander 混淆是 XOR-based 对称操作，实现简单但需注意 PRNG 同步。
- 端口跳跃需要维护可用端口池和连接状态机。
- TLS 证书验证依赖 `crypto/x509` 等效模块。
- UDP MTU 发现（PMTUD）对性能至关重要，`udp-mtu` 默认 1197。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/hysteria2.go` | Hysteria2 适配器 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "hysteria2-proxy"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
```

### Salamander 混淆

```yaml
proxies:
  - name: "hysteria2-obfs"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    obfs: salamander
    obfs-password: obfs-string
```

### 端口跳跃

```yaml
proxies:
  - name: "hysteria2-hop"
    type: hysteria2
    server: server.example.com
    ports: 10000-50000
    hop-interval: 30
    password: your-password
```

### 带宽配置

```yaml
proxies:
  - name: "hysteria2-speed"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    up: "100 Mbps"
    down: "200 Mbps"
    cwnd: 32
```

### 完整配置

```yaml
proxies:
  - name: "hysteria2-full"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    obfs: salamander
    obfs-password: obfs-string
    sni: server.example.com
    skip-cert-verify: false
    alpn:
      - h3
    fingerprint: chrome
    up: "100 Mbps"
    down: "200 Mbps"
    cwnd: 32
    udp-mtu: 1200
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `hysteria2` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口 |
| `ports` | string | 否 | 端口范围 |
| `password` | string | 是 | 认证密码 |
| `obfs` | string | 否 | 混淆类型 |
| `obfs-password` | string | 否 | 混淆密码 |
| `up` | string | 否 | 上传带宽 |
| `down` | string | 否 | 下载带宽 |
| `sni` | string | 否 | TLS SNI |
| `cwnd` | int | 否 | 拥塞窗口 |
| `hop-interval` | int | 否 | 跳跃间隔（秒） |
| `udp-mtu` | int | 否 | UDP MTU（默认 1197） |
| `initial-stream-receive-window` | uint64 | 否 | QUIC 流初始接收窗口 |
| `max-stream-receive-window` | uint64 | 否 | QUIC 流最大接收窗口 |
| `initial-connection-receive-window` | uint64 | 否 | QUIC 连接初始接收窗口 |
| `max-connection-receive-window` | uint64 | 否 | QUIC 连接最大接收窗口 |

## 相关文档

- [[hysteria]] - Hysteria v1 协议
- [[tuic]] - TUIC 协议
- [[dev/debugging/udp|udp]] - UDP 传输
