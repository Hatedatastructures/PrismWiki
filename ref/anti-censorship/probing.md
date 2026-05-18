---
title: "主动探测防御"
category: "anti-censorship"
type: ref
module: ref
source: "概念文档"
tags: [流量对抗, 主动探测, 检测, 防御, 审查, 探测防御]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# 主动探测防御

**类别**: 流量对抗

## 概述

主动探测（Active Probing）是审查机构主动连接可疑服务器，通过分析服务器响应来判断是否运行代理服务的技术。与被动深度包检测（DPI）不同，主动探测是一种主动出击的检测方式，能够识别那些成功规避了被动检测的代理服务器。

主动探测技术在 2010 年左右开始被大规模部署，特别是在中国防火长城（GFW）中得到了广泛应用。研究表明，GFW 每天发送数百万次主动探测请求，覆盖各种协议和端口。主动探测能够有效识别 Shadowsocks、V2Ray、Trojan 等代理服务，是代理对抗中最具挑战性的检测方式之一。

主动探测的工作原理是：审查机构的探测系统向可疑服务器发送特定协议的请求，如果服务器的响应符合代理协议的特征，则判定该服务器为代理服务器并加以封锁。探测请求可以模仿真实客户端的行为，包括完整的 TLS 握手、HTTP 请求、SOCKS5 连接等。

主动探测的优势在于：能够检测加密流量、能够识别协议混淆、能够验证 DPI 的检测结果。即使代理流量完全加密，主动探测仍然可以通过尝试连接和分析响应来识别代理服务。这使得主动探测成为 DPI 的重要补充。

主动探测的局限性在于：需要消耗大量资源、可能被识别和规避、存在误判风险。探测系统需要维护大量的探测节点和探测请求队列，这增加了运营成本。同时，服务器可以通过各种手段识别探测请求并做出不同响应。

Prism 通过多层防御机制对抗主动探测：身份验证（只响应授权客户端）、响应伪装（对未授权请求返回正常网站响应）、探测检测（识别探测请求的特征）等。这些机制组合使用，能够有效抵御大多数主动探测攻击。

理解主动探测的工作原理和防御策略对于设计安全的代理系统至关重要。本文将详细介绍主动探测的技术原理、检测方法、规避策略，以及在 Prism 中的实现。

## 原理详解

### 探测类型

主动探测可以分为多种类型，每种类型针对不同的代理协议和特征：

**协议探测**

协议探测是最常见的探测类型，审查系统向目标服务器发送特定协议的请求，分析服务器的响应是否符合代理协议规范。

```
协议探测示例:

探测系统                              目标服务器
    |                                    |
    |  SOCKS5 握手请求                   |
    |------------------------------------>|
    |                                    |
    |  SOCKS5 响应 (0x05 0x00)           |
    |<------------------------------------|
    |                                    |
    |  确认: 这是一个 SOCKS5 服务器        |
    |                                    |

判断依据:
- SOCKS5 服务器返回 0x05 0x00
- HTTP 代理返回 HTTP 响应
- Shadowsocks 返回加密响应
```

**行为探测**

行为探测分析服务器对不同请求的响应行为，识别异常模式。

```
行为探测示例:

探测请求序列:
1. 发送有效 TLS ClientHello
   - 正常 HTTPS: 返回 ServerHello + 证书
   - 代理服务器: 可能返回非标准响应

2. 发送无效 HTTP 请求
   - 正常 HTTP 服务器: 返回 400/50x 错误
   - 代理服务器: 可能返回连接错误或无响应

3. 发送完整 HTTP 请求
   - 正常 HTTP 服务器: 返回网页内容
   - 代理服务器: 返回代理错误或转发请求

行为分析:
- 响应时序是否正常
- 错误处理是否标准
- 响应内容是否一致
```

**指纹探测**

指纹探测使用已知的代理软件指纹来识别服务器。

