---
title: "DNS 解析原理"
category: "network"
type: ref
layer: ref
module: ref
source: "RFC 1035"
tags: [网络, dns, 解析, 查询, 缓存, 递归]
created: 2026-05-17
updated: 2026-05-17
---

# DNS 解析原理

**类别**: 网络

## 概述

DNS（Domain Name System）是互联网的域名解析系统，将域名转换为 IP 地址。DNS 采用分层分布式架构，是网络通信的基础服务。

### DNS 架构

DNS 的分层架构：

```
DNS 分层架构：

┌─────────────────────────────────────────────┐
│                 Root DNS                     │
│  (.) - 根域名服务器                          │
│  全球 13 组服务器                            │
├─────────────────────────────────────────────┤
│            TLD DNS (顶级域)                  │
│  .com, .net, .org, .cn, ...                 │
│  各顶级域的权威服务器                        │
├─────────────────────────────────────────────┤
│          Second-level DNS (二级域)          │
│  example.com, google.com, ...               │
│  各域名的权威服务器                          │
├─────────────────────────────────────────────┤
│             Local Resolver                   │
│  本地 DNS 解析器                             │
│  ISP 或公共 DNS                              │
├─────────────────────────────────────────────┤
│               客户端                          │
│  应用发起 DNS 查询                           │
└─────────────────────────────────────────────┘

特点：
- 分布式存储
- 分层授权
- 缓存加速
```

### 解析类型

DNS 解析的类型：

| 类型 | 说明 | 用途 |
|------|------|------|
| **A** | IPv4 地址 | 域名 → IPv4 |
| **AAAA** | IPv6 地址 | 域名 → IPv6 |
| **CNAME** | 别名 | 域名别名 |
| **NS** | 名称服务器 | 域名服务器 |
| **MX** | 邮件交换 | 邮件路由 |
| **TXT** | 文本记录 | 验证信息 |
| **PTR** | 反向解析 | IP → 域名 |

## DNS 查询格式

### 报文结构

DNS 查询报文结构：

```
DNS 报文结构：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            ID                |QR| OPCODE |AA|TC|RD|RA| Z| RCODE|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            QDCOUNT            |           ANCOUNT             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            NSCOUNT            |           ARCOUNT             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Question Section                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Answer Section                     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Authority Section                  ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Additional Section                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

头部字段（12 字节）：
- ID (16 bit): 查询标识，响应需匹配
- QR (1 bit): 查询(0)/响应(1)
- OPCODE (4 bit): 操作类型（标准查询=0）
- AA (1 bit): 权威答案标志
- TC (1 bit): 截断标志（UDP 超长）
- RD (1 bit): 期望递归
- RA (1 bit): 支持递归
- RCODE (4 bit): 响应码（成功=0）
- QDCOUNT: 问题数
- ANCOUNT: 回答数
- NSCOUNT: 授权记录数
- ARCOUNT: 附加记录数
```

### Question Section

查询问题部分：

```
Question Section 格式：

域名编码（标签序列）：
example.com 编码为：
7 'e' 'x' 'a' 'm' 'p' 'l' 'e' 3 'c' 'o' 'm' 0

格式：
┌─────────────────────────────────────────────┐
│ 长度(1B) │ 标签字符 │ 长度(1B) │ 标签字符 │ ... │ 0 │
└─────────────────────────────────────────────┘

完整 Question：
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           QNAME (域名编码)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           QTYPE               |           QCLASS             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段：
- QNAME: 域名（标签序列编码）
- QTYPE: 查询类型（A=1, AAAA=28）
- QCLASS: 类（IN=1，互联网）
```

### Answer Section

响应答案部分：

```
Answer Section 格式（资源记录）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           NAME (域名)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           TYPE                 |           CLASS              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          TTL                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          RDLENGTH             |          RDATA               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

字段：
- NAME: 响应的域名（可能压缩）
- TYPE: 记录类型
- CLASS: 类
- TTL: 缓存时间（秒）
- RDLENGTH: 数据长度
- RDATA: 资源数据

A 记录 RDATA：IPv4 地址（4 字节）
AAAA 记录 RDATA：IPv6 地址（16 字节）
CNAME 记录 RDATA：别名域名
```

## 解析流程

### 递归解析

完整的递归解析流程：

```
递归解析流程：

客户端 → Resolver → Root → TLD → 权威 → Resolver → 客户端

详细流程：

1. 客户端查询本地 Resolver
   Query: www.example.com A
   
2. Resolver 检查缓存
   - 有缓存 → 直接返回
   - 无缓存 → 开始递归
   
3. Resolver 查询 Root DNS
   Query: www.example.com A
   Response: NS for .com (a.gtld-servers.net)
   
4. Resolver 查询 .com TLD
   Query: www.example.com A
   Response: NS for example.com (ns1.example.com)
   
5. Resolver 查询 example.com 权威
   Query: www.example.com A
   Response: A = 192.0.2.1
   
6. Resolver 缓存结果
   TTL: 3600 秒
   
7. Resolver 返回客户端
   Response: www.example.com A = 192.0.2.1

缓存作用：
- Root/TLD 信息长期缓存
- 域名记录按 TTL 缓存
- 大幅减少查询次数
```

### 迭代解析

迭代解析的工作方式：

