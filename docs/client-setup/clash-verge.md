---
title: Clash Verge 配置指南
created: 2026-05-23
updated: 2026-05-23
layer: docs
tags: [docs, client, clash-verge, guide]
---

# Clash Verge 配置指南

Clash Verge Rev 是基于 Mihomo（Clash Meta）内核的图形化代理客户端，支持 Prism 的所有协议。本文提供完整的配置指南。

## 安装 Clash Verge

### Windows

从 [GitHub Releases](https://github.com/clash-verge-rev/clash-verge-rev/releases) 下载安装包：

- `Clash.Verge_x.y.z_x64-setup.exe` -- 安装版
- `Clash.Verge_x.y.z_x64_portable.zip` -- 便携版

### macOS

```bash
# Homebrew 安装
brew install --cask clash-verge-rev
```

或从 GitHub Releases 下载 `Clash.Verge_x.y.z_x64.dmg`。

### Linux

从 GitHub Releases 下载对应架构的 `.deb` 或 `.AppImage` 文件。

## 基础配置

### 创建配置文件

在 Clash Verge 中，点击"配置" -> "新建"，选择"Local"类型，粘贴以下模板：

```yaml
# Prism 完整配置模板
# Clash Verge Rev + Mihomo 内核

# ===== 全局设置 =====
mixed-port: 7890          # HTTP + SOCKS5 混合端口
socks-port: 7891          # SOCKS5 独立端口
allow-lan: true           # 允许局域网设备连接
bind-address: '*'         # 监听所有网卡
mode: rule                # 规则模式
log-level: info           # 日志级别
external-controller: 127.0.0.1:9090  # API 端口

# 禁用全局 uTLS 指纹（ShadowTLS 需要标准 TLS）
global-client-fingerprint: none

# ===== DNS 设置 =====
dns:
  enable: true
  ipv6: false             # 禁用 IPv6 查询
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
```

## 协议配置模板

以下配置覆盖 Prism 支持的所有协议。将 `your.server.com` 替换为实际服务器地址，凭据替换为服务端配置的值。

### Trojan 协议

```yaml
proxies:
  # Trojan TCP（基础）
  - name: "Trojan TCP"
    type: trojan
    server: your.server.com
    port: 8081
    password: "your-trojan-password"
    udp: false
    skip-cert-verify: true

  # Trojan UDP
  - name: "Trojan UDP"
    type: trojan
    server: your.server.com
    port: 8081
    password: "your-trojan-password"
    udp: true
    skip-cert-verify: true

  # Trojan + Smux 多路复用
  - name: "Trojan Smux TCP"
    type: trojan
    server: your.server.com
    port: 8081
    password: "your-trojan-password"
    udp: false
    skip-cert-verify: true
    smux:
      enabled: true
      protocol: smux
      only-tcp: true

  # Trojan + Yamux 多路复用
  - name: "Trojan Yamux TCP"
    type: trojan
    server: your.server.com
    port: 8081
    password: "your-trojan-password"
    udp: false
    skip-cert-verify: true
    smux:
      enabled: true
      protocol: yamux
      only-tcp: true
```

### VLESS + Reality

```yaml
proxies:
  # VLESS Reality TCP（推荐，最高安全性）
  - name: "VLESS Reality TCP"
    type: vless
    server: your.server.com
    port: 8081
    uuid: "your-uuid-here"
    network: tcp
    tls: true
    udp: false
    skip-cert-verify: true
    servername: www.microsoft.com           # SNI: 与服务端 reality.server_names 一致
    client-fingerprint: chrome              # TLS 指纹伪装
    reality-opts:
      public-key: "your-reality-public-key" # 公钥（不是私钥！）
      short-id: "your-short-id-hex"         # 与服务端 short_ids 一致

  # VLESS Reality UDP
  - name: "VLESS Reality UDP"
    type: vless
    server: your.server.com
    port: 8081
    uuid: "your-uuid-here"
    network: tcp
    tls: true
    udp: true
    skip-cert-verify: true
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: "your-reality-public-key"
      short-id: "your-short-id-hex"

  # VLESS Reality + Smux
  - name: "VLESS Reality Smux"
    type: vless
    server: your.server.com
    port: 8081
    uuid: "your-uuid-here"
    network: tcp
    tls: true
    udp: false
    skip-cert-verify: true
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: "your-reality-public-key"
      short-id: "your-short-id-hex"
    smux:
      enabled: true
      protocol: smux
      only-tcp: true

  # VLESS Reality + Yamux
  - name: "VLESS Reality Yamux"
    type: vless
    server: your.server.com
    port: 8081
    uuid: "your-uuid-here"
    network: tcp
    tls: true
    udp: false
    skip-cert-verify: true
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: "your-reality-public-key"
      short-id: "your-short-id-hex"
    smux:
      enabled: true
      protocol: yamux
      only-tcp: true
```

### SS2022 (Shadowsocks 2022)

```yaml
proxies:
  # SS2022 裸 TCP
  - name: "SS2022 TCP"
    type: ss
    server: your.server.com
    port: 8081
    cipher: 2022-blake3-aes-128-gcm
    password: "your-base64-psk"    # 16 字节 base64 = AES-128

  # SS2022 + ShadowTLS v3
  - name: "SS2022 ShadowTLS TCP"
    type: ss
    server: your.server.com
    port: 8081
    cipher: 2022-blake3-aes-128-gcm
    password: "your-base64-psk"
    plugin: shadow-tls
    plugin-opts:
      host: "www.apple.com"           # SNI: 与服务端 shadowtls.server_names 一致
      password: "shadowtls-password"   # 与服务端 shadowtls.users.password 一致
      version: 3

  # SS2022 + Restls TLS 1.3
  - name: "SS2022 Restls TLS13"
    type: ss
    server: your.server.com
    port: 8081
    cipher: 2022-blake3-aes-128-gcm
    password: "your-base64-psk"
    plugin: restls
    plugin-opts:
      host: "www.nvidia.com"          # SNI: 与服务端 restls.server_names 一致
      password: "restls-password"      # 与服务端 restls.password 一致
      version-hint: "tls13"
      restls-script: "300?100<1,400~100,350~100,600~100,300~200,300~100"

  # SS2022 + Restls TLS 1.2
  - name: "SS2022 Restls TLS12"
    type: ss
    server: your.server.com
    port: 8081
    cipher: 2022-blake3-aes-128-gcm
    password: "your-base64-psk"
    plugin: restls
    plugin-opts:
      host: "www.nvidia.com"
      password: "restls-password"
      version-hint: "tls12"
      restls-script: "1000?100<1,500~100,350~100,600~100,400~200"
```

### AnyTLS

```yaml
proxies:
  # AnyTLS TCP
  - name: "AnyTLS TCP"
    type: anytls
    server: your.server.com
    port: 8081
    password: "anytls-password"        # 与服务端 anytls.users.password 一致
    sni: "www.cloudflare.com"          # SNI: 与服务端 anytls.server_names 一致
    skip-cert-verify: true
    udp: false

  # AnyTLS UDP
  - name: "AnyTLS UDP"
    type: anytls
    server: your.server.com
    port: 8081
    password: "anytls-password"
    sni: "www.cloudflare.com"
    skip-cert-verify: true
    udp: true

  # AnyTLS + Smux
  - name: "AnyTLS Smux TCP"
    type: anytls
    server: your.server.com
    port: 8081
    password: "anytls-password"
    sni: "www.cloudflare.com"
    skip-cert-verify: true
    udp: false
    smux:
      enabled: true
      protocol: smux
      only-tcp: true

  # AnyTLS + Yamux
  - name: "AnyTLS Yamux TCP"
    type: anytls
    server: your.server.com
    port: 8081
    password: "anytls-password"
    sni: "www.cloudflare.com"
    skip-cert-verify: true
    udp: false
    smux:
      enabled: true
      protocol: yamux
      only-tcp: true
```

### SOCKS5

```yaml
proxies:
  # SOCKS5 无认证
  - name: "SOCKS5"
    type: socks5
    server: your.server.com
    port: 8081
    udp: true

  # SOCKS5 有认证
  - name: "SOCKS5 Auth"
    type: socks5
    server: your.server.com
    port: 8081
    username: "admin"
    password: "your-password"
    udp: true
```

### HTTP

```yaml
proxies:
  # HTTP 代理
  - name: "HTTP Proxy"
    type: http
    server: your.server.com
    port: 8081
    username: username
    password: "your-password"

  # HTTPS 代理
  - name: "HTTPS Proxy"
    type: http
    server: your.server.com
    port: 8081
    username: username
    password: "your-password"
    tls: true
    skip-cert-verify: true
```

## 完整配置示例

```yaml
# Prism 全协议完整配置
mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: rule
log-level: info
external-controller: 127.0.0.1:9090
global-client-fingerprint: none

dns:
  enable: true
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

proxies:
  - name: "VLESS Reality"
    type: vless
    server: your.server.com
    port: 8081
    uuid: "123e4567-e89b-12d3-a456-426614174000"
    tls: true
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: "your-public-key"
      short-id: "your-short-id"
    udp: true

  - name: "SS2022 ShadowTLS"
    type: ss
    server: your.server.com
    port: 8081
    cipher: 2022-blake3-aes-128-gcm
    password: "your-base64-psk"
    plugin: shadow-tls
    plugin-opts:
      host: "www.apple.com"
      password: "shadowtls-password"
      version: 3

  - name: "Trojan"
    type: trojan
    server: your.server.com
    port: 8081
    password: "your-password"
    udp: true
    skip-cert-verify: true

proxy-groups:
  - name: "Prism"
    type: select
    proxies:
      - "VLESS Reality"
      - "SS2022 ShadowTLS"
      - "Trojan"
      - DIRECT

  - name: "Auto"
    type: url-test
    proxies:
      - "VLESS Reality"
      - "SS2022 ShadowTLS"
      - "Trojan"
    url: "http://connect.rom.miui.com/generate_204"
    interval: 300

rules:
  - IP-CIDR6,::/0,REJECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - MATCH,Prism
```

## SNI 路由说明

Prism 使用 SNI 路由表进行协议识别，每个伪装方案使用不同的 SNI：

| SNI | 伪装方案 | 内层协议 |
|-----|----------|----------|
| www.microsoft.com | Reality | VLESS/Trojan |
| www.apple.com | ShadowTLS | SS2022/Trojan |
| www.nvidia.com | Restls | SS2022 |
| www.cloudflare.com | AnyTLS | 任意 |
| www.amazon.com | TrustTunnel | 任意 |

**重要**: 客户端 `sni`/`servername` 必须与服务端 `server_names` 配置一致！

## 常见问题

**Q: ShadowTLS 节点连接失败怎么办？**
A: 确认 `global-client-fingerprint: none` 已设置。ShadowTLS 需要标准 TLS，全局 uTLS 指纹会干扰。

**Q: Reality 连接超时？**
A: 检查 `public-key`（不是 private-key）和 `short-id` 是否与服务端一致。确认 `servername` 匹配。

**Q: SS2022 Restls 握手失败？**
A: 确认 `restls-script` 与服务端配置完全一致。`version-hint` 需要与服务端 `version_hint` 匹配。

**Q: 多路复用连接断开？**
A: Smux/Yamux 的 `max_streams` 不能超过服务端配置。检查心跳间隔配置是否合理。

## 相关页面

- [[docs/client-setup|客户端配置总览]]
- [[docs/client-setup/sing-box|sing-box 配置指南]]
- [[docs/protocol-matrix|协议能力矩阵]]
- [[docs/configuration|配置说明]]
- [[docs/troubleshooting|故障排查]]
