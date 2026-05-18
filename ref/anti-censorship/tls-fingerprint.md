---
title: "TLS 指纹 (JA3/JA4)"
category: "anti-censorship"
type: ref
module: ref
source: "概念文档"
tags: [流量对抗, tls, 指纹, ja3, ja4, 客户端识别, 指纹识别]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# TLS 指纹 (JA3/JA4)

**类别**: 流量对抗

## 概述

TLS 指纹（TLS Fingerprinting）是一种通过分析 TLS 握手过程中 ClientHello 消息的特征来识别客户端软件的技术。由于不同的客户端软件（如浏览器、代理工具、命令行工具）在实现 TLS 协议时存在差异，这些差异形成了一种独特的"指纹"，可以用于识别客户端类型或检测异常流量。

TLS 指纹技术最初由 Salesforce 的安全研究团队在 2015 年提出，被称为 JA3。随着 TLS 协议的演进，尤其是 TLS 1.3 的普及，原始的 JA3 指纹逐渐暴露出局限性。2023 年，FoxIO 提出了 JA4 指纹方案，提供了更细粒度的识别能力和更好的可扩展性。

在审查对抗领域，TLS 指纹检测被广泛用于识别代理软件。许多代理工具使用自研的 TLS 实现，或者使用非标准的配置，这些都产生了与主流浏览器不同的指纹。审查系统可以通过比对指纹库来识别这些异常客户端，即使流量内容被加密。

TLS 指纹的核心价值在于：它不需要解密流量，仅通过分析明文传输的 ClientHello 就能获取大量信息。这使得指纹检测具有极低的计算成本和极高的检测效率，成为 DPI 系统的标准组件之一。

从防御角度来看，对抗 TLS 指纹检测有两种主要策略：一是使用真实的 TLS 实现库（如 BoringSSL、OpenSSL），产生与主流软件相同的指纹；二是使用指纹随机化技术，每次连接生成不同的指纹。前者更可靠，后者更灵活。

Prism 在设计中充分考虑了 TLS 指纹问题。通过使用 BoringSSL 作为 TLS 实现库，并精心配置 TLS 参数，Prism 能够产生与主流浏览器一致的指纹。同时，Prism 的 Reality 和 ShadowTLS 方案能够完全复用真实网站的 TLS 实现，从根本上解决指纹问题。

理解 TLS 指纹技术对于设计安全的代理系统至关重要。本文将详细介绍 JA3 和 JA4 指纹的原理、检测方法、规避策略，以及在 Prism 中的应用实现。

## 原理详解

### TLS ClientHello 结构

TLS 指纹的基础是 ClientHello 消息。在 TLS 握手开始时，客户端发送 ClientHello 消息，其中包含客户端支持的 TLS 版本、密码套件、扩展等关键信息。这些信息大部分以明文形式传输，即使使用 TLS 1.3 也是如此。

```
TLS ClientHello 消息结构:

Record Layer (5 bytes)
├── Content Type: 0x16 (Handshake)
├── Version: TLS 1.0 (0x0301)
└── Length: (2 bytes)

Handshake Layer
├── Handshake Type: 0x01 (ClientHello)
├── Length: (3 bytes)
└── ClientHello Content
    ├── Version: TLS 1.2 (0x0303) [TLS 1.3 使用兼容版本]
    ├── Random: 32 bytes
    ├── Session ID: 0-32 bytes
    ├── Cipher Suites: 变长
    │   └── [TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384, ...]
    ├── Compression Methods: 1-2 bytes
    │   └── [null]
    └── Extensions: 变长
        ├── Server Name (0x0000): 目标域名
        ├── Supported Versions (0x002b): TLS 1.3
        ├── Signature Algorithms (0x000d): 签名算法列表
        ├── Supported Groups (0x000a): 椭圆曲线列表
        ├── EC Point Formats (0x000b): 点格式
        ├── ALPN (0x0010): 应用层协议
        ├── Key Share (0x0033): 密钥共享
        └── ... 更多扩展
```

不同客户端在以下方面存在差异：

**TLS 版本选择**
- 浏览器通常支持最新版本 (TLS 1.3)
- 旧版工具可能只支持 TLS 1.2 或更低
- 版本协商顺序可能不同

**密码套件列表**
- 列表顺序反映客户端优先级
- 不同实现支持的密码套件集合不同
- 某些密码套件可能被禁用

