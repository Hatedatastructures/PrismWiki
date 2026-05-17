---
title: 配置文件完整参考
source:
  - I:/code/Prism/src/configuration.json
  - I:/code/Prism/include/prism/config.hpp
  - I:/code/Prism/include/prism/agent/config.hpp
  - I:/code/Prism/include/prism/channel/connection/pool.hpp
  - I:/code/Prism/include/prism/multiplex/config.hpp
  - I:/code/Prism/include/prism/resolve/dns/config.hpp
  - I:/code/Prism/include/prism/trace/config.hpp
module: config
type: reference
tags: [configuration, json, agent, pool, protocol, multiplex, stealth, dns, trace, glaze]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[modules/agent]]"
  - "[[modules/channel]]"
  - "[[modules/multiplex]]"
  - "[[modules/stealth]]"
  - "[[modules/resolve]]"
---

# 配置文件完整参考

Prism 使用 JSON 配置文件，由 `glaze` 库反序列化为 C++ 强类型结构体。配置覆盖代理服务、连接池、协议、多路复用、伪装、DNS、日志等所有模块。

## 概述

### 配置架构

Prism 配置采用分层结构：

```
+--------------------------------------------------+
|                  configuration.json              |
+--------------------------------------------------+
                         │
       +-----------------+-----------------+
       │                 │                 │
+----------+       +----------+       +----------+
|  agent   |       |   pool   |       |  buffer  |
|代理服务  |       | 连接池   |       | 缓冲区   |
+----------+       +----------+       +----------+
       │                 │                 │
       │                 │                 │
+----------+       +----------+       +----------+
| protocol |       | multiplex|       |  stealth |
|协议配置  |       | 多路复用 |       | 伪装配置 |
+----------+       +----------+       +----------+
       │                 │                 │
       │                 │                 │
+----------+       +----------+       +----------+
|   dns    |       |  trace   |       |   ...    |
|DNS配置   |       | 日志配置 |       |  扩展    |
+----------+       +----------+       +----------+
```

每个模块有独立配置类型，通过 `glaze` 自动映射。

### 配置文件查找顺序

Prism 按以下顺序查找配置文件：

```
1. 命令行参数指定路径
   │   prism --config /path/to/config.json
   │   最高优先级
   ▼
2. 可执行文件同目录下的 configuration.json
   │   默认方式
   │   build_release/src/Prism.exe → configuration.json
   ▼
未找到 → 启动失败
```

### 配置加载流程

```cpp
// 配置加载流程（main.cpp）
int main(int argc, char** argv) {
    // 1. 解析命令行参数
    auto config_path = parse_config_path(argc, argv);
    
    // 2. 加载配置文件
    auto config = psm::loader::load(config_path);
    
    // 3. 验证配置有效性
    validate_config(config);
    
    // 4. 初始化各模块
    psm::trace::init(config.trace);
    psm::agent::directory::build(config.agent.authentication);
    // ...
}
```

### JSON 与 C++ 类型映射

使用 `glaze` 库实现自动映射：

```cpp
// C++ 配置结构
struct config {
    psm::agent::config agent;
    psm::channel::config pool;
    psm::agent::buffer buffer;
    psm::protocol::config protocol;
    psm::multiplex::config multiplex;
    psm::agent::stealth::config stealth;
    psm::resolve::dns::config dns;
    psm::trace::config trace;
};

// JSON 配置
{
    "agent": {...},
    "pool": {...},
    "buffer": {...},
    "protocol": {...},
    "multiplex": {...},
    "stealth": {...},
    "dns": {...},
    "trace": {...}
}

// 自动映射（glaze）
auto result = glz::read_json<config>(json_string);
```

---

## 详解

### agent — 代理服务核心配置

对应类型：`psm::agent::config`（`include/prism/agent/config.hpp`）

#### 基础参数

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `addressable.host` | string | `"0.0.0.0"` | 监听地址（0.0.0.0=所有接口） |
| `addressable.port` | uint16 | `8081` | 监听端口 |
| `limit.blacklist` | bool | `true` | 是否启用 IP 黑名单过滤 |

#### TLS 证书配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `certificate.key` | string | — | TLS 私钥文件路径（PEM 格式） |
| `certificate.cert` | string | — | TLS 证书文件路径（PEM 格式，含完整链） |

