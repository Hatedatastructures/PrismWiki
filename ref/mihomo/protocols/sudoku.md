---
title: Sudoku
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, sudoku]
---
# Sudoku 协议

Sudoku 是一种基于 AEAD 加密的代理协议，以其独特的自定义编码表格和 HTTP 伪装能力著称。它将加密、编码和流量伪装三个层次有机结合，在保持较高安全性的同时提供了丰富的流量特征自定义能力。

## 协议概述

Sudoku 的核心特性：

- **AEAD 加密**：支持 AES-GCM 和 ChaCha20-Poly1305 两种 AEAD 算法
- **自定义编码表格**：可自定义 Base64 编码表，改变输出字符集
- **HTTP Mask 伪装**：将代理流量伪装为 HTTP 请求/响应
- **多路复用**：内置多路复用机制，支持 HTTP 流和轮询模式
- **UDP over TCP**：通过 TCP 隧道传输 UDP 数据
- **WebSocket 模式**：支持 WebSocket 封装，进一步降低检测率
- **Padding 支持**：可配置填充范围，消除包大小特征

Sudoku 的名称来源于其自定义编码表格机制——如同数独游戏中数字的排列组合，Sudoku 允许用户自定义编码字符表的排列，使得编码后的输出与标准 Base64 完全不同，增加特征识别难度。

## 协议原理

### 整体架构

```
应用层数据
    ↓
代理协议封装（目标地址等）
    ↓
AEAD 加密
    ↓
自定义编码表格编码
    ↓
HTTP Mask 封装（可选）
    ↓
传输层（TCP）
```

### 数据加密流程

```
明文数据:
[协议头 | 目标地址 | 载荷数据]
    ↓
AEAD 加密:
[Nonce (8B) | 密文 (变长) | Auth Tag (16B)]
    ↓
自定义编码:
将二进制密文使用自定义 Base64 表格编码为文本
    ↓
HTTP Mask（如启用）:
封装为 HTTP GET/POST 请求格式
    ↓
TCP 传输
```

### AEAD 加密

Sudoku 使用 AEAD（Authenticated Encryption with Associated Data）算法提供加密和认证：

| 算法 | 密钥长度 | Nonce 长度 | Tag 长度 | 说明 |
|------|----------|------------|----------|------|
| `AES-128-GCM` | 16 字节 | 12 字节 | 16 字节 | 硬件加速支持，性能最佳 |
| `AES-256-GCM` | 32 字节 | 12 字节 | 16 字节 | 更高安全强度 |
| `ChaCha20-Poly1305` | 32 字节 | 12 字节 | 16 字节 | 无硬件加速时性能更好 |

AEAD 加密流程：

```
输入:
- 明文 (plaintext)
- 关联数据 (associated data，通常为协议头)
- 密钥 (key)
- Nonce (序号派生)

输出:
- 密文 (ciphertext)
- 认证标签 (authentication tag)

解密时:
- 如果 tag 验证失败 → 拒绝解密（数据被篡改）
- 如果 tag 验证通过 → 返回明文
```

Nonce 通常由连接序号和消息序号共同派生，确保每条消息使用唯一的 Nonce：

```
Nonce = KDF(key, connection_id, message_sequence_number)
```

### 自定义编码表格

Sudoku 的标志性特性是允许自定义 Base64 编码表：

标准 Base64 编码表：
```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

自定义编码表示例：
```
xpxvvpvvABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmno0123456789+/
```

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

| table-type | 说明 |
|------------|------|
| `standard` | 使用标准 Base64 编码表 |
| `prefer_ascii` | 优先使用 ASCII 可打印字符 |
| `custom` | 使用自定义编码表 |

自定义编码表的优势：
1. **增加识别难度**：标准 Base64 容易被 DPI 识别，自定义表改变了输出字符分布
2. **协议混淆**：即使知道是 Base64 编码，不知道编码表也无法正确解码
3. **灵活性**：每个用户可以有不同的编码表，增加大规模检测的难度

### 编码过程详解

```
二进制数据 (密文):
[0xDE 0xAD 0xBE 0xEF 0xCA 0xFE]
    ↓
