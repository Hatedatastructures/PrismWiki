# PrismWiki 知识库深度改造计划

> **编制日期**: 2026/05/23
> **背景**: 基于 6 维度全项目审计（代码完整性、测试覆盖、生产就绪、安全协议、架构质量、路线图完成度）+ 393 个知识库文件逐文件深度审计
> **核心问题**: 知识库整体 8.5/10 但存在结构性缺口、协议文档深度不足、命名与源码脱节

---

## 一、问题总览

### 1.1 结构性缺口

| 缺口 | 影响 | 严重度 |
|------|------|--------|
| `core/stats/` 模块完全空白（7 个 hpp 无对应页面） | 运行时统计、流量计数、worker 负载无文档 | P0 |
| `core/protocol/` 下 conn/process/packet/framing 约 25 个 hpp 无对应页面 | 协议**实现层**核心代码无文档 | P0 |
| 知识库用旧模块名（agent/channel/pipeline）但源码已改为（instance/connect+transport/protocol） | 开发者按知识库找不到源码文件 | P0 |

### 1.2 协议文档深度不足

| 缺失 | 涉及协议 | 说明 |
|------|---------|------|
| 线缆格式字节级全景图 | SS2022（最严重）、Trojan、VLESS | 只有函数级描述，无整体字节布局 |
| 正式状态机转换图 | 全部协议 | 有流程步骤但无状态节点+事件+转换条件 |
| 完整请求/响应时序图 | Trojan、VLESS | 无 Client→Prism→Target 完整时序 |
| 协议级错误码表 | Trojan、VLESS、SS2022 | 仅有 fault::code 系统级引用 |
| 攻击面总结 | 全部协议 | 各协议的威胁模型未集中描述 |
| 配置追踪链 | 全部协议 | 缺少 JSON→config 结构体→构造函数→行为 |
| 故障排查场景 | SOCKS5、HTTP、Trojan、VLESS | relay.md 缺少排障步骤 |

### 1.3 浅文件清单（需重写/深度扩展）

以下是字节数 < 5000 的所有文件，按严重度排序：

#### P0 — 严重不足（< 2000 B，几乎占位符）

| # | 文件 | 字节 | 问题 |
|---|------|------|------|
| 1 | `ref/protocol/http-1-1.md` | 691 | 全库最小协议文件，仅 3 段话 |
| 2 | `ref/logging/spdlog.md` | 654 | 全库最小文件，几乎空白 |
| 3 | `ref/programming/cpp-23.md` | 1176 | C++23 特性仅列标题 |
| 4 | `core/loader/overview.md` | 1303 | 配置加载核心流程太短 |
| 5 | `core/transformer/overview.md` | 1423 | JSON 序列化模块太短 |
| 6 | `core/protocol/common/form.md` | 1419 | 仅列出枚举值 |
| 7 | `core/protocol/common/read.md` | 1891 | 共享 I/O 工具太短 |
| 8 | `core/protocol/common/address.md` | 1896 | 地址类型太短 |
| 9 | `core/protocol/trojan/constants.md` | 1858 | 常量枚举缺使用场景 |
| 10 | `core/protocol/trojan/config.md` | 1768 | 配置缺映射追踪 |
| 11 | `core/protocol/vless/constants.md` | 1942 | 缺跨协议对比扩展 |
| 12 | `core/protocol/vless/config.md` | 1826 | 配置缺映射追踪 |
| 13 | `core/protocol/overview.md` | 1752 | 协议层总览太短，无架构图 |
| 14 | `core/stealth/reality/config.md` | 2010 | 配置缺映射追踪 |
| 15 | `docs/overview.md` | 999 | 用户入口仅 3 段话 |

#### P1 — 明显偏薄（2000-3000 B）

| # | 文件 | 字节 | 问题 |
|---|------|------|------|
| 16 | `core/protocol/shadowsocks/config.md` | 2093 | 缺 PSK 长度→密码套件映射细节 |
| 17 | `core/protocol/socks5/constants.md` | 2126 | 缺 reply_code 的触发场景 |
| 18 | `core/protocol/socks5/config.md` | 2189 | 缺配置验证逻辑 |
| 19 | `core/stealth/reality/request.md` | 2851 | 缺 ClientHello 字节级完整结构 |
| 20 | `core/stealth/reality/response.md` | 3679 | 缺 ServerHello 字节级完整结构 |
| 21 | `core/stealth/reality/auth.md` | 2878 | 5 步认证缺错误场景 |
| 22 | `core/stealth/reality/seal.md` | 2748 | 缺加密记录字节级结构 |
| 23 | `core/stealth/reality/scheme.md` | 2874 | 缺 Tier 0 判定细节 |
| 24 | `core/stealth/reality/config.md` | 2010 | 缺密钥格式说明 |
| 25 | `core/protocol/shadowsocks/format.md` | 2439 | **严重缺少 AEAD 帧结构** |
| 26 | `core/protocol/tls/signal.md` | 2502 | ClientHello 解析缺字节级字段 |
| 27 | `core/protocol/shadowsocks/replay.md` | 2473 | 滑动窗口算法缺图解 |
| 28 | `core/protocol/shadowsocks/salts.md` | 2590 | Salt 池缺容量和淘汰策略 |
| 29 | `core/protocol/trojan/format.md` | 4361 | 缺请求头完整字节级图解 |
| 30 | `core/stealth/anytls/scheme.md` | 4696 | 空壳方案，缺握手细节 |
| 31 | `core/stealth/restls/scheme.md` | 5215 | 空壳方案，缺 Restls Script 语义 |
| 32 | `core/stealth/trusttunnel/scheme.md` | 4258 | 空壳方案 |
| 33 | `docs/getting-started.md` | 4183 | 快速开始太短 |

#### P2 — 可提升（3000-5000 B）