```
迭代解析：

Resolver 角色：
- 不完成完整解析
- 返回下一步查询目标
- 客户端自行迭代查询

流程：
客户端 → Resolver → 返回 TLD NS
客户端 → TLD → 返回权威 NS
客户端 → 权威 → 返回 IP

特点：
- Resolver 负担小
- 客户端负担大
- 实际较少使用
```

## DNS 缓存

### TTL 机制

DNS 缓存的 TTL 控制：

```
TTL (Time To Live)：

作用：
- 控制缓存时间
- 权威服务器指定
- 各级缓存遵循

典型值：
| 记录类型 | TTL 建议 |
|----------|----------|
| A/AAAA | 300-3600 秒 |
| CNAME | 3600-86400 秒 |
| NS | 86400+ 秒 |
| MX | 3600 秒 |

缓存行为：
- TTL 到期前使用缓存
- TTL 到期后重新查询
- 更新时有过渡期（旧缓存仍可用）

注意：
- 修改 DNS 需等待 TTL 过期
- 降低 TTL 可加速更新
- 但增加查询负担
```

### 缓存层级

DNS 缓存的多个层级：

```
缓存层级：

┌─────────────────────────────────────────────┐
│ 应用缓存                                      │
│ 浏览器、应用内部缓存                          │
│ TTL: 通常 60-120 秒                           │
├─────────────────────────────────────────────┤
│ 操作系统缓存                                  │
│ 系统 DNS 缓存                                 │
│ TTL: 遵循 DNS TTL                            │
├─────────────────────────────────────────────┤
│ Resolver 缓存                                 │
│ ISP 或公共 DNS                                │
│ TTL: 遵循 DNS TTL                            │
├─────────────────────────────────────────────┤
│ 权威服务器                                    │
│ 不缓存，提供原始数据                          │
└─────────────────────────────────────────────┘

刷新时机：
- TTL 到期
- 强制刷新（dig @resolver domain +nocookie）
- 服务重启
```

## DNS-over-X

### DNS-over-UDP

传统 DNS 查询：

```
DNS-over-UDP：

特点：
- UDP 53 端口
- 无加密，明文传输
- 快速，低延迟
- 适合小查询

限制：
- 最大 512 字节（传统）
- EDNS0 扩展支持更大
- 可能截断（TC 标志）

安全风险：
- 可被监听
- 可被篡改（DNS 污染）
- 无隐私保护
```

详见 [[ref/protocol/dns-over-udp|DNS-over-UDP]]。

### DNS-over-TLS

加密 DNS 查询：

```
DNS-over-TLS (DoT)：

特点：
- TLS 加密
- 853 端口
- RFC 7858 定义
- 保护隐私

握手：
- TCP 连接
- TLS 1.3 握手
- DNS 查询

优势：
- 加密查询内容
- 防止 DNS 污染
- 隐私保护

劣势：
- TLS 握手开销
- 端口独立（可能被封锁）
```

详见 [[ref/protocol/dns-over-tls|DNS-over-TLS]]。

### DNS-over-HTTPS

HTTPS 封装的 DNS：

```
DNS-over-HTTPS (DoH)：

特点：
- HTTPS 封装
- 443 端口
- RFC 8484 定义
- 与普通 HTTPS 混合

格式：
GET /dns-query?dns=<base64-encoded>
Content-Type: application/dns-message

或 POST：
POST /dns-query
Content-Type: application/dns-message
Body: <DNS wire format>

优势：
- HTTPS 加密
- 443 端口（难以封锁）
- 浏览器原生支持

劣势：
- HTTP 开销
- 可能延迟更高
- 浏览器偏好特定服务商
```

详见 [[ref/protocol/dns-over-https|DNS-over-HTTPS]]。

## 在 Prism 中的应用

### DNS 模块

Prism 的 DNS 解析实现：

```
Prism DNS 模块：

模块：resolve/dns

组件：
- upstream: DNS 上游配置
- cache: DNS 缓存
- coalescer: 查询合并
- format: DNS 报文格式

配置：
{
    "upstreams": [
        {"type": "udp", "address": "8.8.8.8:53"},
        {"type": "dot", "address": "dns.google:853"},
        {"type": "doh", "address": "https://dns.google/dns-query"}
    ],
    "cache": {
        "size": 1000,
        "ttl_override": 300
    }
}

特性：
- 多上游支持
- 查询缓存
- 查询合并（避免重复）
- IPv4/IPv6 双栈查询
```

详见 [[resolve/dns|DNS 模块]]。

### 解析流程

Prism 的 DNS 解析流程：

```
Prism DNS 解析流程：

1. 接收解析请求
   请求域名: www.example.com
   
2. 检查缓存
   - 有缓存 → 返回缓存结果
   
3. 检查并发查询
   - 有相同查询进行中 → 合并等待
   
4. 发起查询
   - 选择上游（UDP/DoT/DoH）
   - 发送 DNS 查询
   
5. 处理响应
   - 解析响应报文
   - 缓存结果
   - 返回给请求者
   
6. Happy Eyeballs 支持
   - 同时查询 A 和 AAAA
   - 并行返回结果
```

## 参见

- [[ref/protocol/dns-over-udp|DNS-over-UDP]] — UDP DNS
- [[ref/protocol/dns-over-tls|DNS-over-TLS]] — DoT
- [[ref/protocol/dns-over-https|DNS-over-HTTPS]] — DoH
- [[resolve/dns|DNS 模块]] — Prism 实现
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 双栈解析
- [[ref/network/overview|网络基础概览]] — 网络索引