证书配置示例：

```json
{
    "agent": {
        "certificate": {
            "key": "/etc/letsencrypt/live/example.com/privkey.pem",
            "cert": "/etc/letsencrypt/live/example.com/fullchain.pem"
        }
    }
}
```

#### 用户认证配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `authentication.users[]` | array | — | 统一用户列表 |
| `authentication.users[].password` | string | — | 密码（Trojan/HTTP/SOCKS5） |
| `authentication.users[].uuid` | string | — | UUID（VLESS） |
| `authentication.users[].max_connections` | uint32 | `0` | 最大并发连接（0=无限） |

用户配置示例：

```json
{
    "agent": {
        "authentication": {
            "users": [
                {
                    "password": "your_password_here",
                    "uuid": "550e8400-e29b-41d4-a716-446655440000",
                    "max_connections": 100
                }
            ]
        }
    }
}
```

密码处理：
- Trojan/HTTP/SOCKS5 使用 SHA224 哈希存储
- 启动时自动转换密码为哈希
- VLESS 使用原始 UUID

#### 正向代理配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `positive.host` | string | `""` | 正向代理目标主机（空=不使用） |
| `positive.port` | uint16 | `0` | 正向代理目标端口 |

正向代理示例：

```json
{
    "agent": {
        "positive": {
            "host": "upstream-proxy.example.com",
            "port": 1080
        }
    }
}
```

#### 伪装与回落

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `camouflage` | string | `""` | 伪装路径（如 `/api/v1/stream`） |
| `reverse_map` | object | `{}` | 反向代理路由表 |

回落配置示例：

```json
{
    "agent": {
        "camouflage": "/api/notification/v1/stream",
        "reverse_map": {
            "/": "127.0.0.1:8080",
            "/api": "127.0.0.1:3000"
        }
    }
}
```

---

### pool — 连接池配置

对应类型：`psm::channel::config`（`include/prism/channel/connection/pool.hpp`）

#### 缓存配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_cache_per_endpoint` | uint32 | `32` | 每端点最大缓存连接数 |
| `max_idle_seconds` | uint64 | `30` | 穯闲连接最大存活时间（秒） |
| `cleanup_interval_sec` | uint64 | `10` | 后台清理间隔（秒） |

#### 连接配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `connect_timeout_ms` | uint64 | `300` | 新建连接超时（毫秒） |
| `recv_buffer_size` | uint32 | `65536` | 接收缓冲区大小（字节） |
| `send_buffer_size` | uint32 | `65536` | 发送缓冲区大小（字节） |

#### TCP 选项

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `tcp_nodelay` | bool | `true` | 启用 TCP_NODELAY（禁用 Nagle） |
| `keep_alive` | bool | `true` | 启用 SO_KEEPALIVE |
| `cache_ipv6` | bool | `false` | 是否缓存 IPv6 连接 |

连接池配置示例：

```json
{
    "pool": {
        "max_cache_per_endpoint": 64,
        "max_idle_seconds": 60,
        "connect_timeout_ms": 500,
        "recv_buffer_size": 131072,
        "send_buffer_size": 131072,
        "tcp_nodelay": true,
        "keep_alive": true,
        "cache_ipv6": false
    }
}
```

连接池工作机制：

```
+------------------------+
|     连接池架构          |
+------------------------+
          │
    +-----+-----+
    │           │
+--------+  +--------+
|LIFO栈 |  |僵尸检测|
|缓存   |  |清理    |
+--------+  +--------+
    │           │
    │           │
最新连接    超时清理
优先复用    max_idle_seconds
```

---

### buffer — 缓冲区配置

对应类型：`psm::agent::buffer`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `size` | uint32 | `262144` (256KB) | 数据传输缓冲区大小 |

缓冲区使用场景：
- 双向隧道转发
- 大文件传输
- 高延迟高带宽链路

调优建议：

| 网络环境 | 建议 size | 原因 |
|----------|-----------|------|
| 低延迟低带宽 | 64KB-128KB | 减少内存占用 |
| 高延迟高带宽 | 256KB-512KB | 提高吞吐量 |
| 高延迟低带宽 | 64KB | 避免过度缓冲 |

---

### protocol — 协议配置

对应类型：`psm::protocol::config`

#### socks5 子配置

