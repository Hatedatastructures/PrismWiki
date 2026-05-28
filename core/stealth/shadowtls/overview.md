---
title: ShadowTLS 协议
created: 2026-05-27
updated: 2026-05-28
layer: core
source:
  - I:/code/Prism/include/prism/stealth/shadowtls/
  - I:/code/Prism/src/prism/stealth/shadowtls/
tags: [stealth, shadowtls, overview]
---

# ShadowTLS 协议

ShadowTLS 是 Tier 1 级 TLS 伪装方案，通过代理真实 TLS 服务器的握手过程来伪装代理流量。

## 子页面

- [[core/stealth/shadowtls/scheme|Scheme]] — 方案基类实现
- [[core/stealth/shadowtls/config|Config]] — 配置参数
- [[core/stealth/shadowtls/constants|Constants]] — 常量定义
- [[core/stealth/shadowtls/auth|Auth]] — HMAC-SHA1 累积认证
- [[core/stealth/shadowtls/handshake|Handshake]] — 握手流程
- [[core/stealth/shadowtls/transport|Transport]] — 传输层包装

## 相关文档

- [[core/stealth/overview|Stealth]] — 伪装模块总览
