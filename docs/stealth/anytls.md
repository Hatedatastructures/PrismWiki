---
title: AnyTLS
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags:
  - stealth
  - tls
  - ech
  - tier2
related:
  - "[[stealth/ech]]"
  - "[[stealth/shadowtls]]"
  - "[[stealth/restls]]"
  - "[[stealth/trusttunnel]]"
---

# AnyTLS

AnyTLS 是 Prism stealth 模块中的 **Tier 2** 伪装方案，使用标准 TLS 证书，通过应用层认证
实现代理功能。与 ShadowTLS/Reality 不同，AnyTLS 不需要连接后端 TLS 服务器，
而是使用自己的证书完成 TLS 握手。可选叠加 ECH 加密 ClientHello SNI。

协议参考: https://github.com/anytls/anytls

## 工作原理

AnyTLS 采用**自有证书 + 应用层认证**架构：

1. 使用配置的 TLS 证书执行标准 TLS 握手
2. TLS 握手完成后，读取 TLS 应用数据（客户端首帧）
3. 解析 AnyTLS 认证帧格式：`[password_length:2][password:N][padding:variable]`
4. 验证用户身份（用户名/密码）
5. 认证成功后检测内层协议（HTTP、SOCKS5 等）

优势：不需要连接后端真实 TLS 服务器，延迟更低。
劣势：使用自有证书，可能被证书指纹识别。

## ECH 加密伪装

AnyTLS 支持叠加 **ECH (Encrypted Client Hello)** 来加密 ClientHello 中的 SNI：

- 配置 `ech_key` 后，客户端可在 ClientHello 中嵌入 ECH 扩展
- 服务端在 Tier 1 verify 阶段使用 ECH 密钥解密
- 解密后获取 inner ClientHello 中的真实 SNI
- ECH 使得 DPI 无法看到目标域名

ECH 检测流程：
1. Tier 0：无特殊特征
2. Tier 1 verify：检查 ClientHello 是否有 ECH 扩展（`has_ech`）
   - 有 ECH 且配置了 ech_key → 得分 300
   - 无 ECH 或未配置密钥 → 得分 0
3. Tier 2 guess：返回基础分 100，依赖 SNI 匹配

## 配置项

```json
{
  "server_names": ["proxy.example.com"],
  "certificate": "/path/to/cert.pem",
  "private_key": "/path/to/key.pem",
  "users": [
    { "username": "alice", "password": "secret123" }
  ],
  "ech_key": "base64-encoded-ech-key",
  "padding_scheme": "random",
  "handshake_timeout_ms": 5000,
  "idle_session_timeout_ms": 30000
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `server_names` | array | 是 | SNI 白名单 |
| `certificate` | string | 是 | TLS 证书文件路径（PEM） |
| `private_key` | string | 是 | TLS 私钥文件路径（PEM） |
| `users` | array | 是 | 用户列表（username + password） |
| `ech_key` | string | 否 | ECH 密钥（base64），叠加 ECH 加密 |
| `padding_scheme` | string | 否 | Padding 方案，隐藏流量特征 |
| `handshake_timeout_ms` | uint32 | 否 | 握手超时（默认 5000ms） |
| `idle_session_timeout_ms` | uint32 | 否 | 空闲会话超时（默认 30000ms） |

## 实现状态

当前实现状态：**基础框架已实现，认证逻辑待完善**。

已完成：
- 方案注册和生命周期管理
- SNI 白名单匹配
- ECH 扩展检测（Tier 1 verify）
- Tier 2 guess 接口

待实现：
- 完整的 AnyTLS 认证帧解析
- 用户身份验证
- ECH 解密（依赖 [[stealth/ech]] 模块的 HPKE 实现）
- 内层协议检测
- Padding 方案引擎

## 相关源码

- `include/prism/stealth/anytls/scheme.hpp` — 方案类定义
- `include/prism/stealth/anytls/config.hpp` — 配置结构
- `src/prism/stealth/anytls/scheme.cpp` — 方案实现
