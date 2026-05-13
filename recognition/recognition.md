---
title: Recognition 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [recognition, protocol-detection, overview, design]
related: [[protocol], [agent], [stealth], [channel/transport], [memory/overview], [fault/overview], [config]]
---

# Recognition 模块

Recognition 模块是 Prism 的**协议智能识别层**，负责在连接建立初期识别客户端使用的协议类型和伪装方案。该模块采用两阶段流水线：外层探测（Probe）快速判断协议类型，对 TLS 连接进一步执行 ClientHello 特征分析和伪装方案识别。

## 概述

### 核心职责

- **外层协议探测**：预读 24 字节，快速判断 HTTP/SOCKS5/TLS/Shadowsocks
- **TLS 伪装方案识别**：解析 ClientHello，通过分层检测匹配伪装方案
- **统一入口**：`recognize()` 封装完整的识别生命周期
- **SNI 路由**：根据 ClientHello SNI 快速路由到对应伪装方案

### 模块结构

```
include/prism/recognition/
├── recognition.hpp           # 统一入口（recognize/identify）
├── result.hpp                # 分析结果结构（analysis_result）
├── confidence.hpp            # 置信度枚举（high/medium/low/none）
├── scheme_route_table.hpp    # SNI 路由表
├── layered_pipeline.hpp      # 分层检测管道
└── probe/
    ├── probe.hpp             # 外层协议探测协程
    └── analyzer.hpp          # 魔术字节检测（纯内存操作）
```

### 识别流程

```
recognition::recognize(ctx)
    │
    ├── Phase 1: Probe（外层探测）
    │   ├── 预读 24 字节
    │   ├── detect() → SOCKS5/TLS/HTTP/Shadowsocks
    │   └── 非 TLS → 直接返回识别结果
    │
    └── Phase 2: Identify（仅 TLS）
        ├── Read: 读取完整 ClientHello
        ├── Parse: 提取 SNI/session_id/key_share
        ├── Detect: 分层检测（Tier 0/1/2）
        └── Execute: 按候选顺序执行伪装方案
```

#### Phase 1: Probe

预读 24 字节，基于魔术字节快速判断：
- `0x05` → SOCKS5
- `0x16 0x03` → TLS（检查两字节避免 SS2022 salt 误判）
- `GET/POST/CONNECT` → HTTP
- 其他 → Shadowsocks（排除法 fallback，无正特征）

#### Phase 2: Identify（仅 TLS）

对 TLS 连接执行四步识别：
1. **Read** — 读取完整 ClientHello
2. **Parse** — 提取 SNI、session_id、key_share、versions
3. **Detect** — 分层检测：Tier 0 零成本字节比较 → Tier 1 HMAC 验证 → Tier 2 模糊匹配
4. **Execute** — 按候选顺序执行伪装方案，失败则 fallback 到 native TLS

### 关键组件

| 组件 | 文件 | 功能 |
|------|------|------|
| `recognize()` | `recognition.hpp` | 统一入口，封装完整流程 |
| `probe()` | `probe/probe.hpp` | 预读 + 检测协议类型 |
| `detect()` | `probe/analyzer.hpp` | 纯内存魔术字节检测 |
| `identify()` | `recognition.hpp` | TLS 伪装方案识别 |
| `layered_detection_pipeline` | `layered_pipeline.hpp` | 分层检测编排 |
| `scheme_route_table` | `scheme_route_table.hpp` | SNI → 方案快速映射 |
| `confidence` | `confidence.hpp` | 检测置信度级别 |

## 详细设计

### 模块职责与边界

#### 职责

- **外层协议探测**：通过预读数据快速判断 HTTP/SOCKS5/TLS/Shadowsocks
- **TLS ClientHello 特征分析**：提取 SNI、session_id、key_share 等特征
- **TLS 伪装方案识别**：分层检测（Tier 0/1/2）匹配伪装方案
- **SNI 路由**：根据 ClientHello SNI 快速路由到对应伪装方案
- **统一入口**：`recognize()` 封装完整的识别生命周期

