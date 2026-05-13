---
title: mihomo Clash 配置
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [proxy-client, configuration, mihomo, clash]
sources:
  - https://wiki.metacubex.one/config/
  - https://github.com/MetaCubeX/mihomo
confidence: high
---

# mihomo Clash 配置文件

mihomo（原 Clash Meta）使用 YAML 格式的配置文件，定义代理节点、规则、DNS、TUN 等所有行为。本文档是配置文件的完整解析参考。

## 配置文件结构总览

```yaml
# 基础设置
mixed-port: 7890
socks-port: 7891
port: 7890
allow-lan: false
bind-address: '*'
mode: rule
log-level: info
ipv6: false

# 外部控制（Web 面板）
external-controller: 127.0.0.1:9090
external-ui: ui
secret: ""

# 代理节点
proxies: []
proxy-groups: []

# 规则
rules: []

# DNS
dns: {}

# TUN
tun: {}

# 主机
hosts: {}

# Profile
profile:
  store-selected: true

# GEO 数据库
geox-url:
  geoip: "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip-lite.dat"
  geosite: "https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"

# 实验性功能
experimental: {}
```

## 基础设置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mixed-port` | int | 7890 | HTTP(S)+SOCKS5 混合代理端口 |
| `socks-port` | int | 7891 | 纯 SOCKS5 代理端口 |
| `port` | int | 7890 | 纯 HTTP 代理端口 |
| `allow-lan` | bool | false | 允许局域网连接（同网段其他设备可使用） |
| `bind-address` | string | `*` | 绑定地址，`*` 表示所有接口 |
| `mode` | string | rule | 模式：`rule`(规则分流) / `global`(全局) / `direct`(直连) |
| `log-level` | string | info | 日志级别：`silent`/`error`/`warning`/`info`/`debug` |
| `ipv6` | bool | false | 是否启用 IPv6 |
| `find-process-mode` | string | off | 进程匹配模式：`off`/`strict`/`always` |
| `global-client-fingerprint` | string | chrome | 全局 TLS 指纹伪装：`chrome`/`firefox`/`edge`/`safari`/`ios`/`android`/`random` |
| `unified-delay` | bool | true | 统一延迟测试（去除 TLS 握手时间） |
| `tcp-concurrent` | bool | true | TCP 并发连接（Happy Eyeballs） |

## 代理节点（proxies）

### 通用字段

所有代理类型共有的字段：

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 节点名称（唯一） |
| `type` | 是 | 协议类型 |
| `server` | 是 | 服务器地址 |
| `port` | 是 | 服务器端口 |
| `udp` | 否 | 是否支持 UDP（默认 false） |

### Shadowsocks

```yaml
- name: "ss-node"
  type: ss
  server: example.com
  port: 8388
  cipher: aes-256-gcm
  password: "your_password"
  udp: true
  # 可选
  plugin: obfs-local
  plugin-opts:
    mode: http
    host: www.example.com
```

**cipher（加密方式）：**
- `aes-128-gcm`, `aes-192-gcm`, `aes-256-gcm` — AEAD，推荐
- `chacha20-ietf-poly1305` — AEAD，移动端推荐
- `xchacha20-ietf-poly1305` — SS2022 使用
- `2022-blake3-aes-128-gcm`, `2022-blake3-aes-256-gcm`, `2022-blake3-chacha20-poly1305` — SS2022

**plugin：**
- `obfs-local` — HTTP/TLS 混淆
- `v2ray-plugin` — V2Ray 插件（WebSocket、gRPC）

### VMess

```yaml
- name: "vmess-node"
  type: vmess
  server: example.com
  port: 443
  uuid: "your-uuid"
  alterId: 0
  cipher: auto
  tls: true
  servername: example.com  # SNI
  # 传输层
  network: ws
  ws-opts:
    path: /path
    headers:
      Host: example.com
  # 或 gRPC
  # network: grpc
  # grpc-opts:
  #   grpc-service-name: grpc
```

**cipher：** `auto`/`aes-128-gcm`/`chacha20-ietf-poly1305`/`none`

