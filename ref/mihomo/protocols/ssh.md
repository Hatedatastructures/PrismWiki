---
title: SSH
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, ssh]
---
# SSH 协议

SSH 隧道利用 SSH 协议建立代理连接。

## 协议概述

SSH 代理特性：
- 基于 SSH 协议（RFC 4254）
- 支持密码和密钥认证
- 支持主机密钥验证
- 仅支持 TCP

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/ssh.go` | SSH 适配器 |

## YAML 配置示例

### 密码认证

```yaml
proxies:
  - name: "ssh-password"
    type: ssh
    server: server.example.com
    port: 22
    username: user
    password: pass
```

### 密钥认证

```yaml
proxies:
  - name: "ssh-key"
    type: ssh
    server: server.example.com
    port: 22
    username: user
    private-key: |
      -----BEGIN OPENSSH PRIVATE KEY-----
      ...
      -----END OPENSSH PRIVATE KEY-----
```

### 密钥文件路径

```yaml
proxies:
  - name: "ssh-key-file"
    type: ssh
    server: server.example.com
    port: 22
    username: user
    private-key: /path/to/private_key
    private-key-passphrase: key-passphrase
```

### 主机密钥验证

```yaml
proxies:
  - name: "ssh-hostkey"
    type: ssh
    server: server.example.com
    port: 22
    username: user
    password: pass
    host-key:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
    host-key-algorithms:
      - ssh-ed25519
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `ssh` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | SSH 端口 |
| `username` | string | 是 | SSH 用户名 |
| `password` | string | 否 | 密码 |
| `private-key` | string | 否 | 私钥内容或路径 |
| `private-key-passphrase` | string | 否 | 私钥密码 |
| `host-key` | []string | 否 | 主机公钥 |
| `host-key-algorithms` | []string | 否 | 主机密钥算法 |

## SSH 隧道原理

SSH（Secure Shell）是一种加密网络协议，最初设计用于安全的远程登录，但其端口转发（Port Forwarding）功能使其成为强大的代理工具。

### SSH 代理的三种模式

| 模式 | 类型 | 说明 |
|------|------|------|
| 本地端口转发 | -L | 本地端口 → SSH 服务器 → 目标服务器 |
| 远程端口转发 | -R | 远程端口 → SSH 客户端 → 目标服务器 |
| 动态端口转发 | -D | SOCKS 代理，任意目标地址 |

mihomo 使用的是 **动态端口转发（-D）** 模式的客户端行为——通过 SSH 连接建立代理隧道。

### SSH 协议架构

```
┌─────────────────────────────────────────────┐
│              SSH 协议栈                        │
├─────────────────────────────────────────────┤
│  传输层协议 (RFC 4253)                        │
│  ├─ 密钥交换 (Diffie-Hellman)                 │
│  ├─ 服务器认证                                 │
│  ├─ 加密 (AES, ChaCha20)                     │
│  └─ 完整性 (HMAC)                             │
│       │                                       │
│       ▼                                       │
│  用户认证协议 (RFC 4252)                        │
│  ├─ 密码认证                                   │
│  ├─ 公钥认证                                   │
│  └─ 键盘交互认证                                │
│       │                                       │
│       ▼                                       │
│  连接协议 (RFC 4254)                           │
│  ├─ 通道多路复用                                │
│  ├─ 全局请求                                   │
│  └─ 通道请求（端口转发/Shell/Exec）             │
└─────────────────────────────────────────────┘
```

## SSH 协议握手流程

### 阶段一：版本交换

```
Client → Server:  SSH-2.0-OpenSSH_9.0
Server → Client:  SSH-2.0-OpenSSH_8.9
```

双方交换版本字符串，协商使用 SSH-2.0 协议。格式为 `SSH-protoversion-softwareversion`。

### 阶段二：密钥交换（KEX）

```
┌──────────────────────────────────────────────────┐
│              密钥交换流程 (ECDH)                    │
├──────────────────────────────────────────────────┤
│                                                  │
│  Client                          Server          │
│    |                               |              │
│    |  --- KEX_INIT (算法协商) ---> |              │
│    |  <-- KEX_INIT (算法选择) ---- |              │
│    |                               |              │
│    |  --- ECDH 公钥 ------------>  |              │
│    |  <-- ECDH 公钥 + 签名 ------  |              │
│    |                               |              │
│    |  [计算共享密钥 → 派生会话密钥]  |              │
│    |                               |              │
│    |  --- NEW_KEYS (切换加密) ----> |              │
│    |  <-- NEW_KEYS (确认) -------- |              │
│    |                               |              │
│    |  [后续所有通信加密传输]         |              │
│                                                  │
└──────────────────────────────────────────────────┘
```

