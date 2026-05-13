---
title: "Prism 中的 TLS ClientHello 特征分析"
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags: [tls, clienthello, feature-bitmap, reality, shadowtls, recognition, prism]
related: ["[[protocol]]", "[[protocol/common]]", "[[stealth]]", "[[recognition]]"]
---

> 相关：[[protocol]] | [[protocol/common]] | [[stealth]] | [[recognition]] | [[protocol/http]] | [[protocol/socks5]]

# TLS ClientHello 特征分析

## 1. 模块定位

TLS 模块位于 `include/prism/protocol/tls/`，是 Prism 中的**中立共享层**。它不实现 TLS 握手，而是专注于从 ClientHello 记录中提取协议特征。该模块被两个上层消费者共同使用：

- **recognition 模块**：在协议识别阶段解析 ClientHello，构建特征位图，驱动伪装方案的 Tier 0 快速检测
- **stealth 模块**：在 Reality/ShadowTLS 等伪装方案中复用 ClientHello 原始数据和解析逻辑

模块设计为无状态纯函数，所有方法均不持有上下文。

## 2. 文件结构

```
include/prism/protocol/tls/
├── types.hpp           # 类型定义与常量
├── signal.hpp          # ClientHello 解析器接口
└── feature_bitmap.hpp  # 特征位图构建与匹配

src/prism/protocol/tls/
└── signal.cpp          # 解析器实现
```

## 3. 类型定义（types.hpp）

### 3.1 TLS 记录层常量

```cpp
constexpr std::size_t RECORD_HEADER_LEN = 5;        // ContentType(1) + Version(2) + Length(2)
constexpr std::size_t MAX_RECORD_PAYLOAD = 16384;    // 最大记录载荷

// Content Type
constexpr std::uint8_t CONTENT_TYPE_HANDSHAKE       = 0x16;
constexpr std::uint8_t CONTENT_TYPE_APPLICATION_DATA = 0x17;

// Handshake Type
constexpr std::uint8_t HANDSHAKE_TYPE_CLIENT_HELLO  = 0x01;

// Extension Type
constexpr std::uint16_t EXT_SERVER_NAME          = 0x0000;
constexpr std::uint16_t EXT_KEY_SHARE            = 0x0033;
constexpr std::uint16_t EXT_SUPPORTED_VERSIONS   = 0x002B;

// Named Groups
constexpr std::uint16_t NAMED_GROUP_X25519       = 0x001D;
constexpr std::uint16_t NAMED_GROUP_X25519_MLKEM768 = 0x11EC;  // 后量子混合
```

### 3.2 ClientHello 特征结构

```cpp
struct client_hello_features
{
    memory::string server_name;                    // SNI 服务器名称
    memory::vector<std::uint8_t> session_id;       // session_id 原始数据
    std::uint8_t session_id_len{0};                // session_id 长度（0-32）
    bool has_x25519{false};                        // 是否有 X25519 key_share
    std::array<std::uint8_t, 32> x25519_key{};     // X25519 公钥
    memory::vector<std::uint16_t> versions;        // 支持的 TLS 版本列表
    std::array<std::uint8_t, 32> random{};         // 客户端随机数
    memory::vector<std::uint8_t> raw_hs_msg;       // 原始握手消息（不含 record header）
    memory::vector<std::byte> raw_record;          // 原始记录（含 record header）
};
```

关键字段说明：

| 字段 | 用途 | 消费者 |
|------|------|--------|
| `server_name` | 路由决策、SNI 匹配 | recognition、pipeline |
| `session_id` | Reality 标记检测（`[0x01, 0x08, 0x02]`） | stealth::reality |
| `x25519_key` | Reality 密钥交换 | stealth::reality |
| `versions` | 特征位图 `has_supported_versions` | recognition |
| `random` | TLS 握手重放 | stealth |
| `raw_hs_msg` | 伪装方案握手转发 | stealth |
| `raw_record` | 完整记录重放 | stealth |

