---
title: Prism 完整日志参考字典
source:
  - src/prism/
module: debugging
type: reference
tags: [debugging, logging, reference, trace, diagnostics]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[dev/debugging/log-analysis]]"
  - "[[core/fault/code]]"
  - "[[core/trace/overview]]"
---

# Prism 完整日志参考字典

本文档是 Prism 运行时日志的完整参考字典，按模块分组，覆盖所有日志标签、消息模板、级别和触发条件。用于排障时快速定位日志含义、关联错误码。

## 日志格式规范

### 默认格式

```
[%Y-%m-%d %H:%M:%S.%e][%l] %v
```

示例输出：

```
[2026-05-21 14:32:01.487][info] handshake completed successfully
```

### 配置覆盖

当 `trace.config` 中自定义 `pattern` 字段时，可使用扩展格式：

```
[%Y-%m-%d %H:%M:%S.%e] [%5t] [%l] %v
```

| 占位符 | 含义 |
|--------|------|
| `%Y-%m-%d` | 日期 |
| `%H:%M:%S` | 时:分:秒 |
| `%e` | 毫秒 |
| `%5t` | 线程 ID，5 字符宽度右对齐 |
| `%l` | 日志级别 |
| `%v` | 消息正文 |

### 级别定义

| 级别 | 数值 | 含义 | 典型用途 |
|------|------|------|----------|
| trace | 0 | 最详细 | 帧级数据跟踪 |
| debug | 1 | 调试信息 | 控制流、状态变化 |
| info | 2 | 常规信息 | 生命周期事件 |
| warn | 3 | 警告 | 可恢复的异常 |
| error | 4 | 错误 | 操作失败 |
| critical | 5 | 严重 | 致命错误，进程可能终止 |

---

## Stealth -- Reality

Reality 是 Prism 的核心隐写协议。握手过程分为多个阶段，每个阶段使用独立的日志标签。

### [Stealth.Handshake] -- 握手主流程

Reality 握手的入口与主控逻辑，涵盖从 ClientHello 读取到握手完成的全部流程。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `failed to read TLS record: {error}` | error | Stage 1 读取 ClientHello 时 socket 错误 | 底层 IO 出错，可能是连接中断或 RST | `io_error`(10), `connection_reset`(24) |
| 2 | `failed to parse ClientHello: {error}` | error | ClientHello 字节流不符合 TLS 格式 | 数据损坏或非 TLS 流量 | `protocol_error`(5), `bad_message`(6) |
| 3 | `invalid dest config: {}` | error | fallback 目标地址配置缺失或格式错误 | 配置层面的问题 | `config_parse_error`(30) |
| 4 | `falling back to {host}:{port}` | info | SNI 不匹配或认证失败，进入标准 TLS fallback | 正常的 fallback 流程启动 | -- |
| 5 | `connect to dest failed: {}` | error | 尝试连接 fallback 目标失败 | 目标服务器不可达 | `upstream_unreachable`(17), `connection_refused`(18), `reality_dest_unreachable`(54) |
| 6 | `write to dest failed: {}` | error | 向 fallback 目标写入 ClientHello 原始数据失败 | fallback 连接建立后写入出错 | `io_error`(10), `connection_reset`(24) |
| 7 | `invalid private key length: {}` | error | 服务端配置的 Reality 私钥长度不等于 32 字节 | 配置错误，X25519 私钥必须为 32 字节 | `config_parse_error`(30) |
| 8 | `SNI mismatch, falling back to standard TLS` | info | ClientHello SNI 不在 server_names 列表中 | 客户端访问了非预期域名 | `reality_sni_mismatch`(51) |
| 9 | `auth failed with empty SNI, falling back to standard TLS` | info | SNI 为空且认证失败 | 客户端未发送 SNI 扩展 | `reality_auth_failed`(50) |
| 10 | `auth failed: {}, not Reality, passing to next scheme` | info | SNI 匹配但 Reality 认证失败 | 可能是普通 TLS 流量误匹配 | `reality_auth_failed`(50) |
| 11 | `authentication successful` | debug | Reality 认证通过 | 握手继续进行后续密钥交换 | -- |
| 12 | `ephemeral X25519 key exchange failed` | error | 服务端临时密钥交换失败 | 内部加密错误 | `reality_key_exchange_failed`(52) |
| 13 | `failed to generate ServerHello: {}` | error | 构造 ServerHello 响应失败 | 内部构造错误 | `reality_handshake_failed`(53) |
| 14 | `failed to derive keys: {}` | error | 握手密钥派生失败 | HKDF 计算出错 | `reality_key_schedule_error`(57) |
| 15 | `failed to send handshake records: {}` | error | 发送 ServerHello + Extensions + Finished 失败 | 网络 IO 错误 | `io_error`(10), `reality_handshake_failed`(53) |
| 16 | `failed to read client record header` | error | 等待客户端 Finished 时读取失败 | 客户端断开 | `io_error`(10) |
| 17 | `client sent TLS ALERT: level={}, desc=0x{:02x} -- server Finished was rejected` | error | 客户端发送 TLS Alert 而非 Finished | 客户端验证了伪装证书但拒绝了 Finished | `reality_handshake_failed`(53), `reality_certificate_error`(55) |
| 18 | `failed to decrypt client record (ec={}), raw {} bytes` | error | 使用客户端握手密钥解密失败 | 客户端发送的数据无法解密 | `crypto_error`(45), `reality_tls_record_error`(56) |
| 19 | `failed to derive application keys` | error | 从 master secret 派生应用密钥失败 | HKDF 计算出错 | `reality_key_schedule_error`(57) |
| 20 | `failed to read inner data: {}` | error | 预读内层数据（探测后续协议）失败 | 内层协议握手的第一包读取失败 | `io_error`(10) |
| 21 | `handshake completed successfully` | info | Reality 握手全流程完成 | 连接进入应用数据阶段 | -- |

