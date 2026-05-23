---
title: Reality 握手深层故障分析
source:
  - I:/code/Prism/src/prism/stealth/reality/handshake.cpp
  - I:/code/Prism/src/prism/stealth/reality/util/auth.cpp
  - I:/code/Prism/src/prism/stealth/reality/util/keygen.cpp
  - I:/code/Prism/src/prism/stealth/reality/seal.cpp
  - I:/code/Prism/src/prism/stealth/reality/util/response.cpp
  - I:/code/Prism/src/prism/stealth/executor.cpp
module: debugging
type: deep-dive
tags: [debugging, reality, handshake, tls, authentication, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[dev/debugging/tls]]"
  - "[[core/stealth/reality/handshake]]"
  - "[[core/stealth/reality/auth]]"
  - "[[core/stealth/reality/seal]]"
  - "[[core/stealth/executor]]"
---

# Reality 握手深层故障分析

## 概述

Reality 握手是 Prism stealth 子系统中最复杂的方案之一。它伪装成向真实 TLS 服务器的正常握手，在 ClientHello 中通过密码学手段嵌入认证信息（short_id），仅被持有正确私钥的服务端识别。

核心数字：

- 认证 12 步，每步有独立失败条件
- 握手 5 阶段共 10 步执行
- 错误码覆盖 9 种 Reality 专属类型（枚举值 49-57）
- 认证为纯密码学比对（HKDF + AES-256-GCM），不涉及时钟/时间窗口

与 Executor 的交互关系：Reality 在 [[core/stealth/executor]] 中注册为 **Tier 0 方案**（最高优先级）。当 recognition 模块确定性命中 Reality 时，Executor 只执行 Reality 单一方案，失败则直接返回错误 -- 没有回退余地。

```
recognition 确定命中 Reality (Tier 0)
        |
        v
  Executor: 仅执行 Reality
        |
        +---> 成功 --> 返回 encrypted_transport + inner_preread
        |
        +---> 失败 --> 直接返回错误（无回退）
```

---

## 错误码速查表

Reality 相关错误码占枚举值 49-57 区间，定义于 [[core/fault/code]]。

| 错误码 | 值 | 含义 | 触发阶段 |
|--------|-----|------|----------|
| `reality_not_configured` | 49 | Reality 功能未配置 | 初始化 |
| `reality_auth_failed` | 50 | 认证失败（多种子原因） | Stage 2 认证 |
| `reality_sni_mismatch` | 51 | SNI 不在 server_names 列表 | Stage 2 认证 |
| `reality_key_exchange_failed` | 52 | X25519 密钥交换失败 | Stage 2 认证 |
| `reality_handshake_failed` | 53 | 握手消息交换失败 | Stage 4 握手 |
| `reality_dest_unreachable` | 54 | 伪装目标不可达 | Stage 1 回退 |
| `reality_certificate_error` | 55 | 证书获取失败 | Stage 1 回退 |
| `reality_tls_record_error` | 56 | TLS 记录层错误 | Stage 4/5 |
| `reality_key_schedule_error` | 57 | 密钥派生失败 | Stage 3 派生 |

通用错误码在 Reality 握手中也频繁出现：

| 错误码 | 值 | 含义 | 触发场景 |
|--------|-----|------|----------|
| `io_error` | 10 | 底层 I/O 失败 | TLS 记录读取、socket 读写 |
| `protocol_error` | 5 | 协议层违规 | seal 层非法 content type |
| `connection_reset` | 24 | 连接被 RST | seal 层收到 TLS ALERT |

---

## 认证 12 步详解

Reality 认证在 `authenticate()` 函数中执行，所有步骤定义于 [[core/stealth/reality/auth]]。日志统一使用 `[Stealth.Auth]` 标签。

### 步骤总览

