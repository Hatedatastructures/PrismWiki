---
title: "Sing-Mux 多路复用"
category: "mihomo"
type: ref
module: ref/mihomo
source: "github.com/metacubex/sing-mux"
tags: [mihomo, singmux, mux, 多路复用, brutal, sing-box]
created: 2026-05-17
updated: 2026-05-17
related: [trojan, vless, vmess, shadowsocks]
layer: ref
---

# Sing-Mux 多路复用

**类别**: Mihomo 多路复用参考

## 概述

Sing-Mux 是 sing-box 项目的多路复用实现，由 `github.com/metacubex/sing-mux` 库提供。它是 Mihomo 推荐的现代多路复用方案，支持多种协议（smux、yamux、h2mux）和 TCP Brutal 加速。

Sing-Mux 主要特点：
- 多协议支持（smux、yamux、h2mux）
- TCP Brutal 加速（需要服务端支持）
- Padding 随机填充
- 统计支持
- TCP/UDP 分离配置

## 配置格式

### 代理节点配置

```yaml
proxies:
  - name: "singmux-node"
    type: trojan
    server: example.com
    port: 443
    password: "trojan-password"
    # Sing-Mux 配置
    smux:
      enabled: true
      protocol: smux        # smux/yamux/h2mux
      max-connections: 4    # 最大连接数
      min-streams: 4        # 最小流数
      max-streams: 0        # 最大流数（0=无限制）
      padding: false        # 随机填充
      statistic: false      # 流量统计
      only-tcp: false       # 仅 TCP
      # TCP Brutal 配置
      brutal-opts:
        enabled: false
        up: "100 Mbps"      # 上行带宽
        down: "100 Mbps"    # 下行带宽
```

### 配置参数说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| enabled | bool | false | 启用多路复用 |
| protocol | string | smux | 协议类型 |
| max-connections | int | 4 | 最大连接数 |
| min-streams | int | 4 | 最小流数 |
| max-streams | int | 0 | 最大流数（0=无限制） |
| padding | bool | false | 随机填充 |
| statistic | bool | false | 流量统计 |
| only-tcp | bool | false | 仅处理 TCP |
| brutal-opts.enabled | bool | false | 启用 TCP Brutal |
| brutal-opts.up | string | - | 上行带宽 |
| brutal-opts.down | string | - | 下行带宽 |

### Protocol 选择

| Protocol | 说明 | 适用场景 |
|----------|------|----------|
| smux | Smux 协议 | 简单场景，高性能 |
| yamux | Yamux 协议 | 需要更多功能 |
| h2mux | HTTP/2 多路复用 | HTTP/2 环境 |

## 源码实现

### SingMuxOption 结构

```go
// 文件: adapter/outbound/singmux.go
package outbound

type SingMuxOption struct {
    Enabled        bool         `proxy:"enabled,omitempty"`
    Protocol       string       `proxy:"protocol,omitempty"`
    MaxConnections int          `proxy:"max-connections,omitempty"`
    MinStreams     int          `proxy:"min-streams,omitempty"`
    MaxStreams     int          `proxy:"max-streams,omitempty"`
    Padding        bool         `proxy:"padding,omitempty"`
    Statistic      bool         `proxy:"statistic,omitempty"`
    OnlyTcp        bool         `proxy:"only-tcp,omitempty"`
    BrutalOpts     BrutalOption `proxy:"brutal-opts,omitempty"`
}

type BrutalOption struct {
    Enabled bool   `proxy:"enabled,omitempty"`
    Up      string `proxy:"up,omitempty"`
    Down    string `proxy:"down,omitempty"`
}
```

### SingMux 客户端

```go
// 文件: adapter/outbound/singmux.go
type SingMux struct {
    ProxyAdapter
    client  *mux.Client
    onlyTcp bool
}

func (s *SingMux) DialContext(ctx context.Context, metadata *C.Metadata) (C.Conn, error) {
    c, err := s.client.DialContext(ctx, "tcp", 
        M.ParseSocksaddrHostPort(metadata.String(), metadata.DstPort))
    if err != nil {
        return nil, err
    }
    return NewConn(c, s), err
}

func (s *SingMux) ListenPacketContext(ctx context.Context, metadata *C.Metadata) (C.PacketConn, error) {
    if s.onlyTcp {
        return s.ProxyAdapter.ListenPacketContext(ctx, metadata)
    }
    // UDP 通过 Mux
    pc, err := s.client.ListenPacket(ctx, M.SocksaddrFromNet(metadata.UDPAddr()))
    return newPacketConn(N.NewThreadSafePacketConn(pc), s), nil
}

func NewSingMux(option SingMuxOption, proxy ProxyAdapter) (ProxyAdapter, error) {
    singDialer := proxydialer.NewSingDialer(proxydialer.New(proxy, option.Statistic))
    
    client, err := mux.NewClient(mux.Options{
        Dialer:         singDialer,
        Logger:         log.SingLogger,
        Protocol:       option.Protocol,
        MaxConnections: option.MaxConnections,
        MinStreams:     option.MinStreams,
        MaxStreams:     option.MaxStreams,
        Padding:        option.Padding,
        Brutal: mux.BrutalOptions{
            Enabled:    option.BrutalOpts.Enabled,
            SendBPS:    StringToBps(option.BrutalOpts.Up),
            ReceiveBPS: StringToBps(option.BrutalOpts.Down),
        },
    })
    
    if err != nil {
        return nil, err
    }
    
    return &SingMux{
        ProxyAdapter: proxy,
        client:       client,
        onlyTcp:      option.OnlyTcp,
    }, nil
}
```