### [Stealth.Auth] -- 认证子阶段

负责解析 ClientHello，提取 SNI / key_share / session_id，进行 Reality 认证。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `SNI mismatch: {}` | debug | SNI 与 server_names 不匹配 | 域名不在白名单 | `reality_sni_mismatch`(51) |
| 2 | `no X25519 public key in key_share` | debug | ClientHello 的 key_share 扩展中无 X25519 组 | 客户端不支持的曲线 | `reality_auth_failed`(50) |
| 3 | `client does not support TLS 1.3` | debug | supported_versions 扩展中无 TLS 1.3 | Reality 要求 TLS 1.3 | `reality_auth_failed`(50) |
| 4 | `session_id too short: {}` | debug | ClientHello 的 session_id 长度不足 | 无法承载 auth_id | `reality_auth_failed`(50) |
| 5 | `X25519 key exchange failed` | error | 使用客户端公钥进行 X25519 计算失败 | 公钥可能是无效点 | `reality_key_exchange_failed`(52) |
| 6 | `shared secret is all zeros (low-order point)` | error | X25519 输出全零 | 客户端发送了低阶点（可能的攻击） | `reality_key_exchange_failed`(52) |
| 7 | `HKDF-Expand failed` | error | HKDF-Expand-Label 计算失败 | 密钥派生内部错误 | `reality_key_schedule_error`(57) |
| 8 | `session_id decryption failed` | error | AES-GCM 解密 session_id 中的 auth_id 失败 | 认证数据解密失败 | `reality_auth_failed`(50), `crypto_error`(45) |
| 9 | `invalid version marker: 0x{:02x}` | error | auth_id 明文中的版本标记不等于 0x01 | 客户端版本不兼容 | `reality_auth_failed`(50) |
| 10 | `short_id mismatch` | debug | auth_id 中的 short_id 与服务端配置不匹配 | 认证失败 | `reality_auth_failed`(50) |

### [Stealth.KeySchedule] -- 密钥派生