| 步骤 | 检查内容 | 失败返回 | 日志特征 |
|------|----------|----------|----------|
| 1 | SNI 匹配 server_names | `reality_sni_mismatch` | `SNI mismatch: {sni}` |
| 2 | X25519 公钥存在 | `reality_auth_failed` | `no X25519 public key` |
| 3 | TLS 1.3 支持 | `reality_auth_failed` | `no TLS 1.3 support` |
| 4 | session_id 长度 >= 32 | `reality_auth_failed` | `session_id too short` |
| 5 | X25519 密钥交换 | `reality_key_exchange_failed` | `X25519 key exchange failed` |
| 6 | 全零共享密钥检查 | `reality_key_exchange_failed` | `all-zero shared secret` |
| 7 | HKDF-Extract | `reality_auth_failed` | `HKDF-Extract failed` |
| 8 | HKDF-Expand | `reality_auth_failed` | `HKDF-Expand failed` |
| 9 | AES-256-GCM 解密 session_id | `reality_auth_failed` | `GCM decrypt failed` |
| 10 | 版本标记 0x01 | `reality_auth_failed` | `invalid version marker` |
| 11 | short_id 匹配 | `reality_auth_failed` | `short_id mismatch` |
| 12 | 认证成功 | -- | `authentication successful` |

### 逐步分析

#### Step 1: SNI 匹配

```
输入: client_hello.server_name
比对: config.stealth.reality.server_names 列表
通过: server_name 在列表中（精确匹配）
失败: 返回 reality_sni_mismatch
```

SNI 不匹配**不是错误** -- 它表示该客户端不是 Reality 客户端。handshake 函数收到此错误码后返回 `not_reality`，交给 Executor 管道中的下一个方案处理。**不触发 fallback_to_dest**。

#### Step 2: X25519 公钥存在

```
输入: client_hello.has_client_public_key
条件: ClientHello key_share 扩展中包含 X25519 公钥
失败: has_x25519 == false
```

没有 X25519 公钥说明客户端不使用 Reality 密钥交换方式。此条件和其他认证失败一起返回 `reality_auth_failed`，handshake 将其视为非 Reality 客户端。

#### Step 3: TLS 1.3 支持

```
输入: client_hello.supported_versions
条件: 列表包含 0x0304 (TLS 1.3)
失败: 不包含 TLS 1.3
```

Reality 依赖 TLS 1.3 的密钥派生和加密机制，TLS 1.2 客户端无法完成后续步骤。

#### Step 4: session_id 长度检查

```
输入: client_hello.session_id
条件: session_id.size() >= 32
失败: 长度不足
```

Reality 客户端在 session_id 中嵌入加密的认证信息（版本标记 + short_id + 填充）。长度不足说明 session_id 未被正确构造。

#### Step 5: X25519 密钥交换

```
输入:
  - 服务端: config 中的 private_key (32 字节，Base64 解码后)
  - 客户端: ClientHello key_share 中的 X25519 公钥
输出: shared_secret (32 字节)
失败: X25519 运算内部错误
```

这一步产生 ECDH 共享密钥，后续所有密钥派生的根基。

#### Step 6: 全零共享密钥检查

```
输入: Step 5 输出的 shared_secret
条件: shared_secret != 0x00...00 (32 字节全零)
失败: 全零 → 低阶点攻击
```

检测小子群攻击（low-order point attack）。攻击者构造特定的 X25519 公钥使得 ECDH 结果为零。全零共享密钥会导致后续密钥完全可预测。

#### Step 7-8: HKDF 密钥派生

```
Step 7: HKDF-Extract
  salt = client_random[0:20]
  IKM  = shared_secret (Step 5 输出)
  输出 → PRK (32 字节)

Step 8: HKDF-Expand
  PRK  = Step 7 输出
  info = "REALITY"
  len  = 32
  输出 → auth_key (32 字节)
```

`auth_key` 用于 Step 9 的 AES-256-GCM 解密。

#### Step 9: AES-256-GCM 解密 session_id

```
密钥: auth_key (Step 8 输出, 32 字节)
Nonce: client_random[20:32] (12 字节)
AAD:  raw ClientHello 中 session_id 部分置零后的数据
密文: client_hello.session_id
输出: [version_marker(1) | padding(...) | short_id(8) | ...]
```

AAD 的构造是关键细节：将原始 ClientHello 消息中 session_id 字段的值替换为全零，作为附加认证数据。这意味着 session_id 的完整性绑定了 ClientHello 的其他字段。

#### Step 10: 版本标记验证

```
输入: 解密结果的第一个字节
条件: == 0x01
失败: 不等于 0x01
```

版本标记用于协议版本协商。当前唯一合法值是 0x01。

#### Step 11: short_id 匹配