#### 边界

- **不负责**：协议编解码（由 protocol 模块处理）
- **不负责**：连接建立和数据转发（由 pipeline 模块处理）
- **不负责**：TLS 伪装方案实现（由 stealth 模块提供 detect/verify/guess 接口）
- **不负责**：会话生命周期管理（由 agent/session 处理）

### 关键类和接口

#### `recognize_context` / `recognize_result`

**文件**: `recognition.hpp`

```cpp
struct recognize_context {
    channel::transport::shared_transmission transport;
    const psm::config *cfg{nullptr};
    resolve::router *router{nullptr};
    agent::session_context *session{nullptr};
    memory::frame_arena *frame_arena{nullptr};
};

struct recognize_result {
    channel::transport::shared_transmission transport;
    protocol::protocol_type detected{protocol::protocol_type::unknown};
    memory::vector<std::byte> preread;
    fault::code error{fault::code::success};
    memory::string executed_scheme;
    bool success{false};
};
```

#### `identify_context` / `identify_result`

**文件**: `recognition.hpp`

```cpp
struct identify_context {
    channel::transport::shared_transmission transport;
    const psm::config *cfg{nullptr};
    std::span<const std::byte> preread;
    resolve::router *router{nullptr};
    agent::session_context *session{nullptr};
    memory::frame_arena *frame_arena{nullptr};
};

struct identify_result {
    channel::transport::shared_transmission transport;
    protocol::protocol_type detected{protocol::protocol_type::unknown};
    memory::vector<std::byte> preread;
    fault::code error{fault::code::success};
    memory::string executed_scheme;
    bool success{false};
};
```

#### `probe::probe()` — 外层协议探测

**文件**: `probe/probe.hpp`

```cpp
struct probe_result {
    protocol::protocol_type type{protocol::protocol_type::unknown};
    std::array<std::byte, 32> pre_read_data{};
    std::size_t pre_read_size{0};
    fault::code ec{fault::code::success};

    [[nodiscard]] auto success() const noexcept -> bool;
    [[nodiscard]] auto preload_view() const noexcept -> std::string_view;
    [[nodiscard]] auto preload_bytes() const noexcept -> std::span<const std::byte>;
};

inline auto probe(channel::transport::transmission &transport, const std::size_t max_peek_size = 24)
    -> net::awaitable<probe_result>;
```

预读最多 24 字节，调用 `detect()` 检测协议类型。预读数据缓冲区最大 32 字节。

#### `probe::detect()` — 魔术字节检测

**文件**: `probe/analyzer.hpp`

```cpp
[[nodiscard]] auto detect(std::string_view peek_data) -> protocol::protocol_type;
```

纯内存操作，无状态，线程安全。检测顺序：SOCKS5（首字节 0x05）→ TLS（前两字节 0x16 0x03）→ HTTP 方法名 → Shadowsocks（排除法 fallback）。

#### `layered_detection_pipeline` — 分层检测管道

**文件**: `layered_pipeline.hpp`

```cpp
struct candidate_entry {
    memory::string name;
    std::uint16_t score{0};
    std::uint8_t tier{2};
    bool is_deterministic{false};
};

struct pipeline_result {
    bool deterministic_hit{false};
    memory::string exclusive_scheme;
    memory::vector<candidate_entry> candidates;
    memory::string reason;
};

class layered_detection_pipeline {
public:
    explicit layered_detection_pipeline(const std::vector<stealth::shared_scheme> &schemes);

    [[nodiscard]] auto detect(
        std::uint32_t bitmap,
        const protocol::tls::client_hello_features &features,
        std::span<const std::byte> raw,
        const psm::config &cfg,
        const std::vector<stealth::shared_scheme> &matched_schemes) const -> pipeline_result;

private:
    std::vector<stealth::shared_scheme> tier0_schemes_;
    std::vector<stealth::shared_scheme> tier1_schemes_;
    std::vector<stealth::shared_scheme> tier2_schemes_;
    stealth::shared_scheme native_scheme_;
};
```

