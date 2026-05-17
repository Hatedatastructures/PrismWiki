---
title: TLS 问题排查指南
source:
  - I:/code/Prism/include/prism/stealth/
  - I:/code/Prism/src/prism/stealth/
  - I:/code/Prism/include/prism/crypto/
module: debugging
type: reference
tags: [tls, certificate, handshake, reality, shadowtls, ssl, troubleshooting]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[core/stealth/overview|stealth]]"
  - "[[dev/tls]]"
  - "[[crypto/aead]]"
---

# TLS 问题排查指南

TLS 问题包括握手失败、证书验证错误、加密配置错误等。本文档提供 TLS 层面的排障方法。

## 概述

### TLS 问题分类

| 问题类型 | 典型症状 | 常见原因 |
|----------|----------|----------|
| 握手失败 | 连接无法建立 | 版本/密码套件不兼容 |
| 证书错误 | 验证失败 | 证书过期/域名不匹配 |
| Reality 失败 | 认证不通过 | 密钥配置错误 |
| ShadowTLS 失败 | 握手中断 | 密码不匹配 |
| 加密错误 | AEAD 解密失败 | 密钥/nonce 错误 |

### TLS 握手流程

```
客户端                                服务器
  │                                     │
  │──── ClientHello ─────────────────→│
  │     · TLS Version                  │
  │     · Cipher Suites                │
  │     · SNI                          │
  │     · Key Share                    │
  │                                     │
  │←─── ServerHello ───────────────────│
  │     · Selected Version             │
  │     · Selected Cipher              │
  │                                     │
  │←─── Certificate ───────────────────│
  │←─── CertificateVerify ────────────│
  │←─── Finished ──────────────────────│
  │                                     │
  │──── Finished ──────────────────────→│
  │                                     │
  │     ====== 加密通道建立 ======       │
```

问题可能发生在任何阶段。

---

## 详解

### TLS 握手失败排查

#### 症状特征

```
日志示例:
[2026-05-17 10:30:45.123] [prism] [error] TLS handshake failed: reason=version_mismatch
[2026-05-17 10:30:45.456] [prism] [error] SSL error: ssl_error_no_cipher_overlap
[2026-05-17 10:30:45.789] [prism] [error] TLS alert: handshake_failure
```

#### 排查流程

```
握手失败
    |
    v
+------------------------+
| 1. 检查 TLS 版本        |
| - 客户端版本支持        |
| - 服务器配置           |
+------------------------+
    |
    v
+------------------------+
| 2. 检查密码套件         |
| - 共同支持的套件       |
| - 优先级配置           |
+------------------------+
    |
    v
+------------------------+
| 3. 检查 SNI             |
| - SNI 是否发送         |
| - 域名是否匹配         |
+------------------------+
    |
    v
+------------------------+
| 4. 检查证书             |
| - 证书有效性           |
| - 域名匹配             |
+------------------------+
```

#### 诊断命令

```bash
# 测试 TLS 连接
openssl s_client -connect server:443 -tls1_3

# 查看证书详情
openssl s_client -connect server:443 -showcerts

# 查看支持的密码套件
openssl s_client -connect server:443 -cipher 'ALL'
```

### 证书验证失败排查

#### 症状特征

```
日志示例:
[2026-05-17 10:35:00.123] [prism] [error] Certificate verification failed: reason=expired
[2026-05-17 10:35:00.456] [prism] [error] Certificate invalid: domain_mismatch
[2026-05-17 10:35:00.789] [prism] [error] Certificate chain incomplete
```

#### 证书验证要点

| 检查项 | 失败原因 | 解决方案 |
|--------|----------|----------|
| 有效期 | 证书过期 | 更新证书 |
| 域名匹配 | SAN 不含域名 | 重新签发 |
| 证书链 | 缺少中间证书 | 使用 fullchain |
| 根信任 | 根 CA 不在信任列表 | 更新系统 CA |
| 吊销状态 | OCSP/CRL 失败 | 检查吊销服务 |

#### 证书检查命令

```bash
# 查看证书有效期
openssl x509 -in cert.pem -text -noout | grep -A 2 Validity

# 查看证书域名
openssl x509 -in cert.pem -text -noout | grep -A 1 "Subject Alternative Name"

# 检查证书链
openssl verify -CAfile ca.pem cert.pem
```

### Reality TLS 问题排查

#### 症状特征