TLS 1.3 密钥调度各个阶段，每步失败都会记录独立日志。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `failed to derive 'derived' secret` | error | early secret 派生 derived secret 失败 | HKDF-Extract/Expand 失败 | `reality_key_schedule_error`(57) |
| 2 | `failed to derive 'c hs traffic'` | error | 派生客户端握手流量密钥失败 | -- | `reality_key_schedule_error`(57) |
| 3 | `failed to derive 's hs traffic'` | error | 派生服务端握手流量密钥失败 | -- | `reality_key_schedule_error`(57) |
| 4 | `failed to derive server handshake key/iv` | error | 从握手流量密钥派生 AEAD key + iv 失败 | -- | `reality_key_schedule_error`(57) |
| 5 | `failed to derive client handshake key/iv` | error | 从握手流量密钥派生客户端 AEAD key + iv 失败 | -- | `reality_key_schedule_error`(57) |
| 6 | `failed to derive master 'derived' secret` | error | 从 handshake secret 派生 master derived secret 失败 | -- | `reality_key_schedule_error`(57) |
| 7 | `failed to derive server finished key` | error | 派生服务端 Finished HMAC key 失败 | -- | `reality_key_schedule_error`(57) |
| 8 | `failed to derive 's ap traffic'` | error | 派生服务端应用流量密钥失败 | -- | `reality_key_schedule_error`(57) |
| 9 | `failed to derive 'c ap traffic'` | error | 派生客户端应用流量密钥失败 | -- | `reality_key_schedule_error`(57) |
| 10 | `failed to derive server app key/iv` | error | 派生服务端应用 AEAD key + iv 失败 | -- | `reality_key_schedule_error`(57) |
| 11 | `failed to derive client app key/iv` | error | 派生客户端应用 AEAD key + iv 失败 | -- | `reality_key_schedule_error`(57) |

> KeySchedule 日志全部为 error 级别，任何一条出现都意味着握手必然失败。排查时应从第一条出现的日志向上回溯。

### [Stealth.ServerHello] -- ServerHello 构造

构造并发送 ServerHello、EncryptedExtensions、Finished 等握手记录。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `AEAD seal failed` | error | AEAD 加密握手记录失败 | 加密操作出错 | `crypto_error`(45) |
| 2 | `invalid auth_key length` | error | auth_key 长度不符合 AES-GCM 要求 | 配置或密钥派生错误 | `reality_key_schedule_error`(57) |
| 3 | `ED25519_keypair returned zero public key` | error | Ed25519 密钥对生成异常 | 系统随机数异常 | `crypto_error`(45) |
| 4 | `plaintext too short for Finished: {}` | error | Finished 消息明文过短 | HMAC 输出异常 | `reality_handshake_failed`(53) |
| 5 | `failed to encrypt handshake record` | error | 加密单条握手记录失败 | AEAD seal 失败 | `crypto_error`(45), `reality_handshake_failed`(53) |

### [Stealth.Session] -- 会话数据传输

握手完成后，管理应用数据的加解密和 TLS 记录层。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `executor called with null transport` | error | 会话传输层为空 | 内部编程错误 | `generic_error`(1) |
| 2 | `first decrypt: seq={}, cipher_len={}` | debug | 首次解密客户端数据 | 正常调试信息 | -- |
| 3 | `received TLS alert record` | warn | 收到对端发送的 TLS Alert 记录 | 对端通知关闭或错误 | `reality_tls_record_error`(56) |
| 4 | `unexpected content type: 0x{:02x}` | error | TLS 记录的 content_type 不是预期的 | 协议层异常 | `protocol_error`(5), `reality_tls_record_error`(56) |
| 5 | `record too short for AEAD tag` | error | 接收到的记录长度不足 AEAD tag 所需 | 数据截断或损坏 | `bad_message`(6), `reality_tls_record_error`(56) |
| 6 | `AEAD decrypt failed` | error | AEAD 解密失败 | 密钥不匹配或数据被篡改 | `crypto_error`(45), `reality_tls_record_error`(56) |
| 7 | `AEAD encrypt failed` | error | AEAD 加密失败 | 加密操作出错 | `crypto_error`(45) |

---

## Stealth -- ShadowTLS

ShadowTLS 是另一种隐写方案，通过 HMAC 验证实现流量伪装。

### [ShadowTLS] -- ShadowTLS 主流程

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `Empty ClientHello` | warn | 收到的 ClientHello 为空或长度为零 | 客户端异常或探测 | `bad_message`(6) |
| 2 | `ClientHello HMAC verification failed` | debug | ClientHello 中的 HMAC 不匹配 | 非授权客户端或密码不匹配 | `auth_failed`(15) |
| 3 | `Client authenticated (user: {})` | info | HMAC 验证通过 | 认证成功 | -- |
| 4 | `connecting to backend: {}:{}` | debug | 开始连接后端代理服务器 | 正常连接流程 | -- |
| 5 | `Backend connection failed: {}` | error | 连接后端服务器失败 | 后端不可达 | `upstream_unreachable`(17), `connection_refused`(18) |
| 6 | `Backend does not support TLS 1.3, strict mode enabled` | error | strict 模式要求后端支持 TLS 1.3 但后端不支持 | 后端配置不兼容 | `tls_handshake_failed`(13) |
| 7 | `relay coroutine did not exit within timeout, socket may be corrupted` | warn | relay 协程超时未退出 | 资源泄漏风险 | `timeout`(11) |