| # | 文件 | 字节 | 问题 |
|---|------|------|------|
| 34 | `core/protocol/shadowsocks/relay.md` | 4983 | 最详细的 relay 但仍缺 AEAD 帧结构 |
| 35 | `core/protocol/socks5/wire.md` | 4005 | 缺边界条件 |
| 36 | `core/protocol/socks5/stream.md` | 4979 | 缺认证失败场景 |
| 37 | `core/protocol/http/parser.md` | 3165 | 缺分块传输 |
| 38 | `core/protocol/http/relay.md` | 2801 | 缺 chunked/keep-alive |
| 39 | `core/protocol/vless/format.md` | 3248 | 缺响应格式字节图 |
| 40 | `core/protocol/vless/relay.md` | 3514 | 缺 mux 处理细节 |
| 41 | `core/protocol/trojan/relay.md` | 3739 | 缺 mux 处理细节 |
| 42 | `core/protocol/tls/feature_bitmap.md` | 3952 | 缺 Reality 检测实战 |
| 43 | `core/protocol/tls/types.md` | 4076 | 缺 CipherSuite 安全分析 |
| 44 | `core/stealth/shadowtls/constants.md` | 3131 | 缺帧格式字节图 |
| 45 | `core/stealth/shadowtls/config.md` | 3329 | 缺 v2/v3 行为差异 |
| 46 | `core/stealth/shadowtls/scheme.md` | 3280 | 缺 Tier 1 判定算法 |
| 47 | `core/stealth/registry.md` | 4199 | 缺注册顺序对检测的影响 |
| 48 | `core/stealth/native.md` | 4318 | 缺 fallback 判定细节 |
| 49 | `core/stealth/ech/config.md` | 4781 | 缺 ECH 密钥生成步骤 |
| 50 | `core/stealth/ech/decrypt.md` | 5466 | 缺 HPKE 完整流程 |
| 51 | `core/stealth/executor.md` | 6880 | 缺 rewind 失败场景 |
| 52 | `dev/performance/overview.md` | 2162 | 性能优化概述太短 |
| 53 | `dev/roadmap.md` | 6216 | 可更详细 |
| 54 | `ref/protocol/overview.md` | 6398 | 缺错误码索引和配置映射 |

---

## 二、Phase 1（第一批）: 补结构性缺口 + 协议实现层

> **目标**: 补全完全空白的模块，补全协议实现层的 25 个缺失页面，统一命名。

### 任务 1.1: 创建 `core/stats/` 模块文档（新建 6 个文件）

`include/prism/stats/` 有 7 个头文件但在知识库中完全空白。

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/stats/overview.md` | `stats/stats.hpp` | 模块总览：stats 的角色（运行时可观测性）、与其他模块的关系、架构图 |
| `core/stats/counter.md` | `stats/counter.hpp` | 原子计数器原语：`counter<T>` 模板、`increment`/`decrement`/`reset`/`load`、内存序选择、热路径开销分析 |
| `core/stats/gauge.md` | `stats/gauge.hpp` | 原子仪表原语：`gauge<T>` 模板、`set`/`get`/`update`、与 counter 区别、适用场景（活跃连接数、内存水位） |
| `core/stats/snapshot.md` | `stats/snapshot.hpp` | 统计快照结构体：`worker_load_snapshot`（active_sessions/pending_handoffs/event_loop_lag_us）、`runtime_snapshot`（全局运行状态）、`traffic_snapshot`（流量统计）、EMA 平滑算法详解 |
| `core/stats/runtime.md` | `stats/runtime.hpp` | Worker 负载统计：per-worker 统计收集、全局聚合、事件循环延迟测量（预热阶段/抖动基线校准/噪声过滤）、与 balancer 的集成方式 |
| `core/stats/traffic.md` | `stats/traffic.hpp` + `account.hpp` | 流量统计：per-worker 上行/下行字节统计、全局流量聚合、per-account 流量观察者、与 `account::entry` 的 `uplink_bytes`/`downlink_bytes` 集成 |

每个文件必须包含：
- 命名空间和源码路径
- 所有公共类/结构体的完整字段说明
- 调用链（谁调用、被谁调用）
- 线程安全分析（原子操作 vs 需要同步）
- 与其他模块的依赖关系图
- 代码示例（如何使用 counter/gauge）

---

### 任务 1.2: 补全 `core/protocol/` 实现层文档（新建 ~25 个文件）

这是最大的缺口。当前知识库覆盖了各协议的 config/constants/format/wire/relay，但缺少实际的连接处理（conn）、数据帧处理（framing）、数据包处理（packet）、协议入口（process）。

#### 1.2.1 公共模块

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/common/framing.md` | `protocol/common/framing.hpp` | 通用帧处理：帧边界检测、长度前缀编解码、与各协议 framing 的关系 |
| `core/protocol/common/mux.md` | `protocol/common/mux.hpp` | 多路复用公共接口：mux 检测逻辑、smux/yamux 的统一抽象、bootstrap 流程 |
| `core/protocol/common/target.md` | `protocol/common/target.hpp` | 目标地址结构体：host/port/address/form、从各协议地址格式到统一 target 的转换 |
| `core/protocol/common/udp-relay.md` | `protocol/common/udp_relay.hpp` | UDP 中继公共逻辑：`udp_over_tls_frame_loop` 模板、各协议如何复用此模板（Trojan/VLESS/SOCKS5） |
| `core/protocol/protocol-type.md` | `protocol/protocol_type.hpp` | 协议类型枚举：所有值、检测判定规则、从 probe 到 dispatch 的映射 |

#### 1.2.2 HTTP 协议

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/http/conn.md` | `protocol/http/conn.hpp` | HTTP 代理连接处理：请求行解析、CONNECT 隧道建立、普通 HTTP 转发、407/403/502 响应构造 |
| `core/protocol/http/process.md` | `protocol/http/process.hpp` | HTTP 协议入口：`handle()` 函数完整流程、与其他协议 handler 的对比 |

#### 1.2.3 SOCKS5 协议

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/socks5/conn.md` | `protocol/socks5/conn.hpp` | SOCKS5 连接处理：方法协商、认证、CONNECT 命令、UDP ASSOCIATE 完整流程、状态机图 |
| `core/protocol/socks5/framing.md` | `protocol/socks5/framing.hpp` | SOCKS5 帧编解码：TCP 帧边界、UDP 数据报封装/解封、FRAG 处理 |
| `core/protocol/socks5/packet.md` | `protocol/socks5/packet.hpp` | SOCKS5 数据包结构 |
| `core/protocol/socks5/process.md` | `protocol/socks5/process.hpp` | SOCKS5 协议入口：`handle()` 函数 |

