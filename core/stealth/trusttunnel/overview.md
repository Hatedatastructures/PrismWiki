---
title: TrustTunnel 协议
created: 2026-05-27
updated: 2026-05-28
layer: core
source:
  - I:/code/Prism/include/prism/stealth/trusttunnel/
  - I:/code/Prism/src/prism/stealth/trusttunnel/
tags: [stealth, trusttunnel, overview, h2]
---

# TrustTunnel 协议

TrustTunnel 是 HTTP/2 CONNECT 代理方案，通过 Basic Auth 认证和 h2mux 多路复用实现安全隧道。

## 子页面

- [[core/stealth/trusttunnel/scheme|Scheme]] — 方案基类实现
- [[core/stealth/trusttunnel/config|Config]] — 配置参数

## 相关文档

- [[core/stealth/overview|Stealth]] — 伪装模块总览
- [[core/multiplex/h2mux/craft|h2mux]] — HTTP/2 多路复用
