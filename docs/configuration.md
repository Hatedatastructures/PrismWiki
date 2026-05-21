---
title: 配置说明
created: 2026-05-17
updated: 2026-05-21
layer: docs
tags: [docs, guide]
---

# 配置说明

Prism 配置文件结构简览，快速了解各模块功能。所有字段名与默认值均与源码 `configuration.json` 对齐。

## 功能简介

配置文件 `configuration.json` 分为 8 大模块：agent、pool、buffer、protocol、multiplex、stealth、dns、trace。程序启动时自动加载，所有模块均可独立配置。

## 加载方式

- 默认路径: 程序同目录下的 `configuration.json`
- 自定义路径: 启动参数 `--config <path>` 指定
- 配置格式: 标准 JSON，不支持注释，不支持尾部逗号

## 完整配置示例

```json
{
  "agent": {
    "limit": {
      "blacklist": true
    },
    "positive": {
      "host": "",
      "port": 0
    },
    "addressable": {
      "host": "0.0.0.0",
      "port": 8081
    },
    "certificate": {
      "key": "/etc/prism/key.pem",
      "cert": "/etc/prism/cert.pem"
    },
    "authentication": {
      "users": [
        {
          "password": "your-strong-password",
          "uuid": "123e4567-e89b-12d3-a456-426614174000",
          "max_connections": 0
        }
      ]
    },
    "camouflage": "/api/notification/v1/stream?id=abcd-1234",
    "reverse_map": {}
  },
  "pool": {
    "max_cache_per_endpoint": 32,
    "max_idle_seconds": 240,
    "connect_timeout_ms": 300,
    "cleanup_interval_sec": 20,
    "recv_buffer_size": 65536,
    "send_buffer_size": 65536,
    "tcp_nodelay": true,
    "keep_alive": true,
    "cache_ipv6": false
  },
  "buffer": {
    "size": 262144
  },
  "protocol": {
    "socks5": {
      "enable_tcp": true,
      "enable_udp": true,
      "enable_bind": false,
      "udp_bind_port": 0,
      "udp_idle_timeout": 60,
      "udp_max_datagram": 65535,
      "enable_auth": true
    },
    "trojan": {
      "enable_tcp": true,
      "enable_udp": true,
      "udp_idle_timeout": 60,
      "udp_max_datagram": 65535
    },
    "vless": {
      "enable_udp": true,
      "udp_idle_timeout": 60,
      "udp_max_datagram": 65535
    },
    "shadowsocks": {
      "psk": "base64-encoded-key",
      "method": "2022-blake3-aes-128-gcm",
      "enable_tcp": true,
      "enable_udp": false,
      "udp_idle_timeout": 60
    }
  },
  "multiplex": {
    "enabled": true,
    "smux": {
      "max_streams": 512,
      "buffer_size": 65535,
      "keepalive_interval_ms": 30000,
      "udp_idle_timeout_ms": 60000,
      "udp_max_datagram": 65535
    },
    "yamux": {
      "max_streams": 256,
      "buffer_size": 65535,
      "initial_window": 4194304,
      "enable_ping": true,
      "ping_interval_ms": 30000,
      "stream_open_timeout_ms": 30000,
      "stream_close_timeout_ms": 30000,
      "udp_idle_timeout_ms": 60000,
      "udp_max_datagram": 65535
    }
  },
  "stealth": {
    "reality": {
      "dest": "www.microsoft.com:443",
      "server_names": ["www.microsoft.com"],
      "private_key": "base64-encoded-private-key",
      "short_ids": ["0123456789abcdef"]
    },
    "shadowtls": {
      "version": 3,
      "password": "",
      "users": [
        {
          "name": "user1",
          "password": "shadowtls_password"
        }
      ],
      "handshake_dest": "www.apple.com:443",
      "server_names": ["www.apple.com"],
      "strict_mode": true,
      "handshake_timeout_ms": 5000
    },
    "restls": {
      "server_names": ["www.nvidia.com"],
      "host": "www.nvidia.com:443",
      "password": "restls_password",
      "version_hint": "tls13",
      "restls_script": "300?100<1,400~100,350~100,600~100,300~200,300~100",
      "handshake_timeout_ms": 5000
    },
    "anytls": {
      "server_names": ["www.cloudflare.com"],
      "certificate": "/etc/prism/cert.pem",
      "private_key": "/etc/prism/key.pem",
      "users": [
        {
          "username": "user1",
          "password": "anytls_password"
        }
      ],
      "padding_scheme": "300?100<1,400~100,350~100",
      "handshake_timeout_ms": 5000,
      "idle_session_timeout_ms": 30000
    },
    "trusttunnel": {
      "server_names": ["www.amazon.com"],
      "certificate": "/etc/prism/cert.pem",
      "private_key": "/etc/prism/key.pem",
      "users": [
        {
          "username": "user1",
          "password": "trusttunnel_password"
        }
      ],
      "network": "both",
      "congestion": "bbr",
      "handshake_timeout_ms": 5000,
      "idle_timeout_ms": 30000
    }
  },
  "dns": {
    "servers": [
      {
        "address": "8.8.8.8",
        "port": 53,
        "protocol": "udp",
        "timeout_ms": 1500
      }
    ],
    "mode": "first",
    "timeout_ms": 5000,
    "cache_enabled": true,
    "cache_size": 10000,
    "cache_ttl": 120,
    "serve_stale": true,
    "negative_ttl": 300,
    "ttl_min": 60,
    "ttl_max": 86400,
    "disable_ipv6": true
  },
  "trace": {
    "file_name": "forward.log",
    "path_name": "/var/log/prism/",
    "max_size": 67108864,
    "max_files": 8,
    "queue_size": 8192,
    "thread_count": 1,
    "enable_console": true,
    "enable_file": true,
    "log_level": "info",
    "pattern": "[%Y-%m-%d %H:%M:%S.%e] [%5t] [%l] %v",
    "trace_name": "forward_engine"
  }
}
```

