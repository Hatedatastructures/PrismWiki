---
title: "sessionid — TLS SessionID 特征"
module: "protocol"
type: api
tags: [protocol, tls, sessionid, 指纹]
created: 2026-05-15
updated: 2026-05-15
---

# TLS SessionID 特征

> 参考文档: [[ref/protocol/tls-sessionid|TLS SessionID 规范]]

## 概述

TLS SessionID 是 ClientHello/ServerHello 中用于会话恢复的标识符。Reality 等伪装方案利用 SessionID 进行会话绑定和身份验证。

## 相关页面

- [[protocol/tls|tls]] — TLS 模块概览
- [[stealth/reality/auth|auth]] — Reality 认证
- [[ref/protocol/tls-sessionid|规范详情]]