### [ShadowTLS.Relay] -- 中继转发

在握手阶段和稳态阶段中继客户端与后端之间的数据。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `client first frame HMAC matched, payload_size={}` | debug | 客户端首个 TLS 帧的 HMAC 验证通过 | 正常握手流程 | -- |
| 2 | `HMAC match failed during handshake relay` | warn | 握手中继阶段 HMAC 不匹配 | 可能是数据被修改 | `auth_failed`(15) |
| 3 | `read_tls_frame from client returned nullopt` | debug | 从客户端读取 TLS 帧失败 | 客户端已断开 | `eof`(3), `io_error`(10) |
| 4 | `write to backend/client failed` | error | 向后端或客户端写入数据失败 | 连接中断 | `io_error`(10), `connection_reset`(24) |

### [ShadowTLS.Transport] -- 传输层

ShadowTLS 稳态传输的 HMAC 校验和 TLS 帧处理。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `hmac_read_ctx is null` | error | HMAC 读取上下文未初始化 | 内部编程错误 | `generic_error`(1) |
| 2 | `HMAC mismatch in transport read_tls_frame` | error | 稳态传输中 HMAC 校验失败 | 数据被篡改或密钥不匹配 | `auth_failed`(15), `crypto_error`(45) |
| 3 | `payload too small for HMAC: {}` | error | 接收到的载荷长度小于 HMAC 长度 | 数据截断 | `bad_message`(6) |
| 4 | `unexpected TLS record type: 0x{:02x}` | error | TLS 记录类型不是预期的 | 协议异常 | `protocol_error`(5) |

---

## Protocol -- SS2022

Shadowsocks 2022 协议的日志，涵盖 TCP 中继和 UDP 包处理。

### [SS2022.Relay] -- TCP 中继

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `invalid PSK configuration` | error | PSK 格式不合法（长度错误、base64 解码失败） | 配置错误 | `invalid_psk`(46) |
| 2 | `salt replay detected` | warn | TCP 连接的 salt 在重放窗口内重复 | 可能的重放攻击或连接异常 | `replay_detected`(48) |
| 3 | `timestamp expired: client_ts={}, server_ts={}, diff={}s` | warn | 请求头中的时间戳与服务器时间差超过阈值 | 客户端时钟偏移或重放 | `timestamp_expired`(47) |
| 4 | `decrypt fixed header failed` | error | 解密 SS2022 固定头部失败 | 密钥不匹配或数据损坏 | `crypto_error`(45) |
| 5 | `decrypt payload block failed` | error | 解密载荷数据块失败 | 传输中途数据损坏 | `crypto_error`(45) |

### [SS2022.UDP] -- UDP 包处理

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `invalid PSK configuration` | error | PSK 配置不合法 | 配置错误 | `invalid_psk`(46) |
| 2 | `packet replay detected: packet_id={}` | warn | UDP 包的 packet_id 在重放窗口内重复 | 可能的重放攻击 | `packet_replay_detected`(59) |
| 3 | `decrypt fixed header failed` | error | UDP 包头部解密失败 | 密钥不匹配 | `crypto_error`(45) |
| 4 | `decrypt payload block failed` | error | UDP 载荷解密失败 | 数据损坏 | `crypto_error`(45) |

---

## Protocol -- 其他协议

### [Protocol.Trojan] -- Trojan 协议

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `credential verification failed` | warn | SHA224(password) 与请求中的 hash 不匹配 | 密码错误 | `auth_failed`(15) |
| 2 | `handshake failed: {code}` | error | Trojan 握手阶段失败 | CRLF 地址行解析失败 | `protocol_error`(5) |
| 3 | `mux session started` | info | Trojan 开启 mux 多路复用会话 | 正常生命周期 | -- |
| 4 | `CONNECT -> {host}:{port}` | debug | 解析到 CONNECT 命令 | 正常连接建立 | -- |

