---
title: "Script 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/script"
tags: [mihomo, script, 脚本, lua, rule-script, shortcuts]
created: 2026-05-17
updated: 2026-05-17
related: [shortcuts, rule-script]
---

# Script 概览

**类别**: Mihomo 脚本功能参考

## 概述

Script（脚本）配置允许通过脚本扩展 Mihomo 功能。脚本可以修改节点配置、动态处理规则、实现自定义逻辑。

### 脚本功能

| 功能 | 说明 |
|------|------|
| [[ref/mihomo/script/shortcuts|Shortcuts]] | 节点配置快捷修改 |
| [[ref/mihomo/script/rule-script|Rule Script]] | 规则脚本处理 |

## 配置位置

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"
  code: |
    def main(ctx, metadata):
        # 规则脚本逻辑
        return "PROXY"
```

## 脚本类型

### Shortcuts

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"
    dns: "skip-cert-verify=true"
```

节点配置快捷修改，通过标签应用预设配置。

### Rule Script

```yaml
script:
  code: |
    def main(ctx, metadata):
        if metadata["host"] == "example.com":
            return "DIRECT"
        return "PROXY"
```

规则脚本，动态处理每个连接。

## 脚本语言

Mihomo 支持以下脚本语言：

| 语言 | 用途 | 说明 |
|------|------|------|
| YAML | Shortcuts | 配置快捷方式 |
| Python | Rule Script | 内嵌 Python |
| Lua | Rule Script | 内嵌 Lua |

## 使用场景

### Shortcuts 场景

- 快速修改节点配置
- 批量应用相同设置
- 简化配置管理

### Rule Script 场景

- 动态规则处理
- 复杂条件判断
- 自定义路由逻辑

## 配置示例

### Shortcuts 基础

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"

proxies:
  - name: ss-quic
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
    shortcuts: quic
```

### Rule Script 基础

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        if host.endswith(".cn"):
            return "DIRECT"
        return "PROXY"

rules:
  - SCRIPT,my-script,PROXY
```

### 完整配置

```yaml
script:
  shortcuts:
    quic: "udp-over-tcp=true&udp-over-tcp-version=2"
    dns: "skip-cert-verify=true"
  code: |
    def main(ctx, metadata):
        # 规则脚本逻辑
        return "PROXY"
```

## 脚本执行流程

```
脚本执行流程：
┌─────────────────────────────────────────────┐
│                                             │
│  连接到达                                    │
│      │                                      │
│      │ 匹配 SCRIPT 规则                      │
│      ▼                                      │
│  调用脚本                                    │
│      │                                      │
│      │ 传入 metadata                        │
│      │   - host                             │
│      │   - process                          │
│      │   - src_ip                           │
│      │   - ...                              │
│      ▼                                      │
│  脚本执行                                    │
│      │                                      │
│      │ 返回结果                              │
│      │   - DIRECT                           │
│      │   - REJECT                           │
│      │   - PROXY                            │
│      │   - 代理组名称                        │
│      ▼                                      │
│  应用路由                                    │
│                                             │
└─────────────────────────────────────────────┘
```

## 性能考量

脚本执行会产生开销：

| 因素 | 影响 |
|------|------|
| 脚本复杂度 | 执行时间 |
| 脚本数量 | 内存占用 |
| 调用频率 | CPU 开销 |

建议：
- 简化脚本逻辑
- 避免复杂计算
- 使用内置规则优先

## 相关链接

- [[ref/mihomo/script/shortcuts|Shortcuts]] — 节点快捷配置详解
- [[ref/mihomo/script/rule-script|Rule Script]] — 规则脚本详解