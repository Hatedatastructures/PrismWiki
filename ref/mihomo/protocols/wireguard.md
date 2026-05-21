---
title: WireGuard
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, wireguard]
---
# WireGuard 协议

WireGuard 是现代 VPN 协议，提供简洁高效的 VPN 连接。

## 协议概述

WireGuard 特性：
- 现代 VPN 协议
- 高性能、低延迟
- 支持多 Peer
- 支持 AmneziaWG 扩展
- 内置加密（ChaCha20-Poly1305）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/wireguard.go` | WireGuard 适配器 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "wg-proxy"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    ipv6: fd00::2
    private-key: your-private-key
    public-key: server-public-key
    udp: true
```

### AmneziaWG 配置

```yaml
proxies:
  - name: "amnezia-wg"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    amnezia-wg-option:
      jc: 3
      jmin: 50
      jmax: 1000
      s1: 75
      s2: 75
      h1: 1234567890
      h2: 1234567890
      h3: 1234567890
      h4: 1234567890
```

### 多 Peer 配置

```yaml
proxies:
  - name: "wg-multi-peer"
    type: wireguard
    private-key: your-private-key
    peers:
      - server: server1.example.com
        port: 51820
        public-key: peer1-public-key
        pre-shared-key: psk1
        allowed-ips:
          - 10.0.0.0/24
      - server: server2.example.com
        port: 51820
        public-key: peer2-public-key
        allowed-ips:
          - 10.1.0.0/24
    ip: 10.0.0.2
```

### 远程 DNS 解析

```yaml
proxies:
  - name: "wg-dns"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    remote-dns-resolve: true
    dns:
      - 1.1.1.1
      - 8.8.8.8
```

### Reserved 字段

```yaml
proxies:
  - name: "wg-reserved"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    reserved:
      - 100
      - 200
      - 300
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `wireguard` |
| `server` | string | 是* | 服务器地址 |
| `port` | int | 是* | 服务器端口 |
| `ip` | string | 是 | 本地 IPv4 |
| `ipv6` | string | 否 | 本地 IPv6 |
| `private-key` | string | 是 | 本地私钥 |
| `public-key` | string | 是 | 远端公钥 |
| `pre-shared-key` | string | 否 | PSK |
| `reserved` | []uint8 | 否 | Reserved 字段 |
| `mtu` | int | 否 | MTU（默认 1408） |
| `workers` | int | 否 | 工线程数 |
| `udp` | bool | 否 | 启用 UDP |

## 协议背景

WireGuard 是由 Jason A. Donenfeld 设计的现代化 VPN 协议，于 2020 年正式进入 Linux 内核 5.6。与 IPSec 和 OpenVPN 等传统 VPN 协议不同，WireGuard 从设计之初就追求简洁性——整个协议仅约 4000 行代码，而 OpenVPN 超过 10 万行。

### 设计哲学

- **简洁性**：协议规范不到 500 行，代码量极小
- **高性能**：内核态实现，零拷贝，使用现代加密原语
- **安全性**：默认强制加密，使用前向保密（Forward Secrecy）
- **漫游支持**：通过 Endpoint 更新实现 NAT 穿透

### 加密原语

WireGuard 使用 Noise_IK 框架（基于 Noise Protocol Framework），具体加密组件：

| 组件 | 算法 | 用途 |
|------|------|------|
| DH 函数 | Curve25519 | 椭圆曲线 Diffie-Hellman 密钥交换 |
| 哈希函数 | BLAKE2s | 散列和 HKDF |
| 对称加密 | ChaCha20-Poly1305 | AEAD 加密数据包 |
| KDF | HKDF | 密钥派生 |
| MAC | Poly1305 | 消息认证码 |

## 帧格式：WireGuard 数据包结构

WireGuard 在 UDP 之上传输，所有数据包都以 32 位 Type 字段开头。

### 通用包头

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Message Type                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Message Type 的值决定了后续字段的含义：
- `1` — Handshake Initiation
- `2` — Handshake Response
- `3` — Cookie Reply
- `4` — Transport Data

### Handshake Initiation（Type = 1）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Type = 1            |                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                 +
|                                                               |
|                  Sender Index (32 bytes LE)                    |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Ephemeral Public Key (32 bytes)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Unencrypted Ephemeral (16 bytes)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Encrypted Static + Timestamp (48 bytes)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               MAC1 (16 bytes)                                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               MAC2 (16 bytes, optional)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Sender Index**：发送方分配的 32 位索引，用于标识发送方
- **Ephemeral Public Key**：临时 Curve25519 公钥
- **Unencrypted Ephemeral**：未加密的临时数据（包含 Noise 协议的 `e` 消息）
- **Encrypted Static**：使用 Noise 协议加密的静态公钥和时间戳
- **MAC1/MAC2**：消息认证码，用于防重放和 Underload 攻击

### Handshake Response（Type = 2）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Type = 2            |                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                 +
|                                                               |
|              Sender Index (32 bytes LE)                        |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              Receiver Index (32 bytes LE)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Null Ciphertext (16 bytes)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               MAC1 (16 bytes)                                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               MAC2 (16 bytes, optional)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Sender Index**：响应方的 32 位索引
- **Receiver Index**：对应 Initiation 发送方的索引
- **Null Ciphertext**：加密的零长度消息，用于完成密钥交换

### Cookie Reply（Type = 3）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Type = 3            |                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                 +
|              Receiver Index (32 bytes LE)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Nonce (24 bytes)                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Encrypted Cookie (32 bytes)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

当接收方检测到 Underload 状态时，会发送 Cookie Reply 要求发送方在后续消息中包含 Cookie 作为反滥用机制。

### Transport Data（Type = 4）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Type = 4            |                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                 +
|              Receiver Index (32 bytes LE)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Counter (8 bytes LE)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Encrypted Payload (variable + 16 bytes tag)      |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Receiver Index**：对应接收方之前分配的索引
- **Counter**：64 位单调递增计数器，用于防重放攻击
- **Encrypted Payload**：使用 ChaCha20-Poly1305 加密的 IP 数据包
- **Auth Tag**：16 字节的 Poly1305 认证标签（附加在密文后）

## 握手流程：Noise_IK 模式

WireGuard 的握手基于 Noise Protocol Framework 的 IK 模式（Identity-Key），包含四次消息交换中的前两次（握手阶段）：

### 阶段一：Handshake Initiation → Response

```
Client                                          Server
  |                                                 |
  |  --- Handshake Initiation (Type 1) ---------->  |
  |    e ->                                         |
  |    s (encrypted)                                |
  |    t (timestamp, encrypted)                     |
  |                                                 |
  |  <--- Handshake Response (Type 2) -----------   |
  |    e <-                                         |
  |    ee, es                                       |
  |    MAC1/MAC2                                    |
  |                                                 |