对应类型：`psm::protocol::socks5::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_tcp` | bool | `true` | 允许 CONNECT 命令 |
| `enable_udp` | bool | `true` | 允许 UDP ASSOCIATE 命令 |
| `enable_bind` | bool | `false` | 允许 BIND 命令 |
| `udp_bind_port` | uint16 | `0` | UDP绑定端口（0=自动分配） |
| `udp_idle_timeout` | uint32 | `60` | UDP 会话空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |
| `enable_auth` | bool | `false` | 启用 RFC 1929 认证 |

SOCKS5 配置示例：

```json
{
    "protocol": {
        "socks5": {
            "enable_tcp": true,
            "enable_udp": true,
            "udp_idle_timeout": 120,
            "enable_auth": true
        }
    }
}
```

#### trojan 子配置

对应类型：`psm::protocol::trojan::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_tcp` | bool | `true` | 允许 TCP 隧道 |
| `enable_udp` | bool | `false` | 允许 UDP over TLS |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

Trojan 配置示例：

```json
{
    "protocol": {
        "trojan": {
            "enable_tcp": true,
            "enable_udp": true,
            "udp_idle_timeout": 60
        }
    }
}
```

#### vless 子配置

对应类型：`psm::protocol::vless::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_udp` | bool | `false` | 允许 UDP 命令 |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

VLESS 配置示例：

```json
{
    "protocol": {
        "vless": {
            "enable_udp": true,
            "udp_idle_timeout": 60
        }
    }
}
```

#### shadowsocks 子配置

对应类型：`psm::protocol::shadowsocks::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `psk` | string | — | Base64 编码预共享密钥 |
| `method` | string | — | 加密方法（如 `2022-blake3-aes-128-gcm`） |
| `enable_tcp` | bool | `true` | 启用 TCP 代理 |
| `enable_udp` | bool | `false` | 启用 UDP 代理 |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |

密钥长度对应：

| 密钥长度 | 加密方法 |
|----------|----------|
| 16 字节 | AES-128-GCM |
| 32 字节 | AES-256-GCM |

Shadowsocks 2022 配置示例：

```json
{
    "protocol": {
        "shadowsocks": {
            "psk": "MTIzNDU2Nzg5MDEyMzQ1Ng==",
            "method": "2022-blake3-aes-256-gcm",
            "enable_tcp": true,
            "enable_udp": true
        }
    }
}
```

---

### multiplex — 多路复用配置

对应类型：`psm::multiplex::config`

#### 全局参数

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enabled` | bool | `false` | 是否启用多路复用 |

#### smux 子配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_streams` | uint32 | `32` | 单会话最大并发流数 |
| `buffer_size` | uint32 | `4096` | 每流读取缓冲区（上限 65535） |
| `keepalive_interval_ms` | uint32 | `30000` | NOP 心跳间隔（ms），0=禁用 |
| `udp_idle_timeout_ms` | uint32 | `60000` | UDP 管道空闲超时（ms） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

#### yamux 子配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_streams` | uint32 | `32` | 单会话最大并发流数 |
| `buffer_size` | uint32 | `4096` | 每流读取缓冲区 |
| `initial_window` | uint32 | `262144` | 初始流窗口大小（字节） |
| `enable_ping` | bool | `true` | 启用心跳 |
| `ping_interval_ms` | uint32 | `30000` | 心跳间隔（ms） |
| `stream_open_timeout_ms` | uint32 | `30000` | 流打开超时（ms） |
| `stream_close_timeout_ms` | uint32 | `30000` | 流关闭超时（ms） |
| `udp_idle_timeout_ms` | uint32 | `60000` | UDP 管道空闲超时（ms） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

多路复用配置示例：

```json
{
    "multiplex": {
        "enabled": true,
        "smux": {
            "max_streams": 64,
            "buffer_size": 8192,
            "keepalive_interval_ms": 15000
        },
        "yamux": {
            "max_streams": 256,
            "buffer_size": 16384,
            "initial_window": 524288,
            "enable_ping": true
        }
    }
}
```

窗口大小调优：

| 场景 | 建议 initial_window | 原因 |
|------|---------------------|------|
| 低延迟 | 64KB-128KB | 减少内存占用 |
| 高延迟 | 256KB-512KB | 提高吞吐量 |
| 大文件传输 | 512KB-1MB | 最大化带宽利用 |

