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

提供高并发、大文件、低延迟、内存受限等场景的优化配置，包含连接池调优、内存调优、网络参数优化。

## 场景化配置

### 高并发短连接

适用于 Web 浏览、API 调用等大量短连接场景。

```json
{
  "pool": {
    "max_cache_per_endpoint": 64,
    "max_idle_seconds": 60,
    "tcp_nodelay": true,
    "connect_timeout_ms": 2000
  },
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 32768,
      "max_streams": 64
    }
  },
  "dns": {
    "cache_size": 15000,
    "timeout_ms": 2000
  }
}
```

**原理**: 增大连接池缓存以减少重复建连开销，开启 TCP_NODELAY 降低短连接延迟，启用 Smux 复用减少连接数。

### 大文件传输

适用于文件下载、视频流媒体等需要高吞吐量的场景。

```json
{
  "pool": {
    "max_cache_per_endpoint": 16,
    "max_idle_seconds": 120,
    "recv_buffer_size": 262144,
    "send_buffer_size": 262144
  },
  "buffer": {
    "default_size": 65536,
    "max_size": 2097152
  },
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 65536,
      "max_streams": 128
    }
  }
}
```

**原理**: 增大收发缓冲区至 256KB+，提升单次数据传输量，减少系统调用次数。大缓冲区需要更多内存，需权衡。

### 低延迟场景

适用于游戏、实时通信、SSH 等对延迟敏感的场景。

```json
{
  "pool": {
    "connect_timeout_ms": 500,
    "tcp_nodelay": true,
    "max_idle_seconds": 300
  },
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 16384,
      "max_streams": 32
    }
  },
  "dns": {
    "timeout_ms": 1000,
    "cache_size": 20000
  }
}
```

**原理**: 小缓冲区（16KB）减少数据在内存中的滞留时间，TCP_NODELAY 立即发送，长空闲超时保持连接存活。

### 内存受限场景

适用于 VPS、容器等资源受限环境（512MB 以下内存）。

```json
{
  "pool": {
    "max_cache_per_endpoint": 16,
    "max_idle_seconds": 30,
    "recv_buffer_size": 32768,
    "send_buffer_size": 32768
  },
  "buffer": {
    "default_size": 8192,
    "max_size": 262144
  },
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 16384,
      "max_streams": 16
    }
  },
  "dns": {
    "cache_size": 2000
  },
  "trace": {
    "log_level": "warn"
  }
}
```

**原理**: 全面降低各模块缓冲区大小，减少缓存条目，降低日志级别减少日志内存开销。

## 连接池调优

### 参数详解

| 参数 | 作用 | 调优方向 | 注意事项 |
|------|------|----------|----------|
| `max_cache_per_endpoint` | 缓存的下游连接数 | 增大提升复用率 | 每个连接占用 ~64-256KB 内存 |
| `max_idle_seconds` | 空闲连接超时 | 增大减少重建 | 过大导致无效连接占用资源 |
| `connect_timeout_ms` | 建连超时 | 减小加快失败感知 | 过小可能在慢网络下误判 |
| `tcp_nodelay` | 禁用 Nagle | 开启降低延迟 | 可能增加小包数量 |

### 调优方法

```
1. 监控当前连接池命中率
   ↓
2. 如果命中率 < 80%，增大 max_cache_per_endpoint
   ↓
3. 如果内存占用过高，减小 max_idle_seconds
   ↓
4. 如果延迟敏感，确保 tcp_nodelay = true
   ↓
5. 循环验证，直到达到目标性能
```

### 连接池命中率估算

如果 Prism 日志中频繁出现 `pool miss` 或 `creating new connection`，说明缓存命中率低，需要增大 `max_cache_per_endpoint`。

## 内存调优

### 内存使用估算

```
总内存 ≈ 连接池内存 + 缓冲区内存 + 多路复用内存 + DNS 缓存 + 日志缓冲

连接池内存 = max_cache_per_endpoint × 连接数 × 单连接大小 (~128KB)
缓冲区内存 = 并发连接数 × (recv_buffer_size + send_buffer_size)
多路复用内存 = 复用连接数 × max_streams × buffer_size
DNS 缓存内存 = cache_size × 单条目大小 (~512B)
```

### 示例估算

以 4 核 2GB 服务器为例，假设 50 并发：

| 模块 | 计算 | 内存 |
|------|------|------|
| 连接池 | 32 × 4 × 128KB | ~16 MB |
| 缓冲区 | 50 × (64KB + 64KB) | ~6.4 MB |
| 多路复用 | 4 × 64 × 32KB | ~8 MB |
| DNS 缓存 | 10000 × 512B | ~5 MB |
| **总计** | | **~36 MB** |

