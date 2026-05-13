---
title: 完整配置参考
tags: [configuration, json, agent, pool, protocol, multiplex, stealth, dns, trace]
source:
  - I:/code/Prism/src/configuration.json
  - I:/code/Prism/include/prism/config.hpp
  - I:/code/Prism/include/prism/agent/config.hpp
  - I:/code/Prism/include/prism/channel/connection/pool.hpp
  - I:/code/Prism/include/prism/multiplex/config.hpp
  - I:/code/Prism/include/prism/resolve/dns/config.hpp
  - I:/code/Prism/include/prism/trace/config.hpp
---

# 完整配置参考

Prism 使用 JSON 配置文件，由 `glaze` 库反序列化为 C++ 强类型结构体 `psm::config`。

## 配置文件查找顺序

1. **命令行参数**指定的路径（优先）
2. **可执行文件同目录**下的 `configuration.json`（默认）

## 顶层结构

```json
{
    "agent": { ... },       // 代理服务核心配置
    "pool": { ... },        // 连接池配置
    "buffer": { ... },      // 缓冲区配置
    "protocol": { ... },    // 协议配置 (socks5/trojan/vless/shadowsocks)
    "multiplex": { ... },   // 多路复用配置 (smux/yamux)
    "stealth": { ... },     // 伪装配置 (reality/shadowtls/restls/anytls/trusttunnel)
    "dns": { ... },         // DNS 解析器配置
    "trace": { ... }        // 日志追踪配置
}
```

对应 C++ 结构：`psm::config`（`include/prism/config.hpp`）

---

## agent — 代理服务

对应类型：`psm::agent::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `limit.blacklist` | bool | `true` | 是否启用 IP 黑名单 |
| `positive.host` | string | `""` | 正向代理目标主机名（空表示不使用） |
| `positive.port` | uint16 | `0` | 正向代理目标端口 |
| `addressable.host` | string | `"0.0.0.0"` | 监听地址 |
| `addressable.port` | uint16 | `8081` | 监听端口 |
| `certificate.key` | string | — | TLS 私钥文件路径（PEM） |
| `certificate.cert` | string | — | TLS 证书文件路径（PEM） |
| `authentication.users[]` | array | — | 统一用户列表 |
| `authentication.users[].password` | string | — | 密码（用于 Trojan/HTTP/SOCKS5，启动时自动转 SHA224） |
| `authentication.users[].uuid` | string | — | VLESS UUID |
| `authentication.users[].max_connections` | uint32 | `0` | 最大并发连接数（0 = 无限制） |
| `camouflage` | string | `""` | 伪装路径，抗探测用（如 `/api/notification/v1/stream`） |
| `reverse_map` | object | `{}` | 反向代理路由表（域名→后端端点） |

---

## pool — 连接池

对应类型：`psm::channel::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_cache_per_endpoint` | uint32 | `32` | 单端点最大缓存连接数 |
| `max_idle_seconds` | uint64 | `30` | 空闲连接最大存活时间（秒） |
| `connect_timeout_ms` | uint64 | `300` | 新建连接超时（毫秒） |
| `cleanup_interval_sec` | uint64 | `10` | 后台清理间隔（秒） |
| `recv_buffer_size` | uint32 | `65536` | 接收缓冲区大小（字节） |
| `send_buffer_size` | uint32 | `65536` | 发送缓冲区大小（字节） |
| `tcp_nodelay` | bool | `true` | 启用 TCP_NODELAY |
| `keep_alive` | bool | `true` | 启用 SO_KEEPALIVE |
| `cache_ipv6` | bool | `false` | 是否缓存 IPv6 连接 |

连接池采用 LIFO 栈式缓存，支持僵尸检测和线程隔离。详见 [[channel]]。

---

## buffer — 缓冲区

对应类型：`psm::agent::buffer`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `size` | uint32 | `262144` (256KB) | 数据传输缓冲区大小（字节） |

高延迟高带宽环境建议增大此值。

---

## protocol — 协议配置

### socks5

对应类型：`psm::protocol::socks5::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_tcp` | bool | `true` | 允许 CONNECT 命令 |
| `enable_udp` | bool | `true` | 允许 UDP_ASSOCIATE 命令 |
| `enable_bind` | bool | `false` | 允许 BIND 命令 |
| `udp_bind_port` | uint16 | `0` | UDP 绑定端口（0 = 自动分配） |
| `udp_idle_timeout` | uint32 | `60` | UDP 会话空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |
| `enable_auth` | bool | `false` | 启用 RFC 1929 用户名/密码认证 |

### trojan

对应类型：`psm::protocol::trojan::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_tcp` | bool | `true` | 允许 TCP 隧道 |
| `enable_udp` | bool | `false` | 允许 UDP over TLS |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

### vless

对应类型：`psm::protocol::vless::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enable_udp` | bool | `false` | 允许 UDP 命令 |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

### shadowsocks

对应类型：`psm::protocol::shadowsocks::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `psk` | string | — | Base64 编码的预共享密钥（16B → AES-128, 32B → AES-256） |
| `method` | string | — | 加密方法（如 `2022-blake3-aes-128-gcm`） |
| `enable_tcp` | bool | `true` | 启用 TCP 代理 |
| `enable_udp` | bool | `false` | 启用 UDP 代理 |
| `udp_idle_timeout` | uint32 | `60` | UDP 空闲超时（秒） |

---

## multiplex — 多路复用

