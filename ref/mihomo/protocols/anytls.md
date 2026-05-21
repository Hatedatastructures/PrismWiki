---
title: AnyTLS
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, anytls]
---
# AnyTLS 协议

AnyTLS 是一种隐蔽性 TLS 代理协议，通过维持空闲 TLS 会话连接池、流量 Padding 和时间特征混淆等技术，使代理流量在统计特征上与正常 HTTPS 流量难以区分。

## 协议概述

AnyTLS 的核心设计理念是"流量特征伪装"——不仅仅加密数据内容，更关注流量的统计特征（包大小分布、时间间隔、连接行为等），使其与正常的 HTTPS 浏览行为一致。其特性包括：

- **TLS 隐蔽性代理**：基于标准 TLS 传输，外观与普通 HTTPS 无异
- **空闲会话连接池**：维持预建立的 TLS 连接池，减少握手特征
- **UDP over TCP 隧道**：通过 TLS 隧道传输 UDP 数据
- **客户端指纹**：支持多种浏览器 TLS 指纹（JA3/JA4）
- **流量 Padding**：对数据包大小进行填充，消除特征大小模式
- **时间特征混淆**：调整数据包发送间隔，模拟人类浏览行为
- **ECH 集成**：支持 Encrypted Client Hello 防止 SNI 泄露

AnyTLS 与 Reality 和 ECH 形成 TLS 隐蔽技术栈的三个层级：
1. **Reality**：借用真实网站的 TLS 握手
2. **ECH**：加密 Client Hello 中的 SNI
3. **AnyTLS**：流量统计特征伪装

## Padding 技术

### 为什么需要 Padding

在代理通信中，数据包大小分布是一个重要的流量指纹特征：

```
正常 HTTPS 流量:
包大小分布: [200, 500, 1500, 300, 800, 1200, ...]  (连续分布)

传统代理流量:
包大小分布: [64, 128, 256, 64, 128, ...]           (离散分布，有特征值)
```

GFW 等防火墙可以通过分析包大小分布来识别代理流量。AnyTLS 的 Padding 技术通过填充数据包到随机大小来解决这个问题。

### Padding 机制

```
原始数据包                          填充后
+-------------------+              +-------------------------------+
| Header | Payload  |     →        | Header | Payload | Padding    |
| 12 B   | 64 B     |              | 12 B   | 64 B    | 128 B 随机  |
+-------------------+              +-------------------------------+
总大小: 76 B                       总大小: 204 B (随机填充)
```

Padding 策略：
1. **最小填充**：确保每个包至少达到最小大小
2. **最大填充**：包大小不超过最大限制
3. **随机填充**：在范围内随机选择填充大小，消除固定模式

### Padding 参数

AnyTLS 支持配置 Padding 范围：

```yaml
proxies:
  - name: "anytls-padded"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    padding-min: 100
    padding-max: 1400
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `padding-min` | 100 | 最小包大小（字节） |
| `padding-max` | 1400 | 最大包大小（字节，接近 MTU） |

合理的 Padding 范围：
- **太小**（如 100-200）：填充效果有限，仍可识别
- **适中**（如 100-1400）：良好的隐蔽性和带宽效率平衡
- **太大**（如 100-4000）：浪费带宽，增加延迟

## 空闲会话连接池

### 原理

AnyTLS 维护一组预建立的 TLS 连接（空闲会话），当需要代理数据时直接使用已有连接，而不是临时建立新连接。这消除了"按需建立 TLS 连接"的特征模式。

```
连接池管理:
┌──────────────────────────────────┐
│  空闲连接池                       │
│  ┌─────┐ ┌─────┐ ┌─────┐        │
│  │TLS-1│ │TLS-2│ │TLS-3│ ...    │
│  │空闲  │ │空闲  │ │空闲  │        │
│  └─────┘ └─────┘ └─────┘        │
└──────────────────────────────────┘
      ↑           ↑           ↑
   使用时取出    使用后放回   定期健康检查
```

### 连接池参数

```yaml
proxies:
  - name: "anytls-pool"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    idle-session-check-interval: 30    # 空闲检查间隔（秒）
    idle-session-timeout: 60           # 空闲超时时间（秒）
    min-idle-session: 1                # 最小空闲会话数
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `idle-session-check-interval` | 30 | 检查空闲连接的间隔（秒） |
| `idle-session-timeout` | 60 | 空闲连接超时时间（秒），超过后关闭 |
| `min-idle-session` | 1 | 保持的最小空闲连接数 |

### 连接池生命周期

```
1. 初始化: 建立 min-idle-session 个 TLS 连接
2. 使用时: 从池中取出一个空闲连接用于数据传输
3. 使用后: 将连接放回池中
4. 检查: 每隔 idle-session-check-interval 秒检查池中连接
5. 清理: 关闭超过 idle-session-timeout 的空闲连接
6. 补充: 如果池中连接数 < min-idle-session，建立新连接
```

