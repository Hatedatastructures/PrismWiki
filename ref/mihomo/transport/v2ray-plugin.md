---
title: "V2ray-Plugin Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/v2ray-plugin"
tags: [mihomo, v2ray-plugin, websocket, http-upgrade, shadowsocks]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, vmess, websocket]
layer: ref
---

# V2ray-Plugin Transport

**类别**: Mihomo 传输层参考

## 概述

V2ray-Plugin 是 V2Ray 的 WebSocket 传输层插件实现，用于 Shadowsocks 等协议。它支持 WebSocket 传输、TLS 加密、HTTP Upgrade 等功能，可以将代理流量伪装为正常的 WebSocket 流量或 HTTPS 流量。

主要功能：
- WebSocket 传输
- TLS 加密支持
- HTTP Upgrade 模式
- ECH 支持
- Mux 多路复用（可选）

## 配置格式

### Mihomo 配置

```yaml
proxies:
  - name: "v2ray-plugin-node"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: v2ray-plugin
    plugin-opts:
      mode: websocket        # 传输模式
      tls: true              # 启用 TLS
      path: /ray             # WebSocket 路径
      host: example.com      # Host 头
      headers:               # 自定义 Headers（可选）
        X-Header: value
      mux: false             # 启用 Mux（可选）
      v2ray-http-upgrade: false  # HTTP Upgrade 模式
      v2ray-http-upgrade-fast-open: false  # Fast Open
      skip-cert-verify: false    # 跳过证书验证
      fingerprint: chrome    # TLS 指纹（可选）
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| mode | string | 是 | 传输模式（websocket） |
| tls | bool | 否 | 启用 TLS 加密 |
| path | string | 否 | WebSocket 路径（默认 /） |
| host | string | 否 | Host 头（默认 server） |
| headers | map | 否 | 自定义 HTTP Headers |
| mux | bool | 否 | 启用多路复用 |
| v2ray-http-upgrade | bool | 否 | HTTP Upgrade 模式 |
| skip-cert-verify | bool | 否 | 跳过证书验证 |
| fingerprint | string | 否 | TLS 指纹模拟 |

## 协议原理

### WebSocket 传输

```
WebSocket 连接流程：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- HTTP GET /ray -------->|              │
│   |   Upgrade: websocket     |              │
│   |   Connection: Upgrade    |              │
│   |                         |               │
│   |-- HTTP 101 Switching ---|              │
│   |   升级为 WebSocket        |              │
│   |                         |               │
│   |-- WebSocket Frame ------>|              │
│   |   包含代理数据            |              │
│   |                         |               │
│   |<- WebSocket Frame -------|              │
│   |                         |               │
└─────────────────────────────────────────────┘
```

### HTTP Upgrade 模式

HTTP Upgrade 模式优化了 WebSocket 升级过程：

```
HTTP Upgrade 流程：
┌─────────────────────────────────────────────┐
│ 普通 WebSocket：                             │
│   GET /path → 101 → WebSocket Data         │
│   (需要等待 101 响应)                         │
│                                             │
│ HTTP Upgrade：                               │
│   GET /path + 数据 → 101 + 数据             │
│   (首个请求携带数据，减少延迟)                 │
└─────────────────────────────────────────────┘
```

## 源码实现

### Mihomo 实现

```go
// 文件: transport/v2ray-plugin/websocket.go
package obfs

type Option struct {
    Host                     string
    Port                     string
    Path                     string
    Headers                  map[string]string
    TLS                      bool
    ECHConfig                *ech.Config
    SkipCertVerify           bool
    Fingerprint              string
    Certificate              string
    PrivateKey               string
    Mux                      bool
    V2rayHttpUpgrade         bool
    V2rayHttpUpgradeFastOpen bool
}

func NewV2rayObfs(ctx context.Context, conn net.Conn, option *Option) (net.Conn, error) {
    header := http.Header{}
    for k, v := range option.Headers {
        header.Add(k, v)
    }
    
    config := &vmess.WebsocketConfig{
        Host:                     option.Host,
        Port:                     option.Port,
        Path:                     option.Path,
        V2rayHttpUpgrade:         option.V2rayHttpUpgrade,
        V2rayHttpUpgradeFastOpen: option.V2rayHttpUpgradeFastOpen,
        ECHConfig:                option.ECHConfig,
        Headers:                  header,
    }
    
    // TLS 配置
    if option.TLS {
        config.TLS = true
        config.TLSConfig, err = ca.GetTLSConfig(ca.Option{
            TLSConfig: &tls.Config{
                ServerName:         option.Host,
                InsecureSkipVerify: option.SkipCertVerify,
                NextProtos:         []string{"http/1.1"},
            },
            Fingerprint: option.Fingerprint,
        })
    }
    
    conn, err = vmess.StreamWebsocketConn(ctx, conn, config)
    
    // Mux 多路复用
    if option.Mux {
        conn = NewMux(conn, MuxOption{
            ID:   [2]byte{0, 0},
            Host: "127.0.0.1",
            Port: 0,
        })
    }
    
    return conn, nil
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 完整支持 V2ray-Plugin WebSocket 传输。

```cpp
// Prism WebSocket 实现
// 文件: src/prism/transport/websocket.hpp
namespace psm::transport::websocket {

struct websocket_config {
    std::string host;
    std::string port;
    std::string path;
    bool tls = false;
    bool http_upgrade = false;
    bool http_upgrade_fast_open = false;
    std::map<std::string, std::string> headers;
};

class websocket_layer : public transport_layer {
public:
    explicit websocket_layer(const websocket_config& cfg);
    
    auto connect(net::tcp::socket& underlying)
        -> net::awaitable<void> override;
    
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t> override;
    
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t> override;
    
private:
    websocket_config cfg_;
    beast::websocket::stream<net::tcp::socket&> ws_;
};

} // namespace psm::transport::websocket
```

### HTTP Upgrade 实现

```cpp
// HTTP Upgrade 优化
auto http_upgrade_handshake(
    beast::websocket::stream<net::tcp::socket&>& ws,
    const websocket_config& cfg,
    std::span<const std::byte> initial_data
) -> net::awaitable<void> {
    
    // 构造 HTTP Upgrade 请求
    http::request<http::string_body> req;
    req.method(http::verb::get);
    req.target(cfg.path);
    req.set(http::field::host, cfg.host);
    req.set(http::field::upgrade, "websocket");
    req.set(http::field::connection, "upgrade");
    
    // Fast Open：首个请求携带数据
    if (cfg.http_upgrade_fast_open && !initial_data.empty()) {
        req.body() = encode_websocket_frame(initial_data);
    }
    
    co_await ws.async_handshake(req, net::use_awaitable);
}
```

## 使用建议

### 配置优化

- **路径选择**：使用不明显的路径（如 `/api`, `/chat`）
- **Host 设置**：与服务器域名一致
- **TLS**：建议启用 TLS 加密
- **HTTP Upgrade**：适合低延迟场景

### 性能对比

| 模式 | 延迟 | 吞吐量 | 伪装效果 |
|------|------|--------|----------|
| WebSocket | 中 | 高 | 中 |
| WebSocket + TLS | 中 | 高 | 高 |
| HTTP Upgrade | 低 | 高 | 中 |
| HTTP Upgrade + TLS | 低 | 高 | 高 |

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/simple-obfs|Simple-Obfs]] — HTTP/TLS 混淆
- [[ref/mihomo/transport/gost-plugin|Gost-Plugin]] — WebSocket + Mux
- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用协议