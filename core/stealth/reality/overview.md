---
title: Reality 协议
created: 2026-05-27
updated: 2026-05-29
layer: core
source:
  - include/prism/stealth/facade/reality/scheme.hpp
  - include/prism/stealth/facade/reality/handshake.hpp
  - include/prism/stealth/facade/reality/seal.hpp
  - include/prism/stealth/facade/reality/config.hpp
  - include/prism/stealth/facade/reality/util/auth.hpp
  - include/prism/stealth/facade/reality/util/keygen.hpp
  - include/prism/stealth/facade/reality/util/response.hpp
  - src/prism/stealth/facade/reality/scheme.cpp
  - src/prism/stealth/facade/reality/handshake.cpp
  - src/prism/stealth/facade/reality/seal.cpp
  - src/prism/stealth/facade/reality/util/auth.cpp
  - src/prism/stealth/facade/reality/util/keygen.cpp
  - src/prism/stealth/facade/reality/util/response.cpp
tags: [stealth, reality, overview, tier-0, x25519, tls-1.3]
---

# Reality 协议

> 源码: `include/prism/stealth/facade/reality/` | 实现: `src/prism/stealth/facade/reality/`

Reality 是 Stealth 模块中最高优先级的 TLS 伪装方案（Tier 0，独占检测），通过 X25519 密钥交换完成客户端认证，自实现 TLS 1.3 握手建立加密传输层，未认证连接透明回退到真实目标网站。

## 模块架构

```
ClientHello 到达
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│  scheme::sniff() — Tier 0 快速检测                       │
│  检查 reality_marker / X25519 / session_id 特征位        │
│  命中 → 独占（solo=true），跳过其他方案                    │
└──────────────────────────────────────────────────────────┘
       │ 命中
       ▼
┌──────────────────────────────────────────────────────────┐
│  handshake() — 四阶段流水线                                │
│                                                          │
│  ① authenticate_client                                   │
│     read_tls_record → parse_client_hello → authenticate  │
│     ├─ SNI 不匹配 → preread 原始记录，交给下一方案         │
│     ├─ ClientHello 异常 → fallback_dest（透传真实网站）     │
│     └─ 认证通过 → auth_result                             │
│                                                          │
│  ② negotiate_tls（纯本地计算，无网络 I/O）                │
│     X25519(临时密钥, 客户端公钥) → 共享密钥               │
│     generate_shello → ServerHello + 伪造证书              │
│     derive_hs_keys → 握手密钥                             │
│     derive_and_encrypt_finished → 重算 Finished           │
│                                                          │
│  ③ complete_hello（网络 I/O）                             │
│     发送 ServerHello + CCS + 加密握手记录                  │
│     consume_client_finished → 验证客户端 Finished         │
│                                                          │
│  ④ 建立 seal 传输层                                       │
│     derive_app_keys → 应用流量密钥                        │
│     构造 seal（AES-128-GCM 加解密）                       │
│     preread 64 字节内层数据                               │
└──────────────────────────────────────────────────────────┘
       │
       ▼
  handshake_result { transport=seal, detected, preread }
```

### 核心组件

| 组件 | 头文件 | 职责 |
|------|--------|------|
| [[core/stealth/reality/scheme\|scheme]] | `facade/reality/scheme.hpp` | 方案注册、Tier 0 sniff、handshake 入口 |
| [[core/stealth/reality/handshake\|handshake]] | `facade/reality/handshake.hpp` | 四阶段握手流水线、fallback、证书获取 |
| [[core/stealth/reality/seal\|seal]] | `facade/reality/seal.hpp` | TLS 1.3 ApplicationData 加解密传输层 |
| [[core/stealth/reality/auth\|auth]] | `facade/reality/util/auth.hpp` | SNI 匹配、X25519 密钥交换、short_id 验证 |
| [[core/stealth/reality/keygen\|keygen]] | `facade/reality/util/keygen.hpp` | TLS 1.3 HKDF 密钥调度（RFC 8446 §7） |
| [[core/stealth/reality/response\|response]] | `facade/reality/util/response.hpp` | ServerHello 生成、TLS 记录构造/加密 |
| [[core/stealth/reality/config\|config]] | `facade/reality/config.hpp` | Reality 配置结构体（dest/SNI/private_key/short_ids） |

## 子页面

- [[core/stealth/reality/scheme|Scheme]] — 方案基类实现
- [[core/stealth/reality/config|Config]] — 配置参数
- [[core/stealth/reality/keygen|KeyGen]] — TLS 1.3 密钥调度
- [[core/stealth/reality/auth|Auth]] — 客户端认证流程
- [[core/stealth/reality/handshake|Handshake]] — 完整握手流程
- [[core/stealth/reality/response|Response]] — ServerHello 响应构造
- [[core/stealth/reality/seal|Seal]] — AEAD 加密封装传输层

## 设计决策（WHY）

### 为什么自实现 TLS 1.3 握手而不用 BoringSSL？