这种机制的优势：
- **减少握手特征**：不需要每次请求都进行 TLS 握手
- **降低延迟**：直接使用已有连接，省去握手 RTT
- **连接复用**：多条代理流共享 TLS 连接
- **降低检测率**：连接行为更像浏览器的连接复用

## 流量特征伪装

### 时间特征混淆

人类浏览行为的网络流量具有特定的时间模式：

```
人类浏览模式:
[请求] --2.3s--> [请求] --0.5s--> [请求] --5.1s--> [请求] --1.2s-->

传统代理模式:
[请求] --0.01s--> [请求] --0.01s--> [请求] --0.01s--> (过于规律)
```

AnyTLS 通过以下方式模拟人类浏览模式：
1. **延迟随机化**：在数据包之间添加随机的微小延迟
2. **突发/空闲模式**：模拟人类的"突发浏览-阅读-再浏览"模式
3. **心跳模拟**：在空闲连接上发送模拟的心跳数据

### 包大小分布伪装

AnyTLS 确保输出流量的包大小分布与正常 HTTPS 流量一致：

| 统计量 | 正常 HTTPS | AnyTLS 目标 |
|--------|-----------|------------|
| 平均包大小 | ~500 字节 | 接近 |
| 中位数包大小 | ~300 字节 | 接近 |
| 最大包大小 | ~1460 字节 (MSS) | 接近 |
| 小包比例 | ~20% | 接近 |

### 与正常 TLS 流量的对比

| 特征 | 正常 HTTPS | 传统代理 | AnyTLS |
|------|-----------|----------|--------|
| TLS 握手 | 标准 | 标准 | 标准（+ 指纹匹配） |
| 包大小分布 | 连续随机 | 离散固定 | 连续随机（Padding） |
| 时间间隔 | 不规则 | 规律 | 不规则（随机延迟） |
| 连接行为 | 复用 + 新建 | 按需新建 | 连接池复用 |
| SNI | 域名 | 代理服务器 | 域名（+ ECH） |

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/anytls.go` | AnyTLS 适配器，配置解析和连接管理 |
| `transport/anytls/client.go` | AnyTLS 客户端实现，TLS 连接管理和 Padding |
| `transport/anytls/session/` | 会话管理，连接池和空闲会话处理 |

### 连接池实现

AnyTLS 的会话管理模块负责：

1. **连接创建**：使用配置的 TLS 参数建立连接
2. **连接复用**：从池中取出空闲连接
3. **健康检查**：定期检测连接可用性
4. **超时清理**：关闭过期连接
5. **容量管理**：维持最小空闲连接数

## ECH 集成

AnyTLS 支持 ECH（Encrypted Client Hello）以防止 SNI 泄露：

```yaml
proxies:
  - name: "anytls-ech"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    ech-opts:
      enable: true
      config: base64-encoded-ech-config
```

ECH 与 AnyTLS 的结合提供了双层保护：
1. **ECH** 加密 Client Hello 中的 SNI
2. **AnyTLS** 伪装流量统计特征

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "anytls-proxy"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
```

### 完整配置

```yaml
proxies:
  - name: "anytls-full"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    sni: server.example.com
    skip-cert-verify: false
    fingerprint: chrome
    client-fingerprint: chrome
    alpn:
      - h2
      - http/1.1
    idle-session-check-interval: 30
    idle-session-timeout: 60
    min-idle-session: 1
    udp: true
```

### ECH 配置

```yaml
proxies:
  - name: "anytls-ech"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    ech-opts:
      enable: true
      config: base64-encoded-ech-config
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `anytls` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `password` | string | 是 | 认证密码 |
| `sni` | string | 否 | TLS SNI |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `fingerprint` | string | 否 | TLS 证书指纹 |
| `client-fingerprint` | string | 否 | TLS Client Hello 指纹 |
| `alpn` | []string | 否 | ALPN 协议列表 |
| `idle-session-check-interval` | int | 否 | 空闲检查间隔（秒） |
| `idle-session-timeout` | int | 否 | 空闲超时时间（秒） |
| `min-idle-session` | int | 否 | 最小空闲会话数 |
| `udp` | bool | 否 | 启用 UDP over TCP |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | AnyTLS 协议实现 |
| Pipeline | 支持 | TLS 流 |
| Stealth | 完全兼容 | TLS 隐蔽技术 |
| Multiplex | 支持 | TLS 连接复用 |

## 相关文档

- [[reality]] - Reality TLS 隐蔽
- [[ech]] - ECH 加密 Client Hello
- [[core/stealth/anytls|anytls]] - AnyTLS 隐蔽技术
- [[ref/mihomo/protocols/overview|protocols]] - 代理协议概述