#### 1.2.4 Trojan 协议

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/trojan/conn.md` | `protocol/trojan/conn.hpp` | Trojan 连接处理：56B 凭证验证、CMD 分发（CONNECT/UDP/MUX）、mux 引导 |
| `core/protocol/trojan/framing.md` | `protocol/trojan/framing.hpp` | Trojan 帧编解码：TCP 流帧边界、UDP over TLS 帧格式 |
| `core/protocol/trojan/packet.md` | `protocol/trojan/packet.hpp` | Trojan 数据包结构 |
| `core/protocol/trojan/process.md` | `protocol/trojan/process.hpp` | Trojan 协议入口 |

#### 1.2.5 VLESS 协议

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/vless/conn.md` | `protocol/vless/conn.hpp` | VLESS 连接处理：UUID 验证、Version/Command/AddnlInfo 解析、mux 引导 |
| `core/protocol/vless/framing.md` | `protocol/vless/framing.hpp` | VLESS 帧编解码 |
| `core/protocol/vless/packet.md` | `protocol/vless/packet.hpp` | VLESS 数据包结构 |
| `core/protocol/vless/process.md` | `protocol/vless/process.hpp` | VLESS 协议入口 |

#### 1.2.6 Shadowsocks 2022 协议

| 新建文件 | 对应源码 | 内容要求 |
|---------|---------|---------|
| `core/protocol/shadowsocks/conn.md` | `protocol/shadowsocks/conn.hpp` | SS2022 连接处理：AEAD 流加密、salt 交换、固定头/变长头加密、时间戳验证 |
| `core/protocol/shadowsocks/framing.md` | `protocol/shadowsocks/framing.hpp` | SS2022 帧编解码：AEAD 分帧加密的具体字节级结构 |
| `core/protocol/shadowsocks/packet.md` | `protocol/shadowsocks/packet.hpp` | SS2022 数据包结构 |
| `core/protocol/shadowsocks/process.md` | `protocol/shadowsocks/process.hpp` | SS2022 协议入口 |
| `core/protocol/shadowsocks/datagram.md` | `protocol/shadowsocks/util/datagram.hpp` | SS2022 UDP 数据报：Separate Header、SessionID/PacketID、AES-ECB 加密头部 |
| `core/protocol/shadowsocks/tracker.md` | `protocol/shadowsocks/util/tracker.hpp` | SS2022 UDP 会话跟踪器：per-SessionID 管理、超时清理、资源上限 |
| `core/protocol/shadowsocks/salts-detail.md` | `protocol/shadowsocks/util/salts.hpp` | Salt 重放保护池的深度实现：thread_local、FNV-1a、分摊清理策略 |

**每个 conn.md 必须包含**:
- 完整的握手字节级时序图（Client→Prism→Target）
- 状态机转换图（状态节点 + 事件 + 转换条件 + 错误分支）
- 线缆格式字节级全景图
- 错误码与错误处理场景表
- 攻击面分析（该协议的威胁模型）
- 与其他实现的兼容性说明
- 源码函数名 + 文件路径
- 故障排查场景

**每个 framing.md 必须包含**:
- 帧的完整字节级结构图（每个字段的偏移量、长度、含义）
- 编码/解码算法伪代码
- 边界条件（最大长度、最小长度、截断处理）
- 与 smux/yamux 帧的嵌套关系

**每个 process.md 必须包含**:
- `handle()` 函数的完整流程图
- 与 dispatcher 的注册关系
- 与其他协议 handler 的对比表
- 配置追踪链（JSON → config → handle() → 具体行为）

---

### 任务 1.3: 统一知识库模块命名

将知识库目录结构对齐到源码的实际模块名。

**需要迁移的目录**:

| 当前知识库路径 | 新路径 | 说明 |
|--------------|--------|------|
| `core/agent/` | `core/instance/` | agent 已重命名为 instance |
| `core/channel/` | 拆分为 `core/connect/` + `core/transport/` | channel 已拆分 |
| `core/channel/transport/` | `core/transport/` | 传输层已独立 |
| `core/channel/connection/` | `core/connect/pool/` | 连接池已移至 connect |
| `core/channel/eyeball/` | `core/connect/dial/` | Happy Eyeballs 已移至 connect/dial |
| `core/pipeline/` | 删除或重定向 | pipeline 已拆散 |

**操作步骤**:
1. 在知识库中创建新目录结构
2. 移动文件到新位置（保留旧路径的 redirect 文件或删除）
3. 更新所有 wikilink 引用
4. 更新 `index.md` 总索引
5. 更新 `core/overview.md` 架构图

---

## 三、Phase 2（第二批）: 协议文档深度扩展

> **目标**: 将协议相关文档从"函数级描述"提升到"专业协议规范"级别。
> **标准**: 真实的协议规范文档应该包含线缆格式的每个字节、状态机的每个转换、错误码的每个场景。

### 任务 2.1: SS2022 协议深度扩展（最高优先级）

SS2022 是 Prism 最重要的自加密协议，当前文档严重缺少 AEAD 帧的字节级结构。

#### 2.1.1 重写 `core/protocol/shadowsocks/format.md`（当前 2439 B → 目标 15000+ B）

**必须新增的内容**:

1. **AEAD 加密帧的完整字节级结构**（这是最关键的缺失）:
   ```
   TCP 流格式:
   ┌────────────┬──────────────────────────┬─────────────────────┐
   │  Salt       │  Encrypted Fixed Header  │  AEAD Data Chunks   │
   │  (32/16 B)  │  (加密的固定头)           │  (加密数据块流)      │
   └────────────┴──────────────────────────┴─────────────────────┘

   Fixed Header (加密前):
   ┌──────────────────┬──────────────────┬──────────────┐
   │  ATYP (1 B)       │  地址 (var)       │  端口 (2 B)   │
   └──────────────────┴──────────────────┴──────────────┘

   AEAD Data Chunk:
   ┌─────────────────────────────┬─────────────────────────┐
   │  Encrypted Length Block      │  Encrypted Payload Block │
   │  [2B 加密长度 + 16B tag]     │  [N 加密数据 + 16B tag]  │
   └─────────────────────────────┴─────────────────────────┘

   每个 AEAD 块的 Nonce 构造:
   ┌──────────────────────────────┐
   │ 固定前缀 (4B, 全零) + 计数器 (8B, 大端递增) │
   └──────────────────────────────┘
   ```

