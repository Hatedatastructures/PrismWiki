---
title: Reject
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, reject]
---
# Reject 拒绝协议

Reject 是 mihomo 中的拒绝连接协议类型，用于主动拦截并拒绝特定的网络连接请求。它在代理规则系统中扮演"黑名单终端"的角色，与 Direct（直连）和各类代理协议共同构成完整的流量控制体系。

## 技术概述

Reject 协议不建立任何实际网络连接，而是在 mihomo 的规则引擎层面直接拦截请求。根据具体的拒绝策略，它可以返回错误响应或静默丢弃数据包。其核心特性包括：

- **REJECT**：返回连接拒绝响应，客户端收到明确的错误通知
- **REJECT-DROP**：静默丢弃连接，不返回任何响应（类似黑洞）
- **PASS**：放行连接，主要用于调试和规则测试
- **DNS 级拦截**：在 DNS 解析阶段即拦截恶意域名查询
- **零资源消耗**：不建立任何 socket 连接，无网络开销

Reject 协议在 mihomo 的出站适配器体系中属于"拦截终端"——它不是连接的中转站，而是连接的终止点。

## 拒绝机制

### REJECT（明确拒绝）

`REJECT` 策略会在连接建立阶段返回明确的拒绝信号：

- **TCP 连接**：向客户端返回 RST（Reset）包，立即终止连接
- **UDP 连接**：返回 ICMP Port Unreachable 或类似的错误响应
- **HTTP 请求**：返回 `503 Service Unavailable` 或 `403 Forbidden`

```
Client (应用)          mihomo
    |  SYN                |
    |------------------->|
    |                     |  规则匹配 → REJECT
    |  RST                |
    |<-------------------|
    |  连接被拒绝          |
```

客户端应用会收到一个明确的连接失败错误，可以用于调试和日志记录。

### REJECT-DROP（静默丢弃）

`REJECT-DROP` 策略更加隐蔽，它不返回任何响应：

- **TCP 连接**：不回复 SYN 包，客户端等待超时（通常 30-120 秒）
- **UDP 连接**：直接丢弃数据包，无任何响应
- **DNS 查询**：返回 NXDOMAIN（域名不存在）或空响应

```
Client (应用)          mihomo
    |  SYN                |
    |------------------->|
    |                     |  规则匹配 → REJECT-DROP
    |                     |  （无响应，客户端等待超时）
    |  ... 超时 ...        |
```

REJECT-DROP 模拟了"目标服务器不存在或不可达"的效果，使客户端无法区分是服务器宕机还是被主动拦截。这种机制在广告拦截和隐私保护场景中尤为重要，因为它不会向广告SDK暴露"该域名被拦截"的信息。

### PASS（调试放行）

`PASS` 是一个特殊的 Reject 变体，用于调试：

- 匹配 PASS 规则的流量不会被拒绝，也不会被代理
- 通常用于测试规则匹配是否正确
- 在日志中会标记为 PASS，便于排查

## 拒绝类型对比

### 详细对比表

| 特性 | REJECT | REJECT-DROP | PASS |
|------|--------|-------------|------|
| TCP 响应 | 返回 RST | 无响应（超时） | 放行 |
| UDP 响应 | 返回错误 | 静默丢弃 | 放行 |
| DNS 响应 | NXDOMAIN | NXDOMAIN / 空 | 正常解析 |
| 客户端感知 | 明确拒绝 | 像服务器不存在 | 正常连接 |
| 超时时间 | 立即 | 30-120 秒 | N/A |
| 日志记录 | 记录 | 记录 | 记录 |
| 适用场景 | 调试、测试 | 广告拦截、安全 | 调试规则 |

### 选择指南

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 广告域名拦截 | `REJECT-DROP` | 静默丢弃，广告SDK无法感知 |
| 恶意/钓鱼域名 | `REJECT-DROP` | 彻底阻断，不泄露拦截信息 |
| 跟踪器/分析域名 | `REJECT-DROP` | 防止指纹追踪 |
| 调试规则匹配 | `REJECT` | 明确的拒绝反馈 |
| 测试规则优先级 | `PASS` | 验证规则是否正确命中 |
| 已知安全漏洞域名 | `REJECT-DROP` | 防止恶意连接建立 |

## 使用场景

### 广告拦截

广告拦截是 Reject 协议最常见的应用场景。通过匹配广告域名的规则，将所有广告请求拦截：

```yaml
rules:
  # 广告域名拦截
  - DOMAIN-SUFFIX,adserver.com,REJECT-DROP
  - DOMAIN-SUFFIX,ads.example.com,REJECT-DROP
  - DOMAIN-KEYWORD,ad_,REJECT-DROP
  - DOMAIN-KEYWORD,tracking,REJECT-DROP

  # 常见广告 SDK 域名
  - DOMAIN-SUFFIX,doubleclick.net,REJECT-DROP
  - DOMAIN-SUFFIX,googlesyndication.com,REJECT-DROP
  - DOMAIN-SUFFIX,facebook.com/ad,REJECT-DROP
```

使用 `REJECT-DROP` 而非 `REJECT` 的原因是：广告SDK在收到明确的拒绝响应后，可能会尝试更换域名或调整请求策略，而静默丢弃则使SDK误认为域名暂时不可用，从而降低重试频率。

### 恶意域名拦截

