---
title: VLESS
created: 2026-05-17
updated: 2026-05-18
layer: ref
tags: [mihomo, protocol, vless]
---
# VLESS 协议

VLESS 是 V2Ray 的轻量级协议，移除了 VMess 的冗余加密。

## 协议概述

VLESS 特性：
- 轻量级设计，无内置加密
- 支持多种传输层（TCP、WS、gRPC、H2）
- 支持 XTLS 流控
- 支持 Reality 和 ECH
- 支持 XUDP 多路复用

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/vless.go` | VLESS 适配器 |
| `transport/vless/vless.go` | VLESS 协议实现 |
| `transport/vless/encryption/` | VLESS 加密扩展 |

## 协议背景

VLESS 由 V2Ray 项目引入，作为 VMess 协议的轻量化替代方案而设计。

### 设计动机

VMess 协议在 TLS 之上额外实现了完整的 AEAD 加密层，这在以下场景中构成冗余开销：

1. **TLS 已提供加密**：当 VLESS 运行在 TLS/Reality/XTLS 之上时，传输层已具备机密性和完整性保护
2. **CPU 开销**：VMess 的 AEAD 加密每数据包都需要额外的 AES-GCM 计算
3. **延迟**：VMess 的加密/解密在用户态引入额外处理路径

VLESS 的核心设计理念：**在 TLS 已加密的场景下，移除冗余的用户态加密层**。

### 与 VMess 的对比

| 特性 | VMess | VLESS |
|------|-------|-------|
| 内置加密 | AEAD（aes-128-gcm / chacha20-poly1305） | 无（依赖外部 TLS） |
| 认证 | UUID + AlterID（v2） / UUID（v4） | UUID |
| 请求头 | 动态加密（MD5/SHA1 派生） | 明文（TLS 保护下） |
| CPU 开销 | 较高（双重加密） | 低（单层加密） |
| XTLS 支持 | 不支持 | 支持（Vision 零延迟） |

## 帧格式

### 请求帧

VLESS 请求帧采用简洁的二进制格式，无内置加密：

```
[VLESS Request Header]
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Version (1B) |                   UUID (16B)                   |
|               |                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| AddnlInfoLen  |            Additional Info (variable)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Cmd       |           Port (2B, Big Endian)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   ATYP        |           Address (variable)                   |
|               |                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### 字段详解

| 字段 | 长度 | 取值 | 说明 |
|------|------|------|------|
| `Version` | 1 byte | `0x00` | 协议版本，固定为 0x00 |
| `UUID` | 16 bytes | 任意 | 用户 UUID，用于服务端认证。格式为标准 UUID 的 16 字节二进制形式（如 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` 编码后的二进制） |
| `AddnlInfoLen` | 1 byte | `0x00`~`0xFF` | 附加信息长度。Plain VLESS 模式下为 `0x00` |
| `Additional Info` | 可变 | - | 仅当 `AddnlInfoLen > 0` 时存在。用于 XTLS/Encryption 扩展 |
| `Cmd` | 1 byte | `0x01`=TCP<br>`0x02`=UDP | 命令类型。TCP 表示流式连接，UDP 表示数据报转发 |
| `Port` | 2 bytes | Big Endian | 目标端口，网络字节序（大端） |
| `ATYP` | 1 byte | `0x01`=IPv4<br>`0x02`=Domain<br>`0x03`=IPv6 | 目标地址类型 |
| `Address` | 可变 | - | 目标地址，编码方式取决于 ATYP |

#### 地址编码规则

| ATYP | Address 格式 | 示例 |
|------|-------------|------|
| `0x01` (IPv4) | 4 bytes | `127.0.0.1` -> `7F 00 00 01` |
| `0x02` (Domain) | 1 byte 长度 + N bytes | `example.com` -> `0B 65 78 61 6D 70 6C 65 2E 63 6F 6D` |
| `0x03` (IPv6) | 16 bytes | `::1` -> `00...00 01` |

### 响应帧

VLESS 响应帧固定为 2 字节：

```
[VLESS Response Header]
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+
|  Version (1B) |   0x00  |
+-+-+-+-+-+-+-+-+-+-+-+-+

固定值：[0x00, 0x00]
```

响应帧不携带额外信息，仅确认请求已接受。之后进入数据转发阶段。

## 通信流程

### TCP 模式（Cmd = 0x01）

```
Client                              Server
  |                                  |
  |-- [TCP Connect] ---------------->|  (外层 TLS 握手如启用)
  |                                  |
  |-- [VLESS Request Header] ------->|  发送请求帧
  |   Version=0x00, UUID, Cmd=0x01   |
  |   ATYP, Address, Port            |
  |                                  |
  |                                  |  [认证 UUID]
  |                                  |  验证失败 -> 关闭连接
  |                                  |
  |<-- [VLESS Response 0x00,0x00] ---|  验证成功 -> 发送响应帧
  |                                  |
  |<-- [Target Connect] ------------->|  服务端连接目标地址
  |                                  |
  |<======== Data Stream (Bidirectional) =========>|
  |                                  |