### [Protocol.Vless] -- VLESS 协议

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `credential verification failed` | warn | UUID 认证失败 | UUID 不匹配 | `auth_failed`(15) |
| 2 | `handshake failed: {code}` | error | VLESS 握手失败 | 版本号或命令不合法 | `protocol_error`(5) |
| 3 | `mux session started` | info | VLESS mux 会话建立 | 正常生命周期 | -- |
| 4 | `UDP associate started` | debug | VLESS UDP 关联开始 | 正常流程 | -- |
| 5 | `UDP associate completed` | info | VLESS UDP 关联正常结束 | 正常流程 | -- |
| 6 | `UDP associate failed` | error | VLESS UDP 关联失败 | -- | `protocol_error`(5) |

### [Protocol.Socks5] -- SOCKS5 协议

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `credential verification failed` | warn | 用户名/密码认证失败 | 认证信息错误 | `socks5_auth_negotiation_failed`(28) |
| 2 | `handshake failed: {code}` | error | SOCKS5 握手失败 | 版本号不匹配或方法不支持 | `protocol_error`(5) |
| 3 | `UDP associate started` | debug | SOCKS5 UDP associate 开始 | 正常流程 | -- |
| 4 | `UDP associate completed` | info | SOCKS5 UDP associate 结束 | 正常流程 | -- |
| 5 | `UDP associate failed` | error | SOCKS5 UDP associate 失败 | -- | `protocol_error`(5) |
| 6 | `IPv6 disabled: {host}:{port}` | info | 请求的地址是 IPv6 但 IPv6 被禁用 | 配置限制 | `ipv6_disabled`(37) |

### [Protocol.Http] -- HTTP CONNECT 协议

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `credential verification failed` | warn | HTTP Proxy-Authorization 认证失败 | 认证信息错误 | `auth_failed`(15) |
| 2 | `handshake failed: {code}` | error | HTTP CONNECT 请求解析失败 | 请求格式错误 | `protocol_error`(5) |
| 3 | `CONNECT -> {host}:{port}` | debug | 解析到 HTTP CONNECT 目标地址 | 正常连接建立 | -- |
| 4 | `IPv6 disabled: {host}:{port}` | info | CONNECT 目标为 IPv6 但被禁用 | 配置限制 | `ipv6_disabled`(37) |

---

## Multiplex

多路复用模块，包含 sing-mux、smux、yamux 三种子协议。

### [Mux.Bootstrap] -- 多路复用启动

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `sing-mux handshake completed, protocol={}` | debug | sing-mux 客户端/服务端握手成功 | 正常启动 | -- |
| 2 | `sing-mux negotiate failed: {}` | warn | sing-mux 版本协商失败 | 版本不兼容 | `mux_session_error`(39) |

### [Mux.Core] -- 多路复用会话管理

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `session exception: {}` | error | mux 会话发生异常 | 底层连接断开或协议错误 | `mux_session_error`(39) |
| 2 | `session closed` | debug | mux 会话正常关闭 | 生命周期结束 | -- |
| 3 | `stream {} pending, waiting for address` | debug | 新子流创建，等待目标地址 | 正常流创建流程 | -- |
| 4 | `stream {} connected to {}:{}` | debug | 子流成功连接到目标 | 正常连接完成 | -- |
| 5 | `stream {} UDP idle timeout` | debug | UDP 子流因空闲超时被回收 | 正常资源回收 | -- |
| 6 | `stream {} open timeout, resetting` | warn | 子流打开超时 | 目标不可达或网络延迟 | `timeout`(11), `mux_stream_error`(40) |

### [Mux.Duct] -- 数据通道

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `stream {} window update received, delta={}, new_window={}` | debug | 收到流量控制窗口更新 | Yamux 流控机制 | -- |
| 2 | `stream {} window exceeded, resetting` | warn | 发送数据超过对端窗口 | 流量控制违规 | `mux_window_exceeded`(41) |

### [Mux.Parcel] -- 帧编解码

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `invalid frame header` | error | 帧头格式不合法 | 数据损坏或版本不匹配 | `mux_protocol_error`(42) |
| 2 | `frame too large: {}` | error | 帧长度超过最大限制 | 可能的 DoS 攻击 | `message_too_large`(9) |

### [Smux.Craft] -- SMUX 协议实现

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `smux version mismatch: {}` | error | SMUX 协议版本不匹配 | 版本不兼容 | `mux_protocol_error`(42) |
| 2 | `smux stream {} closed` | debug | SMUX 子流关闭 | 正常关闭 | -- |
| 3 | `smux stream {} reset` | warn | SMUX 子流被重置 | 异常关闭 | `mux_stream_error`(40) |