2. **密钥派生完整流程**:
   - PSK → Base64 解码 → 原始密钥（16B→AES-128-GCM, 32B→AES-256-GCM/ChaCha20）
   - BLAKE3 KDF：`subkey = blake3_derive_key(psk, salt, "shadowsocks 2022 session subkey")`
   - Salt 的作用和来源

3. **三种加密方法的参数对比表**:
   | 方法 | 密钥长度 | Salt 长度 | Tag 长度 | 块大小 |
   |------|---------|----------|---------|--------|
   | 2022-blake3-aes-128-gcm | 16 B | 16 B | 16 B | ? |
   | 2022-blake3-aes-256-gcm | 32 B | 32 B | 16 B | ? |
   | 2022-blake3-chacha20-poly1305 | 32 B | 32 B | 16 B | ? |

4. **时间戳验证算法**:
   - 固定头中时间戳字段的提取
   - 验证窗口计算
   - 时钟偏移容忍度

5. **完整请求/响应时序图**（Client→Prism→Target）:
   ```
   Client                    Prism                      Target
     │                        │                           │
     │── TCP Connect ────────→│                           │
     │                        │                           │
     │── Salt + FixedHeader ─→│                           │
     │   (AEAD 加密)          │── 验证时间戳 ──→          │
     │                        │── 检查 salt 重放 ──→     │
     │                        │── KDF 派生密钥 ──→       │
     │                        │── 解密固定头 ──→         │
     │                        │── 解析目标地址 ──→       │
     │                        │                           │
     │                        │──── TCP Connect ─────────→│
     │                        │                           │
     │── Data Chunk 1 ───────→│── 解密 → 转发 ──────────→│
     │←─ Data Chunk 1 ───────│←── 接收 → 加密 ──────────│
     │   ...                  │   ...                     │
   ```

6. **UDP Separate Header 字节级结构**:
   ```
   Type 0x00 (明文):
   ┌─────────────┬─────────────┬──────────────┬──────────────┐
   │  SessionID   │  PacketID    │  Header Pad   │  Payload     │
   │  (8 B)       │  (8 B)       │  (var)        │  (var)       │
   └─────────────┴─────────────┴──────────────┴──────────────┘

   Type 0x01 (AES-ECB 加密):
   ┌────────────────────────────────┬──────────────┐
   │  AES-ECB(SessionID+PacketID)   │  AEAD(Payload)│
   │  (16 B)                        │  (var)        │
   └────────────────────────────────┴──────────────┘
   ```

7. **PacketID 滑动窗口重放检测算法图解**:
   - 64 位窗口的位图表示
   - 判定逻辑（within window / below window / duplicate）
   - 与 WireGuard 算法的对比

8. **SIP022 规范符合性声明**:
   - 已实现的功能清单（打 ✅）
   - 未实现的功能清单（EIH、多 PSK）
   - 与 shadowsocks-rust/go-shadowsocks2 的兼容性

#### 2.1.2 扩展 `core/protocol/shadowsocks/relay.md`（当前 4983 B → 目标 12000+ B）

**必须新增**:
- AEAD 读取状态机的完整状态转换图（header → 解密2B长度 → payload → 下一个 chunk）
- Scatter-gather 写入优化详解（`writev` 等效，减少系统调用）
- 乐观响应（optimistic response）设计：为什么不等密钥派生完成就发回第一个 chunk
- 配置追踪链：`psk` → `decode_psk()` → `resolve_cipher_method()` → `aes-256-gcm key` → `KDF` → `session key`
- 完整的错误场景表：每个 fault::code 在什么条件下触发

#### 2.1.3 扩展 `core/protocol/shadowsocks/replay.md`（当前 2473 B → 目标 8000+ B）

**必须新增**:
- 滑动窗口算法的逐步图解（含 8 个示例：normal、duplicate、out-of-order、wrap-around）
- 与 TCP 序列号滑动窗口的对比
- 与 WireGuard `replay_filter` 的实现差异
- 性能分析：O(1) 位图操作的时间复杂度证明

#### 2.1.4 扩展 `core/protocol/shadowsocks/salts.md`（当前 2590 B → 目标 8000+ B）

**必须新增**:
- Salt 池的内存布局和容量管理
- FNV-1a 哈希选择的原因（vs std::hash、vs siphash）
- 分摊清理策略的触发条件和算法
- thread_local 隔离的线程安全保证
- Salt 碰撞概率分析

---

### 任务 2.2: Trojan 协议深度扩展

#### 2.2.1 重写 `core/protocol/trojan/format.md`（当前 4361 B → 目标 15000+ B）

**必须新增**:

1. **Trojan 请求头完整字节级结构**:
   ```
   Trojan TCP 请求:
   ┌──────────────┬───────┬──────┬──────┬──────────┬──────┬───────┐
   │  CREDENTIAL   │ CRLF  │ CMD  │ ATYP │  地址     │ PORT │ CRLF  │
   │  (56 B hex)   │(2 B)  │(1 B) │(1 B) │  (var)   │(2 B) │(2 B)  │
   └──────────────┴───────┴──────┴──────┴──────────┴──────┴───────┘

   CMD 值:
   ┌────────┬──────────────────────┐
   │ 0x01    │ Connect (TCP 代理)    │
   │ 0x03    │ UDP Associate        │
   │ 0x7F    │ Mux (多路复用)        │
   └────────┴──────────────────────┘

   ATYP + 地址 + 端口:
   ┌─────────┬──────────────┬───────────┐
   │ 0x01     │ IPv4 (4 B)    │ Port (2B) │
   │ 0x03     │ 域名 (1+len)  │ Port (2B) │
   │ 0x04     │ IPv6 (16 B)   │ Port (2B) │
   └─────────┴──────────────┴───────────┘
   ```

