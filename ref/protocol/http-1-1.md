---
title: HTTP/1.1 协议
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [protocol, http, web]
---

# HTTP/1.1 协议

> HTTP/1.1 是 Web 的基础协议，代理服务器处理 HTTP CONNECT 隧道。

---

## 概述

HTTP/1.1 特点：
- 持久连接（Keep-Alive）
- 管道化请求
- 分块传输编码

---

## 在代理中的应用

HTTP CONNECT 方法用于建立 TCP 隧道：
- 客户端发送 `CONNECT host:port HTTP/1.1`
- 代理连接目标服务器
- 转发双向数据流

详见 [[ref/mihomo/protocols/http|HTTP 代理]]。

---

## 相关参考

- HTTP RFC 7230-7235
- [[ref/protocol/http-proxy-spec|HTTP CONNECT 规范]]
- [[core/protocol/http|HTTP 协议实现]]