---

### stealth — TLS 伪装配置

对应类型：`psm::agent::stealth::config`

#### reality 子配置

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `dest` | string | 伪装目标（如 `www.microsoft.com:443`） |
| `server_names` | string[] | 允许的 SNI 列表 |
| `private_key` | string | Base64 编码 X25519 私钥 |
| `short_ids` | string[] | Short ID 列表（客户端标记） |

Reality 配置示例：

```json
{
    "stealth": {
        "reality": {
            "dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com", "microsoft.com"],
            "private_key": "YOUR_PRIVATE_KEY_BASE64",
            "short_ids": ["0123456789abcdef", "abcdef0123456789"]
        }
    }
}
```

密钥生成：

```bash
# 使用 Prism 脚本生成密钥对
./scripts/GenRealityKeys.sh

# 输出
Private Key: ABCDEF...（服务器配置）
Public Key: 123456...（客户端配置）
```

#### shadowtls 子配置

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `version` | int | ShadowTLS 版本（2 或 3） |
| `password` | string | 全局密码 |
| `users[]` | array | 用户列表（name + password） |
| `handshake_dest` | string | TLS握手目标 |
| `server_names` | string[] | 允许的 SNI 列表 |
| `strict_mode` | bool | 严格模式 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |

ShadowTLS 配置示例：

```json
{
    "stealth": {
        "shadowtls": {
            "version": 3,
            "password": "your_shadowtls_password",
            "handshake_dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com"],
            "strict_mode": true,
            "handshake_timeout_ms": 5000
        }
    }
}
```

#### restls 子配置

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | 允许的 SNI 列表 |
| `host` | string | 伪装主机 |
| `password` | string | 密码 |
| `version_hint` | string | TLS 版本提示 |
| `restls_script` | string | 流量伪装脚本 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |

#### anytls 子配置

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | SNI 列表 |
| `certificate` | string | 证书文件路径 |
| `private_key` | string | 私钥文件路径 |
| `users[]` | array | 用户列表 |
| `ech_key` | string | ECH 密钥 |
| `padding_scheme` | string | 填充方案 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |
| `idle_session_timeout_ms` | uint32 | 空闲会话超时（ms） |

#### trusttunnel 子配置

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | SNI 列表 |
| `certificate` | string | 证书文件路径 |
| `private_key` | string | 私钥文件路径 |
| `users[]` | array | 用户列表 |
| `network` | string | 网络类型（tcp/udp/both） |
| `congestion` | string | 拥塞控制算法（如 bbr） |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |
| `idle_timeout_ms` | uint32 | 空闲超时（ms） |

---

### dns — DNS 解析器配置

对应类型：`psm::resolve::dns::config`

#### 上游服务器配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `servers[]` | array | — | DNS 服务器列表 |
| `servers[].address` | string | — | 服务器地址 |
| `servers[].port` | uint16 | `53` | 端口 |
| `servers[].protocol` | enum | `"udp"` | 协议（udp/tcp/tls/https） |
| `servers[].timeout_ms` | uint32 | `5000` | 单服务器超时（ms） |

协议类型：

| 值 | 协议 | 说明 |
|----|------|------|
| `udp` | DNS over UDP | 传统 DNS |
| `tcp` | DNS over TCP | 大响应、DNSSEC |
| `tls` | DNS over TLS (DoT) | 端口 853，加密 |
| `https` | DNS over HTTPS (DoH) | 端口 443，加密伪装 |

#### 查询模式配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `mode` | enum | `"fastest"` | 查询模式 |
| `timeout_ms` | uint32 | `5000` | 全局超时（ms） |

查询模式：

| 值 | 行为 |
|----|------|
| `fastest` | 并行查询，使用最低延迟响应 |
| `first` | 并行查询，使用首个成功响应 |
| `fallback` | 顺序查询，失败时尝试下一个 |

#### 缓存配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `cache_enabled` | bool | `true` | 启用 DNS 缓存 |
| `cache_size` | size_t | `10000` | 缓存最大条目数 |
| `cache_ttl` | seconds | `120` | 缓存默认 TTL |
| `serve_stale` | bool | `true` | 过期后仍提供缓存 |
| `negative_ttl` | seconds | `300` | 负缓存 TTL |
| `ttl_min` | uint32 | `60` | 最小 TTL 钳制 |
| `ttl_max` | uint32 | `86400` | 最大 TTL 钳制 |