```
输入: 解密结果中的 short_id (8 字节)
比对: config.stealth.reality.short_ids 列表
  - 空字符串 → 接受任意 short_id
  - hex 编码比较
通过: 匹配列表中任一项
失败: reality_auth_failed
```

short_id 是客户端身份标识。不同客户端可使用不同 short_id，服务端通过列表管理合法客户端。

### short_id 认证密码学细节

整个认证流程不涉及时钟、时间窗口或网络往返。它是纯本地的密码学验证：

```
client_random (32 字节, ClientHello 中)
     |
     ├── [0:20] ──→ HKDF-Extract salt
     |                    |
     |                    v
     |              shared_secret ──→ HKDF-Extract IKM
     |                    |
     |                    v
     |                   PRK
     |                    |
     |                    v
     |              HKDF-Expand(PRK, "REALITY", 32) ──→ auth_key (32 字节)
     |
     └── [20:32] ──→ AES-256-GCM Nonce (12 字节)

AES-256-GCM:
  Key   = auth_key
  Nonce = client_random[20:32]
  AAD   = ClientHello (session_id 部分置零)
  密文  = session_id

解密成功 → [0x01 | ... | short_id[8] | ...]
                     |
                     v
              short_id 匹配 ──→ 认证通过
```

---

## 握手 5 阶段故障详解

Reality 握手在 `handshake()` 函数中执行，分为 5 个阶段，共 10 步。日志使用 `[Stealth.Handshake]` 标签。

### Stage 1: ClientHello 读取与解析

**涉及步骤**: Step 1-2（读取 + 解析）

| 故障点 | 错误码 | 行为 | 日志 |
|--------|--------|------|------|
| TLS 记录读取失败 | `io_error` | 直接返回 | `failed to read TLS record: {ec}` |
| ClientHello 解析失败 | 透传 | **执行 fallback_to_dest** | `failed to parse ClientHello: {ec}` |
| dest 配置格式错误 | `reality_dest_unreachable` | 返回 failed | `invalid dest config: {dest}` |
| dest 连接失败 | `reality_dest_unreachable` | 返回 failed | `connect to dest failed: {ec}` |
| 写入 dest 失败 | `reality_dest_unreachable` | 返回 failed | `write to dest failed: {msg}` |
| dest TLS 握手失败 | `reality_certificate_error` | 返回 failed | 依赖 fetch_dest_certificate |

#### 裸 socket 泄漏风险

fallback_to_dest 中存在一个微妙的资源管理问题：

```
auto *dest_socket_raw = dest_conn.release();  // 释放所有权

co_await net::async_write(*dest_socket_raw, ...);  // 如果这里失败？
```

`dest_conn.release()` 将 socket 的所有权从智能指针转移到裸指针。如果后续 `async_write` 失败，代码返回 `reality_dest_unreachable`，此时 `dest_socket_raw` 指向的 socket **不会被自动回收**。

在正常流程中（`async_write` 成功后），socket 会被 `make_reliable` 包装成传输层，最终由 `pipeline::primitives::tunnel` 管理。但如果 `async_write` 失败，需要确认是否有 RAII 包装或手动 delete。

### Stage 2: 认证

**涉及步骤**: Step 3（私钥解码）+ Step 4（authenticate 调用）

| 故障点 | 错误码 | 行为 | 日志 |
|--------|--------|------|------|
| 私钥解码后长度 != 32 | 透传 | **执行 fallback_to_dest** | `invalid private key length: {len}` |
| SNI 不匹配 | `reality_sni_mismatch` | 返回 not_reality（不 fallback） | `SNI mismatch, falling back to standard TLS` |
| SNI 为空 + auth 失败 | `reality_auth_failed` | 返回 not_reality | `auth failed with empty SNI` |
| SNI 匹配但 auth 失败 | 透传 | 返回 not_reality | `auth failed: {ec}, not Reality` |
| 认证成功 | -- | 继续 Stage 3 | `authentication successful` |

关键行为区分：

- **私钥无效** → 执行 fallback_to_dest（透明代理到 dest）
- **SNI 不匹配** → 返回 not_reality（交给下一个方案，不 fallback）
- **auth 失败** → 返回 not_reality（交给下一个方案，不 fallback）

