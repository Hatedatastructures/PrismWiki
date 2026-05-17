---
title: "mihomo relay 代理组"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/parser.go"
tags: [mihomo, proxy-groups, relay, 链式代理]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo relay 代理组

relay 类型代理组已在新版本中移除，推荐使用 dialer-proxy 替代。

## 重要说明

根据源码 `parser.go` 第 188 行：

```go
case "relay":
    return nil, fmt.Errorf("%w: The group [%s] with relay type was removed, please using dialer-proxy instead", errType, groupName)
```

relay 代理组已被移除，请使用节点级别的 `dialer-proxy` 配置替代。

## 替代方案：dialer-proxy

### YAML 配置示例

```yaml
proxies:
  # 入口节点
  - name: "入口节点"
    type: vless
    server: entry.example.com
    port: 443
    
  # 中间节点（通过入口节点连接）
  - name: "中间节点"
    type: vless
    server: middle.example.com
    port: 443
    dialer-proxy: "入口节点"
    
  # 出口节点（通过中间节点连接）
  - name: "出口节点"
    type: vless
    server: exit.example.com
    port: 443
    dialer-proxy: "中间节点"
```

### 链式代理效果

```
客户端 → 入口节点 → 中间节点 → 出口节点 → 目标服务器
```

## dialer-proxy 字段说明

| 字段 | 说明 |
|------|------|
| `dialer-proxy` | 连接该节点时使用的代理 |

## 原 relay 工作原理

relay 曾实现多跳代理链：

```
客户端 → 入口节点 → 中间节点 → 出口节点 → 目标
```

隐私优势：
- 入口节点只知道客户端 IP
- 中间节点只知道前后节点
- 出口节点只知道目标地址

## 迁移建议

1. **单跳替代**：直接使用 `dialer-proxy` 配置节点链
2. **性能考虑**：每增加一跳会增加延迟
3. **稳定性**：确保每一跳节点都稳定

## 相关文档

- [[overview]] - 代理组概述
- [[../protocols/vless|VLESS 协议]]
- [[../protocols/vmess|VMess 协议]]