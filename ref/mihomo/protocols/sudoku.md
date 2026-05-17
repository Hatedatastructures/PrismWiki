---
title: Sudoku
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, sudoku]
---
# Sudoku 协议

Sudoku 是一种基于 AEAD 加密的代理协议，支持 HTTP 伪装。

## 协议概述

Sudoku 特性：
- AEAD 加密（AES-GCM、ChaCha20-Poly1305）
- 支持 HTTP Mask 伪装
- 支持多路复用
- 支持 UDP over TCP
- 支持自定义表格

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/sudoku.go` | Sudoku 适配器 |
| `transport/sudoku/sudoku.go` | Sudoku 协议实现 |
| `transport/sudoku/obfs/httpmask/` | HTTP 伪装 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "sudoku-proxy"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
```

### HTTP Mask 配置

```yaml
proxies:
  - name: "sudoku-httpmask"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
    http-mask: true
    http-mask-mode: auto
    http-mask-tls: true
```

### 多路复用

```yaml
proxies:
  - name: "sudoku-mux"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
    http-mask: true
    http-mask-mode: stream
    http-mask-multiplex: on
```

### 自定义表格

```yaml
proxies:
  - name: "sudoku-table"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
    table-type: prefer_ascii
    custom-table: xpxvvpvv
```

### WebSocket 模式

```yaml
proxies:
  - name: "sudoku-ws"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
    http-mask: true
    http-mask-mode: ws
    http-mask-tls: true
    path-root: /sudoku
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `sudoku` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `key` | string | 是 | 密钥 |
| `aead-method` | string | 否 | AEAD 方法 |
| `padding-min` | int | 否 | 最小填充 |
| `padding-max` | int | 否 | 最大填充 |
| `http-mask` | bool | 否 | 启用 HTTP 伪装 |
| `http-mask-mode` | string | 否 | HTTP Mask 模式 |

## HTTP Mask 模式

| Mode | 描述 |
|------|------|
| `legacy` | 传统 HTTP 伪装 |
| `stream` | HTTP 流模式 |
| `poll` | HTTP Poll 模式 |
| `auto` | 自动选择 |
| `ws` | WebSocket 模式 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Sudoku 协议实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 内置多路复用 |

## 相关文档

- [[../../crypto/aead]] - AEAD 加密
- [[../../protocol/proxy-protocols]] - 代理协议概述