#### IPv6 配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `disable_ipv6` | bool | `false` | 禁用 IPv6（跳过 AAAA 查询） |

DNS 配置示例：

```json
{
    "dns": {
        "servers": [
            {
                "address": "8.8.8.8",
                "port": 53,
                "protocol": "udp",
                "timeout_ms": 3000
            },
            {
                "address": "dns.google",
                "port": 443,
                "protocol": "https",
                "timeout_ms": 5000
            }
        ],
        "mode": "fastest",
        "cache_enabled": true,
        "cache_size": 10000,
        "cache_ttl": 300,
        "serve_stale": true,
        "disable_ipv6": false
    }
}
```

---

### trace — 日志配置

对应类型：`psm::trace::config`（基于 spdlog）

#### 文件配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `file_name` | string | `"prism.log"` | 日志文件名 |
| `path_name` | string | `"logs"` | 日志目录路径 |
| `max_size` | uint64 | `67108864` (64MB) | 单文件最大大小 |
| `max_files` | uint32 | `8` | 最大轮转文件数 |

#### 队列配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `queue_size` | uint32 | `8192` | 异步队列大小 |
| `thread_count` | uint32 | `1` | 后台刷盘线程数 |

#### 输出配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_console` | bool | `true` | 启用控制台输出 |
| `enable_file` | bool | `true` | 启用文件输出 |
| `log_level` | string | `"info"` | 日志级别 |

日志级别：

| 值 | 级别 | 说明 |
|----|------|------|
| `trace` | TRACE | 最详细调试信息 |
| `debug` | DEBUG | 调试信息 |
| `info` | INFO | 常规信息 |
| `warn` | WARN | 警告 |
| `error` | ERROR | 错误 |
| `critical` | CRITICAL | 严重错误 |

#### 格式配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `pattern` | string | `"[...] [%l] %v"` | 日志格式 |
| `trace_name` | string | `"prism"` | 日志追踪名称 |

日志配置示例：

```json
{
    "trace": {
        "file_name": "prism.log",
        "path_name": "/var/log/prism",
        "max_size": 134217728,
        "max_files": 10,
        "enable_console": false,
        "enable_file": true,
        "log_level": "info"
    }
}
```

---

## 使用示例

### 完整配置示例

```json
{
    "agent": {
        "addressable": {
            "host": "0.0.0.0",
            "port": 443
        },
        "certificate": {
            "key": "/etc/ssl/private/server.key",
            "cert": "/etc/ssl/certs/server.crt"
        },
        "authentication": {
            "users": [
                {
                    "password": "secure_password_123",
                    "uuid": "550e8400-e29b-41d4-a716-446655440000",
                    "max_connections": 100
                }
            ]
        },
        "camouflage": "/api/v1/stream",
        "limit": {
            "blacklist": true
        }
    },
    
    "pool": {
        "max_cache_per_endpoint": 64,
        "max_idle_seconds": 60,
        "connect_timeout_ms": 500,
        "recv_buffer_size": 131072,
        "send_buffer_size": 131072,
        "tcp_nodelay": true,
        "keep_alive": true
    },
    
    "buffer": {
        "size": 524288
    },
    
    "protocol": {
        "socks5": {
            "enable_tcp": true,
            "enable_udp": true,
            "enable_auth": true
        },
        "trojan": {
            "enable_tcp": true,
            "enable_udp": true
        },
        "vless": {
            "enable_udp": true
        }
    },
    
    "multiplex": {
        "enabled": true,
        "yamux": {
            "max_streams": 128,
            "initial_window": 524288
        }
    },
    
    "stealth": {
        "reality": {
            "dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com"],
            "private_key": "YOUR_KEY",
            "short_ids": ["abc123"]
        }
    },
    
    "dns": {
        "servers": [
            {"address": "dns.google", "protocol": "https"}
        ],
        "mode": "fastest",
        "cache_enabled": true
    },
    
    "trace": {
        "log_level": "info",
        "enable_file": true
    }
}
```

### Reality 代理配置

```json
{
    "agent": {
        "addressable": {"port": 443}
    },
    "stealth": {
        "reality": {
            "dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com"],
            "private_key": "GENERATED_PRIVATE_KEY",
            "short_ids": ["6ba85179e30d"]
        }
    },
    "protocol": {
        "vless": {"enable_udp": true}
    }
}
```

