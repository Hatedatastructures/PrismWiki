---
title: 性能建议
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide, performance]
---

# 性能建议

Prism 性能优化配置建议。

## 功能简介

提供高并发、大文件、低延迟、内存受限等场景的优化配置。

## 高并发短连接

```json
{
  "pool": {
    "max_cache_per_endpoint": 64,
    "max_idle_seconds": 60
  },
  "multiplex": {
    "smux": {
      "buffer_size": 32768
    }
  }
}
```

## 大文件传输

```json
{
  "pool": {
    "recv_buffer_size": 131072,
    "send_buffer_size": 131072
  },
  "multiplex": {
    "smux": {
      "buffer_size": 65536,
      "max_streams": 128
    }
  }
}
```

## 低延迟场景

```json
{
  "pool": {
    "connect_timeout_ms": 500,
    "tcp_nodelay": true
  },
  "dns": {
    "timeout_ms": 1000
  }
}
```

## 内存受限

```json
{
  "pool": {
    "max_cache_per_endpoint": 16
  },
  "multiplex": {
    "smux": {
      "buffer_size": 16384
    }
  },
  "dns": {
    "cache_size": 2000
  }
}
```

## 关键参数

| 参数 | 影响 | 建议值 |
|------|------|--------|
| `buffer_size` | 吞吐量关键 | >= 32KB |
| `max_streams` | 并发流数 | 128-256 |
| `tcp_nodelay` | 降低延迟 | 必须开启 |

## 常见问题

**Q: buffer_size 太小会怎样？**
A: 吞吐量大幅下降，必须 >= 32KB。

**Q: 如何测试性能？**
A: 使用 [[performance/benchmark|性能基准]] 工具测试。

## 相关页面

- [[performance/benchmark|性能基准]]
- [[dev/stress|压力测试]]
- [[docs/configuration|配置说明]]
- [[core/multiplex/smux/craft|Smux 实现]]