可见 Prism 内存占用远低于 2GB 上限，有充足余量。可适当增大各参数以换取更好性能。

### 内存泄漏排查

如果发现内存持续增长不回收：

1. 开启 `trace.log_level = "debug"` 观察连接建立/关闭是否对称
2. 检查 `pool.max_idle_seconds` 是否过大导致僵尸连接
3. 使用 `top` 或 `htop` 监控 RSS 内存
4. 必要时重启服务回收内存

## 网络参数优化

### 系统级优化（Linux）

```bash
# 增大文件描述符限制（影响最大并发连接数）
ulimit -n 65536

# 优化 TCP 参数
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
sudo sysctl -w net.ipv4.tcp_fin_timeout=15

# 持久化配置
sudo tee /etc/sysctl.d/99-prism.conf > /dev/null << 'EOF'
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=15
EOF
sudo sysctl -p /etc/sysctl.d/99-prism.conf
```

### Windows 网络优化

```powershell
# 增大临时端口范围
netsh int ipv4 set dynamicport tcp start=10000 num=55536

# 禁用延迟确认（类似 TCP_NODELAY）
netsh int tcp set global autotuninglevel=normal
```

## 多路复用调优

### Smux vs Yamux 选择

| 特性 | Smux | Yamux |
|------|------|-------|
| 协议复杂度 | 简单 | 复杂 |
| 开销 | 小 | 较大 |
| 适用场景 | 高并发、低延迟 | 需要高可靠 |
| 推荐 | 大多数场景 | 特殊需求 |

### Smux 参数调优

```json
{
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 32768,
      "max_streams": 128
    }
  }
}
```

- **buffer_size**: 读写缓冲区大小。>= 32KB 是性能甜点
- **max_streams**: 每连接最大流数。128-256 适合大多数场景

### 何时启用多路复用？

| 场景 | 建议 |
|------|------|
| 大量短连接用户 | 强烈建议，显著减少连接开销 |
| 高延迟网络 | 建议，减少握手次数 |
| 单用户大文件传输 | 不需要，单连接已足够 |
| 内存极度受限 | 不建议，多路复用增加内存占用 |

## 性能测试

### 使用基准工具

```bash
# 查看性能基准工具用法
# 详见 [[dev/performance/report|性能基准]]

# 简单测试
time curl -x http://127.0.0.1:8081 http://example.com -o /dev/null

# 并发测试（使用 ab）
ab -n 1000 -c 50 -x 127.0.0.1:8081 http://example.com/
```

### 性能指标参考

| 指标 | 优秀 | 良好 | 需优化 |
|------|------|------|--------|
| 首包延迟 | < 50ms | 50-200ms | > 200ms |
| 吞吐量 | > 100 Mbps | 10-100 Mbps | < 10 Mbps |
| 并发连接 | > 1000 | 100-1000 | < 100 |
| 内存占用 | < 100MB | 100-500MB | > 500MB |

## 常见问题

**Q: buffer_size 太小会怎样？**
A: 吞吐量大幅下降。建议 >= 32KB。过小会导致大量系统调用，CPU 开销增加，带宽利用率降低。

**Q: buffer_size 越大越好吗？**
A: 不是。大缓冲区增加内存占用和延迟。低延迟场景用 16KB，大吞吐场景用 64-256KB。需根据场景权衡。

**Q: 如何测试性能？**
A: 使用 [[dev/performance/report|性能基准]] 工具测试。也可用 `curl`/`ab`/`wrk` 等工具进行简单验证。

**Q: 开启多路复用一定更快吗？**
A: 不一定。多路复用减少了连接数，但增加了协议开销。短连接场景收益明显，单连接大文件传输反而可能略慢。

**Q: DNS 缓存大小怎么设？**
A: 一般用户 10000 条目足够。如果访问域名非常分散（如 CDN 场景），可增至 20000。内存受限时可降至 2000。

**Q: 如何平衡延迟和吞吐量？**
A: 延迟优先：小 buffer + TCP_NODELAY + 开启多路复用。吞吐优先：大 buffer + 连接池 + 适度多路复用。

## 相关页面

- [[dev/performance/report|性能基准]]
- [[dev/testing/stress|压力测试]]
- [[docs/configuration|配置说明]]
- [[core/multiplex/smux/craft|Smux 实现]]
- [[docs/faq|常见问题]]
