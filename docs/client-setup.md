---
title: 客户端配置
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 客户端配置

Mihomo/Clash 客户端连接 Prism 的配置示例。

## 功能简介

提供各协议客户端配置示例，包括 Trojan、VLESS、SS2022、Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel，以及 mihomo 完整配置流程。

## Mihomo 客户端完整配置流程

### 1. 安装 mihomo

```bash
# 从 GitHub Releases 下载对应平台的 mihomo 二进制
# Linux
curl -Lo mihomo.tar.gz https://github.com/MetaCubeX/mihomo/releases/latest/download/mihomo-linux-amd64.tar.gz
tar -xzf mihomo.tar.gz
chmod +x mihomo

# macOS
brew install mihomo
```

### 2. 创建 mihomo 配置文件

```yaml
# config.yaml - mihomo 基础配置
mixed-port: 7890
allow-lan: false
log-level: info

dns:
  enable: true
  listen: 0.0.0.0:1053
  nameserver:
    - 8.8.8.8
    - 1.1.1.1

proxies:
  # 在此添加 Prism 节点（见下方协议示例）

proxy-groups:
  - name: PROXY
    type: select
    proxies:
      - Prism-Trojan
      - Prism-VLESS
      - DIRECT

rules:
  - GEOSITE,cn,DIRECT
  - GEOIP,cn,DIRECT
  - MATCH,PROXY
```

### 3. 启动 mihomo

```bash
./mihomo -d . -f config.yaml
```

### 4. 系统代理设置

- **Windows**: 设置 > 网络 > 代理 > 手动代理 `127.0.0.1:7890`
- **macOS**: 系统设置 > 网络 > 代理 > 配置 HTTP/SOCKS 代理
- **Linux**: 导出环境变量 `export http_proxy=http://127.0.0.1:7890`

## 协议示例

### Trojan

最基础的 Prism 协议，适合内网或信任网络环境。

```yaml
- name: "Prism-Trojan"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  udp: true
  skip-cert-verify: false
  sni: your.server.com
```

### VLESS + Reality

高安全级别，适合对抗 DPI 检测。

```yaml
- name: "Prism-VLESS-Reality"
  type: vless
  server: your.server.com
  port: 8081
  uuid: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  flow: xtls-rprx-vision
  tls: true
  servername: www.microsoft.com
  client-fingerprint: chrome
  reality-opts:
    public-key: "your-reality-public-key-base64"
    short-id: "a1b2c3d4"
  udp: true
```

### SS2022 + ShadowTLS v3

双层加密伪装，适合高审查网络环境。

```yaml
- name: "Prism-SS-STLS"
  type: ss
  server: your.server.com
  port: 8081
  cipher: 2022-blake3-aes-128-gcm
  password: "your-base64-encoding-of-32-byte-key"
  plugin: shadow-tls
  plugin-opts:
    host: www.apple.com
    password: "shadowtls_password"
    version: 3
  udp: true
```

### Trojan + Smux 多路复用

启用多路复用减少连接数。

```yaml
- name: "Prism-Trojan-Smux"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  udp: true
  sni: your.server.com
  smux:
    enabled: true
    protocol: smux
    max_connections: 4
    min_streams: 2
    max_streams: 8
```

### Trojan + TLS

启用 TLS 加密传输。

```yaml
- name: "Prism-Trojan-TLS"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  udp: true
  sni: your.server.com
  skip-cert-verify: false
  fingerprint: chrome
```

### Restls

适用于特定反检测场景。

```yaml
- name: "Prism-Restls"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  udp: true
  sni: your.server.com
  restls-script: "300?<100,1000~100,5000~100"
```

### AnyTLS

通用 TLS 隧道伪装。

```yaml
- name: "Prism-AnyTLS"
  type: trojan
  server: your.server.com
  port: 8081
  password: "prism"
  udp: true
  sni: your.server.com
  anytls:
    idle-check-interval: 30
    idle-traffic-count: 10
    min-idle-interval: 100
    max-idle-interval: 300
```

## 客户端配置验证

### 连通性测试

```bash
# 测试代理是否正常转发
curl -x http://127.0.0.1:7890 https://api.ipify.org

# 测试 DNS 解析
nslookup example.com 127.0.0.1 -port=1053
```

### mihomo 内置 API

```bash
# 查看当前活跃连接
curl http://127.0.0.1:9090/connections

# 查看节点延迟
curl http://127.0.0.1:9090/proxies/Prism-Trojan/delay?timeout=5000&url=http://www.gstatic.com/generate_204
```

### 常见问题排查

| 症状 | 可能原因 | 解决方法 |
|------|----------|----------|
| 连接超时 | 服务端未启动或防火墙拦截 | 检查 `netstat` 和防火墙规则 |
| TLS 握手失败 | SNI 不匹配或证书问题 | 确保 `sni` 与 `server_names` 一致 |
| 认证失败 | 密码不正确 | 核对 `password` / `uuid` 与服务端配置 |
| 流量被阻断 | 协议未启用 | 检查服务端 `protocol.*.enabled` |

## 常见问题

**Q: SNI 不匹配怎么办？**
A: 客户端 `servername` 或 `sni` 必须与服务端 `server_names` 一致。Reality 伪装下，客户端的 `servername` 必须匹配 Reality 配置的伪装域名。

**Q: 如何启用多路复用？**
A: 在节点配置中添加 `smux` 配置块并设置 `enabled: true`，同时确保服务端也启用了 Smux 或 Yamux。

**Q: 多个协议可以同时使用吗？**
A: 可以。在服务端 `protocol` 模块中启用多个协议，客户端分别配置对应节点即可。每个节点使用不同的协议类型。

**Q: udp 参数有什么用？**
A: 开启 `udp: true` 后，mihomo 会通过代理节点转发 UDP 流量（如 DNS 查询、游戏流量）。如果不需要 UDP 代理，关闭可节省资源。

## 相关页面

- [[ref/mihomo/index|mihomo]]
- [[ref/mihomo/proxy-groups/overview|proxy-groups]]
- [[core/protocol/trojan|Trojan 协议]]
- [[core/stealth/reality/handshake|Reality 伪装]]
- [[core/stealth/shadowtls|ShadowTLS 伪装]]
- [[docs/configuration|配置说明]]
