---
layer: core
source: include/prism/stealth/reality/keygen.hpp
title: Reality Keygen
tags:
  - stealth
  - reality
  - keygen
  - TLS-1.3
  - HKDF
---

# Reality Keygen

TLS 1.3 密钥调度实现。

## 源码位置

- 头文件：`include/prism/stealth/reality/keygen.hpp`

## 设计决策（WHY）

### 为什么密钥调度与标准 TLS 1.3 完全一致

Reality 的核心设计是**伪装成正常 TLS 1.3 握手**。如果密钥调度与标准不同，客户端的 TLS 库会立即检测到异常。完全一致的密钥调度意味着 Reality 握手在中间人看来就是一次普通的 TLS 1.3 连接。唯一的差异在于共享密钥的来源——标准 TLS 使用证书认证的 ECDH，Reality 使用预共享的 X25519 静态密钥。

### 为什么 `key_material` 包含 `master_secret`

`master_secret` 是从 `Handshake Secret` 派生 `Master Secret` 的中间值。虽然 TLS 1.3 规范中 `Master Secret` 不直接用于加密，但 Reality 需要它来进一步派生应用阶段密钥（`derive_application_keys` 的输入）。将其存储在 `key_material` 中避免了重新计算。

### 为什么使用 AES-128-GCM 而非 AES-256-GCM

TLS 1.3 默认密码套件 `TLS_AES_128_GCM_SHA256` 使用 128 位密钥。Reality 模拟此套件以保持握手外观一致性。128 位密钥对性能更友好（AES-NI 硬件加速下 128 位比 256 位快约 30%）。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `shared_secret` 必须 32 字节 | X25519 输出 | HKDF-Extract 输入长度不匹配会改变派生结果 |
| `transcript_hash` 必须是完整握手消息的 SHA256 | TLS 1.3 规范 | 缺失任何握手消息会导致密钥不一致 |
| 密钥长度固定：key=16B, iv=12B, finished_key=32B | TLS 1.3 密码套件 | 不支持其他长度 |
| `derive_handshake_keys` 和 `derive_application_keys` 必须按顺序调用 | TLS 1.3 密钥层级 | 跳过握手密钥直接派生应用密钥会产生错误结果 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| HKDF-Extract 失败 | BoringSSL 内部错误 | 返回 `reality_key_schedule_error` |
| transcript hash 不一致 | ClientHello/ServerHello 字节被修改 | 派生密钥错误，AEAD 解密必然失败 |
| X25519 共享密钥全零 | 低概率事件（特殊输入点） | 密钥调度产生弱密钥 |
| `key_material` 未初始化 | 零值 key_material 传入 `generate_server_hello` | 产生无效的加密记录 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `keygen` → `crypto::hkdf` | 调用 | HKDF-Extract 和 HKDF-Expand-Label 实现 |
| `keygen` → `crypto::sha256` | 调用 | transcript hash 计算 |
| `keygen` → `crypto::aead` | 间接 | 密钥材料传给 AEAD 进行加密/解密 |
| `handshake` → `keygen` | 调用 | Stage 3 和 Stage 5 派生密钥 |
| `seal` → `key_material` | 使用 | seal 的构造函数接收完整的 key_material |
| `response` → `key_material` | 使用 | 加密握手记录使用 handshake_key |

## 密钥材料

### key_material

```cpp
struct key_material
{
    std::array<std::uint8_t, 16> server_handshake_key{};
    std::array<std::uint8_t, 12> server_handshake_iv{};
    std::array<std::uint8_t, 16> client_handshake_key{};
    std::array<std::uint8_t, 12> client_handshake_iv{};
    std::array<std::uint8_t, 16> server_app_key{};
    std::array<std::uint8_t, 12> server_app_iv{};
    std::array<std::uint8_t, 16> client_app_key{};
    std::array<std::uint8_t, 12> client_app_iv{};
    std::array<std::uint8_t, 32> server_finished_key{};
    std::array<std::uint8_t, 32> master_secret{};
};
```

| 字段 | 说明 |
|------|------|
| `server_handshake_key/iv` | 握手阶段服务端加密 |
| `client_handshake_key/iv` | 握手阶段客户端加密 |
| `server_app_key/iv` | 应用阶段服务端加密 |
| `client_app_key/iv` | 应用阶段客户端加密 |
| `server_finished_key` | Finished 验证密钥 |
| `master_secret` | 主密钥 |

## 密钥派生

### derive_handshake_keys

```cpp
[[nodiscard]] auto derive_handshake_keys(constspan shared_secret, constspan client_hello_msg, constspan server_hello_msg)
    -> std::pair<fault::code, key_material>
```

从共享密钥和握手消息派生握手流量密钥。

#### 派生流程（RFC 8446 Section 7）

```
Early Secret = HKDF-Extract(salt: 0, IKM: 0)
Handshake Secret = HKDF-Extract(salt: Early Secret, IKM: shared_secret)
Master Secret = HKDF-Extract(salt: Handshake Secret, IKM: 0)

server_handshake_key = HKDF-Expand-Label(Handshake Secret, "s hs traffic", transcript_hash, 16)
client_handshake_key = HKDF-Expand-Label(Handshake Secret, "c hs traffic", transcript_hash, 16)
server_finished_key = HKDF-Expand-Label(Handshake Secret, "finished", "", 32)
```

### derive_application_keys

```cpp
[[nodiscard]] auto derive_application_keys(constspan master_secret, constspan server_finished_hash, key_material &keys) -> fault::code
```

从主密钥派生应用流量密钥。

```
server_app_key = HKDF-Expand-Label(Master Secret, "s ap traffic", server_finished_hash, 16)
client_app_key = HKDF-Expand-Label(Master Secret, "c ap traffic", server_finished_hash, 16)
```

### compute_finished_verify_data

```cpp
[[nodiscard]] auto compute_finished_verify_data(constspan finished_key, constspan transcript_hash)
    -> std::array<std::uint8_t, 32>
```

计算 Finished 消息的 verify_data。

```
verify_data = HMAC(finished_key, transcript_hash)
```

## 密钥调度图

```
                0
                |
                v
          Early Secret
                |
                v
     HKDF-Extract(shared_secret)
                |
                v
       Handshake Secret
                |
    +-----------+-----------+
    |                       |
    v                       v
server_hs_traffic       client_hs_traffic
    |                       |
    v                       v
server_finished_key    (client_finished)
    |
    +---- HKDF-Extract(0) ----+
    |                        |
    v                        v
Master Secret           (server Finished)
    |
    +-----------+-----------+
    |                       |
    v                       v
server_app_traffic     client_app_traffic
```

## 与标准 TLS 1.3 的差异

Reality 密钥调度与标准 TLS 1.3 **完全一致**：

- 使用相同的 HKDF 函数
- 使用相同的标签名称
- 仅共享密钥来源不同（Reality 使用自定义 X25519）

## 调用链

- [[handshake]] ← 调用密钥派生
- [[seal]] ← 使用密钥材料
- [[response]] ← 使用 finished_key
- [[core/crypto/hkdf|crypto-hkdf]] ← HKDF 实现
- [[constants]] ← 密钥长度常量