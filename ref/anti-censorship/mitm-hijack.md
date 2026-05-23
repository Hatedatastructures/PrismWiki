---
title: "中间人劫持防御"
category: "anti-censorship"
type: ref
module: ref
source: "概念文档"
tags: [流量对抗, mitm, 劫持, 安全, 证书, tls, 中间人]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# 中间人劫持防御

**类别**: 流量对抗

## 概述

中间人攻击（Man-in-the-Middle Attack，简称 MITM）是一种网络攻击方式，攻击者插入通信双方之间，拦截、篡改或伪造双方的通信内容。在审查对抗领域，中间人攻击是审查机构识别和封锁代理服务的重要手段，也是攻击者窃取用户信息的常用方法。

中间人攻击的威胁模型是：攻击者控制了通信路径上的某个节点（如路由器、代理服务器、DNS 服务器等），能够截获客户端和服务器之间的所有通信。攻击者可以读取通信内容（如果未加密）、修改通信内容、注入恶意数据、或者伪造服务器身份。这种攻击在代理场景中尤为危险，因为代理服务器本身就处于通信路径中间，可能被审查机构控制。

在审查场景中，中间人攻击通常表现为：TLS 拦截（使用伪造证书解密 HTTPS 流量）、TLS 降级（强制使用弱加密算法）、SNI 篡改（修改或阻止 TLS 握手中的域名信息）、主动探测伪装（冒充真实服务器探测客户端行为）。这些攻击手段可以识别代理流量、获取代理配置、或者直接封锁代理服务器。

TLS 拦截是最常见的中间人攻击形式。审查机构向用户浏览器安装根证书，然后使用该证书签发伪造的服务器证书，从而能够解密 HTTPS 流量。这种攻击在企业环境中常用于流量监控，在审查环境中用于识别代理协议。用户可能无意识地在浏览器中接受了伪造证书，导致通信被解密。

TLS 降级攻击利用协议协商的弱点，强制双方使用安全性较低的协议版本或加密算法。例如，攻击者可能阻止 TLS 1.3 的协商，强制使用 TLS 1.2，然后利用 TLS 1.2 的已知漏洞。或者攻击者可能移除某些密码套件选项，强制使用容易破解的算法。

SNI 篡改攻击针对 TLS 握手中的 Server Name Indication（服务器名称指示）字段。由于 SNI 是明文传输的，攻击者可以阻止、修改或替换 SNI 字段。如果 SNI 被阻止，TLS 握手可能失败；如果 SNI 被替换，客户端可能连接到错误的服务器。这种攻击可以用于封锁特定域名或进行钓鱼攻击。

现代代理协议通过多种机制防御中间人攻击：证书验证（验证服务器证书的真实性）、证书固定（预先配置信任的证书）、前向保密（确保历史通信不被后续密钥泄露影响）、密钥交换验证（验证密钥交换过程未被篡改）。Reality 协议通过使用真实网站的证书来绕过 TLS 拦截，ShadowTLS 通过密码认证来验证客户端身份。

Prism 在设计中充分考虑了中间人攻击威胁。通过 Reality 协议使用真实网站的证书、通过严格的证书验证机制、通过前向保密的密码套件选择，Prism 能够有效抵御中间人攻击。同时，Prism 的认证机制能够识别伪装的探测请求。

理解中间人攻击的原理和防御策略对于设计安全的代理系统至关重要。本文将详细介绍中间人攻击的技术原理、攻击类型、防御策略，以及在 Prism 中的实现。

## 原理详解

### 攻击原理

中间人攻击的基本原理是攻击者控制通信路径，能够截获、修改和伪造通信内容。攻击者需要在通信双方之间插入自己的节点，这可以通过多种方式实现。