**扩展列表**
- 扩展的类型和数量不同
- 扩展的顺序可能不同
- 扩展的内容格式不同

**椭圆曲线和签名算法**
- 支持的曲线类型不同
- 签名算法优先级不同
- 点格式偏好不同

### JA3 指纹

JA3 是最早的 TLS 指纹方案，由 Salesforce 于 2015 年提出。它通过提取 ClientHello 中的五个关键字段生成 MD5 哈希值作为指纹。

**JA3 计算过程**

```
JA3 字符串格式:
TLSVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats

示例:
771,4865-4866-4867-49195-49199-49196-49200-52393-52392-49171-49172-156-157-47-53,0-23-65281-10-11-35-16-5-13-18-51-45-43-27-21,29-23-24,0

JA3 指纹 = MD5(JA3字符串) = cd08e30a67c0d2e6f3d6b3d5e8e0d4a2
```

**字段详解**

1. **TLSVersion**: 客户端声明的 TLS 版本号
   - TLS 1.0 = 769 (0x0301)
   - TLS 1.1 = 770 (0x0302)
   - TLS 1.2 = 771 (0x0303)
   - TLS 1.3 使用兼容版本 771

2. **Ciphers**: 密码套件列表，用连字符连接
   - 顺序表示客户端优先级
   - TLS 1.3 密码套件以 4865/4866/4867 开头
   - TLS 1.2 密码套件如 49195 (TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256)

3. **Extensions**: 扩展列表，用连字符连接
   - 扩展 ID 是数字
   - 顺序在不同客户端中可能不同

4. **EllipticCurves**: 支持的椭圆曲线列表
   - 29 = X25519
   - 23 = secp256r1
   - 24 = secp384r1

5. **EllipticCurvePointFormats**: 点格式列表
   - 0 = uncompressed
   - 1 = ansiX962_compressed_prime
   - 2 = ansiX962_compressed_char2

**JA3 的局限性**

JA3 存在一些已知的局限性：

1. **TLS 1.3 支持不完善**
   - TLS 1.3 引入了新的扩展 (supported_versions, key_share)
   - JA3 无法充分利用这些新信息

2. **扩展顺序敏感**
   - 某些客户端可能随机化扩展顺序
   - 导致同一客户端产生不同指纹

3. **信息丢失**
   - 不包含 SNI 信息
   - 不包含 ALPN 信息
   - 无法区分不同配置

4. **MD5 碰撞风险**
   - 理论上存在碰撞可能
   - 实际影响较小

### JA4 指纹

JA4 是 JA3 的升级版，由 FoxIO 于 2023 年提出，针对 TLS 1.3 进行了优化，并提供更细粒度的识别能力。

**JA4 指纹结构**

JA4 由四个部分组成，每个部分独立哈希：

```
JA4 指纹结构:
JA4 = <protocol>_<features>_<extensions_hash>
JA4S = <protocol>_<features>_<extensions_hash>  (服务器指纹)
JA4H = <protocol>_<features>_<extensions_hash>_<headers_hash>  (HTTP指纹)
JA4L = <protocol>_<features>_<extensions_hash>  (轻量级指纹)
JA4X = <protocol>_<features>_<extensions_hash>  (扩展指纹)
```

**JA4 计算示例**

```
JA4 指纹示例:
t13d1515h2_8daaf6152771_ecb9c73d4dae

分解:
- t13: TLS 1.3
- d: 使用域名 SNI
- 1515: 15 个密码套件，15 个扩展
- h2: HTTP/2 ALPN
- 8daaf6152771: 密码套件哈希 (前 12 字符)
- ecb9c73d4dae: 扩展哈希 (前 12 字符)
```

**JA4 的改进**

1. **TLS 1.3 原生支持**
   - 正确处理 supported_versions 扩展
   - 包含 key_share 信息

2. **更细粒度**
   - 区分 SNI 类型 (域名/IP/无)
   - 区分 ALPN 协议
   - 独立哈希各组件

3. **更好的可扩展性**
   - 模块化设计
   - 支持新扩展
   - 支持指纹更新

4. **标准化**
   - 使用 SHA256 截断代替 MD5
   - 更清晰的指纹格式

**JA4 与 JA3 对比**

| 特性 | JA3 | JA4 |
|------|-----|-----|
| 发布年份 | 2015 | 2023 |
| 哈希算法 | MD5 | SHA256 截断 |
| TLS 1.3 支持 | 有限 | 完整 |
| 扩展处理 | 简单列表 | 分类处理 |
| SNI 信息 | 无 | 包含 |
| ALPN 信息 | 无 | 包含 |
| 指纹长度 | 32 字符 | 变长 |
| 标准化程度 | 社区标准 | 行业标准 |