```
指纹探测示例:

已知代理指纹库:
- Shadowsocks:
  * 响应特征: 加密的固定长度头部
  * 时序特征: 特定的响应延迟

- V2Ray VMess:
  * 响应特征: 特定的头部结构
  * 命令特征: 特定的指令格式

- Trojan:
  * 响应特征: TLS 内部的特定格式
  * 认证特征: 密码验证的特定行为

探测过程:
1. 发送已知代理协议的请求
2. 比对响应与指纹库
3. 匹配则确认为代理服务器
```

**重放探测**

重放探测截获真实客户端的请求并重放，检测服务器是否按代理协议处理。

```
重放探测示例:

客户端                              探测系统                    目标服务器
    |                                   |                           |
    |  1. 发送代理请求                   |                           |
    |----------------------------------->|                           |
    |                                   |                           |
    |                                   |  2. 重放请求              |
    |                                   |-------------------------->|
    |                                   |                           |
    |                                   |  3. 分析响应              |
    |                                   |<--------------------------|
    |                                   |                           |
    |                                   |  如果响应符合代理协议       |
    |                                   |  → 确认为代理服务器         |
```

### 探测技术

**TLS 探测**

TLS 探测分析服务器的 TLS 实现和证书信息。

```
TLS 探测流程:

1. 发送 TLS ClientHello
   ┌─────────────────────────────────────┐
   │ TLS ClientHello                     │
   │ - SNI: target.domain.com            │
   │ - Version: TLS 1.3                  │
   │ - Cipher Suites: [...]              │
   └─────────────────────────────────────┘

2. 分析 ServerHello
   ┌─────────────────────────────────────┐
   │ TLS ServerHello                     │
   │ - Version: TLS 1.3                  │
   │ - Cipher Suite: TLS_AES_128_GCM...  │
   │ - Extensions: [...]                 │
   └─────────────────────────────────────┘

3. 分析证书
   ┌─────────────────────────────────────┐
   │ Certificate                         │
   │ - Issuer: Let's Encrypt             │
   │ - Subject: target.domain.com        │
   │ - Validity: 2024-01-01 to 2025-01-01│
   │ - SAN: [...]                        │
   └─────────────────────────────────────┘

判断依据:
- 自签名证书 → 可疑代理
- 证书与 SNI 不匹配 → 可疑代理
- 非标准 TLS 实现 → 可疑代理
```

**HTTP 探测**

HTTP 探测发送各种 HTTP 请求，分析服务器的响应。

```
HTTP 探测类型:

1. 基本请求探测
   GET / HTTP/1.1
   Host: target.domain.com

   正常响应: 200 OK + HTML 内容
   可疑响应: 无响应、连接关闭、非 HTTP 响应

2. 代理请求探测
   GET http://example.com/ HTTP/1.1
   Host: example.com

   正常响应: 400 Bad Request (非代理服务器)
   可疑响应: 200 OK (代理服务器转发请求)

3. CONNECT 探测
   CONNECT example.com:443 HTTP/1.1
   Host: example.com

   正常响应: 405 Method Not Allowed
   可疑响应: 200 Connection Established

4. 路径探测
   GET /shadowsocks HTTP/1.1
   GET /v2ray HTTP/1.1
   GET /trojan HTTP/1.1

   正常响应: 404 Not Found
   可疑响应: 特定响应 (代理协议入口)
```

**端口扫描探测**

端口扫描探测检测常见代理端口的开放情况。

```
常见代理端口:
- 1080: SOCKS5 默认端口
- 10808: V2Ray HTTP 默认端口
- 10809: V2Ray SOCKS 默认端口
- 443: HTTPS/Trojan/Reality 常用端口
- 8388: Shadowsocks 默认端口
- 10000-65535: 自定义端口范围

探测策略:
- 扫描常见端口
- 对开放端口发送协议探测
- 记录响应特征
```

**流量模式探测**

流量模式探测分析服务器在不同负载下的行为。

```
流量模式探测:

1. 小数据探测
   发送: 10 字节数据
   观察: 响应大小、响应时间

2. 大数据探测
   发送: 100KB 数据
   观察: 响应大小、响应时间、吞吐量

3. 并发探测
   建立: 10 个并发连接
   观察: 连接处理行为、资源消耗

模式分析:
- 代理服务器可能有特定的负载均衡行为
- 正常网站通常有缓存、CDN 行为
- 响应时间模式可能暴露代理特征
```

