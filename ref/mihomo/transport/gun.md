---
title: "Gun (gRPC) Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/gun"
tags: [mihomo, gun, grpc, http2, vless, trojan]
created: 2026-05-17
updated: 2026-05-17
related: [vless, trojan, http2]
layer: ref
---

# Gun (gRPC) Transport

**类别**: Mihomo 传输层参考

## 概述

Gun 是基于 gRPC/HTTP/2 的传输层实现，最初来源于 Qv2ray 的 gun-lite 项目。它将代理数据封装在 gRPC 消息中，通过 HTTP/2 流传输，使流量看起来像是正常的 gRPC API 请求。

Gun 主要用于 VLESS 和 Trojan 协议，提供：
- HTTP/2 多路复用
- TLS 加密
- 流量伪装为 gRPC API

## 配置格式

### VLESS + Gun

```yaml
proxies:
  - name: "vless-gun-node"
    type: vless
    server: example.com
    port: 443
    uuid: "uuid-string"
    tls: true
    servername: example.com
    grpc-service-name: "GunService"  # gRPC 服务名
    # 或使用 network: grpc
    network: grpc
    grpc-opts:
      grpc-service-name: "GunService"
```

### Trojan + Gun

```yaml
proxies:
  - name: "trojan-gun-node"
    type: trojan
    server: example.com
    port: 443
    password: "trojan-password"
    grpc-service-name: "GunService"
    # TLS 配置
    tls: true
    sni: example.com
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| grpc-service-name | string | 是 | gRPC 服务名称 |
| servername/sni | string | 否 | TLS SNI |
| tls | bool | 否 | 启用 TLS |

## 协议原理

### HTTP/2 gRPC 封装

```
Gun/gRPC 协议栈：
┌─────────────────────────────────────────────┐
│ 应用层：VLESS/Trojan                        │
├─────────────────────────────────────────────┤
│ RPC 层：gRPC                                 │
│   - Service: GunService                     │
│   - Method: Tun                             │
│   - Protobuf 消息封装                        │
├─────────────────────────────────────────────┤
│ HTTP/2 层：                                  │
│   - Stream                                   │
│   - Headers: content-type: application/grpc │
│   - Data Frames                              │
├─────────────────────────────────────────────┤
│ TLS 层：                                     │
│   - ALPN: h2                                 │
├─────────────────────────────────────────────┤
│ 网络层：TCP                                  │
└─────────────────────────────────────────────┘
```

### gRPC 消息格式

```
gRPC 消息帧格式：
┌─────────────────────────────────────────────┐
│ Compressed-Flag: 1 byte (0x00)              │
│ Message-Length: 4 bytes (大端序)             │
│ Protobuf-Payload:                           │
│   ┌─────────────────────────────────────┐   │
│   │ Field 1 (Tag: 0x0A): 代理数据        │   │
│   │ Length: UVarint                      │   │
│   │ Data: VLESS/Trojan 数据              │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### HTTP/2 流程

```
Gun HTTP/2 流程：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- TLS Handshake (ALPN=h2) -->|          │
│   |                         |               │
│   |-- HTTP/2 HEADERS -------->|             │
│   |   POST /GunService/Tun    |             │
│   |   content-type: grpc      |             │
│   |                         |               │
│   |-- HTTP/2 DATA帧 ---------->|            │
│   |   gRPC消息(代理数据)       |             │
│   |                         |               │
│   |<- HTTP/2 DATA帧 -----------|            │
│   |   gRPC消息(响应数据)       |             │
│   |                         |               │
│   |-- HTTP/2 RST_STREAM ----->|             │
│   |                         |               │
└─────────────────────────────────────────────┘
```

## 源码实现

### Gun Conn 实现

```go
// 文件: transport/gun/gun.go
package gun

var defaultHeader = http.Header{
    "content-type": []string{"application/grpc"},
    "user-agent":   []string{"grpc-go/1.36.0"},
}

type Conn struct {
    initFn func() (io.ReadCloser, NetAddr, error)
    writer io.Writer
    closer io.Closer
    NetAddr
    initOnce sync.Once
    reader   io.ReadCloser
    br       *bufio.Reader
    remain   int
}

type Config struct {
    ServiceName string
    UserAgent   string
    Host        string
}

func (g *Conn) Read(b []byte) (n int, err error) {
    if err = g.Init(); err != nil {
        return
    }
    
    // 解析 gRPC 消息
    // 0x00 grpclength(uint32) 0x0A uleb128 payload
    _, err = g.br.Discard(6)
    if err != nil {
        return 0, err
    }
    
    protobufPayloadLen, err := binary.ReadUvarint(g.br)
    
    // 读取 protobuf payload
    return io.ReadFull(g.br, b[:size])
}

func (g *Conn) Write(b []byte) (int, error) {
    // 构造 protobuf header
    protobufHeader := [binary.MaxVarintLen64 + 1]byte{0x0A}
    varuintSize := binary.PutUvarint(protobufHeader[1:], uint64(len(b)))
    
    // gRPC header
    var grpcHeader [5]byte
    grpcPayloadLen := uint32(varuintSize + 1 + len(b))
    binary.BigEndian.PutUint32(grpcHeader[1:5], grpcPayloadLen)
    
    buf := pool.GetBuffer()
    buf.Write(grpcHeader[:])
    buf.Write(protobufHeader[:varuintSize+1])
    buf.Write(b)
    
    _, err := g.writer.Write(buf.Bytes())
    
    // Flush HTTP/2
    if flusher, ok := g.writer.(http.Flusher); ok {
        flusher.Flush()
    }
    
    return len(b), err
}
```

