---
title: "Health Check"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/provider/healthcheck.go"
tags: [mihomo, provider, health-check, 健康检查, 节点测试]
created: 2026-05-17
updated: 2026-05-17
related: [overview, proxy-provider]
---

# Health Check

**类别**: Mihomo 健康检查配置

## 概述

Health Check（健康检查）用于自动检测代理节点的可用性和延迟。Proxy Provider 和代理组都可以配置健康检查。

## 配置位置

健康检查可以在两个位置配置：

### Proxy Provider

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
```

### Proxy Group

```yaml
proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - my-provider
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
```

## 配置参数

### enable

```yaml
enable: true
```

| 值 | 说明 |
|-----|------|
| true | 启用健康检查 |
| false | 禁用健康检查 |

### url

```yaml
url: https://www.gstatic.com/generate_204
```

测试 URL，用于测量节点延迟。

常用测试 URL：

| URL | 说明 |
|-----|------|
| `https://www.gstatic.com/generate_204` | Google 204（推荐） |
| `http://www.gstatic.com/generate_204` | Google HTTP |
| `https://cp.cloudflare.com/generate_204` | Cloudflare |
| `https://www.apple.com` | Apple（较慢） |
| `http://ping.test()` | 内置测试 |

### interval

```yaml
interval: 300
```

检查间隔（秒）：

| 推荐值 | 场景 |
|--------|------|
| 30 | 实时监控 |
| 60 | 常规监控 |
| 300 | 低频检查 |
| 600 | 低负载 |

### timeout

```yaml
timeout: 5000
```

超时时间（毫秒），默认 5000ms。

### expected-status

```yaml
expected-status: 204
```

期望的 HTTP 璀态码。

| 值 | 说明 |
|-----|------|
| 204 | 无内容（常用） |
| 200 | 成功 |

### tolerance

```yaml
tolerance: 50
```

延迟容忍度（毫秒），仅用于 url-test 代理组。

当节点延迟差异小于 tolerance 时，不切换节点。

### lazy

```yaml
lazy: true
```

懒惰模式，仅在无可用节点时检查。

## 健康检查流程

```
健康检查流程：
┌─────────────────────────────────────────────┐
│                                             │
│  定时触发                                    │
│      │                                      │
│      ├── 遍历所有节点                        │
│      │                                      │
│      ├── 发送 HTTP HEAD 请求                 │
│      │   到 test URL                        │
│      │                                      │
│      ├── 测量响应时间                        │
│      │                                      │
│      ├── 检查响应状态码                      │
│      │                                      │
│      ├── 更新节点状态                        │
│      │   alive/dead                         │
│      │                                      │
│      └── 记录延迟值                          │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 节点状态                              │   │
│  │   alive: 延迟 < timeout              │   │
│  │   dead: 超时或错误                    │   │
│  └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘
```

## 配置示例

### Provider 健康检查

```yaml
proxy-providers:
  provider-1:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
      timeout: 5000
```

### url-test 代理组

```yaml
proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - provider-1
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: false
```

### fallback 代理组

```yaml
proxy-groups:
  - name: "fallback"
    type: fallback
    use:
      - provider-1
    url: "https://www.gstatic.com/generate_204"
    interval: 300
```

### load-balance 代理组

```yaml
proxy-groups:
  - name: "balance"
    type: load-balance
    use:
      - provider-1
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    strategy: consistent-hashing
```

### 完整配置

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    path: ./provider/my-provider.yaml
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
      timeout: 5000
      expected-status: 204

proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - my-provider
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: false
```

## 健康检查与代理组类型

| 代理组类型 | 健康检查行为 |
|-----------|--------------|
| select | 不自动检查 |
| url-test | 自动选择最低延迟 |
| fallback | 自动切换到可用节点 |
| load-balance | 分散负载到可用节点 |

## 性能考量

健康检查会产生额外网络请求：

| 因素 | 影响 |
|------|------|
| interval | 检查频率影响负载 |
| timeout | 超时时间影响响应 |
| 节点数量 | 节点多则检查慢 |

建议：
- 合理设置 interval，避免过于频繁
- 设置合适的 timeout，快速剔除不可用节点
- 使用 lazy 模式减少检查次数

## 相关链接

- [[ref/mihomo/provider/overview|Provider 概览]] — 提供者总览
- [[ref/mihomo/provider/proxy-provider|Proxy Provider]] — 代理提供者
- [[client/mihomo-proxy-groups|代理组配置]] — 代理组配置参考