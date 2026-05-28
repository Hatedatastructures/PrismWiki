---
title: SOCKS5
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, socks5]
---
# SOCKS5 协议

SOCKS5 是一种网络代理协议，支持 TCP 和 UDP 传输，提供认证机制。

## 协议概述

SOCKS5（Socket Secure 5）是最广泛使用的代理协议之一：
- 支持 TCP 和 UDP 代理
- 支持用户名/密码认证
- 支持 TLS 加密（SOCKS5 over TLS）
- 支持无认证模式

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/socks5.go` | SOCKS5 适配器实现 |
| `transport/socks5/socks5.go` | SOCKS5 协议传输层 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "socks5-proxy"
    type: socks5
    server: server.example.com
    port: 1080
```

### 认证配置

```yaml
proxies:
  - name: "socks5-auth"
    type: socks5
    server: server.example.com
    port: 1080
    username: user
    password: pass
```

### TLS 配置

```yaml
proxies:
  - name: "socks5-tls"
    type: socks5
    server: server.example.com
    port: 1080
    tls: true
    skip-cert-verify: false
    fingerprint: chrome
```

### 完整配置选项

```yaml
proxies:
  - name: "socks5-full"
    type: socks5
    server: server.example.com
    port: 1080
    username: user
    password: pass
    tls: true
    skip-cert-verify: false
    fingerprint: chrome
    udp: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `socks5` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `username` | string | 否 | 用户名 |
| `password` | string | 否 | 密码 |
| `tls` | bool | 否 | 启用 TLS |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `fingerprint` | string | 否 | TLS 指纹 |
| `udp` | bool | 否 | 启用 UDP |

## 协议背景

SOCKS5 由 RFC 1928 定义，是 SOCKS（SOCKet Secure）协议的第五个版本。与 HTTP 代理不同，SOCKS5 工作在会话层（OSI 第 5 层），透明地转发任何上层协议的流量。

### 与 SOCKS4 的对比

| 特性 | SOCKS4 | SOCKS5 |
|------|--------|--------|
| 认证 | 无（仅用户名） | 可协商多种认证方法 |
| 地址类型 | 仅 IPv4 | IPv4、IPv6、域名 |
| UDP | 不支持 | 支持 UDP ASSOCIATE |
| 域名解析 | 客户端解析 | 服务器端解析 |

### 协议架构

SOCKS5 协议包含三个阶段：
1. **认证协商**：客户端和服务器协商认证方法
2. **请求处理**：客户端发送代理请求
3. **数据转发**：建立代理隧道转发数据

## 帧格式详解

### 阶段一：认证协商

#### 客户端方法选择请求

```
+----+----------+----------+
|VER | NMETHODS | METHODS  |
+----+----------+----------+
| 1  |    1     | 1-255    |
+----+----------+----------+
```

- **VER**（1 字节）：协议版本，固定为 `0x05`
- **METHODS**（1 字节）：客户端支持的认证方法数量
- **METHODS**（可变）：认证方法列表

| 方法值 | 含义 |
|--------|------|
| `0x00` | 无认证（NO AUTHENTICATION REQUIRED） |
| `0x01` | GSSAPI |
| `0x02` | 用户名/密码（USERNAME/PASSWORD） |
| `0x03` - `0x7F` | IANA 分配 |
| `0x80` - `0xFE` | 私有方法 |
| `0xFF` | 无可接受方法 |

#### 服务器方法选择响应

```
+----+--------+
|VER | METHOD |
+----+--------+
| 1  |   1    |
+----+--------+
```

- **VER**（1 字节）：`0x05`
- **METHOD**（1 字节）：服务器选择的认证方法

### 阶段二：认证（RFC 1929）

当服务器选择 `0x02`（用户名/密码）时：

#### 客户端认证请求

```
+----+------+----------+------+----------+
|VER | ULEN |   UNAME  | PLEN |   PASSWD |
+----+------+----------+------+----------+
| 1  |  1   | 1-255    |  1   | 1-255    |
+----+------+----------+------+----------+
```

- **VER**（1 字节）：认证子协议版本 `0x01`
- **ULEN**（1 字节）：用户名长度
- **UNAME**（1-255 字节）：用户名
- **PLEN**（1 字节）：密码长度
- **PASSWD**（1-255 字节）：密码

#### 服务器认证响应

```
+----+--------+
|VER | STATUS |
+----+--------+
| 1  |   1    |
+----+--------+
```

- **STATUS**：`0x00` = 成功，`0x01` = 失败

### 阶段三：代理请求

#### 客户端请求

```
+----+-----+-------+------+----------+----------+
|VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  |  X'00'|  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+
```

| 字段 | 值 | 含义 |
|------|-----|------|
| VER | `0x05` | 协议版本 |
| CMD | `0x01` | CONNECT（TCP 连接） |
| CMD | `0x02` | BIND（监听入站连接） |
| CMD | `0x03` | UDP ASSOCIATE（UDP 代理） |
| ATYP | `0x01` | IPv4 地址 |
| ATYP | `0x03` | 域名（第一个字节为长度） |
| ATYP | `0x04` | IPv6 地址 |

#### 地址编码格式

**IPv4 地址（ATYP = 0x01）**：

```
+----+----+----+----+----+----+----+
|ATYP| B1 | B2 | B3 | B4 | P1 | P2 |
+----+----+----+----+----+----+----+
|0x01|  4 bytes IPv4   | 2 bytes Port|
+----+----+----+----+----+----+----+
```

**域名地址（ATYP = 0x03）**：

```
+----+------+----------+------+----+----+
|ATYP| NLEN |   NAME   | P1   | P2 |
+----+------+----------+------+----+----+
|0x03|  1   | 1-255 bytes  |  2 bytes Port|
+----+------+----------+------+----+----+
```

- **NLEN**：域名长度（1 字节），不包含零终止符
- **NAME**：域名本身（ASCII 编码）

**IPv6 地址（ATYP = 0x04）**：

```
+----+----+----+ ... +----+----+----+
|ATYP|        16 bytes IPv6       | P1 | P2 |
+----+----+----+ ... +----+----+----+
```

#### 服务器响应

```
+----+-----+-------+------+----------+----------+
|VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
+----+-----+-------+------+----------+----------+
| 1  |  1  | X'00' |  1   | Variable |    2     |
+----+-----+-------+------+----------+----------+
```

| REP 值 | 含义 | 说明 |
|--------|------|------|
| `0x00` | Succeeded | 请求成功 |
| `0x01` | General SOCKS server failure | 通用服务器故障 |
| `0x02` | Connection not allowed by ruleset | 规则集不允许连接 |
| `0x03` | Network unreachable | 网络不可达 |
| `0x04` | Host unreachable | 主机不可达 |
| `0x05` | Connection refused | 连接被拒绝 |
| `0x06` | TTL expired | TTL 过期 |
| `0x07` | Command not supported | 命令不支持 |
| `0x08` | Address type not supported | 地址类型不支持 |

### UDP ASSOCIATE 帧格式

#### UDP 代理请求

```
+----+------+------+----------+----------+----------+
|RSV | FRAG | ATYP | DST.ADDR | DST.PORT |   DATA   |
+----+------+------+----------+----------+----------+
| 2  |  1   |  1   | Variable |    2     | Variable |
+----+------+------+----------+----------+----------+
```

- **RSV**（2 字节）：保留，固定为 `0x0000`
- **FRAG**（1 字节）：分片编号，`0x00` 表示不分片
- **DST.ADDR/DST.PORT**：最终目标地址
- **DATA**：UDP 数据载荷

#### UDP 关联流程

```
客户端                                          SOCKS5 服务器
  |                                                    |
  |  --- UDP ASSOCIATE Request (CMD=0x03) ---------->  |
  |    (通过 TCP 控制连接)                               |
  |                                                    |
  |  <--- UDP ASSOCIATE Response --------------------  |
  |    BND.ADDR = UDP 监听地址                          |
  |    BND.PORT = UDP 监听端口                          |
  |                                                    |
  |  === UDP 数据报 =================================  |
  |  [RSV][FRAG][ATYP][DST.ADDR][DST.PORT][DATA]      |
  |                                                    |
  |  <=== 响应数据报 =================================  |
  |  [RSV][FRAG][ATYP][SRC.ADDR][SRC.PORT][DATA]      |
  |                                                    |
