---
title: Shadowsocks
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, shadowsocks]
---
# Shadowsocks 协议

Shadowsocks 是一种基于 SOCKS5 的加密代理协议，最初由 clowwindy 于 2012 年发布。经过多轮迭代，该协议从最初的流式加密演进到 AEAD 模式，最终在 2022 年推出了基于 SIP022 规范的 Shadowsocks 2022（简称 SS2022），在安全性、性能和多用户支持方面实现了全面升级。mihomo 在 `transport/shadowsocks/` 下实现了完整的 Shadowsocks 客户端，涵盖传统 AEAD 和 SS2022。

## 协议背景

### 演进历史

```
2012  table (RC4-MD5 等)      ← 流式加密，无完整性校验，已被破解
  │
2014  AEAD (SIP004)           ← 分块 AEAD 加密，解决重放问题
  │
2022  SS2022 (SIP022)         ← BLAKE3 密钥派生 + 全新 TCP/UDP 帧格式 + 多用户
```

### SS2022 相比旧版 AEAD 的改进

| 特性 | 旧版 AEAD | SS2022 (SIP022) |
|------|-----------|-----------------|
| 密钥派生 | HKDF-SHA1 | BLAKE3 KeyDerive |
| TCP 盐 | 固定长度 Salt + 数据盐 | 无盐设计，基于会话状态 |
| UDP 帧 | 单次 AEAD 加密 | SeparateHeader + EncryptedHeader + Payload |
| 多用户 | 不支持 | 原生支持（PSK 数组） |
| 重放检测 | 无 | 服务端会话去重 |
| 头部加密 | 目标地址明文 | 目标地址加密 |

### 设计动机

SS2022 的重新设计旨在解决以下问题：

1. **Salt 开销**：旧版每个 TCP 连接需要发送随机 Salt（通常 16-32 字节），增加了握手延迟。
2. **UDP 效率**：旧版 UDP 将整个数据报用单个 AEAD 加密，大包时分片严重。
3. **密钥派生强度**：HKDF-SHA1 在现代计算能力下强度不足，BLAKE3 提供了更快的安全派生。
4. **多用户场景**：共享服务器需要多个密钥，SS2022 原生支持 PSK 数组。

## 帧格式

### 传统 AEAD TCP 帧

```
+------------------+----------------------------------------------+
| Salt (16-32B)    | 随机盐，首次连接时发送                         |
+------------------+----------------------------------------------+
| Encrypted Chunk  | AEAD 加密的数据块（见下方 Chunk 格式）         |
+------------------+----------------------------------------------+
```

### AEAD Chunk 格式（传统模式）

```
+-------------------+------------------+------------------+
| AuthLen (2 bytes) | Encrypted Data   | AEAD Tag (16 B)  |
+-------------------+------------------+------------------+
```

- `AuthLen`：大端序 uint16，表示加密数据长度（最高位为 1 表示最后一个 Chunk）。
- 每个 Chunk 使用独立派生的 Key/IV，计数器递增。

### SS2022 TCP 帧

SS2022 采用无盐设计，TCP 连接直接开始传输加密数据：

```
+--------------------------------------------------------------+
| AEAD Encrypted Stream (连续 AEAD Chunk 流)                    |
+--------------------------------------------------------------+
```

每个 Chunk 内部结构：

```
+----------+----------+---------------+------------------+
| Type (1) | Len (2)  | Payload (N)   | AEAD Tag (16 B)  |
+----------+----------+---------------+------------------+
```

- `Type`：0x00 = 数据，0x01 = 关闭连接。
- `Len`：大端序 uint16，Payload 长度。
- Key 和 IV 从 BLAKE3 派生的会话 Key 按计数器派生。

### SS2022 UDP 帧

UDP 帧由三部分组成：

```
+-------------------+---------------------+------------------+
| SeparateHeader    | EncryptedHeader     | EncryptedPayload |
| (16 bytes)        | (变长, AES-ECB 加密) | (变长, AEAD 加密)|
+-------------------+---------------------+------------------+
```

#### SeparateHeader

固定 16 字节，由会话状态派生：

```
+------------------+------------------+
| Nonce (8 bytes)  | Padding (8 bytes)|
+------------------+------------------+
```

#### EncryptedHeader

使用 AES-ECB 单块加密，包含目标地址信息：

```
+----------+----------+----------+----------+
| ATYP (1) | Address  | Port (2) | Padding  |
+----------+----------+----------+----------+
```

加密过程：

1. 将 Header 填充到 16 字节边界。
2. 使用 AES-ECB 模式加密（密钥从 BLAKE3 派生）。
3. AES-ECB 选择理由：单块加密无需 IV，简化 UDP 无状态传输。