```
日志示例:
[2026-05-17 10:40:00.123] [prism] [error] Reality authentication failed: short_id_mismatch
[2026-05-17 10:40:00.456] [prism] [error] Reality handshake error: key_share_invalid
[2026-05-17 10:40:00.789] [prism] [error] Reality probe rejected: no_short_id
```

#### Reality 配置检查

| 配置项 | 检查要点 |
|--------|----------|
| dest | 伪装目标可达性 |
| server_names | SNI 匹配 |
| private_key | Base64 正确性 |
| short_ids | 客户端服务器一致 |

#### Reality 排障流程

```
Reality 失败
    |
    v
+------------------------+
| 1. 检查伪装目标         |
| curl -v https://dest   |
+------------------------+
    |
    v
+------------------------+
| 2. 检查密钥配置         |
| - 公钥私钥匹配         |
| - Base64 编码正确      |
+------------------------+
    |
    v
+------------------------+
| 3. 检查 Short ID        |
| - 客户端配置正确       |
| - 格式正确             |
+------------------------+
    |
    v
+------------------------+
| 4. 检查 SNI 配置        |
| - server_names 匹配    |
+------------------------+
```

#### 密钥生成

```bash
# Reality 密钥生成
./scripts/GenRealityKeys.sh

# 输出
Private Key: ABCD...（服务器配置）
Public Key: EFGH...（客户端配置）
```

### ShadowTLS 问题排查

#### 症状特征

```
日志示例:
[2026-05-17 10:45:00.123] [prism] [error] ShadowTLS handshake failed: password_mismatch
[2026-05-17 10:45:00.456] [prism] [error] ShadowTLS verification failed: hmac_invalid
[2026-05-17 10:45:00.789] [prism] [error] ShadowTLS timeout: handshake_timeout_ms=5000
```

#### ShadowTLS 配置检查

| 配置项 | 检查要点 |
|--------|----------|
| version | v2/v3 版本匹配 |
| password | 客户端服务器一致 |
| handshake_dest | 伪装目标可达 |
| server_names | SNI 匹配 |
| handshake_timeout_ms | 超时是否合理 |

#### ShadowTLS 排障流程

```
ShadowTLS 失败
    |
    v
+------------------------+
| 1. 检查版本匹配         |
| - 客户端版本           |
| - 服务器 version       |
+------------------------+
    |
    v
+------------------------+
| 2. 检查密码配置         |
| - 两端密码一致         |
+------------------------+
    |
    v
+------------------------+
| 3. 检查伪装目标         |
| curl -v https://dest   |
+------------------------+
    |
    v
+------------------------+
| 4. 检查超时设置         |
| - 增加超时时间         |
+------------------------+
```

### AEAD 加密错误排查

#### 症状特征

```
日志示例:
[2026-05-17 10:50:00.123] [prism] [error] AEAD decryption failed: authentication_failed
[2026-05-17 10:50:00.456] [prism] [error] Invalid ciphertext: tag_mismatch
[2026-05-17 10:50:00.789] [prism] [error] Crypto error: nonce_overflow
```

#### AEAD 错误原因

| 错误类型 | 原因 | 解决方案 |
|----------|------|----------|
| 认证失败 | 密钥错误/数据损坏 | 检查密钥配置 |
| nonce 溢出 | nonce 重复使用 | 重置 nonce |
| tag 不匹配 | 数据被篡改 | 重新传输 |
| 密钥不匹配 | 两端密钥不同 | 统一配置 |

---

## 日志分析示例

### 正常 TLS 握手

```
[10:30:00.100] [debug] TLS handshake started: version=TLS1.3
[10:30:00.150] [info] TLS handshake completed: cipher=AES-128-GCM
[10:30:00.200] [info] Session encrypted: tls_established=true
```

### TLS 握手失败

```
[10:30:00.100] [debug] TLS handshake started
[10:30:00.150] [error] SSL error: no_cipher_overlap
[10:30:00.200] [error] TLS handshake failed
[10:30:00.250] [warn] Session cleanup: reason=ssl_error
```

### Reality 成功

```
[10:30:00.100] [debug] Reality ClientHello received: short_id=abc123
[10:30:00.150] [info] Reality authentication success
[10:30:00.200] [info] Reality session established
```

### Reality 失败

```
[10:30:00.100] [debug] Reality ClientHello received
[10:30:00.150] [warn] Short ID not found: received=xyz, expected=abc
[10:30:00.200] [error] Reality authentication failed
```

---

## 使用示例

### TLS 连接测试