### 探测特征

识别探测请求的特征有助于防御：

**探测来源特征**

```
探测 IP 特征:
- 已知的探测 IP 段
- 数据中心 IP
- 云服务商 IP
- 异常地理位置

探测行为特征:
- 短时间内多次连接
- 连接后立即发送探测请求
- 不完整的握手流程
- 异常的请求序列
```

**探测请求特征**

```
TLS 探测特征:
- 使用已知的探测 SNI
- 使用过时的 TLS 版本
- 使用异常的密码套件
- 缺少常见扩展

HTTP 探测特征:
- 使用默认的 User-Agent
- 缺少常见请求头
- 使用 HEAD 方法
- 请求不存在路径

SOCKS 探测特征:
- 使用默认端口
- 使用空认证
- 请求访问内部地址
- 使用无效的目标地址
```

**探测时序特征**

```
时间模式:
- 集中在特定时间段
- 固定的探测间隔
- 与 DPI 检测结果关联
- 随机化探测时间

连接模式:
- 连接持续时间短
- 连接后无数据传输
- 快速重试不同端口
- 使用不同的探测协议
```

### 防御策略

对抗主动探测的防御策略：

**身份验证**

最有效的防御方式是只响应授权客户端：

```
身份验证机制:

1. 预共享密钥 (PSK)
   - 客户端和服务器共享密码
   - 密码嵌入在请求中
   - 服务器验证密码后响应

2. 身份标记
   - 客户端在请求中嵌入特殊标记
   - 标记位置隐蔽（如 TLS 扩展）
   - 服务器检测标记后响应

3. 证书固定
   - 客户端固定服务器证书
   - 防止中间人攻击
   - 确保连接唯一性
```

**响应伪装**

对未授权请求返回正常网站响应：

```
响应伪装机制:

1. 回退机制
   ┌─────────────────────────────────────┐
   │ 收到未授权请求                       │
   │         ↓                           │
   │ 判断是否有身份标记                   │
   │    ├── 有 → 代理服务                 │
   │    └── 无 → 回退到真实网站           │
   └─────────────────────────────────────┘

2. 前端伪装
   - 部署真实网站作为前端
   - 代理服务作为后端
   - 根据请求路由

3. 协议分层
   - 外层协议返回正常响应
   - 内层协议处理代理请求
   - 多层嵌套增加检测难度
```

**探测检测**

识别探测请求并做出特殊响应：

```
探测检测机制:

1. IP 黑名单
   - 维护已知探测 IP 列表
   - 对黑名单 IP 返回正常响应
   - 定期更新黑名单

2. 行为分析
   - 分析请求模式
   - 识别异常行为
   - 动态调整响应策略

3. 挑战-响应
   - 对可疑请求发送挑战
   - 验证客户端是否为真实用户
   - 浏览器可以响应，探测工具无法响应
```

**流量混淆**

混淆代理流量，增加探测难度：

```
流量混淆技术:

1. 协议模拟
   - 模拟 HTTP/2 流量
   - 模拟 WebSocket 流量
   - 模拟 gRPC 流量

2. 流量填充
   - 添加随机数据
   - 固定包大小
   - 随机化时序

3. 协议嵌套
   - TLS 内嵌 TLS
   - HTTP 内嵌代理
   - 多层协议栈
```

## 在 Prism 中的应用

Prism 实现了多层主动探测防御机制：

### Reality 认证

Reality 协议使用身份标记来区分合法客户端和探测者：

```cpp
// src/prism/stealth/reality/auth.cpp

auto reality_authenticator::verify_client_hello(
    const client_hello_message& hello,
    const reality_config& config
) -> auth_result {
    // 1. 检查 short_id
    auto short_id_opt = extract_short_id(hello);
    if (!short_id_opt) {
        return auth_result::fallback;  // 无标记，回退
    }

    // 2. 验证 short_id 是否有效
    if (!is_valid_short_id(*short_id_opt, config.short_ids)) {
        return auth_result::fallback;  // 无效标记，回退
    }

    // 3. 提取认证数据
    auto auth_data = extract_auth_data(hello);
    if (!auth_data) {
        return auth_result::fallback;
    }

    // 4. 验证认证数据
    if (!verify_auth_data(*auth_data, config)) {
        return auth_result::rejected;  // 认证失败
    }

    return auth_result::accepted;
}
```

