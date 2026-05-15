---
title: "analyzer.hpp — 外层协议检测"
source: "include/prism/recognition/probe/analyzer.hpp"
module: "recognition"
type: api
tags: [recognition, probe, analyzer, 协议检测, 魔术字节]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/probe/probe
  - recognition/recognition
  - protocol/analysis
  - ref/protocol/tls-clienthello
  - ref/anti-censorship/tls-fingerprint
---

# analyzer.hpp

> 源码: `include/prism/recognition/probe/analyzer.hpp` + `src/prism/recognition/probe/analyzer.cpp`
> 模块: [[recognition|Recognition]] / probe

## 概述

通过魔术字节快速判断连接的外层协议类型（HTTP/SOCKS5/TLS/Shadowsocks）。该函数是纯内存操作，不涉及任何网络 I/O，可安全并发调用。检测采用排除法：匹配已知协议特征后直接返回，否则 fallback 到 Shadowsocks。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/analysis\|analysis]] | protocol_type 枚举 |
| 被依赖 | [[recognition/probe/probe\|probe]] | 探测函数调用 detect |

## 命名空间

`psm::recognition::probe`

---

## 匿名命名空间: is_http_request()

### 功能说明

内部辅助函数，检查预读数据是否以已知 HTTP 方法名开头。匹配 9 种标准 HTTP 方法：GET、POST、HEAD、PUT、DELETE、CONNECT、OPTIONS、TRACE、PATCH。每个方法名后必须紧跟空格字符。

### 签名

```cpp
bool is_http_request(const std::string_view data) noexcept;
```

### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `data` | `const std::string_view` | 输入 | 预读数据 |

### 返回值

`bool` — 匹配任一 HTTP 方法前缀时返回 `true`。

### 调用（向下）

无（纯内存比较）。

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `detect()` | 本文件 | TLS 和 SOCKS5 未命中后检查 HTTP |

### 知识域

- [[ref/protocol/http-connect\|HTTP CONNECT]]

---

## 函数: detect()

### 功能说明

从预读数据检测外层协议类型。检测顺序为 SOCKS5 -> TLS -> HTTP -> Shadowsocks（排除法 fallback）。TLS 检测必须检查两字节 `0x16 0x03`，因为 SS2022 salt 有约 1/256 概率首字节恰好为 `0x16`。空数据返回 `unknown`。函数为纯计算操作，无状态，线程安全。

### 签名

```cpp
[[nodiscard]] auto detect(std::string_view peek_data) -> protocol::protocol_type;
```

### 参数表格

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `peek_data` | `std::string_view` | 输入 | 预读数据（通常是前 24 字节） |

### 返回值

`protocol::protocol_type` — 协议类型枚举值。可能为 `socks5`、`tls`、`http`、`shadowsocks` 或 `unknown`。

### 调用（向下）

| 被调用函数 | 模块 | 说明 |
|------------|------|------|
| `is_http_request()` | 本文件（匿名命名空间） | 检查是否为 HTTP 方法前缀 |

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `probe()` | [[recognition/probe/probe\|probe]] | 预读完成后调用 detect 判断协议 |

### 知识域

- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹]]
- [[ref/protocol/socks5-rfc1928\|SOCKS5]]
- [[ref/protocol/http-connect\|HTTP]]

---

## 检测逻辑详解

| 优先级 | 条件 | 结果 | 说明 |
|--------|------|------|------|
| 1 | 首字节 `0x05` | `socks5` | SOCKS5 版本协商 |
| 2 | 前两字节 `0x16 0x03` | `tls` | TLS 记录层（ContentType=Handshake, Version=3.x） |
| 3 | 以 HTTP 方法名 + 空格开头 | `http` | GET/POST/HEAD/PUT/DELETE/CONNECT/OPTIONS/TRACE/PATCH |
| 4 | 以上均不匹配 | `shadowsocks` | 排除法 fallback |
| - | 空数据 | `unknown` | 无法判断 |

## 知识域

- [[ref/protocol/tls-clienthello\|TLS ClientHello]]
- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹]]
- [[recognition/probe/probe\|外层协议探测]]
- [[protocol/analysis\|魔术字节检测]]