这意味着只有**配置错误**（私钥无效）才会触发真正的回退代理，所有认证失败都被视为"非 Reality 客户端"。

### Stage 3: 密钥派生

**涉及步骤**: Step 5-7（ECDH + ServerHello + 密钥派生 + Finished）

密钥派生链包含 9 个 HKDF-Expand-Label 步骤，完整遵循 TLS 1.3 RFC 8446 Section 7：

```
                    0 (全零)
                    |
                    v
              Early Secret  ← HKDF-Extract(salt=0, IKM=0)
                    |
                    v
         Handshake Secret  ← HKDF-Extract(salt=Early Secret, IKM=shared_secret)
                    |
        +-----------+-----------+
        |                       |
        v                       v
  server_hs_traffic      client_hs_traffic
  (HKDF-Expand-Label)    (HKDF-Expand-Label)
        |                       |
        v                       v
  server_finished_key    (client_finished_key)
        |
        +---- HKDF-Extract(salt=Handshake Secret, IKM=0) ----+
        |                                                     |
        v                                                     v
    Master Secret                                      (server Finished)
        |
        +-----------+-----------+
        |                       |
        v                       v
  server_app_traffic      client_app_traffic
  (HKDF-Expand-Label)    (HKDF-Expand-Label)
```

| 故障点 | 错误码 | 日志 |
|--------|--------|------|
| TLS ECDH (临时密钥) 失败 | 透传 | `ephemeral X25519 key exchange failed` |
| ServerHello 生成失败 | 透传 | `failed to generate ServerHello: {ec}` |
| derive_handshake_keys 失败 | 透传 | `failed to derive keys: {ec}` |
| derive_and_encrypt_finished 失败 | 透传 | （内部日志） |
| 明文长度不足 (<36 字节) | `reality_key_schedule_error` | `plaintext too short for Finished: {size}` |

#### 合成证书

Reality 不使用真实证书。`generate_server_hello` 生成一个合成的 Ed25519 证书：

- 有效期：1 小时
- 签名方式：HMAC-SHA512（不是真正的 Ed25519 签名）
- 用途：仅用于 TLS 握手的外层伪装，客户端不会验证此证书

这是因为 Reality 客户端通过预共享公钥认证服务端，不依赖 CA 证书链。

### Stage 4: 握手消息交换

**涉及步骤**: Step 8-9（发送 ServerHello + 消费客户端 Finished）

服务端发送三个 TLS 记录（scatter-gather 写入）：

```
[ServerHello TLS 记录] + [ChangeCipherSpec 兼容性记录] + [加密握手记录]
```

一次性写入减少系统调用。

| 故障点 | 错误码 | 日志 |
|--------|--------|------|
| scatter-gather 写入失败 | `to_code(write_ec)` | `failed to send handshake records: {msg}` |
| 读取客户端记录头失败 | `io_error` | `failed to read client record header` |
| 读取客户端记录体失败 | `io_error` | `failed to read client record body` |
| 解密客户端记录失败 | `reality_handshake_failed` | `failed to decrypt client record (ec={}), raw {} bytes` |
| 客户端 TLS ALERT | `reality_handshake_failed` | `client sent TLS ALERT: level={}, desc=0x{:02x}` |

#### CCS 循环无迭代限制

`consume_client_finished` 使用 `while (!consumed)` 循环跳过 CCS 记录：

```cpp
while (!consumed) {
    // 读取记录头
    // 读取记录体
    if (rec_content_type == CONTENT_TYPE_CHANGE_CIPHER_SPEC) {
        continue;  // 跳过 CCS，继续循环
    }
    // ... 解密处理 ...
    consumed = true;
}
```

如果客户端持续发送 CCS 记录，服务端将无限循环。TLS 1.3 规范允许中间件插入 CCS 记录（兼容性），但没有限制数量。恶意客户端可利用此点进行 DoS。

#### 客户端 TLS ALERT 解码

当客户端发送 ALERT 而非 Finished 时，表示客户端拒绝了服务端的 Finished 消息。常见 ALERT：

| ALERT 值 | 名称 | 含义 |
|----------|------|------|
| 0x28 (40) | `handshake_failure` | 握手通用失败 |
| 0x2A (42) | `decrypt_error` | Finished verify_data 不匹配 |
| 0x50 (80) | `internal_error` | 客户端内部错误 |