```

1. **客户端生成临时密钥对**：`(ephemeral_secret, ephemeral_public)`
2. **计算 DH**：
   - `e`: 客户端临时公钥 → 服务器静态公钥（`es` DH）
   - `ee`: 客户端临时公钥 → 服务器临时公钥（`ee` DH）
3. **密钥派生**：通过 HKDF 从 DH 结果派生出会话密钥
4. **服务端响应**：发送自己的临时公钥，完成 `ee` 和 `es` DH 交换

### 阶段二：密钥派生

握手成功后，双方通过 HKDF 从以下 DH 结果派生出两对密钥：

| 密钥对 | 用途 |
|--------|------|
| 密钥 1（发送 → 接收） | 加密从客户端到服务器的数据 |
| 密钥 2（接收 ← 发送） | 加密从服务器到客户端的数据 |

每个密钥对包含：
- ChaCha20 加密密钥（32 字节）
- 计数器状态（用于防重放）

### 密钥重协商

WireGuard 没有显式的密钥重协商协议。而是使用以下机制：

- **时间触发**：密钥存活超过 120 秒后，下一次发送数据会触发新握手
- **数据量触发**：密钥加密超过 2^60 个数据包后强制重协商
- **静默重协商**：新握手在后台进行，不影响数据传输

## CryptoKey 路由表：AllowedIPs 机制

WireGuard 的核心路由机制通过 `AllowedIPs` 实现，它既是路由表也是加密密钥绑定表。

### 工作原理

```
┌─────────────────────────────────────────────────┐
│              WireGuard CryptoKey Routing          │
├─────────────────────────────────────────────────┤
│  Peer A (PublicKey: AAAA...)                     │
│    AllowedIPs: 10.0.0.0/24, fd00::/64            │
│    Endpoint: 1.2.3.4:51820                       │
│                                                  │
│  Peer B (PublicKey: BBBB...)                     │
│    AllowedIPs: 192.168.1.0/24                    │
│    Endpoint: 5.6.7.8:51820                       │
└─────────────────────────────────────────────────┘
```

- **发送路由**：当需要发送数据包到某个 IP 时，查找匹配 `AllowedIPs` 的 Peer，使用该 Peer 的公钥加密
- **接收验证**：当收到数据包解密后，检查源 IP 是否在该 Peer 的 `AllowedIPs` 范围内，否则丢弃

### mihomo 中的多 Peer 路由

在 mihomo 中，`AllowedIPs` 通过 `peers[].allowed-ips` 配置：

```yaml
peers:
  - server: peer1.example.com
    port: 51820
    public-key: key1
    allowed-ips:
      - "0.0.0.0/0"        # 所有 IPv4 流量
      - "::/0"              # 所有 IPv6 流量
  - server: peer2.example.com
    port: 51821
    public-key: key2
    allowed-ips:
      - "10.0.0.0/8"       # 仅内网流量
