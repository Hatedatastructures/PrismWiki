---
title: Reality 协议
created: 2026-05-27
updated: 2026-05-28
layer: core
source:
  - I:/code/Prism/include/prism/stealth/reality/
  - I:/code/Prism/src/prism/stealth/reality/
tags: [stealth, reality, overview]
---

# Reality 协议

Reality 是 Tier 0 级 TLS 伪装方案，通过 X25519 密钥交换和 TLS 指纹伪装实现抗主动探测。基于 XTLS Reality 协议设计。

## 子页面

- [[core/stealth/reality/scheme|Scheme]] — 方案基类实现
- [[core/stealth/reality/config|Config]] — 配置参数
- [[core/stealth/reality/constants|Constants]] — 常量定义
- [[core/stealth/reality/keygen|KeyGen]] — X25519 密钥生成工具
- [[core/stealth/reality/auth|Auth]] — 认证流程
- [[core/stealth/reality/handshake|Handshake]] — 完整握手流程
- [[core/stealth/reality/request|Request]] — ClientHello 请求构造
- [[core/stealth/reality/response|Response]] — ServerHello 响应处理
- [[core/stealth/reality/seal|Seal]] — AEAD 加密封装

## 相关文档

- [[core/stealth/overview|Stealth]] — 伪装模块总览
- [[core/crypto/x25519|X25519]] — 密钥交换
- [[core/crypto/aead|AEAD]] — 认证加密
