---
title: Reality
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, reality]
---
# Reality TLS 隐蔽技术

Reality 是一种先进的 TLS 隐蔽技术，由 Xray-core 项目首创。它通过将代理流量伪装成到真实网站的正常 TLS 连接，使主动探测（Active Probing）和流量指纹分析（Traffic Fingerprinting）无法区分代理流量和正常 HTTPS 流量。

## 技术概述

Reality 的核心设计理念是"不要伪造，要借用"——不同于传统的 TLS 伪装技术（如伪装成普通的自签名证书），Reality 直接将流量伪装成到真实知名网站（如微软、苹果、谷歌等）的 TLS 连接。其特性包括：

- **TLS 完全伪装**：客户端的 TLS Client Hello 与真实浏览器完全一致
- **真实网站借用**：握手阶段连接到真实的 TLS 服务器
- **X25519 密钥交换**：使用椭圆曲线 Diffie-Hellman 进行密钥协商
- **X25519MLKEM768 支持**：后量子安全密钥交换（混合经典 + 后量子）
- **Short ID 匹配**：服务端通过 Short ID 识别代理连接和正常连接
- **零额外证书**：不需要自签名证书或特殊 CA

Reality 解决了传统 TLS 伪装方案的几个根本问题：

1. **证书问题**：传统伪装需要自签名证书，易被指纹检测识别
2. **SNI 问题**：传统伪装的 SNI 指向代理服务器，与 TLS 指纹不匹配
3. **主动探测**：GFW 可以主动向代理服务器发起 TLS 握手，检测异常响应

## Reality TLS 隐蔽原理

### 传统 TLS 伪装的缺陷

传统的代理 TLS 伪装方案通常采用以下方式：

```
Client              Proxy Server
  |  TLS Client Hello   |
  |  SNI=proxy.server   |
  |  自定义证书          |
  |------------------->|
  |                     | GFW 检测到非标准证书
  |                     | 或 SNI 与指纹不匹配
```

这种方案的问题：
- 自签名证书容易被检测（证书链异常）
- SNI 指向代理服务器而非知名网站
- TLS 指纹与真实浏览器不一致
- 主动探测会暴露代理特征

### Reality 的工作流程

Reality 从根本上改变了这一模式：

```
阶段一：TLS 握手伪装
Client                    Reality Server              真实网站（Fallback）
  |  TLS Client Hello        |                            |
  |  SNI=microsoft.com       |                            |
  |  真实浏览器 TLS 指纹      |                            |
  |------------------------>|                            |
  |                          | 检查 Short ID              |
  |                          |                            |
  |                          | Short ID 不匹配 → 转发到真实网站
  |                          |-------------------------->|
  |                          |  真实网站的 TLS 响应         |
  |                          |<--------------------------|
  |  TLS Server Hello        |                            |
  |  真实网站的证书           |                            |
  |<------------------------|                            |
  |  正常 HTTPS 连接         |                            |

阶段二：代理连接（Short ID 匹配时）
Client                    Reality Server
  |  TLS Client Hello        |
  |  SNI=microsoft.com       |
  |  携带 Short ID           |
  |------------------------>|
  |                          | Short ID 匹配！
  |                          | 截断 TLS 握手
  |                          | 使用 X25519 派生共享密钥
  |  代理协议数据（加密）     |
  |------------------------>|
  |                          | 解密代理协议
  |                          | 转发到目标
```

### 关键隐蔽技术

1. **真实 TLS 指纹**：Reality 客户端使用真实浏览器的 TLS 指纹（JA3/JA4），包括：
   - Cipher Suites 列表与真实浏览器一致
   - Extension 顺序和类型完全匹配
   - Supported Groups、Key Shares 等参数一致
   - 压缩方法、签名算法等与真实浏览器无异

2. **合法 SNI**：SNI 设置为真实的知名网站域名（如 `www.microsoft.com`、`apple.com`），而非代理服务器地址。

3. **合法证书链**：在 Short ID 不匹配时，Reality 服务器将 TLS 握手转发到真实的 Fallback 网站，客户端收到的是该网站的真实证书。

4. **无额外 RTT**：与传统 VLESS Vision 等方案相比，Reality 在 TLS 握手内部完成认证，不增加额外的往返时间。

## X25519 密钥交换

### 原理

Reality 使用 X25519 椭圆曲线 Diffie-Hellman（ECDH）密钥交换协议来建立客户端和服务端之间的共享密钥：

```
客户端                              服务端
  |                                  |
  | 生成客户端密钥对                  |
  | (priv_c, pub_c)                  |
  |                                  | 生成服务器密钥对
  |                                  | (priv_s, pub_s)
  |                                  |
  | Client Hello                     |
  | 包含: pub_c, encrypted Short ID  |
  |--------------------------------->|
  |                                  |
  |                                  | 计算共享密钥:
  |                                  | shared = X25519(priv_s, pub_c)
  |                                  |
  |                                  | 验证 Short ID
  |                                  | 使用 shared 解密后续数据
  |
  | Server Hello
  | 包含: pub_s
  |<---------------------------------|
  |                                  |
  | 计算共享密钥:                     |
  | shared = X25519(priv_c, pub_s)   |
  |                                  |
  | 使用 shared 加密代理协议数据      |
```

### X25519 曲线参数

X25519 基于 Curve25519 椭圆曲线：

| 参数 | 值 |
|------|-----|
| 曲线 | Curve25519 (Montgomery 曲线) |
| 域大小 | 256 位 |
| 安全强度 | 128 位 |
| 公钥大小 | 32 字节 |
| 私钥大小 | 32 字节 |
| 算法 | Montgomery Ladder 标量乘法 |

