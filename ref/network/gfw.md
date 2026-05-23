---
title: "GFW 原理"
category: "network"
type: ref
layer: ref
module: ref
source: []
tags: [网络, gfw, 防火墙, 审查, dpi, 封锁]
created: 2026-05-17
updated: 2026-05-17
---

# GFW 原理

**类别**: 网络

## 概述

防火墙（GFW，Great Firewall）是中国互联网审查系统的俗称，通过多种技术手段限制特定网络内容的访问。了解 GFW 的检测原理对于设计有效的代理方案至关重要。

### 检测技术概览

GFW 使用的主要检测技术：

| 技术 | 说明 | 检测对象 |
|------|------|----------|
| **IP 封锁** | 黑名单 IP 无法访问 | 服务器、代理 |
| **DNS 污染** | 返回错误 DNS 结果 | 域名解析 |
| **SNI 封锁** | 检测 TLS ClientHello SNI | HTTPS 连接 |
| **DPI** | 深度包检测协议特征 | 代理协议 |
| **主动探测** | 连接可疑服务验证 | 代理服务器 |
| **流量分析** | 分析流量统计特征 | 加密流量 |

### 检测层级

GFW 在不同层级的检测：

```
检测层级：

┌─────────────────────────────────────────────┐
│ 应用层 (Layer 7)                             │
│ - HTTP 内容审查                              │
│ - 协议特征识别                               │
│ - 主动探测                                   │
├─────────────────────────────────────────────┤
│ TLS 层                                       │
│ - SNI 检测                                   │
│ - TLS 指纹识别                               │
│ - 证书检测                                   │
├─────────────────────────────────────────────┤
│ 传输层 (Layer 4)                             │
│ - TCP 连接跟踪                               │
│ - 端口封锁                                   │
├─────────────────────────────────────────────┤
│ 网络层 (Layer 3)                             │
│ - IP 封锁                                    │
│ - 路由黑洞                                   │
├─────────────────────────────────────────────┤
│ DNS 层                                       │
│ - DNS 污染                                   │
│ - DNS 查询拦截                               │
└─────────────────────────────────────────────┘
```

## IP 封锁

### 封锁机制

IP 封锁的工作方式：

```
IP 封锁机制：

黑名单：
- 维护被封 IP 列表
- 定期更新
- 包括代理服务器、敏感网站

封锁方式：
- 路由黑洞（BGP）
- ACL 拒绝
- TCP RST 注入

效果：
- 连接超时（黑洞）
- 连接被拒（ACL）
- 连接被重置（RST）

检测触发：
- DPI 检测到代理协议
- 主动探测确认代理
- 人工添加黑名单
```

### 封锁特点

IP 封锁的特点：

```
IP 封锁特点：

影响范围：
- 整个 IP 无法访问
- 同 IP 的其他服务受影响
- 云服务/VPS IP 易被封

绕过方法：
- 使用 CDN（IP 混合）
- 前置代理（隐藏 IP）
- 动态 IP（更换 IP）

封封锁时效：
- 可能长期封锁
- 可能周期性封锁
- 可能临时封锁（高峰时段）
```

## DNS 污染

### 污染机制

DNS 污染的工作方式：

```
DNS 污染机制：

原理：
- 监听 DNS 查询（UDP 53）
- 匹配黑名单域名
- 返回错误 IP 地址

流程：
客户端 → DNS 查询 (UDP)
         ↓
GFW 检测到黑名单域名
         ↓
GFW → 返回错误 IP (伪造响应)
         ↓
客户端 → 收到错误 IP
         ↓
客户端 → 连接错误地址

特点：
- 比 DNS 查询响应更快
- 无需等待上游响应
- 直接伪造响应
```

### 污染结果

DNS 污染的返回结果：

```
污染结果：

典型污染 IP：
- 国内无效 IP
- 广告 IP
- 返回 IP 无响应

检测方法：
dig www.example.com @8.8.8.8
返回结果与预期不符

绕过方法：
- DNS-over-TLS (DoT)
- DNS-over-HTTPS (DoH)
- 使用境外 DNS（UDP 可能被污染）
- 本地缓存正确 IP
```

## SNI 封锁

### SNI 检测

TLS ClientHello SNI 检测：

```
SNI 检测机制：

TLS ClientHello 结构：
┌─────────────────────────────────────────────┐
│ Handshake Type: ClientHello (0x01)          │
│ Random (32 bytes)                           │
│ Session ID                                  │
│ Cipher Suites                               │
│ Compression Methods                         │
│ Extensions:                                 │
│   - server_name (SNI): www.example.com      │ ← 检测点
│   - supported_groups                       │
│   - key_share                               │
│   ...                                       │
└─────────────────────────────────────────────┘

检测流程：
1. 检测到 TLS ClientHello
2. 解析 SNI 扩展
3. 匹配黑名单域名
4. 触发封锁（RST 或 IP 封锁）

封锁方式：
- TCP RST 注入（立即断开）
- IP 封锁（后续连接失败）
```

### SNI 绕过

绕过 SNI 封锁的方法：