### 服务端 Mux 配置

```go
// 文件: listener/inbound/mux.go
package inbound

type MuxOption struct {
    Padding bool          `inbound:"padding,omitempty"`
    Brutal  BrutalOptions `inbound:"brutal,omitempty"`
}

type BrutalOptions struct {
    Enabled bool   `inbound:"enabled,omitempty"`
    Up      string `inbound:"up,omitempty"`
    Down    string `inbound:"down,omitempty"`
}

func (m MuxOption) Build() sing.MuxOption {
    return sing.MuxOption{
        Padding: m.Padding,
        Brutal: sing.BrutalOptions{
            Enabled: m.Brutal.Enabled,
            Up:      m.Brutal.Up,
            Down:    m.Brutal.Down,
        },
    }
}
```

## TCP Brutal

### Brutal 概述

TCP Brutal 是一种激进的多路复用优化策略：
- 通过多连接并行传输
- 提高带宽利用率
- 减少单连接瓶颈
- 需要服务端支持

### Brutal 配置

```yaml
smux:
  enabled: true
  protocol: h2mux
  brutal-opts:
    enabled: true
    up: "100 Mbps"     # 或使用数值
    down: "200 Mbps"
```

带宽格式支持：
- `100 Mbps`：100 Megabits
- `1 Gbps`：1 Gigabit
- `10000000`：直接数值（bytes/s）

### Brutal 工作原理

```
TCP Brutal 工作模式：
┌─────────────────────────────────────────────┐
│                                             │
│  单连接：                                    │
│    |--- TCP 1 ----|                         │
│    100 Mbps 带宽                             │
│                                             │
│  Brutal 多连接：                             │
│    |--- TCP 1 ----|                         │
│    |--- TCP 2 ----|                         │
│    |--- TCP 3 ----|                         │
│    |--- TCP 4 ----|                         │
│    4x 100 Mbps = 400 Mbps                   │
│                                             │
└─────────────────────────────────────────────┘
```

## 与 Prism 兼容性

### Prism Sing-Mux 支持

Prism 计划支持 Sing-Mux 的核心功能。

```cpp
// 文件: src/prism/multiplex/singmux/client.hpp（计划）
namespace psm::multiplex::singmux {

enum class singmux_protocol {
    smux,
    yamux,
    h2mux
};

struct brutal_config {
    bool enabled = false;
    uint64_t send_bps = 0;
    uint64_t receive_bps = 0;
};

struct singmux_config {
    singmux_protocol protocol = singmux_protocol::smux;
    int max_connections = 4;
    int min_streams = 4;
    int max_streams = 0;  // 无限制
    bool padding = false;
    bool statistic = false;
    bool only_tcp = false;
    brutal_config brutal;
};

class singmux_client {
public:
    explicit singmux_client(const singmux_config& cfg);
    
    auto dial_tcp(std::string_view address)
        -> net::awaitable<mux_stream>;
    
    auto dial_udp(std::string_view address)
        -> net::awaitable<mux_packet_stream>;
};

} // namespace psm::multiplex::singmux
```

## 使用建议

### Protocol 选择

| 场景 | 推荐 Protocol |
|------|---------------|
| 高性能代理 | smux |
| HTTP/2 环境 | h2mux |
| 需要 Brutal | h2mux |
| 兼容性优先 | yamux |

### 连接数配置

- `max-connections`：4-16，根据带宽调整
- `min-streams`：与 max-connections 相同
- `max-streams`：0（无限制）

### Brutal 使用条件

1. 服务端必须支持 Brutal
2. 带宽配置必须准确
3. 网络质量稳定
4. Linux 系统效果更好

## 相关链接

- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用总览
- [[ref/mihomo/mux/smux|Smux]] — Smux 协议
- [[ref/mihomo/mux/yamux|Yamux]] — Yamux 协议
- [[ref/mihomo/mux/config|Mux 配置]] — 配置参数详解