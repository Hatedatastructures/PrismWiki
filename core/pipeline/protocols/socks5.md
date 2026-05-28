---
tags: [pipeline, socks5, protocol-handler, handshake, udp]
title: "SOCKS5 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/socks5.hpp"
module: pipeline
updated: 2026-05-27
---

# SOCKS5 协议处理管道

SOCKS5 代理协议处理器。支持 CONNECT（TCP 隧道）和 UDP_ASSOCIATE（UDP 中继）命令。无内层加密。

## 处理流程

1. `wrap_with_preview()` 包装入站传输（预读数据为空，SOCKS5 从方法协商开始）
2. 创建 SOCKS5 relay，执行握手（方法协商 + 用户名密码认证）
3. 解析请求命令和目标地址
4. CONNECT：拨号上游 → 发送成功响应 → `tunnel()` 双向转发
5. UDP_ASSOCIATE：`make_datagram_router()` 创建路由回调 → `async_associate()` 启动 UDP 中继

## 命令类型

| 命令 | 值 | 说明 |
|------|-----|------|
| CONNECT | 0x01 | TCP 隧道 |
| UDP_ASSOCIATE | 0x03 | UDP 中继 |

## 认证方法

| 方法 | 值 | 说明 |
|------|-----|------|
| No Auth | 0x00 | 无认证 |
| Username/Password | 0x02 | 用户名密码认证 |

## 地址类型

| 类型 | 值 | 说明 |
|------|-----|------|
| IPv4 | 0x01 | 4 字节 |
| Domain | 0x03 | 长度 + 域名 |
| IPv6 | 0x04 | 16 字节 |

## 设计决策

### 为什么 SOCKS5 预读数据为空？

SOCKS5 握手从方法协商（客户端发送 `VER:0x05, NMETHODS, METHODS`）开始。probe 阶段检测到 SOCKS5 特征（版本号 0x05）后，预读的 24 字节不够完整协商。实际实现中 relay 从头开始读取完整握手帧。

**后果**: `data` 参数传入后 relay 不使用预读数据，由 `wrap_with_preview()` 透传给传输层。

## 引用关系

### 被调用

- [[core/instance/dispatch/table|dispatch]] — 注册为 SOCKS5 处理器

### 依赖

- [[core/pipeline/primitives|primitives]] — `wrap_with_preview`, `dial`, `tunnel`, `make_datagram_router`
