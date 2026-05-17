---
title: "mihomo fallback 代理组"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/fallback.go"
tags: [mihomo, proxy-groups, fallback, 故障转移]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo fallback 代理组

fallback 类型代理组按顺序选择第一个可用的节点，实现故障转移。

## 源码位置

- `adapter/outboundgroup/fallback.go`

## Fallback 结构

```go
type Fallback struct {
    *GroupBase
    disableUDP     bool
    testUrl        string
    selected       string
    expectedStatus string
    Hidden         bool
    Icon           string
}
```

## YAML 配置示例

### 基本配置

```yaml
proxy-groups:
  - name: "故障转移"
    type: fallback
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "主节点"
      - "备用节点1"
      - "备用节点2"
```

### 带懒加载配置

```yaml
proxy-groups:
  - name: "高可用"
    type: fallback
    url: "http://www.gstatic.com/generate_204"
    interval: 60
    lazy: true
    timeout: 5000
    proxies:
      - "付费节点"
      - "免费节点A"
      - "免费节点B"
```

### 使用 provider

```yaml
proxy-groups:
  - name: "故障转移"
    type: fallback
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    use:
      - provider1
```

### 完整配置

```yaml
proxy-groups:
  - name: "故障转移"
    type: fallback
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    lazy: true
    timeout: 5000
    expected-status: "200"
    proxies:
      - "主节点"
      - "备用节点1"
      - "备用节点2"
    disable-udp: false
    hidden: false
```

## 配置字段说明

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | 是 | - | 代理组名称 |
| `type` | string | 是 | - | 固定为 `fallback` |
| `url` | string | 是 | - | 健康检查 URL |
| `interval` | int | 否 | 300 | 检查间隔（秒） |
| `lazy` | bool | 否 | true | 懒加载模式 |
| `timeout` | int | 否 | 5000 | 检查超时（毫秒） |
| `expected-status` | string | 否 | "*" | 预期 HTTP 状态码 |
| `proxies` | []string | 否* | - | 节点列表（按优先级排序） |

## 核心方法

### findAliveProxy (查找可用节点)

```go
func (f *Fallback) findAliveProxy(touch bool) C.Proxy {
    proxies := f.GetProxies(touch)
    for _, proxy := range proxies {
        if len(f.selected) == 0 {
            if proxy.AliveForTestUrl(f.testUrl) {
                return proxy
            }
        } else {
            if proxy.Name() == f.selected {
                if proxy.AliveForTestUrl(f.testUrl) {
                    return proxy
                } else {
                    f.selected = ""
                }
            }
        }
    }
    return proxies[0]
}
```

## 与 url-test 的区别

| 特性 | url-test | fallback |
|------|----------|----------|
| 选择依据 | 延迟最低 | 第一个可用 |
| 节点优先级 | 无优先级 | 有明确顺序 |
| 切换条件 | 延迟差距超过 tolerance | 当前节点不可用 |
| 适用场景 | 追求速度 | 追求可靠性 |

## 工作流程

```
定期健康检查
    │
    ▼
按顺序测试节点
    │
    ├── 节点1 可用 → 使用节点1
    │
    ├── 节点1 不可用 → 测试节点2
    │       │
    │       ├── 节点2 可用 → 使用节点2
    │       │
    │       └── 继续测试后续节点
```

## 适用场景

1. **付费主节点 + 免费备用**
2. **地区优先**：香港 → 日本 → 美国
3. **协议优先**：Hysteria2 → VLESS → SS

## 相关文档

- [[overview]] - 代理组概述
- [[url-test]] - url-test 代理组
- [[load-balance]] - load-balance 代理组
- [[config]] - 代理组配置详解