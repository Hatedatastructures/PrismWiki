---
title: ShadowTLS
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags:
  - stealth
  - tls
  - hmac
  - tier1
related:
  - "[[ech]]"
  - "[[restls]]"
  - "[[anytls]]"
  - "[[trusttunnel]]"
---

# ShadowTLS

ShadowTLS 是社区开发的 TLS 伪装方案，Prism 在 stealth 模块中实现了 **Tier 1** 级别的兼容版本，通过将代理流量包装成标准 TLS 1.3
连接来对抗深度包检测（DPI）。其核心特征是在 TLS ClientHello 的 SessionID 字段中嵌入
HMAC 标签进行身份验证。

## 工作原理

ShadowTLS 采用 **后端中继** 架构：

1. 客户端发送 TLS ClientHello，SessionID（32 字节）末尾 4 字节嵌入 HMAC-SHA1 标签
2. 服务端验证 HMAC，通过后建立到真实 TLS 服务器（如 www.microsoft.com）的 TCP 连接
3. 将客户端的 ClientHello 转发给后端，获取真实 ServerHello
4. 将 ServerHello 返回给客户端，提取 ServerRandom
5. 双工转发握手阶段数据，通过 HMAC 帧匹配定位客户端首帧认证数据
6. 握手完成后进入透明转发模式

外部观察者看到的是完全标准的 TLS 1.3 握手，证书链来自真实的后端服务器。

## 版本差异

### v2（兼容模式）

- 使用单一 `password` 进行 HMAC 认证
- 配置较简单，适合单用户场景

### v3（当前版本）

- 支持**多用户**认证，每个用户有独立的 `name` 和 `password`
- 认证时遍历用户列表，任一匹配即通过
- 支持 `strict_mode`：强制要求后端返回 TLS 1.3

## 认证流程

### ClientHello HMAC 验证

认证算法参照 sing-shadowtls 实现：

```
SessionID = 32 字节，末尾 4 字节为 HMAC 标签
HMAC 数据 = ClientHello[10:hmac_index] + 0x00000000 + ClientHello[hmac_index+4:]
HMAC 标签 = HMAC-SHA1(password, HMAC 数据)[:4]
```

验证过程：
1. 检查 TLS 记录类型为 `0x16`（Handshake）
2. 检查握手类型为 `0x01`（ClientHello）
3. 检查 SessionID 长度为 32 字节
4. 构建 HMAC 输入数据（SessionID HMAC 位置填零）
5. 计算 HMAC-SHA1 并与客户端标签进行**恒定时间比较**

### 数据帧 HMAC 验证

握手完成后，数据帧也带有 HMAC 保护：

- 客户端→服务端：`HMAC-SHA1(password, serverRandom + payload)[:4]`
- 服务端→客户端：`HMAC-SHA1(password, serverRandom + "S" + payload)[:4]`
- 写入密钥：`SHA256(password + serverRandom)`

数据帧格式：`[TLS Header(5)] [HMAC(4)] [payload]`

## 握手过程

完整握手流程（`handshake.cpp`）：

1. **ClientHello HMAC 验证** — 遍历用户列表，验证 SessionID 中的 HMAC
2. **建立后端连接** — 解析 `handshake_dest`（host:port），TCP 连接到后端
3. **转发 ClientHello** — 将原始 ClientHello 发送到后端
4. **读取 ServerHello** — 从后端读取并转发给客户端
5. **提取 ServerRandom** — 从 ServerHello 中提取 32 字节 ServerRandom
6. **TLS 1.3 检查** — strict_mode 下验证后端是否返回 TLS 1.3
7. **双工数据帧处理** — 后端→客户端透传；客户端→后端逐帧读取，HMAC 匹配后剥离 HMAC 头
8. **返回首帧** — 认证成功的客户端首帧（去除 HMAC 包装）返回给上层

## 配置项

```json
{
  "version": 3,
  "users": [
    { "name": "alice", "password": "secret123" },
    { "name": "bob",   "password": "pass456" }
  ],
  "handshake_dest": "www.microsoft.com:443",
  "server_names": ["www.microsoft.com"],
  "strict_mode": true,
  "handshake_timeout_ms": 5000
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | int | 协议版本：2 或 3（默认 3） |
| `password` | string | v2 兼容密码 |
| `users` | array | v3 用户列表（name + password） |
| `handshake_dest` | string | 后端 TLS 服务器 host:port |
| `server_names` | array | SNI 白名单 |
| `strict_mode` | bool | 强制 TLS 1.3（默认 true） |
| `handshake_timeout_ms` | uint32 | 握手超时毫秒数（默认 5000） |

## 协议常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `tls_header_size` | 5 | TLS 记录头长度 |
| `tls_random_size` | 32 | TLS Random 长度 |
| `tls_session_id_size` | 32 | ShadowTLS 要求的 SessionID 长度 |
| `hmac_size` | 4 | HMAC 标签长度 |

## Detection 特征

- **Tier 0 sniff**：检测非标准 SessionID 长度（`session_id_non_standard`）— 零成本快速检测
- **Tier 1 verify**：需要 SessionID 长度 == 32 且 ClientHello >= 76 字节，进行 HMAC 验证
- HMAC 验证得分 900，独占（solo_flag = 0xFFFF）
- 不匹配得分 50，不独占

## 与 Reality 的区别

| 特性 | ShadowTLS | Reality |
|------|-----------|---------|
| 认证位置 | ClientHello SessionID | ClientHello 后扩展 |
| 证书伪造 | 不需要（使用真实后端证书） | 需要伪造证书 |
| 证书验证 | 真实证书链 | 自签证书 |
| 后端连接 | 需要连接后端 TLS 服务器 | 不需要 |
| 加密层 | 标准 TLS | TLS + Reality 扩展 |

## 相关源码

头文件：`include/prism/stealth/shadowtls/` — scheme.hpp, auth.hpp, handshake.hpp, config.hpp, constants.hpp
源文件：`src/prism/stealth/shadowtls/` — scheme.cpp, auth.cpp, handshake.cpp
