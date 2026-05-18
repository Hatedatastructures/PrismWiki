---
title: "mihomo url-test 代理组"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/urltest.go"
tags: [mihomo, proxy-groups, url-test, 自动选优]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo url-test 代理组

url-test 类型代理组自动测试所有节点延迟，选择最快的节点。

## 源码位置

- `adapter/outboundgroup/urltest.go`

## URLTest 结构

```go
type URLTest struct {
    *GroupBase
    selected       string
    testUrl        string
    expectedStatus string
    tolerance      uint16
    disableUDP     bool
    Hidden         bool
    Icon           string
    fastNode       C.Proxy
    fastSingle     *singledo.Single[C.Proxy]
}
```

## YAML 配置示例

### 基本配置

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

### 带容差配置

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true
    timeout: 5000
    proxies:
      - "节点A"
      - "节点B"
```

### 使用 provider

```yaml
proxy-groups:
  - name: "香港节点"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    use:
      - provider1
    filter: "香港"
```

### 完整配置

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true
    timeout: 5000
    max-failed-times: 5
    expected-status: "200"
    proxies:
      - "节点A"
      - "节点B"
    disable-udp: false
    hidden: false
```

## 配置字段说明

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | 是 | - | 代理组名称 |
| `type` | string | 是 | - | 固定为 `url-test` |
| `url` | string | 是 | - | 测试 URL |
| `interval` | int | 否 | 300 | 测试间隔（秒） |
| `tolerance` | int | 否 | 150 | 延迟容差（毫秒） |
| `lazy` | bool | 否 | true | 懒加载模式 |
| `timeout` | int | 否 | 5000 | 测试超时（毫秒） |
| `max-failed-times` | int | 否 | - | 最大失败次数 |
| `expected-status` | string | 否 | "*" | 预期 HTTP 状态码 |
| `proxies` | []string | 否* | - | 节点列表 |
| `use` | []string | 否* | - | 引用的 provider |

## 核心方法

### fast (选择最快节点)

```go
func (u *URLTest) fast(touch bool) C.Proxy {
    elm, _, shared := u.fastSingle.Do(func() (C.Proxy, error) {
        proxies := u.GetProxies(touch)
        if u.selected != "" {
            for _, proxy := range proxies {
                if !proxy.AliveForTestUrl(u.testUrl) {
                    continue
                }
                if proxy.Name() == u.selected {
                    u.fastNode = proxy
                    return proxy, nil
                }
            }
        }

        fast := proxies[0]
        minDelay := fast.LastDelayForTestUrl(u.testUrl)
        
        for _, proxy := range proxies[1:] {
            if !proxy.AliveForTestUrl(u.testUrl) {
                continue
            }
            delay := proxy.LastDelayForTestUrl(u.testUrl)
            if delay < minDelay {
                fast = proxy
                minDelay = delay
            }
        }
        
        // tolerance 机制
        if u.fastNode == nil || !u.fastNode.AliveForTestUrl(u.testUrl) ||
           u.fastNode.LastDelayForTestUrl(u.testUrl) > fast.LastDelayForTestUrl(u.testUrl)+u.tolerance {
            u.fastNode = fast
        }
        return u.fastNode, nil
    })
    return elm
}
```

## tolerance 机制

tolerance 参数避免因延迟抖动频繁切换节点：

```
当前选择：节点A (100ms)
新最优：节点B (80ms)
差距：20ms < tolerance(50ms)
不切换，继续使用节点A
```

## 推荐测试 URL

| URL | 说明 |
|-----|------|
| `http://www.gstatic.com/generate_204` | Google 204 端点 |
| `http://cp.cloudflare.com/generate_204` | Cloudflare 204 端点 |
| `http://connectivitycheck.gstatic.com/generate_204` | Android 检测端点 |

## 相关文档

- [[overview]] - 代理组概述
- [[selector]] - select 代理组
- [[ref/mihomo/proxy-groups/fallback|fallback]] - fallback 代理组
- [[config]] - 代理组配置详解