**network（传输方式）：**
- 不填或 `tcp` — 原始 TCP
- `ws` — WebSocket
- `h2` — HTTP/2
- `grpc` — gRPC
- `http` — HTTP 伪装

### VLESS

```yaml
- name: "vless-node"
  type: vless
  server: example.com
  port: 443
  uuid: "your-uuid"
  tls: true
  servername: example.com
  # Reality
  reality-opts:
    public-key: "your_public_key"
    short-id: "your_short_id"
  # 传输层
  network: grpc
  grpc-opts:
    grpc-service-name: grpc
  # 或 ws
  # network: ws
  # ws-opts:
  #   path: /path
  # flow: xtls-rprx-vision  # 流控模式
```

**flow（流控）：**
- `xtls-rprx-vision` — XTLS Vision 模式（推荐）
- 不填 — 普通模式

### Trojan

```yaml
- name: "trojan-node"
  type: trojan
  server: example.com
  port: 443
  password: "your_password"
  # TLS
  sni: example.com
  alpn:
    - h2
    - http/1.1
  skip-cert-verify: false
  # 传输层
  network: ws
  ws-opts:
    path: /path
    headers:
      Host: example.com
  # UDP
  udp: true
```

### TUIC

```yaml
- name: "tuic-node"
  type: tuic
  server: example.com
  port: 443
  token: "your_token"
  # 或 uuid + password
  # uuid: "your-uuid"
  # password: "your_password"
  congestion-controller: bbr
  udp-relay-mode: native
  alpn:
    - h3
  sni: example.com
  skip-cert-verify: false
```

### Hysteria

```yaml
- name: "hysteria-node"
  type: hysteria
  server: example.com
  port: 443
  auth: "your_password"
  # 或 auth-str
  # auth-str: "your_password"
  up: 50 Mbps
  down: 100 Mbps
  sni: example.com
  alpn:
    - h3
  skip-cert-verify: false
  # 混淆
  obfs: obfs
  obfs-password: "your_password"
```

### Hysteria 2

```yaml
- name: "hysteria2-node"
  type: hysteria2
  server: example.com
  port: 443
  password: "your_password"
  up: 50 Mbps
  down: 100 Mbps
  sni: example.com
  skip-cert-verify: false
  # 混淆
  obfs: salamander
  obfs-password: "your_password"
```

### ShadowsocksR (SSR)

```yaml
- name: "ssr-node"
  type: ssr
  server: example.com
  port: 8388
  cipher: aes-256-cfb
  password: "your_password"
  protocol: auth_aes128_md5
  protocol-param: ""
  obfs: tls1.2_ticket_auth
  obfs-param: ""
```

### WireGuard

```yaml
- name: "wg-node"
  type: wireguard
  server: example.com
  port: 51820
  ip: 10.0.0.2
  ipv6: ""
  private-key: "your_private_key"
  public-key: "server_public_key"
  pre-shared-key: ""
  mtu: 1420
  udp: true
```

### SSH

```yaml
- name: "ssh-node"
  type: ssh
  server: example.com
  port: 22
  username: user
  password: "your_password"
  # 或私钥
  # private-key: |
  #   -----BEGIN OPENSSH PRIVATE KEY-----
  #   ...
  #   -----END OPENSSH PRIVATE KEY-----
  # private-key-passphrase: ""
```

### 直连与拒绝

```yaml
- name: "DIRECT"
  type: direct

- name: "REJECT"
  type: reject

- name: "COMPATIBLE"
  type: compatible

- name: "PASS"
  type: pass
```

## 代理组（proxy-groups）

代理组是节点选择和策略的核心：

### select（手动选择）

```yaml
- name: "Proxy"
  type: select
  proxies:
    - "Node-A"
    - "Node-B"
    - "Node-C"
    - "DIRECT"
  # 默认选中的节点
  use:
    - "my-provider"
```

### url-test（自动选优）

```yaml
- name: "Auto"
  type: url-test
  proxies:
    - "Node-A"
    - "Node-B"
    - "Node-C"
  url: "http://www.gstatic.com/generate_204"
  interval: 300        # 测试间隔（秒）
  tolerance: 50        # 容差（毫秒），延迟差在此范围内不切换
  lazy: true           # 有流量时才测试
```

