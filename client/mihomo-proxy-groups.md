---
title: mihomo 代理组
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [proxy-client, mihomo, configuration]
sources:
  - mihomo 文档
confidence: high
---

# mihomo 代理组

代理组（Proxy Group）是 mihomo 的核心概念之一，用于组织和管理代理节点。每个代理组定义了一种节点选择策略，规则将流量导向特定的代理组，代理组再根据策略选择实际的节点进行连接。

## 代理组的作用

```
规则匹配 → 代理组 → 节点选择策略 → 具体节点 → 目标服务器
```

代理组解决的核心问题：
- 手动选择：用户在多个节点间切换
- 自动选优：自动选择延迟最低的节点
- 故障转移：主节点不可用时自动切换备份
- 负载均衡：分散流量到多个节点
- 链式代理：多跳代理增强隐私

## 代理组类型详解

### select（手动选择）

最基础的代理组类型，用户手动选择使用哪个节点。

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
      - "自动选择"
      - "DIRECT"
```

- **用法**：在 mihomo 面板中手动切换节点
- **场景**：需要指定特定节点访问特定服务
- **特性**：选择会持久化，重启后保持上次选择
- **常见模式**：在 select 组中嵌套 url-test 组，兼顾手动和自动

### url-test（自动选优）

自动测试所有节点延迟，选择延迟最低的节点。

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `url` | 测试用的 URL，返回 204 状态码最佳 | 必填 |
| `interval` | 测试间隔，单位秒 | 300 |
| `tolerance` | 延迟容差，单位毫秒。差距在此范围内不切换节点 | 150 |
| `lazy` | true 时仅在有实际连接时才测试 | false |
| `timeout` | 测试超时时间，单位毫秒 | 5000 |

- **tolerance 的意义**：避免延迟相近时频繁切换（抖动）。设为 50 表示只有新节点比当前节点快 50ms 以上才切换。
- **lazy 的意义**：节省资源，空闲时不主动发起测试连接。
- **测试 URL 选择**：推荐使用 `generate_204` 端点（响应体为空，仅返回 204 状态码）。

### fallback（故障转移）

按节点列表顺序选择第一个可用的节点。

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

- **工作方式**：从上到下测试，使用第一个通过健康检查的节点
- **场景**：有明确优先级的节点，如付费主节点 + 免费备用节点
- **与 url-test 的区别**：url-test 选延迟最低的，fallback 选第一个可用的

### load-balance（负载均衡）

将流量分散到多个节点，避免单个节点过载。

```yaml
proxy-groups:
  - name: "负载均衡"
    type: load-balance
    strategy: consistent-hashing
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

#### 负载均衡策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `round-robin` | 轮询，依次分配 | 通用负载均衡 |
| `consistent-hashing` | 一致性哈希，相同目标域名分配到相同节点 | 需要会话保持的服务 |
| `sticky-sessions` | 粘性会话，同一来源 IP 固定到同一节点 | 需要 IP 保持的场景 |
| `round-robin(url-test)` | 轮询 + 自动剔除不可用节点 | 高可用负载均衡 |

- **consistent-hashing** 最常用：同一网站总是走同一节点，避免频繁切换导致的会话中断
- 需要配合 `url` 和 `interval` 进行健康检查

### relay（链式代理）

按顺序经过多个代理节点，形成多跳代理链。

```yaml
proxy-groups:
  - name: "多跳代理"
    type: relay
    proxies:
      - "入口节点"
      - "中间节点"
      - "出口节点"
```

- **数据流向**：客户端 → 入口 → 中间 → 出口 → 目标服务器
- **隐私增强**：每个节点只知道相邻节点，无法确定完整路径
- **性能影响**：每一跳都增加延迟，通常 2-3 跳足够
- **注意**：relay 组内的节点必须都支持链式连接

## 代理组嵌套（组中组）

代理组可以引用其他代理组，实现灵活的分层管理：

```yaml
proxy-groups:
  # 基础组：自动选择延迟最低的日本节点
  - name: "日本节点"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    proxies:
      - "东京节点1"
      - "东京节点2"
      - "大阪节点1"

  # 基础组：自动选择延迟最低的美国节点
  - name: "美国节点"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    proxies:
      - "洛杉矶节点1"
      - "硅谷节点1"

  # 顶层组：手动选择地区
  - name: "Proxy"
    type: select
    proxies:
      - "日本节点"
      - "美国节点"
      - "DIRECT"
```

嵌套的好处：
- 顶层手动选择地区，底层自动选最优节点
- 修改节点不影响顶层组结构
- 逻辑清晰，易于维护

## use 字段（引用 proxy-providers）

使用 `use` 字段引用 proxy-providers 中定义的节点列表：

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    use:
      - provider1
      - provider2
    proxies:
      - "手动节点"
```

`use` 和 `proxies` 可以同时使用，provider 中的节点和手动定义的节点合并。

## proxy-providers（远程节点列表）

proxy-providers 用于从远程 URL 或本地文件加载节点列表：

```yaml
proxy-providers:
  provider1:
    type: http
    url: "https://example.com/nodes.yaml"
    path: ./providers/provider1.yaml
    interval: 3600
    health-check:
      enable: true
      url: "http://www.gstatic.com/generate_204"
      interval: 300
```

### 配置说明

| 字段 | 说明 |
|------|------|
| `type` | `http`（远程）或 `file`（本地） |
| `url` | 远程节点列表地址 |
| `path` | 本地缓存路径 |
| `interval` | 自动更新间隔（秒） |
| `health-check.enable` | 启用健康检查 |
| `health-check.url` | 健康检查 URL |
| `health-check.interval` | 健康检查间隔（秒） |

### proxy-providers 与直接定义节点的区别

| 特性 | proxy-providers | 直接在 proxies 中定义 |
|------|----------------|---------------------|
| 远程更新 | ✅ 自动拉取最新节点 | ❌ 需手动修改配置 |
| 健康检查 | ✅ 内置支持 | ❌ 需在组级别配置 |
| 节点管理 | 集中管理，多组共享 | 分散定义 |
| 适用场景 | 机场订阅 | 少量自建节点 |

## 常见组合模式

### 经典三层结构

```yaml
proxy-groups:
  # 第一层：手动选择策略
  - name: "Proxy"
    type: select
    proxies:
      - "自动选择"
      - "手动选择"
      - "负载均衡"
      - "DIRECT"

  # 第二层：自动选优
  - name: "自动选择"
    type: url-test
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"

  # 第二层：手动选择
  - name: "手动选择"
    type: select
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"

  # 第二层：负载均衡
  - name: "负载均衡"
    type: load-balance
    strategy: consistent-hashing
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

### 按用途分流

```yaml
proxy-groups:
  - name: "AI服务"
    type: select
    proxies:
      - "美国节点"
      - "日本节点"

  - name: "流媒体"
    type: select
    proxies:
      - "香港节点"
      - "新加坡节点"

  - name: "通用代理"
    type: url-test
    proxies:
      - "节点A"
      - "节点B"
```

规则中将 `openai.com` 指向 AI 服务组，`netflix.com` 指向流媒体组，其他走通用代理。

## 相关页面

- [[Mihomo-Clash-Config]] — mihomo 完整配置参考
- [[mihomo-Meta]] — mihomo 客户端概述
- [[protocol/proxy-protocols]] — 代理协议概览
- [[mihomo-rules]] — 规则如何将流量导向代理组
- [[protocol/http]] — 代理节点使用的协议
- [[SOCKS5]] — 本地代理协议