### ShadowTLS 认证

ShadowTLS v3 引入了密码认证机制：

```cpp
// src/prism/stealth/shadowtls/auth.cpp

auto shadowtls_authenticator::authenticate(
    const std::vector<uint8_t>& data,
    const shadowtls_config& config
) -> auth_result {
    // 1. 检查是否有认证数据
    if (data.size() < auth_header_size) {
        return auth_result::fallback;
    }

    // 2. 提取密码哈希
    auto received_hash = extract_password_hash(data);

    // 3. 计算期望的密码哈希
    auto expected_hash = compute_password_hash(
        config.password,
        get_current_timestamp()
    );

    // 4. 比较哈希
    if (received_hash == expected_hash) {
        return auth_result::accepted;
    }

    // 5. 检查时间窗口内的其他有效哈希
    for (int offset = -time_window; offset <= time_window; ++offset) {
        auto alt_hash = compute_password_hash(
            config.password,
            get_current_timestamp() + offset
        );
        if (received_hash == alt_hash) {
            return auth_result::accepted;
        }
    }

    return auth_result::fallback;
}
```

### 回退机制

当检测到探测请求时，Prism 回退到真实网站响应：

```cpp
// src/prism/stealth/reality/handshake.cpp

auto reality_handshake::handle_unauthorized_client(
    const tcp_socket& socket,
    const std::string& dest_host,
    uint16_t dest_port
) -> awaitable<void> {
    // 1. 建立到真实网站的连接
    auto dest_socket = co_await connect_to_dest(dest_host, dest_port);

    // 2. 转发客户端的 ClientHello
    co_await async_write(dest_socket, buffer(client_hello_data_));

    // 3. 转发真实网站的响应给客户端
    std::vector<uint8_t> response(buffer_size);
    auto bytes_read = co_await dest_socket.async_read_some(buffer(response));
    co_await async_write(socket, buffer(response.data(), bytes_read));

    // 4. 双向转发数据
    co_await bidirectional_transfer(socket, dest_socket);

    co_return;
}
```

### 探测检测

Prism 实现了探测请求检测：

```cpp
// src/prism/recognition/probe/detector.cpp

class probe_detector {
public:
    auto detect(const connection_info& info) -> probe_type {
        // 1. 检查 IP 黑名单
        if (is_blacklisted_ip(info.remote_ip)) {
            return probe_type::known_probe_ip;
        }

        // 2. 检查连接模式
        if (is_suspicious_pattern(info.remote_ip)) {
            return probe_type::suspicious_pattern;
        }

        // 3. 检查请求特征
        if (has_probe_characteristics(info.request_data)) {
            return probe_type::probe_request;
        }

        return probe_type::normal;
    }

private:
    auto is_suspicious_pattern(const std::string& ip) -> bool {
        // 检查短时间内多次连接
        auto count = get_connection_count(ip, std::chrono::minutes(5));
        return count > threshold_;
    }

    auto has_probe_characteristics(const std::span<uint8_t>& data) -> bool {
        // 检查已知的探测特征
        for (const auto& pattern : probe_patterns_) {
            if (match_pattern(data, pattern)) {
                return true;
            }
        }
        return false;
    }
};
```

### 配置示例

Reality 配置：

```json
{
  "stealth": {
    "reality": {
      "dest": "www.microsoft.com:443",
      "server_names": ["www.microsoft.com", "microsoft.com"],
      "private_key": "GENERATED_PRIVATE_KEY",
      "short_ids": [
        "0123456789abcdef",
        "fedcba9876543210",
        "abcdef0123456789"
      ],
      "probe_detection": {
        "enabled": true,
        "blacklist_ttl": 3600,
        "rate_limit": 10
      }
    }
  }
}
```

