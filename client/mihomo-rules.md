---
title: mihomo 规则系统
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [proxy-client, mihomo, configuration]
sources:
  - mihomo 文档
confidence: high
---

# mihomo 规则系统

mihomo 的规则系统（Rule System）是流量分流的核心机制。通过定义一系列规则，决定每个网络连接走哪个代理组（或直连/拒绝），实现精细化的流量控制。

## 规则的作用

当 mihomo 收到一个网络连接请求时，按顺序匹配规则列表：

```
请求到达 → 逐条匹配规则 → 匹配成功 → 转发到对应的代理组
                           → 全部未匹配 → 使用 MATCH 兜底规则
```

规则系统的目标：
- 被墙网站走代理
- 国内网站直连（更快）
- 特定应用走特定代理组
- 广告/恶意域名直接拒绝

## 规则类型详解

### 域名规则

**DOMAIN** — 精确匹配完整域名
```yaml
- DOMAIN,www.google.com,Proxy
```
完全匹配 `www.google.com`，不含子域名。精确度最高，但需要枚举每个域名。

**DOMAIN-SUFFIX** — 匹配域名后缀
```yaml
- DOMAIN-SUFFIX,google.com,Proxy
```
匹配 `google.com` 及所有子域名（`www.google.com`、`mail.google.com` 等）。最常用的域名规则。

**DOMAIN-KEYWORD** — 域名关键词匹配
```yaml
- DOMAIN-KEYWORD,google,Proxy
```
只要域名中包含 `google` 就匹配。简单但容易误匹配（如 `notgoogle.com`）。

### GeoIP 和 GeoSite

**GEOIP** — 基于 IP 地理位置匹配
```yaml
- GEOIP,CN,Direct
```
根据 IP 归属地判断。`CN` 匹配中国大陆 IP 段。用于判断目标 IP 是否为国内地址。

**GEOSITE** — 基于域名分类匹配
```yaml
- GEOSITE,google,Proxy
- GEOSITE,cn,Direct
```
使用预编译的域名数据库。`google` 包含 Google 相关所有域名；`cn` 包含中国网站域名。数据库由社区维护。

### IP 规则

**IP-CIDR** — IPv4 CIDR 匹配
```yaml
- IP-CIDR,192.168.0.0/16,Direct,no-resolve
```
匹配目标 IP 是否在指定 CIDR 范围内。`no-resolve` 跳过 DNS 解析（仅当域名规则已处理时使用）。

**IP-CIDR6** — IPv6 CIDR 匹配
```yaml
- IP-CIDR6,2001:db8::/32,Direct,no-resolve
```
与 IP-CIDR 相同，用于 IPv6 地址。

### 来源规则

**SRC-IP-CIDR** — 源 IP 匹配
```yaml
- SRC-IP-CIDR,192.168.1.100/32,Proxy
```
匹配发起连接的客户端 IP。用于对不同设备/用户设置不同策略。

**SRC-PORT** — 源端口匹配
```yaml
- SRC-PORT,12345,Direct
```
匹配发起连接的本地端口。使用场景较少。

**DST-PORT** — 目标端口匹配
```yaml
- DST-PORT,443,Proxy
- DST-PORT,80,Proxy
```
匹配目标服务器的端口号。可用于按端口分流。

### 进程规则

**PROCESS-NAME** — 进程名匹配
```yaml
- PROCESS-NAME,chrome.exe,Proxy
- PROCESS-NAME,steam.exe,Direct
```
按进程可执行文件名匹配。仅在 TUN 模式或系统代理模式下有效。Windows 上通常是 `.exe` 文件名。

**PROCESS-PATH** — 进程路径匹配
```yaml
- PROCESS-PATH,C:\Program Files\Google\Chrome\Application\chrome.exe,Proxy
```
按进程完整路径匹配。比 PROCESS-NAME 更精确，避免同名进程混淆。

### 规则集

**RULE-SET** — 引用远程规则集
```yaml
- RULE-SET,https://example.com/rules.yaml,Proxy
```
引用 `rule-providers` 中定义的规则集。支持大量规则的批量管理。

### 逻辑规则

**AND** — 所有条件都满足
```yaml
- AND,((DOMAIN-SUFFIX,google.com),(DST-PORT,443)),Proxy
```

**OR** — 任一条件满足
```yaml
- OR,((DOMAIN-KEYWORD,google),(DOMAIN-KEYWORD,facebook)),Proxy
```

**NOT** — 条件取反
```yaml
- NOT,((DOMAIN-KEYWORD,google)),Direct
```

逻辑规则可以组合多个条件，实现复杂的分流逻辑。

### 兜底规则

**MATCH** — 匹配所有未命中的流量
```yaml
- MATCH,Proxy
```
必须放在规则列表的最后，作为默认策略。

## 规则匹配顺序

规则按**从上到下**的顺序匹配，**第一个匹配的规则生效**：

```yaml
rules:
  - DOMAIN,www.google.com,Proxy      # ① 精确匹配
  - DOMAIN-SUFFIX,google.com,Proxy   # ② 后缀匹配
  - DOMAIN-KEYWORD,google,Proxy      # ③ 关键词匹配
  - GEOIP,CN,Direct                  # ④ 国内 IP 直连
  - MATCH,Proxy                      # ⑤ 兜底
```

匹配 `www.google.com` 时，第一条规则就命中，不会再往下匹配。

### 最佳排序

1. 精确 DOMAIN 规则（最精确）
2. DOMAIN-SUFFIX 规则
3. DOMAIN-KEYWORD 规则
4. GEOSITE 规则
5. IP-CIDR / GEOIP 规则（需要 DNS 解析，放在后面）
6. 进程规则
7. MATCH 兜底

## rule-providers（远程规则集）

规则集允许从远程 URL 加载大量预编译规则：

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://example.com/reject-list.yaml"
    path: ./ruleset/reject.yaml
    interval: 86400

  direct:
    type: http
    behavior: domain
    url: "https://example.com/direct-list.yaml"
    path: ./ruleset/direct.yaml
    interval: 86400

  proxy-ip:
    type: http
    behavior: ipcidr
    url: "https://example.com/proxy-ip.yaml"
    path: ./ruleset/proxy-ip.yaml
    interval: 86400
```

### type 字段

| 值 | 说明 |
|----|------|
| `http` | 从远程 URL 下载，配合 `url` 和 `interval` |
| `file` | 本地文件，配合 `path` |

### behavior 字段

| 值 | 说明 | 规则格式 |
|----|------|---------|
| `domain` | 域名规则集 | DOMAIN / DOMAIN-SUFFIX / DOMAIN-KEYWORD |
| `ipcidr` | IP 段规则集 | IP-CIDR / IP-CIDR6 |
| `classical` | 经典规则集 | 支持所有规则类型 |

### interval

自动更新间隔，单位为秒。`86400` = 每天更新一次。

## GeoIP 和 GeoSite 数据库

mihomo 内置 GeoIP 和 GeoSite 数据库：

- **GeoIP**：IP 地址 → 国家/地区映射，基于 MaxMind 数据库
- **GeoSite**：域名 → 分类映射，社区维护（如 `geosite:cn` 包含中国常用域名）

数据库自动随 mihomo 更新，也可手动替换。

## 规则编写最佳实践

### 通用规则组合

```yaml
rules:
  # 1. 私有 IP 直连
  - IP-CIDR,127.0.0.0/8,Direct,no-resolve
  - IP-CIDR,192.168.0.0/16,Direct,no-resolve
  - IP-CIDR,10.0.0.0/8,Direct,no-resolve

  # 2. 广告拦截
  - RULE-SET,reject,REJECT

  # 3. 明确走代理的域名
  - GEOSITE,google,Proxy
  - GEOSITE,github,Proxy
  - GEOSITE,telegram,Proxy

  # 4. 国内域名直连
  - GEOSITE,cn,Direct

  # 5. 国内 IP 直连（需要 DNS 解析）
  - GEOIP,CN,Direct

  # 6. 兜底
  - MATCH,Proxy
```

### 注意事项

- `no-resolve`：避免对已有域名结果的规则重复 DNS 解析
- 规则数量影响性能，合理使用 RULE-SET 批量管理
- 进程规则仅在 TUN 或系统代理模式下有效
- 逻辑规则增加复杂度，非必要不使用

## 相关页面

- [[Mihomo-Clash-Config]] — mihomo 完整配置参考
- [[mihomo-Meta]] — mihomo 客户端概述
- [[protocol/proxy-protocols]] — 代理协议概览
- [[client/mihomo-dns]] — DNS 解析与规则的关系
- [[dev/gfw]] — 防火长城，理解为什么需要规则分流
