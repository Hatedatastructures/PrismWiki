---
title: Direct
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, direct]
---
# Direct 直连协议

Direct 是 mihomo 中的直接连接协议类型，它使流量绕过所有代理节点，直接通过本地网络接口建立到目标服务器的 TCP 或 UDP 连接。该协议是代理规则系统中不可或缺的基础组件，与 Reject、DNS 等内置协议共同构成 mihomo 的路由决策终端。

## 技术概述

Direct 协议的核心原理是将出站连接请求直接交给操作系统的网络协议栈，不经过任何中间代理层。其特性包括：

- **直接网络连接**：通过操作系统 socket API 建立连接，无额外协议封装开销
- **支持 TCP 和 UDP**：同时处理面向连接和无连接的数据传输
- **DNS 解析集成**：使用本地或配置的 DNS resolver 进行域名解析
- **回环检测**：防止代理流量循环导致的死锁
- **多接口支持**：可绑定指定网络接口或源 IP 地址
- **零额外延迟**：相比代理模式，省去握手、加密、转发等环节

Direct 在 mihomo 的出站适配器体系中属于"终端节点"类型——它不封装任何数据，也不建立任何隧道，仅仅是将连接请求透传给操作系统。

## 直连原理

### 连接建立流程

当一条流量规则匹配到 Direct 代理时，mihomo 执行以下步骤：

```
应用层请求
    ↓
规则引擎匹配 → DIRECT 规则
    ↓
Direct 适配器接收 DialRequest
    ↓
调用 net.Dial / net.DialUDP
    ↓
操作系统 TCP/IP 协议栈
    ↓
目标服务器
```

1. **域名解析**：通过配置的 DNS resolver 将目标域名解析为 IP 地址。如果启用 Fake-IP 模式，则需要将 Fake-IP 映射回真实域名再解析。
2. **Socket 创建**：根据目标协议类型（TCP/UDP）调用相应的系统调用。TCP 使用 `connect()` 建立三次握手，UDP 直接绑定本地端口。
3. **数据透传**：建立连接后，mihomo 的 Direct 适配器仅作为数据中转桥梁，将上游应用的数据原封不动地写入 socket，并将 socket 的响应数据返回给应用。

### TCP 直连

TCP 直连通过标准 TCP 三次握手建立连接：

```
Client (mihomo)          Server
    |  SYN                  |
    |--------------------->|
    |  SYN-ACK              |
    |<---------------------|
    |  ACK                  |
    |--------------------->|
    |  数据传输              |
```

Direct 适配器在此过程中不干预握手参数，所有 TCP 选项（MSS、窗口缩放、SACK 等）由操作系统协商。

### UDP 直连

UDP 直连无需握手，直接通过 `sendto()` / `recvfrom()` 系统调用收发数据包。mihomo 会为每个 UDP 会话创建一个独立的 socket，并维护会话状态以正确映射响应数据到对应的上游请求。

## 使用场景

### 白名单模式

白名单模式是 Direct 协议最典型的应用场景。在这种模式下，默认所有流量走代理（使用 `MATCH` 或 `FINAL` 规则指向代理节点），只有明确列出的域名或 IP 段走 Direct：

```yaml
rules:
  # 国内网站直连
  - DOMAIN-SUFFIX,cn,DIRECT
  - DOMAIN-KEYWORD,bilibili,DIRECT
  - DOMAIN-SUFFIX,qq.com,DIRECT
  - DOMAIN-SUFFIX,163.com,DIRECT

  # 本地服务直连
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve

  # 兜底规则
  - MATCH,PROXY
```

这种模式适用于代理服务器位于海外、主要访问海外资源的用户。国内网站直连可以避免不必要的延迟和带宽消耗。

### 本地服务访问

Direct 协议确保对本地网络服务的访问不被意外代理：

- **路由器管理界面**（192.168.1.1）
- **本地 DNS 服务器**（127.0.0.1:53）
- **局域网 NAS**（192.168.x.x）
- **开发服务器**（localhost:8080）
- **打印机等 IoT 设备**

```yaml
rules:
  # 回环地址直连
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR6,::1/128,DIRECT,no-resolve

  # RFC 1918 私有地址直连
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve

  # 链路本地地址
  - IP-CIDR,169.254.0.0/16,DIRECT,no-resolve
```

### 域名分流

结合 GeoIP 数据库实现基于地理位置的智能分流：

```yaml
rules:
  - GEOSITE,cn,DIRECT
  - GEOIP,cn,DIRECT
  - MATCH,PROXY
```

mihomo 使用 GeoIP 数据库判断目标 IP 所属的国家或地区，中国大陆 IP 走 Direct，其余走代理。

## 与代理的区别

### 对比表

| 特性 | Direct | 代理模式（Shadowsocks/VLESS 等） |
|------|--------|--------------------------------|
| 网络路径 | 本地 → 目标服务器 | 本地 → 代理服务器 → 目标服务器 |
| 延迟 | 仅网络物理延迟 | 额外增加代理节点跳数延迟 |
| 加密 | 依赖目标协议（如 HTTPS） | 代理协议额外加密层 |
| IP 可见性 | 目标服务器看到真实 IP | 目标服务器看到代理出口 IP |
| DNS 解析 | 使用本地 resolver | 可能使用远程 DNS |
| 带宽消耗 | 仅目标流量 | 目标流量 + 代理协议开销 |
| 适用场景 | 国内网站、本地服务 | 海外网站、被封锁资源 |