### 检测方法

DPI 系统使用 TLS 指纹检测代理流量的方法：

**精确匹配**

最简单的方法是将观察到的指纹与已知代理指纹库进行精确匹配。

```
已知代理指纹库:
- V2Ray VMess: cd08e30a67c0d2e6f3d6b3d5e8e0d4a2
- Shadowsocks: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
- Trojan: q1w2e3r4t5y6u7i8o9p0a1s2d3f4g5h6

检测流程:
1. 提取 ClientHello
2. 计算 JA3/JA4 指纹
3. 查询指纹库
4. 匹配则标记为代理流量
```

**模糊匹配**

对于未知指纹，可以使用模糊匹配来判断是否为可疑客户端：

```
可疑特征检查:
1. 密码套件顺序异常
   - 正常浏览器按安全性排序
   - 代理工具可能按字母顺序排序

2. 扩展顺序异常
   - 浏览器有固定的扩展顺序
   - 代理工具可能随机化

3. 缺失常见扩展
   - 缺少 status_request (OCSP)
   - 缺少 signed_certificate_timestamp

4. 非标准密码套件
   - 包含已弃用的算法
   - 包含不常见的密码套件
```

**机器学习检测**

高级检测系统使用机器学习模型分析 TLS 指纹：

```
特征向量:
[版本, 密码套件数量, 扩展数量, 曲线数量, 密码套件熵,
 扩展熵, 点格式熵, 特征组合...]

分类器:
- 随机森林
- 支持向量机
- 深度神经网络

输出:
- 代理概率: 0.0 - 1.0
- 客户端类型: 浏览器/代理/爬虫/未知
```

### 规避策略

对抗 TLS 指纹检测的几种策略：

**使用标准 TLS 库**

最简单有效的方法是使用主流浏览器的 TLS 库：

```
推荐配置:
- BoringSSL (Chrome 使用)
- OpenSSL (广泛使用)
- Secure Transport (Safari 使用)

避免使用:
- 自研 TLS 实现
- 非标准配置
- 过时版本的库
```

**指纹伪装**

通过精心配置 TLS 参数来模拟目标客户端：

```
Chrome 浏览器指纹特征:
- TLS Version: 771 (兼容模式)
- Ciphers: 4865-4866-4867-4868-49195-49199-...
- Extensions: 0-23-65281-10-11-35-16-5-13-18-51-45-43-27-21
- Curves: 29-23-24-25
- Point Formats: 0

配置要点:
1. 使用相同的密码套件顺序
2. 使用相同的扩展顺序
3. 包含所有必要的扩展
4. 使用标准的曲线和点格式
```

**指纹随机化**

每次连接生成不同的指纹，增加检测难度：

```
随机化策略:
1. 扩展顺序随机化
   - 打乱扩展列表顺序
   - 但保持关键扩展优先

2. 密码套件顺序随机化
   - 在安全等价组内随机化
   - 不影响安全性

3. 添加随机扩展
   - 添加无害的扩展
   - 增加指纹熵

注意:
- 过度随机化可能被检测
- 需要保持语义正确性
```

**流量嵌套**

使用真实的 TLS 连接作为外层：

```
TLS-in-TLS 结构:
┌─────────────────────────────────────┐
│ 外层 TLS (真实指纹)                   │
│   ┌─────────────────────────────┐   │
│   │ 内层 TLS (代理协议)          │   │
│   │   ┌─────────────────────┐   │   │
│   │   │ 应用层数据           │   │   │
│   │   └─────────────────────┘   │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘

方案:
- Reality: 使用真实网站的 TLS
- ShadowTLS: 代理真实 TLS 握手
- Restls: 模拟真实 TLS 行为
```

## 在 Prism 中的应用

Prism 在多个层面实现了 TLS 指纹对抗机制。

### 特征分析器

Prism 的 `recognition` 模块实现了 ClientHello 特征分析，用于检测客户端指纹：