### 3.3 工具函数

types.hpp 提供 TLS 记录层写入工具：

```cpp
write_u8(buf, val);    // 写 1 字节
write_u16(buf, val);   // 写 2 字节（大端序）
write_u24(buf, val);   // 写 3 字节（大端序）
```

## 4. ClientHello 解析器（signal.hpp / signal.cpp）

### 4.1 接口

```cpp
// 从传输层读取完整 TLS 记录
auto read_tls_record(transmission &transport)
    -> awaitable<pair<fault::code, vector<uint8_t>>>;

// 带预读数据的变体
auto read_tls_record(transmission &transport, span<const byte> preread)
    -> awaitable<pair<fault::code, vector<uint8_t>>>;

// 从完整记录解析 ClientHello 特征
auto parse_client_hello(span<const uint8_t> record)
    -> pair<fault::code, client_hello_features>;
```

### 4.2 解析流程（signal.cpp）

`parse_client_hello` 是纯函数，解析流程如下：

```
TLS 记录（含 5 字节 record header）
  │
  ├── 验证 content_type == 0x16 (Handshake)
  ├── 验证 record_length <= 16384
  │
  ├── Handshake 层
  │   ├── 验证 handshake_type == 0x01 (ClientHello)
  │   ├── 提取 ClientVersion（跳过 2 字节）
  │   ├── 提取 Random（32 字节 → features.random）
  │   ├── 提取 SessionID（1 字节长度 + 数据）
  │   ├── 跳过 CipherSuites（2 字节长度 + 数据）
  │   └── 跳过 CompressionMethods（1 字节长度 + 数据）
  │
  └── Extensions 层（parse_extensions）
      ├── EXT_SERVER_NAME (0x0000) → parse_sni → features.server_name
      ├── EXT_KEY_SHARE (0x0033) → parse_key_share → features.x25519_key
      │   ├── 纯 X25519 (0x001D, 32B) → 直接提取
      │   └── X25519MLKEM768 (0x11EC, ≥32B) → 提取前 32 字节
      └── EXT_SUPPORTED_VERSIONS (0x002B) → parse_versions → features.versions
```

### 4.3 X25519MLKEM768 混合密钥支持

解析器同时支持纯 X25519 和后量子 X25519MLKEM768 混合密钥交换：

```
X25519MLKEM768 格式: X25519(32B) + ML-KEM-768(1184B)
                      ^^^^^^^^
                      提取前 32 字节作为 X25519 公钥
```

这确保 Reality 等伪装方案在客户端使用后量子密钥交换时仍能正常工作。

### 4.4 read_tls_record 的两种变体

**无预读版本**：直接从 transport 读取 5 字节 header，解析 record_length，再读取 record body。

**带预读版本**：利用 probe 阶段已读取的数据作为前缀，三种情况：

| 情况 | 处理 |
|------|------|
| preread < 5 字节 | 回退到无预读版本 |
| preread 包含完整记录 | 直接返回，零额外读取 |
| preread 部分记录 | 补读剩余字节 |

## 5. 特征位图（feature_bitmap.hpp）

### 5.1 位定义

将 ClientHello 特征压缩为 32 位整数，支持位运算快速匹配：