### ShadowTLS + SS2022 配置

```json
{
    "agent": {
        "addressable": {"port": 443}
    },
    "stealth": {
        "shadowtls": {
            "version": 3,
            "password": "shadowtls_password",
            "handshake_dest": "www.microsoft.com:443"
        }
    },
    "protocol": {
        "shadowsocks": {
            "psk": "BASE64_PSK",
            "method": "2022-blake3-aes-256-gcm",
            "enable_tcp": true
        }
    }
}
```

---

## 最佳实践

### 安全配置要点

| 参数 | 建议 | 原因 |
|------|------|------|
| 用户密码 | 强密码 + SHA224 | 防止弱密码攻击 |
| max_connections | 设置限制 | 防止资源耗尽 |
| certificate | Let's Encrypt 或 Reality | 避免自签名 |
| blacklist | 启用 | 过滤恶意 IP |

### 性能配置要点

| 参数 | 建议 | 原因 |
|------|------|------|
| max_cache_per_endpoint | 32-128 | 平衡内存和复用 |
| buffer_size | 根据网络延迟调整 | 高延迟用大缓冲 |
| initial_window | 256KB-512KB | 高延迟链路吞吐 |
| cache_ttl | 300-600 | DNS 缓存效率 |

### 伪装配置要点

| 场景 | 推荐 |
|------|------|
| 无域名无证书 | Reality |
| 有域名有证书 | Trojan |
| 极端审查 | Reality + 回落 |

---

## 常见问题

### Q1: 配置文件修改后如何生效？

**A**: 需重启 Prism 服务：
```bash
systemctl restart prism
# 或直接重启进程
kill -HUP <pid>  # 不支持热加载，需要完全重启
```

### Q2: 配置验证失败怎么办？

**A**: 检查：
- JSON 格式是否正确（使用 JSON 验证器）
- 字段名是否正确（区分大小写）
- 类型是否匹配（字符串/数字/布尔）

### Q3: 如何生成 Reality 密钥？

**A**: 使用 Prism 脚本：
```bash
./scripts/GenRealityKeys.sh
# 或 PowerShell
./scripts/GenRealityKeys.ps1
```

### Q4: 多个伪装方案如何选择？

**A**: Prism 按识别结果选择方案：
- ClientHello 匹配 Reality → Reality 处理
- 匹配 ShadowTLS → ShadowTLS 处理
- 其他 → 按配置默认处理

### Q5: DNS 配置如何防止泄露？

**A**: 
- 使用加密 DNS（DoH/DoT）
- DNS 查询走代理通道
- 禁用系统 DNS 缓存

---

## 排障指南

### 问题：配置加载失败

**症状**: 启动时报错找不到配置

**排查步骤**:

1. 检查配置文件路径
   ```bash
   ls configuration.json
   ```

2. 检查命令行参数
   ```bash
   prism --config /path/to/config.json
   ```

3. 检查 JSON 格式
   ```bash
   python -m json.tool configuration.json
   ```

---

### 问题：配置字段无效

**症状**: 启动报字段错误

**排查步骤**:

1. 检查字段名拼写
2. 检查字段类型（字符串需引号）
3. 检查枚举值是否有效

---

### 问题：Reality握手失败

**症状**: Reality 连接建立失败

**排查步骤**:

1. 检查密钥配置
   - 公钥私钥是否匹配
   - Base64 编码是否正确

2. 检查伪装目标
   ```bash
   curl -v https://www.microsoft.com
   ```

3. 检查 Short ID
   - 客户端和服务器是否匹配

---

### 问题：DNS 解析失败

**症状**: 无法解析域名

**排查步骤**:

1. 检查 DNS 服务器配置
   - 地址是否正确
   - 协议是否支持

2. 测试 DNS 连接
   ```bash
   dig @dns.google example.com
   ```

3. 检查超时设置
   - timeout_ms 是否足够

---

## 相关链接

- [[modules/agent]] — Agent 模块
- [[modules/channel]] — Channel 连接池
- [[modules/multiplex]] — Multiplex 多路复用
- [[modules/stealth]] — Stealth TLS伪装
- [[modules/resolve]] — DNS 解析模块