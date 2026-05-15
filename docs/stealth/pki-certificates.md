---
title: "PKI 证书体系"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [pki, certificate, tls, security, stealth]
related: ["[[stealth/reality]]", "[[dev/tls]]", "[[crypto]]", "[[docs/stealth/proxy-detection]]"]
---

> 相关：[[stealth/reality]] | [[dev/tls]] | [[crypto]] | [[docs/stealth/proxy-detection]] | [[agent]]

# PKI 证书体系

## 1. 概述

公钥基础设施（PKI, Public Key Infrastructure）是互联网信任体系的基石。在 Prism 的隐蔽代理场景中，证书既是建立 TLS 加密通道的必要组件，也是被审查系统（如 GFW）检测的主要对象之一。理解 PKI 的工作原理对于设计不可检测的代理协议至关重要。

## 2. PKI 基本概念

### 2.1 证书颁发机构（CA）

CA 是 PKI 信任链的根节点，负责签发和管理数字证书。CA 层级通常为：

```
Root CA（根证书，预装在操作系统/浏览器中）
  └─ Intermediate CA（中间 CA，由根 CA 签发）
       └─ Leaf Certificate（终端证书，由中间 CA 签发）
```

根 CA 私钥通常离线存储，中间 CA 用于日常签发，叶子证书绑定到具体域名。这种层级设计限制了根 CA 泄露的影响范围。

### 2.2 X.509 证书结构

TLS 证书遵循 X.509 标准，核心字段包括：

| 字段 | 说明 |
|------|------|
| Version | 证书格式版本（通常 v3） |
| Serial Number | CA 分配的唯一序列号 |
| Signature Algorithm | 签名算法（如 SHA256-RSA, SHA256-ECDSA） |
| Issuer | 签发者 DN（Distinguished Name） |
| Validity | 有效期（Not Before / Not After） |
| Subject | 持有者 DN |
| Subject Public Key | 公钥及其算法 |
| Extensions | SAN, Key Usage, Basic Constraints 等扩展 |
| Signature | CA 对上述内容的数字签名 |

### 2.3 证书链验证

客户端验证证书时执行以下步骤：

1. 检查叶子证书的签名是否由中间 CA 的公钥验证通过
2. 检查中间 CA 的签名是否由根 CA 的公钥验证通过
3. 验证每个证书的有效期
4. 检查证书吊销状态（CRL / OCSP）
5. 验证 Subject Alternative Name（SAN）是否匹配目标域名

## 3. Let's Encrypt 与 ACME 协议

Let's Encrypt 提供免费、自动化的证书签发，通过 ACME 协议验证域名控制权：

- **HTTP-01 验证**：在 `/.well-known/acme-challenge/` 下放置验证文件
- **DNS-01 验证**：添加特定 TXT 记录
- **TLS-ALPN-01**：通过 TLS 握手中的 ALPN 扩展验证

ACME 证书的有效期为 90 天，强制自动续期。对代理场景而言，使用 Let's Encrypt 证书具有"看起来完全正常"的优势——因为这是互联网上最常见的证书来源。

## 4. 自签名证书及其检测风险

自签名证书不经过 CA 验证，客户端必须手动信任。在代理场景中使用自签名证书存在严重问题：

- **主动探测**：GFW 可以连接疑似代理服务器，如果证书不是由受信 CA 签发，即刻标记为可疑
- **指纹特征**：自签名证书通常缺少正常证书的扩展字段（如 CT SCT、OCSP Stapling）
- **信任链断裂**：浏览器会弹出警告，暴露代理使用者

## 5. Reality 协议的证书合成

Reality 协议采用革命性的方式解决证书问题——**不使用自己的证书**，而是复用真实网站（如 `www.microsoft.com`）的 TLS 握手：

```
客户端 ────── TLS ClientHello ──────► Prism (Reality Server)
                                              │
                                              ▼
                                    生成 X25519 临时密钥对
                                    使用 Ed25519 签名扩展 (认证)
                                              │
                                              ▼
                                    TLS ServerHello (with real SNI)
                                    [SNI: www.microsoft.com]
                                    [证书: 实际由远端 microsoft.com 返回]
                                              │
                                              ▼
客户端 ◄──── TLS 握手完成 ──────── Prism
         (看起来连接的是 microsoft.com)
```

Reality 的关键创新：

- **X25519 密钥交换**：在 TLS 握手的 `key_share` 扩展中嵌入 X25519 公钥
- **Ed25519 认证**：客户端通过预共享的公钥验证服务器身份
- **Session ID 复用**：利用 TLS 1.3 的 session ID 字段传递认证数据
- 任何主动探测者看到的都是合法的 microsoft.com 证书——因为它就是

详见 [[stealth/reality]]。

## 6. 证书透明度日志（Certificate Transparency）

CT 日志是公开的、仅追加的证书记录，旨在检测错误签发：

- 所有主流 CA 被要求将签发的证书提交到 CT 日志
- 浏览器（Chrome）要求证书必须包含 SCT（Signed Certificate Timestamp）
- CT 日志可通过 [crt.sh](https://crt.sh) 等工具公开查询

**对隐蔽代理的影响**：

- 使用 Let's Encrypt 等公开 CA 签发的证书会出现在 CT 日志中
- 审查者可以扫描 CT 日志，发现疑似代理的域名（虽然无法直接确认）
- Reality 协议通过复用真实网站证书，完全规避了 CT 日志暴露风险

## 7. 证书与检测的关系

| 证书来源 | 主动探测 | CT 暴露 | 用户感知 |
|----------|----------|---------|----------|
| 自签名 | ❌ 立即被标记 | 无 | 浏览器警告 |
| 正规 CA 签发 | ✅ 看起来正常 | ⚠️ 出现在 CT 日志 | 正常 |
| Let's Encrypt | ✅ 最常见 | ⚠️ CT 日志可查 | 正常 |
| Reality (复用) | ✅ 就是真实证书 | ✅ 不产生新证书 | 正常 |

## 8. 在 Prism 中的实践

Prism 支持多种证书配置模式：

- **标准 TLS**：配置证书链 + 私钥路径，用于常规 TLS 代理
- **Self-signed**：开发/测试环境使用
- **Reality**：生产环境推荐，配置 `reality` 段，指定 SNI 目标和密钥对

证书相关配置详见 [[docs/agent/configuration]]，加密原语详见 [[crypto]]。