按 6 位分组:
[110111 101010 110110 111011 001011 111010]
    ↓
查表 (自定义编码表):
索引 55→'v', 42→'q', 54→'u', 59→'z', 11→'L', 58→'y'
    ↓
编码输出:
"vquzLy"
```

## HTTP Mask 伪装

### 原理

HTTP Mask 将加密的代理数据封装在看似正常的 HTTP 请求或响应中，使流量在 DPI（深度包检测）层面表现为普通的 Web 流量。

```
加密数据 + 自定义编码
    ↓
封装为 HTTP 格式:

HTTP GET 模式:
GET /path?data=<base64-encoded-data> HTTP/1.1
Host: server.example.com
User-Agent: Mozilla/5.0 ...
Accept: */*

HTTP POST 模式:
POST /upload HTTP/1.1
Host: server.example.com
Content-Type: application/octet-stream
Content-Length: <length>

<binary-data>
```

### HTTP Mask 模式

| Mode | 描述 | 流量特征 |
|------|------|----------|
| `legacy` | 传统 HTTP 伪装 | GET/POST 请求格式，每个请求独立 |
| `stream` | HTTP 流模式 | 保持 HTTP 连接，持续传输数据 |
| `poll` | HTTP Poll 模式 | 周期性 HTTP 请求，模拟轮询行为 |
| `auto` | 自动选择 | 根据网络条件自动选择最佳模式 |
| `ws` | WebSocket 模式 | 升级为 WebSocket 连接，全双工传输 |

### WebSocket 模式

WebSocket 模式提供最佳的性能和隐蔽性：

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

WebSocket 流程：

```
Client                          Server
  |  HTTP GET (升级请求)          |
  |  Upgrade: websocket          |
  |  Connection: Upgrade          |
  |----------------------------->|
  |  HTTP 101 Switching Protocols |
  |<-----------------------------|
  |  WebSocket 帧传输              |
  |  (加密的代理数据)              |
```

WebSocket 模式的优势：
- **全双工**：同时收发数据，无需轮询
- **低开销**：WebSocket 帧头部仅 2-14 字节
- **持久连接**：一条连接承载所有数据
- **隐蔽性强**：WebSocket 流量与正常 Web 应用无异

### HTTP Mask 多路复用

在流模式下，Sudoku 支持 HTTP 连接的多路复用：

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

多路复用允许：
- 单个 HTTP 连接承载多条代理流
- 减少连接建立开销
- 更好地模拟正常浏览行为

## Padding 技术

Sudoku 支持数据填充以消除包大小特征：

```yaml
proxies:
  - name: "sudoku-padded"
    type: sudoku
    server: server.example.com
    port: 443
    key: your-key
    padding-min: 100
    padding-max: 1400
```

| 参数 | 说明 |
|------|------|
| `padding-min` | 最小填充后包大小 |
| `padding-max` | 最大填充后包大小 |

Padding 在每个数据帧加密后、编码前进行，确保填充数据也被加密。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/sudoku.go` | Sudoku 适配器，配置解析和出站集成 |
| `transport/sudoku/sudoku.go` | Sudoku 协议实现，加密/解密/编码 |
| `transport/sudoku/obfs/httpmask/` | HTTP 伪装模块，支持多种模式 |
| `transport/sudoku/obfs/ws/` | WebSocket 模式实现 |

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
| `key` | string | 是 | 加密密钥 |
| `aead-method` | string | 否 | AEAD 加密方法 |
| `padding-min` | int | 否 | 最小填充大小 |
| `padding-max` | int | 否 | 最大填充大小 |
| `http-mask` | bool | 否 | 启用 HTTP 伪装 |
| `http-mask-mode` | string | 否 | HTTP Mask 模式 |
| `http-mask-tls` | bool | 否 | HTTP Mask 使用 TLS |
| `http-mask-multiplex` | string | 否 | HTTP Mask 多路复用 |
| `table-type` | string | 否 | 编码表格类型 |
| `custom-table` | string | 否 | 自定义编码表 |
| `path-root` | string | 否 | WebSocket 路径 |

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
| Crypto | 完全兼容 | AEAD 加密 |

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 加密
- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
