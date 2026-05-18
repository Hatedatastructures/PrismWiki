---
title: "Kcptun Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/kcptun"
tags: [mihomo, kcptun, kcp, udp, smux, shadowsocks]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, udp, smux]
layer: ref
---

# Kcptun Transport

**类别**: Mihomo 传输层参考

## 概述

Kcptun 是基于 KCP 协议的 UDP 传输插件，通过将 TCP 流量转换为 UDP + KCP 传输，在不稳定的网络环境下可以获得更好的传输性能。KCP 是一个快速可靠的 ARQ（Automatic Repeat-reQuest）协议，专门针对高速传输场景设计。

主要功能：
- UDP + KCP 传输
- 多连接支持
- 自动过期机制
- 数据压缩
- Smux 多路复用
- FEC（前向纠错）

## 配置格式

### Mihomo 配置

```yaml
proxies:
  - name: "kcptun-node"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: kcptun
    plugin-opts:
      key: "kcptun-key"          # KCP 密钥
      crypt: aes                 # 加密算法
      mode: fast                 # KCP 模式
      conn: 1                    # 连接数
      autoexpire: 0              # 自动过期（秒）
      scavengettl: 600           # 清理 TTL
      mtu: 1350                  # MTU
      sndwnd: 128                # 发送窗口
      rcvwnd: 512                # 接收窗口
      datashard: 10              # 数据分片
      parityshard: 3             # 校验分片
      dscp: 0                    # DSCP
      nocomp: false              # 禁用压缩
      acknodelay: false          # ACK 无延迟
      smuxver: 1                 # Smux 版本
      smuxbuf: 4194304           # Smux 缓冲区
      keepalive: 10              # Keepalive（秒）
```

### 配置参数说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| key | string | it's a secrect | KCP 密钥 |
| crypt | string | aes | 加密算法 |
| mode | string | fast | KCP 模式 |
| conn | int | 1 | 并行连接数 |
| mtu | int | 1350 | MTU 大小 |
| sndwnd | int | 128 | 发送窗口 |
| rcvwnd | int | 512 | 接收窗口 |
| datashard | int | 10 | FEC 数据分片 |
| parityshard | int | 3 | FEC 校验分片 |
| nocomp | bool | false | 禁用压缩 |
| keepalive | int | 10 | Keepalive 间隔 |

### KCP 模式

| Mode | NoDelay | Interval | Resend | NC | 适用场景 |
|------|---------|----------|--------|-----|----------|
| normal | 0 | 40 | 2 | 1 | 正常网络 |
| fast | 0 | 30 | 2 | 1 | 稳定网络 |
| fast2 | 1 | 20 | 2 | 1 | 一般网络 |
| fast3 | 1 | 10 | 2 | 1 | 不稳定网络 |

### 加密算法

| Crypt | 说明 |
|-------|------|
| aes | AES-128 |
| aes-128 | AES-128 |
| aes-192 | AES-192 |
| aes-128-gcm | AES-128-GCM |
| salsa20 | Salsa20 |
| blowfish | Blowfish |
| twofish | Twofish |
| xor | XOR（弱） |
| none | 无加密 |

## 协议原理

### KCP 协议

KCP 是一个快速可靠协议，相比 TCP：
- 更低的延迟
- 更快的重传
- 更激进的拥塞控制

```
KCP vs TCP：
┌─────────────────────────────────────────────┐
│ TCP：                                        │
│   - 丢包重传：超时重传                        │
│   - 延迟：较高                                │
│   - 拥塞控制：保守                            │
│                                             │
│ KCP：                                        │
│   - 丢包重传：快速重传 + 选择性重传           │
│   - 延迟：低                                  │
│   - 拥塞控制：激进（可牺牲带宽换延迟）        │
└─────────────────────────────────────────────┘
```

### Kcptun 协议栈

```
Kcptun 协议栈：
┌─────────────────────────────────────────────┐
│ 应用层：Shadowsocks                          │
├─────────────────────────────────────────────┤
│ 多路复用层：Smux                             │
├─────────────────────────────────────────────┤
│ 压缩层：Snappy（可选）                        │
├─────────────────────────────────────────────┤
│ 可靠传输层：KCP                              │
│   - ARQ 协议                                 │
│   - 快速重传                                 │
│   - FEC（可选）                              │
├─────────────────────────────────────────────┤
│ 加密层：KCP BlockCrypt                       │
├─────────────────────────────────────────────┤
│ 网络层：UDP                                  │
└─────────────────────────────────────────────┘
```