2. **CREDENTIAL 生成算法**:
   ```
   password (明文)
     → SHA-224(password)
     → 56 字节十六进制字符串 (小写)
     → 作为 CREDENTIAL 字段发送
   ```
   包含代码示例：`echo -n "mypassword" | openssl dgst -sha224`

3. **UDP over TLS 帧格式**:
   ```
   ┌───────────┬──────────┬──────┬──────────┬──────┐
   │  ATYP      │  地址     │ PORT │  Length   │ CRLF │
   │  (1 B)     │  (var)   │(2 B) │  (2 B)    │(2 B) │
   └───────────┴──────────┴──────┴──────────┴──────┘
   ┌───────────────────────────┐
   │  Payload (Length 字节)     │
   └───────────────────────────┘
   ```

4. **Mux 命令 (0x7F) 的处理流程**:
   - 检测到 0x7F 后的 mux 引导
   - smux/yamux 的接入点
   - 虚假地址过滤（mihomo 兼容性）

5. **完整请求/响应时序图**

6. **安全分析**:
   - Trojan 本身无加密，完全依赖外层 TLS
   - 56B SHA-224 凭证的熵分析
   - 凭证在 TLS 明文后可见的风险
   - 中间人攻击下的安全性
   - 与 SS2022 自加密方案的对比

7. **兼容性说明**:
   - 与 mihomo 的兼容（虚假地址、UDP 包格式）
   - 与 trojan-go 的不兼容点（自定义加密、WebSocket）
   - 与 sing-box 的兼容

#### 2.2.2 扩展 `core/protocol/trojan/relay.md`（当前 3739 B → 目标 10000+ B）

**必须新增**: 完整状态机、mux 处理细节、UDP over TLS 完整流程、故障排查场景。

---

### 任务 2.3: VLESS 协议深度扩展

#### 2.3.1 重写 `core/protocol/vless/format.md`（当前 3248 B → 目标 15000+ B）

**必须新增**:

1. **VLESS 请求头完整字节级结构**:
   ```
   VLESS 请求:
   ┌─────────┬──────────┬──────────────┬──────┬──────┬──────────┬──────┐
   │ Version  │  UUID     │ AddnlInfoLen │ CMD  │ PORT │  ATYP     │ 地址  │
   │ (1 B)    │ (16 B)    │ (1 B)        │(1 B) │(2 B) │ (1 B)     │(var) │
   └─────────┴──────────┴──────────────┴──────┴──────┴──────────┴──────┘

   Version: 0x00 (唯一合法值)
   UUID: 16 字节原始二进制 (不是十六进制字符串!)
   AddnlInfoLen: 0x00 (当前不支持非零值)
   CMD: 0x01=TCP, 0x02=UDP, 0x7F=Mux
   ```

2. **VLESS 响应格式**:
   ```
   VLESS 响应:
   ┌─────────┬──────────────┐
   │ Version  │  AddnlInfo    │
   │ (1 B)    │  (var)        │
   └─────────┴──────────────┘
   最小响应: [0x00] (仅 1 字节)
   ```