#### EncryptedPayload

业务数据使用 AEAD 加密（与 TCP 相同算法）。

### 传统 UDP 帧

```
+------------------+-------------------------------------------+
| Salt (16-32B)    | 随机盐                                    |
+------------------+-------------------------------------------+
| AEAD Data        | 加密的 SOCKS5 地址 + Payload              |
+------------------+-------------------------------------------+
```

## 密钥派生

### SS2022 BLAKE3 派生流程

```
                    PSK (Password)
                         │
                         ▼
                   ┌───────────┐
                   │  BLAKE3   │  ← derive_key("shadowsocks 2022 session key" || PSK)
                   │  derive   │     产生 32 字节会话主密钥
                   └───────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │ TCP Key│ │UDP Key │ │Header│
         │        │ │        │ │  Key  │
         └────────┘ └────────┘ └────────┘
```

具体步骤：

1. **会话主密钥**：`BLAKE3::derive_key("shadowsocks 2022 session key", PSK, 32)`。
2. **TCP 传输密钥**：从主密钥按方向（客户端/服务端）派生。
3. **UDP Separate 密钥**：用于 SeparateHeader 的 Nonce 生成。
4. **UDP Header 密钥**：用于 AES-ECB 加密 EncryptedHeader。
5. **Chunk 密钥**：从传输密钥按计数器递增派生每个 Chunk 的 Key/IV。

多用户场景：

```
PSK_Array = [PSK_0, PSK_1, ..., PSK_N]
                 │
    对每个 PSK 计算 BLAKE3::derive_key → SessionKey
                 │
    服务端遍历 SessionKey 尝试解密，匹配即认证通过
```

### 传统 AEAD 密钥派生

```
Password + Cipher
       │
       ▼
  ┌─────────┐
  │HKDF-SHA1│  ← info = "ss-subkey"
  └─────────┘
       │
       ▼
  Subkey (按 Cipher 长度: 16/32 bytes)
```

## 通信流程

### TCP 通信时序（SS2022）

```
客户端                                                    服务端
   │                                                         │
   │  1. 从 PSK 派生会话密钥（BLAKE3）                        │
   │  2. 建立 TCP 连接                                       │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  3. 从 PSK 派生相同会话密钥
   │                                                         │
   │  4. 发送加密目标地址 Chunk                               │
   │     - Type=0x00, Len=地址长度                            │
   │     - Payload=SOCKS5 地址编码                            │
   │     - AEAD Tag                                          │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  5. 解密地址 Chunk
   │                                                         │  6. 建立到目标的连接
   │                                                         │
   │  7. 发送业务数据 Chunk 流                                │
   │ ◄────────────────────────────────────────────────────── │  8. 转发目标响应
   │                                                         │
   │  9. Type=0x01 关闭连接                                  │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │
```

### UDP 通信时序（SS2022）

```
客户端                                                    服务端
   │                                                         │
   │  1. 构造 SeparateHeader (Nonce + Padding)               │
   │  2. 构造目标地址 Header，AES-ECB 加密                    │
   │  3. 加密业务数据 (AEAD)                                 │
   │  4. 发送 UDP 包: SeparateHeader + EncHeader + EncData    │
   │ ──────────────────────────────────────────────────────► │
   │                                                         │  5. 用 Nonce 识别会话
   │                                                         │  6. AES-ECB 解密 EncryptedHeader
   │                                                         │  7. 解析目标地址
   │                                                         │  8. AEAD 解密 Payload
   │                                                         │  9. 转发到目标
   │                                                         │
   │                                                         │  10. 构造响应包（反向流程）
   │ ◄────────────────────────────────────────────────────── │
   │                                                         │
```

## 错误处理

### 认证失败

| 错误类型 | 触发条件 | 处理方式 |
|---------|---------|---------|
| PSK 不匹配（SS2022） | 所有 PSK 派生的密钥均无法解密数据 | 丢弃数据包，不返回任何信息 |
| 密钥派生失败 | PSK 格式无效或长度不足 | 连接初始化阶段报错 |
| AEAD Tag 校验失败 | 数据被篡改或密钥错误 | 丢弃当前 Chunk，标记连接异常 |

### 解密失败

| 场景 | 处理方式 |
|------|---------|
| TCP Chunk 解密失败 | 丢弃该 Chunk，继续处理后续（可能恢复） |
| UDP EncryptedHeader 解密失败 | 整个 UDP 包丢弃 |
| UDP Payload 解密失败 | 丢弃 Payload，不转发 |
| 连续解密失败 | 连接标记为不可信，超时关闭 |

### 重放检测

SS2022 服务端通过以下方式检测重放：

