---
title: "中间人劫持 (MITM)"
category: "anti-censorship"
type: ref
tags: [流量对抗, mitm, 劫持, 安全]
created: 2026-05-15
updated: 2026-05-15
---

# 中间人劫持 (MITM)

**类别**: 流量对抗

## 概述

中间人劫持是攻击者插入通信双方之间，拦截和篡改数据的攻击方式。在审查场景中，审查机构可能使用 MITM 技术劫持 TLS 连接。

## 原理

### 攻击方式

1. **证书伪造**: 伪造目标网站的证书
2. **TLS 降级**: 强制使用弱加密
3. **SNI 篡改**: 修改 SNI 域名

### 防御机制

- **证书验证**: 严格验证证书链
- **证书固定**: 固定目标网站的证书
- **前向保密**: 使用 ECDHE 密钥交换

## 在 Prism 中的应用

Prism 使用 Reality 协议对抗 MITM：
- **证书获取**: 从真实网站获取证书
- **证书验证**: 验证证书的真实性

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| fetch_dest_certificate | 获取目标证书 | [[stealth/reality/handshake|handshake]] |

## 参考资料

- [Man-in-the-middle attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)

## 相关知识
- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/replay-attack|重放攻击]] — 重放攻击