ShadowTLS 配置：

```json
{
  "stealth": {
    "shadowtls": {
      "version": 3,
      "password": "your-secure-password-here",
      "handshake": {
        "dest": "www.cloudflare.com:443",
        "server_name": "www.cloudflare.com"
      },
      "probe_detection": {
        "enabled": true,
        "fallback_behavior": "proxy_real_response"
      }
    }
  }
}
```

## 最佳实践

### 部署建议

**端口选择**
- 使用 443 端口（HTTPS）最隐蔽
- 避免使用常见代理端口（1080, 8388 等）
- 使用与伪装网站相同的端口

**域名选择**
- 使用真实域名（需要域名和证书）
- Reality 可以使用任意域名
- 避免使用可疑域名

**密钥管理**
- 定期轮换密钥
- 使用强密码
- 安全存储私钥

### 防御策略

**多层防御**
1. 身份验证（Reality/ShadowTLS）
2. 探测检测（IP 黑名单）
3. 响应伪装（回退机制）
4. 流量混淆（协议嵌套）

**监控告警**
```json
{
  "monitoring": {
    "probe_detection": {
      "alert_threshold": 5,
      "alert_interval_minutes": 60,
      "log_suspicious": true
    }
  }
}
```

**动态调整**
- 根据探测频率调整防御级别
- 在探测高峰期启用严格模式
- 在正常时期使用宽松模式

## 常见问题

### Q1: 主动探测能被完全防御吗？

不能。主动探测者可以模拟真实客户端的行为，存在以下挑战：
- 探测者可以获取合法客户端的请求模式
- 探测者可以长期潜伏收集信息
- 探测者可以使用大量 IP 地址

防御的目标是提高探测成本，而非完全阻止。

### Q2: 如何检测自己被探测了？

可以通过以下迹象判断：
- 服务器收到大量异常连接
- 连接来自可疑 IP 段
- 连接后无正常数据传输
- 收到非标准协议请求

启用日志监控可以实时检测探测行为。

### Q3: Reality 和 ShadowTLS 哪个更安全？

两者都有效，但侧重点不同：

| 特性 | Reality | ShadowTLS |
|------|---------|-----------|
| 主动探测防护 | 极强（回退机制） | 强（密码认证） |
| 部署复杂度 | 中等 | 低 |
| 性能开销 | 较高 | 低 |
| 需要外部连接 | 是 | 否 |

### Q4: 如何选择回退目标？

选择回退目标的建议：
- 选择知名网站（流量大，不易引起怀疑）
- 选择支持 TLS 1.3 的网站
- 选择与代理服务同一地区的网站
- 避免选择已知的敏感网站

### Q5: 探测 IP 黑名单如何获取？

获取方式：
- 公开黑名单（如 GitHub 项目）
- 自己收集（监控日志）
- 社区共享
- 商业情报服务

### Q6: 如何应对大规模探测？

应对策略：
- 启用严格的身份验证
- 使用分布式部署
- 动态切换端口和域名
- 启用 IP 限速

### Q7: 回退机制会增加延迟吗？

是的，回退需要：
- 建立到真实网站的连接
- 完成 TLS 握手
- 转发请求和响应

但回退只对探测者发生，合法客户端不受影响。

## 参考资料

- [The Great Firewall's Active Probing](https://gfw.report/talks/ccs2020/en/)
- [Active Probing in China](https://www.usenix.org/conference/foci19/presentation/wustrow)
- [ShadowTLS Protocol Specification](https://github.com/ihciah/shadow-tls)
- [Reality Protocol Design](https://github.com/XTLS/REALITY)
- [Detecting Proxies with Active Probing](https://www.internetsociety.org/doc/detecting-proxy-servers-active-probing)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹识别
- [[ref/anti-censorship/mitm-hijack|中间人劫持]] — MITM 攻击防御
- [[ref/anti-censorship/traffic-analysis|流量分析]] — 流量分析对抗
- [[prism/stealth/reality|Reality 协议]] — Reality 实现细节
- [[prism/stealth/shadowtls|ShadowTLS 协议]] — ShadowTLS 实现细节