```
中间人攻击位置:

正常通信:
客户端 ──────────────────────> 服务器
        直接通信，攻击者无法介入

中间人攻击:
客户端 ───> 攻击者 ───> 服务器
        攻击者位于通信路径中
        可以截获、修改、伪造数据

攻击者插入方式:
1. 网络层: ARP 欺骗、路由劫持、DNS 欺骗
2. 应用层: 代理服务器、浏览器扩展
3. 物理层: 网络设备控制、WiFi 劫持
```

**ARP 欺骗**

ARP 欺骗是在局域网中进行中间人攻击的常用方法：

```
ARP 欺骗流程:

正常 ARP:
客户端广播: "谁是 192.168.1.1?"
网关回应: "192.168.1.1 是 MAC-AA:BB:CC:DD:EE:FF"
客户端缓存: IP → MAC 映射

ARP 欺骗:
攻击者广播: "192.168.1.1 是 MAC-攻击者"
客户端缓存: 192.168.1.1 → 攻击者 MAC
攻击者广播: "192.168.1.100 是 MAC-攻击者"
网关缓存: 192.168.1.100 → 攻击者 MAC

结果:
客户端 ───> 攻击者 ───> 网关
所有流量经过攻击者
```

**DNS 欺骗**

DNS 欺骗通过篡改 DNS 响应来劫持通信：

```
DNS 欺骗流程:

正常 DNS:
客户端请求: "www.example.com 的 IP?"
DNS 服务器回应: "93.184.216.34"
客户端连接: 93.184.216.34

DNS 欺骗:
客户端请求: "www.example.com 的 IP?"
攻击者回应: "攻击者 IP"（伪造响应，更快到达）
客户端连接: 攻击者 IP（伪装成 example.com）

结果:
客户端连接到攻击者服务器
攻击者可以代理或篡改流量
```

**路由劫持**

路由劫持通过控制路由器或网关来劫持流量：

```
路由劫持方式:

1. BGP 劫持
   - 通告错误的 IP 范围
   - 流量被路由到攻击者网络

2. 路由器入侵
   - 入侵路由器设备
   - 配置流量转发规则

3. 网关控制
   - 控制网络网关
   - 部署流量监控设备

结果:
目标 IP 的流量被路由到攻击者
攻击者可以分析、修改、阻止流量
```

### TLS 中间人攻击

TLS 是防御中间人攻击的主要手段，但 TLS 本身也可能受到中间人攻击：

**证书伪造攻击**

证书伪造攻击使用伪造的证书来伪装服务器身份：

```
证书伪造攻击流程:

客户端                          攻击者                      真实服务器
    |                             |                             |
    |  TLS ClientHello            |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  攻击者生成伪造证书          |
    |                             |  (使用攻击者控制的 CA)       |
    |                             |                             |
    |  TLS ServerHello            |                             |
    |  + 伪造证书                  |                             |
    |<----------------------------|                             |
    |                             |                             |
    |  客户端验证证书              |                             |
    |  - 证书链有效？              |                             |
    |  - 如果客户端信任攻击者 CA    |                             |
    |    → 证书验证通过            |                             |
    |                             |                             |
    |  TLS 握手完成               |                             |
    |  (使用攻击者的密钥)          |                             |
    |                             |                             |
    |  加密数据                    |                             |
    |  (攻击者可以解密)            |                             |
    |<===========================>|                             |
    |                             |                             |
    |                             |  攻击者解密数据              |
    |                             |  可能修改后转发到真实服务器  |
    |                             |                             |
```

证书伪造的前提：
- 攻击者控制的 CA 证书被客户端信任
- 客户端不进行额外的证书验证
- 客户端没有证书固定（Certificate Pinning）

企业环境中的 TLS 拦截：
```
企业 TLS 拦截:

1. 企业向员工设备安装拦截代理的根证书
2. 拦截代理生成伪造证书，签发给每个 HTTPS 网站
3. 员工设备信任伪造证书（因为是企业 CA 签发）
4. 拦截代理可以解密所有 HTTPS 流量
5. 拦截代理检查流量，然后转发到真实网站

应用:
- 企业合规监控
- 数据泄露防护
- 恶意软件检测
```