```cpp
enum feature_bit : std::uint32_t {
    // 基础特征（0-3）
    has_sni              = 1 << 0,   // 有 SNI 扩展
    sni_matched_config   = 1 << 1,   // SNI 匹配配置中的 server_names
    has_x25519           = 1 << 2,   // 有 X25519 key_share
    has_full_session_id  = 1 << 3,   // session_id_len == 32

    // 确定性标记（4-5）
    reality_marker_01_08_02 = 1 << 4,  // Reality 独占标记
    shadowtls_hmac_valid   = 1 << 5,   // ShadowTLS HMAC 有效

    // 结构特征（6-9）
    session_id_non_standard = 1 << 6,  // session_id 长度非标准
    has_ech                 = 1 << 7,  // 有 ECH 扩展
    has_esni                = 1 << 8,  // 有 ESNI 扩展（已废弃）
    greased_extensions      = 1 << 9,  // 有 GREASE 扩展

    // 扩展组合（10-13）
    has_supported_versions     = 1 << 10,
    has_alpn                   = 1 << 11,
    has_psk                    = 1 << 12,
    has_signature_algorithms   = 1 << 13,

    // 高级特征（14-15）
    key_share_multiple    = 1 << 14,  // 多个 key_share
    early_data_attempt    = 1 << 15,  // 尝试 0-RTT

    // 保留位（16-31）预留给未来扩展
};
```

### 5.2 位图构建

`build_feature_bitmap` 从 `client_hello_features` 构建位图：

```cpp
auto build_feature_bitmap(const client_hello_features &features) noexcept
    -> std::uint32_t;
```

构建逻辑：

| 条件 | 设置的位 |
|------|----------|
| `server_name` 非空 | `has_sni` |
| `has_x25519 == true` | `has_x25519` |
| `session_id_len == 32` | `has_full_session_id` |
| `session_id_len > 0 && != 32` | `session_id_non_standard` |
| `session_id[0:3] == [0x01, 0x08, 0x02]` | `reality_marker_01_08_02` |
| `versions` 非空 | `has_supported_versions` |

注意：`sni_matched_config` 位需要在路由阶段由上层设置，`shadowtls_hmac_valid` 需要 Layer 1 验证。

### 5.3 匹配工具

```cpp
// 检查单个特征
auto has_feature(uint32_t bitmap, feature_bit bit) -> bool;

// 检查所有指定特征
auto has_all_features(uint32_t bitmap, uint32_t bits) -> bool;
```

使用示例：

```cpp
auto bitmap = build_feature_bitmap(features);

// 快速检测 Reality 标记
if (has_feature(bitmap, reality_marker_01_08_02)) { ... }

// 组合检测：有 SNI + 有 X25519 + session_id 标准
constexpr auto mask = has_sni | has_x25519 | has_full_session_id;
if (has_all_features(bitmap, mask)) { ... }
```

## 6. 与 Stealth 模块的关系

TLS 模块是 recognition 和 stealth 之间的**共享基础设施**：

```
recognition 模块                     stealth 模块
  │                                     │
  ├── probe 阶段                        ├── Reality 方案
  │   读取 ClientHello                   │   使用 x25519_key 做密钥交换
  │                                     │   使用 raw_record 重放
  ├── identify 阶段                     │   使用 session_id 检测标记
  │   build_feature_bitmap()            │
  │   各方案 detect(bitmap)             ├── ShadowTLS 方案
  │                                     │   验证 session_id 中的 HMAC
  └── 路由阶段                          │
      sni_matched_config 位设置         └── 其他方案...
```

Reality 独占标记 `session_id[0:3] == [0x01, 0x08, 0x02]` 是分层检测管道的关键：
- **Tier 0**（位图）：仅需一次位运算 `has_feature(bitmap, reality_marker_01_08_02)`
- **Tier 1**（密码学验证）：使用 x25519_key 做实际的密钥交换验证

## 7. 设计原理

- **中立共享层**：types 和 signal 不依赖 recognition 或 stealth，两者通过 using 声明引用
- **无状态纯函数**：parse_client_hello 不持有上下文，线程安全、可测试
- **零拷贝友好**：特征结构使用 PMR 容器，与帧竞技场兼容
- **位图优化**：32 位位图压缩特征，Tier 0 检测 O(1) 复杂度
- **后量子兼容**：X25519MLKEM768 混合密钥的自动降级提取

## 相关页面

- [[protocol]] — 协议模块概览
- [[protocol/common]] — 协议公共组件
- [[stealth]] — TLS 伪装模块
- [[recognition]] — 协议识别模块