1. **会话状态**：维护活跃连接的 Nonce 集合。
2. **Nonce 去重**：相同 Nonce 在窗口期内只接受一次。
3. **时间窗口**：超过窗口期的 Nonce 被视为过期。
4. **无状态 UDP**：UDP 不维护完整会话，仅依赖 Nonce 短期去重。

### 协议版本不兼容

当客户端尝试使用 SS2022 格式连接仅支持旧版 AEAD 的服务端时：

- 服务端将 SS2022 无盐数据误认为 Salt 前缀。
- 首次 AEAD 解密必然失败。
- 连接立即关闭。

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Shadowsocks 协议实现，支持传统 AEAD 和 SS2022 |
| Pipeline | 完全兼容 | StreamConnContext 封装 TCP 流 |
| Crypto | 完全兼容 | crypto/aead、crypto/blake3、crypto/block 模块 |
| Multiplex | 部分兼容 | SS2022 TCP 内置多路复用，无需外部 mux |

Prism 的 Shadowsocks 实现注意点：

- `crypto/blake3` 模块用于 SS2022 密钥派生。
- `crypto/aead` 模块提供 Chunk 级 AEAD 加解密。
- `crypto/block` 模块提供 UDP Header 的 AES-ECB 单块加密。
- UDP SeparateHeader 的 Nonce 生成需要保证唯一性。
- 多用户场景下，服务端需维护 PSK 数组的派生密钥缓存。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/shadowsocks.go` | Shadowsocks 适配器 |
| `transport/shadowsocks/core/cipher.go` | 加密核心 |
| `transport/shadowsocks/shadowaead/` | AEAD 实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "ss-proxy"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    udp: true
```

### 支持的加密算法

| Cipher | 描述 |
|--------|------|
| `aes-128-gcm` | AES-128-GCM |
| `aes-256-gcm` | AES-256-GCM |
| `chacha20-ietf-poly1305` | ChaCha20-Poly1305 |
| `2022-blake3-aes-128-gcm` | Shadowsocks 2022 |
| `2022-blake3-aes-256-gcm` | Shadowsocks 2022 |
| `2022-blake3-chacha20-poly1305` | Shadowsocks 2022 |

### obfs 插件

```yaml
proxies:
  - name: "ss-obfs"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: obfs
    plugin-opts:
      mode: tls
      host: bing.com
```

### v2ray-plugin 插件

```yaml
proxies:
  - name: "ss-v2ray"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: v2ray-plugin
    plugin-opts:
      mode: websocket
      tls: true
      host: server.example.com
      path: /ss
```

### ShadowTLS 插件

```yaml
proxies:
  - name: "ss-shadowtls"
    type: ss
    server: server.example.com
    port: 443
    cipher: aes-128-gcm
    password: your-password
    plugin: shadowtls
    plugin-opts:
      password: shadowtls-password
      host: bing.com
      version: 2
```

### Restls 插件

```yaml
proxies:
  - name: "ss-restls"
    type: ss
    server: server.example.com
    port: 443
    cipher: aes-128-gcm
    password: your-password
    plugin: restls
    plugin-opts:
      password: restls-password
      host: bing.com
      version-hint: tls12
```

### kcptun 插件

```yaml
proxies:
  - name: "ss-kcptun"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    plugin: kcptun
    plugin-opts:
      key: kcptun-key
      crypt: aes
      mode: fast
```

### UDP over TCP

```yaml
proxies:
  - name: "ss-uot"
    type: ss
    server: server.example.com
    port: 8388
    cipher: aes-128-gcm
    password: your-password
    udp-over-tcp: true
    udp-over-tcp-version: 2
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `ss` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `cipher` | string | 是 | 加密算法 |
| `password` | string | 是 | 密码 |
| `udp` | bool | 否 | 启用 UDP |
| `plugin` | string | 否 | 插件名称 |
| `plugin-opts` | map | 否 | 插件配置 |

## Prism 实现边界

- **timestamp_window 默认 30s**：两端时钟需同步，偏移超过窗口导致 `timestamp_expired`
- **"乐观响应"设计**：先 ack 再拨号，拨号失败客户端看到空连接
- salt_pool 高连接速率下内存线性增长
- UDP session_tracker 无效 session 不清理

详见 [[core/protocol/shadowsocks/relay#实现边界|SS2022 实现边界]]

## 相关文档

- [[shadowsocksr]] - ShadowsocksR 协议
- [[ref/protocol/shadowsocks-spec|shadowsocks-spec]] - Shadowsocks 协议规范
- [[core/crypto/aead|aead]] - AEAD 加密
- [[../transport/shadowsocks]] - Shadowsocks 传输层
