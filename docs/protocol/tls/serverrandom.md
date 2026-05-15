---
title: "serverrandom — TLS ServerRandom 特征"
module: "protocol"
type: api
tags: [protocol, tls, serverrandom, 指纹]
created: 2026-05-15
updated: 2026-05-15
---

# TLS ServerRandom 特征

> 参考文档: [[ref/protocol/tls-serverrandom|TLS ServerRandom 规范]]

## 概述

TLS ServerRandom 是 ServerHello 消息中的 32 字节随机数。Reality 等伪装方案利用 ServerRandom 的特定字节模式进行身份验证。

## 相关页面

- [[protocol/tls|tls]] — TLS 模块概览
- [[stealth/reality/auth|auth]] — Reality 认证
- [[ref/protocol/tls-serverrandom|规范详情]]