### [Yamux.Craft] -- Yamux 协议实现

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `yamux ping received, id={}` | debug | 收到 Yamux Ping 帧 | 保活机制 | -- |
| 2 | `yamux stream {} window update, delta={}` | debug | Yamux 窗口更新 | 流控 | -- |
| 3 | `yamux protocol error: {}` | error | Yamux 协议帧解析失败 | 数据损坏 | `mux_protocol_error`(42) |
| 4 | `yamux stream {} go away: {}` | warn | 收到 Go Away 帧 | 对端要求关闭会话 | `mux_session_error`(39) |
| 5 | `yamux connection limit reached` | warn | 连接数达上限 | 资源限制 | `mux_connection_limit`(43) |
| 6 | `yamux stream limit reached` | warn | 子流数达上限 | 资源限制 | `mux_stream_limit`(44) |

---

## 连接管理

连接池、拨号、隧道、转发和竞速模块的日志。

### [Pool] -- 连接池

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `total acquires: {}, total hits: {}, total creates: {}, total evictions: {}` | debug | 连接池统计信息输出 | 定期或按需输出 | -- |
| 2 | `pool connection idle timeout, evicting` | debug | 空闲连接超时被驱逐 | 正常资源回收 | -- |
| 3 | `pool capacity reached, evicting oldest` | debug | 连接池满，驱逐最旧连接 | 正常管理 | -- |

### [Connect.Dial] -- 拨号连接

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `connect timed out to {addr}:{port}` | warn | TCP 连接超时 | 目标不可达或防火墙阻断 | `timeout`(11) |
| 2 | `connect failed: {error}` | error | TCP 连接失败 | 网络不可达 | `connection_refused`(18), `network_unreachable`(25) |
| 3 | `IPv6 connection not cached` | debug | IPv6 连接不放入连接池 | IPv6 连接通常不复用 | -- |

### [Connect.Tunnel] -- 隧道转发

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `Transfer: Upload {N} B, Download {N} B, duration: {N} ms` | debug | 隧道传输统计 | 连接结束时输出 | -- |
| 2 | `tunnel relay error: {}` | error | 隧道中继传输错误 | 传输中断 | `io_error`(10), `connection_reset`(24) |

### [Connect.Forward] -- 端口转发

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `forward listening on {addr}:{port}` | info | 转发监听启动 | 正常启动 | -- |
| 2 | `forward connection from {addr}:{port}` | debug | 收到新的转发连接 | 正常连接 | -- |

### [Racer] -- 竞速连接

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `racer: first to connect wins, others canceled` | debug | 多个候选同时拨号，最快的胜出 | Happy Eyeballs 策略 | -- |
| 2 | `racer: all attempts failed` | error | 所有候选地址连接失败 | 全部不可达 | `upstream_unreachable`(17) |

---

## DNS 解析

DNS 解析模块的日志，涵盖缓存命中、规则匹配、上游查询。

### [Resolve] -- DNS 解析

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `{domain} blocked by rule` | debug | 域名被路由规则拦截 | 规则生效 | `blocked`(21) |
| 2 | `{domain} negative rule hit` | debug | 域名匹配了否定规则（返回 NXDOMAIN） | 规则生效 | -- |
| 3 | `{domain} -> static address ({N} IPs)` | debug | 域名解析到静态地址（hosts 文件或配置） | 静态解析 | -- |
| 4 | `{domain} negative cache hit` | debug | 域名在否定缓存中 | 之前解析失败的结果缓存 | -- |
| 5 | `{domain} cache hit ({N} IPs)` | debug | 域名在正向缓存中命中 | 缓存有效 | -- |
| 6 | `{domain} -> {N} IPs in {ms}ms via {server}` | debug | 域名通过上游 DNS 解析成功 | 正常解析 | -- |
| 7 | `{domain} all IPs blacklisted` | warn | 解析到的所有 IP 都在黑名单中 | 可能的 DNS 污染 | `dns_failed`(16) |
| 8 | `{domain} failed: {error}` | error | DNS 查询失败 | 上游不可达 | `dns_failed`(16) |
| 9 | `upstream {server} timeout` | warn | 上游 DNS 服务器超时 | 网络问题 | `timeout`(11), `dns_failed`(16) |
| 10 | `cache eviction: {domain}` | debug | DNS 缓存条目被驱逐 | TTL 过期 | `ttl_expired`(35) |

