---
title: 从 Xray 迁移到 Prism
created: 2026-05-23
updated: 2026-05-23
layer: docs
tags: [docs, migration, xray, guide]
---

# 从 Xray 迁移到 Prism

Xray-core 到 Prism 的迁移指南，包含配置映射、协议兼容性和迁移步骤。

## 概述

Prism 与 Xray-core 支持的协议高度重合（VLESS、Trojan、Shadowsocks），但配置格式和架构有显著差异。本文帮助 Xray 用户平滑迁移到 Prism。

### 主要差异

| 维度 | Xray-core | Prism |
|------|-----------|-------|
| 配置格式 | JSON（V2Ray 风格） | JSON（扁平化） |
| 编程语言 | Go | C++23 |
| 异步模型 | Goroutine | 协程（Boost.Asio） |
| 内存管理 | GC | PMR 手动管理 |
| 多协议共端口 | fallbacks | SNI 路由 + 三阶段识别 |
| 多路复用 | S-Mux（内置） | Smux/Yamux（sing-mux 协商） |
| TLS 伪装 | Reality（内置） | Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel |
| 协议检测 | mannual (配置指定) | 自动识别（probe + identify） |
| 证书管理 | 自行配置 | 自行配置 / Reality 免证书 |
| 运行方式 | 单进程 | 单进程（多 worker 线程） |

## 协议兼容性

### 完全兼容

以下协议的客户端配置无需修改，Prism 直接兼容：

| 协议 | Xray 出站 | Prism 接入 |
|------|-----------|-----------|
| VLESS | `vless` outbound | 自动识别 |
| Trojan | `trojan` outbound | 自动识别 |
| VLESS + Reality | `vless` + realitySettings | 自动识别 |
| VLESS + XTLS | `vless` + flow | 不支持 flow（使用标准 TLS） |
| Trojan + TLS | `trojan` + tlsSettings | 自动识别 |
| Shadowsocks | `shadowsocks` outbound | 自动识别（SS2022） |

### 部分兼容

| 特性 | Xray 支持 | Prism 支持 | 差异说明 |
|------|-----------|-----------|----------|
| XTLS Vision | 是 | 否 | Prism 使用标准 TLS，不依赖 XTLS |
| gRPC 传输 | 是 | 否 | Prism 不支持 gRPC 传输 |
| WebSocket | 是 | 计划中 | WebSocket 传输在路线图中 |
| HTTP/2 传输 | 是 | 否 | Prism 不支持 h2 传输 |
| Shadowsocks 旧版 | 是 | 否 | 仅支持 SS2022（SIP022） |
| VMess | 是 | 否 | Prism 不支持 VMess |

### Prism 独有

以下协议/功能是 Prism 支持但 Xray 不支持的：

| 特性 | 说明 |
|------|------|
| ShadowTLS v3 | TLS 伪装方案，HMAC-SHA1 认证 |
| Restls | TLS 流量伪装，Script 控制 |
| AnyTLS | 标准 TLS 证书 + 应用层认证 |
| TrustTunnel | TCP/QUIC 双栈，BBR 拥塞控制 |
| 自动协议识别 | 单端口多协议自动识别 |
| 分层检测管道 | Tier 0/1/2 三层检测 |
| SNI 路由表 | 根据 SNI 自动路由到伪装方案 |
| ECH 叠加 | 加密 ClientHello SNI |

## 配置映射表

### 基础结构映射

| Xray 配置项 | Prism 配置项 | 说明 |
|-------------|-------------|------|
| `inbounds[0].port` | `agent.addressable.port` | 监听端口 |
| `inbounds[0].listen` | `agent.addressable.host` | 监听地址 |
| `inbounds[0].protocol` | 自动识别 | Prism 自动检测协议 |
| `inbounds[0].settings.clients` | `agent.authentication.users` | 用户认证 |
| `inbounds[0].streamSettings.network` | 不适用 | Prism 使用标准 TCP |
| `inbounds[0].streamSettings.security` | `stealth.*` | TLS/伪装方案配置 |
| `outbounds[0]` | `agent.positive` | 上游代理（可选） |
| `log.loglevel` | `trace.log_level` | 日志级别 |
| `routing` | 不适用 | Prism 是代理服务，不做路由 |

### VLESS 配置映射

**Xray 配置**:

```json
{
  "inbounds": [{
    "protocol": "vless",
    "port": 443,
    "settings": {
      "clients": [
        { "id": "uuid-here", "flow": "" }
      ],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "tls",
      "tlsSettings": {
        "certificates": [{
          "certificateFile": "/path/to/cert.pem",
          "keyFile": "/path/to/key.pem"
        }]
      }
    }
  }]
}
```