```

UDP 关联的完整生命周期：
1. 客户端通过 TCP 发送 `UDP ASSOCIATE` 请求
2. 服务器返回 UDP 监听地址和端口
3. 客户端向该地址发送封装的 UDP 数据报
4. 当 TCP 控制连接关闭时，UDP 关联自动终止

## 完整通信流程图

```
┌─────────────────────────────────────────────────────────┐
│                    SOCKS5 完整流程                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  TCP 连接建立                                            │
│     │                                                    │
│     ▼                                                    │
│  ┌─────────────────────┐                                │
│  │  认证协商             │                                │
│  │  C: VER + METHODS   │                                │
│  │  S: VER + METHOD    │                                │
│  └────────┬────────────┘                                │
│           │ METHOD = 0x02?                              │
│           ▼                                             │
│  ┌─────────────────────┐                                │
│  │  用户名/密码认证       │                                │
│  │  C: VER + UNAME+PASS │                               │
│  │  S: VER + STATUS    │                                │
│  └────────┬────────────┘                                │
│           │ STATUS = 0x00?                              │
│           ▼                                             │
│  ┌─────────────────────┐                                │
│  │  代理请求             │                                │
│  │  C: VER+CMD+ATYP+... │                                │
│  │  S: VER+REP+BND+...  │                                │
│  └────────┬────────────┘                                │
│           │ REP = 0x00?                                 │
│           ▼                                             │
│  ┌─────────────────────┐                                │
│  │  数据隧道转发          │                                │
│  │  ←─── 双向流 ───→    │                                │
│  └─────────────────────┘                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 错误处理