```cpp
// include/prism/recognition/probe/analyzer.hpp

/// @brief ClientHello 特征结构
struct client_hello_features {
    uint16_t version;                    ///< TLS 版本
    std::vector<uint16_t> cipher_suites; ///< 密码套件列表
    std::vector<uint16_t> extensions;    ///< 扩展列表
    std::vector<uint16_t> curves;        ///< 椭圆曲线列表
    std::vector<uint8_t> point_formats;  ///< 点格式列表
    std::string sni;                     ///< SNI 域名
    std::vector<std::string> alpn;       ///< ALPN 协议列表
};
```

### 指纹计算

Prism 实现了 JA3 和 JA4 指纹计算：

```cpp
// src/prism/protocol/tls/fingerprint.cpp

auto calculate_ja3(const client_hello_features& features) -> std::string {
    std::string ja3_str = fmt::format("{},", features.version);

    // 密码套件
    for (size_t i = 0; i < features.cipher_suites.size(); ++i) {
        if (i > 0) ja3_str += "-";
        ja3_str += std::to_string(features.cipher_suites[i]);
    }
    ja3_str += ",";

    // 扩展
    for (size_t i = 0; i < features.extensions.size(); ++i) {
        if (i > 0) ja3_str += "-";
        ja3_str += std::to_string(features.extensions[i]);
    }
    ja3_str += ",";

    // 曲线
    for (size_t i = 0; i < features.curves.size(); ++i) {
        if (i > 0) ja3_str += "-";
        ja3_str += std::to_string(features.curves[i]);
    }
    ja3_str += ",";

    // 点格式
    for (size_t i = 0; i < features.point_formats.size(); ++i) {
        if (i > 0) ja3_str += "-";
        ja3_str += std::to_string(features.point_formats[i]);
    }

    return crypto::md5(ja3_str);
}
```

### Reality 指纹复用

Reality 方案通过复用真实网站的 TLS 实现来获得完美指纹：

```cpp
// src/prism/stealth/reality/handshake.cpp

auto reality_handshake::fetch_dest_certificate(
    const std::string& dest_host,
    uint16_t dest_port
) -> awaitable<certificate_info> {
    // 建立到真实网站的 TLS 连接
    auto socket = co_await connect_to_dest(dest_host, dest_port);

    // 执行 TLS 握手
    auto cert = co_await get_peer_certificate(socket);

    // 提取证书指纹
    auto fingerprint = calculate_certificate_fingerprint(cert);

    co_return certificate_info{cert, fingerprint};
}
```

### ShadowTLS 配置

ShadowTLS 配置中可以指定 TLS 参数：

```json
{
  "stealth": {
    "shadowtls": {
      "version": 3,
      "password": "your-password",
      "handshake": {
        "dest": "www.microsoft.com:443",
        "server_name": "www.microsoft.com",
        "cipher_suites": [
          "TLS_AES_128_GCM_SHA256",
          "TLS_AES_256_GCM_SHA384",
          "TLS_CHACHA20_POLY1305_SHA256"
        ],
        "curves": ["x25519", "secp256r1", "secp384r1"]
      }
    }
  }
}
```

### 特征位图

Prism 使用特征位图进行快速指纹匹配：

```cpp
// include/prism/protocol/tls/feature_bitmap.hpp

/// @brief TLS 特征位图
/// @details 用于快速匹配 ClientHello 特征
class feature_bitmap {
public:
    /// 设置密码套件位
    void set_cipher_suite(uint16_t cipher);

    /// 设置扩展位
    void set_extension(uint16_t ext);

    /// 设置曲线位
    void set_curve(uint16_t curve);

    /// 检查是否匹配指纹模式
    auto matches(const fingerprint_pattern& pattern) const -> bool;

private:
    std::bitset<65536> cipher_mask_;   ///< 密码套件位图
    std::bitset<65536> extension_mask_; ///< 扩展位图
    std::bitset<64> curve_mask_;        ///< 曲线位图
};
```

### 指纹库管理

Prism 维护已知指纹库用于检测和规避：

```cpp
// include/prism/recognition/probe/fingerprint_db.hpp

/// @brief 指纹数据库
class fingerprint_database {
public:
    /// 加载指纹库
    void load(const std::filesystem::path& db_path);

    /// 查询指纹
    auto lookup(const std::string& ja3) const -> std::optional<fingerprint_info>;

    /// 添加指纹
    void add(const std::string& ja3, const fingerprint_info& info);

    /// 获取浏览器指纹列表
    auto get_browser_fingerprints() const -> std::vector<std::string>;

    /// 获取代理指纹列表
    auto get_proxy_fingerprints() const -> std::vector<std::string>;

private:
    std::unordered_map<std::string, fingerprint_info> fingerprints_;
};
```