**问题**: Reality 需要对 TLS 握手消息进行细粒度控制——伪造证书、修改 session_id、重算 Finished——这些操作在 BoringSSL 的 TLS 状态机中不可插拔。

**选择**: 在 `keygen.hpp` 和 `response.hpp` 中自行实现 TLS 1.3 密钥调度（RFC 8446 §7）和 ServerHello 构造，完全绕过 BoringSSL 的握手流程。仅依赖 BoringSSL 的底层加密原语（AEAD、HKDF、X25519）。

**后果**: 对握手行为有完全控制，可精确模拟目标站点的 TLS 指纹。但需要自行维护与 TLS 1.3 规范的兼容性，协议更新需手动跟进。`fetch_dest_cert()` 中连接真实网站时仍使用 BoringSSL 的标准 TLS 客户端。

### 为什么用 X25519 密钥交换内嵌认证？

**问题**: TLS 握手阶段需要同时完成密钥协商和客户端身份认证，不能增加额外 RTT。

**选择**: Reality 复用 TLS ClientHello 中的 `key_share` 扩展（X25519 公钥）和 `session_id` 字段来传递认证数据。服务端用静态私钥与客户端公钥做 X25519 得到共享密钥，派生 `auth_key`，再用 AES-128-GCM 解密 session_id 中的 short_id 进行认证。认证和密钥交换在一次 ClientHello 中完成，零额外 RTT。

**后果**: 认证强度取决于 X25519 的 128 位安全强度。session_id 中嵌入认证数据是 Reality 协议的独有特征，DPI 可通过 session_id 前缀 `[0x01, 0x08, 0x02]` 识别（Tier 0 sniff 正是利用此特征）。

### 为什么回退到真实目标站？

**问题**: 未认证的连接（审查者主动探测）必须返回合法响应，否则探测者可确认服务器运行了代理。

**选择**: `fallback_dest()` 将客户端的原始 ClientHello 转发给配置的 `dest` 目标网站（如 `www.microsoft.com:443`），之后双向透传。探测者看到的是与真实网站完全正常的 TLS 通信。

**后果**: 回退路径需要建立到目标站的 TCP 连接，增加延迟。如果目标站不可达，回退失败，连接关闭——但此时已写入数据（`polluted=true`），无法 rewind 到其他方案。dest 站点的选择直接影响回退成功率。

### 为什么使用 AES-128-GCM 而非 ChaCha20-Poly1305？

**问题**: seal 传输层需要高性能的加解密操作，在数据转发热路径中频繁调用。

**选择**: `seal` 构造函数中硬编码使用 `crypto::aead_cipher::aes_128_gcm`。AES-128-GCM 在支持 AES-NI 的 x86 平台上有硬件加速，吞吐量显著高于 ChaCha20-Poly1305。

**后果**: 不支持 AES-NI 的平台上性能较差（Prism 当前面向的服务端部署通常支持 AES-NI）。密钥长度 128 位对 Reality 的安全需求足够（Reality 的安全边界不在加密强度，而在握手伪装）。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 单次握手不可重试 | Reality 认证在阶段①完成，失败后只能 fallback 或交给下一方案 | 认证失败后 `polluted=true`，无法 rewind 到其他方案 | `handshake.cpp` |
| session_id 标记唯一性 | `[0x01, 0x08, 0x02]` 前缀是 Reality 协议硬编码 | DPI 可通过此特征识别 Reality 流量（已知风险，换来零成本检测） | `scheme.cpp:30` |
| short_id 空字符串接受任意 | `match_shortid()` 中空字符串匹配所有 short_id | 安全性降低，建议配置具体的 short_id | `auth.cpp` |
| private_key 必须 32 字节 | base64 解码后必须恰好 32 字节 | 解码后长度不匹配直接 fallback，握手失败 | `handshake.cpp:264` |
| 握手 30 秒超时 | `steady_timer` 设置 30 秒，超时 cancel 传输层 | 超时后连接中断，日志记录 timeout | `handshake.cpp:670` |
| seal nonce 序列号防溢出 | `read_seq_` / `write_seq_` 达到 `UINT64_MAX-1` 时返回 `crypto_error` | 序列号溢出后连接关闭（实际不可能在正常使用中发生） | `seal.cpp:154,232` |
| `unique()==true` 独占方案 | Tier 0 sniff 命中后 `solo=true`，跳过所有其他方案 | 独占方案失败无回退，必须极其可靠 | `scheme.hpp:25` |

## 故障场景

### 1. X25519 密钥交换失败

**触发条件**: 客户端 key_share 中的公钥为低阶点或无效数据

**传播路径**: `authenticate()` → `auth_result.authenticated=false` → `authenticate_client` 判断 `!auth_res.authenticated` → preread 原始记录，`detected=tls`

**外部表现**: 客户端收到标准 TLS 处理，非 Reality 客户端不受影响

**恢复机制**: 自动交给下一个伪装方案处理

**日志关键字**: `"auth failed"` + `"not Reality, passing to next scheme"`

### 2. short_id 不匹配

**触发条件**: 客户端 session_id 解密后的 short_id 不在配置的 `short_ids` 列表中

