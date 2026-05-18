---
title: "Simple-Obfs Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/simple-obfs"
tags: [mihomo, simple-obfs, http, tls, 混淆, shadowsocks]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, obfuscation]
layer: ref
---

# Simple-Obfs Transport

**类别**: Mihomo 传输层参考

## 概述

Simple-Obfs 是 Shadowsocks 的经典混淆插件，提供 HTTP 和 TLS 两种混淆模式。通过将 Shadowsocks 流量伪装为 HTTP 或 TLS 流量，可以对抗基于流量特征的检测。

Simple-Obfs 支持两种模式：
- **HTTP 模式**：流量伪装为 HTTP 请求
- **TLS 模式**：流量伪装为 TLS 握手和数据

## 配置格式

### Mihomo 配置

```yaml
# HTTP 模式
proxies:
  - name: "simple-obfs-http"
    type: ss
    server: example.com
    port: 80
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: obfs
    plugin-opts:
      mode: http        # HTTP 混淆
      host: www.bing.com  # 伪装 Host

# TLS 模式
proxies:
  - name: "simple-obfs-tls"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: obfs
    plugin-opts:
      mode: tls         # TLS 混淆
      host: www.bing.com  # 伪装 SNI
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| mode | string | 是 | 混淆模式（http/tls） |
| host | string | 否 | 伪装域名（默认 windowsupdate.microsoft.com） |

## 协议原理

### HTTP 模式

HTTP 模式将 Shadowsocks 流量伪装为 WebSocket Upgrade 请求：

```
HTTP 混淆流程：
┌─────────────────────────────────────────────┐
│ 首次请求：                                   │
│ GET / HTTP/1.1                              │
│ Host: www.bing.com                          │
│ User-Agent: curl/7.x.x                      │
│ Upgrade: websocket                          │
│ Connection: Upgrade                         │
│ Sec-WebSocket-Key: [随机 Base64]            │
│ Content-Length: [数据长度]                   │
│ [Shadowsocks 数据]                           │
│                                             │
│ 首次响应：                                   │
│ HTTP/1.1 101 Switching Protocols            │
│ Upgrade: websocket                          │
│ Connection: Upgrade                         │
│ Sec-WebSocket-Accept: [响应 Key]            │
│ [Shadowsocks 数据]                           │
│                                             │
│ 后续传输：直接传输数据                        │
└─────────────────────────────────────────────┘
```

### TLS 模式

TLS 模式构造伪造的 TLS ClientHello：

```
TLS 混淆流程：
┌─────────────────────────────────────────────┐
│ 首次请求：伪造 TLS ClientHello               │
│ ┌─────────────────────────────────────────┐ │
│ │ Handshake Type: 0x01 (ClientHello)      │ │
│ │ Version: TLS 1.2                        │ │
│ │ Random: [时间戳 + 28 bytes 随机]         │ │
│ │ Session ID: [32 bytes 随机]              │ │
│ │ Cipher Suites: [浏览器列表]              │ │
│ │ Extensions:                              │ │
│ │   - Session Ticket: [SS 数据]            │ │
│ │   - Server Name: [伪装域名]              │ │
│ │   - ...                                  │ │
│ └─────────────────────────────────────────┘ │
│                                             │
│ 后续数据：TLS Application Data 格式          │
│ ┌─────────────────────────────────────────┐ │
│ │ Content Type: 0x17 (Application Data)   │ │
│ │ Version: 0x03 0x03                      │ │
│ │ Length: [数据长度]                        │ │
│ │ [Shadowsocks 数据]                        │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## 源码实现

### HTTP Obfs 实现