### fallback（故障转移）

```yaml
- name: "Fallback"
  type: fallback
  proxies:
    - "Node-A"
    - "Node-B"
  url: "http://www.gstatic.com/generate_204"
  interval: 300
  lazy: true
```

按列表顺序使用，第一个可用的即为当前节点。

### load-balance（负载均衡）

```yaml
- name: "Balance"
  type: load-balance
  proxies:
    - "Node-A"
    - "Node-B"
  url: "http://www.gstatic.com/generate_204"
  interval: 300
  strategy: consistent-hashing  # 策略
```

**strategy：**
- `round-robin` — 轮询
- `consistent-hashing` — 一致性哈希（同一目标走同一节点）
- `sticky-sessions` — 粘性会话

### relay（链式代理）

```yaml
- name: "Chain"
  type: relay
  proxies:
    - "Entry-Node"
    - "Middle-Node"
    - "Exit-Node"
```

流量依次经过 Entry → Middle → Exit，实现多跳代理。

### fallback + url-test 组合示例

```yaml
- name: "Smart-Proxy"
  type: fallback
  proxies:
    - name: "Auto-Fast"
      type: url-test
      proxies:
        - "Node-A"
        - "Node-B"
      url: "http://www.gstatic.com/generate_204"
      interval: 300
    - "Node-C"  # 备用
  url: "http://www.gstatic.com/generate_204"
  interval: 60
```

## 规则（rules）

规则决定流量走哪个代理组。

### 规则类型

| 规则 | 格式 | 说明 |
|------|------|------|
| DOMAIN | `DOMAIN,google.com,Proxy` | 精确匹配域名 |
| DOMAIN-SUFFIX | `DOMAIN-SUFFIX,google.com,Proxy` | 域名后缀匹配 |
| DOMAIN-KEYWORD | `DOMAIN-KEYWORD,google,Proxy` | 域名关键词匹配 |
| GEOIP | `GEOIP,CN,DIRECT` | GeoIP 匹配 |
| GEOSITE | `GEOSITE,google,Proxy` | GeoSite 匹配 |
| IP-CIDR | `IP-CIDR,127.0.0.0/8,DIRECT` | IP 段匹配 |
| IP-CIDR6 | `IP-CIDR6,::1/128,DIRECT` | IPv6 段匹配 |
| SRC-IP-CIDR | `SRC-IP-CIDR,192.168.0.0/16,DIRECT` | 源 IP 匹配 |
| SRC-PORT | `SRC-PORT,1234,DIRECT` | 源端口匹配 |
| DST-PORT | `DST-PORT,443,Proxy` | 目标端口匹配 |
| PROCESS-NAME | `PROCESS-NAME,chrome.exe,Proxy` | 进程名匹配 |
| RULE-SET | `RULE-SET,my-rules,Proxy` | 规则集 |
| AND | `AND,((DOMAIN,google.com),(DST-PORT,443)),Proxy` | 逻辑与 |
| OR | `OR,((DOMAIN,google.com),(DOMAIN,youtube.com)),Proxy` | 逻辑或 |
| NOT | `NOT,((DOMAIN,google.com)),DIRECT` | 逻辑非 |
| MATCH | `MATCH,DIRECT` | 兜底规则（必须放最后） |

### 规则集（rule-providers）

规则集是外部规则文件，支持远程更新：

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://github.com/..."
    path: ./ruleset/reject.yaml
    interval: 86400

  proxy:
    type: http
    behavior: domain
    url: "https://github.com/..."
    path: ./ruleset/proxy.yaml
    interval: 86400

  direct:
    type: http
    behavior: domain
    url: "https://github.com/..."
    path: ./ruleset/direct.yaml
    interval: 86400

rules:
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,Proxy
  - RULE-SET,direct,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

**behavior：** `domain`(域名) / `ipcidr`(IP段) / `classical`(传统规则格式)

## DNS 配置

详见 [[client/mihomo-dns]] 页面。核心配置：