日志格式：`client sent TLS ALERT: level={level}, desc=0x{desc:02x} -- server Finished was rejected`

### Stage 5: 应用密钥派生与内层预读

**涉及步骤**: Step 10（transcript hash + 应用密钥 + seal 创建 + 预读）

| 故障点 | 错误码 | 日志 |
|--------|--------|------|
| derive_application_keys 失败 | 透传 | `failed to derive application keys` |
| 预读内层数据失败 | `to_code(read_ec)` | `failed to read inner data: {msg}` |

#### 记录长度无上限检查

`consume_client_finished` 中读取 TLS 记录时：

```cpp
const auto rec_len = (static_cast<std::size_t>(hdr_raw[3]) << 8) |
                     static_cast<std::size_t>(hdr_raw[4]);
// rec_len 最大 65535 字节
memory::vector<std::byte> rec_body(rec_len);
```

`rec_len` 来自 TLS 记录头的 2 字节长度字段，理论上限 65535 字节。攻击者可以构造一个声称长度为 65535 的记录，迫使服务端分配对应大小的缓冲区。虽然单次分配 64KB 通常不是问题，但在高并发场景下可能放大内存消耗。

---

## seal 加密传输层

握手完成后，`seal` 类接管所有后续的加密读写。seal 继承 `transport::transmission` 接口，封装 TLS 1.3 应用数据记录的加密/解密。日志使用 `[Stealth.Session]` 标签。

### seal 错误条件

| 条件 | 错误码 | 日志 |
|------|--------|------|
| 收到 TLS ALERT (content type 0x15) | `connection_reset` | `received TLS alert record` |
| 非 ApplicationData (content type != 0x17) | `protocol_error` | `unexpected content type: 0x{:02x}` |
| 记录长度 < AEAD_TAG_LEN (16 字节) | `protocol_error` | `record too short: {} bytes` |
| AEAD 解密失败 | `crypto_error` | `AEAD decrypt failed` |
| AEAD 加密失败 | `crypto_error` | `AEAD encrypt failed` |

### Nonce 生成与序列号

```
nonce = IV XOR big_endian(sequence_number)
```

- `read_sequence_` 和 `write_sequence_` 独立递增
- 类型为 `uint64_t`，理论上在 2^64 个记录后溢出导致 nonce 重用
- 实际中不可能达到此上限（按每秒 100 万记录计，需约 584,942 年）

### 内容类型处理

```
收到加密记录
    |
    v
解密后检查最后一个字节 (inner content type)
    |
    +---> 0x17 (ApplicationData) --> 正常处理
    |
    +---> 0x15 (ALERT) --> connection_reset
    |
    +---> 其他 --> protocol_error
```

---

## Executor 层面的故障传播

Reality 作为 Tier 0 方案，在 [[core/stealth/executor]] 中的行为与其他方案不同。

### rewind 不可逆

```
Reality handshake 执行流程:

ensure_snapshot()          ← 保存传输层快照
    |
    v
execute_single(reality)   ← 开始握手
    |
    v
async_write_scatter()     ← 发送 ServerHello + CCS + EncryptedHandshake
    |                         此时 snapshot.wrote_ = true
    v
如果后续步骤失败...
    |
    v
try_rewind()              ← 尝试回滚
    |
    v
wrote_ == true → 无法 rewind!
```

一旦 `async_write_scatter` 将 ServerHello 发送到客户端，传输层的 snapshot 标记 `wrote_` 变为 true。此时即使 Reality 在 Stage 4（消费客户端 Finished）或 Stage 5（应用密钥派生）失败，也**无法回滚传输层状态**。

这意味着：
- 后续方案看到的传输层数据已被 Reality 污染
- 客户端已收到 ServerHello，处于不一致状态
- Executor 只能返回错误

### 确定性命中

当 recognition 模块将连接标记为 Reality（Tier 0）时：

```
execute_by_analysis(analysis, ctx)
    |
    v
解析候选列表 → [reality]  ← 仅一个方案
    |
    v
execute_pipeline([reality], ctx)
    |
    v
ensure_snapshot()
execute_single(reality)
    |
    +---> detected == 具体协议 → 返回成功
    |
    +---> detected == TLS → 不可能，Reality 不会返回 TLS
    |
    +---> 失败 → 返回错误
```

