---
title: TLS 协议实践指南
source:
  - I:/code/Prism/include/prism/stealth/
  - I:/code/Prism/src/prism/stealth/
  - I:/code/Prism/include/prism/crypto/
module: stealth
type: reference
tags: [tls, encryption, security, certificate, handshake, fingerprint, reality, shadowtls]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[modules/stealth]]"
  - "[[modules/crypto]]"
  - "[[dev/tcp]]"
  - "[[dev/gfw]]"
  - "[[configuration]]"
---

# TLS 协议实践指南

TLS（Transport Layer Security）是互联网加密通信的基石。在代理领域，TLS 不仅保护数据传输，还是对抗审查的关键伪装手段。

## 概述

### TLS 协议演进

TLS 协议的发展历程：

| 版本 | 发布年份 | 安全状态 | 主要改进 |
|------|----------|----------|----------|
| SSL 1.0 | 1994 | 未发布 | 原始设计 |
| SSL 2.0 | 1995 | 已弃用 | 有严重漏洞 |
| SSL 3.0 | 1996 | 已弃用 | POODLE 漏洞 |
| TLS 1.0 | 1999 | 已弃用 | 协议升级 |
| TLS 1.1 | 2006 | 已弃用 | 安全增强 |
| TLS 1.2 | 2008 | 广泛使用 | AEAD 支持 |
| TLS 1.3 | 2018 | 推荐使用 | 大幅简化 |

TLS 1.3 相比 TLS 1.2 的关键改进：
- 握手从 2-RTT 减少到 1-RTT
- 移除不安全密码套件（RSA 密钥交换、CBC 等）
- 加密更多握手信息（证书等）
- 支持 0-RTT 早期数据

### TLS 在代理中的角色

TLS 在代理系统中有双重作用：

```
        +------------------------+
        |   TLS 的双重角色        |
        +------------------------+
                 │
       +---------+---------+
       │                   │
+----------+          +----------+
| 数据保护 |          | 流量伪装 |
+----------+          +----------+
       │                   │
       ▼                   ▼
+----------+          +----------+
| 加密传输 |          | 抗审查   |
| 身份验证 |          | 降低指纹 |
| 完整性   |          | 探测抵抗 |
+----------+          +----------+
```

数据保护功能：
- 加密：防止流量内容被窥探
- 身份验证：确保连接到正确的服务器
- 完整性：防止数据被篡改

流量伪装功能：
- TLS 指纹控制：让代理流量看起来像正常 HTTPS
- 探测抵抗：主动探测时返回正常网站响应
- 协议混淆：将代理数据包装在 TLS 内

### TLS 1.3 握手流程

TLS 1.3 将握手简化为 1-RTT：

```
客户端                                服务器
  │                                     │
  │──── ClientHello ─────────────────→│  第一轮
  │     · supported_versions           │
  │     · supported_groups             │
  │     · key_share (ECDHE 公钥)       │
  │     · signature_algorithms         │
  │     · cipher_suites                │
  │                                     │
  │←─── ServerHello ───────────────────│  第二轮
  │     · selected_version             │
  │     · selected_group               │
  │     · key_share (ECDHE 公钥)       │
  │     · selected_cipher              │
  │                                     │
  │←─── {EncryptedExtensions} ─────────│  加密扩展
  │←─── {Certificate*} ────────────────│  服务器证书
  │←─── {CertificateVerify*} ──────────│  证书签名验证
  │←─── {Finished} ────────────────────│  握手完成
  │                                     │
  │──── {Finished} ──────────────────→│  客户端确认
  │                                     │
  │     ====== 应用数据加密传输 ======   │
```

关键点：
- ClientHello 包含密钥共享（key_share），服务器可立即计算共享密钥
- ServerHello 后所有消息已加密
- Certificate 等敏感信息不再明文传输

### 0-RTT 早期数据

TLS 1.3 支持恢复会话时发送早期数据：

```
客户端                                服务器
  │──── ClientHello + Early Data ───→│  0-RTT
  │     (之前的会话密钥)               │
  │                                     │
  │←─── ServerHello + Finished ────────│
  │                                     │
  │──── Finished + 应用数据 ─────────→│  1-RTT
```

风险：
- 早期数据可能被重放攻击
- 不适用于需要唯一性的请求
- 代理协议中需谨慎使用