审查环境中的 TLS 拦截：
```
审查 TLS 拦截:

场景 1: 强制安装证书
- 审查机构要求用户安装监控证书
- 通过浏览器或系统更新推送
- 所有 HTTPS 流量可被解密

场景 2: 浏览器劫持
- 审查机构控制浏览器厂商
- 预装信任审查 CA
- 特定网站的证书被伪造

场景 3: DNS 劫持 + 伪造
- DNS 欺骗指向审查服务器
- 审查服务器提供伪造证书
- 用户可能接受伪造证书
```

**TLS 降级攻击**

TLS 降级攻击强制使用安全性较低的协议版本：

```
TLS 降级攻击流程:

客户端                          攻击者                      服务器
    |                             |                             |
    |  TLS 1.3 ClientHello        |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  攻击者修改 ClientHello     |
    |                             |  - 移除 TLS 1.3 支持        |
    |                             |  - 只保留 TLS 1.2           |
    |                             |                             |
    |                             |  TLS 1.2 ClientHello        |
    |                             |---------------------------->|
    |                             |                             |
    |                             |                             |  TLS 1.2 ServerHello
    |                             |<----------------------------|
    |                             |                             |
    |  TLS 1.2 ServerHello        |                             |
    |<----------------------------|                             |
    |                             |                             |
    |  客户端被迫使用 TLS 1.2      |                             |
    |                             |                             |

TLS 1.2 的安全问题:
- 不强制前向保密
- 支持弱密码套件
- 存在已知攻击（如 POODLE, BEAST）
```

降级攻击类型：
```
协议版本降级:
- TLS 1.3 → TLS 1.2
- TLS 1.2 → TLS 1.1 或 1.0
- 强制使用不安全版本

密码套件降级:
- 强制使用弱加密算法（如 RC4, DES）
- 强制使用无前向保密的套件
- 移除强加密选项

椭圆曲线降级:
- 强制使用弱曲线
- 强制使用非 ECDHE 密钥交换
- 降低密钥交换安全性
```

**SNI 篡改攻击**

SNI 篡改攻击修改 TLS 握手中的服务器名称：

```
SNI 篡改攻击:

阻止 SNI:
客户端                          攻击者                      服务器
    |                             |                             |
    |  TLS ClientHello            |                             |
    |  SNI: blocked-site.com      |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  攻击者检测 SNI             |
    |                             |  SNI 在黑名单               |
    |                             |                             |
    |                             |  阻止连接                   |
    |                             |  (不转发 ClientHello)       |
    |                             |                             |
    |  连接超时或重置              |                             |
    |<----------------------------|                             |

替换 SNI:
客户端                          攻击者                      服务器
    |                             |                             |
    |  TLS ClientHello            |                             |
    |  SNI: blocked-site.com      |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  攻击者修改 SNI             |
    |                             |  SNI: allowed-site.com      |
    |                             |                             |
    |                             |  TLS ClientHello            |
    |                             |---------------------------->|
    |                             |                             |
    |                             |                             |  allowed-site.com 的证书
    |                             |<----------------------------|
    |                             |                             |
    |  证书验证失败                |                             |
    |  (证书主题与期望 SNI 不匹配)  |                             |
    |<----------------------------|                             |
```

### 针对代理的中间人攻击

代理服务本身处于通信路径中间，可能成为中间人攻击的目标：

**代理服务器探测**

审查机构通过中间人攻击探测代理服务器：

```
代理探测流程:

审查机构                        代理服务器                    目标服务器
    |                             |                             |
    |  TLS 连接请求                |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  代理检测客户端              |
    |                             |  验证身份                    |
    |                             |                             |
    |                             |  如果验证失败                |
    |                             |  → 可能返回代理错误          |
    |                             |  或者回退到真实服务器        |
    |                             |                             |
    |  代理错误响应                |                             |
    |<----------------------------|                             |
    |                             |                             |
    |  审查机构分析响应            |                             |
    |  → 确认代理服务器            |                             |
```