Reality 方案的 `handshake()` 返回 `not_reality` 时，`detected` 类型是 `tls`，这意味着 Reality 认为连接不是自己的。但在确定性命中场景下，recognition 已经确认是 Reality，这种情况不应发生。

如果 Reality 确实失败了（认证通过但后续阶段出错），Executor 的行为：

- 日志：`[SchemeExecutor] Scheme 'reality' failed but cannot rewind`
- 尝试下一个方案（如果有）→ 但 rewind 失败，传输层已污染
- 最终：`[SchemeExecutor] All candidates failed, executing native fallback`

---

## 排障流程

### 从日志反推故障阶段

以下决策树帮助从日志中的 `[Stealth.*]` 标签定位故障阶段。

```
观察到连接失败
    |
    +-- 没有任何 [Stealth.*] 日志？
    |       |
    |       v
    |   可能原因:
    |     1. Reality 未配置 (reality_not_configured)
    |     2. inbound 传输层无效 (io_error)
    |     3. Executor 未选择 Reality 方案
    |
    +-- 有 [Stealth.Handshake] 日志？
            |
            v
        查看"failed to"关键字
            |
            +--- "failed to read TLS record"
            |       |
            |       v
            |   Stage 1, Step 1: TLS 记录读取失败
            |   排查: 网络连通性、客户端是否发送了数据
            |
            +--- "failed to parse ClientHello"
            |       |
            |       v
            |   Stage 1, Step 2: ClientHello 格式错误
            |   后续: 应看到 "falling back to {host}:{port}"
            |         如果看到 → fallback_to_dest 执行中
            |         如果看到 "connect to dest failed" → dest 不可达
            |         如果看到 "write to dest failed" → dest 连接异常
            |
            +--- "invalid private key length"
            |       |
            |       v
            |   Stage 2: 私钥配置错误
            |   排查: 检查 private_key 的 Base64 编码，解码后应为 32 字节
            |
            +--- "SNI mismatch"
            |       |
            |       v
            |   Stage 2, Step 1: SNI 不在 server_names 列表
            |   行为: 返回 not_reality（正常行为，非错误）
            |   排查: 检查客户端 servername 与服务端 server_names 是否一致
            |
            +--- 有 [Stealth.Auth] 日志？
            |       |
            |       v
            |   Stage 2 认证子步骤:
            |       "no X25519 public key"   → Step 2: 客户端未发送 X25519 key_share
            |       "no TLS 1.3 support"     → Step 3: 客户端不支持 TLS 1.3
            |       "session_id too short"   → Step 4: session_id 未正确构造
            |       "X25519 key exchange"    → Step 5: 密钥交换运算失败
            |       "all-zero shared secret" → Step 6: 可能遭受低阶点攻击
            |       "HKDF" 相关              → Step 7-8: HKDF 运算失败
            |       "GCM decrypt failed"     → Step 9: 认证密钥不匹配
            |       "invalid version marker" → Step 10: 协议版本不兼容
            |       "short_id mismatch"      → Step 11: short_id 不在列表中
            |
            +--- "ephemeral X25519 key exchange failed"
            |       |
            |       v
            |   Stage 3: TLS ECDH 密钥交换失败
            |   排查: 客户端公钥格式、服务端临时密钥对生成
            |
            +--- "failed to generate ServerHello"
            |       |
            |       v
            |   Stage 3: ServerHello 构造失败
            |   排查: 密钥材料、证书生成
            |
            +--- "failed to derive keys"
            |       |
            |       v
            |   Stage 3: HKDF 密钥派生失败
            |   排查: shared_secret 值、transcript hash
            |
            +--- "plaintext too short for Finished"
            |       |
            |       v
            |   Stage 3: 加密握手记录明文异常
            |   排查: 合成证书生成、EncryptedExtensions 构造
            |
            +--- "failed to send handshake records"
            |       |
            |       v
            |   Stage 4: scatter-gather 写入失败
            |   排查: socket 状态、网络连通性
            |
            +--- "failed to read client record header/body"
            |       |
            |       v
            |   Stage 4: 等待客户端 Finished 时连接断开
            |   排查: 客户端是否在收到 ServerHello 后断开
            |
            +--- "failed to decrypt client record"
            |       |
            |       v
            |   Stage 4: 客户端记录解密失败
            |   原因: client_handshake_key/iv 不匹配
            |   排查: 密钥派生是否正确、transcript hash
            |
            +--- "client sent TLS ALERT"
            |       |
            |       v
            |   Stage 4: 客户端拒绝了 server Finished
            |   查看 desc 值:
            |     0x28 (handshake_failure) → 握手参数不匹配
            |     0x2A (decrypt_error)     → Finished verify_data 错误
            |     0x50 (internal_error)    → 客户端内部错误
            |   排查: server_finished_key、transcript hash
            |
            +--- "failed to derive application keys"
            |       |
            |       v
            |   Stage 5: 应用密钥派生失败
            |   排查: master_secret、server Finished transcript hash
            |
            +--- "failed to read inner data"
                    |
                    v
                Stage 5: 预读内层数据失败
                排查: seal 解密、client_app_key/iv、客户端是否发送了数据
```