---

## 会话与识别

协议识别、方案执行和会话管理的日志。

### [Session] -- 会话管理

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `Starting recognize lifecycle` | debug | 新会话开始协议识别 | 正常生命周期 | -- |
| 2 | `session closed normally` | debug | 会话正常关闭 | 生命周期结束 | -- |
| 3 | `session closed with error: {}` | warn | 会话异常关闭 | 传输错误 | `io_error`(10) |
| 4 | `session stats: upload={N}B, download={N}B, duration={N}ms` | debug | 会话结束统计 | 性能分析 | -- |

### [Recognition] -- 协议识别

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `Probe result: type={protocol}` | debug | 探针识别出协议类型 | 正常识别 | -- |
| 2 | `probe timeout, assuming unknown` | warn | 探针超时未能识别 | 网络延迟或数据不足 | `timeout`(11) |
| 3 | `probe read error: {}` | error | 探针读取数据失败 | 连接中断 | `io_error`(10) |

### [SchemeExecutor] -- 方案执行器

管理协议方案的逐个尝试和 fallback。

| # | 消息模板 | 级别 | 触发条件 | 含义 | 关联错误码 |
|---|----------|------|----------|------|-----------|
| 1 | `SNI '{sni}' matched {N} schemes` | debug | SNI 匹配到 N 个候选方案 | 正常匹配 | -- |
| 2 | `Deterministic hit: {scheme_name}` | debug | 唯一匹配到特定方案 | 确定性命中 | -- |
| 3 | `Scheme '{}' succeeded, protocol: {}` | info | 方案认证/握手成功 | 连接建立 | -- |
| 4 | `Scheme '{}' returned TLS, continuing to next` | debug | 方案返回 TLS 类型但未认证 | 继续尝试下一个 | -- |
| 5 | `Scheme '{}' failed but snapshot rewound, trying next` | debug | 方案失败，连接状态回滚 | 正常 fallback | -- |
| 6 | `All candidates failed, executing native fallback` | warn | 所有候选方案都失败 | 进入最终 fallback | `auth_failed`(15) |

---

## 错误码到日志的交叉索引

下表列出主要 `fault::code` 值对应的日志标签和消息关键词，用于从错误码快速定位日志。

### 通用与网络错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `io_error` | 10 | `[Stealth.Handshake]`, `[Stealth.Session]`, `[ShadowTLS.Relay]` | `failed to read`, `write to`, `read_tls_frame` |
| `timeout` | 11 | `[Connect.Dial]`, `[Mux.Core]`, `[Recognition]`, `[ShadowTLS]` | `timed out`, `open timeout`, `probe timeout`, `did not exit within timeout` |
| `connection_reset` | 24 | `[Stealth.Handshake]`, `[Connect.Tunnel]`, `[ShadowTLS.Relay]` | `failed to write`, `relay error`, `connection reset` |
| `auth_failed` | 15 | `[Protocol.Trojan]`, `[Protocol.Vless]`, `[Protocol.Socks5]`, `[Protocol.Http]`, `[ShadowTLS]` | `credential verification failed`, `HMAC verification failed` |

### Reality 错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `reality_auth_failed` | 50 | `[Stealth.Auth]`, `[Stealth.Handshake]` | `auth failed`, `session_id decryption failed`, `short_id mismatch` |
| `reality_sni_mismatch` | 51 | `[Stealth.Auth]`, `[Stealth.Handshake]` | `SNI mismatch`, `falling back to standard TLS` |
| `reality_key_exchange_failed` | 52 | `[Stealth.Auth]`, `[Stealth.Handshake]` | `X25519 key exchange failed`, `low-order point` |
| `reality_handshake_failed` | 53 | `[Stealth.Handshake]`, `[Stealth.ServerHello]` | `failed to generate ServerHello`, `TLS ALERT`, `too short for Finished` |
| `reality_dest_unreachable` | 54 | `[Stealth.Handshake]` | `connect to dest failed` |
| `reality_certificate_error` | 55 | `[Stealth.Handshake]` | `server Finished was rejected` |
| `reality_tls_record_error` | 56 | `[Stealth.Session]` | `unexpected content type`, `AEAD decrypt failed`, `too short for AEAD tag` |
| `reality_key_schedule_error` | 57 | `[Stealth.KeySchedule]`, `[Stealth.Handshake]` | `failed to derive`, `HKDF-Expand failed` |