3. **UUID 的二进制表示**:
   - 文本格式 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` → 16 字节二进制
   - 字节序说明（RFC 4122 的 mixed-endian）
   - Prism 中的 UUID 验证逻辑

4. **完整请求/响应时序图**

5. **安全分析**:
   - VLESS 无加密，完全依赖外层 TLS（与 Trojan 相同风险）
   - UUID 作为唯一凭证的熵（128 bit，足够但不可轮换）
   - AddnlInfo 的潜在用途和当前拒绝策略
   - XTLS/Vision 不支持的说明和原因

#### 2.3.2 扩展 `core/protocol/vless/relay.md`（当前 3514 B → 目标 10000+ B）

---

### 任务 2.4: SOCKS5 协议深度扩展

#### 2.4.1 重写 `core/protocol/socks5/wire.md`（当前 4005 B → 目标 12000+ B）

**必须新增**:
- 每种消息类型的完整字节级图解（方法协商、认证、CONNECT 请求/响应、UDP ASSOCIATE）
- UDP 数据报封装格式的详细说明
- 边界条件：domain 长度 0、port 超范围、nmethods 255

#### 2.4.2 重写 `core/protocol/socks5/stream.md`（当前 4979 B → 目标 12000+ B）

**必须新增**:
- 完整状态机图（SOCKS5_INIT → METHOD_SELECTED → AUTHENTICATING → CONNECTING → ESTABLISHED → CLOSED）
- UDP ASSOCIATE 完整流程图
- 认证失败的日志关键词和排查步骤
- IPv4-only UDP 绑定 bug 的详细说明

---

### 任务 2.5: HTTP 代理深度扩展

#### 2.5.1 重写 `core/protocol/http/parser.md`（当前 3165 B → 目标 10000+ B）

**必须新增**:
- CONNECT 请求解析的完整字节级流程
- Basic 认证的 Base64 解码 → user:password 分割 → SHA224 哈希 → 目录查找
- 407/403/502 响应的构造逻辑
- 头部大小限制（64KB）的实现位置

#### 2.5.2 重写 `core/protocol/http/relay.md`（当前 2801 B → 目标 10000+ B）

**必须新增**:
- HTTP 隧道 vs 普通 HTTP 代理的分支逻辑
- `forward()` 的实现细节
- 分块传输编码（chunked）的处理或跳过逻辑
- Keep-Alive 管理

---

### 任务 2.6: Stealth 模块深度扩展

#### 2.6.1 重写 `core/stealth/reality/seal.md`（当前 2748 B → 目标 10000+ B）

**必须新增**:
- Reality 加密传输层的完整字节级结构
- AEAD 加密记录格式（与 TLS 1.3 record 的对比）
- Nonce 构造（序列号递增方式）
- 6 个内部缓冲区的作用详解

#### 2.6.2 扩展 `core/stealth/shadowtls/auth.md`（当前 5466 B → 目标 12000+ B）

**必须新增**:
- HMAC-SHA1 累积认证的完整算法图解
- ClientHello 中 HMAC 标签位置的确定算法
- v3 数据帧 HMAC 规则表的完整示例
- 与 sing-box shadowtls 实现的兼容性对照

#### 2.6.3 扩展 `core/stealth/ech/decrypt.md`（当前 5466 B → 目标 12000+ B）

**必须新增**:
- HPKE (Hybrid Public Key Encryption) 完整流程图解
- ECH outer ClientHello 格式的字节级结构
- HPKE SetupBaseR 的 7 步算法
- 密钥封装 → 共享密钥 → 上下文 → 解密的完整链路

---

## 四、Phase 3（第三批）: 浅文件重写 + 专业文档补全

> **目标**: 重写所有浅文件，补全专业文档类型。

### 任务 3.1: 重写 P0 浅文件（15 个文件）

每个文件目标：至少 8000 B，包含完整的知识体系。

| # | 文件 | 重写要求 |
|---|------|---------|
| 1 | `ref/protocol/http-1-1.md` | HTTP/1.1 完整规范：请求/响应格式、方法列表、状态码大全、首部字段、分块传输、Keep-Alive、Pipeline、与 HTTP/2 的演进关系。目标 20000+ B |
| 2 | `ref/logging/spdlog.md` | spdlog 完整使用：异步日志原理、sink 类型、格式化器、轮转策略、Prism 中的集成方式（trace 模块）。目标 10000+ B |
| 3 | `ref/programming/cpp-23.md` | C++23 新特性详解：每个特性的语法、语义、Prism 中的应用。重点：协程（co_await/co_spawn）、ranges、expected、std::print、deducing this。目标 15000+ B |
| 4 | `core/loader/overview.md` | 配置加载完整流程：JSON 反序列化（glaze 库）、配置验证、字段缺失处理、配置热重载机制。目标 10000+ B |
| 5 | `core/transformer/overview.md` | JSON 序列化/反序列化：glaze 库集成、自定义序列化器、配置文件到 C++ 结构体的映射。目标 8000+ B |
| 6 | `core/protocol/common/form.md` | 传输形态详解：stream vs datagram 的设计选择、各协议的形态、与 TCP/UDP 的映射关系。目标 6000+ B |
| 7 | `core/protocol/common/read.md` | 共享 I/O 读取工具详解：`read_at_least`/`read_remaining` 的实现、零拷贝设计、与 `frame_arena` 的配合。目标 8000+ B |
| 8 | `core/protocol/common/address.md` | 地址类型体系：IPv4/IPv6/域名的 variant 设计、各协议地址格式的统一抽象、ATYP 值的跨协议对比。目标 8000+ B |
| 9 | `core/protocol/trojan/constants.md` | 常量详解：每个枚举值的使用场景、与 Trojan 协议规范的对应关系、CMD 的扩展历史。目标 6000+ B |
| 10 | `core/protocol/trojan/config.md` | 配置详解：每个字段的 JSON 键名 → C++ 字段名 → 默认值 → 验证逻辑 → 运行时行为的完整追踪链。目标 8000+ B |
| 11 | `core/protocol/vless/constants.md` | 常量详解：跨协议 ATYP 对比表的扩展、Version 的兼容性说明、CMD 值与 Trojan 的差异。目标 8000+ B |
| 12 | `core/protocol/vless/config.md` | 配置详解：UUID 格式验证、enable_udp 的代码路径追踪。目标 8000+ B |
| 13 | `core/protocol/overview.md` | 协议层架构总览：模块架构图、各协议间的关系图、公共抽象层、错误处理策略、dispatch 注册机制。目标 12000+ B |
| 14 | `core/stealth/reality/config.md` | 配置详解：密钥格式（X25519）、short_id 格式、dest 解析逻辑、配置验证。目标 8000+ B |
| 15 | `docs/overview.md` | 用户入口总览：Prism 是什么、核心特性、协议支持矩阵、架构概览、快速导航。目标 10000+ B |

### 任务 3.2: 重写 P1 浅文件（18 个文件）

| # | 文件 | 重写要求 |
|---|------|---------|
| 16 | `core/protocol/shadowsocks/config.md` | PSK 长度→密码套件映射、时间戳窗口调节策略、salt_pool_ttl 的容量计算。目标 8000+ B |
| 17 | `core/protocol/socks5/constants.md` | reply_code 每个值的触发场景、与 RFC 1928 的对应关系。目标 6000+ B |
| 18 | `core/protocol/socks5/config.md` | 配置验证逻辑（enable_auth 但无用户时的行为）。目标 6000+ B |
| 19 | `core/stealth/reality/request.md` | ClientHello 完整字节级字段图解（与 ref/protocol/tls-clienthello.md 的交叉引用）。目标 10000+ B |
| 20 | `core/stealth/reality/response.md` | ServerHello 完整字节级字段图解、加密记录构造流程。目标 10000+ B |
| 21 | `core/stealth/reality/auth.md` | 5 步认证的每步错误场景、失败日志关键词、排查步骤。目标 8000+ B |
| 22 | `core/stealth/reality/seal.md` | 加密记录字节级结构、AEAD nonce 构造、缓冲区管理。目标 10000+ B |
| 23 | `core/stealth/reality/scheme.md` | Tier 0 判定规则的完整描述、hint 5 级评分的计算逻辑。目标 8000+ B |
| 24 | `core/protocol/shadowsocks/format.md` | **已在任务 2.1 中覆盖** |
| 25 | `core/protocol/tls/signal.md` | `parse_client_hello()` 的完整字段提取逻辑、返回结构体每个字段的含义。目标 8000+ B |
| 26 | `core/protocol/shadowsocks/replay.md` | **已在任务 2.1 中覆盖** |
| 27 | `core/protocol/shadowsocks/salts.md` | **已在任务 2.1 中覆盖** |
| 28 | `core/protocol/trojan/format.md` | **已在任务 2.2 中覆盖** |
| 29 | `core/stealth/anytls/scheme.md` | AnyTLS 认证算法、空壳方案说明、实现路线图。目标 8000+ B |
| 30 | `core/stealth/restls/scheme.md` | Restls Script 语法详解、空壳方案说明、实现路线图。目标 10000+ B |
| 31 | `core/stealth/trusttunnel/scheme.md` | TrustTunnel 双模式说明、空壳方案说明。目标 8000+ B |
| 32 | `docs/getting-started.md` | 完整的分步指南：下载/编译/配置/启动/验证/停止。目标 10000+ B |

### 任务 3.3: 重写 P2 浅文件（21 个文件）

| # | 文件 | 重写要求 |
|---|------|---------|
| 33-37 | `shadowsocks/relay.md`, `socks5/wire.md`, `socks5/stream.md`, `http/parser.md`, `http/relay.md` | **已在任务 2.4-2.5 中覆盖** |
| 38 | `core/protocol/vless/format.md` | **已在任务 2.3 中覆盖** |
| 39 | `core/protocol/vless/relay.md` | verifier 回调机制、mux 引导流程、UDP over TLS 复用。目标 10000+ B |
| 40 | `core/protocol/trojan/relay.md` | 凭证验证回调、mux 引导、UDP over TLS 帧。目标 10000+ B |
| 41 | `core/protocol/tls/feature_bitmap.md` | 16+ 位特征的逐一说明、Reality 独占标记的检测代码。目标 8000+ B |
| 42 | `core/protocol/tls/types.md` | CipherSuite 安全分析、每个常量在 Prism 中的使用位置。目标 8000+ B |
| 43 | `core/stealth/shadowtls/constants.md` | v3 帧格式字节图、HMAC 偏移计算示例。目标 8000+ B |
| 44 | `core/stealth/shadowtls/config.md` | v2 vs v3 行为差异、多用户配置格式。目标 8000+ B |
| 45 | `core/stealth/shadowtls/scheme.md` | Tier 1 判定算法、HMAC 匹配流程。目标 8000+ B |
| 46 | `core/stealth/registry.md` | 注册顺序对检测的影响、优先级变更的后果。目标 8000+ B |
| 47 | `core/stealth/native.md` | fallback 判定细节、权重计算、与 tier 系统的关系。目标 8000+ B |
| 48 | `core/stealth/ech/config.md` | ECH 密钥生成步骤、密钥格式、与 Reality 密钥的区别。目标 10000+ B |
| 49 | `core/stealth/ech/decrypt.md` | **已在任务 2.6 中覆盖** |
| 50 | `core/stealth/executor.md` | rewind 失败场景、快照/回滚机制的实现细节。目标 10000+ B |
| 51 | `dev/performance/overview.md` | 性能优化方法论、Prism 的性能目标、基准测试体系。目标 10000+ B |
| 52 | `dev/roadmap.md` | 更新为最新的 10 周 5 阶段路线图。目标 10000+ B |
| 53 | `ref/protocol/overview.md` | 添加错误码索引、配置映射表、各协议的安全对比。目标 12000+ B |

### 任务 3.4: 创建专业文档类型

#### 3.4.1 Prism 协议能力矩阵（新建 `docs/protocol-matrix.md`）

```
| 协议      | TCP | UDP | MUX(smux) | MUX(yamux) | 认证方式        | 自加密 | 依赖TLS |
|-----------|-----|-----|-----------|------------|----------------|--------|---------|
| HTTP      | ✅  | ❌  | ❌        | ❌         | Basic(SHA224)  | ❌     | 可选    |
| SOCKS5    | ✅  | ✅  | ❌        | ❌         | UserPass(SHA224)| ❌    | ❌      |
| Trojan    | ✅  | ✅  | ✅        | ✅         | SHA224凭证     | ❌     | 必须    |
| VLESS     | ✅  | ✅  | ✅        | ✅         | UUID           | ❌     | 必须    |
| SS2022    | ✅  | ✅  | ❌        | ❌         | PSK(AEAD)      | ✅     | 可选    |