## 模块详解

### agent（代理服务）

核心模块，定义监听地址、TLS 证书、认证信息和访问控制。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `addressable.host` | string | `"0.0.0.0"` | 监听 IP 地址，`0.0.0.0` 接受所有连接 |
| `addressable.port` | int | `8081` | 监听端口号 |
| `certificate.cert` | string | — | TLS 证书路径，不配置则使用明文传输 |
| `certificate.key` | string | — | TLS 私钥路径 |
| `authentication.users` | array | `[]` | 认证用户列表，见下方用户对象 |
| `limit.blacklist` | bool | `true` | 是否启用黑名单机制 |
| `positive.host` | string | `""` | 正向代理目标地址（为空则按请求路由） |
| `positive.port` | int | `0` | 正向代理目标端口 |
| `camouflage` | string | — | WebSocket 伪装路径（如 `/api/notification/v1/stream?id=xxx`） |
| `reverse_map` | object | `{}` | 反向代理映射规则 |

**用户对象字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `password` | string | 用户密码（Trojan/SS2022 协议使用） |
| `uuid` | string | 用户 UUID（VLESS 协议使用） |
| `max_connections` | int | 最大并发连接数，`0` 表示不限制 |

### pool（连接池）

管理下游连接的生命周期，提升复用率。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_cache_per_endpoint` | int | `32` | 每个目标地址的最大缓存连接数，越大内存占用越高 |
| `max_idle_seconds` | int | `240` | 空闲连接超时时间（秒），超时后自动关闭 |
| `connect_timeout_ms` | int | `300` | 建立连接的超时时间（毫秒） |
| `cleanup_interval_sec` | int | `20` | 连接池清理间隔（秒），定期扫描并移除过期连接 |
| `recv_buffer_size` | int | `65536` | 接收缓冲区大小（字节） |
| `send_buffer_size` | int | `65536` | 发送缓冲区大小（字节） |
| `tcp_nodelay` | bool | `true` | 是否禁用 Nagle 算法，开启可降低延迟 |
| `keep_alive` | bool | `true` | 是否启用 TCP Keep-Alive |
| `cache_ipv6` | bool | `false` | 是否缓存 IPv6 连接 |

> **注意**: `max_cache_per_endpoint` 只限制单个目标的缓存数量，不限制总 fd 数。高并发场景下新连接不受此上限约束。详见 [[dev/debugging/deep-dive/system-risks\|系统风险分析]]。

### buffer（缓冲区）

控制内存缓冲区的分配策略。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | int | `262144` | 缓冲区大小（字节），默认 256KB |

### protocol（协议）

控制各代理协议的启用状态与行为。每个协议均可独立配置 TCP/UDP 支持和超时参数。

#### socks5

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_tcp` | bool | `true` | 是否启用 TCP 代理 |
| `enable_udp` | bool | `true` | 是否启用 UDP ASSOCIATE |
| `enable_bind` | bool | `false` | 是否启用 BIND 命令 |
| `udp_bind_port` | int | `0` | UDP 绑定端口，`0` 表示自动分配 |
| `udp_idle_timeout` | int | `60` | UDP 关联空闲超时（秒） |
| `udp_max_datagram` | int | `65535` | UDP 最大数据报大小 |
| `enable_auth` | bool | `true` | 是否要求用户名/密码认证 |

> **实现边界**: UDP ASSOCIATE 内部硬编码 `udp::v4()`，IPv6 客户端 UDP 会静默失败。详见 [[core/protocol/socks5/stream\|SOCKS5 实现边界]]。