```

**步骤说明**：

1. **TCP 连接建立**：客户端连接到 VLESS 服务器（如启用 TLS 则先完成 TLS 握手）
2. **发送请求帧**：客户端发送 VLESS 请求头（Version + UUID + Cmd + Port + ATYP + Address）
3. **服务端认证**：服务端验证 UUID 是否匹配配置的用户列表
4. **响应**：UUID 验证通过后，服务端返回 `[0x00, 0x00]` 响应帧
5. **目标连接**：服务端根据请求帧中的 ATYP + Address + Port 建立到目标的连接
6. **双向转发**：进入透明的双向数据转发阶段，客户端与目标端之间无协议开销

### UDP 模式（Cmd = 0x02）

UDP 模式下，VLESS 请求帧后跟随 UDP 数据包：

```
[VLESS UDP Packet]
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       VLESS Request Header                    |
|                 (同 TCP 模式, Cmd = 0x02)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Length (2B)          |         Payload (variable)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- `Length`：UDP Payload 长度（2 字节，大端）
- `Payload`：UDP 数据报内容

### XUDP（扩展 UDP）

XUDP 是 V2Ray 的增强型 UDP 多路复用机制：

- 在 XUDP 模式下，多个 UDP 流可以在单个 TCP 连接上复用
- 通过 `packet-addr` 和 `packet-encoding` 字段控制行为
- `xudp` 模式：使用 VLESS 原生 XUDP 多路复用
- `packetaddr` 模式：使用 PacketAddr 地址编码方案
- 两者互斥，不能同时启用

## XTLS Vision 原理

### 概述

XTLS Vision 是 VLESS 协议的核心创新之一，实现了真正的零延迟 TLS 代理。

### 传统 TLS 代理的瓶颈

```
[传统 TLS 代理数据流]

Application Data
       |
       v
+------------------+
|   User-space     |
|   TLS Encrypt    |  <-- 额外 CPU 开销
+------------------+
       |
       v
+------------------+
|   Protocol Layer |  <-- VMess AEAD（双重加密）
+------------------+
       |
       v
+------------------+
|   TLS Library    |  <-- TLS 层再次加密
+------------------+
       |
       v
  Network Socket
```

传统方案中，数据经历两次加密：用户态协议层（VMess AEAD）+ TLS 层。即使 VMess 被移除，TLS 加密/解密仍在用户态执行。

### XTLS Vision 的优化

```
[XTLS Vision 数据流]

Application Data
       |
       v
+------------------+
|   Protocol Layer |  <-- VLESS 协议头（无加密）
+------------------+
       |
       v
+------------------+
|  Direct TLS      |  <-- 绕过用户态，直接操作 TLS 明文
|  Stream Access   |
+------------------+
       |
       v
  Network Socket
```

XTLS Vision 通过以下技术实现零延迟：

1. **Direct Memory Access**：VLESS 直接读取 TLS 库（如 TLS 1.3）的内部缓冲区，绕过用户态加密/解密循环
2. **Vision Flow Control**：通过 `flow` 字段协商流控模式，服务端动态调整数据读取策略
3. **Zero-Copy Forwarding**：在支持的平台上，使用零拷贝技术减少内存复制次数

### Flow 模式详解

| Flow 值 | 机制 | 性能 | 适用场景 |
|---------|------|------|----------|
| `xtls-rprx-vision` | Vision 模式：服务端直接读取 TLS 明文缓冲区 | 零延迟，最低 CPU 开销 | 推荐。需要 Linux kernel 5.8+ 或特定 TLS 实现 |
| `xtls-rprx-origin` | 原始模式：兼容但不使用零拷贝 | 低延迟，较低 CPU 开销 | 兼容旧版内核或不支持 Vision 的环境 |
| *(空/不设置)* | 无 XTLS：标准 VLESS 模式 | 正常延迟 | 无 XTLS 支持的环境 |

### Vision 与 Reality 的协同

Reality 是 XTLS Vision 的配套隐蔽方案：

- Reality 伪装 TLS 握手，使 VLESS 流量看起来像正常的 TLS 连接
- Vision 在 Reality 提供的 TLS 通道上运行，实现零延迟数据转发
- 两者结合：既隐蔽（Reality 伪装）又高效（Vision 零延迟）

## 错误处理

### 认证失败

当服务端 UUID 验证失败时：

1. **行为**：服务端立即关闭 TCP 连接，不发送响应帧
2. **客户端表现**：连接被对端重置（RST）或在 TLS 场景下表现为 TLS 握手成功但连接断开
3. **隐蔽性**：无 UUID 的客户端无法区分 VLESS 服务器与普通 TLS 服务器（在 TLS 模式下）

### 目标连接失败

当服务端无法连接目标地址时：

1. **行为**：服务端关闭连接
2. **无错误码**：VLESS 协议不定义错误码，连接关闭即为失败信号

### 协议版本不匹配

当客户端发送的 Version 不为 `0x00` 时：

1. 服务端应拒绝该请求
2. 行为与认证失败一致：关闭连接