**Prism 等效配置**:

```json
{
  "agent": {
    "addressable": { "host": "0.0.0.0", "port": 443 },
    "certificate": { "key": "/path/to/key.pem", "cert": "/path/to/cert.pem" },
    "authentication": {
      "users": [
        { "uuid": "uuid-here", "max_connections": 0 }
      ]
    }
  },
  "protocol": {
    "vless": { "enable_udp": true }
  }
}
```

**关键差异**:
- Xray 需要指定 `protocol: "vless"`，Prism 通过协议识别自动检测
- Xray 使用 `clients` 数组，Prism 使用统一的 `authentication.users`
- Prism 的 VLESS 不支持 `flow`（XTLS Vision 等）

### VLESS + Reality 配置映射

**Xray 配置**:

```json
{
  "inbounds": [{
    "protocol": "vless",
    "port": 443,
    "settings": {
      "clients": [
        { "id": "uuid-here", "flow": "xtls-rprx-vision" }
      ],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp",
      "security": "reality",
      "realitySettings": {
        "dest": "www.microsoft.com:443",
        "serverNames": ["www.microsoft.com"],
        "privateKey": "base64-private-key",
        "shortIds": ["hex-short-id"]
      }
    }
  }]
}
```

**Prism 等效配置**:

```json
{
  "agent": {
    "addressable": { "host": "0.0.0.0", "port": 443 },
    "authentication": {
      "users": [
        { "uuid": "uuid-here", "max_connections": 0 }
      ]
    }
  },
  "protocol": {
    "vless": { "enable_udp": true }
  },
  "stealth": {
    "reality": {
      "dest": "www.microsoft.com:443",
      "server_names": ["www.microsoft.com"],
      "private_key": "base64-private-key",
      "short_ids": ["hex-short-id"]
    }
  }
}
```

**关键差异**:
- Xray 使用 `realitySettings`，Prism 使用 `stealth.reality`
- Xray 字段名 `privateKey`，Prism 使用 `private_key`（snake_case）
- Xray 字段名 `serverNames`，Prism 使用 `server_names`
- Prism 不需要 `flow: "xtls-rprx-vision"`，使用标准 TLS
- Prism 的 Reality 密钥对使用 `GenRealityKeys.sh` 生成，格式与 Xray 兼容

### Trojan 配置映射

**Xray 配置**:

```json
{
  "inbounds": [{
    "protocol": "trojan",
    "port": 443,
    "settings": {
      "clients": [
        { "password": "trojan-password" }
      ]
    },
    "streamSettings": {
      "network": "tcp",
      "security": "tls",
      "tlsSettings": {
        "certificates": [{
          "certificateFile": "/path/to/cert.pem",
          "keyFile": "/path/to/key.pem"
        }]
      }
    }
  }]
}
```

**Prism 等效配置**:

```json
{
  "agent": {
    "addressable": { "host": "0.0.0.0", "port": 443 },
    "certificate": { "key": "/path/to/key.pem", "cert": "/path/to/cert.pem" },
    "authentication": {
      "users": [
        { "password": "trojan-password", "max_connections": 0 }
      ]
    }
  },
  "protocol": {
    "trojan": { "enable_tcp": true, "enable_udp": true }
  }
}
```

### Shadowsocks 配置映射

**Xray 配置**:

```json
{
  "inbounds": [{
    "protocol": "shadowsocks",
    "port": 443,
    "settings": {
      "method": "2022-blake3-aes-128-gcm",
      "password": "base64-psk",
      "network": "tcp,udp"
    }
  }]
}
```

**Prism 等效配置**:

```json
{
  "agent": {
    "addressable": { "host": "0.0.0.0", "port": 443 }
  },
  "protocol": {
    "shadowsocks": {
      "psk": "base64-psk",
      "method": "2022-blake3-aes-128-gcm",
      "enable_tcp": true,
      "enable_udp": false
    }
  }
}
```

**关键差异**:
- Prism 仅支持 SS2022（SIP022），不支持旧版 Shadowsocks
- Xray 使用 `password`，Prism 使用 `psk`
- Xray 使用 `network: "tcp,udp"`，Prism 使用 `enable_tcp` + `enable_udp` 布尔开关

### 多路复用配置映射

**Xray 配置** (S-Mux):

```json
{
  "inbounds": [{
    "protocol": "vless",
    "settings": { "clients": [...] },
    "streamSettings": {
      "network": "tcp",
      "security": "tls"
    },
    "mux": {
      "enabled": true,
      "concurrency": 8
    }
  }]
}
```