**分层策略**：
- **Tier 0**：调用 `scheme->sniff(bitmap, features)`，零成本位图比较，独占命中直接返回
- **Tier 1**：调用 `scheme->verify(features, raw, cfg)`，有成本 HMAC/解密验证
- **Tier 2**：调用 `scheme->guess(cfg)`，基于配置的模糊匹配，返回多候选列表

构造时从 stealth 注册表按方案类型分配到对应 tier。执行顺序：Tier 0 → 确定性跳过后续 → Tier 1 → 确定性跳过 → Tier 2 + native 兜底。

#### `scheme_route_table` — SNI 路由表

**文件**: `scheme_route_table.hpp`

```cpp
class scheme_route_table {
public:
    static auto build(const psm::config &cfg) -> scheme_route_table;

    [[nodiscard]] auto lookup(std::string_view sni) const -> memory::vector<memory::string>;
    [[nodiscard]] auto matches_any(std::string_view sni) const -> bool;
    [[nodiscard]] auto all_registered_snis() const -> memory::vector<memory::string>;
    [[nodiscard]] auto empty() const noexcept -> bool;

private:
    memory::map<memory::string, memory::vector<memory::string>> route_map_;
};
```

启动时从各伪装方案的 server_names 配置构建。同一 SNI 可配置在多个方案中，返回多个候选。

#### `analysis_result` / `confidence`

**文件**: `result.hpp`, `confidence.hpp`

```cpp
struct analysis_result {
    memory::vector<memory::string> candidates;
    confidence score{confidence::none};
    protocol::tls::client_hello_features features;
    fault::code error{fault::code::success};
};

enum class confidence : std::uint8_t {
    high,    // 特征完全匹配，可直接执行
    medium,  // 特征部分匹配，需完整验证
    low,     // 特征部分匹配但不确定
    none     // 无特征匹配，执行 native 兜底
};
```

## 模块依赖

| 方向 | 模块 | 用途 |
|------|------|------|
| 依赖 | `protocol` | `protocol_type`、`analysis`、`tls::client_hello_features`、`tls::feature_bitmap` |
| 依赖 | `stealth` | `shared_scheme`（sniff/verify/guess 接口） |
| 依赖 | `channel/transport` | `shared_transmission` 传输层抽象 |
| 依赖 | `memory` | PMR 容器和内存池 |
| 依赖 | `fault` | 错误码定义 |
| 被依赖 | `agent/session` | 调用 `recognize()` 进行协议识别 |

## 设计原理

- **分层检测优化**：不同检测操作成本差异巨大——Tier 0（位图比较）纳秒级，Tier 1（HMAC 验证）微秒级，Tier 2（模糊匹配）毫秒级。按成本分层执行，确定性命中时提前返回
- **独占特征优化**：多个方案可能同时匹配。Reality session_id `[0x01, 0x08, 0x02]` 是独占特征，独占命中时直接执行，跳过多候选流程
- **SNI 路由预过滤**：全量方案扫描成本高。启动时从配置构建 SNI → 方案映射，Tier 2 检测时仅执行 SNI 匹配的方案，未匹配时才执行全量扫描
- **Native 兜底策略**：始终添加 native 方案作为兜底，确保任何 TLS 连接都能被处理，避免识别失败导致连接断开
- **预读数据管理**：协议嗅探消耗了部分数据。preview 包装器将预读数据保存在内部缓冲区，后续读取时优先返回预读数据，对上层透明

## 相关链接

- [[protocol]] — Protocol 模块
- [[stealth]] — Stealth 模块
- [[agent]] — Agent 模块
- [[memory]] — Memory 模块