---

## 详解

### TLS 密码套件

密码套件定义 TLS 连接的加密参数。

#### TLS 1.3 密码套件

TLS 1.3 简化了密码套件，只保留 AEAD 模式：

| 密码套件 | 名称 | 密钥交换 | 认证 | 加密 | MAC |
|----------|------|----------|------|------|-----|
| TLS_AES_128_GCM_SHA256 | AEAD | ECDHE | RSA/ECDSA | AES-128-GCM | 内置 |
| TLS_AES_256_GCM_SHA384 | AEAD | ECDHE | RSA/ECDSA | AES-256-GCM | 内置 |
| TLS_CHACHA20_POLY1305_SHA256 | AEAD | ECDHE | RSA/ECDSA | ChaCha20-Poly1305 | 内置 |

TLS 1.3 移除的不安全密码套件：
- RSA 密钥交换（无前向保密）
- CBC 模式加密（BEAST、Lucky13 等攻击）
- RC4、3DES 等弱加密算法
- SHA-1 哈希（已被攻破）

#### 密钥交换流程

TLS 1.3 使用 ECDHE（椭圆曲线 Diffie-Hellman）密钥交换：

```
客户端                            服务器
  │ 生成临时密钥对                 │
  │   private_key_c              │
  │   public_key_c               │
  │                               │
  │─── ClientHello ─────────────→│
  │    key_share: public_key_c   │
  │                               │
  │                               │ 生成临时密钥对
  │                               │   private_key_s
  │                               │   public_key_s
  │                               │
  │←── ServerHello ───────────────│
  │    key_share: public_key_s   │
  │                               │
  │                               │
  │ 计算共享密钥:                  │ 计算共享密钥:
  │ shared_secret =               │ shared_secret =
  │   ECDH(public_key_s,          │   ECDH(public_key_c,
  │        private_key_c)         │        private_key_s)
  │                               │
  │←─── 派生会话密钥 ──────────────│
```

前向保密（Forward Secrecy）：
- 每次连接使用新的临时密钥对
- 即使服务器长期私钥泄露，历史连接仍安全
- TLS 1.3 强制前向保密

### 证书链与 PKI

TLS 依赖 PKI（公钥基础设施）验证服务器身份。

#### 证书链结构

```
根 CA（Root Certificate Authority）
  │
  │ 签名
  ▼
中间 CA（Intermediate CA）
  │
  │ 签名
  ▼
服务器证书（Server Certificate）
  · 主题：*.example.com
  · 公钥：RSA/ECDSA
  · 有效期：1年
  · 签发者：中间 CA
```

#### 证书验证流程

```
客户端收到证书链
    │
    ▼
+------------------------+
|  1. 证书链完整性        |
|  检查每级签名是否有效   |
+------------------------+
    │
    ▼
+------------------------+
|  2. 根 CA 可信性        |
|  根证书在系统信任列表   |
+------------------------+
    │
    ▼
+------------------------+
|  3. 域名匹配            |
| 证书域名与请求域名匹配  |
| (CN 或 SAN 扩展)       |
+------------------------+
    │
    ▼
+------------------------+
|  4. 有效期检查          |
| 当前时间在证书有效期内  |
+------------------------+
    │
    ▼
+------------------------+
|  5. 吊销状态检查        |
| CRL 或 OCSP 检查       |
| 证书未被吊销           |
+------------------------+
    │
    ▼
证书验证成功
```

####证书扩展字段

关键扩展：

| 扩展 | OID | 作用 |
|------|-----|------|
| Subject Alternative Name (SAN) | 2.5.29.17 | 多域名支持（代替 CN） |
| Basic Constraints | 2.5.29.19 | 标识 CA 证书 |
| Key Usage | 2.5.29.15 | 密钥用途限制 |
| Extended Key Usage | 2.5.29.37 | 应用用途（如 serverAuth） |
| Authority Information Access | 1.3.6.1.5.5.7.1.1 | OCSP/CRL 位置 |

### TLS 指纹

TLS 指纹是识别客户端/服务器的重要手段。

#### JA3 指纹

JA3 从 ClientHello 提取参数生成指纹：

```
JA3 = MD5(TLSVersion + Ciphers + Extensions + EllipticCurves + PointFormats)
```