## 安全考量

### 无内置加密

VLESS 移除了内置加密层，因此：

- **纯 TCP 模式（无 TLS）不推荐使用**：流量完全明文
- **必须搭配 TLS/Reality**：生产环境中 VLESS 应始终运行在 TLS 或 Reality 之上
- **Reality 提供最佳隐蔽性**：伪装为标准 TLS 1.3 握手

### UUID 安全

- UUID 同时用作认证凭据和路由标识
- UUID 泄露等同于密码泄露，应使用强随机 UUID（RFC 4122 v4）
- 服务端应支持多 UUID 配置以支持多用户

### Replay 攻击

- VLESS 本身不提供 Replay 保护
- 在 TLS 1.3 + Reality 场景下，TLS 1.3 提供有限的 Replay 保护
- 生产环境建议配合服务端配置进行连接频率限制

## 与 Prism 的兼容性

### 基本配置

```yaml
proxies:
  - name: "vless-proxy"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    udp: true
```

### XTLS 配置

```yaml
proxies:
  - name: "vless-xtls"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    flow: xtls-rprx-vision
    tls: true
```

### WebSocket 配置

```yaml
proxies:
  - name: "vless-ws"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: ws
    tls: true
    ws-opts:
      path: /vless
      headers:
        Host: server.example.com
```

### gRPC 配置

```yaml
proxies:
  - name: "vless-grpc"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: grpc
    tls: true
    grpc-opts:
      grpc-service-name: vless
```

### Reality 配置

```yaml
proxies:
  - name: "vless-reality"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: xxx
      short-id: xxx
    client-fingerprint: chrome
```

### HTTP/2 配置

```yaml
proxies:
  - name: "vless-h2"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: h2
    tls: true
    h2-opts:
      host:
        - server.example.com
      path: /vless
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `vless` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `uuid` | string | 是 | 用户 UUID |
| `flow` | string | 否 | XTLS 流控模式 |
| `tls` | bool | 否 | 启用 TLS |
| `network` | string | 否 | 传输方式 |
| `udp` | bool | 否 | 启用 UDP |
| `xudp` | bool | 否 | 启用 XUDP |
| `packet-addr` | bool | 否 | 启用 PacketAddr 模式（与 xudp 互斥） |
| `packet-encoding` | string | 否 | UDP 包编码：packetaddr/packet/xudp |
| `encryption` | string | 否 | VLESS 加密扩展（如 aes-128-gcm） |
| `ws-headers` | map | 否 | WebSocket 自定义 Headers |
| `servername` | string | 否 | TLS 服务器名称（覆盖 SNI） |

## XTLS Flow 模式

| Flow | 描述 |
|------|------|
| `xtls-rprx-vision` | Vision 流控，零延迟 |
| `xtls-rprx-origin` | 原始模式 |

## 与 Prism 的兼容性

### 六阶段流水线映射

Prism 使用六阶段流水线架构处理 VLESS 协议：

| 阶段 | 职责 | VLESS 处理内容 |
|------|------|----------------|
| `recognition` | 协议识别 | 识别 VLESS 请求头（Version=0x00 + UUID 模式匹配） |
| `pipeline` | 协议解析 | 解析 VLESS 请求帧，提取 Cmd、ATYP、Address、Port |
| `channel` | 连接建立 | 根据解析结果建立到目标地址的 TCP/UDP 连接 |
| `multiplex` | 多路复用 | 支持 XUDP 多路复用、gRPC 传输 |
| `transport` | 传输层 | TLS/Reality/XTLS 加密处理 |
| `forward` | 数据转发 | 双向透明数据转发（无协议开销） |

### 模块兼容性详情

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | VLESS 协议完整实现，支持所有传输模式 |
| Pipeline | 完全兼容 | 支持 StreamConnContext，VLESS 帧解析与转发 |
| Multiplex | 支持 | 通过 XUDP 实现 UDP 多路复用，通过 gRPC 实现连接多路复用 |
| Crypto | 间接依赖 | VLESS 自身无加密，依赖 TLS 模块提供加密 |
| TLS | 完全兼容 | 支持 TLS 1.2/1.3、Reality、ECH 等 |

### Prism 特有优化

- **内存池分配**：VLESS 请求帧使用 PMR 内存池分配，减少动态内存开销
- **零拷贝转发**：在支持的平台（Linux `sendfile`、Windows `TransmitFile`）上实现零拷贝数据转发
- **Coroutine 异步**：使用 C++23 协程实现异步 VLESS 帧处理，无阻塞等待

## Prism 实现边界

- **不支持 XTLS/Vision flow**：`addnl_info != 0` 时返回 `bad_message`，使用 Vision 的客户端会连接失败
- 固定 320 字节读取缓冲区

详见 [[core/protocol/vless/format#实现边界|VLESS 实现边界]]

## 相关文档

- [[vmess]] - VMess 协议
- [[ref/protocol/vless-spec|vless-spec]] - VLESS 协议规范
- [[reality]] - Reality TLS 隐蔽
- [[ech]] - ECH 加密 Client Hello