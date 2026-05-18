---
title: "SOCKS5 协议规范"
category: "protocol"
type: ref
layer: ref
module: ref
source: "RFC 1928"
tags: [协议, socks5, 代理, 认证, 隧道]
created: 2026-05-17
updated: 2026-05-17
---

# SOCKS5 协议规范

**类别**: 协议

## 概述

SOCKS5 是一种网络代理协议，由 RFC 1928 定义，支持 TCP 和 UDP 代理转发。SOCKS5 提供认证机制，支持多种地址类型，是应用层代理的标准协议。

### 协议特性

SOCKS5 的核心特性：

| 特性 | 说明 |
|------|------|
| **版本** | SOCKS 协议版本 5 |
| **认证** | 支持多种认证方法 |
| **地址类型** | IPv4、域名、IPv6 |
| **TCP 支持** | CONNECT 命令建立隧道 |
| **UDP 支持** | UDP ASSOCIATE 命令 |
| **无加密** | 协议本身不加密（需 TLS 包装） |

### 协议流程

SOCKS5 协议的基本流程：

```
SOCKS5 协议流程：

1. 握手阶段（方法协商）
   Client → Server: VER NMETHODS METHODS
   Server → Client: VER METHOD

2. 认证阶段（如果方法需要）
   子协议完成认证

3. 连接请求阶段
   Client → Server: VER CMD ATYP DST.ADDR DST.PORT
   Server → Client: VER REP ATYP BND.ADDR BND.PORT

4. 数据传输阶段
   双向转发数据
```

## 方法协商

### 客户端请求

客户端发起方法协商：

```
客户端握手请求格式：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  | NMETHODS |    METHODS[0]    |    METHODS[1]           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   ...   METHODS[NMETHODS-1]                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 版本号，固定为 0x05
- NMETHODS: 支持的方法数量（1 字节）
- METHODS: 支持的方法列表（NMETHODS 字节）

方法代码：
- 0x00: 无需认证 (NO AUTHENTICATION REQUIRED)
- 0x01: GSSAPI 认证
- 0x02: 用户名/密码认证 (USERNAME/PASSWORD)
- 0x03-0x7F: IANA 分配的方法
- 0x80-0xFE: 私有方法
- 0xFF: 无可接受方法 (NO ACCEPTABLE METHODS)
```

### 服务端响应

服务端选择认证方法：

```
服务端握手响应格式：

 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  |  METHOD               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 版本号，固定为 0x05
- METHOD: 服务端选择的方法

如果 METHOD = 0xFF，表示无可接受方法，连接关闭。
```

## 用户名密码认证

### 认证子协议

RFC 1929 定义的用户名密码认证：

```
用户名密码认证流程：

客户端认证请求：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  |  ULEN  |  UNAME[0]  |  UNAME[1]  |  ...  |  UNAME[ULEN-1] |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  PLEN  |  PASSWD[0]  |  PASSWD[1]  |  ...  |  PASSWD[PLEN-1]   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 子协议版本，固定为 0x01（注意：不是 0x05）
- ULEN: 用户名长度（1 字节）
- UNAME: 用户名（ULEN 字节）
- PLEN: 密码长度（1 字节）
- PASSWD: 密码（PLEN 字节）

服务端认证响应：
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  |  STATUS               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 子协议版本，固定为 0x01
- STATUS: 认证状态
  - 0x00: 成功
  - 其他: 失败
```

## 连接请求

### 请求格式

客户端发起连接请求：

```
连接请求格式：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  |  CMD  |  RSV    |  ATYP   |    DST.ADDR              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    DST.PORT                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 版本号，固定为 0x05
- CMD: 命令类型
  - 0x01: CONNECT（建立 TCP 连接）
  - 0x02: BIND（绑定端口，用于 FTP 等）
  - 0x03: UDP ASSOCIATE（UDP 关联）
- RSV: 保留字段，固定为 0x00
- ATYP: 地址类型
  - 0x01: IPv4（4 字节）
  - 0x03: 域名（1 字节长度 + 域名）
  - 0x04: IPv6（16 字节）
- DST.ADDR: 目标地址
- DST.PORT: 目标端口（2 字节，网络序）

地址格式示例：
ATYP = 0x01 (IPv4):
  DST.ADDR = 4 字节 IPv4 地址

ATYP = 0x03 (域名):
  DST.ADDR = 长度(1B) + 域名字符串

ATYP = 0x04 (IPv6):
  DST.ADDR = 16 字节 IPv6 地址
```

### 响应格式

服务端响应连接请求：

```
连接响应格式：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  VER  |  REP  |  RSV    |  ATYP   |    BND.ADDR              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    BND.PORT                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- VER: 版本号，固定为 0x05
- REP: 响应状态
  - 0x00: 成功
  - 0x01: 一般 SOCKS 服务器失败
  - 0x02: 不允许的连接
  - 0x03: 网络不可达
  - 0x04: 主机不可达
  - 0x05: 连接被拒绝
  - 0x06: TTL 过期
  - 0x07: 不支持的命令
  - 0x08: 不支持的地址类型
  - 0x09-0xFF: 未分配
- RSV: 保留字段，固定为 0x00
- ATYP: 地址类型
- BND.ADDR: 绑定地址
- BND.PORT: 绑定端口

成功响应后，TCP 连接已建立，开始双向转发数据。
```