提取字段：
- TLSVersion：客户端支持的 TLS 版本
- Ciphers：密码套件列表
- Extensions：扩展列表
- EllipticCurves：支持的椭圆曲线
- PointFormats：点格式

示例：
```
Chrome JA3:
771,4865-4866-4867-49195-49199-...,0-5-10-11-...,29-23-24-...,0

Firefox JA3:
771,4865-4866-4867-49195-...,0-5-10-...,29-23-...,0

不同客户端有不同指纹
```

#### JA4 指纹

JA4 是 JA3 的改进版，提供更精细分类：

```
JA4 = {protocol}_{alpn_count}_{cipher_count}_{extension_count}_{first_alpn}_{last_extension}
```

例如：
```
JA4 = t13d1516h2_2da4f0b02c5b: Chrome fingerprint
JA4 = t13d1516h2_8da4f0b02c5b: Firefox fingerprint
```

#### JARM 指纹

JARM 从服务端角度指纹化服务器：

```
发送 10 个特制 ClientHello
    │
    ▼
分析服务器响应差异
    │
    ▼
生成服务器指纹
```

不同 TLS 实现（OpenSSL、BoringSSL、Go 等）的响应不同，可识别服务器类型。

#### 指纹对代理的影响

GFW 使用 TLS 指纹检测代理：

```
客户端发送 ClientHello
    │
    │ SNI: www.microsoft.com
    │ 指纹: Go TLS 库特征
    ▼
GFW 分析
    │
    │ SNI 声称访问 Microsoft
    │ 但 TLS 指纹是 Go TLS
    │ Microsoft 网站应该用 Chrome/Edge 指纹
    ▼
标记为可疑流量
```

绕过方法：
- 使用 uTLS 模拟浏览器指纹
- Reality 技术（握手指纹与真实网站一致）
- ShadowTLS（转发握手到真实 TLS 服务器）

### BoringSSL

Prism 使用 BoringSSL（Google 维护的 OpenSSL 分支）。

#### BoringSSL vs OpenSSL

| 特性 | BoringSSL | OpenSSL |
|------|-----------|---------|
| TLS 1.3 | 原生支持 | 需要 1.1.1+ |
| 代码审计 | Google 持续审计 | 社区维护 |
| API 简洁性 | 更现代 | 历史包袱重 |
| 0-RTT | 完善支持 | 有限支持 |
| 体积 | 更小 | 较大 |
| 兼容性 | OpenSSL API 兼容 | 标准 |

#### Prism 使用 BoringSSL 的原因

1. **安全性**：Google 持续安全审计
2. **TLS 1.3**：完整支持现代 TLS 特性
3. **指纹控制**：可以精细控制 ClientHello 扩展
4. **性能**：优化的加密实现

### Reality TLS

Reality 是革命性的 TLS 伪装技术。

#### 标准 TLS 的探测问题

```
GFW 主动探测
    │
    │ 发送 ClientHello (SNI: proxy.example.com)
    ▼
代理服务器
    │
    │ 返回自签名证书
    │ 证书主题: proxy.example.com
    │ 签发者: 自签名 CA
    ▼
GFW 分析
    │
    │ 证书不是真实 CA签发
    │ proxy.example.com 不是知名网站
    ▼
确认为代理，封锁 IP
```

#### Reality 的解决方案

Reality 使用"偷来的"握手：

```
客户端                            Reality服务器                 真实网站
  │                                  │                            │
  │ ── ClientHello ───────────────→ │                            │
  │    SNI: www.microsoft.com       │                            │
  │    key_share: 客户端公钥        │                            │
  │    特殊标记（Short ID）         │                            │
  │                                  │                            │
  │                                  │ ── ClientHello ──────────→ │
  │                                  │    SNI: www.microsoft.com │
  │                                  │                            │
  │                                  │ ←── ServerHello ────────── │
  │                                  │     真实网站的证书         │
  │                                  │                            │
  │ ←── ServerHello ─────────────── │                            │
  │     使用真实网站证书             │                            │
  │     但用 Reality私钥计算密钥     │                            │
  │                                  │                            │
  │                                  │                            │
  │ 计算共享密钥                     │                            │
  │ 使用预共享公钥                   │                            │
  │                                  │                            │
```