### 与 COMPATIBLE 的关系

`COMPATIBLE` 是 Direct 的一种特殊变体，主要用于兼容旧版 Clash 配置。它与标准 Direct 的区别在于：

- `DIRECT`：明确的直连指令，规则引擎主动匹配
- `COMPATIBLE`：兜底直连，当没有其他规则匹配时的默认行为

```yaml
proxies:
  - name: "DIRECT"
    type: direct

  - name: "COMPATIBLE"
    type: direct
```

在大多数配置中，两者可以互换使用，但推荐明确使用 `DIRECT` 以保持配置语义清晰。

## 回环检测机制

回环检测是 Direct 协议的关键安全特性。当 TUN 模式或透明代理模式下，如果没有回环检测，可能出现以下死锁循环：

```
应用发出请求
    ↓
TUN 设备捕获
    ↓
mihomo 处理
    ↓
匹配到 DIRECT 规则
    ↓
Direct 尝试连接目标
    ↓
DNS 解析请求再次进入 TUN 设备  ← 回环！
    ↓
mihomo 再次处理...（死锁）
```

mihomo 通过以下机制防止回环：

1. **进程标记**：为 mihomo 自身进程发出的连接设置标记，Direct 适配器识别后直接放行
2. **UID/GID 过滤**：Linux 上通过 socket 的 SO_MARK 选项标记流量
3. **路由表分离**：使用策略路由将 mihomo 流量排除在 TUN 路由规则之外
4. **Fake-IP 过滤**：在 Fake-IP 模式下，Direct 不处理 Fake-IP 范围内的地址

## DNS 解析策略

Direct 协议的 DNS 解析行为受以下配置影响：

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  fake-ip-range: 198.18.0.1/16

  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  # 直连域名使用特定 DNS
  nameserver-policy:
    "geosite:cn":
      - https://dns.alidns.com/dns-query
    "*":
      - https://1.1.1.1/dns-query
```

- **普通模式**：Direct 使用配置的 `nameserver` 进行域名解析
- **Fake-IP 模式**：Direct 收到 Fake-IP 后，通过映射表找到真实域名，再执行解析
- **no-resolve 选项**：规则中设置 `no-resolve` 时，Direct 跳过 DNS 解析，直接使用 IP 连接

## 接口绑定

可以指定 Direct 连接使用的网络接口或源 IP：

```yaml
# 全局配置
interface-name: en0
routing-mark: 1234

# 或直接指定源 IP
proxies:
  - name: "DIRECT"
    type: direct
```

这在多网卡环境中非常有用，例如同时有有线网络和蜂窝网络时，可以指定特定流量使用特定接口直连。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/direct.go` | Direct 适配器，实现 outbound.Plan 接口 |
| `adapter/outbound/base.go` | 基础适配器，Direct 的部分逻辑复用 |

Direct 适配器的核心方法是 `DialContext`，它接收 `context.Context`、`C.NetworkMetadata` 和 `Dialer` 参数，返回一个 `net.Conn` 或 `PacketConn`。实现逻辑大致如下：

1. 从 `NetworkMetadata` 中提取目标地址和协议类型
2. 根据协议类型选择 TCP Dialer 或 UDP Dialer
3. 应用接口绑定、SO_MARK 等配置
4. 执行连接并返回

## 性能特性

Direct 协议的性能特征：

- **延迟**：等于网络物理延迟（RTT），无额外处理开销
- **吞吐量**：受限于本地网络带宽，无协议 overhead
- **CPU 占用**：极低，仅涉及系统调用和内存拷贝
- **内存占用**：每个连接约数 KB（socket 缓冲区）

在代理规则引擎中，Direct 是性能最优的连接方式。对于不需要代理的流量，使用 Direct 可以避免不必要的代理开销。

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "DIRECT"
    type: direct

  - name: "COMPATIBLE"
    type: direct
```

### 结合规则使用

```yaml
rules:
  # 本地回环
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve

  # 国内网站
  - GEOSITE,cn,DIRECT

  # 特定域名
  - DOMAIN-SUFFIX,example.cn,DIRECT
  - DOMAIN,api.example.com,DIRECT

  # 特定 IP 段
  - IP-CIDR,223.0.0.0/8,DIRECT,no-resolve

  # 兜底
  - MATCH,PROXY
```

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Direct 实现 |
| Pipeline | 完全兼容 | L3 协议 |
| Resolver | 完全兼容 | DNS 解析 |
| TUN | 完全兼容 | 回环检测集成 |

## 相关文档

- [[reject]] - Reject 协议
- [[dns]] - DNS 代理
- [[ref/mihomo/proxy-groups|proxy-groups]] - 代理组配置
- [[ref/mihomo/rules|rules]] - 规则配置