**代理流量解密**

如果代理使用 TLS 但证书被伪造：

```
代理流量解密:

1. 审查机构获取代理服务器证书
2. 生成伪造证书（使用受信任的 CA）
3. 客户端连接代理时收到伪造证书
4. 客户端可能接受伪造证书
5. 审查机构可以解密代理流量
6. 识别代理协议和目标

风险因素:
- 客户端不验证证书
- 客户端信任伪造 CA
- 代理无证书固定
```

### 防御机制

对抗中间人攻击的防御机制：

**证书验证**

严格的证书验证是防御中间人攻击的基础：

```
证书验证流程:

客户端收到证书后验证:

1. 验证证书链
   ┌─────────────────────────────────────────────────────────────┐
   │ 服务器证书                                                   │
   │   ↓ 签发者                                                   │
   │ 中间 CA 证书                                                 │
   │   ↓ 签发者                                                   │
   │ 根 CA 证书                                                   │
   │   ↓                                                          │
   │ 系统信任存储                                                  │
   └─────────────────────────────────────────────────────────────┘
   - 每一级证书签名有效
   - 最终到达信任的根 CA

2. 验证证书属性
   - 有效期内（not expired）
   - 主题匹配（subject matches hostname）
   - 用途正确（key usage, extended key usage）
   - 未吊销（not revoked）

3. 验证证书指纹
   - 计算证书指纹（SHA256）
   - 与预存指纹比对
   - 完全匹配才信任

验证失败处理:
- 显示警告（浏览器）
- 拒绝连接（严格模式）
- 提示用户选择（用户决定）
```

**证书固定**

证书固定（Certificate Pinning）预先配置信任的证书：

```
证书固定类型:

1. 公钥固定
   - 固定服务器公钥
   - 证书更换时公钥不变
   - 更灵活

2. 证书固定
   - 固定完整证书
   - 证书更换需要更新固定
   - 更严格

3. CA 固定
   - 固定签发 CA
   - 只信任特定 CA 签发的证书
   - 中等严格度

实现方式:

浏览器固定（HPKP，已废弃）:
Public-Key-Pins: pin-sha256="base64-hash";
                 max-age=seconds; includeSubDomains

应用固定（代码中）:
const pinned_certs = [
    "sha256/ABCDEF...",
    "sha256/123456...",
];
verify_cert(cert, pinned_certs);

配置固定（用户配置）:
{
    "server": {
        "pinned_certificate": "sha256/ABCDEF..."
    }
}
```

**前向保密**

前向保密（Forward Secrecy）确保历史通信不被后续密钥泄露影响：

```
前向保密原理:

无前向保密（RSA 密钥交换）:
服务器长期私钥泄露 → 所有历史通信可解密
威胁: 服务器被入侵后历史数据泄露

有前向保密（ECDHE 密钥交换）:
┌────────────────────────────────────────────────────────────────┐
│ 每次连接生成临时密钥对                                          │
│   ↓                                                            │
│ 使用临时密钥进行密钥交换                                        │
│   ↓                                                            │
│ 连接结束后临时密钥丢弃                                          │
│   ↓                                                            │
│ 即使长期私钥泄露，历史通信仍然安全                               │
└────────────────────────────────────────────────────────────────┘

实现:
- 使用 ECDHE 密码套件
- TLS 1.3 强制前向保密
- TLS 1.2 需要选择支持前向保密的套件

推荐密码套件:
- TLS_AES_128_GCM_SHA256 (TLS 1.3)
- TLS_AES_256_GCM_SHA384 (TLS 1.3)
- TLS_CHACHA20_POLY1305_SHA256 (TLS 1.3)
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (TLS 1.2)
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 (TLS 1.2)
```

**密钥交换验证**

验证密钥交换过程未被篡改：