**密钥交换算法协商**：

KEX_INIT 消息中包含双方支持的算法列表：

| 算法类别 | 示例 |
|---------|------|
| KEX 算法 | curve25519-sha256, diffie-hellman-group16-sha512 |
| Host Key 算法 | ssh-ed25519, rsa-sha2-512, ecdsa-sha2-nistp256 |
| 加密算法 | chacha20-poly1305@openssh.com, aes256-gcm@openssh.com |
| MAC 算法 | hmac-sha2-256-etm@openssh.com |
| 压缩算法 | none, zlib |

### 阶段三：用户认证

```
Client → Server:  SSH_MSG_USERAUTH_REQUEST
Server → Client:  SSH_MSG_USERAUTH_SUCCESS / FAILURE
```

#### 密码认证

```
SSH_MSG_USERAUTH_REQUEST:
  +------------------+
  | user_name        |  ← 用户名
  | service_name     |  ← "ssh-connection"
  | method_name      |  ← "password"
  | FALSE            |  ← 非密码变更
  | password         |  ← 密码（明文，已加密传输）
  +------------------+
```

#### 公钥认证

```
SSH_MSG_USERAUTH_REQUEST:
  +------------------+
  | user_name        |  ← 用户名
  | service_name     |  ← "ssh-connection"
  | method_name      |  ← "publickey"
  | TRUE             |  ← 实际签名
  | algorithm        |  ← 公钥算法
  | public_key       |  ← 公钥 blob
  | signature        |  ← 会话签名
  +------------------+
```

公钥认证流程：
1. 客户端先发送不带签名的请求探测服务器是否接受该公钥
2. 服务器返回 `PK_OK` 表示接受
3. 客户端使用私钥对会话数据签名后发送完整请求
4. 服务器使用公钥验证签名

### 阶段四：通道建立（Channel）

```
┌──────────────────────────────────────────┐
│        代理通道建立流程                     │
├──────────────────────────────────────────┤
│                                          │
│  1. 客户端发送通道打开请求                  │
│     SSH_MSG_CHANNEL_OPEN                  │
│     channel_type = "direct-tcpip"         │
│     target_host = "example.com"           │
│     target_port = 443                     │
│                                          │
│  2. 服务器确认通道打开                      │
│     SSH_MSG_CHANNEL_OPEN_CONFIRMATION      │
│                                          │
│  3. 双向数据传输                            │
│     SSH_MSG_CHANNEL_DATA                  │
│     SSH_MSG_CHANNEL_EXTENDED_DATA         │
│                                          │
│  4. 通道关闭                                │
│     SSH_MSG_CHANNEL_CLOSE                 │
│                                          │
└──────────────────────────────────────────┘
```

`direct-tcpip` 通道格式（RFC 4254 Section 7.2）：

```
string    channel_type = "direct-tcpip"
uint32    sender_channel
uint32    initial_window_size
uint32    maximum_packet_size
string    target_host     ← 目标主机名
uint32    target_port     ← 目标端口
string    originator_IP   ← 客户端 IP
uint32    originator_port ← 客户端端口
```

## 主机密钥验证

SSH 通过主机密钥验证防止中间人攻击：

```yaml
host-key:
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..."
host-key-algorithms:
  - ssh-ed25519
```

| 主机密钥类型 | 密钥大小 | 安全性 |
|-------------|---------|--------|
| ssh-ed25519 | 256 bit | 最高 |
| ecdsa-sha2-nistp256 | 256 bit | 高 |
| rsa-sha2-512 | 4096 bit | 高 |
| ssh-rsa | 2048+ bit | 中等 |

## 错误处理

| 错误类型 | 原因 | 处理 |
|---------|------|------|
| 主机密钥不匹配 | 服务器密钥与缓存不同 | 连接拒绝（防 MITM） |
| 认证失败 | 密码错误或公钥不被接受 | 连接断开 |
| 通道打开失败 | 目标地址不可达 | 返回通道错误 |
| 连接超时 | 服务器无响应 | 重连或失败 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SSH 隧道实现 |
| Pipeline | 部分 | 仅 TCP 流 |
| UDP 支持 | 不兼容 | SSH 不支持 UDP |

## 相关文档

- [[dev/debugging/tcp|tcp]] - TCP 连接