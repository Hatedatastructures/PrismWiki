---
title: TrustTunnel
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags:
  - stealth
  - tls
  - quic
  - h3
  - tier2
related:
  - "[[stealth/anytls]]"
  - "[[stealth/restls]]"
  - "[[stealth/shadowtls]]"
  - "[[stealth/ech]]"
---

# TrustTunnel

TrustTunnel 是 Prism stealth 模块中的 **Tier 2** 伪装方案，支持 TCP（HTTP/2）和 UDP
（HTTP/3/QUIC）两种传输模式，使用标准 TLS 证书。与其他 Tier 2 方案类似，无 ClientHello
独占特征，依赖 SNI 匹配。

协议参考: https://github.com/trusttunnel/trusttunnel

## 工作原理

TrustTunnel 的工作流程：

1. 执行标准 TLS 握手（使用配置的证书）
2. 读取 TLS 应用数据（客户端首帧）
3. 验证用户身份
4. 根据网络配置选择传输模式（TCP/UDP）
5. 认证成功后检测内层协议

### 信任链模型

TrustTunnel 基于**用户认证**的信任链：

```
客户端 ──(TLS 握手)──> 服务端
         ──(用户认证)──> 验证身份
         ──(内层协议)──> 代理流量
```

与 ShadowTLS（依赖后端服务器证书链）不同，TrustTunnel 使用自有证书，
信任基础是预共享的用户凭证。

## 传输模式

TrustTunnel 支持三种网络模式：

| 模式 | 协议 | 说明 |
|------|------|------|
| `tcp` | HTTP/2 | 仅 TCP 传输 |
| `udp` | HTTP/3 (QUIC) | 仅 UDP 传输 |
| `both` | HTTP/2 + HTTP/3 | 同时支持（默认） |

QUIC 模式支持配置拥塞控制算法：

| 算法 | 说明 |
|------|------|
| `cubic` | 传统 CUBIC 算法 |
| `bbr` | Google BBR（默认） |
| `new_reno` | New Reno 算法 |

## 配置项

```json
{
  "server_names": ["tunnel.example.com"],
  "certificate": "/path/to/cert.pem",
  "private_key": "/path/to/key.pem",
  "users": [
    { "username": "alice", "password": "secret123" }
  ],
  "network": "both",
  "congestion": "bbr",
  "handshake_timeout_ms": 5000,
  "idle_timeout_ms": 30000
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `server_names` | array | 是 | SNI 白名单 |
| `certificate` | string | 是 | TLS 证书文件路径（PEM） |
| `private_key` | string | 是 | TLS 私钥文件路径（PEM） |
| `users` | array | 是 | 用户列表（username + password） |
| `network` | enum | 否 | `tcp` / `udp` / `both`（默认 `both`） |
| `congestion` | enum | 否 | `cubic` / `bbr` / `new_reno`（默认 `bbr`） |
| `handshake_timeout_ms` | uint32 | 否 | 握手超时（默认 5000ms） |
| `idle_timeout_ms` | uint32 | 否 | 空闲超时（默认 30000ms） |

## Detection

TrustTunnel 在检测流水线中的位置：

- **Tier 0 sniff**：无
- **Tier 1 verify**：无
- **Tier 2 guess**：返回基础分 100，依赖 SNI 匹配

与 AnyTLS 和 Restls 一样，TrustTunnel 完全依赖 SNI 路由阶段过滤。

## 实现状态

当前实现状态：**基础框架已实现，认证逻辑待完善**。

已完成：
- 方案注册和生命周期管理
- SNI 白名单匹配
- 网络类型/拥塞控制枚举（glaze 序列化）
- Tier 2 guess 接口

待实现：
- 标准 TLS 握手执行
- 用户身份验证
- TCP/UDP 传输模式切换
- 内层协议检测

## 相关源码

- `include/prism/stealth/trusttunnel/scheme.hpp` — 方案类定义
- `include/prism/stealth/trusttunnel/config.hpp` — 配置结构（含网络类型和拥塞控制枚举）
- `src/prism/stealth/trusttunnel/scheme.cpp` — 方案实现
