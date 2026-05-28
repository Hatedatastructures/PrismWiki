---
title: ShadowsocksR
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, shadowsocksr]
---
# ShadowsocksR 协议

ShadowsocksR（SSR）是 Shadowsocks 的扩展版本，增加混淆和协议扩展。

## 协议概述

ShadowsocksR 特性：
- 扩展加密算法
- 支持混淆（obfs）
- 支持协议扩展
- 兼容原版 Shadowsocks

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/shadowsocksr.go` | SSR 适配器 |
| `transport/ssr/obfs/` | 混淆实现 |
| `transport/ssr/protocol/` | 协议实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "ssr-proxy"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: tls1.2_ticket_auth
    obfs-param: ""
    protocol: auth_aes128_md5
    protocol-param: ""
    udp: true
```

### HTTP Simple 混淆

```yaml
proxies:
  - name: "ssr-http"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: http_simple
    obfs-param: bing.com
    protocol: origin
```

### TLS 混淆

```yaml
proxies:
  - name: "ssr-tls"
    type: ssr
    server: server.example.com
    port: 8388
    cipher: aes-128-cfb
    password: your-password
    obfs: tls1.2_ticket_auth
    protocol: auth_aes128_sha1
```

## 支持的加密算法

| Cipher | 描述 |
|--------|------|
| `aes-128-cfb` | AES-128-CFB |
| `aes-192-cfb` | AES-192-CFB |
| `aes-256-cfb` | AES-256-CFB |
| `aes-128-ctr` | AES-128-CTR |
| `aes-192-ctr` | AES-192-CTR |
| `aes-256-ctr` | AES-256-CTR |
| `rc4-md5` | RC4-MD5（不推荐） |
| `chacha20` | ChaCha20 |
| `chacha20-ietf` | ChaCha20-IETF |
| `none` | 无加密 |

## 混淆类型

| Obfs | 描述 |
|------|------|
| `plain` | 无混淆 |
| `http_simple` | HTTP 混淆 |
| `http_post` | HTTP POST 混淆 |
| `tls1.2_ticket_auth` | TLS 混淆 |
| `random_head` | 随机头部 |

## 协议类型

| Protocol | 描述 |
|----------|------|
| `origin` | 原始协议 |
| `auth_sha1_v4` | SHA1 认证 |
| `auth_aes128_md5` | AES128-MD5 认证 |
| `auth_aes128_sha1` | AES128-SHA1 认证 |
| `auth_chain_a` | Chain A 认证 |
| `auth_chain_b` | Chain B 认证 |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `ssr` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `cipher` | string | 是 | 加密算法 |
| `password` | string | 是 | 密码 |
| `obfs` | string | 是 | 混淆类型 |
| `obfs-param` | string | 否 | 混淆参数 |
| `protocol` | string | 是 | 协议类型 |
| `protocol-param` | string | 否 | 协议参数 |
| `udp` | bool | 否 | 启用 UDP |

## 协议背景

ShadowsocksR（SSR）是 Shadowsocks 的一个分支，由 breakwa11 于 2016 年开发。SSR 在原版 Shadowsocks 的基础上增加了**混淆层**（Obfuscation）和**协议层**（Protocol），使其能够更有效地绕过深度包检测（DPI）。

### SSR 与原版 SS 的架构差异

```
┌──────────────────────────────────────────────────┐
│              Shadowsocks 原版                       │
│  [Payload] → [流加密] → [TCP/UDP]                  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│              ShadowsocksR                          │
│  [Payload] → [协议层] → [流加密] → [混淆层] → [TCP]│
└──────────────────────────────────────────────────┘
```

SSR 增加了两个额外层次：
1. **协议层（Protocol）**：在加密前对数据包进行变换，增加认证和抗重放能力
2. **混淆层（Obfuscation）**：在加密后对数据进行伪装，使其看起来像正常流量

### 历史背景

- 2016 年：breakwa11 发布 SSR，引入混淆和协议插件
- 2017 年：SSR 成为流行的代理协议
- 2017 年后：原版 Shadowsocks 转向 AEAD 加密，SSR 逐渐被取代

## 协议头部格式

### SSR 数据包结构