### 方法协商失败

```
客户端                              服务器
  |  C: VER=5, METHODS=[0x01,0x03]   |
  |-------------------------------->|
  |  S: VER=5, METHOD=0xFF          |
  |<--------------------------------|
  |  (无可接受方法，客户端断开)         |
```

当服务器返回 `METHOD = 0xFF` 时，表示服务器不支持客户端提供的任何认证方法，客户端应立即断开连接。

### 认证失败

| 错误场景 | 服务器响应 | 客户端行为 |
|---------|-----------|-----------|
| 用户名或密码错误 | STATUS = `0x01` | 断开连接 |
| 认证超时 | 无响应 | 断开连接并重试 |
| 认证方法不被支持 | METHOD = `0xFF` | 尝试其他方法或断开 |

### 连接错误

| 错误代码 | 常见原因 | 排查建议 |
|---------|---------|---------|
| `0x01` 通用故障 | 服务器内部错误 | 检查服务器日志 |
| `0x02` 规则集拒绝 | ACL 或白名单限制 | 检查目标地址是否在允许列表 |
| `0x03` 网络不可达 | 路由问题 | 检查服务器网络连接 |
| `0x05` 连接被拒绝 | 目标端口关闭 | 检查目标服务是否运行 |
| `0x08` 地址类型不支持 | 服务器不支持 IPv6/域名 | 使用 IPv4 地址 |

### TLS over SOCKS5

mihomo 支持 SOCKS5 over TLS，在 TCP 连接建立后先进行 TLS 握手，然后再进行 SOCKS5 协议协商。

```
TCP 连接 → TLS 握手 → SOCKS5 认证协商 → SOCKS5 代理请求 → 数据隧道
```

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | SOCKS5 作为标准协议 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 部分 | UDP over TCP 需特殊处理 |

## Prism 实现边界

- **UDP ASSOCIATE 仅绑定 IPv4**：`udp::v4()` 硬编码，IPv6 客户端 UDP 无法工作
- 无错误日志，静默失败

详见 [[core/protocol/socks5/stream#实现边界|SOCKS5 实现边界]]

## 相关文档

- [[socks4]] - SOCKS4 协议
- [[ref/mihomo/transport/socks5]] - SOCKS5 传输层
- [[core/crypto/aead|aead]] - AEAD 加密
- [[ref/protocol/socks5-spec|socks5-spec]] - SOCKS5 协议规范