```go
// 文件: transport/simple-obfs/http.go
package obfs

type HTTPObfs struct {
    net.Conn
    host          string
    port          string
    buf           []byte
    offset        int
    firstRequest  bool
    firstResponse bool
}

func (ho *HTTPObfs) Write(b []byte) (int, error) {
    if ho.firstRequest {
        // 构造 HTTP Upgrade 请求
        randBytes := make([]byte, 16)
        rand.Read(randBytes)
        
        req, _ := http.NewRequest("GET", fmt.Sprintf("http://%s/", ho.host), bytes.NewBuffer(b))
        req.Header.Set("User-Agent", fmt.Sprintf("curl/7.%d.%d", randv2.Int()%54, randv2.Int()%2))
        req.Header.Set("Upgrade", "websocket")
        req.Header.Set("Connection", "Upgrade")
        req.Header.Set("Sec-WebSocket-Key", base64.URLEncoding.EncodeToString(randBytes))
        req.ContentLength = int64(len(b))
        
        err = req.Write(ho.Conn)
        ho.firstRequest = false
        return len(b), err
    }
    
    return ho.Conn.Write(b)
}

func (ho *HTTPObfs) Read(b []byte) (int, error) {
    if ho.firstResponse {
        // 解析 HTTP 101 响应
        buf := pool.Get(pool.RelayBufferSize)
        n, err := ho.Conn.Read(buf)
        idx := bytes.Index(buf[:n], []byte("\r\n\r\n"))
        if idx == -1 {
            return 0, io.EOF
        }
        ho.firstResponse = false
        // 返回响应体中的数据
        return copy(b, buf[idx+4:n]), nil
    }
    
    return ho.Conn.Read(b)
}
```

### TLS Obfs 实现

```go
// 文件: transport/simple-obfs/tls.go
package obfs

const chunkSize = 1 << 14  // 16KB

type TLSObfs struct {
    net.Conn
    server        string
    remain        int
    firstRequest  bool
    firstResponse bool
}

func (to *TLSObfs) write(b []byte) (int, error) {
    if to.firstRequest {
        // 构造伪造的 ClientHello
        helloMsg := makeClientHelloMsg(b, to.server)
        _, err := to.Conn.Write(helloMsg)
        to.firstRequest = false
        return len(b), err
    }
    
    // TLS Application Data 格式
    buf := pool.GetBuffer()
    buf.Write([]byte{0x17, 0x03, 0x03})
    binary.Write(buf, binary.BigEndian, uint16(len(b)))
    buf.Write(b)
    _, err := to.Conn.Write(buf.Bytes())
    
    return len(b), err
}

func makeClientHelloMsg(data []byte, server string) []byte {
    // 构造完整的 TLS ClientHello
    // 将 SS 数据嵌入 Session Ticket Extension
    // ...
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 完整支持 Simple-Obfs HTTP 和 TLS 模式。

```cpp
// Prism Simple-Obfs 实现
// 文件: src/prism/transport/simple_obfs.hpp
namespace psm::transport::simple_obfs {

enum class obfs_mode { http, tls };

class simple_obfs_layer : public transport_layer {
public:
    simple_obfs_layer(
        obfs_mode mode,
        std::string_view host
    );
    
    auto connect(net::tcp::socket& underlying)
        -> net::awaitable<void> override;
    
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t> override;
    
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t> override;
    
private:
    obfs_mode mode_;
    std::string host_;
    bool first_request_ = true;
    bool first_response_ = true;
};

} // namespace psm::transport::simple_obfs
```

### HTTP Upgrade 构造

```cpp
// 构造 HTTP Upgrade 请求
auto build_http_upgrade_request(
    std::string_view host,
    std::span<const std::byte> initial_data
) -> std::vector<std::byte> {
    
    std::ostringstream oss;
    oss << "GET / HTTP/1.1\r\n";
    oss << "Host: " << host << "\r\n";
    oss << "User-Agent: curl/7." << rand() % 54 << "." << rand() % 2 << "\r\n";
    oss << "Upgrade: websocket\r\n";
    oss << "Connection: Upgrade\r\n";
    
    // 随机 WebSocket Key
    auto key = base64_encode(random_bytes(16));
    oss << "Sec-WebSocket-Key: " << key << "\r\n";
    oss << "Content-Length: " << initial_data.size() << "\r\n";
    oss << "\r\n";
    
    auto header = oss.str();
    std::vector<std::byte> result(header.size() + initial_data.size());
    std::copy(header.begin(), header.end(), result.begin());
    std::copy(initial_data.begin(), initial_data.end(), result.begin() + header.size());
    
    return result;
}
```

## 使用建议

### 模式选择

| 模式 | 适用端口 | 伪装效果 | 性能 |
|------|---------|---------|------|
| HTTP | 80 | 中 | 高 |
| TLS | 443 | 高 | 中 |

### Host 选择

推荐伪装域名：
- `www.bing.com`
- `www.microsoft.com`
- `www.google.com`
- CDN 常见域名

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/v2ray-plugin|V2ray-Plugin]] — WebSocket 插件
- [[ref/mihomo/transport/shadowtls|ShadowTLS]] — TLS 握手伪装