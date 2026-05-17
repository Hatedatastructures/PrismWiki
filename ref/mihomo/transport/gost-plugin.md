---
title: "Gost-Plugin Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/gost-plugin"
tags: [mihomo, gost-plugin, websocket, mux, smux, shadowsocks]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, websocket, smux]
---

# Gost-Plugin Transport

**类别**: Mihomo 传输层参考

## 概述

Gost-Plugin 是基于 GOST 项目的 WebSocket 传输插件，支持 WebSocket 传输和 Smux 多路复用。与 V2ray-Plugin 类似，但增加了内置的 Mux 支持，可以在 WebSocket 之上建立多路复用会话。

主要功能：
- WebSocket 传输
- TLS 加密支持
- Smux 多路复用
- ECH 支持

## 配置格式

### Mihomo 配置

```yaml
proxies:
  - name: "gost-plugin-node"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: gost-plugin
    plugin-opts:
      mode: websocket        # 传输模式
      tls: true              # 启用 TLS
      path: /gost            # WebSocket 路径
      host: example.com      # Host 头
      mux: true              # 启用 Smux 多路复用
      skip-cert-verify: false
      fingerprint: chrome    # TLS 指纹
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| mode | string | 是 | 传输模式（websocket） |
| tls | bool | 否 | 启用 TLS 加密 |
| path | string | 否 | WebSocket 路径 |
| host | string | 否 | Host 头 |
| mux | bool | 否 | 启用 Smux 多路复用 |
| skip-cert-verify | bool | 否 | 跳过证书验证 |
| fingerprint | string | 否 | TLS 指纹模拟 |

## 协议原理

### WebSocket + Smux

```
Gost-Plugin 协议栈：
┌─────────────────────────────────────────────┐
│ 应用层：Shadowsocks                          │
├─────────────────────────────────────────────┤
│ 多路复用层：Smux（可选）                      │
│   - Stream 1, Stream 2, ...                 │
├─────────────────────────────────────────────┤
│ 传输层：WebSocket                            │
│   - HTTP Upgrade                            │
│   - WebSocket Frames                        │
├─────────────────────────────────────────────┤
│ 加密层：TLS（可选）                          │
├─────────────────────────────────────────────┤
│ 网络层：TCP                                  │
└─────────────────────────────────────────────┘
```

### Mux 会话流程

```
Mux 模式工作流程：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- WebSocket Upgrade --->|              │
│   |                         |               │
│   |-- Smux SYN (Stream 1) -->|             │
│   |-- Smux SYN (Stream 2) -->|             │
│   |                         |               │
│   |-- Smux PSH (Stream 1) -->|             │
│   |    数据帧                |              │
│   |-- Smux PSH (Stream 2) -->|             │
│   |                         |               │
│   |<- Smux PSH (Stream 1) ---|              │
│   |                         |               │
│   |-- Smux FIN (Stream 1) -->|             │
│   |                         |               │
└─────────────────────────────────────────────┘
```

## 源码实现

### Mihomo 实现

```go
// 文件: transport/gost-plugin/websocket.go
package gost

import (
    "github.com/metacubex/smux"
)

type Option struct {
    Host           string
    Port           string
    Path           string
    Headers        map[string]string
    TLS            bool
    ECHConfig      *ech.Config
    SkipCertVerify bool
    Fingerprint    string
    Mux            bool
}

// muxConn 包装 smux.Stream
type muxConn struct {
    net.Conn
    session *smux.Session
}

func (m *muxConn) Close() error {
    streamErr := m.Conn.Close()
    sessionErr := m.session.Close()
    
    if streamErr != nil {
        return streamErr
    }
    return sessionErr
}

func NewGostWebsocket(ctx context.Context, conn net.Conn, option *Option) (net.Conn, error) {
    // WebSocket 配置
    config := &vmess.WebsocketConfig{
        Host:      option.Host,
        Port:      option.Port,
        Path:      option.Path,
        ECHConfig: option.ECHConfig,
        Headers:   header,
    }
    
    // TLS 配置
    if option.TLS {
        config.TLS = true
        config.TLSConfig, err = ca.GetTLSConfig(...)
    }
    
    conn, err = vmess.StreamWebsocketConn(ctx, conn, config)
    
    // Mux 多路复用
    if option.Mux {
        smuxConfig := smux.DefaultConfig()
        smuxConfig.KeepAliveDisabled = true
        
        session, err := smux.Client(conn, smuxConfig)
        if err != nil {
            return nil, err
        }
        
        stream, err := session.OpenStream()
        if err != nil {
            session.Close()
            return nil, err
        }
        
        return &muxConn{
            Conn:    stream,
            session: session,
        }, nil
    }
    
    return conn, nil
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 支持 Gost-Plugin WebSocket + Smux 模式。

```cpp
// Prism Gost-Plugin 实现
// 文件: src/prism/transport/gost_plugin.hpp
namespace psm::transport::gost_plugin {

struct gost_config {
    std::string host;
    std::string path;
    bool tls = false;
    bool mux = false;
    std::string fingerprint;
};

class gost_layer : public transport_layer {
public:
    explicit gost_layer(const gost_config& cfg);
    
    auto connect(net::tcp::socket& underlying)
        -> net::awaitable<void> override;
    
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t> override;
    
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t> override;
    
    auto close() -> net::awaitable<void> override;
    
private:
    gost_config cfg_;
    std::unique_ptr<mux_session> mux_session_;
    std::unique_ptr<mux_stream> mux_stream_;
};

} // namespace psm::transport::gost_plugin
```

### Mux 会话管理

```cpp
// Mux 模式连接
auto connect_with_mux(
    beast::websocket::stream<net::tcp::socket&>& ws
) -> net::awaitable<mux_stream> {
    
    // 创建 Smux 客户端会话
    auto session = smux::client(ws, smux::default_config());
    
    // 打开第一个 Stream
    auto stream = co_await session.open_stream();
    
    co_return stream;
}
```

## 使用建议

### Mux 配置

Mux 模式优势：
- 连接复用，减少 TCP/TLS 握手
- 多请求并行处理
- 资源效率更高

适用场景：
- 高并发请求
- 需要稳定长连接
- 服务器连接数有限

### 与 V2ray-Plugin 对比

| 特性 | Gost-Plugin | V2ray-Plugin |
|------|-------------|---------------|
| WebSocket | 支持 | 支持 |
| Mux | 内置 Smux | 可选 |
| HTTP Upgrade | 无 | 支持 |
| 配置复杂度 | 低 | 中 |

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/v2ray-plugin|V2ray-Plugin]] — WebSocket 插件
- [[ref/mihomo/mux/smux|Smux]] — Smux 多路复用
- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用总览