```
密钥交换验证:

TLS 1.2 (ECDHE):
1. 服务器发送 ECDHE 参数 + 签名
2. 客户端验证签名（使用服务器证书公钥）
3. 如果签名无效 → 密钥交换被篡改 → 拒绝连接

TLS 1.3:
1. 服务器发送 Key Share + 签名
2. 签名覆盖整个握手消息
3. 客户端验证签名
4. 如果签名无效 → 握手被篡改 → 拒绝连接

Reality 的密钥交换验证:
1. 使用真实网站的证书
2. 真实网站签名保证握手完整性
3. 攻击者无法伪造签名
4. 完全防御 TLS 中间人
```

### 协议特定的防御

**Reality 协议防御**

Reality 通过使用真实网站的证书来防御中间人攻击：

```
Reality 中间人防御:

核心原理:
┌────────────────────────────────────────────────────────────────┐
│ Reality 服务器不自己签发证书                                    │
│   ↓                                                            │
│ Reality 使用真实网站的证书                                      │
│   ↓                                                            │
│ 攻击者无法伪造真实网站的证书                                    │
│   ↓                                                            │
│ 完全防御 TLS 中间人攻击                                         │
└────────────────────────────────────────────────────────────────┘

工作流程:
客户端                          Reality服务器                 真实网站
    |                             |                             |
    |  ClientHello + 身份标记     |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  验证身份标记                |
    |                             |  → 通过                     |
    |                             |                             |
    |                             |  建立到真实网站的 TLS        |
    |                             |---------------------------->|
    |                             |                             |
    |                             |                             |  ServerHello
    |                             |                             |  + 真实证书
    |                             |<----------------------------|
    |                             |                             |
    |                             |  使用真实证书响应客户端      |
    |  ServerHello                |                             |
    |  + 真实网站证书              |                             |
    |<----------------------------|                             |
    |                             |                             |
    |  客户端验证证书              |                             |
    |  → 证书来自真实 CA           |                             |
    |  → 证书主题匹配 SNI          |                             |
    |  → 验证通过                  |                             |
    |                             |                             |
    |  建立安全隧道                |                             |
    |<===========================>|                             |

防御效果:
- 攻击者无法伪造真实网站证书
- 攻击者无法进行 TLS 拦截
- 攻击者无法降级 TLS（真实网站使用 TLS 1.3）
- 完全防御中间人攻击
```

**ShadowTLS 协议防御**

ShadowTLS 通过密码认证来防御中间人攻击：

```
ShadowTLS 中间人防御:

核心原理:
┌────────────────────────────────────────────────────────────────┐
│ ShadowTLS 使用真实 TLS 握手                                    │
│   ↓                                                            │
│ 握手后进行密码认证                                             │
│   ↓                                                            │
│ 无密码的连接被回退                                             │
│   ↓                                                            │
│ 中间人无法获得密码                                             │
└────────────────────────────────────────────────────────────────┘

ShadowTLS v3:
客户端                          ShadowTLS服务器               真实网站
    |                             |                             |
    |  TLS ClientHello            |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  代理到真实网站              |
    |                             |---------------------------->|
    |                             |                             |
    |                             |                             |  ServerHello
    |                             |<----------------------------|
    |                             |                             |
    |  ServerHello + 真实证书      |                             |
    |<----------------------------|                             |
    |                             |                             |
    |  TLS 握手完成               |                             |
    |                             |                             |
    |  发送密码认证                |                             |
    |---------------------------->|                             |
    |                             |                             |
    |                             |  验证密码                   |
    |                             |  → 通过                     |
    |                             |                             |
    |                             |  开始代理服务               |
    |<===========================>|                             |

防御效果:
- TLS 使用真实证书，防御 TLS 拦截
- 密码认证防御探测
- 攻击者无法冒充客户端（无密码）
```

## 在 Prism 中的应用

Prism 在多个层面实现了中间人攻击防御机制：

### Reality 证书获取

Reality 方案通过获取真实网站证书来防御中间人攻击：

