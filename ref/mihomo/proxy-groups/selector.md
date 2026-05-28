---
title: "mihomo select 代理组"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/selector.go"
tags: [mihomo, proxy-groups, select, 手动选择]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo select 代理组

select 类型代理组允许用户手动选择节点。

## 源码位置

- `adapter/outboundgroup/selector.go`

## Selector 结构

```go
type Selector struct {
    *GroupBase
    disableUDP bool
    selected   string
    testUrl    string
    Hidden     bool
    Icon       string
}
```

## YAML 配置示例

### 基本配置

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

### 包含特殊选项

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "自动选择"
      - "负载均衡"
      - DIRECT
      - REJECT
```

### 使用 provider

```yaml
proxy-groups:
  - name: "手动选择"
    type: select
    use:
      - provider1
      - provider2
    filter: "香港|日本"
```

### 完整配置

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点A"
      - "节点B"
    disable-udp: false
    hidden: false
    icon: "https://example.com/icon.png"
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理组名称 |
| `type` | string | 是 | 固定为 `select` |
| `proxies` | []string | 否* | 节点列表 |
| `use` | []string | 否* | 引用的 provider |
| `filter` | string | 否 | 节点过滤正则 |
| `disable-udp` | bool | 否 | 禁用 UDP |
| `hidden` | bool | 否 | 面板隐藏 |
| `icon` | string | 否 | 面板图标 URL |

*`proxies` 或 `use` 至少需要填一个

## 核心方法

### DialContext

```go
func (s *Selector) DialContext(ctx context.Context, metadata *C.Metadata) (C.Conn, error) {
    c, err := s.selectedProxy(true).DialContext(ctx, metadata)
    if err == nil {
        c.AppendToChains(s)
    }
    return c, err
}
```

### Set (切换节点)

```go
func (s *Selector) Set(name string) error {
    for _, proxy := range s.GetProxies(false) {
        if proxy.Name() == name {
            s.selected = name
            return nil
        }
    }
    return errors.New("proxy not exist")
}
```

## 特性说明

- **持久化选择**：节点选择会持久化保存
- **嵌套支持**：可以引用其他代理组
- **特殊节点**：支持 DIRECT 和 REJECT
- **面板操作**：通过 Web 面板或 API 切换

## API 切换示例

```bash
# 切换到节点B
curl -X PUT http://127.0.0.1:9090/proxies/Proxy \
  -H "Content-Type: application/json" \
  -d '{"name": "节点B"}'
```

## 相关文档

- [[overview]] - 代理组概述
- [[url-test]] - url-test 代理组
- [[config]] - 代理组配置详解
- [[ref/mihomo/provider/overview|Provider 概述]]