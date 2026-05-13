---
title: ECH (Encrypted Client Hello)
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags:
  - stealth
  - tls
  - ech
  - hpke
related:
  - "[[anytls]]"
  - "[[shadowtls]]"
  - "[[restls]]"
  - "[[trusttunnel]]"
---

# ECH (Encrypted Client Hello)

ECH 是 TLS 扩展，用于加密 ClientHello 中的 SNI 等敏感字段，防止中间人（包括 DPI 设备）
看到目标域名。在 Prism 中，ECH 作为**叠加层**实现，可叠加在任意 TLS 伪装协议上。

协议参考: https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni

## 工作原理

ECH 使用 **HPKE (Hybrid Public Key Encryption)** 加密 ClientHello：

```
ClientHello
├── outer ClientHello（公开可见）
│   ├── SNI: 公开域名（如 cdn.example.com）
│   └── ECH extension
│       ├── version: 0xfe0d (TLS 1.3)
│       ├── config_id: 1 byte
│       ├── enc: KEM encapsulated key
│       └── payload: 加密的 inner ClientHello
└── inner ClientHello（加密隐藏）
    ├── SNI: 真实目标域名
    └── 其他扩展
```

观察者只能看到 outer ClientHello 中的公开 SNI，真实 SNI 被 ECH 加密保护。

## 解密流程

Prism 服务端的 ECH 解密流程（`decrypt.cpp`）：

1. **格式检查** — ECH outer payload 最小长度 7 字节
2. **版本验证** — 检查 version 字段为 `0xfe0d`（TLS 1.3 ECH）
3. **解析结构**：
   - `version`: 2 bytes (`0xfe 0x0d`)
   - `config_id`: 1 byte（密钥标识）
   - `enc`: variable（KEM 封装的临时公钥）
   - `payload`: variable（加密的 inner ClientHello）
4. **HPKE 解密**：
   - 使用 ECH 私钥和 `enc` 计算 shared secret（HPKE SetupBaseS）
   - 使用 AEAD 解密 `payload`，获取 inner ClientHello
5. **解析 inner ClientHello** — 提取真实 SNI

## ECH 在各方案中的使用

ECH 可以叠加在以下方案上：

| 方案 | ECH 支持 | 说明 |
|------|----------|------|
| [[anytls]] | 支持 | `ech_key` 配置项，Tier 1 verify 中解密 |
| [[trusttunnel]] | 可扩展 | 当前未直接集成 |
| [[restls]] | 可扩展 | 当前未直接集成 |
| [[shadowtls]] | 不需要 | 认证在 SessionID 中，与 SNI 无关 |

### AnyTLS 中的 ECH

AnyTLS 方案在 Tier 1 verify 阶段检查 ECH：

```
1. 检查 ClientHello 是否有 ECH 扩展（has_ech 特征位）
2. 有 ECH 且配置了 ech_key → 得分 300
3. 解密 ECH payload，获取 inner SNI
4. 检查 inner SNI 是否匹配 server_names
```

## 配置项

```json
{
  "ech_key": "base64-encoded-ech-private-key",
  "public_name": "cdn.example.com"
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `ech_key` | string | 是 | ECH 私钥（base64 编码） |
| `public_name` | string | 否 | 公开的伪装域名（outer SNI） |

## 实现状态

当前实现状态：**框架已实现，HPKE 解密逻辑待完善**。

已完成：
- ECH outer payload 格式解析
- version 字段验证（`0xfe0d`）
- 错误码定义（`ech_payload_invalid`、`ech_version_mismatch`）
- 配置结构和 glaze 序列化

待实现：
- HPKE SetupBaseS 解密（需要 KEM + KDF + AEAD 实现）
- config_id 匹配和密钥管理
- inner ClientHello 解析
- 多密钥轮换支持

## 错误码

| 错误码 | 说明 |
|--------|------|
| `ech_payload_invalid` | ECH payload 长度不足（< 7 字节） |
| `ech_version_mismatch` | ECH version 不是 `0xfe0d` |
| `not_supported` | HPKE 解密尚未实现 |

## 相关源码

- `include/prism/stealth/ech/config.hpp` — ECH 配置结构
- `include/prism/stealth/ech/decrypt.hpp` — 解密接口和结果结构
- `src/prism/stealth/ech/decrypt.cpp` — 解密实现
