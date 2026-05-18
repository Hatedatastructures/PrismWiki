---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/keygen.hpp
title: Reality Keygen
---

# Reality Keygen

TLS 1.3 密钥调度实现。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/keygen.hpp`

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
- [[crypto-hkdf]] ← HKDF 实现
- [[constants]] ← 密钥长度常量