```

### 与系统路由表的对比

| 特性 | 系统路由表 | WireGuard AllowedIPs |
|------|-----------|---------------------|
| 匹配依据 | 目标 IP | 目标 IP + 源 IP |
| 加密绑定 | 无 | 与 Peer 公钥绑定 |
| 入口验证 | 无 | 自动验证源 IP 合法性 |

## 错误处理

### 握手超时

```
┌──────────────────────────────────────────┐
│  握手超时处理流程                          │
├──────────────────────────────────────────┤
│  1. 发送 Handshake Initiation            │
│  2. 等待 Response（默认 5 秒）             │
│  3. 超时 → 重传（指数退避）                │
│  4. 最大重试次数耗尽 → 标记 Peer 不可达    │
│  5. Keepalive 间隔内无响应 → 连接失效      │
└──────────────────────────────────────────┘
```

- **重传策略**：指数退避，初始 5 秒，最大 10 秒
- **Keepalive**：可配置 `persistent-keepalive`，定期发送空数据包维持 NAT 映射
- **Peer 状态**：长时间无响应的 Peer 不会被自动删除，但数据包会被丢弃

### 认证失败

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| MAC1 验证失败 | 消息被篡改或伪造 | 直接丢弃，不响应 |
| MAC2 验证失败 | Cookie 过期或不匹配 | 发送新 Cookie Reply |
| Counter 重放 | 计数器值小于等于已接收值 | 丢弃数据包 |
| 解密失败 | AEAD 认证标签不匹配 | 丢弃数据包 |
| 时间戳过期 | Initiation 中的时间戳过期（>120 秒） | 丢弃并忽略 |

### Underload 防护

当服务器检测到大量无效握手请求时（Underload 状态），会：

1. 停止处理无 MAC2 的 Initiation
2. 发送 Cookie Reply 要求客户端携带 Cookie
3. Cookie 有效期 120 秒，绑定客户端 IP

## AmneziaWG 扩展

AmneziaWG 是 WireGuard 的分支，增加反 DPI（深度包检测）混淆能力。

### 混淆参数

| 参数 | 含义 | 作用 |
|------|------|------|
| `jc` | Junk Count | 数据包中添加的垃圾数据块数量 |
| `jmin` | Junk Min Size | 每个垃圾数据块的最小大小 |
| `jmax` | Junk Max Size | 每个垃圾数据块的最大大小 |
| `s1` | Init Size | Initiation 消息的额外填充大小 |
| `s2` | Response Size | Response 消息的额外填充大小 |
| `h1-h4` | Header Magic | 自定义魔数，替换 WireGuard 标准 Type 字段 |

### 工作原理

通过在数据包中添加随机垃圾数据和修改头部魔数，使 WireGuard 流量无法被 DPI 识别为 VPN 流量：

```
标准 WireGuard:  [Type=1][固定格式...]  ← 容易被 DPI 识别
AmneziaWG:       [h1][垃圾数据][h2][加密数据][h3]... ← 看起来像随机数据
```

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | WireGuard VPN 实现 |
| Pipeline | 完全兼容 | L3 协议，支持 IP 包封装 |
| Multiplex | 不适用 | VPN 协议本身已多路复用 |
| Crypto | 部分兼容 | 需 ChaCha20-Poly1305 + Curve25519 |

## 相关文档

- [[masque]] - MASQUE 协议
- [[ref/crypto/chacha20-poly1305|chacha20-poly1305]] - ChaCha20 加密