```cpp
// src/prism/stealth/reality/handshake.cpp

auto reality_handshake::fetch_dest_certificate(
    const std::string& dest_host,
    uint16_t dest_port
) -> awaitable<certificate_info> {
    // 建立到真实网站的 TLS 连接
    ssl_socket socket;
    co_await socket.async_connect(dest_host, dest_port);

    // 执行 TLS 握手
    co_await socket.async_handshake(ssl::handshake_type::client);

    // 获取真实网站的证书
    auto cert = socket.get_peer_certificate();

    // 验证证书真实性
    auto verified = verify_certificate_chain(cert);
    if (!verified) {
        throw security_exception("Certificate verification failed");
    }

    // 提取证书信息
    certificate_info info;
    info.certificate = cert;
    info.fingerprint = compute_cert_fingerprint(cert);
    info.subject = get_certificate_subject(cert);
    info.issuer = get_certificate_issuer(cert);
    info.expiry = get_certificate_expiry(cert);

    co_return info;
}
```

### 证书验证

Prism 实现严格的证书验证：

```cpp
// include/prism/stealth/cert_verifier.hpp

/// @brief 证书验证器
class certificate_verifier {
public:
    /// @brief 验证证书链
    /// @param cert 目标证书
    /// @param chain 证书链
    /// @return 验证结果
    auto verify_chain(const certificate& cert, const cert_chain& chain) -> verify_result;

    /// @brief 验证证书固定
    /// @param cert 目标证书
    /// @param pinned_fingerprint 固定的指纹
    /// @return 是否匹配
    auto verify_pinning(const certificate& cert, const std::string& pinned_fingerprint) -> bool;

    /// @brief 检查证书吊销状态
    /// @param cert 目标证书
    /// @return 是否已吊销
    auto check_revocation(const certificate& cert) -> revoke_status;

private:
    /// 验证证书有效期
    auto verify_validity(const certificate& cert) -> bool;

    /// 验证证书用途
    auto verify_key_usage(const certificate& cert) -> bool;

    /// 验证主题名称匹配
    auto verify_subject_match(const certificate& cert, const std::string& hostname) -> bool;

    /// 验证证书签名
    auto verify_signature(const certificate& cert, const certificate& issuer) -> bool;
};
```

### 密码套件选择

Prism 强制使用前向保密密码套件：

```cpp
// src/prism/agent/tls.cpp

auto configure_tls_context(ssl::context& ctx) -> void {
    // TLS 1.3 默认启用前向保密
    ctx.set_options(ssl::context::default_workarounds |
                    ssl::context::no_sslv2 |
                    ssl::context::no_sslv3 |
                    ssl::context::no_tlsv1 |
                    ssl::context::no_tlsv1_1);

    // TLS 1.2 只使用前向保密套件
    ctx.set_cipher_list(
        "ECDHE-ECDSA-AES128-GCM-SHA256:"
        "ECDHE-RSA-AES128-GCM-SHA256:"
        "ECDHE-ECDSA-AES256-GCM-SHA384:"
        "ECDHE-RSA-AES256-GCM-SHA384:"
        "ECDHE-ECDSA-CHACHA20-POLY1305:"
        "ECDHE-RSA-CHACHA20-POLY1305"
    );

    // 验证模式：要求证书验证
    ctx.set_verify_mode(ssl::verify_peer | ssl::verify_fail_if_no_peer_cert);

    // 设置验证回调
    ctx.set_verify_callback([](bool preverified, ssl::verify_context& ctx) {
        // 额外验证逻辑
        return verify_certificate_extra(preverified, ctx);
    });
}
```

### 身份验证

Prism 实现客户端身份验证来防御中间人伪装：