关键机制：
- **Short ID**：标记合法连接，无标记的是探测
- **预共享公钥**：客户端和服务器事先知道彼此公钥
- **伪装目标**：选择真实网站作为 TLS 外观

#### Reality 配置

```json
{
    "stealth": {
        "reality": {
            "dest": "www.microsoft.com:443",       // 伪装目标
            "server_names": ["www.microsoft.com"],// 允许的 SNI
            "private_key": "Base64编码的私钥",     // X25519 私钥
            "short_ids": ["abc123", "def456"]      // Short ID 列表
        }
    }
}
```

密钥生成：

```bash
# 使用 Prism 提供的脚本
./scripts/GenRealityKeys.sh

# 输出
Private Key: ABCD...（填入服务器配置）
Public Key: EFGH...（填入客户端配置）
```

### ShadowTLS

ShadowTLS 是另一种 TLS 伪装方案。

#### ShadowTLS 原理

```
客户端                            ShadowTLS服务器              真实TLS服务器
  │                                  │                            │
  │ ── TLS ClientHello ────────────→ │                            │
  │                                  │ ── TLS ClientHello ──────→ │
  │                                  │                            │
  │ ←── TLS ServerHello ─────────── │ ←── TLS ServerHello ───── │
  │     来自真实服务器               │                            │
  │                                  │                            │
  │ ── 隐藏数据（密码验证） ───────→ │                            │
  │                                  │                            │
  │                                  │ ←── 验证成功 ──────────── │
  │                                  │                            │
  │ ←── 开始代理数据传输 ────────── │                            │
```

ShadowTLS v2/v3 区别：

| 版本 | 特点 | 密码验证方式 |
|------|------|--------------|
| v2 | 简单转发握手 | 隐藏在 TLS 记录中 |
| v3 | 更强安全性 | HMAC 认证 |

#### ShadowTLS 配置

```json
{
    "stealth": {
        "shadowtls": {
            "version": 3,
            "password": "your_password",
            "handshake_dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com"],
            "strict_mode": true,
            "handshake_timeout_ms": 5000
        }
    }
}
```

---

## 使用示例

### Reality 代理配置

服务器端配置：

```json
{
    "agent": {
        "addressable": {
            "host": "0.0.0.0",
            "port": 443
        }
    },
    "stealth": {
        "reality": {
            "dest": "www.microsoft.com:443",
            "server_names": ["www.microsoft.com", "microsoft.com"],
            "private_key": "YOUR_PRIVATE_KEY_HERE",
            "short_ids": ["0123456789abcdef"]
        }
    },
    "protocol": {
        "vless": {
            "enable_udp": true
        }
    }
}
```

客户端配置（Mihomo）：

```yaml
proxies:
  - name: "reality-proxy"
    type: vless
    server: YOUR_SERVER_IP
    port: 443
    uuid: YOUR_UUID
    network: tcp
    tls: true
    udp: true
    servername: www.microsoft.com
    reality-opts:
      public-key: YOUR_PUBLIC_KEY
      short-id: 0123456789abcdef
```

### TLS 证书配置

使用 Let's Encrypt 证书：

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

### TLS 指纹控制

使用 BoringSSL 控制指纹：

```cpp
// 控制 ClientHello 扩展顺序
SSL_CTX* ctx = SSL_CTX_new(TLS_method());

// 设置密码套件顺序（模拟 Chrome）
SSL_CTX_set_cipher_list(ctx, 
    "TLS_AES_128_GCM_SHA256:"
    "TLS_AES_256_GCM_SHA384:"
    "TLS_CHACHA20_POLY1305_SHA256");

// 设置椭圆曲线顺序
SSL_CTX_set1_groups_list(ctx, "x25519:P-256:P-384");

// 设置 ALPN（模拟 HTTP/2）
SSL_CTX_set_alpn_protos(ctx, (unsigned char*)"h2,http/1.1", 11);
```

---

## 最佳实践

### TLS 配置建议

| 参数 | 建议 | 原因 |
|------|------|------|
| TLS 版本 | 仅 TLS 1.3 | 移除不安全版本 |
| 密码套件 | AES-128-GCM/ChaCha20 | 平衡性能和安全 |
| 密钥交换 | ECDHE + x25519 | 前向保密，性能好 |
| 证书 | Let's Encrypt 或 Reality | 避免 self-signed |