## UDP ASSOCIATE

### UDP 关联请求

UDP 代理通过 UDP ASSOCIATE 命令：

```
UDP ASSOCIATE 流程：

1. 客户端发送 UDP ASSOCIATE 请求
   Client → Server: VER=5 CMD=3 ATYP DST.ADDR DST.PORT

2. 服务端响应 UDP 关联
   Server → Client: VER=5 REP=0 ATYP BND.ADDR BND.PORT
   BND.ADDR/PORT 是 UDP 中继的地址

3. 客户端发送 UDP 数据包
   Client → UDP Relay: UDP 请求帧

4. UDP 中继转发数据
   UDP Relay → Target: 原始 UDP 数据
   Target → UDP Relay: 响应 UDP 数据
   UDP Relay → Client: UDP 响应帧
```

### UDP 请求帧格式

UDP 数据帧的封装格式：

```
UDP 请求帧格式：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  RSV  |  FRAG |  ATYP   |    DST.ADDR                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    DST.PORT                   |    DATA                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段说明：
- RSV: 保留字段，固定为 0x0000
- FRAG: 分片号（通常为 0x00，表示不分片）
- ATYP: 地址类型
- DST.ADDR: 目标地址
- DST.PORT: 目标端口
- DATA: UDP 数据

注意：
- UDP 关联期间，TCP 连接保持打开
- TCP 连接关闭时，UDP 关联终止
- FRAG = 0 表示不分片（通常情况）
```

## 地址类型详解

### IPv4 地址

IPv4 地址格式：

```
IPv4 地址格式 (ATYP = 0x01):

示例：目标地址 192.168.1.1，端口 80

字节序列：
0x01 0xC0 0xA8 0x01 0x01 0x00 0x50

解析：
- ATYP: 0x01
- DST.ADDR: 192.168.1.1 (C0 A8 01 01)
- DST.PORT: 80 (00 50)
```

### 域名地址

域名地址格式：

```
域名地址格式 (ATYP = 0x03):

示例：目标地址 example.com，端口 443

字节序列：
0x03 0x0B 'e' 'x' 'a' 'm' 'p' 'l' 'e' '.' 'c' 'o' 'm' 0x01 0xBB

解析：
- ATYP: 0x03
- 长度: 0x0B (11 字节)
- 域名: "example.com"
- DST.PORT: 443 (01 BB)
```

### IPv6 地址

IPv6 地址格式：

```
IPv6 地址格式 (ATYP = 0x04):

示例：目标地址 ::1，端口 80

字节序列：
0x04 0x00 0x00 0x00 0x00 0x00 0x00 0x00
     0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x01
     0x00 0x50

解析：
- ATYP: 0x04
- DST.ADDR: ::1 (16 字节)
- DST.PORT: 80 (00 50)
```

## 在 Prism 中的应用

### SOCKS5 入站

Prism 支持 SOCKS5 作为入站协议：

```
Prism SOCKS5 入站配置：

{
    "type": "socks5",
    "listen": "0.0.0.0:1080",
    "users": [
        {
            "username": "user",
            "password": "pass"
        }
    ]
}

处理流程：
1. 接收客户端握手请求
2. 方法协商（支持无认证和用户名密码）
3. 认证（如果配置了用户）
4. 连接请求处理
5. 建立到目标的连接
6. 双向转发数据
```

详见 [[core/protocol/socks5|socks5]]。

### SOCKS5 流处理器

Prism 的 SOCKS5 流处理：

```
SOCKS5 流处理器：

class socks5_stream_handler {
    // 解析握手请求
    auto parse_greeting(buffer) → methods;
    
    // 选择方法
    auto select_method(methods) → chosen_method;
    
    // 认证处理
    auto handle_auth(buffer) → auth_result;
    
    // 解析连接请求
    auto parse_request(buffer) → target_address;
    
    // 响应连接请求
    auto send_response(status, bind_addr, bind_port);
};
```

## 安全考量

### 明文传输

SOCKS5 本身不加密：

```
安全风险：

1. 凭据明文传输
   - 用户名密码认证时，凭据明文传输
   - 需要 TLS 包装保护

2. 目标地址暴露
   - 目标域名/地址明文传输
   - 可能被中间人观察到

3. 数据明文传输
   - SOCKS5 只做转发，不加密
   - 需要 TLS 包装或应用层加密

解决方案：
- 使用 SOCKS5 over TLS
- 服务端配置 TLS 入站
- 客户端使用 TLS 连接代理
```

### 认证安全

用户名密码认证的安全：

```
认证安全建议：

1. 强密码
   - 使用足够强度的密码
   - 違免简单密码

2. TLS 包装
   - 在 TLS 连接上运行 SOCKS5
   - 保护凭据传输

3. 访问控制
   - 服务端限制目标地址
   - 防止滥用代理

4. 认证日志
   - 记录认证尝试
   - 监控异常行为
```

## 参见

- [[ref/protocol/http-proxy-spec|HTTP 代理规范]] — HTTP CONNECT 协议
- [[core/protocol/socks5|socks5]] — SOCKS5 实现接口
- [[ref/protocol/tcp-basics|TCP 基础]] — TCP 协议原理
- [[ref/protocol/udp-basics|UDP 基础]] — UDP 协议原理
- [[dispatch/handlers/socks5|SOCKS5 处理器]] — Prism 实现