```
┌─────────────────────────────────────────────┐
│                  SSR 数据包                    │
├─────────────────────────────────────────────┤
│  [协议头部] [加密数据] [HMAC 校验]            │
│     │           │          │                 │
│     │           │          └─ 完整性验证       │
│     │           └─ 流加密载荷                │
│     └─ rand_len + data_len + CRC            │
└─────────────────────────────────────────────┘
```

### 协议头部详解

```
+-----------+----------+------+------+-----------+------+
| rand_len  | data_len | CRC  | IV   | rand_data | HMAC |
| (1-2 字节) |  (2 字节) | 2 字节 | 可变  | 可变      | 可变  |
+-----------+----------+------+------+-----------+------+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| rand_len | 1-2 字节 | 随机数据长度（由协议类型决定） |
| data_len | 2 字节 | 数据长度（小端序） |
| CRC | 2 字节 | CRC32 校验值的高 16 位 |
| IV | 可变 | 初始化向量（由加密算法决定） |
| rand_data | 可变 | 随机填充数据 |
| HMAC | 可变 | HMAC 校验值 |

## 混淆模式详解

### HTTP 混淆（http_simple）

将数据包伪装成 HTTP 请求，使 DPI 误认为是正常的 Web 流量。

```
混淆后的数据流示例：
GET / HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
[加密的代理数据]
```

**http_post** 变体使用 POST 方法：

```
POST / HTTP/1.1
Host: www.example.com
Content-Length: [加密数据长度]
[加密的代理数据]
```

### TLS 混淆（tls1.2_ticket_auth）

模拟 TLS 1.2 握手流程中的 Client Hello 消息：

```
混淆后的数据流：
[伪造的 TLS Client Hello 头部]
[加密的代理数据]
[伪造的 TLS Application Data 标记]
```

特点：
- 模拟 TLS 1.2 的 `Content Type = 0x17`（Application Data）
- 使用 TLS 握手的 `0x16` 类型标记
- 支持 `tls1.2_ticket_fastauth` 变体（简化版认证）

### 随机头部（random_head）

在数据包前添加随机长度的随机数据，使流量特征更加模糊：

```
[随机字节序列（4-128 字节）]
[加密的代理数据]
```

## 协议类型详解

### origin（原始协议）

等同于原版 Shadowsocks，不做任何变换。

### auth_sha1_v4

基于 SHA1 的认证协议，增加数据包完整性验证：

```
数据包格式：
[数据长度（2 字节）] [随机填充] [加密数据] [SHA1 HMAC]
```

- 每个数据包附加 SHA1 HMAC 校验
- 防止数据篡改和重放攻击
- 随机填充增加流量分析的难度

### auth_aes128_md5 / auth_aes128_sha1

基于 AES-128 的认证协议，提供更强的安全性：

```
数据包格式：
[协议头部] [加密数据] [AES-128 HMAC]
    │           │           │
    ├─ 包含连接 ID             │
    ├─ 包含包序号              │
    └─ 防止重放攻击            │
```

特点：
- 每个连接有唯一 ID，防止连接重放
- 每个数据包有序号，防止包重放
- 使用 AES-128 计算 HMAC

### auth_chain_a / auth_chain_b

Chain 系列协议进一步优化了性能和安全性：

- **auth_chain_a**：优化了随机数生成，减少开销
- **auth_chain_b**：增加了更多随机填充，提高抗分析能力

## 混淆参数（obfs-param）

不同混淆模式支持不同的参数：

| 混淆模式 | 参数格式 | 示例 |
|---------|---------|------|
| http_simple | 目标主机名 | `bing.com` |
| http_post | 目标主机名 | `www.google.com` |
| tls1.2_ticket_auth | 目标主机名 | `cloudflare.com` |
| random_head | 无 | （留空） |

## 错误处理

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| HMAC 校验失败 | 数据被篡改 | 丢弃数据包 |
| 协议版本不匹配 | 客户端/服务器协议不兼容 | 连接失败 |
| 混淆参数错误 | obfs-param 配置无效 | 回退到 plain |
| CRC 校验失败 | 数据包损坏 | 丢弃并请求重传 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SSR 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Crypto | 部分 | 使用流加密（非 AEAD） |

## 相关文档

- [[shadowsocks]] - Shadowsocks 协议
- [[ref/protocol/shadowsocks-spec|shadowsocks-spec]] - Shadowsocks 协议规范
- [[ref/mihomo/transport/ssr]] - SSR 传输层