```cpp
// src/prism/stealth/reality/auth.cpp

auto reality_authenticator::verify_client_identity(
    const client_hello_features& features,
    const reality_config& config
) -> auth_result {
    // 1. 提取 short_id
    auto short_id = extract_short_id(features);
    if (!short_id) {
        // 无身份标记，可能是探测者或中间人
        return auth_result::fallback;
    }

    // 2. 验证 short_id 是否在配置中
    if (!config.short_ids.contains(*short_id)) {
        // 无效 short_id，可能是中间人伪造
        return auth_result::rejected;
    }

    // 3. 验证时间戳（防重放）
    auto timestamp = extract_timestamp(features);
    if (!verify_timestamp(timestamp)) {
        // 时间戳无效，可能是重放攻击
        return auth_result::rejected;
    }

    // 4. 验证认证哈希
    auto auth_hash = compute_auth_hash(features, config);
    auto expected_hash = get_expected_hash(config);
    if (auth_hash != expected_hash) {
        // 认证哈希不匹配，可能是中间人篡改
        return auth_result::rejected;
    }

    return auth_result::accepted;
}
```

### 配置示例

Reality 配置：

```json
{
  "stealth": {
    "reality": {
      "dest": "www.google.com:443",
      "server_names": ["www.google.com", "google.com"],
      "private_key": "GENERATED_PRIVATE_KEY",
      "short_ids": ["0123456789abcdef", "fedcba9876543210"],
      "certificate_verification": {
        "verify_chain": true,
        "verify_expiry": true,
        "verify_revocation": true,
        "min_tls_version": "tls1.3"
      }
    }
  }
}
```

TLS 配置：

```json
{
  "tls": {
    "min_version": "tls1.2",
    "preferred_version": "tls1.3",
    "cipher_suites": [
      "TLS_AES_128_GCM_SHA256",
      "TLS_AES_256_GCM_SHA384",
      "TLS_CHACHA20_POLY1305_SHA256"
    ],
    "ecdhe_curves": ["x25519", "secp256r1", "secp384r1"],
    "verify_peer": true,
    "forward_secrecy_required": true
  }
}
```

### 证书固定配置

```json
{
  "certificate_pinning": {
    "enabled": true,
    "mode": "public_key",
    "pins": [
      {
        "hostname": "proxy.example.com",
        "pin": "sha256/ABCDEF1234567890...",
        "enforce": true
      }
    ],
    "backup_pins": [
      "sha256/FEDCBA0987654321..."
    ]
  }
}
```

## 最佳实践

### 证书管理

**Reality 目标选择**

选择 Reality 目标网站的建议：

```
目标网站选择标准:
- 知名度高（流量大，不易引起怀疑）
- 使用 TLS 1.3（强制前向保密）
- 证书由知名 CA 签发（用户信任）
- 服务器稳定（连接可靠）
- 支持 HTTP/2（流量特征正常）

推荐目标:
- www.google.com
- www.microsoft.com
- www.cloudflare.com
- www.apple.com

避免目标:
- 小型网站（流量特征明显）
- 自签名证书网站（可能被警告）
- 已知的敏感网站（可能被特殊处理）
```

**证书固定策略**

```
证书固定部署:

开发阶段:
- 使用 CA 固定（允许更换证书）
- 配置备份 pin（防止锁死）

生产阶段:
- 使用公钥固定（更灵活）
- 设置合理的过期时间
- 监控证书更换

紧急情况:
- 如果固定失败，检查备份 pin
- 如果备份失败，允许临时跳过
- 记录所有固定失败事件
```

### 密码套件选择

**推荐配置**

```
TLS 1.3 优先:
- 所有 TLS 1.3 套件都有前向保密
- 所有 TLS 1.3 套件都安全
- 优先使用 TLS 1.3

TLS 1.2 后备:
- 只使用 ECDHE 套件（前向保密）
- 避免 RSA 密钥交换套件
- 避免弱算法（RC4, DES, 3DES）

推荐顺序:
1. TLS_AES_128_GCM_SHA256 (TLS 1.3)
2. TLS_AES_256_GCM_SHA384 (TLS 1.3)
3. TLS_CHACHA20_POLY1305_SHA256 (TLS 1.3)
4. ECDHE-ECDSA-AES128-GCM-SHA256
5. ECDHE-RSA-AES128-GCM-SHA256
```

