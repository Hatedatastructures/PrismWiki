---
title: "decrypt — ECH 解密接口"
source: "include/prism/stealth/ech/decrypt.hpp"
module: "stealth"
submodule: "ech"
type: api
tags: [stealth, ech, decrypt, hpke, 解密, aead]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/anytls/scheme|anytls/scheme]]"
  - "[[protocol/tls/types|tls/types]]"
  - "[[fault/code|code]]"
  - "[[memory/container|container]]"
  - "[[ref/protocol/tls-extensions|TLS 扩展]]"
  - "[[ref/crypto/aes-gcm|AES-GCM]]"
---

# decrypt.hpp

> 源码: `include/prism/stealth/ech/decrypt.hpp`
> 实现: `src/prism/stealth/ech/decrypt.cpp`
> 模块: [[stealth|stealth]] > [[stealth/ech|ech]]

## 概述

ECH (Encrypted Client Hello) 解密接口。使用 HPKE (Hybrid Public Key Encryption) 解密 ECH outer payload，获取 inner ClientHello。ECH 是 TLS 1.3 扩展，允许客户端加密 ClientHello 中的 SNI 等敏感字段。

**当前状态**：框架已实现，HPKE 解密逻辑待完善。当前仅验证 ECH outer 格式（version 检查），实际解密返回 `not_supported`。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 依赖 | [[protocol/tls/types|types]] | 使用 client_hello_features 结构 |
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[trace|trace]] | 日志记录 |
| 被依赖 | [[stealth/anytls/scheme|anytls/scheme]] | verify() 中调用 decrypt_ech_payload |
| 被依赖 | [[stealth/executor|executor]] | 执行器调用解密函数 |

## 命名空间

`psm::stealth::ech`

---

## 结构体: decrypt_result

> 源码: `include/prism/stealth/ech/decrypt.hpp:20`

### 概述

ECH 解密结果。包含解密状态、解密后的 inner 特征、原始字节和错误码。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| valid | `bool` | 解密是否成功 | `false` |
| inner_features | `protocol::tls::client_hello_features` | 解密后的 inner ClientHello 特征 | 默认构造 |
| inner_client_hello | `memory::vector<std::byte>` | 解密后的 inner ClientHello 原始字节 | 空 |
| error | `fault::code` | 错误码 | `fault::code::success` |

---

## 函数: decrypt_ech_payload()

> 源码: `include/prism/stealth/ech/decrypt.hpp:45`
> 实现: `src/prism/stealth/ech/decrypt.cpp:14`

### 功能说明

解密 ECH outer payload。使用 HPKE 解密流程获取 inner ClientHello。

**当前实现**：
1. 验证 ECH outer 最小长度（7 字节：version(2) + config_id(1) + enc_len(2) + payload_len(2)）
2. 检查 version 是否为 TLS 1.3 (`0xfe0d`)
3. HPKE 解密逻辑为 TODO 桩，返回 `fault::code::not_supported`

**计划实现的 HPKE 解密流程**：
1. 解析 ECH outer 结构（version, config_id, enc, payload）
2. 提取 config_id，匹配配置中的 ECH 密钥
3. 提取 enc（KEM encapsulated key）
4. 使用 ech_key 解码私钥
5. 执行 HPKE SetupBaseS：使用私钥和 enc 计算 shared secret
6. 使用 AEAD 解密 payload
7. 解析 inner ClientHello，提取真实 SNI

### 签名

```cpp
[[nodiscard]] auto decrypt_ech_payload(
    std::span<const std::byte> outer_payload,
    std::string_view ech_key) -> decrypt_result;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| outer_payload | `std::span<const std::byte>` | ECH outer payload（从 ClientHello 扩展提取） |
| ech_key | `std::string_view` | ECH 配置密钥（base64 编码） |

### 返回值

`decrypt_result` — 解密结果：
- `valid=true` — 解密成功（当前未实现）
- `valid=false, error=ech_payload_invalid` — payload 长度不足
- `valid=false, error=ech_version_mismatch` — version 不匹配
- `valid=false, error=not_supported` — HPKE 解密未实现

### 错误处理

| 错误场景 | error 码 |
|----------|----------|
| payload 长度 < 7 字节 | `fault::code::ech_payload_invalid` |
| version != `0xfe0d` | `fault::code::ech_version_mismatch` |
| HPKE 解密未实现 | `fault::code::not_supported` |

### 调用（向下）

- 无外部库调用（当前为 TODO 桩）
- 计划调用：HPKE SetupBaseS、AEAD Open

### 被调用（向上）

- [[stealth/anytls/scheme|anytls::scheme::verify]] — AnyTLS Tier 1 检测中尝试 ECH 解密
- [[stealth/executor|executor]] — 执行器调用解密函数

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-GCM 认证加密（HPKE AEAD 算法）
- [[ref/protocol/tls-extensions|TLS 扩展]] — ECH 扩展格式
- [[ref/crypto/x25519|x25519]] — HPKE KEM 密钥交换

### ECH outer 格式

```
ECHOuter (from ClientHello extension):
  version:      2 bytes (0xfe 0x0d for TLS 1.3)
  config_id:    1 byte
  enc:          variable (KEM encapsulated key)
  payload:      variable (encrypted inner ClientHello)
```

### 注意事项

- ECH 扩展在 ClientHello 中的位置不固定，需要通过扩展类型 ID 定位
- 需要完整的 ECH outer payload（不是整个 ClientHello）
- 解密失败时返回错误码，调用方应 fallback 到普通 TLS 处理
- HPKE 使用 X25519 作为 KEM，AES-256-GCM 作为 AEAD