X25519 的优势：
- **高性能**：比传统 ECDH（如 P-256）更快
- **安全**：无已知的实现漏洞
- **简洁**：API 简单，不易误用
- **广泛支持**：所有现代 TLS 库都支持

### 配置和密钥生成

服务端生成密钥对：

```bash
# 使用 xray 生成 Reality 密钥
xray x25519
# 输出:
# Private key: <base64-encoded-private-key>
# Public key: <base64-encoded-public-key>
```

客户端配置中使用公钥：

```yaml
proxies:
  - name: "vless-reality"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

### X25519MLKEM768（后量子安全）

X25519MLKEM768 是 X25519 和 ML-KEM-768（原 Kyber-768）的混合密钥交换方案：

```
shared_key = KDF(
    X25519(priv_c, pub_s) ||
    ML-KEM-768.EncapsulateDecapsulate()
)
```

这种混合方案确保：
- 即使量子计算机破解了 ML-KEM，X25519 仍然安全
- 即使 X25519 被攻破，ML-KEM 仍然安全
- 两者同时被攻破的概率极低

```yaml
reality-opts:
  public-key: public-key-base64url
  short-id: short-id-hex
  support-x25519mlkem768: true
```

## Short ID 匹配机制

### 原理

Short ID 是 Reality 用于区分"代理连接"和"正常 TLS 连接"的关键机制：

```
服务端收到 TLS Client Hello 时：

1. 提取 Client Hello 中的 Short ID
   Short ID 被编码在 TLS 扩展或特定字段中

2. 与预设的 Short ID 列表进行匹配

3. 匹配成功：
   - 截断正常 TLS 握手
   - 使用 X25519 共享密钥解密代理协议数据
   - 进入代理流程

4. 匹配失败：
   - 完成正常 TLS 握手到 Fallback 网站
   - 表现为到该网站的普通 HTTPS 连接
```

### Short ID 格式

Short ID 是一个 1-8 字节的十六进制值：

```yaml
reality-opts:
  short-id: "a1b2c3d4"    # 4 字节 Short ID
```

| 特性 | 说明 |
|------|------|
| 长度 | 1-8 字节（可变） |
| 格式 | 十六进制字符串 |
| 安全性 | 8 字节 Short ID 有 2^64 种可能，暴力破解不可行 |
| 多 ID 支持 | 服务端可配置多个 Short ID，每个客户端使用不同 ID |

### 安全性分析

Short ID 的安全性取决于其长度：

| Short ID 长度 | 搜索空间 | 安全性 |
|--------------|----------|--------|
| 1 字节 | 256 | 低（可暴力枚举） |
| 2 字节 | 65,536 | 中 |
| 4 字节 | 4,294,967,296 | 高 |
| 8 字节 | 18,446,744,073,709,551,616 | 极高 |

推荐使用至少 4 字节的 Short ID。更短的 Short ID 可能被主动探测攻击枚举。

### 多 Short ID 管理

服务端可以为不同客户端分配不同的 Short ID，实现：

1. **用户隔离**：每个客户端使用唯一 Short ID，便于追踪和管理
2. **吊销机制**：如果某个 Short ID 泄露，可以单独吊销而不影响其他客户端
3. **限流**：基于 Short ID 进行速率限制

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/reality.go` | Reality 配置解析和出站代理集成 |
| `component/tls/reality.go` | Reality TLS 实现，包括握手和密钥交换 |

## YAML 配置示例

### VLESS + Reality

```yaml
proxies:
  - name: "vless-reality"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

### Trojan + Reality

```yaml
proxies:
  - name: "trojan-reality"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

### VMess + Reality

```yaml
proxies:
  - name: "vmess-reality"
    type: vmess
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    client-fingerprint: chrome
```

## 客户端指纹

| Fingerprint | 描述 | 适用场景 |
|-------------|------|----------|
| `chrome` | Chrome TLS 指纹 | 最常用，兼容性最好 |
| `firefox` | Firefox TLS 指纹 | 备选方案 |
| `safari` | Safari TLS 指纹 | macOS/iOS 设备 |
| `edge` | Edge TLS 指纹 | Windows 环境 |
| `ios` | iOS TLS 指纹 | iOS 设备 |
| `android` | Android TLS 指纹 | Android 设备 |
| `randomized` | 随机化 TLS 指纹 | 最大化不可预测性 |

选择与你的使用场景匹配的指纹。如果使用 Chrome 浏览器上网，使用 `chrome` 指纹可以确保 Reality 流量与正常 Chrome 流量在 TLS 层面完全一致。

## Fallback 网站选择

选择合适的 Fallback 网站对 Reality 的隐蔽性至关重要：

| 推荐网站 | 原因 |
|----------|------|
| `www.microsoft.com` | 全球最大 TLS 流量来源之一，CDN 分布广泛 |
| `apple.com` | TLS 指纹丰富，证书链标准 |
| `www.google.com` | 流量大，不易被特别关注 |
| `cloudflare.com` | 支持多种 TLS 配置 |

选择 Fallback 时的注意事项：
- 网站必须支持 TLS 1.2 或更高
- 网站的 TLS 配置应支持客户端的 Cipher Suite
- 避免选择被 GFW 封锁的网站（否则 Fallback 也会失败）
- 优先选择 CDN 分布广泛的网站

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Reality 实现 |
| Pipeline | 完全兼容 | TLS 流 |
| Stealth | 完全兼容 | TLS 隐蔽技术 |
| Crypto | 完全兼容 | X25519 密钥交换 |

## 相关文档

- [[ech]] - ECH 加密 Client Hello
- [[core/stealth/reality/overview|reality]] - Reality 隐蔽技术详解
- [[core/crypto/x25519|x25519]] - X25519 密钥交换
- [[vless]] - VLESS 协议
