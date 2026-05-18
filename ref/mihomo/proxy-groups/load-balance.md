---
title: "mihomo load-balance 代理组"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/loadbalance.go"
tags: [mihomo, proxy-groups, load-balance, 负载均衡]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo load-balance 代理组

load-balance 类型代理组将流量分散到多个节点，实现负载均衡。

## 源码位置

- `adapter/outboundgroup/loadbalance.go`

## LoadBalance 结构

```go
type LoadBalance struct {
    *GroupBase
    disableUDP     bool
    strategyFn     strategyFn
    testUrl        string
    expectedStatus string
    Hidden         bool
    Icon           string
}
```

## YAML 配置示例

### 基本配置

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

### round-robin 策略

```yaml
proxy-groups:
  - name: "轮询"
    type: load-balance
    strategy: round-robin
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

### sticky-sessions 策略

```yaml
proxy-groups:
  - name: "粘性会话"
    type: load-balance
    strategy: sticky-sessions
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    proxies:
      - "节点A"
      - "节点B"
      - "节点C"
```

### 使用 provider

```yaml
proxy-groups:
  - name: "负载均衡"
    type: load-balance
    strategy: consistent-hashing
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    use:
      - provider1
```

## 配置字段说明

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | 是 | - | 代理组名称 |
| `type` | string | 是 | - | 固定为 `load-balance` |
| `strategy` | string | 否 | consistent-hashing | 负载均衡策略 |
| `url` | string | 是 | - | 健康检查 URL |
| `interval` | int | 否 | 300 | 检查间隔（秒） |
| `proxies` | []string | 否* | - | 节点列表 |

## 负载均衡策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `round-robin` | 轮询分配 | 通用负载均衡 |
| `consistent-hashing` | 一致性哈希（按域名） | 需要会话保持 |
| `sticky-sessions` | 粘性会话（按来源 IP） | 需要 IP 保持 |

## 核心方法

### strategyConsistentHashing

```go
func strategyConsistentHashing(url string) strategyFn {
    maxRetry := 5
    return func(proxies []C.Proxy, metadata *C.Metadata, touch bool) C.Proxy {
        key := utils.MapHash(getKey(metadata))
        buckets := int32(len(proxies))
        for i := 0; i < maxRetry; i, key = i+1, key+1 {
            idx := jumpHash(key, buckets)
            proxy := proxies[idx]
            if proxy.AliveForTestUrl(url) {
                return proxy
            }
        }
        return proxies[0]
    }
}
```

### strategyRoundRobin

```go
func strategyRoundRobin(url string) strategyFn {
    idx := 0
    idxMutex := sync.Mutex{}
    return func(proxies []C.Proxy, metadata *C.Metadata, touch bool) C.Proxy {
        idxMutex.Lock()
        defer idxMutex.Unlock()
        
        length := len(proxies)
        for i := 0; i < length; i++ {
            id := (idx + i) % length
            proxy := proxies[id]
            if proxy.AliveForTestUrl(url) {
                idx = (idx + i + 1) % length
                return proxy
            }
        }
        return proxies[0]
    }
}
```

### strategyStickySessions

```go
func strategyStickySessions(url string) strategyFn {
    ttl := time.Minute * 10
    lruCache := lru.New[uint64, int](
        lru.WithAge[uint64, int](int64(ttl.Seconds())),
        lru.WithSize[uint64, int](1000))
    return func(proxies []C.Proxy, metadata *C.Metadata, touch bool) C.Proxy {
        key := utils.MapHash(getKeyWithSrcAndDst(metadata))
        // ... 粘性会话逻辑
    }
}
```

## jumpHash (跳跃哈希)

```go
func jumpHash(key uint64, buckets int32) int32 {
    var b, j int64
    for j < int64(buckets) {
        b = j
        key = key*2862933555777941757 + 1
        j = int64(float64(b+1) * (float64(int64(1)<<31) / float64((key>>33)+1)))
    }
    return int32(b)
}
```

## 一致性哈希优势

- 同一域名始终使用同一节点
- 避免 CDN 缓存失效
- 减少会话中断

## 相关文档

- [[overview]] - 代理组概述
- [[ref/mihomo/proxy-groups/fallback|fallback]] - fallback 代理组
- [[url-test]] - url-test 代理组
- [[config]] - 代理组配置详解