```
SNI 绕过方法：

方法 1: 前置域名（Domain Fronting）
- SNI 使用允许的域名
- HTTP Host 使用真实域名
- 问题：服务商已禁用

方法 2: Encrypted Client Hello (ECH)
- 加密整个 ClientHello
- SNI 不明文可见
- 问题：部署有限

方法 3: Reality
- SNI 使用真实网站域名
- 连接真实网站作为掩护
- Reality 秘密验证客户端
- 详见 [[core/stealth/reality/handshake|reality]]

方法 4: ShadowTLS
- 使用真实 TLS 握手作为掩护
- 秘密传递代理认证
- 详见 [[core/stealth/shadowtls|shadowtls]]
```

## DPI 检测

### 协议特征检测

深度包检测代理协议：

```
DPI 协议特征检测：

检测对象：
- Shadowsocks 流量特征
- V2Ray 协议特征
- Trojan 协议特征
- 其他代理协议

检测方法：
- 流量统计特征
- 协议模式识别
- 特定字节序列

Shadowsocks 检测：
- 固定长度握手
- 特定加密特征
- 流量统计分析

Trojan 检测：
- TLS + 特定认证格式
- 流量特征分析

绕过方法：
- Reality（伪装为正常 TLS）
- ShadowTLS（伪装 TLS 握手）
- 流量混淆
```

详见 [[ref/anti-censorship/dpi|DPI]]。

### TLS 指纹检测

TLS 指纹识别客户端：

```
TLS 指纹检测：

指纹内容：
- Cipher Suites 顺序
- Extensions 顺序和内容
- Supported Groups
- Signature Algorithms

JA3 签名：
MD5(ClientHello Version + CipherSuites + Extensions + SupportedGroups + SignatureAlgorithms)
示例：ja3 = "769,47-53-5-10-49-11-..."

检测逻辑：
- 正常浏览器：已知 JA3 值
- 代理客户端：非标准 JA3 值
- 标记异常指纹

绕过方法：
- 使用标准 TLS 库
- 模拟浏览器指纹
- Reality 使用真实 TLS
```

详见 [[ref/anti-censorship/tls-fingerprint|TLS 指纹]]。

## 主动探测

### 探测机制

主动探测可疑服务：

```
主动探测机制：

触发条件：
- DPI 检测到可疑流量
- 统计特征异常
- 端口行为异常

探测方式：
- 发送模拟客户端请求
- 尝试建立 TLS 连接
- 发送协议探测包

判断逻辑：
- 正常网站：返回标准响应
- 代理服务：返回代理响应或无响应
- 确认代理后封锁

探测特征：
- 特定 IP 来源
- 特定 TLS 指纹
- 特定探测模式
```

### 探测绕过

绕过主动探测的方法：

```
探测绕过方法：

方法 1: 验证真实客户端
- Reality: 验证客户端是真实用户
- 不响应探测请求
- 详见 [[core/stealth/reality/handshake|reality]]

方法 2: 伪装为正常服务
- 返回标准 HTTP/HTTPS 响应
- 不暴露代理特征

方法 3: 检测探测特征
- 检测探测 IP
- 检测探测 TLS 指纹
- 拒绝探测连接
```

## 流量分析

### 统计特征分析

流量统计分析：

```
流量统计分析：

分析维度：
- 包大小分布
- 包间隔时间
- 流量速率变化
- 连接持续时间

特征：
- 加密流量有特定统计特征
- 代理流量与正常流量不同
- 长连接特征明显

检测方法：
- 机器学习模型
- 统计阈值
- 时序分析

绕过方法：
- 流量混淆
- 模拟正常流量模式
- 分散流量
```

详见 [[ref/anti-censorship/traffic-analysis|流量分析]]。

## 在 Prism 中的应用

### Stealth 模块

Prism 的反审查方案：

```
Prism Stealth 模块：

方案：
- Reality: TLS 伪装，使用真实网站掩护
- ShadowTLS: TLS 握手伪装
- Restls: REST API 伪装

设计考量：
- 绕过 SNI 封锁
- 绕过 DPI 检测
- 绕过主动探测
- 模拟正常 TLS 流量

Reality 核心思想：
- SNI 使用真实网站域名
- 连接真实网站作为掩护
- 秘密验证客户端
- 通过加密隧道传输代理数据
```

详见 [[core/stealth/reality/handshake|Reality]]。

### 检测规避策略

Prism 的检测规避策略：

```
检测规避策略：

SNI 封锁规避：
- Reality: SNI 是真实网站
- ShadowTLS: TLS 握手伪装

DPI 规避：
- Reality: 看起来是正常 TLS
- 无代理协议特征

主动探测规避：
- Reality: 验证客户端
- 探测无法获得代理响应

流量分析规避：
- 使用真实 TLS 流量模式
- 难以区分代理和正常访问
```

## 参见

- [[core/stealth/reality/handshake|Reality]] — Reality TLS 伪装
- [[core/stealth/shadowtls|ShadowTLS]] — ShadowTLS 伪装
- [[ref/anti-censorship/dpi|DPI]] — 深度包检测
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹检测
- [[ref/anti-censorship/traffic-analysis|流量分析]] — 流量分析
- [[ref/network/proxy-detection|代理检测]] — 代理检测原理
- [[ref/network/overview|网络基础概览]] — 网络索引