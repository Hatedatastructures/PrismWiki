---
title: "message -- DNS 报文编解码"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/format.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, format, wire, encode, decode, rfc1035]
created: 2026-05-17
updated: 2026-05-28
related:
  - core/resolve/dns/upstream
  - core/resolve/dns/detail/cache
---

# message -- DNS 报文编解码

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/format.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

该文件实现 DNS 二进制报文的构造与解析（RFC 1035），完全不依赖系统 resolver。支持域名压缩指针、多种记录类型，以及 TCP 帧格式的封装。

## qtype 枚举

| 值 | 名称 | 说明 |
|------|------|------|
| 1 | `a` | IPv4 地址记录 |
| 2 | `ns` | 权威名称服务器 |
| 5 | `cname` | 规范名称（别名） |
| 6 | `soa` | 区域起始授权 |
| 15 | `mx` | 邮件交换 |
| 16 | `txt` | 文本记录 |
| 28 | `aaaa` | IPv6 地址记录 |
| 41 | `opt` | EDNS0 选项 |

## 数据结构

### question -- DNS 查询段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `memory::string` | 域名，小写无末尾点号 |
| `query_type` | `qtype` | 查询类型 |
| `qclass` | `uint16_t` | 查询类，默认 IN(1) |

### record -- DNS 资源记录

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `memory::string` | 拥有者名称 |
| `type` | `qtype` | 记录类型 |
| `rclass` | `uint16_t` | 记录类 |
| `ttl` | `uint32_t` | 生存时间（秒） |
| `rdata` | `memory::vector<uint8_t>` | 原始 RDATA |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `extract_ipv4(record) -> optional<address_v4>` | 从 A 记录提取 IPv4（rdata 需恰好 4 字节） |
| `extract_ipv6(record) -> optional<address_v6>` | 从 AAAA 记录提取 IPv6（rdata 需恰好 16 字节） |

## message 类

### Header 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `uint16_t` | 报文标识（默认 0，调用方发送前设置） |
| `qr` | `bool` | 0=查询, 1=响应 |
| `opcode` | `uint8_t` | 操作码（0=标准查询） |
| `aa` | `bool` | 权威应答 |
| `tc` | `bool` | 截断标志 |
| `rd` | `bool` | 期望递归 |
| `ra` | `bool` | 可用递归 |
| `rcode` | `uint8_t` | 响应码 |

### 四段数据

| 段 | 字段 | 说明 |
|------|------|------|
| Question | `questions` | 查询段列表 |
| Answer | `answers` | 应答段列表 |
| Authority | `authority` | 权威段列表 |
| Additional | `additional` | 附加段列表 |

### 核心方法

| 方法 | 说明 |
|------|------|
| `pack() -> vector<uint8_t>` | 序列化：12 字节 Header -> Question -> Answer/Authority/Additional。域名编码采用压缩指针优化 |
| `unpack(data, mr) -> optional<message>` | 反序列化：检查 `size>=12` -> 解析 Header -> 逐段解码 -> 处理域名压缩指针 -> 检测循环引用（跳转 >255 次返回 nullopt） |
| `make_query(domain, qt, mr) -> message` | 构造标准递归查询：`id=0, rd=true, opcode=0`，域名自动转小写去末尾点号 |
| `extract_ips() -> vector<address>` | 遍历 answers，A 记录提取 IPv4，AAAA 记录提取 IPv6 |
| `min_ttl() -> uint32_t` | 遍历 answers/authority/additional 取最小 TTL |

### TCP 帧格式

| 函数 | 说明 |
|------|------|
| `pack_tcp(msg) -> vector<uint8_t>` | 在 DNS 报文前添加 2 字节大端长度前缀 |
| `unpack_tcp(data, mr) -> optional<message>` | 先读 2 字节长度，再解析报文主体 |

## DNS Wire Format 要点

### 域名编码

- 标签格式：`[length byte (0-63)] [label bytes]`，末尾 `[0]` 表示结束
- 压缩指针：高 2 位为 `11` 表示指针，后 14 位为报文内偏移位置
- 重复域名使用指针引用，例如 `"api.example.com"` 中的 `"example.com"` 部分可复用已编码位置

### Header 布局（12 字节）

ID(2) + FLAGS(2: QR/OPCODE/AA/TC/RD/RA/Z/RCODE) + QDCOUNT(2) + ANCOUNT(2) + NSCOUNT(2) + ARCOUNT(2)

### Record 布局

NAME(compressed) + TYPE(2) + CLASS(2) + TTL(4) + RDLENGTH(2) + RDATA(variable)

## 关键设计决策

### 为什么检测压缩指针循环引用

**问题**: 恶意或损坏的 DNS 报文可能包含循环压缩指针，导致无限解析。

**选择**: `unpack` 中跟踪跳转次数，超过 255 次返回 `nullopt`。

**后果**: 安全地防御畸形报文，但解析深度有限的极端合法报文会被拒绝（实践中 255 远超合法报文深度）。

### 为什么 id 默认为 0

**选择**: `make_query` 设置 `id=0`，由调用方在发送前自行设置。这使得多个调用方可以复用查询模板后设置不同 ID。

## 调用链

```
upstream::query_udp(server, query)
  +-> message::make_query(domain, qtype::a) -> 设置 rd=true, opcode=0
  +-> message::pack() -> 编码 header + questions（域名压缩）+ records
  -> 发送 UDP 报文

upstream::query_tcp(server, query)
  +-> pack_tcp(query) -> [2 bytes length] + message::pack()
  -> 发送 TCP 帧

接收响应后:
  +-> message::unpack(data) -> 解析 header + questions + records（处理压缩指针）
  +-> message::extract_ips() -> 遍历 answers -> extract_ipv4/extract_ipv6
```

## 参见

- [[core/resolve/dns/upstream|upstream]] -- DNS 查询客户端
- [[core/resolve/dns/detail/cache|cache]] -- DNS 结果缓存（使用 qtype）
