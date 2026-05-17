---
title: "Authentication 实现"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/implementation
source: "mihomo-Meta/adapter/outbound/"
tags: [mihomo, authentication, auth, password, uuid, tls, implementation]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/implementation/overview]]"
  - "[[ref/crypto/aead-basics]]"
  - "[[ref/crypto/hkdf]]"
---

# Authentication 实现

**类别**: Mihomo 实现机制 | **模块**: 认证机制详解

## 概述

认证是代理协议的核心安全机制，确保只有授权用户可以使用代理服务。mihomo 支持多种认证方式。

## 认证方式总览

| 协议 | 认证方式 | 说明 |
|------|----------|------|
| SOCKS5 | 用户名/密码 | RFC 1929 |
| HTTP | Basic Auth | Base64 编码 |
| Trojan | 密码 | SHA224 哈希 |
| VLESS | UUID | 无状态认证 |
| VMess | UUID + AlterID | 动态 ID |
| Shadowsocks | 密码 | 密钥派生 |
| Hysteria2 | 密码 | 自定义认证 |
| TUIC | UUID + 密码 | 双重认证 |
| WireGuard | 密钥对 | 公钥认证 |

## 各协议认证详解

### SOCKS5 认证

SOCKS5 支持用户名/密码认证 (RFC 1929):

```yaml
proxies:
  - name: "socks5-auth"
    type: socks5
    server: server.com
    port: 1080
    username: "user"
    password: "password"
```

认证流程:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    SOCKS5 认证流程 (RFC 1929)                        │
└─────────────────────────────────────────────────────────────────────┘

客户端连接
    │
    ▼
┌─────────────────────┐
│  发送认证方法请求   │
│  VER: 05           │
│  NMETHODS: 02      │
│  METHODS: [00,02]  │  # 无认证 + 用户名密码
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  服务端选择方法     │
│  VER: 05           │
│  METHOD: 02        │  # 选择用户名密码
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  客户端发送认证     │
│  VER: 01           │
│  ULEN: 04          │
│  UNAME: "user"     │
│  PLEN: 08          │
│  PASSWD: "password"│
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  服务端返回结果     │
│  VER: 01           │
│  STATUS: 00        │  # 成功
└─────────────────────┘
```

### Trojan 认证

Trojan 使用密码进行 TLS握手后认证:

```yaml
proxies:
  - name: "trojan-auth"
    type: trojan
    server: server.com
    port: 443
    password: "password123"
    sni: server.com
```

认证流程:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Trojan 认证流程                                   │
└─────────────────────────────────────────────────────────────────────┘

1. TLS 连接
   ┌─────────────────────────────────────────────────────────────────┐
   │  客户端 TLS 连接到服务端                                        │
   │  SNI: server.com                                               │
   └─────────────────────────────────────────────────────────────────┘

2. 发送密码哈希
   ┌─────────────────────────────────────────────────────────────────┐
   │  密码: password123                                              │
   │  SHA224(password): hex string                                   │
   │  发送: SHA224_HASH + CRLF                                       │
   └─────────────────────────────────────────────────────────────────┘

3. 服务端验证
   ┌─────────────────────────────────────────────────────────────────┐
   │  服务端比对 SHA224 哈希                                         │
   │  匹配: 允许连接                                                 │
   │  不匹配: 模拟 HTTP 响应                                         │
   └─────────────────────────────────────────────────────────────────┘
```

SHA224 哈希:
```python
# Trojan 密码处理
import hashlib
password = "password123"
sha224_hash = hashlib.sha224(password.encode()).hexdigest()
# 结果: 56 字符十六进制串
```

### VLESS 认证

VLESS 使用 UUID 认证，无状态、无额外开销:

```yaml
proxies:
  - name: "vless-auth"
    type: vless
    server: server.com
    port: 443
    uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    # 无 AlterID，无加密开销
```

认证流程:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    VLESS 认证流程                                    │
└─────────────────────────────────────────────────────────────────────┘

连接请求
    │
    ▼
┌─────────────────────┐
│  发送 UUID          │
│  作为身份标识       │
│  无加密/无哈希      │
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  服务端验证 UUID    │
│  直接比对           │
│  快速高效           │
└─────────────────────┘