### Reality 目标选择

选择伪装目标的原则：

| 原则 | 说明 |
|------|------|
| TLS 指纹相似 | 目标网站使用主流 TLS 库 |
| 支持 TLS 1.3 | 确保握手兼容 |
| 高流量网站 | 流量大，不易被识别异常 |
| 国外知名网站 | 通常不会被GFW主动探测 |

推荐目标：
- www.microsoft.com
- www.apple.com
- www.cloudflare.com
- gateway.icloud.com

### ShadowTLS vs Reality

| 场景 | 推荐方案 |
|------|----------|
| 无域名无证书 | Reality |
| 有域名有证书 | Trojan 或 ShadowTLS |
| 极端审查环境 | Reality + 多层伪装 |

---

## 常见问题

### Q1: Reality 的 Short ID 是什么？

**A**: Short ID 是客户端和服务器之间的"暗号"：
- 客户端在 ClientHello 中嵌入 Short ID
- 服务器检测到 Short ID → 正常代理
- 无 Short ID（探测）→ 转发到真实网站

### Q2: TLS 1.3 的 0-RTT 为什么不推荐？

**A**: 0-RTT 有重放风险：
- 攻击者可截获早期数据并重发
- 适用于幂等操作（如 GET 请求）
- 不适用于需要唯一性的操作（如支付）
- 代理协议中可能重复执行命令

### Q3: 如何检测 TLS 指纹？

**A**: 使用工具检测：
```bash
# JA3 检测
ja3 -c www.example.com:443

# JARM 检测
python jarm.py www.example.com
```

### Q4: BoringSSL 和 OpenSSL 可以共存吗？

**A**: 可以，API 兼容：
- Prism 静态链接 BoringSSL
- 不与系统 OpenSSL 冲突
- 使用 FetchContent 自动获取

### Q5: Reality 密钥需要定期更换吗？

**A**: 不需要：
- Reality 使用临时密钥交换（前向保密）
- 长期密钥仅用于身份验证
- 如怀疑泄露可更换

---

## 排障指南

### 问题：TLS 握手失败

**症状**: 连接建立失败，握手错误

**排查步骤**:

1. 检查 TLS 版本支持
   ```bash
   openssl s_client -connect server:443 -tls1_3
   ```

2. 检查证书有效性
   ```bash
   openssl s_client -connect server:443 -showcerts
   ```

3. 检查密码套件匹配
   - 客户端和服务器是否有共同支持的密码套件

4. 检查 ALPN 协商
   - h2 或 http/1.1 是否匹配

---

### 问题：Reality 探测失败

**症状**: 探测连接返回错误

**排查步骤**:

1. 检查伪装目标可达性
   ```bash
   curl -v https://www.microsoft.com
   ```

2. 检查 Short ID 配置
   - 客户端和服务器是否匹配
   - Short ID 格式是否正确

3. 检查密钥配置
   - 公钥私钥是否匹配
   - Base64 编码是否正确

---

### 问题：证书验证失败

**症状**: 客户端报告证书无效

**排查步骤**:

1. 检查证书链完整性
   - 是否包含中间证书
   - fullchain.pem 是否正确

2. 检查域名匹配
   - 证书域名与请求域名一致
   - SAN 扩展是否包含域名

3. 检查有效期
   ```bash
   openssl x509 -in cert.pem -text -noout | grep -A 2 Validity
   ```

---

### 问题：TLS 性能差

**症状**: TLS 连接延迟高

**排查步骤**:

1. 检查握手时间
   - 使用 Wireshark 分析握手延迟
   - 检查是否有多余往返

2. 检查密钥交换算法
   - x25519 比 P-256 更快
   - 确认使用正确的曲线

3. 启用 session resumption
   ```cpp
   SSL_CTX_set_session_cache_mode(ctx, SSL_SESS_CACHE_SERVER);
   ```

---

## 相关链接

- [[modules/stealth]] — Stealth 模块详解
- [[modules/crypto]] — 加密模块
- [[dev/tcp]] — TCP 协议基础
- [[dev/gfw]] — GFW 封锁原理
- [[configuration]] — TLS 配置参数