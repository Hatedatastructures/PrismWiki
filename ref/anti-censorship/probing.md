---
title: "主动探测"
category: "anti-censorship"
type: ref
tags: [流量对抗, 主动探测, 检测]
created: 2026-05-15
updated: 2026-05-15
---

# 主动探测

**类别**: 流量对抗

## 概述

主动探测是审查机构主动连接可疑服务器，通过分析响应来判断是否运行代理服务的技术。

## 原理

### 探测方式

1. **协议探测**: 发送特定协议的探测包
2. **行为探测**: 分析服务器的响应行为
3. **指纹探测**: 匹配已知代理软件的指纹

### 常见探测

- **TLS 探测**: 发送 TLS ClientHello，分析 ServerHello
- **HTTP 探测**: 发送 HTTP 请求，分析响应
- **端口探测**: 连接常见代理端口

### 防御机制

- **伪装**: 伪装成正常服务
- **验证**: 只响应合法客户端
- **速率限制**: 限制探测频率

## 在 Prism 中的应用

Prism 使用 Reality 协议防御主动探测：
- **独占特征**: 只响应包含正确标记的客户端
- **回退机制**: 非 Reality 客户端回退到真实 TLS

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| verify_client_hello | 验证客户端身份 | [[stealth/shadowtls/auth|auth]] |
| authenticate | Reality 认证 | [[stealth/reality/auth|auth]] |

## 参考资料

- [Active Probing](https://gfw.report/talks/ccs2020/en/)

## 相关知识
- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹
