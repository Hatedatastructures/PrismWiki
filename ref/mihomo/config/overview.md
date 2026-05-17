---
title: "mihomo 配置参考"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/config
source: "mihomo-Memo/config/"
tags: [mihomo, config, yaml, configuration, reference]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/config/basic.yaml]]"
  - "[[ref/mihomo/config/advanced.yaml]]"
  - "[[ref/mihomo/config/tun.yaml]]"
  - "[[ref/mihomo/config/dns.yaml]]"
  - "[[ref/mihomo/config/rules.yaml]]"
  - "[[ref/mihomo/config/proxy-groups.yaml]]"
---

# mihomo 配置参考

**类别**: Mihomo 配置参考 | **模块**: YAML 配置文件

## 概述

mihomo 使用 YAML 格式配置文件，支持完整的代理配置、DNS 处理、规则分流等功能。配置文件是 mihomo 运行的核心。

### 配置结构

```yaml
# 基础配置
mixed-port: 7890           # HTTP/SOCKS5 混合端口
allow-lan: true            # 允许局域网访问
bind-address: "*"          # 绑定地址
mode: rule                 # 模式: rule/global/direct
log-level: info            # 日志级别

# DNS 配置
dns:
  enable: true
  enhanced-mode: fake-ip

# 代理节点
proxies:
  - name: "node1"
    type: trojan
    server: server.com
    port: 443

# 代理组
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - node1

# 规则
rules:
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

## 配置模块

| 模块 | 描述 | 文件 |
|------|------|------|
| 基础配置 | 端口、模式等 | [[basic.yaml]] |
| 高级配置 | 完整生产配置 | [[advanced.yaml]] |
| TUN 配置 | 透明代理配置 | [[tun.yaml]] |
| DNS 配置 | DNS 处理配置 | [[dns.yaml]] |
| 规则配置 | 分流规则 | [[rules.yaml]] |
| 代理组 | 代理组配置 | [[proxy-groups.yaml]] |
| 多路复用 | Mux 配置 | [[mux.yaml]] |
| 完整示例 | 完整配置 | [[full-example.yaml]] |

## 配置位置

| 平台 | 默认位置 |
|------|----------|
| Windows | `./config.yaml` 或 `C:\Users\...\config.yaml` |
| macOS | `./config.yaml` 或 `~/.config/mihomo/config.yaml` |
| Linux | `/etc/mihomo/config.yaml` 或 `./config.yaml` |
| Docker | `/data/config.yaml` |

## 配置验证

```bash
# mihomo 启动时会自动验证配置
mihomo -f config.yaml -t

# 通过 API 检查配置
curl http://127.0.0.1:9090/configs
```

## 子页面导航

| 页面 | 描述 |
|------|------|
| [[basic.yaml]] | 基础配置模板 |
| [[advanced.yaml]] | 高级配置模板 |
| [[tun.yaml]] | TUN 模式配置 |
| [[dns.yaml]] | DNS 配置详解 |
| [[rules.yaml]] | 规则配置详解 |
| [[proxy-groups.yaml]] | 代理组配置 |
| [[mux.yaml]] | 多路复用配置 |
| [[full-example.yaml]] | 完整配置示例 |

## 源码位置

- 配置解析: `config/config.go`
- 配置结构: `config/type.go`

## 相关链接

- [[ref/mihomo/tun/overview]] — TUN 配置参考
- [[ref/mihomo/dns/overview]] — DNS 配置参考
- [[ref/mihomo/rules/overview]] — 规则参考
- [[client/mihomo-meta]] — mihomo 概述