#### trojan

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_tcp` | bool | `true` | 是否启用 TCP 代理 |
| `enable_udp` | bool | `true` | 是否启用 UDP |
| `udp_idle_timeout` | int | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | int | `65535` | UDP 最大数据报大小 |

#### vless

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_udp` | bool | `true` | 是否启用 UDP |
| `udp_idle_timeout` | int | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | int | `65535` | UDP 最大数据报大小 |

> **实现边界**: 不支持 XTLS/Vision flow（`addnl_info != 0` 直接拒绝）。详见 [[core/protocol/vless/format\|VLESS 实现边界]]。

#### shadowsocks

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `psk` | string | — | **必填**，Base64 编码的预共享密钥 |
| `method` | string | — | **必填**，加密方法（如 `2022-blake3-aes-128-gcm`） |
| `enable_tcp` | bool | `true` | 是否启用 TCP 代理 |
| `enable_udp` | bool | `false` | 是否启用 UDP |
| `udp_idle_timeout` | int | `60` | UDP 空闲超时（秒） |

> **实现边界**: `timestamp_window` 硬编码 30 秒，客户端与服务端时钟偏差超过此值会被拒绝。详见 [[core/protocol/shadowsocks/relay\|SS2022 实现边界]]。

### multiplex（多路复用）

配置底层传输的多路复用策略。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | `false` | 全局多路复用开关 |

#### smux

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_streams` | int | `512` | 每个连接的最大并发流数 |
| `buffer_size` | int | `65535` | 读写缓冲区大小（字节） |
| `keepalive_interval_ms` | int | `30000` | Keep-alive 心跳间隔（毫秒） |
| `udp_idle_timeout_ms` | int | `60000` | UDP 流空闲超时（毫秒） |
| `udp_max_datagram` | int | `65535` | UDP 最大数据报大小 |

#### yamux

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_streams` | int | `256` | 每个连接的最大并发流数 |
| `buffer_size` | int | `65535` | 读写缓冲区大小（字节） |
| `initial_window` | int | `4194304` | 初始流控窗口大小（字节），默认 4MB |
| `enable_ping` | bool | `true` | 是否启用 Ping/Pong 保活 |
| `ping_interval_ms` | int | `30000` | Ping 间隔（毫秒） |
| `stream_open_timeout_ms` | int | `30000` | 流打开超时（毫秒） |
| `stream_close_timeout_ms` | int | `30000` | 流关闭超时（毫秒） |
| `udp_idle_timeout_ms` | int | `60000` | UDP 流空闲超时（毫秒） |
| `udp_max_datagram` | int | `65535` | UDP 最大数据报大小 |

### stealth（伪装/反检测）

配置流量伪装模块，对抗深度包检测（DPI）。支持 5 种方案，可同时配置多种但运行时只会命中一种（由识别器决定）。

#### reality

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `dest` | string | — | **必填**，伪装目标地址（格式 `host:port`） |
| `server_names` | array | — | **必填**，允许的 SNI 域名列表 |
| `private_key` | string | — | **必填**，Base64 编码的 Reality 私钥 |
| `short_ids` | array | — | **必填**，允许的 short_id 列表（十六进制字符串） |

#### shadowtls

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `version` | int | `3` | ShadowTLS 协议版本，当前仅支持 v3 |
| `password` | string | `""` | 全局密码（`users` 为空时使用） |
| `users` | array | `[]` | 用户列表，每项含 `name` 和 `password` |
| `handshake_dest` | string | — | **必填**，TLS 握手代理目标地址（格式 `host:port`） |
| `server_names` | array | — | 允许的 SNI 域名列表 |
| `strict_mode` | bool | `true` | 严格模式，拒绝不符合预期的客户端 |
| `handshake_timeout_ms` | int | `5000` | 握手超时（毫秒） |

> **注意**: ShadowTLS 传输阶段写入方向发送明文 payload（不加密），且 HMAC 验证累积计算会随时间增加 CPU 开销。详见 [[dev/debugging/deep-dive/shadowtls-limitations\|ShadowTLS 限制]]。

#### restls

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `server_names` | array | — | 允许的 SNI 域名列表 |
| `host` | string | — | 伪装目标地址（格式 `host:port`） |
| `password` | string | — | 认证密码 |
| `version_hint` | string | `"tls13"` | TLS 版本提示 |
| `restls_script` | string | — | 填充脚本，控制伪装流量的长度分布 |
| `handshake_timeout_ms` | int | `5000` | 握手超时（毫秒） |