### 日志标签索引

| 日志标签 | 模块 | 输出内容 |
|----------|------|----------|
| `[Stealth.Handshake]` | handshake.cpp | 握手各阶段进度和错误 |
| `[Stealth.Auth]` | auth.cpp | 认证步骤详情 |
| `[Stealth.KeySchedule]` | keygen.cpp | 密钥派生每步详情 |
| `[Stealth.Session]` | seal.cpp | 加密传输层错误 |
| `[SchemeExecutor]` | executor.cpp | 方案选择和执行结果 |

### 常见故障场景与排查

#### 场景 1: 客户端连接超时，服务端无日志

```
可能原因:
  1. Reality 未配置 → 检查配置文件 stealth.reality 段
  2. Executor 未选择 Reality → 检查 recognition 分析结果
  3. inbound 传输层无效 → 检查监听端口和网络

排查步骤:
  1. 确认配置中存在 stealth.reality 段
  2. 确认 private_key 非空
  3. 确认 dest 地址可达: curl -v https://{dest_host}:{dest_port}
```

#### 场景 2: 服务端日志显示 SNI mismatch

```
可能原因:
  1. 客户端 servername 与服务端 server_names 不一致
  2. 客户端未发送 SNI（server_name 为空）

排查步骤:
  1. 检查客户端配置中的 servername / server-name
  2. 检查服务端配置中的 server_names 列表
  3. 确保大小写完全匹配（SNI 比较区分大小写）
```

#### 场景 3: 认证成功但握手失败 (client sent TLS ALERT)

```
可能原因:
  1. server Finished verify_data 计算错误
  2. transcript hash 不一致
  3. 密钥派生中间步骤出错

排查步骤:
  1. 开启 debug 日志级别，查看密钥派生中间值
  2. 对比服务端和客户端的 transcript hash
  3. 检查 server_finished_key 是否正确
  4. 确认合成证书生成未出错
```

#### 场景 4: dest 回退后连接中断

```
可能原因:
  1. dest 服务器不可达
  2. dest 配置格式错误
  3. 裸 socket 泄漏（async_write 失败后未回收）

排查步骤:
  1. 手动测试 dest 可达性: curl -v https://{dest}
  2. 检查 dest 格式是否为 "host:port" 或 "[ipv6]:port"
  3. 检查防火墙规则是否允许出站连接到 dest
```

#### 场景 5: seal 层 connection_reset

```
可能原因:
  1. 客户端发送了 TLS ALERT（如关闭通知）
  2. 中间网络设备注入了 RST
  3. 客户端内部错误后关闭连接

排查步骤:
  1. 查看 ALERT 级别和描述码
  2. 检查是否为正常的连接关闭（level=1, desc=0x00 即 close_notify）
  3. 检查网络中间设备（NAT、防火墙）是否超时
```

---

## 相关文档

- [[dev/debugging/tls]] -- TLS 协议参考与问题排查指南
- [[core/stealth/reality/handshake]] -- Reality 握手状态机源码分析
- [[core/stealth/reality/auth]] -- Reality 认证逻辑
- [[core/stealth/reality/seal]] -- Reality 加密传输层
- [[core/stealth/executor]] -- 伪装方案执行器
- [[core/fault/code]] -- 全局错误码枚举
- [[core/fault/handling]] -- 错误码检查与传播