## 源码实现

### Kcptun Client

```go
// 文件: transport/kcptun/client.go
package kcptun

const Mode = "kcptun"

type Client struct {
    once   sync.Once
    config Config
    block  kcp.BlockCrypt
    ctx    context.Context
    cancel context.CancelFunc
    numconn uint16
    muxes   []timedSession
}

func NewClient(config Config) *Client {
    config.FillDefaults()
    block := config.NewBlock()
    
    ctx, cancel := context.WithCancel(context.Background())
    
    return &Client{
        config: config,
        block:  block,
        ctx:    ctx,
        cancel: cancel,
    }
}

func (c *Client) createConn(ctx context.Context, dial DialFn) (*smux.Session, error) {
    conn, addr, err := dial(ctx)
    
    // 创建 KCP 连接
    convid := randv2.Uint32()
    kcpconn, err := kcp.NewConn4(convid, addr, c.block, 
        config.DataShard, config.ParityShard, true, conn)
    
    // KCP 配置
    kcpconn.SetStreamMode(true)
    kcpconn.SetNoDelay(config.NoDelay, config.Interval, config.Resend, config.NoCongestion)
    kcpconn.SetWindowSize(config.SndWnd, config.RcvWnd)
    kcpconn.SetMtu(config.MTU)
    
    // Smux 配置
    smuxConfig := smux.DefaultConfig()
    smuxConfig.Version = config.SmuxVer
    smuxConfig.MaxReceiveBuffer = config.SmuxBuf
    smuxConfig.KeepAliveInterval = time.Duration(config.KeepAlive) * time.Second
    
    // 压缩
    var netConn net.Conn = kcpconn
    if !config.NoComp {
        netConn = NewCompStream(netConn)
    }
    
    // Smux 会话
    return smux.Client(netConn, smuxConfig)
}
```

### Config 默认值

```go
// 文件: transport/kcptun/common.go
func (config *Config) FillDefaults() {
    if config.Key == "" {
        config.Key = "it's a secrect"
    }
    if config.Crypt == "" {
        config.Crypt = "aes"
    }
    if config.MTU == 0 {
        config.MTU = 1350
    }
    if config.SndWnd == 0 {
        config.SndWnd = 128
    }
    if config.RcvWnd == 0 {
        config.RcvWnd = 512
    }
    if config.DataShard == 0 {
        config.DataShard = 10
    }
    if config.ParityShard == 0 {
        config.ParityShard = 3
    }
    
    // Mode 配置 KCP 参数
    switch config.Mode {
    case "normal":
        config.NoDelay, config.Interval, config.Resend, config.NoCongestion = 0, 40, 2, 1
    case "fast":
        config.NoDelay, config.Interval, config.Resend, config.NoCongestion = 0, 30, 2, 1
    case "fast2":
        config.NoDelay, config.Interval, config.Resend, config.NoCongestion = 1, 20, 2, 1
    case "fast3":
        config.NoDelay, config.Interval, config.Resend, config.NoCongestion = 1, 10, 2, 1
    }
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 计划支持 Kcptun 传输，当前为待实现状态。

```cpp
// Prism Kcptun 计划实现
// 文件: src/prism/transport/kcptun.hpp（计划）
namespace psm::transport::kcptun {

struct kcptun_config {
    std::string key;
    std::string crypt = "aes";
    std::string mode = "fast";
    int mtu = 1350;
    int sndwnd = 128;
    int rcvwnd = 512;
    int datashard = 10;
    int parityshard = 3;
    bool nocomp = false;
    int keepalive = 10;
};

class kcptun_layer : public transport_layer {
    // KCP + Smux 实现
};

} // namespace psm::transport::kcptun
```

## 使用建议

### 场景选择

Kcptun 适用场景：
- 不稳定网络（丢包率高）
- 需要低延迟传输
- UDP 可用的环境

不适用场景：
- UDP 受限的网络
- 带宽成本敏感场景
- 高丢包环境（超过 30%）

### 参数优化

根据网络条件调整：
- **稳定网络**：mode=fast, sndwnd=128
- **一般网络**：mode=fast2, sndwnd=256
- **不稳定网络**：mode=fast3, sndwnd=512, 增加 FEC

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/mux/smux|Smux]] — Smux 多路复用
- [[dev/udp|UDP 传输]] — UDP 传输开发
- [[dev/tcp|TCP 传输]] — TCP 传输开发