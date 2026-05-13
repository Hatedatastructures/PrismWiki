---
title: Prism 配置详解
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [configuration, proxy-server, cpp]
sources:
  - I:/code/Prism/CLAUDE.md
confidence: high
---

# Prism 配置详解

Prism 使用 JSON 配置文件，支持代理、连接池、协议、多路复用、TLS 伪装、DNS、日志等配置。

## 配置文件位置

- 默认配置文件：`src/configuration.json`
- 查找顺序：
  1. 命令行参数指定的路径
  2. 可执行文件同目录下的 `configuration.json`

## 核心配置项

### agent — 代理服务器配置

```json
{
  "agent": {
    "addressable": "0.0.0.0:8081",
    "positive": null,
    "certificate": {
      "cert": "cert.pem",
      "key": "key.pem"
    },
    "authentication": {
      "users": [
        {
          "uuid": "00000000-0000-0000-0000-000000000000",
          "password": "prism",
          "max_connections": 100
        }
      ]
    },
    "camouflage": "/path/to/camouflage",
    "limit": {
      "blacklist": false
    }
  }
}
```

### pool — 连接池配置

```json
{
  "pool": {
    "max_cache_per_endpoint": 16,
    "max_idle_seconds": 300,
    "connect_timeout_ms": 5000,
    "recv_buffer_size": 65536,
    "send_buffer_size": 65536
  }
}
```

### protocol — 协议配置

```json
{
  "protocol": {
    "socks5": {
      "enable_tcp": true,
      "enable_udp": true,
      "enable_auth": false
    },
    "trojan": {
      "enable_tcp": true,
      "enable_udp": true
    },
    "vless": {
      "enable_udp": true
    },
    "shadowsocks": {
      "psk": "your_pre_shared_key",
      "method": "aes-128-gcm",
      "enable_tcp": true,
      "enable_udp": true
    }
  }
}
```

### multiplex — 多路复用配置

```json
{
  "multiplex": {
    "smux": {
      "max_streams": 256,
      "buffer_size": 65536,
      "keepalive_interval_ms": 30000
    },
    "yamux": {
      "max_streams": 256,
      "buffer_size": 65536,
      "initial_window": 262144,
      "enable_ping": true
    }
  }
}
```

### stealth — TLS 伪装配置

```json
{
  "stealth": {
    "reality": {
      "dest": "www.example.com:443",
      "server_names": ["www.example.com"],
      "private_key": "your_private_key",
      "short_ids": ["your_short_id"]
    },
    "shadowtls": {
      "version": 3,
      "password": "your_password",
      "handshake_dest": "www.example.com:443"
    },
    "restls": {
      "host": "www.example.com",
      "password": "your_password",
      "restls_script": "your_script"
    }
  }
}
```

### dns — DNS 配置

```json
{
  "dns": {
    "servers": [
      {
        "address": "8.8.8.8",
        "port": 53,
        "protocol": "udp",
        "timeout_ms": 5000
      }
    ],
    "mode": "first",
    "cache_enabled": true,
    "cache_size": 1000,
    "cache_ttl": 300,
    "serve_stale": true,
    "disable_ipv6": false
  }
}
```

### trace — 日志配置

```json
{
  "trace": {
    "file_name": "prism.log",
    "path_name": "./logs",
    "max_size": 10485760,
    "max_files": 5,
    "log_level": "info",
    "enable_console": true,
    "enable_file": true
  }
}
```

## 配置工具

- `scripts/GenRealityKeys.sh` / `.ps1` — Reality 密钥对生成

## 相关页面

- [[agent]] — 项目主页
- [[agent/architecture]] — 架构设计
- [[dev/modules]] — 模块结构
- [[Mihomo-Clash-Config]] — 客户端配置对比
- [[mihomo-Meta]] — 客户端