对应类型：`psm::multiplex::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `enabled` | bool | `false` | 是否启用多路复用 |

### smux 子配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_streams` | uint32 | `32` | 单会话最大并发流数 |
| `buffer_size` | uint32 | `4096` | 每流读取缓冲区（上限 65535） |
| `keepalive_interval_ms` | uint32 | `30000` | NOP 心跳间隔（ms），0 = 禁用 |
| `udp_idle_timeout_ms` | uint32 | `60000` | UDP 管道空闲超时（ms） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

### yamux 子配置

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `max_streams` | uint32 | `32` | 单会话最大并发流数 |
| `buffer_size` | uint32 | `4096` | 每流读取缓冲区 |
| `initial_window` | uint32 | `262144` | 初始流窗口大小（字节），增大可提升高延迟链路吞吐 |
| `enable_ping` | bool | `true` | 启用心跳 |
| `ping_interval_ms` | uint32 | `30000` | 心跳间隔（ms） |
| `stream_open_timeout_ms` | uint32 | `30000` | 流打开超时（ms） |
| `stream_close_timeout_ms` | uint32 | `30000` | 流关闭超时（ms） |
| `udp_idle_timeout_ms` | uint32 | `60000` | UDP 管道空闲超时（ms） |
| `udp_max_datagram` | uint32 | `65535` | UDP 数据报最大长度 |

---

## stealth — 伪装配置

对应类型：`psm::agent::stealth::config`

### reality

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `dest` | string | 伪装目标（如 `www.microsoft.com:443`） |
| `server_names` | string[] | 允许的 SNI 列表 |
| `private_key` | string | Base64 编码的 X25519 私钥 |
| `short_ids` | string[] | 短 ID 列表 |

### shadowtls

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `version` | int | ShadowTLS 版本（2 或 3） |
| `password` | string | 全局密码 |
| `users[]` | array | 用户列表（name + password） |
| `handshake_dest` | string | TLS 握手目标 |
| `server_names` | string[] | 允许的 SNI 列表 |
| `strict_mode` | bool | 严格模式 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |

### restls

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | 允许的 SNI 列表 |
| `host` | string | 伪装主机 |
| `password` | string | 密码 |
| `version_hint` | string | TLS 版本提示 |
| `restls_script` | string | 流量伪装脚本 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |

### anytls

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | SNI 列表 |
| `certificate` | string | 证书文件路径 |
| `private_key` | string | 私钥文件路径 |
| `users[]` | array | 用户列表（username + password） |
| `ech_key` | string | ECH 密钥 |
| `padding_scheme` | string | 填充方案 |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |
| `idle_session_timeout_ms` | uint32 | 空闲会话超时（ms） |

### trusttunnel

| JSON 字段 | 类型 | 说明 |
|-----------|------|------|
| `server_names` | string[] | SNI 列表 |
| `certificate` | string | 证书文件路径 |
| `private_key` | string | 私钥文件路径 |
| `users[]` | array | 用户列表（username + password） |
| `network` | string | 网络类型（`tcp`/`udp`/`both`） |
| `congestion` | string | 拥塞控制算法（如 `bbr`） |
| `handshake_timeout_ms` | uint32 | 握手超时（ms） |
| `idle_timeout_ms` | uint32 | 空闲超时（ms） |

---

## dns — DNS 解析器

对应类型：`psm::resolve::dns::config`

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `servers[]` | array | — | 上游 DNS 服务器列表 |
| `servers[].address` | string | — | 服务器地址 |
| `servers[].port` | uint16 | `53` | 端口 |
| `servers[].protocol` | enum | `"udp"` | 协议：`udp`/`tcp`/`tls`(DoT)/`https`(DoH) |
| `servers[].timeout_ms` | uint32 | `5000` | 单服务器超时（ms） |
| `mode` | enum | `"fastest"` | 查询模式：`fastest`（最低延迟）/`first`（首个成功）/`fallback`（顺序重试） |
| `timeout_ms` | uint32 | `5000` | 全局超时（ms） |
| `cache_enabled` | bool | `true` | 启用 DNS 缓存 |
| `cache_size` | size_t | `10000` | 缓存最大条目数 |
| `cache_ttl` | seconds | `120` | 缓存默认 TTL |
| `serve_stale` | bool | `true` | 过期后仍提供缓存数据 |
| `negative_ttl` | seconds | `300` | 负缓存 TTL（失败域名缓存时间） |
| `ttl_min` | uint32 | `60` | 最小 TTL 钳制值（秒） |
| `ttl_max` | uint32 | `86400` | 最大 TTL 钳制值（秒） |
| `disable_ipv6` | bool | `false` | 禁用 IPv6（跳过 AAAA 查询） |

---

## trace — 日志

对应类型：`psm::trace::config`（基于 spdlog）

| JSON 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `file_name` | string | `"prism.log"` | 日志文件名 |
| `path_name` | string | `"logs"` | 日志目录路径 |
| `max_size` | uint64 | `67108864` (64MB) | 单文件最大大小 |
| `max_files` | uint32 | `8` | 最大轮转文件数 |
| `queue_size` | uint32 | `8192` | 异步队列大小 |
| `thread_count` | uint32 | `1` | 后台刷盘线程数 |
| `enable_console` | bool | `true` | 启用控制台输出 |
| `enable_file` | bool | `true` | 启用文件输出 |
| `log_level` | string | `"info"` | 日志级别（trace/debug/info/warn/error/critical） |
| `pattern` | string | `"[...] [%l] %v"` | 日志格式 |
| `trace_name` | string | `"prism"` | 日志追踪名称 |

---

## 相关链接

- [[Architecture]] — 分层架构概览
- [[Agent]] — 代理服务模块
- [[channel]] — 连接池实现
- [[Multiplex]] — 多路复用模块
- [[Stealth]] — TLS 伪装模块
- [[DNS]] — DNS 解析模块
