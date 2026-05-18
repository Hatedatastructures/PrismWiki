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

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SSH 隧道实现 |
| Pipeline | 部分 | 仅 TCP 流 |
| UDP 支持 | 不兼容 | SSH 不支持 UDP |

## 相关文档

- [[dev/debugging/tcp|tcp]] - TCP 连接