### HTTP/2 Client

```go
// 文件: transport/gun/gun.go
func NewHTTP2Client(dialFn DialFn, tlsConfig *vmess.TLSConfig) *TransportWrap {
    dialFunc := func(ctx context.Context, network, addr string, cfg *tls.Config) (net.Conn, error) {
        pconn, err := dialFn(ctx, network, addr)
        
        if tlsConfig == nil {
            return pconn, nil
        }
        
        conn, err := vmess.StreamTLSConn(ctx, pconn, tlsConfig)
        
        // 验证 ALPN
        if tlsConfig.Reality == nil {
            state := tlsConn.ConnectionState()
            if p := state.NegotiatedProtocol; p != http.Http2NextProtoTLS {
                return nil, fmt.Errorf("unexpected ALPN %s", p)
            }
        }
        
        return conn, nil
    }
    
    transport := &http.Http2Transport{
        DialTLSContext:     dialFunc,
        AllowHTTP:          false,
        DisableCompression: true,
    }
    
    return wrap
}

func ServiceNameToPath(serviceName string) string {
    if strings.HasPrefix(serviceName, "/") {
        return serviceName
    }
    return "/" + serviceName + "/Tun"
}
```

## 与 Prism 充容性

### Prism 支持

Prism 完整支持 Gun/gRPC 传输。

```cpp
// Prism Gun/gRPC 实现
// 文件: src/prism/transport/gun.hpp
namespace psm::transport::gun {

struct gun_config {
    std::string service_name = "GunService";
    std::string user_agent = "grpc-go/1.36.0";
    std::string host;
};

class gun_layer : public transport_layer {
public:
    explicit gun_layer(const gun_config& cfg);
    
    auto connect(net::tcp::socket& underlying)
        -> net::awaitable<void> override;
    
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t> override;
    
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t> override;
    
private:
    gun_config cfg_;
    http2::stream h2_stream_;
};

} // namespace psm::transport::gun
```

### HTTP/2 连接

```cpp
// HTTP/2 + gRPC 连接
auto connect_http2_grpc(
    net::tcp::socket& socket,
    const gun_config& cfg
) -> net::awaitable<void> {
    
    // TLS 握手（ALPN=h2）
    auto tls_sock = co_await tls::handshake(socket, cfg.host, {"h2"});
    
    // HTTP/2 连接
    h2_stream_ = co_await http2::connect(tls_sock);
    
    // 发送 gRPC HEADERS
    http2::headers_frame headers;
    headers.set(http2::field::method, "POST");
    headers.set(http2::field::path, "/" + cfg.service_name + "/Tun");
    headers.set("content-type", "application/grpc");
    headers.set("user-agent", cfg.user_agent);
    
    co_await h2_stream_.send_headers(headers);
}
```

### gRPC 消息封装

```cpp
// gRPC 消息封装
auto encode_grpc_message(
    std::span<const std::byte> data
) -> std::vector<std::byte> {
    
    std::vector<std::byte> frame;
    
    // Compressed-Flag
    frame.push_back(std::byte{0x00});
    
    // Protobuf Header
    frame.push_back(std::byte{0x0A});  // Field 1 tag
    append_varint(frame, data.size());  // Length
    
    // Payload
    frame.insert(frame.end(), data.begin(), data.end());
    
    // Message Length (4 bytes, 大端序)
    uint32_t msg_len = frame.size() - 1;  // 不含 compressed-flag
    frame.insert(frame.begin() + 1, encode_be32(msg_len));
    
    return frame;
}
```

## 使用建议

### Service Name 选择

推荐使用不明显的服务名：
- `GunService`
- `TunnelService`
- `MediaService`
- 自定义路径如 `/api/tunnel`

### 性能特点

| 特性 | Gun/gRPC |
|------|----------|
| 多路复用 | HTTP/2 原生 |
| 头部开销 | 低（6+Varint bytes） |
| 伪装效果 | 高（gRPC API） |
| TLS 要求 | 必须启用 |

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/v2ray-plugin|V2ray-Plugin]] — WebSocket 插件
- [[dev/http2|HTTP/2 传输]] — HTTP/2 开发指南
- [[dev/tls|TLS 传输]] — TLS 开发指南