```bash
#!/bin/bash
# test_tls.sh

SERVER=$1
PORT=${2:-443}

echo "Testing TLS connection to $SERVER:$PORT"

# TLS 1.3 测试
openssl s_client -connect $SERVER:$PORT -tls1_3 2>&1 | head -20

# 证书信息
openssl s_client -connect $SERVER:$PORT -showcerts 2>&1 | openssl x509 -text -noout | head -30

# 支持的密码套件
nmap --script ssl-enum-ciphers -p $PORT $SERVER
```

### 证书检查

```bash
#!/bin/bash
# check_cert.sh

CERT=$1

echo "Certificate details:"
openssl x509 -in $CERT -text -noout | grep -E "Subject|Issuer|Validity|DNS"

echo ""
echo "Expiration check:"
EXPIRY=$(openssl x509 -in $CERT -enddate -noout | cut -d= -f2)
echo "Expires: $EXPIRY"

# 检查是否即将过期
EXPIRY_SEC=$(date -d "$EXPIRY" +%s)
NOW_SEC=$(date +%s)
DAYS_LEFT=$(( ($EXPIRY_SEC - $NOW_SEC) / 86400 ))

if [ $DAYS_LEFT -lt 30 ]; then
    echo "WARNING: Certificate expires in $DAYS_LEFT days"
fi
```

---

## 最佳实践

### TLS 配置建议

| 配置项 | 建议 | 原因 |
|--------|------|------|
| TLS 版本 | 仅 TLS 1.3 | 安全性最高 |
| 密码套件 | AES-128-GCM/ChaCha20 | 平衡性能和安全 |
| 证书来源 | Let's Encrypt 或 Reality | 避免 self-signed |

### Reality 配置建议

| 配置项 | 建议 | 原因 |
|--------|------|------|
| dest | 高流量网站 | 减少异常 |
| server_names | 与 dest 匹配 | 正常握手 |
| short_ids | 多个 ID | 客户端标识 |

### 证书管理建议

| 建议 | 操作 |
|------|------|
| 自动续期 | Let's Encrypt 自动续期 |
| 监控过期 | 设置告警（30天前） |
| 备份证书 | 保存私钥和证书 |

---

## 常见问题

### Q1: TLS 握手失败 "no cipher overlap"？

**A**: 
1. 检查两端支持的密码套件
2. 确认 TLS 版本匹配
3. 检查 BoringSSL/OpenSSL 版本

### Q2: 证书验证失败怎么办？

**A**: 
1. 检查证书有效期
2. 确认域名匹配
3. 检查证书链完整性

### Q3: Reality Short ID 不匹配？

**A**: 
1. 确认客户端 short-id 配置
2. 检查服务器 short_ids 列表
3. 确认格式正确

### Q4: ShadowTLS 密码错误？

**A**: 
1. 确认两端密码一致
2. 检查版本匹配（v2/v3）
3. 确认 handshake_dest 可达

---

## 排障指南

### 问题：TLS 握手超时

**症状**: TLS 握手长时间不完成

**排查步骤**:

1. 检查网络连通性
   ```bash
   ping server
   telnet server 443
   ```

2. 检查防火墙
   ```bash
   iptables -L -n
   ```

3. 检查证书配置
   ```json
   "agent": {
       "certificate": {
           "key": "/path/to/key.pem",
           "cert": "/path/to/cert.pem"
       }
   }
   ```

---

### 问题：Reality 认证失败

**症状**: Reality 客户端连接失败

**排查步骤**:

1. 检查密钥配置
   ```bash
   # 重新生成密钥
   ./scripts/GenRealityKeys.sh
   ```

2. 检查伪装目标
   ```bash
   curl -v https://www.microsoft.com
   ```

3. 检查配置一致性
   ```json
   // 服务器
   "stealth": {
       "reality": {
           "private_key": "...",
           "short_ids": ["abc123"]
       }
   }
   
   // 客户端
   "reality-opts": {
       "public-key": "...",  // 对应私钥
       "short-id": "abc123"  // 匹配
   }
   ```

---

### 问题：证书即将过期

**症状**: 证书有效期即将到期

**排查步骤**:

1. 检查过期时间
   ```bash
   openssl x509 -in cert.pem -enddate -noout
   ```

2. Let's Encrypt 续期
   ```bash
   certbot renew
   ```

3. Reality 方案无需续期
   - Reality 不需要证书管理
   - 使用目标网站证书

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[stealth]] — Stealth TLS伪装模块
- [[dev/tls]] — TLS 协议基础
- [[crypto/aead]] — AEAD 加密实现
- [[stealth/reality]] — Reality TLS详解
- [[stealth/shadowtls]] — ShadowTLS详解