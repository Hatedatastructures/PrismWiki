---
title: VLESS
created: 2026-05-17
updated: 2026-05-17
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

## YAML 配置示例

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

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | VLESS 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 通过 XUDP/gRPC |

## 相关文档

- [[vmess]] - VMess 协议
- [[ref/protocol/vless-spec|vless-spec]] - VLESS 协议规范
- [[reality]] - Reality TLS 隐蔽
- [[ech]] - ECH 加密 Client Hello