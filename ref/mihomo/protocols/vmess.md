---
title: VMess
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, vmess]
---
# VMess 协议

VMess 是 V2Ray 的核心代理协议，提供内置加密和认证。

## 协议概述

VMess 特性：
- 内置 AEAD 加密
- 使用 UUID 认证
- 支持多种传输层（TCP、WS、gRPC、H2、HTTP）
- 支持 AlterID（已弃用）
- 支持 Global Padding 和 Authenticated Length

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/vmess.go` | VMess 适配器 |
| `transport/vmess/vmess.go` | VMess 传输层实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "vmess-proxy"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    udp: true
```

### WebSocket 配置

```yaml
proxies:
  - name: "vmess-ws"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: ws
    tls: true
    ws-opts:
      path: /vmess
      headers:
        Host: server.example.com
```

### gRPC 配置

```yaml
proxies:
  - name: "vmess-grpc"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: grpc
    tls: true
    grpc-opts:
      grpc-service-name: vmess
```

### HTTP/2 配置

```yaml
proxies:
  - name: "vmess-h2"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    network: h2
    tls: true
    h2-opts:
      host:
        - server.example.com
      path: /vmess
```

### HTTP 配置

```yaml
proxies:
  - name: "vmess-http"
    type: vmess
    server: server.example.com
    port: 80
    uuid: your-uuid
    network: http
    http-opts:
      method: GET
      path:
        - /vmess
      headers:
        Host:
          - server.example.com
```

### Reality 配置

```yaml
proxies:
  - name: "vmess-reality"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    reality-opts:
      public-key: xxx
      short-id: xxx
    client-fingerprint: chrome
```

### XUDP 配置

```yaml
proxies:
  - name: "vmess-xudp"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    xudp: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `vmess` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `uuid` | string | 是 | 用户 UUID |
| `alterId` | int | 否 | AlterID（已弃用） |
| `cipher` | string | 否 | 加密算法 |
| `tls` | bool | 否 | 启用 TLS |
| `network` | string | 否 | 传输方式 |
| `udp` | bool | 否 | 启用 UDP |
| `xudp` | bool | 否 | 启用 XUDP |
| `servername` | string | 否 | TLS 服务器名称（覆盖 SNI） |
| `packet-addr` | bool | 否 | 启用 PacketAddr 模式（与 xudp 互斥） |
| `packet-encoding` | string | 否 | UDP 包编码：packetaddr/xudp |
| `global-padding` | bool | 否 | 启用全局填充（增强隐蔽性） |
| `authenticated-length` | bool | 否 | 启用认证长度字段 |
| `client-fingerprint` | string | 否 | TLS 客户端指纹（如 chrome） |

## 加密选项

| Cipher | 描述 |
|--------|------|
| `auto` | 自动选择（默认） |
| `aes-128-gcm` | AES-128-GCM |
| `chacha20-poly1305` | ChaCha20-Poly1305 |
| `none` | 无加密（不推荐） |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | VMess 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 通过 XUDP/gRPC |

## 相关文档

- [[vless]] - VLESS 协议
- [[../../protocol/vmess]] - VMess 协议规范
- [[../../crypto/aead]] - AEAD 加密