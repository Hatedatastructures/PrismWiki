---
title: "ShadowTLS Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/shadowtls"
tags: [mihomo, shadowtls, tls, 伪装, shadowsocks]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, tls, stealth]
---

# ShadowTLS Transport

**类别**: Mihomo 传输层参考

## 概述

ShadowTLS 是一种 TLS 伪装传输层，通过在 Shadowsocks 代理流量前包裹真实的 TLS 握手，使流量看起来像是正常的 TLS 连接。这种技术可以有效对抗基于 TLS 指纹的检测。

ShadowTLS 的核心原理：
1. 发起真实的 TLS ClientHello（连接到伪装服务器）
2. 在 TLS 应用层数据中嵌入 Shadowsocks 流量
3. 使用 HMAC-SHA1 验证数据完整性

## 配置格式

### Mihomo 配置

```yaml
proxies:
  - name: "shadowtls-node"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: shadow-tls
    plugin-opts:
      host: "www.bing.com"        # TLS 伪装目标
      password: "shadowtls-password"  # ShadowTLS 密码
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| host | string | 是 | TLS 握手伪装目标域名 |
| password | string | 是 | ShadowTLS 认证密码 |

## 协议原理

### TLS 伪装机制

```
ShadowTLS 工作流程：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |--- TLS ClientHello ---->|               │
│   |    (伪装为 www.bing.com)|               │
│   |                         |               │
│   |<-- TLS ServerHello -----|               │
│   |                         |               │
│   |--- TLS Application Data -->|           │
│   |    包含 HMAC + SS 数据    |             │
│   |                         |               │
│   |<-- TLS Application Data --|            │
│   |    包含 HMAC + SS 数据    |             │
│   |                         |               │
└─────────────────────────────────────────────┘
```

### 数据帧格式

ShadowTLS 在 TLS Application Data 中封装数据：

```
TLS Application Data Frame：
┌─────────────────────────────────────────────┐
│ TLS Header: 0x17 0x03 0x03                  │
│ Length: 2 bytes (大端序)                     │
│ HMAC Hash: 8 bytes (首次发送)                │
│ Payload: Shadowsocks 数据                    │
└─────────────────────────────────────────────┘
```

### HMAC 验证

首次请求携带 HMAC-SHA1 哈希值：
- 用于验证连接合法性
- 哈希长度：8 bytes
- 密钥：ShadowTLS password

## 源码实现

### Mihomo 实现

```go
// 文件: transport/shadowtls/shadowtls.go
package shadowtls

const (
    chunkSize           = 1 << 13  // 8KB
    Mode         string = "shadow-tls"
    hashLen      int    = 8
    tlsHeaderLen int    = 5
)

type ShadowTLS struct {
    net.Conn
    password     []byte
    remain       int
    firstRequest bool
    tlsConfig    *tls.Config
}

func (s *ShadowTLS) Write(b []byte) (int, error) {
    length := len(b)
    for i := 0; i < length; i += chunkSize {
        end := i + chunkSize
        if end > length {
            end = length
        }
        n, err := s.write(b[i:end])
        if err != nil {
            return n, err
        }
    }
    return length, nil
}

func (s *ShadowTLS) write(b []byte) (int, error) {
    var hashVal []byte
    if s.firstRequest {
        // 首次请求：计算 HMAC
        hashedConn := newHashedStream(s.Conn, s.password)
        tlsConn := tls.Client(hashedConn, s.tlsConfig)
        if err := tlsConn.HandshakeContext(ctx); err != nil {
            return 0, err
        }
        hashVal = hashedConn.hasher.Sum(nil)[:hashLen]
        s.firstRequest = false
    }
    
    // 构造 TLS Application Data 帧
    buf.Write([]byte{0x17, 0x03, 0x03})
    binary.Write(buf, binary.BigEndian, uint16(len(b)+len(hashVal)))
    buf.Write(hashVal)
    buf.Write(b)
    
    return len(b), nil
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 完整支持 ShadowTLS 传输层。

```cpp
// Prism ShadowTLS 实现
// 文件: src/prism/transport/shadowtls.hpp
namespace psm::transport::shadowtls {

class shadowtls_layer : public transport_layer {
public:
    shadowtls_layer(
        std::string_view host,
        std::string_view password
    );
    
    auto connect(
        net::tcp::socket& underlying,
        const config& cfg
    ) -> net::awaitable<void> override;
    
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t> override;
    
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t> override;
    
private:
    std::string host_;
    std::string password_;
    bool first_request_ = true;
};

} // namespace psm::transport::shadowtls
```

### HMAC 计算

```cpp
// HMAC-SHA1 计算首次握手验证
auto compute_handshake_hash(
    std::span<const std::byte> handshake_data,
    std::string_view password
) -> std::array<std::byte, 8> {
    
    hmac_sha1 ctx(password);
    ctx.update(handshake_data);
    auto full_hash = ctx.final();
    
    // 取前 8 bytes
    std::array<std::byte, 8> result;
    std::copy(full_hash.begin(), full_hash.begin() + 8, result.begin());
    
    return result;
}
```

## 使用建议

### 伪装域名选择

推荐选择：
- 大型 CDN 网站（如 www.bing.com）
- 流量特征稳定的网站
- TLS 1.2/1.3 支持的网站

避免选择：
- 小流量网站
- 特殊协议网站
- 已被标记的网站

### 性能影响

- 首次连接：额外 TLS 握手开销
- 后续传输：每帧 5 bytes + 首次 8 bytes HMAC
- 分块大小：8KB（可调整）

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/restls|RestLS]] — TLS 指纹伪装
- [[core/stealth/shadowtls|ShadowTLS 协议]] — ShadowTLS 设计原理
- [[core/crypto/hmac-sha1|HMAC-SHA1]] — HMAC 认证算法