### 监控与响应

**证书监控**

```
证书监控内容:
- 证书有效期（提前预警）
- 证书吊销状态（定期检查）
- 证书指纹变化（检测篡改）
- TLS 版本分布（检测降级）

告警条件:
- 证书即将过期（< 30 天）
- 证书被吊销
- 证书指纹不匹配
- TLS 降级尝试
```

**中间人检测**

```
中间人检测方法:
- 证书指纹与预期不符
- TLS 版本被降级
- 握手过程异常
- 响应时序异常
- 数据篡改迹象

响应措施:
- 记录事件
- 拒绝连接
- 通知管理员
- 启用备用配置
```

## 常见问题

### Q1: Reality 能完全防御中间人攻击吗？

Reality 能防御 TLS 层面的中间人攻击，但不能防御：
- 网络层的流量截获（仍可分析流量模式）
- 应用层的攻击（如钓鱼网站）
- DNS 层的劫持（可能指向错误服务器）
- 客户端层面的攻击（如恶意软件）

Reality 主要防御 TLS 拦截和证书伪造。

### Q2: 如何检测自己是否被 TLS 拦截？

检测方法：
- 检查证书颁发者是否异常
- 比对证书指纹与预期
- 检查 TLS 版本是否被降级
- 使用证书固定工具

企业环境中的 TLS 拦截通常会显示企业 CA 签发的证书。

### Q3: 证书固定有什么风险？

证书固定的风险：
- 锁死风险：证书更换后无法连接
- 维护成本：需要更新固定配置
- 部署复杂：需要客户端支持
- 备份失效：备份 pin 也失效时完全无法连接

需要合理配置备份 pin 和过期时间。

### Q4: TLS 1.3 比 TLS 1.2 更安全吗？

是的，TLS 1.3 有多项改进：
- 强制前向保密
- 加密更多握手内容
- 移除不安全算法
- 更快的握手速度
- 简化的协议设计

建议优先使用 TLS 1.3。

### Q5: 如何应对企业 TLS 拦截？

应对策略：
- 使用 Reality 方案（绕过企业拦截）
- 使用证书固定（拒绝企业证书）
- 使用应用层加密（在 TLS 内加密）
- 使用代理隧道（绕过拦截点）

Reality 是最有效的方案。

### Q6: ECDHE 和 RSA 密钥交换有什么区别？

主要区别：
- ECDHE：临时密钥，每次连接新密钥，有前向保密
- RSA：长期密钥，服务器私钥泄露可解密历史通信

建议使用 ECDHE 或 DHE。

### Q7: 如何配置 Prism 的证书验证？

Prism 配置示例：
```json
{
  "tls": {
    "verify_peer": true,
    "verify_fail_if_no_peer_cert": true,
    "min_version": "tls1.3",
    "forward_secrecy_required": true
  },
  "stealth": {
    "reality": {
      "certificate_verification": {
        "verify_chain": true,
        "verify_expiry": true
      }
    }
  }
}
```

## 参考资料

- [Man-in-the-middle attack - Wikipedia](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)
- [TLS 1.3 RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)
- [Certificate Pinning](https://owasp.org/www-community-guide/Certificate_and_Public_Key_Pinning)
- [Forward Secrecy](https://en.wikipedia.org/wiki/Forward_secrecy)
- [Reality Protocol](https://github.com/XTLS/REALITY)
- [SSL/TLS Attack Overview](https://www.ssllabs.com/projects/tls-ssl-attacks/)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/probing|主动探测]] — 主动探测防御
- [[ref/anti-censorship/replay-attack|重放攻击]] — 重放攻击防御
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹识别
- [[core/stealth/reality|Reality 协议]] — Reality 实现细节
- [[core/crypto/hkdf|证书验证]] — 证书验证实现