安全场景下，Reject 用于拦截已知的恶意域名：

```yaml
rules:
  # 钓鱼网站
  - DOMAIN-SUFFIX,phishing-example.com,REJECT-DROP
  - DOMAIN-SUFFIX,malware-c2.example.net,REJECT-DROP

  # 僵尸网络 C2 服务器
  - DOMAIN-SUFFIX,botnet-controller.example,REJECT-DROP

  # 挖矿池
  - DOMAIN-SUFFIX,crypto-pool.example,REJECT-DROP
```

### DNS 污染拦截

当 DNS 查询匹配到 Reject 规则时，mihomo 可以在 DNS 层面即进行拦截：

```yaml
rules:
  # DNS 级拦截
  - DOMAIN,bad.example.com,REJECT-DROP
```

拦截流程：

```
应用发起 DNS 查询
    ↓
mihomo DNS 服务器接收
    ↓
查询规则引擎
    ↓
匹配到 REJECT 规则
    ↓
返回 NXDOMAIN（域名不存在）
    ↓
应用收到"域名不存在"的错误
```

这种机制的优势在于：

1. **更早拦截**：在连接建立之前即拦截，减少资源浪费
2. **更彻底**：即使应用尝试直接 IP 连接，DNS 层面的拦截也能确保域名无法被解析
3. **缓存友好**：NXDOMAIN 响应可以被 DNS 缓存，减少重复查询

### 隐私保护

拦截跟踪器和指纹收集域名：

```yaml
rules:
  # 分析跟踪
  - DOMAIN-SUFFIX,analytics.google.com,REJECT-DROP
  - DOMAIN-SUFFIX,appsflyer.com,REJECT-DROP
  - DOMAIN-SUFFIX,adjust.com,REJECT-DROP

  # 遥测
  - DOMAIN-SUFFIX,telemetry.microsoft.com,REJECT-DROP
  - DOMAIN-SUFFIX,smartscreen-prod.microsoft.com,REJECT-DROP
```

## 规则优先级

Reject 规则在规则列表中的位置很重要。mihomo 按顺序匹配规则，第一条匹配的规则生效：

```yaml
rules:
  # 优先级高：先处理拦截规则
  - DOMAIN-SUFFIX,ads.example.com,REJECT-DROP
  - DOMAIN-SUFFIX,tracking.example.com,REJECT-DROP

  # 中等优先级：代理规则
  - DOMAIN-SUFFIX,example.com,PROXY

  # 最低优先级：兜底直连
  - MATCH,DIRECT
```

如果代理规则写在 Reject 规则之前，流量可能先被代理转发而无法被拦截。因此，最佳实践是将所有 Reject 规则放在规则列表的最前面。

## DNS 级拦截详解

### 配置方式

通过 `dns.reject` 配置可以实现 DNS 级别的域名拦截：

```yaml
dns:
  enable: true
  reject:
    - "geosite:category-ads-all"
    - "geosite:category-porn"
    - "example-bad-domain.com"
```

被 `dns.reject` 匹配的域名将在 DNS 解析阶段被拦截，永远不会触发连接建立。这比在 `rules` 中使用 `REJECT` 更加高效和彻底。

### 与 Rules 中 REJECT 的区别

| 特性 | dns.reject | rules → REJECT |
|------|------------|---------------|
| 拦截阶段 | DNS 解析 | 连接建立 |
| 返回结果 | NXDOMAIN | RST / 超时 |
| IP 连接 | 无法拦截（直接 IP） | 可以拦截（IP-CIDR 规则） |
| 性能 | 更高（更早拦截） | 稍低（需要建立连接后拦截） |
| 灵活性 | 较低（仅域名） | 更高（域名、IP、协议等） |

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/reject.go` | Reject 适配器，实现三种拒绝策略 |

Reject 适配器的实现非常轻量：

1. 根据代理名称（`REJECT`、`REJECT-DROP`、`PASS`）判断策略
2. `REJECT`：返回一个立即 EOF 的虚拟连接
3. `REJECT-DROP`：返回一个不读写的空连接，或直接丢弃
4. `PASS`：将请求传递给下一层处理

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Reject 实现 |
| Pipeline | 完全兼容 | 虚拟连接 |
| Rules | 完全兼容 | 规则匹配 |
| DNS | 完全兼容 | DNS 级拦截 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "REJECT"
    type: reject

  - name: "REJECT-DROP"
    type: reject

  - name: "PASS"
    type: reject
```

### 广告拦截规则

```yaml
rules:
  # 广告拦截
  - DOMAIN-SUFFIX,adserver.com,REJECT-DROP
  - DOMAIN-SUFFIX,googleadservices.com,REJECT-DROP
  - DOMAIN-KEYWORD,-ad.,REJECT-DROP
  - DOMAIN-KEYWORD,.ads.,REJECT-DROP

  # 跟踪器
  - DOMAIN-SUFFIX,facebook.com/tr,REJECT-DROP
  - DOMAIN-SUFFIX,google-analytics.com,REJECT-DROP

  # 兜底
  - MATCH,PROXY
```

## 相关文档

- [[direct]] - Direct 协议
- [[dns]] - DNS 代理
- [[ref/mihomo/rules|rules]] - 规则配置
- [[ref/mihomo/dns|dns]] - DNS 配置