### SS2022 加密错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `crypto_error` | 45 | `[SS2022.Relay]`, `[SS2022.UDP]`, `[Stealth.Session]`, `[Stealth.ServerHello]` | `decrypt`, `AEAD`, `encrypt` |
| `invalid_psk` | 46 | `[SS2022.Relay]`, `[SS2022.UDP]` | `invalid PSK configuration` |
| `timestamp_expired` | 47 | `[SS2022.Relay]` | `timestamp expired` |
| `replay_detected` | 48 | `[SS2022.Relay]` | `salt replay detected` |
| `packet_replay_detected` | 59 | `[SS2022.UDP]` | `packet replay detected` |

### DNS 错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `dns_failed` | 16 | `[Resolve]` | `failed`, `all IPs blacklisted`, `upstream timeout` |
| `blocked` | 21 | `[Resolve]` | `blocked by rule` |

### 多路复用错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `mux_session_error` | 39 | `[Mux.Core]`, `[Yamux.Craft]` | `session exception`, `go away`, `negotiate failed` |
| `mux_stream_error` | 40 | `[Mux.Core]`, `[Smux.Craft]` | `stream reset`, `open timeout` |
| `mux_window_exceeded` | 41 | `[Mux.Duct]` | `window exceeded` |
| `mux_protocol_error` | 42 | `[Mux.Parcel]`, `[Yamux.Craft]`, `[Smux.Craft]` | `invalid frame header`, `protocol error`, `version mismatch` |
| `mux_connection_limit` | 43 | `[Yamux.Craft]` | `connection limit reached` |
| `mux_stream_limit` | 44 | `[Yamux.Craft]` | `stream limit reached` |

### ECH 错误

| 错误码 | 值 | 日志标签 | 消息关键词 |
|--------|----|----------|-----------|
| `ech_payload_invalid` | 60 | ECH 处理相关标签 | `invalid ECH payload` |
| `ech_version_mismatch` | 61 | ECH 处理相关标签 | `ECH version` |
| `ech_decrypt_failed` | 62 | ECH 处理相关标签 | `ECH decrypt` |
| `ech_config_mismatch` | 63 | ECH 处理相关标签 | `ECH config` |

---

## 快速排查指南

### 按 symptom 查找

| 症状 | 应查找的日志标签 | 典型消息 |
|------|-----------------|----------|
| 客户端连接被拒绝 | `[Stealth.Handshake]` | `auth failed`, `SNI mismatch` |
| 握手挂起无响应 | `[Stealth.KeySchedule]` | `failed to derive` |
| 连接后立即断开 | `[Stealth.Session]` | `received TLS alert`, `AEAD decrypt failed` |
| 数据传输中断 | `[Connect.Tunnel]` | `Transfer:`, `relay error` |
| DNS 无法解析 | `[Resolve]` | `failed`, `all IPs blacklisted` |
| 多路复用连接异常 | `[Mux.Core]` | `session exception`, `open timeout` |
| SS2022 解密失败 | `[SS2022.Relay]` | `decrypt fixed header failed` |
| UDP 重放告警 | `[SS2022.UDP]` | `packet replay detected` |

### 按日志级别过滤

```bash
# 仅显示 error 级别（最紧急）
grep "\[error\]" prism.log

# error + warn（异常事件）
grep -E "\[error\]|\[warn\]" prism.log

# 特定模块的 debug 信息
grep "\[Stealth\." prism.log

# Reality 握手全流程
grep -E "\[Stealth\.(Handshake|Auth|KeySchedule|ServerHello)\]" prism.log

# ShadowTLS 相关
grep "\[ShadowTLS" prism.log

# 多路复用相关
grep -E "\[Mux\.|Smux\.|Yamux\." prism.log

# DNS 解析相关
grep "\[Resolve\]" prism.log
```

---

## 相关页面

- [[dev/debugging/log-analysis]] -- 日志分析方法与脚本
- [[core/fault/code]] -- 完整错误码枚举定义
- [[core/trace/overview]] -- Trace 日志模块配置
- [[dev/debugging/common-issues]] -- 常见问题汇总
- [[dev/debugging/protocol]] -- 协议问题排查
- [[dev/debugging/connection]] -- 连接问题排查
- [[dev/debugging/tls]] -- TLS 问题排查