#### anytls

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `server_names` | array | — | 允许的 SNI 域名列表 |
| `certificate` | string | — | TLS 证书路径 |
| `private_key` | string | — | TLS 私钥路径 |
| `users` | array | `[]` | 用户列表，每项含 `username` 和 `password` |
| `padding_scheme` | string | — | 填充策略脚本 |
| `handshake_timeout_ms` | int | `5000` | 握手超时（毫秒） |
| `idle_session_timeout_ms` | int | `30000` | 空闲会话超时（毫秒） |

#### trusttunnel

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `server_names` | array | — | 允许的 SNI 域名列表 |
| `certificate` | string | — | TLS 证书路径 |
| `private_key` | string | — | TLS 私钥路径 |
| `users` | array | `[]` | 用户列表，每项含 `username` 和 `password` |
| `network` | string | `"both"` | 网络模式：`"tcp"`、`"udp"` 或 `"both"` |
| `congestion` | string | `"bbr"` | 拥塞控制算法 |
| `handshake_timeout_ms` | int | `5000` | 握手超时（毫秒） |
| `idle_timeout_ms` | int | `30000` | 空闲超时（毫秒） |

### dns（域名解析）

配置 DNS 解析策略。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `servers` | array | `[]` | DNS 服务器列表，每项为对象（见下方服务器对象） |
| `mode` | string | `"first"` | 解析模式：`"first"`（使用第一个成功结果）或 `"round_robin"`（轮询） |
| `timeout_ms` | int | `5000` | 整体 DNS 查询超时（毫秒） |
| `cache_enabled` | bool | `true` | 是否启用 DNS 缓存 |
| `cache_size` | int | `10000` | 缓存条目上限 |
| `cache_ttl` | int | `120` | 正向缓存 TTL（秒） |
| `serve_stale` | bool | `true` | 缓存过期后是否返回过期结果（同时后台刷新） |
| `negative_ttl` | int | `300` | 负缓存 TTL（秒），查询失败时缓存负面结果 5 分钟 |
| `ttl_min` | int | `60` | 最小 TTL 限制（秒） |
| `ttl_max` | int | `86400` | 最大 TTL 限制（秒） |
| `disable_ipv6` | bool | `true` | 是否禁用 AAAA 记录查询 |

**DNS 服务器对象：**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `address` | string | — | DNS 服务器 IP 地址 |
| `port` | int | `53` | DNS 服务器端口 |
| `protocol` | string | `"udp"` | 查询协议：`"udp"` 或 `"tcp"` |
| `timeout_ms` | int | `1500` | 单次查询超时（毫秒） |

> **注意**: `negative_ttl` 默认 300 秒意味着临时 DNS 故障会被放大为 5 分钟不可用。可通过降低此值缓解。详见 [[dev/debugging/deep-dive/system-risks\|系统风险分析]]。

### trace（追踪/日志）

配置日志输出行为。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `file_name` | string | `"forward.log"` | 日志文件名 |
| `path_name` | string | — | 日志文件输出目录 |
| `max_size` | int | `67108864` | 单个日志文件最大大小（字节），默认 64MB，超过后自动轮转 |
| `max_files` | int | `8` | 保留的历史日志文件数 |
| `queue_size` | int | `8192` | 异步日志队列大小，队列满时阻塞写入线程 |
| `thread_count` | int | `1` | 后台日志写入线程数 |
| `enable_console` | bool | `true` | 是否同时输出到控制台 |
| `enable_file` | bool | `true` | 是否输出到文件 |
| `log_level` | string | `"info"` | 日志级别：`"trace"`/`"debug"`/`"info"`/`"warn"`/`"error"` |
| `pattern` | string | `"[%Y-%m-%d %H:%M:%S.%e] [%5t] [%l] %v"` | 日志格式模板 |
| `trace_name` | string | `"forward_engine"` | 日志记录器名称 |

## 配置验证与热更新

### 配置验证

Prism 启动时会验证 JSON 格式和必填字段。如果配置无效，程序会输出错误信息并退出：

```json
// 错误示例：port 不是数字
{ "agent": { "addressable": { "host": "0.0.0.0", "port": "8081" } } }
// 错误信息: "agent.addressable.port must be an integer"
```

建议在修改配置后使用 `--config` 参数进行一次验证启动，确认无误后再重启正式服务。

### 热更新

当前版本**不支持**配置热更新。修改配置后需要：

1. 停止服务（`systemctl stop prism` 或 `sc stop Prism`）
2. 修改配置文件
3. 重新启动服务

> **提示**: 可以先用 `diff` 对比新旧配置，避免遗漏关键字段。

## 相关页面

- [[dev/configuration|配置详解]]
- [[core/agent/worker/worker|Worker 配置]]
- [[dev/testing|测试配置]]
- [[docs/getting-started|快速开始]]
- [[docs/deployment|部署指南]]
- [[dev/debugging/deep-dive/system-risks|系统风险分析]]