**传播路径**: `match_shortid()` 返回 false → `authenticate()` 返回 `fault::code::authfail` → preread + 交给下一方案

**外部表现**: 同上，流量转给其他伪装方案

**日志关键字**: `"auth failed"` + 具体错误码描述

### 3. SNI 不在白名单

**触发条件**: ClientHello 的 SNI 不在 `server_names` 配置中

**传播路径**: `match_sni()` 返回 false → `authenticate()` 返回 `fault::code::badsni` → preread 原始记录

**外部表现**: 非白名单 SNI 的客户端被视为非 Reality 客户端

**日志关键字**: `"SNI mismatch, falling back to standard TLS"`

### 4. dest 目标站不可达

**触发条件**: ClientHello 解析失败或私钥配置错误时触发 fallback，但 dest 站点无法连接

**传播路径**: `fallback_dest()` → `async_forward()` 失败 → 返回 `fault::code::unreach` → `handshake_result.error=unreach`

**外部表现**: 连接直接关闭

**日志关键字**: `"connect to dest failed"` + `"invalid dest config"`

### 5. 客户端 Finished 验证失败

**触发条件**: 客户端发送的 Finished 消息解密失败或内容异常（ALERT）

**传播路径**: `consume_client_finished()` → AEAD open 失败或收到 ALERT → 返回 `fault::code::hsfail`

**外部表现**: 握手终止，连接关闭

**日志关键字**: `"client sent TLS ALERT"` 或 `"failed to decrypt client record"`

### 6. AEAD 解密封装层传输断开

**触发条件**: seal 传输层在读取/写入过程中底层连接断开

**传播路径**: `seal::recv_record()` → `tls::record::read()` 失败 → `connection_reset`

**外部表现**: 代理连接中断

**恢复机制**: session 层清理资源，统计记录流量

**日志关键字**: `"AEAD decrypt failed"` 或 `"received TLS alert record"`

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `recognition` → `reality` | 调用 | recognition 解析 ClientHello 生成特征位图，`sniff()` 据此判断是否 Reality |
| `reality` → [[core/crypto/overview\|crypto]] | 依赖 | X25519 密钥交换、HKDF 密钥派生、AES-128-GCM AEAD 加解密 |
| `reality` → [[core/protocol/tls/types\|protocol/tls]] | 依赖 | TLS 记录层常量（content_type、AEAD_NONCE_LEN、REALITY_KEY_LEN） |
| `reality` → [[core/connect/dial/dial\|connect]] | 依赖 | `async_forward()` 建立 TCP 连接到 dest 站点 |
| `reality` → [[core/connect/tunnel/tunnel\|connect]] | 依赖 | `fallback_dest()` 使用 `connect::tunnel()` 双向透传 |
| `reality` → [[core/transport/transmission\|transport]] | 依赖 | seal 继承 `transport::transmission`，作为 session 层的传输层 |
| [[core/instance/overview\|instance]] ← `reality` | 被依赖 | session 通过 `scheme::handshake()` 调用 Reality 握手 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| Reality private_key 更换 | 所有客户端 | 客户端必须更新配置中的 public_key，否则认证失败 |
| server_names 变更 | 所有客户端 | SNI 不匹配的客户端被当作非 Reality 客户端 |
| short_ids 变更 | 所有客户端 | 使用旧 short_id 的客户端认证失败 |
| dest 站点变更 | 回退行为 | 探测者看到的目标站不同 |
| `sniff()` 特征位逻辑变更 | 检测管道 | 可能导致误判或漏判 Reality 连接 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| `recognition::tls::feature_bit` 枚举变更 | `scheme::sniff()` 的特征位检测 | 所有 `has_feature` / `has_all` 调用 |
| `crypto::x25519()` 返回值语义变更 | `authenticate()` 和 `negotiate_tls()` | 错误处理和全零共享密钥检查 |
| `crypto::hkdf::expand_label()` label 格式变更 | `keygen.hpp` 的密钥派生 | `"tls13 "` 前缀必须匹配 RFC 8446 |
| `protocol::tls` 常量值变更 | seal 和 response 的记录构造 | AEAD_NONCE_LEN、REALITY_KEY_LEN、CT_* |
| BoringSSL API 变更 | `fetch_dest_cert()` 的 SSL_* 调用 | 证书提取逻辑 |
| `transport::transmission` 接口变更 | seal 的 read/write 实现 | async_read_some / async_write_some 签名 |

## 相关文档

- [[core/stealth/overview|Stealth 模块总览]] — 三级检测架构和方案执行器
- [[core/crypto/x25519|X25519]] — 密钥交换原语
- [[core/crypto/aead|AEAD]] — 认证加密原语
- [[core/crypto/hkdf|HKDF]] — 密钥派生原语
- [[core/recognition/overview|Recognition 模块]] — ClientHello 特征提取
- [[core/transport/transmission|Transport]] — 传输层抽象接口
- [[core/connect/tunnel/tunnel|Tunnel]] — 双向转发（fallback 使用）