| 伪装方案      | 协议层 | 特征等级 | 多用户 | 状态 |
|--------------|--------|---------|--------|------|
| Reality      | TLS    | Tier 0  | ❌     | 完成 |
| ShadowTLS v3 | TLS    | Tier 1  | ✅     | 完成 |
| Restls       | TLS    | Tier 2  | ❌     | TODO |
| AnyTLS       | TLS    | Tier 2  | ✅     | TODO |
| ECH          | TLS    | Tier 1  | ❌     | TODO |
| TrustTunnel  | TLS    | Tier 2  | ❌     | TODO |
```

#### 3.4.2 架构决策记录（新建 `dev/adr/` 目录，5 个文件）

| 文件 | 主题 |
|------|------|
| `dev/adr/001-cpp23-coroutine.md` | 为什么选择 C++23 + Boost.Asio 协程而非 Rust/Go/Callback |
| `dev/adr/002-pmr-memory.md` | 为什么选择 PMR 而非自定义分配器或 arena |
| `dev/adr/003-layered-recognition.md` | 为什么选择分层检测而非端口匹配或 DPI |
| `dev/adr/004-transport-abstraction.md` | 为什么设计 transmission 基类和装饰器链 |
| `dev/adr/005-worker-per-core.md` | 为什么选择 per-core io_context 而非线程池 + strand |

每个 ADR 包含：背景、决策、原因、后果、替代方案及否决理由。

#### 3.4.3 客户端配置模板（新建 `docs/client-setup/` 目录）

| 文件 | 内容 |
|------|------|
| `docs/client-setup/clash-verge.md` | Clash Verge Rev 完整配置模板（覆盖所有协议+伪装+多路复用） |
| `docs/client-setup/nekobox.md` | NekoBox for Android 配置 |
| `docs/client-setup/shadowrocket.md` | Shadowrocket (iOS) 配置 |
| `docs/client-setup/sing-box.md` | sing-box 配置 |

#### 3.4.4 迁移指南（新建 `docs/migration/` 目录）

| 文件 | 内容 |
|------|------|
| `docs/migration/from-xray.md` | Xray/V2Ray 配置转换到 Prism |
| `docs/migration/from-mihomo.md` | mihomo/Clash 配置转换到 Prism |
| `docs/migration/from-singbox.md` | sing-box 配置转换到 Prism |

---

## 五、Phase 4（第四批）: 知识网络增强

> **目标**: 在各知识点之间建立深度交叉引用，增加"参见"和"发散知识"链接。

### 任务 4.1: 协议间交叉引用增强

在以下位置添加交叉引用：

1. **每个协议的 conn.md** 添加"与其他协议对比"小节：
   - 连接建立流程对比（SOCKS5 多步协商 vs Trojan 单步凭证 vs SS2022 AEAD 握手）
   - 安全模型对比（自加密 vs 依赖 TLS vs 无加密）
   - 性能特征对比（延迟、吞吐、内存开销）

2. **每个协议的 format.md/framing.md** 添加"与 RFC 的对应关系"小节：
   - Trojan 格式与 Trojan 协议规范的对应
   - SOCKS5 与 RFC 1928/1929 的逐字段对应
   - SS2022 与 SIP022 规范的逐字段对应

3. **加密模块** 添加"在协议中的应用"链接：
   - `ref/crypto/aes-gcm.md` → SS2022 AEAD 加密、Reality seal
   - `ref/crypto/x25519.md` → Reality 密钥交换
   - `ref/crypto/hkdf.md` → TLS 1.3 密钥派生、SS2022 KDF
   - `ref/crypto/blake3.md` → SS2022 密钥派生
   - `ref/crypto/hmac-sha1.md` → ShadowTLS v3 认证

4. **反审查** 模块添加"与 Prism 实现的对应"链接：
   - `ref/anti-censorship/tls-fingerprint.md` → recognition 分层检测
   - `ref/anti-censorship/probing.md` → stealth 伪装方案
   - `ref/anti-censorship/replay-attack.md` → SS2022 salt 池 + PacketID 窗口

### 任务 4.2: 配置→代码追踪链

为每个协议的 config.md 添加完整的追踪链：

```
JSON 配置项 → glaze 反序列化 → config 结构体字段 →
  → relay 构造函数参数 → 运行时行为