特点:
┌─────────────────────────────────────────────────────────────────────┐
│  - 无状态: 无需握手                                                 │
│  - 无加密: 无额外开销                                               │
│  - 快速: 直接比对                                                   │
│  - 安全: 依赖 TLS 加密                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### VMess 认证

VMess 使用 UUID + AlterID 动态认证:

```yaml
proxies:
  - name: "vmess-auth"
    type: vmess
    server: server.com
    port: 443
    uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    alterId: 0      # 现代推荐 alterId=0
    cipher: auto
```

AlterID 机制:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    VMess AlterID 机制                                │
└─────────────────────────────────────────────────────────────────────┘

alterId = 0:
┌─────────────────────────────────────────────────────────────────────┐
│  - 使用原始 UUID                                                    │
│  - 无动态 ID                                                        │
│  - 更安全、更简单                                                   │
│  - 推荐配置                                                         │
└─────────────────────────────────────────────────────────────────────┘

alterId > 0 (已弃用):
┌─────────────────────────────────────────────────────────────────────┐
│  - 动态生成 AlterID                                                 │
│  - 增加复杂性                                                       │
│  - 已不再推荐                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Shadowsocks 认证

Shadowsocks AEAD 使用密钥派生:

```yaml
proxies:
  - name: "ss-auth"
    type: ss
    server: server.com
    port: 8388
    cipher: aes-128-gcm
    password: "password123"
```

密钥派生:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Shadowsocks AEAD 密钥派生                         │
└─────────────────────────────────────────────────────────────────────┘

密码 → 密钥 (HKDF):
┌─────────────────────────────────────────────────────────────────────┐
│  password: "password123"                                            │
│                                                                      │
│  HKDF-SHA256:                                                       │
│  salt (随机) + password → key                                       │
│                                                                      │
│  key 用于 AEAD 加密                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

Shadowsocks 2022:
```yaml
proxies:
  - name: "ss2022-auth"
    type: ss
    server: server.com
    port: 443
    cipher: 2022-blake3-aes-128-gcm
    password: "base64-encoded-key"
```

特点:
- 使用预生成的密钥
- Blake3 哈希
- 更强的安全性

### Hysteria2 认证

Hysteria2 使用密码认证:

```yaml
proxies:
  - name: "hysteria2-auth"
    type: hysteria2
    server: server.com
    port: 443
    password: "password123"
    # QUIC 内置 TLS 认证 + 自定义密码
```

认证流程:
- TLS 证书验证
- 密码验证
- 双重保障

### TUIC 认证

TUIC 使用 UUID + 密码双重认证:

```yaml
proxies:
  - name: "tuic-auth"
    type: tuic
    server: server.com
    port: 443
    uuid: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    password: "password123"
```

认证方式:
- UUID 标识用户
- 密码验证身份
- QUIC TLS 加密

### WireGuard 认证

WireGuard 使用公钥认证:

```yaml
proxies:
  - name: "wireguard-auth"
    type: wireguard
    server: server.com
    port: 51820
    ip: 10.0.0.2
    private-key: "clientPrivateKeyBase64"
    public-key: "serverPublicKeyBase64"
    # 可选预共享密钥
    pre-shared-key: "pskBase64"
```

认证原理:
- 公钥加密握手
- 预共享密钥增强安全
- 无需传统密码

## 认证安全建议

### 密码选择

| 类型 | 建议 |
|------|------|
| Trojan 密码 | 随机生成 16+ 字符 |
| UUID | 真随机 UUID |
| Shadowsocks 密码 | 随机生成 |
| 预共享密钥 | 32 字节随机 |

### 密钥管理

```
┌─────────────────────────────────────┐
│       密钥管理建议                    │
├─────────────────────────────────────┤
│                                      │
│  1. 定期更换密码/密钥                 │
│  2. 不在多处使用相同密码               │
│  3. 安全存储配置文件                  │
│  4. 使用配置加密或混淆                 │
│                                      │
└─────────────────────────────────────┘
```

## 源码位置

- 各协议认证: `adapter/outbound/*.go`
- 密钥派生: 内部实现

## 相关链接

- [[overview]] — 实现总览
- [[ref/crypto/aead-basics]] — AEAD 加密基础
- [[ref/crypto/hkdf]] — HKDF 密钥派生
- [[ref/mihomo/compatibility/protocol-matrix]] — 协议支持矩阵