```yaml
dns:
  enable: true
  listen: 0.0.0.0:53
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "localhost.ptlogin2.qq.com"
  default-nameserver:        # 用于解析 nameserver 中的域名
    - 223.5.5.5
    - 119.29.29.29
  nameserver:                # 国内 DNS
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:                  # 国外 DNS（防污染）
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4
    domain:
      - "+.google.com"
      - "+.facebook.com"
```

### fake-ip-filter

fake-ip 模式下返回真实 IP 的域名列表（某些服务不兼容 fake-ip）：

```yaml
fake-ip-filter:
  - "*.lan"
  - "*.local"
  - "*.localhost"
  - "localhost.ptlogin2.qq.com"
  - "dns.msftncsi.com"        # Windows 网络检测
  - "www.msftncsi.com"
  - "www.msftconnecttest.com"
  - "time.*.com"
  - "time.*.gov"
  - "time.*.edu.cn"
  - "time.*.apple.com"
  - "time-ios.apple.com"
  - "time-macos.apple.com"
  - "*.ntp.org.cn"
  - "+.pool.ntp.org"
  - "time1.cloud.tencent.com"
```

## TUN 配置

TUN 模式通过虚拟网卡劫持系统级流量（包括 UDP 和非代理协议应用）：

```yaml
tun:
  enable: true
  stack: mixed              # gvisor / system / mixed
  dns-hijack:
    - any:53                # 劫持所有 DNS 查询
  auto-route: true          # 自动配置路由
  auto-detect-interface: true  # 自动检测出口网卡
  mtu: 9000
  strict-route: true
  # Windows 专用
  # windows:
  #   interface-name: "Clash"
```

**stack 选择：**
- `gvisor` — 用户态网络栈，兼容性好
- `system` — 使用系统网络栈，性能好
- `mixed` — 自动选择

## hosts 配置

本地 DNS 映射，优先级最高：

```yaml
hosts:
  "*.example.com": 127.0.0.1
  "test.local": 192.168.1.100
  "domain.com": "::1"
```

## Profile 配置

```yaml
profile:
  store-selected: true       # 保存每个代理组的选择
  store-fake-ip: true        # 保存 fake-ip 映射（重启不丢失）
```

## 完整配置示例

一个典型的 mihomo 配置文件结构：

```yaml
# === 基础 ===
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: false
global-client-fingerprint: chrome

# === 外部控制 ===
external-controller: 127.0.0.1:9090
secret: ""

# === DNS ===
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN

# === 代理 ===
proxies:
  - name: "Trojan-Reality"
    type: trojan
    server: example.com
    port: 443
    password: "pass"
    sni: www.example.com
    udp: true

  - name: "VLESS-Reality"
    type: vless
    server: example.com
    port: 443
    uuid: "uuid"
    tls: true
    servername: www.example.com
    reality-opts:
      public-key: "key"
      short-id: "id"
    flow: xtls-rprx-vision

  - name: "Hysteria2"
    type: hysteria2
    server: example.com
    port: 443
    password: "pass"
    up: 50 Mbps
    down: 100 Mbps

# === 代理组 ===
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "Trojan-Reality"
      - "VLESS-Reality"
      - "Hysteria2"
      - "DIRECT"

  - name: "Auto"
    type: url-test
    proxies:
      - "Trojan-Reality"
      - "VLESS-Reality"
      - "Hysteria2"
    url: http://www.gstatic.com/generate_204
    interval: 300

# === 规则 ===
rules:
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,github.com,Proxy
  - DOMAIN-SUFFIX,baidu.com,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,Proxy

# === TUN ===
tun:
  enable: true
  stack: mixed
  dns-hijack:
    - any:53
  auto-route: true
```

## 相关页面

- [[mihomo-Meta]] — mihomo 项目介绍
- [[client/mihomo-dns]] — DNS 配置详解
- [[client/tun]] — TUN 模式配置
- [[protocol/proxy-protocols]] — 各协议的技术细节
- [[stealth/reality]] — Reality 伪装配置
- [[dev/tcp]] — TCP 传输优化
- [[dev/udp]] — UDP 代理处理
