---
title: "RestLS Transport"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/transport/restls"
tags: [mihomo, restls, tls, fingerprint, 指纹伪装]
created: 2026-05-17
updated: 2026-05-17
related: [shadowsocks, tls-fingerprint, stealth]
---

# RestLS Transport

**类别**: Mihomo 传输层参考

## 概述

RestLS 是一种 TLS 客户端指纹伪装传输层，通过模拟常见浏览器/客户端的 TLS 指纹（ClientHello），使代理流量看起来像是正常的 HTTPS 连接。与 ShadowTLS 不同，RestLS 专注于 ClientHello 指纹的精确模拟。

RestLS 使用 `restls-client-go` 库实现 TLS 指纹伪装，支持多种客户端指纹模板：
- Chrome（默认）
- Firefox
- Safari
- Edge

## 配置格式

### Mihomo 配置

```yaml
proxies:
  - name: "restls-node"
    type: ss
    server: example.com
    port: 443
    cipher: aes-128-gcm
    password: "ss-password"
    plugin: restls
    plugin-opts:
      host: "www.google.com"      # TLS 目标域名
      password: "restls-password" # RestLS 密码
      client-id: "chrome"         # 指纹模板（可选）
```

### 配置参数说明

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| host | string | 是 | TLS ServerName |
| password | string | 是 | RestLS 认证密钥 |
| client-id | string | 否 | TLS 指纹模板（默认 chrome） |

### Client ID 选项

| Client ID | 指纹特征 | 推荐场景 |
|-----------|---------|----------|
| chrome | Chrome 最新版指纹 | 通用 |
| firefox | Firefox 指纹 | 欧美地区 |
| safari | Safari/iOS 指纹 | 移动端 |
| edge | Edge 指纹 | Windows 环境 |

## 协议原理

### TLS 指纹伪装

```
RestLS 工作原理：
┌─────────────────────────────────────────────┐
│                                             │
│  标准 TLS ClientHello:                       │
│  ┌───────────────────────────────────────┐  │
│  │ Random Cipher Suites Extensions       │  │
│  │ (随机) (固定列表) (UA/支持的协议)       │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  RestLS 伪装 ClientHello:                    │
│  ┌───────────────────────────────────────┐  │
│  │ Random Cipher Suites Extensions       │  │
│  │ (模拟) (浏览器顺序) (浏览器特征)        │  │
│  │  ↑ 精确模拟目标浏览器 TLS 指纹          │  │
│  └───────────────────────────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

### 指纹要素

RestLS 伪装的关键 TLS 指纹要素：
1. **Cipher Suites 顺序**：浏览器特定的加密套件优先级
2. **Extensions**：浏览器支持的 TLS 扩展
3. **ALPN**：应用层协议协商（h2, http/1.1）
4. **签名算法**：证书签名算法优先级
5. **版本号**：TLS 版本协商

## 源码实现

### Mihomo 实现

```go
// 文件: transport/restls/restls.go
package restls

import (
    "context"
    "net"
    tls "github.com/metacubex/restls-client-go"
)

const (
    Mode string = "restls"
)

type Restls struct {
    *tls.UConn
}

type Config = tls.Config

var NewRestlsConfig = tls.NewRestlsConfig

// NewRestls return a Restls Connection
func NewRestls(ctx context.Context, conn net.Conn, config *Config) (net.Conn, error) {
    clientHelloID := tls.HelloChrome_Auto  // 默认 Chrome 指纹
    if config != nil {
        clientIDPtr := config.ClientID.Load()
        if clientIDPtr != nil {
            clientHelloID = *clientIDPtr
        }
    }
    
    restls := &Restls{
        UConn: tls.UClient(conn, config, clientHelloID),
    }
    
    if err := restls.HandshakeContext(ctx); err != nil {
        return nil, err
    }
    
    return restls, nil
}
```

### restls-client-go 指纹模板

```go
// 预定义的 TLS ClientHello 指纹模板
var HelloChrome_Auto = HelloID{
    Version:    0x0303,  // TLS 1.2
    CipherSuites: []uint16{
        TLS_AES_128_GCM_SHA256,
        TLS_AES_256_GCM_SHA384,
        TLS_CHACHA20_POLY1305_SHA256,
        // ... Chrome 特定顺序
    },
    Extensions: ChromeExtensions,
    ALPN: []string{"h2", "http/1.1"},
}

var HelloFirefox_Auto = HelloID{
    Version:    0x0303,
    CipherSuites: FirefoxCipherSuites,
    Extensions: FirefoxExtensions,
    ALPN: []string{"h2", "http/1.1"},
}
```

## 与 Prism 兼容性

### Prism 支持

Prism 支持 RestLS 传输层，使用 UTLS 库实现 TLS 指纹伪装。

```cpp
// Prism RestLS 实现
// 文件: src/prism/transport/restls.hpp
namespace psm::transport::restls {

enum class client_fingerprint {
    chrome,
    firefox,
    safari,
    edge,
    ios,
    android
};

class restls_layer : public transport_layer {
public:
    restls_layer(
        std::string_view host,
        std::string_view password,
        client_fingerprint fingerprint = client_fingerprint::chrome
    );
    
    auto connect(
        net::tcp::socket& underlying
    ) -> net::awaitable<void> override;
    
private:
    std::string host_;
    client_fingerprint fingerprint_;
    utls::uconn uconn_;
};

} // namespace psm::transport::restls
```

### UTLS 指纹选择

```cpp
// 选择 TLS 指纹模板
auto select_fingerprint(client_fingerprint fp) 
    -> utls::hello_id {
    switch (fp) {
        case client_fingerprint::chrome:
            return utls::hello_chrome_auto;
        case client_fingerprint::firefox:
            return utls::hello_firefox_auto;
        case client_fingerprint::safari:
            return utls::hello_safari_auto;
        case client_fingerprint::edge:
            return utls::hello_edge_auto;
        default:
            return utls::hello_chrome_auto;
    }
}
```

## 使用建议

### 指纹选择策略

根据目标环境选择：
- **中国大陆**：Chrome（最常见）
- **欧美地区**：Firefox 或 Chrome
- **移动网络**：Safari/iOS 或 Android
- **企业网络**：Edge

### 与 ShadowTLS 对比

| 特性 | RestLS | ShadowTLS |
|------|--------|-----------|
| 伪装方式 | ClientHello 指纹 | TLS 握手包 |
| 密码验证 | 无（可选） | HMAC-SHA1 |
| 指纹精度 | 高 | 中 |
| 配置复杂度 | 低 | 低 |
| 性能开销 | 低 | 中 |

## 相关链接

- [[ref/mihomo/transport/overview|Transport 概览]] — Transport 模块总览
- [[ref/mihomo/transport/shadowtls|ShadowTLS]] — TLS 握手伪装
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹检测原理
- [[stealth/restls|RestLS 协议]] — RestLS 设计原理