示例（Trojan enable_udp）:
"protocol.trojan.enable_udp": true
  → trojan_config::enable_udp (bool)
    → trojan_relay 构造: enable_udp 参数
      → process: CMD==0x03 时检查 enable_udp
        → true: async_associate() → UDP over TLS 帧循环
        → false: 返回 fault::code::not_supported
```

### 任务 4.3: 安全最佳实践文档（新建 `docs/security-best-practices.md`）

- 密钥管理最佳实践
- TLS 配置推荐（与 `src/prism/instance/worker/tls.cpp` 对应）
- 防火墙规则建议
- 日志安全（不记录敏感数据）
- DDoS 防护配置

---

## 六、执行顺序与工作量估算

```
Phase 1 (结构性缺口):
  ├─ 1.1 创建 core/stats/ (6 个新文件 × ~3000 B)     ≈ 18 KB
  ├─ 1.2 补全 protocol/ 实现层 (25 个新文件 × ~5000 B) ≈ 125 KB
  └─ 1.3 统一模块命名 (迁移 + 更新链接)                ≈ 全局

Phase 2 (协议深度扩展):
  ├─ 2.1 SS2022 深度 (1 重写 + 3 扩展)                ≈ 50 KB 新增
  ├─ 2.2 Trojan 深度 (1 重写 + 1 扩展)                ≈ 25 KB 新增
  ├─ 2.3 VLESS 深度 (1 重写 + 1 扩展)                 ≈ 25 KB 新增
  ├─ 2.4 SOCKS5 深度 (2 重写)                         ≈ 20 KB 新增
  ├─ 2.5 HTTP 深度 (2 重写)                           ≈ 15 KB 新增
  └─ 2.6 Stealth 深度 (3 扩展)                        ≈ 20 KB 新增

Phase 3 (浅文件重写 + 专业文档):
  ├─ 3.1 P0 浅文件 (15 重写 × ~8000 B)                ≈ 120 KB
  ├─ 3.2 P1 浅文件 (18 重写 × ~8000 B)                ≈ 144 KB
  ├─ 3.3 P2 浅文件 (21 重写 × ~8000 B)                ≈ 168 KB
  └─ 3.4 专业文档 (协议矩阵+ADR+客户端+迁移 ≈ 10 文件) ≈ 80 KB

Phase 4 (知识网络增强):
  ├─ 4.1 协议间交叉引用                               ≈ 全局
  ├─ 4.2 配置追踪链                                   ≈ 全局
  └─ 4.3 安全最佳实践                                 ≈ 10 KB

总计: ~50 个新建文件 + ~54 个重写文件 ≈ 800 KB 新增内容
```

---

## 七、质量标准

每个重写/新建文件必须包含：

1. **Frontmatter**: `--- aliases: [...] tags: [...] ---`
2. **命名空间和源码路径**: 准确的 `psm::xxx` 命名空间 + `...` 文件路径
3. **至少 3 个 wikilink**: 链接到相关的 ref 层（协议规范）或 core 层（关联模块）
4. **至少 1 个代码示例**: C++ 函数签名或配置 JSON
5. **调用链**: 谁调用此函数、此函数调用谁
6. **协议文档额外要求**:
   - 线缆格式字节级图解（ASCII 艺术或表格）
   - 状态机转换图
   - 完整请求/响应时序图
   - 错误码与错误处理表
   - 攻击面分析
   - 兼容性说明
7. **配置文档额外要求**:
   - JSON → C++ → 行为的完整追踪链
   - 验证逻辑
   - 默认值和取值范围

---

## 八、风险与注意事项

| 风险 | 缓解 |
|------|------|
| 工作量巨大（~100 个文件） | 按 Phase 优先级执行，每个 Phase 可独立交付 |
| 命名迁移可能导致链接断裂 | 迁移后全局搜索更新 wikilink |
| 新文档可能与源码不同步 | 每个文件标注源码版本和最后验证日期 |
| 协议字节级图解可能有误 | 与 RFC 原文和实际抓包交叉验证 |
| 空壳方案（Restls/AnyTLS/TrustTunnel/ECH）文档如何写 | 标注"实现中"，写出协议规范但说明 Prism 尚未实现 |