**Prism 等效配置** (Smux/Yamux):

```json
{
  "multiplex": {
    "enabled": true,
    "smux": {
      "max_streams": 32,
      "buffer_size": 65535,
      "keepalive_interval_ms": 30000
    }
  }
}
```

**关键差异**:
- Xray 使用内置 S-Mux，Prism 使用 sing-mux 协商 + Smux/Yamux
- Xray 的 `mux.concurrency` 控制并发连接数，Prism 使用 `max_streams` 控制单会话流数
- Prism 的客户端需要配置 `smux` 或 `yamux` 协议（通过 Mihomo/sing-box 客户端）
- Prism 支持 Smux 和 Yamux 两种多路复用协议

### Fallbacks 映射

**Xray 配置** (fallbacks):

```json
{
  "inbounds": [{
    "protocol": "vless",
    "settings": {
      "clients": [...],
      "fallbacks": [
        { "dest": 8080, "path": "/ws", "xver": 1 }
      ]
    }
  }]
}
```

**Prism 等效**: Prism 使用自动协议识别 + SNI 路由替代 fallbacks。不需要显式配置 fallback，识别失败时自动回退到真实 TLS 服务或 Native TLS。

## 迁移步骤

### 第 1 步: 评估兼容性

1. 列出当前 Xray 使用的协议和传输方式
2. 对照协议兼容性表确认 Prism 支持情况
3. 如果使用了 gRPC/WebSocket/XTLS 等 Prism 暂不支持的功能，需要调整方案

### 第 2 步: 准备 Prism 配置

1. 将 Xray 配置映射为 Prism 格式（参考上方映射表）
2. 将 `clients` 列表转换为 `authentication.users`
3. 将 `realitySettings` 转换为 `stealth.reality`
4. 添加 `pool`、`buffer`、`dns`、`trace` 等新配置项（使用默认值即可）

### 第 3 步: 安装 Prism

```bash
# 从 GitHub Releases 下载
# Windows
curl -Lo prism.zip https://github.com/Hatedatastructures/Prism/releases/latest/download/prism-windows-x64.zip

# Linux
curl -Lo prism.tar.gz https://github.com/Hatedatastructures/Prism/releases/latest/download/prism-linux-x64.tar.gz
tar -xzf prism.tar.gz
chmod +x Prism
```

### 第 4 步: 配置文件

将转换后的配置文件保存为 `configuration.json`，放置在 Prism 可执行文件同目录下。

### 第 5 步: 测试运行

```bash
# 启动 Prism
./Prism

# 在另一个终端测试连接
# 使用 Clash Verge 或 curl 测试各协议
curl -x socks5://127.0.0.1:7890 https://api.ipify.org
```

### 第 6 步: 客户端更新

客户端配置可能需要微调：
- Reality 节点配置基本不变（VLESS + Reality 客户端兼容）
- 去掉 `flow: "xtls-rprx-vision"`（Prism 不支持）
- 确认 SNI 与服务端 `server_names` 一致

## 常见迁移问题

**Q: Xray 的 XTLS Vision 在 Prism 中能用吗？**
A: 不能。Prism 使用标准 TLS，不依赖 XTLS。建议使用 Reality + VLESS 替代，安全性相当。

**Q: gRPC/WebSocket 传输能用吗？**
A: 目前不能。Prism 仅支持标准 TCP 传输。WebSocket 传输在路线图中。

**Q: VMess 能用吗？**
A: 不能。Prism 不支持 VMess。建议迁移到 VLESS 或 Trojan。

**Q: 多协议共端口怎么配置？**
A: Prism 自动识别协议，不需要像 Xray 那样配置 fallbacks。只需在同一端口启用多个协议即可。通过 SNI 路由区分不同的伪装方案。

**Q: 迁移后客户端需要改什么？**
A: 大多数情况下客户端配置无需修改。VLESS + Reality、Trojan + TLS 的客户端配置与 Xray 完全兼容。只需去掉不支持的参数（如 `flow`）。

**Q: Prism 的性能和 Xray 比如何？**
A: Prism 使用 C++23 + 纯协程架构 + PMR 内存管理，在高并发场景下内存占用更低、延迟更稳定。具体性能数据参见 [[dev/testing/benchmark|基准测试]]。

## 相关页面

- [[docs/getting-started|快速开始]]
- [[docs/configuration|配置说明]]
- [[docs/protocol-matrix|协议能力矩阵]]
- [[docs/client-setup|客户端配置]]
- [[docs/client-setup/clash-verge|Clash Verge 配置]]
- [[docs/upgrade|升级指南]]
