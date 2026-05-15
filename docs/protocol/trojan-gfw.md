---
title: "Trojan-GFW 协议实现"
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags: [trojan, tls, proxy, gfw, camouflage]
related: ["[[protocol/trojan]]", "[[protocol/vless]]", "[[protocol/shadowsocks]]", "[[stealth/reality]]", "[[dev/tls]]", "[[dev/gfw]]", "[[docs/protocol/proxy-protocols]]", "[[client/mihomo-meta]]"]
---

> 相关：[[protocol/trojan]] | [[protocol/vless]] | [[protocol/shadowsocks]] | [[stealth/reality]] | [[dev/tls]] | [[dev/gfw]] | [[docs/protocol/proxy-protocols]] | [[client/mihomo-meta]]

# Trojan-GFW 协议实现

> 原始 Trojan 协议（trojan-gfw/trojan）的规范解析，以及与 Prism 实现的对比。

---

## 1. 什么是 Trojan-GFW

Trojan-GFW 是由 **GreaterFire** 于 2018 年发布的代理协议及其实现（[GitHub 仓库](https://github.com/trojan-gfw/trojan)）。它是"Trojan"这个名字的原始出处——后续出现的 Trojan-Go 等变体均为社区 fork 或重写。

核心设计思想：**让代理流量与普通 HTTPS 流量对任何外部观察者不可区分**。与 Shadowsocks 用自定义密码加密流量不同，Trojan 直接使用标准 TLS 作为传输层，在 TLS 隧道内部以极简的应用层协议传输代理命令。

---

## 2. 协议格式

### 2.1 整体结构

```
+-------------------------------------------------------+
|                 TCP 连接                               |
+-------------------------------------------------------+
|              TLS 1.2+ 隧道（标准握手）                  |
+-------------------------------------------------------+
|           Trojan 应用层（认证 + 命令 + 数据）            |
+-------------------------------------------------------+
```

### 2.2 握手请求格式

TLS 握手完成后，客户端发送第一个应用层数据，格式如下：

```
偏移   长度    字段
------ ------ -----
  0      56    凭据（密码的 SHA224 十六进制摘要，ASCII 字符）
 56       2    CRLF（0x0D 0x0A）
 58       1    CMD（命令字节）
 59       1    ATYP（地址类型）
 60      可变  目标地址
 60+N     2    目标端口（大端序）
 60+N+2   2    CRLF（0x0D 0x0A）
```

**关键字段说明：**

| 字段 | 值 | 含义 |
|------|-----|------|
| 凭据 | 56 字节 ASCII hex | 密码的 `SHA224(password)` 输出，用小写十六进制字符串表示 |
| CMD | `0x01` | CONNECT — TCP 代理 |
| CMD | `0x03` | UDP_ASSOCIATE — UDP 代理 |
| ATYP | `0x01` | IPv4（4 字节） |
| ATYP | `0x03` | 域名（1 字节长度 + N 字节域名） |
| ATYP | `0x04` | IPv6（16 字节） |

### 2.3 为什么用 SHA224

SHA224 输出 28 字节，十六进制编码后恰好 56 个 ASCII 字符。选择 SHA224 而非 SHA256 的原因是：

1. **固定长度前缀**：56 字节固定长度使得协议头部具有可预测结构，便于服务端快速校验
2. **安全性足够**：密码哈希仅用于认证而非加密，SHA224 的抗碰撞能力完全满足需求
3. **与 HTTP 请求体相似**：56 个十六进制字符在外观上很像随机的 POST 请求体，增加伪装性

### 2.4 CRLF 分隔符的作用

Trojan 使用 `\r\n`（CRLF）作为字段分隔符，这有两个目的：

1. **与 HTTP 请求对齐**：CRLF 是 HTTP 协议的标准行终止符，使得 Trojan 请求头在字节层面更像 HTTP 请求体
2. **快速边界检测**：服务端可以扫描 `\r\n` 来快速定位 CMD 字节的位置

### 2.5 服务端响应

Trojan 协议在认证成功后**不发送任何协议级响应**：

- **成功**：服务端直接开始中继目标服务器的数据，客户端收到的第一个字节就是目标服务器的响应
- **失败**：服务端直接关闭连接（或转发到诱饵 Web 服务器）

这是与 SOCKS5 的关键区别——SOCKS5 在 CONNECT 成功后会发送一个包含状态码的响应头。

---

## 3. 与 Prism 实现的关系

### 3.1 协议兼容性

Prism 的 Trojan 实现与 Trojan-GFW **完全协议兼容**。任何符合 Trojan-GFW 规范的客户端（如 Clash/Mihomo、Shadowrocket、v2rayNG）可以直接连接 Prism 服务器，无需任何修改。

### 3.2 Prism 的实现增强

在协议格式不变的前提下，Prism 在工程实现上做了以下增强：

| 维度 | Trojan-GFW (C++14) | Prism (C++23) |
|------|---------------------|---------------|
| 异步模型 | `boost::asio::spawn` 协程 | C++23 原生协程 + `co_await` |
| 内存管理 | `new`/`delete` + `std::vector` | PMR 内存池（`std::pmr::polymorphic_allocator`） |
| 多路复用 | 不支持 | smux/yamux 兼容 MUX 命令（`cmd=0x7F`） |
| UDP 帧 | 原始格式 | 兼容 Mihomo 的 Length+CRLF 帧编码 |
| 伪装方案 | 仅标准 TLS | Reality / ShadowTLS / 原生 TLS 三层方案 |
| TLS 库 | OpenSSL | BoringSSL |
| 错误处理 | 异常 + 错误码 | `std::expected` + fault 体系 |

### 3.3 Prism 的 MUX 扩展

Trojan-GFW 原始规范仅定义了 `CONNECT (0x01)` 和 `UDP_ASSOCIATE (0x03)` 两个命令。Prism 新增了 `MUX (0x7F)` 命令，用于在单个 TLS 连接上承载多个逻辑流：

```
原始 Trojan-GFW:
  客户端 --[CONNECT]--> 服务器 --[中继]--> 目标
  （每个连接一条 TLS 隧道）

Prism MUX 扩展:
  客户端 --[MUX]--> 服务器 --[smux/yamux]--> 多个逻辑流
  （一条 TLS 隧道承载多个连接）
```

此外，Prism 还兼容 Mihomo 的 MUX 伪装模式：客户端发送 `cmd=0x01`（CONNECT），但目标地址以 `.mux.sing-box.arpa` 结尾。Prism 的 `primitives::is_mux_target()` 检测此特殊后缀，自动路由到多路复用模式。

详见 [[protocol/trojan]] 中的 MUX 命令扩展章节。

---

## 4. TLS 指纹考量

### 4.1 JA3/JA4 指纹

Trojan-GFW 使用标准 TLS 库（通常是 OpenSSL），其 TLS ClientHello 会携带典型的密码套件列表和扩展组合。DPI 系统可以通过 JA3/JA4 指纹识别出这是代理客户端而非普通浏览器。

**缓解方案：**

| 方案 | 说明 |
|------|------|
| uTLS | 模拟浏览器的 TLS 指纹（Chrome、Firefox 等） |
| Reality | [[stealth/reality]] — 使用真实网站的证书，客户端 TLS 指纹伪装为目标浏览器 |
| XTLS | 在 TLS 握手后切换到更高效的传输模式 |

Prism 的 Reality 集成允许 Trojan 流量使用伪造的 TLS 证书，这些证书在证书透明度日志中有对应的合法网站，使得基于证书的检测完全失效。

### 4.2 TLS 版本与密码套件

Trojan-GFW 原始规范要求 TLS 1.2+。推荐使用：

- TLS 1.3（首选）：更少的握手轮次，更强的密码套件
- TLS 1.2 + AEAD 套件（如 `TLS_AES_256_GCM_SHA384`）

Prism 使用 BoringSSL 作为 TLS 库，默认协商到 TLS 1.3。

---

## 5. 与其他协议的对比

### 5.1 Trojan-GFW vs VLESS

| 维度 | Trojan-GFW | VLESS |
|------|-----------|-------|
| 认证 | SHA224 密码哈希 (56B ASCII) | UUID 原始字节 (16B) |
| 请求头最小尺寸 | 68 字节 | 26 字节 |
| 文本 vs 二进制 | 半文本（hex + CRLF） | 纯二进制 |
| 内置加密 | 无（依赖 TLS） | 无（依赖 TLS） |
| 伪装性 | 更高（CRLF + hex 像 HTTP body） | 较低（纯二进制头） |
| ATYP Domain | `0x03` | `0x02` |
| ATYP IPv6 | `0x04` | `0x03` |
| UDP 支持 | Length + CRLF 帧格式 | ATYP+ADDR+PORT+Payload |
| MUX 命令 | `0x7F`（Prism 扩展） | `0x7F` |

### 5.2 Trojan-GFW vs Shadowsocks

| 维度 | Trojan-GFW | Shadowsocks |
|------|-----------|-------------|
| 传输层 | TLS（必须） | TCP/UDP（无 TLS） |
| 加密 | TLS 提供 | AEAD（AES-256-GCM / ChaCha20-Poly1305） |
| 证书 | 需要 | 不需要 |
| DPI 抗性 | 高（伪装 HTTPS） | 中（加密但可统计分析） |
| 性能 | TLS 握手开销较高 | 无 TLS 握手，延迟更低 |
| 主动探测 | 高抗性（不响应错误凭据） | 中等（需 Shadowsocks 协议特征） |

---

## 6. 部署与兼容性

### 6.1 兼容客户端

以下客户端可直接连接 Prism 的 Trojan 端口：

| 客户端 | 平台 | MUX 支持 |
|--------|------|----------|
| [[client/mihomo-meta]] (Clash Meta) | 全平台 | 是（smux） |
| Shadowrocket | iOS | 否 |
| v2rayNG | Android | 否 |
| Trojan-Qt5 | Windows/macOS/Linux | 否 |
| sing-box | 全平台 | 是（smux） |

### 6.2 配置要点

Prism 中启用 Trojan 协议的最小配置：

```yaml
inbound:
  - type: trojan
    port: 443
    tls:
      cert: /path/to/cert.pem
      key: /path/to/key.pem
    users:
      - password: "your-password"  # SHA224 哈希在内部自动计算
```

客户端（Mihomo）对应配置参见 [[client/mihomo-meta]]。

---

## 7. 相关页面

- [[protocol/trojan]] — Prism 中的 Trojan 协议完整实现文档
- [[protocol/vless]] — VLESS 协议规范
- [[protocol/shadowsocks]] — Shadowsocks 协议规范
- [[stealth/reality]] — Reality 伪装方案
- [[dev/tls]] — TLS 实现细节（BoringSSL）
- [[dev/gfw]] — GFW 对抗技术笔记
- [[docs/protocol/proxy-protocols]] — 代理协议总览
- [[client/mihomo-meta]] — Mihomo/Clash Meta 客户端对接