## 最佳实践

### 生成合规指纹

使用 Prism 工具生成符合主流浏览器的指纹：

```bash
# 生成 Reality 密钥对和指纹
scripts/GenRealityKeys.sh

# 查看当前配置的指纹
build_release/src/Prism.exe --show-fingerprint
```

### 配置建议

**Reality 配置**
```json
{
  "stealth": {
    "reality": {
      "dest": "www.google.com:443",
      "server_names": ["www.google.com", "google.com"],
      "private_key": "...",
      "short_ids": ["0123456789abcdef", "fedcba9876543210"]
    }
  }
}
```

选择目标网站的准则：
- 知名度高，流量大
- 支持 TLS 1.3
- 支持 HTTP/2
- 服务器稳定

**ShadowTLS 配置**
```json
{
  "stealth": {
    "shadowtls": {
      "version": 3,
      "password": "secure-password-here",
      "handshake": {
        "dest": "www.cloudflare.com:443",
        "server_name": "www.cloudflare.com"
      }
    }
  }
}
```

### 指纹测试

测试配置是否产生正确的指纹：

```bash
# 使用在线工具测试
curl -v https://your-server:443 2>&1 | grep -A 20 "ClientHello"

# 使用 openssl 测试
openssl s_client -connect your-server:443 -tlsextdebug

# 使用 Wireshark 分析
# 过滤器: tls.handshake.type == 1
```

### 验证指纹一致性

确保服务器指纹与目标网站一致：

```bash
# 比较指纹
diff <(openssl s_client -connect your-server:443 2>/dev/null | openssl x509 -fingerprint -sha256 -noout) \
     <(openssl s_client -connect www.google.com:443 2>/dev/null | openssl x509 -fingerprint -sha256 -noout)
```

## 常见问题

### Q1: JA3 指纹会随时间变化吗？

会的。浏览器更新、TLS 库升级都会导致指纹变化。需要定期更新指纹库。

### Q2: 如何处理指纹随机化？

某些客户端会随机化扩展顺序或密码套件顺序。对于这种情况：
- 使用 JA4 的分类处理方式
- 在匹配前对指纹组件排序
- 使用模糊匹配而非精确匹配

### Q3: TLS 1.3 对指纹有什么影响？

TLS 1.3 引入了新的扩展和密码套件：
- `supported_versions` 扩展替代了 ClientHello 中的版本字段
- `key_share` 扩展包含密钥共享数据
- 新的 TLS 1.3 专用密码套件

JA4 比 JA3 更好地处理了这些变化。

### Q4: ECH 如何影响指纹？

Encrypted Client Hello (ECH) 加密了大部分 ClientHello 信息：
- 外层 ClientHello 包含伪装的 SNI
- 内层 ClientHello（加密）包含真实信息
- DPI 只能看到外层 ClientHello

这使得传统的 TLS 指纹技术失效，需要新的检测方法。

### Q5: 如何识别代理软件？

代理软件常见的指纹特征：
- 使用非标准的密码套件顺序
- 缺少浏览器常见的扩展
- 包含不常见的扩展组合
- 使用自签名证书或固定证书

### Q6: Prism 如何确保指纹一致性？

Prism 采用多层保障：
1. 使用 BoringSSL 作为 TLS 实现（与 Chrome 相同）
2. Reality 方案复用真实网站的完整 TLS 实现
3. 定期更新指纹配置
4. 自动化测试验证指纹正确性

### Q7: 可以自定义指纹吗？

可以，但不推荐：
- 自定义指纹容易产生异常特征
- 需要深入研究浏览器行为
- 维护成本高

推荐使用 Reality 或 ShadowTLS 方案，自动获得正确指纹。

## 参考资料

- [JA3 - A Method for Profiling SSL/TLS Clients](https://github.com/salesforce/ja3)
- [JA4+ TLS Client Fingerprinting](https://github.com/FoxIO-LLC/ja4)
- [TLS 1.3 RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)
- [TLS ClientHello Parsing](https://tls.ulfheim.net/)
- [Browser TLS Fingerprint Database](https://tls.peet.ws/)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/probing|主动探测]] — 主动探测防御
- [[ref/anti-censorship/traffic-analysis|流量分析]] — 流量分析技术
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 结构解析
- [[prism/stealth/reality|Reality 协议]] — Reality 实现细节
- [[prism/recognition|协议识别]] — 协议识别模块