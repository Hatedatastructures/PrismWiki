---
title: "mihomo DNS listen 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, listen, 监听地址]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS listen 配置

`listen` 定义 DNS 服务器的监听地址。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/server.go` | DNS 服务器实现 |

## Server 结构

```go
type Server struct {
    service   resolver.Service
    tcpServer *D.Server
    udpServer *D.Server
}

func ReCreateServer(addr string, service resolver.Service) {
    // UDP DNS
    p, err := inbound.ListenPacket("udp", addr)
    server.udpServer = &D.Server{Addr: addr, PacketConn: p, Handler: server}
    
    // TCP DNS
    l, err := inbound.Listen("tcp", addr)
    server.tcpServer = &D.Server{Addr: addr, Listener: l, Handler: server}
}
```

## YAML 配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053  # 监听所有接口的 1053 端口
```

```yaml
dns:
  listen: 127.0.0.1:53  # 仅监听本地回环的 53 端口
```

## 配置说明

| 格式 | 说明 |
|------|------|
| `0.0.0.0:port` | 监听所有网络接口 |
| `127.0.0.1:port` | 仅监听本地回环 |
| `192.168.1.1:port` | 监听特定 IP |

## 默认值

- 不设置 `listen` 时，DNS 模块不会对外提供服务
- 常用端口：`53`（标准 DNS）、`1053`（非标准）

## 端口选择建议

| 端口 | 说明 |
|------|------|
| `53` | 标准 DNS 端口，需注意端口冲突 |
| `1053` | 非标准端口，避免冲突 |
| `5353` | mDNS 端口，不推荐 |

## listen-tcp 配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  listen-tcp: true  # 同时监听 TCP
```

| 值 | 说明 |
|---|------|
| `true` | 同时监听 TCP 和 UDP |
| `false` | 仅监听 UDP |

## 配置示例

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  listen-tcp: true
  enhanced-mode: fake-ip
  nameserver:
    - https://dns.alidns.com/dns-query
```

## 使用场景

### 本地使用

```yaml
dns:
  listen: 127.0.0.1:1053
```

仅本机应用使用。

### 局域网使用

```yaml
dns:
  listen: 0.0.0.0:53
```

局域网设备可使用（需权限）。

## 相关文档

- [[overview]] - DNS 配置概述
- [[enable]] - 启用 DNS
- [[servers]] - DNS 服务器配置