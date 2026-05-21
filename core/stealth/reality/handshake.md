---
layer: core
source: I:/code/Prism/src/prism/stealth/reality/handshake.cpp
title: Reality Handshake
---

# Reality Handshake

Reality 协议核心握手状态机，协调 ClientHello 解析、认证、TLS 1.3 握手和回退逻辑。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/handshake.hpp`
- 实现文件：`I:/code/Prism/src/prism/stealth/reality/handshake.cpp`

## 握手结果

### handshake_result_type

```cpp
enum class handshake_result_type
{
    authenticated,  // Reality 认证成功
    not_reality,    // 非 Reality 客户端
    fallback,       // 回退到 dest 完成
    failed          // 错误
};
```

### handshake_result

```cpp
struct handshake_result
{
    handshake_result_type type;
    channel::transport::shared_transmission encrypted_transport;  // 认证成功时的加密传输层
    memory::vector<std::byte> inner_preread;                       // 内层预读数据
    memory::vector<std::byte> raw_tls_record;                      // 非 Reality 时原始 TLS 记录
    fault::code error;
};
```

---

## Reality 握手五阶段

Reality 握手分为 5 个主要阶段：

| 阶段 | 名称 | 核心操作 |
|------|------|----------|
| Stage 1 | ClientHello 解析 | 读取并解析 TLS ClientHello |
| Stage 2 | Reality 认证 | SNI 匹配、密钥交换、short_id 验证 |
| Stage 3 | TLS 1.3 密钥派生 | ECDH 共享密钥 → 握手密钥 → Finished |
| Stage 4 | 握手消息交换 | ServerHello → CCS → 加密握手记录 |
| Stage 5 | 应用密钥派生 | transcript hash → 应用密钥 → seal |

---

## Stage 1: ClientHello 解析

### 源码逐行解释（行 399-441）

```cpp
auto handshake(channel::transport::shared_transmission inbound,
               const psm::config &cfg,
               psm::agent::session_context &session)
    -> net::awaitable<handshake_result>
{
    handshake_result result;
```

**行 399-403**：函数签名，接收入站传输层、配置和会话上下文，返回异步握手结果。

```cpp
    if (!inbound)
    {
        result.error = fault::code::io_error;
        co_return result;
    }
```

**行 406-410**：验证入站传输层有效性，无效时返回 IO 错误。

```cpp
    const auto &reality_cfg = cfg.stealth.reality;
```

**行 412**：获取 Reality 配置引用，避免后续重复访问。

```cpp
    // 1. 读取 ClientHello
    auto [read_ec, raw_record] = co_await protocol::tls::read_tls_record(*inbound);
    if (fault::failed(read_ec))
    {
        trace::warn("{} failed to read TLS record: {}", HsTag, fault::describe(read_ec));
        result.error = read_ec;
        co_return result;
    }
```

**行 414-421**：
- 调用 [[request#read_tls_record]] 读取完整 TLS 记录
- 失败时记录警告日志并返回错误
- `raw_record` 包含 5 字节 TLS 记录头 + ClientHello 载荷

```cpp
    auto [parse_ec, ch_features] = protocol::tls::parse_client_hello(raw_record);
    if (fault::failed(parse_ec))
    {
        trace::warn("{} failed to parse ClientHello: {}", HsTag, fault::describe(parse_ec));
        const auto fb_ec = co_await fallback_to_dest(session, inbound, raw_record);
        result.type = (fault::succeeded(fb_ec)) ? handshake_result_type::fallback : handshake_result_type::failed;
        result.error = fb_ec;
        co_return result;
    }
```

**行 423-431**：
- 调用 [[request#parse_client_hello]] 解析 ClientHello 提取关键字段
- 解析失败时执行 **回退逻辑**：连接 dest 服务器进行透明代理
- 回退成功返回 `fallback`，失败返回 `failed`

```cpp
    // TODO: 统一 client_hello_features 与 client_hello_info 类型，消除此转换
    client_hello_info client_hello;
    client_hello.server_name = std::move(ch_features.server_name);
    client_hello.session_id = std::move(ch_features.session_id);
    client_hello.has_client_public_key = ch_features.has_x25519;
    client_hello.client_public_key = ch_features.x25519_key;
    client_hello.supported_versions = std::move(ch_features.versions);
    client_hello.random = ch_features.random;
    client_hello.raw_message = std::move(ch_features.raw_hs_msg);

    trace::debug("{} ClientHello parsed, SNI: {}", HsTag, client_hello.server_name);
```

**行 433-441**：
- 将 `client_hello_features` 转换为 `client_hello_info`（TODO 标记需统一类型）
- 提取字段：SNI、session_id、X25519 公钥、支持的 TLS 版本、客户端随机数、原始消息
- 记录调试日志显示 SNI

---

## Stage 2: Reality 认证

### 源码逐行解释（行 445-491）

```cpp
    // 2. 解码私钥
    const auto private_key_str = std::string(reality_cfg.private_key.data(), reality_cfg.private_key.size());
    auto decoded_key_str = crypto::base64_decode(private_key_str);
    if (decoded_key_str.size() != protocol::tls::REALITY_KEY_LEN)
    {
        trace::warn("{} invalid private key length: {}", HsTag, decoded_key_str.size());
        const auto fb_ec = co_await fallback_to_dest(session, inbound, raw_record);
        result.type = (fault::succeeded(fb_ec)) ? handshake_result_type::fallback : handshake_result_type::failed;
        result.error = fb_ec;
        co_return result;
    }
```

**行 445-455**：
- 从配置获取 base64 编码的私钥字符串
- 调用 [[crypto-base64]] 解码为 32 字节原始数据
- 验证私钥长度必须是 32 字节（X25519 密钥长度）
- 私钥无效时执行回退

```cpp
    const auto decoded_private_key = std::span<const std::uint8_t>(
        reinterpret_cast<const std::uint8_t *>(decoded_key_str.data()), decoded_key_str.size());
```

**行 457-458**：将解码后的私钥转换为字节 span，供后续密钥交换使用。

```cpp
    // 3. Reality 认证
    auto [auth_ec, auth_res] = authenticate(reality_cfg, client_hello, decoded_private_key);
    if (!auth_res.authenticated)
    {
```

**行 460-462**：调用 [[auth#authenticate]] 执行认证，检查结果。

```cpp
        // SNI 不匹配 → 标准 TLS
        if (auth_ec == fault::code::reality_sni_mismatch)
        {
            trace::debug("{} SNI mismatch, falling back to standard TLS", HsTag);
            result.type = handshake_result_type::not_reality;
            result.raw_tls_record.assign(
                reinterpret_cast<const std::byte *>(raw_record.data()),
                reinterpret_cast<const std::byte *>(raw_record.data() + raw_record.size()));
            co_return result;
        }
```

**行 464-472**：
- SNI 不匹配表示客户端访问的是其他网站
- 返回 `not_reality`，原始 TLS 记录保留供后续方案处理
- **不执行回退**，交给下一个 stealth 方案

```cpp
        // SNI 为空 + auth 失败 → 非 Reality 客户端
        if (client_hello.server_name.empty())
        {
            trace::debug("{} auth failed with empty SNI, falling back to standard TLS", HsTag);
            result.type = handshake_result_type::not_reality;
            result.raw_tls_record.assign(
                reinterpret_cast<const std::byte *>(raw_record.data()),
                reinterpret_cast<const std::byte *>(raw_record.data() + raw_record.size()));
            co_return result;
        }
```

**行 474-483**：
- SNI 为空说明客户端未发送 server_name 扩展
- 非 Reality 客户端可能不发送 SNI
- 同样返回 `not_reality`

```cpp
        // SNI 匹配但 auth 失败 → 非 Reality 客户端，交给下一个方案处理
        trace::debug("{} auth failed: {}, not Reality, passing to next scheme", HsTag, fault::describe(auth_ec));
        result.type = handshake_result_type::not_reality;
        result.raw_tls_record.assign(
            reinterpret_cast<const std::byte *>(raw_record.data()),
            reinterpret_cast<const std::byte *>(raw_record.data() + raw_record.size()));
        co_return result;
    }
```

**行 485-491**：
- SNI 匹配但认证失败（如 short_id 不匹配、密钥错误）
- 返回 `not_reality`，可能是普通 TLS 客户端
- 交给下一个 stealth 方案（如 ShadowTLS）处理

```cpp
    trace::info("{} authentication successful", HsTag);
```

**行 493**：认证成功，记录信息日志。

---

## Stage 3: TLS 1.3 密钥派生

### 源码逐行解释（行 495-548）

```cpp
    // 4. TLS ECDH 密钥交换
    // 注意：不为认证客户端获取 dest 证书 — generate_reality_certificate() 会生成合成 ed25519 证书
    auto [ephemeral_ec, tls_shared_secret] = crypto::x25519(
        std::span<const std::uint8_t>(auth_res.server_ephemeral_key.private_key.data(),
                                      auth_res.server_ephemeral_key.private_key.size()),
        std::span<const std::uint8_t>(client_hello.client_public_key.data(),
                                      client_hello.client_public_key.size()));
    if (fault::failed(ephemeral_ec))
    {
        trace::warn("{} ephemeral X25519 key exchange failed", HsTag);
        result.error = ephemeral_ec;
        co_return result;
    }
```

**行 495-507**：
- 使用服务端临时私钥 + 客户端公钥进行 X25519 密钥交换
- 注意注释：Reality 认证客户端不需要 dest 证书，使用合成 Ed25519 证书
- `tls_shared_secret` 是 TLS 1.3 ECDH 的共享密钥（32 字节）

```cpp
    // 诊断日志：TLS ECDH 共享密钥
    trace::debug("{} TLS ECDH shared_secret: {}", HsTag,
                 format_hex_short({tls_shared_secret.data(), tls_shared_secret.size()}));
```

**行 509-511**：诊断日志，显示共享密钥的前 16 字节 hex。

```cpp
    // 5. 生成 ServerHello + 派生握手密钥
    key_material dummy_keys{};
    auto [sh_ec, sh_result] = generate_server_hello(
        client_hello,
        auth_res.server_ephemeral_key.public_key,
        dummy_keys,
        {}, // 不需要 dest 证书 — Reality 认证客户端使用合成证书
        client_hello.raw_message,
        std::span<const std::uint8_t>(auth_res.auth_key.data(), auth_res.auth_key.size()));
```

**行 513-522**：
- 调用 [[response#generate_server_hello]] 生成 ServerHello
- 使用 dummy_keys（空密钥材料），后续会重算 Finished
- dest_certificate 为空（使用合成证书）
- auth_key 用于合成证书的 Ed25519 签名

```cpp
    if (fault::failed(sh_ec))
    {
        trace::warn("{} failed to generate ServerHello: {}", HsTag, fault::describe(sh_ec));
        result.error = sh_ec;
        co_return result;
    }
```

**行 524-528**：ServerHello 生成失败时返回错误。

```cpp
    auto [ks_ec, keys] = derive_handshake_keys(
        tls_shared_secret,
        client_hello.raw_message,
        sh_result.server_hello_msg);

    if (fault::failed(ks_ec))
    {
        trace::warn("{} failed to derive keys: {}", HsTag, fault::describe(ks_ec));
        result.error = ks_ec;
        co_return result;
    }
```

**行 530-540**：
- 调用 [[keygen#derive_handshake_keys]] 派生握手密钥
- 输入：ECDH 共享密钥、ClientHello、ServerHello
- 输出：握手阶段密钥（server_handshake_key/iv、client_handshake_key/iv、finished_key）

```cpp
    // 6. 用正确密钥重算 Finished
    const auto finished_ec = derive_and_encrypt_finished(keys, sh_result, client_hello.raw_message);
    if (fault::failed(finished_ec))
    {
        result.error = finished_ec;
        co_return result;
    }
```

**行 542-548**：
- 调用内部函数 `derive_and_encrypt_finished` 重算 Finished
- 原因：generate_server_hello 使用了 dummy_keys，Finished 需用正确密钥重算

### derive_and_encrypt_finished 详解（行 245-300）

```cpp
static auto derive_and_encrypt_finished(
    const key_material &keys,
    server_hello_result &sh_result,
    std::span<const std::uint8_t> client_hello_raw_msg)
    -> fault::code
{
    constexpr std::size_t FINISHED_MSG_SIZE = 36; // Type(1) + Length(3) + verify_data(32)
    const auto &old_plaintext = sh_result.encrypted_handshake_plaintext;

    if (old_plaintext.size() < FINISHED_MSG_SIZE)
    {
        trace::warn("{} plaintext too short for Finished: {}", HsTag, old_plaintext.size());
        return fault::code::reality_key_schedule_error;
    }
```

**行 245-258**：
- Finished 消息结构：Type(1) + Length(3) + verify_data(32) = 36 字节
- 验证明文长度

```cpp
    const auto ee_cert_cv = std::span<const std::uint8_t>(
        old_plaintext.data(), old_plaintext.size() - FINISHED_MSG_SIZE);
```

**行 260-262**：提取 EncryptedExtensions + Certificate + CertificateVerify 部分（不包括旧 Finished）。

```cpp
    const auto transcript_for_finished = crypto::sha256(
        client_hello_raw_msg,
        sh_result.server_hello_msg,
        ee_cert_cv);
```

**行 264-267**：
- 计算到 Finished 的 transcript hash
- 输入：ClientHello + ServerHello + (EE + Cert + CertVerify)
- 使用 SHA256

```cpp
    const auto verify_data = compute_finished_verify_data(
        keys.server_finished_key, transcript_for_finished);
```

**行 268-269**：调用 [[keygen#compute_finished_verify_data]] 计算 verify_data。

```cpp
    // 诊断日志：Finished 计算
    trace::debug("{} server Finished transcript: {}", HsTag,
                 format_hex_short({transcript_for_finished.data(), transcript_for_finished.size()}));
    trace::debug("{} server Finished verify_data: {}", HsTag,
                 format_hex_short({verify_data.data(), verify_data.size()}));
```

**行 271-276**：诊断日志，显示 transcript 和 verify_data。

```cpp
    memory::vector<std::uint8_t> correct_plaintext(ee_cert_cv.begin(), ee_cert_cv.end());
    correct_plaintext.push_back(protocol::tls::HANDSHAKE_TYPE_FINISHED);
    correct_plaintext.push_back(0x00);
    correct_plaintext.push_back(0x00);
    correct_plaintext.push_back(static_cast<std::uint8_t>(verify_data.size()));
    correct_plaintext.insert(correct_plaintext.end(), verify_data.begin(), verify_data.end());
```

**行 277-282**：构造正确的 Finished 明文（Type + Length + verify_data）。

```cpp
    auto [enc_ec, encrypted_record] = encrypt_tls_record(
        keys.server_handshake_key,
        keys.server_handshake_iv,
        0,
        protocol::tls::CONTENT_TYPE_HANDSHAKE,
        correct_plaintext);
```

**行 284-289**：
- 调用 [[response#encrypt_tls_record]] 加密握手记录
- 使用正确的握手密钥和 IV
- 序列号为 0（第一个加密记录）

```cpp
    if (fault::failed(enc_ec))
    {
        trace::warn("{} failed to encrypt handshake record", HsTag);
        return enc_ec;
    }

    sh_result.encrypted_handshake_plaintext = std::move(correct_plaintext);
    sh_result.encrypted_handshake_record = std::move(encrypted_record);
    return fault::code::success;
}
```

**行 291-300**：更新 sh_result 的明文和加密记录。

---

## Stage 4: 握手消息交换

### 源码逐行解释（行 550-579）

```cpp
    // 7. 发送握手记录（scatter-gather）
    {
        std::error_code write_ec;
        const std::span<const std::byte> handshake_parts[] = {
            std::span<const std::byte>(
                reinterpret_cast<const std::byte *>(sh_result.server_hello_record.data()),
                sh_result.server_hello_record.size()),
            std::span<const std::byte>(
                reinterpret_cast<const std::byte *>(sh_result.change_cipher_spec_record.data()),
                sh_result.change_cipher_spec_record.size()),
            std::span<const std::byte>(
                reinterpret_cast<const std::byte *>(sh_result.encrypted_handshake_record.data()),
                sh_result.encrypted_handshake_record.size()),
        };
        co_await inbound->async_write_scatter(handshake_parts, 3, write_ec);
        if (write_ec)
        {
            trace::warn("{} failed to send handshake records: {}", HsTag, write_ec.message());
            result.error = fault::to_code(write_ec);
            co_return result;
        }
    }
```

**行 550-571**：
- 使用 scatter-gather 写入三个握手记录：
  1. ServerHello TLS 记录
  2. ChangeCipherSpec 兼容性记录
  3. 加密的握手记录（EE + Cert + CertVerify + Finished）
- 一次写入减少系统调用开销

```cpp
    // 8. 消费客户端 CCS + Finished
    const auto consumed_ec = co_await consume_client_finished(*inbound, keys);
    if (fault::failed(consumed_ec))
    {
        result.error = consumed_ec;
        co_return result;
    }
```

**行 573-579**：
- 调用 `consume_client_finished` 等待并处理客户端的 CCS + Finished
- 失败时返回错误

### consume_client_finished 详解（行 303-393）

```cpp
static auto consume_client_finished(
    channel::transport::transmission &inbound,
    const key_material &keys)
    -> net::awaitable<fault::code>
{
    bool consumed = false;
    while (!consumed)
    {
```

**行 303-309**：循环读取客户端记录直到消费到 Finished。

```cpp
        std::array<std::byte, protocol::tls::RECORD_HEADER_LEN> rec_hdr{};
        if (!co_await read_exact(inbound, rec_hdr))
        {
            trace::warn("{} failed to read client record header", HsTag);
            co_return fault::code::io_error;
        }

        const auto *hdr_raw = reinterpret_cast<const std::uint8_t *>(rec_hdr.data());
        const auto rec_content_type = hdr_raw[0];
        const auto rec_len = (static_cast<std::size_t>(hdr_raw[3]) << 8) |
                             static_cast<std::size_t>(hdr_raw[4]);
```

**行 311-321**：
- 读取 5 字节 TLS 记录头
- 解析 Content Type 和长度

```cpp
        trace::debug("{} client rec: type=0x{:02x} len={}", HsTag,
                     static_cast<unsigned>(rec_content_type), rec_len);

        memory::vector<std::byte> rec_body(rec_len);
        if (rec_len > 0 && !co_await read_exact(inbound, rec_body))
        {
            trace::warn("{} failed to read client record body", HsTag);
            co_return fault::code::io_error;
        }
```

**行 323-331**：读取记录体。

```cpp
        if (rec_content_type == protocol::tls::CONTENT_TYPE_CHANGE_CIPHER_SPEC)
        {
            trace::debug("{} skipping client CCS record", HsTag);
            continue;
        }
```

**行 333-337**：CCS 记录跳过（TLS 1.3 兼容性记录）。

```cpp
        // 非 CCS 记录：尝试解密以判断是 Finished 还是 Alert
        {
            std::array<std::uint8_t, protocol::tls::AEAD_NONCE_LEN> client_nonce{};
            std::memcpy(client_nonce.data(), keys.client_handshake_iv.data(), protocol::tls::AEAD_NONCE_LEN);
```

**行 339-342**：准备 nonce（复制 IV，序列号为 0）。

```cpp
            const auto ad_span = std::span<const std::uint8_t>(
                reinterpret_cast<const std::uint8_t *>(rec_hdr.data()), rec_hdr.size());
            const auto ct_span = std::span<const std::uint8_t>(
                reinterpret_cast<const std::uint8_t *>(rec_body.data()), rec_body.size());
```

**行 344-347**：准备 AAD（记录头）和密文。

```cpp
            crypto::aead_context client_aead(
                crypto::aead_cipher::aes_128_gcm,
                std::span<const std::uint8_t>(keys.client_handshake_key.data(), keys.client_handshake_key.size()));

            const auto pt_size = crypto::aead_context::open_output_size(rec_body.size());
            memory::vector<std::uint8_t> decrypted(pt_size);
            const auto nonce_span = std::span<const std::uint8_t>(client_nonce.data(), client_nonce.size());

            const auto open_ec = client_aead.open(decrypted, ct_span, nonce_span, ad_span);
```

**行 349-357**：
- 创建客户端 AEAD 解密上下文
- 解密记录体

```cpp
            if (!fault::failed(open_ec) && decrypted.size() >= 2)
            {
                const auto inner_content_type = decrypted.back();
                if (inner_content_type == protocol::tls::CONTENT_TYPE_ALERT && decrypted.size() >= 3)
                {
                    trace::error("{} client sent TLS ALERT: level={}, desc=0x{:02x} — server Finished was rejected",
                                 HsTag,
                                 static_cast<unsigned>(decrypted[0]),
                                 static_cast<unsigned>(decrypted[1]));
                    co_return fault::code::reality_handshake_failed;
                }
```

**行 359-369**：
- 解密成功后检查 inner content type（最后一字节）
- ALERT 表示客户端拒绝了 server Finished
- 记录错误日志并返回握手失败

```cpp
                else
                {
                    trace::debug("{} consumed client Finished record ({} bytes, inner_type=0x{:02x})",
                                 HsTag, rec_len, static_cast<unsigned>(inner_content_type));
                    // 诊断日志：客户端 Finished verify_data
                    if (inner_content_type == protocol::tls::CONTENT_TYPE_HANDSHAKE && decrypted.size() >= 36)
                    {
                        trace::info("{} client Finished verify_data: {}", HsTag,
                                    format_hex_short({decrypted.data() + 4, 32}));
                    }
                }
            }
            else
            {
                trace::warn("{} failed to decrypt client record (ec={}), raw {} bytes",
                            HsTag, static_cast<int>(open_ec), rec_len);
                co_return fault::code::reality_handshake_failed;
            }
        }
        consumed = true;
    }

    co_return fault::code::success;
}
```

**行 370-393**：
- 正常情况：消费到 Finished，记录 verify_data
- 解密失败：返回握手失败
- 标记 consumed=true 结束循环

---

## Stage 5: 应用密钥派生

### 源码逐行解释（行 581-631）

```cpp
    // 9. 派生应用数据密钥
    const auto full_transcript_hash = crypto::sha256(
        {client_hello.raw_message.data(), client_hello.raw_message.size()},
        {sh_result.server_hello_msg.data(), sh_result.server_hello_msg.size()},
        {sh_result.encrypted_handshake_plaintext.data(), sh_result.encrypted_handshake_plaintext.size()});
```

**行 581-586**：
- 计算完整 transcript hash（到 server Finished）
- 输入：ClientHello + ServerHello + 加密握手记录明文

```cpp
    const auto app_ec = derive_application_keys(keys.master_secret,
                                                {full_transcript_hash.data(), full_transcript_hash.size()}, keys);
    if (fault::failed(app_ec))
    {
        trace::warn("{} failed to derive application keys", HsTag);
        result.error = app_ec;
        co_return result;
    }
```

**行 587-594**：
- 调用 [[keygen#derive_application_keys]] 派生应用密钥
- 输入：master_secret + transcript hash
- 输出：server_app_key/iv、client_app_key/iv

```cpp
    // 诊断日志：应用密钥
    trace::debug("{} app transcript hash: {}", HsTag,
                 format_hex_short({full_transcript_hash.data(), full_transcript_hash.size()}));
    trace::debug("{} server_app_key: {}", HsTag,
                 format_hex_short({keys.server_app_key.data(), keys.server_app_key.size()}));
    trace::debug("{} server_app_iv: {}", HsTag,
                 format_hex_short({keys.server_app_iv.data(), keys.server_app_iv.size()}));
    trace::debug("{} client_app_key: {}", HsTag,
                 format_hex_short({keys.client_app_key.data(), keys.client_app_key.size()}));
    trace::debug("{} client_app_iv: {}", HsTag,
                 format_hex_short({keys.client_app_iv.data(), keys.client_app_iv.size()}));
```

**行 596-607**：诊断日志，显示所有应用密钥参数。

```cpp
    // 10. 创建加密传输层 + 预读内层数据
    auto reality_session = std::make_shared<seal>(
        std::move(inbound), std::move(keys));
```

**行 608-610**：
- 创建 [[seal]] 加密传输层
- 使用应用密钥初始化

```cpp
    constexpr std::size_t preread_size = 64;
    memory::vector<std::byte> inner_buf(preread_size);
    std::error_code read_inner_ec;
    const auto inner_n = co_await reality_session->async_read_some(
        std::span<std::byte>(inner_buf.data(), preread_size), read_inner_ec);

    if (read_inner_ec || inner_n == 0)
    {
        trace::warn("{} failed to read inner data: {}", HsTag, read_inner_ec.message());
        result.error = fault::to_code(read_inner_ec);
        co_return result;
    }
```

**行 612-623**：
- 预读 64 字节内层数据（用于协议检测）
- seal 使用 client_app_key 解密应用数据

```cpp
    result.type = handshake_result_type::authenticated;
    result.encrypted_transport = std::move(reality_session);
    result.inner_preread.assign(inner_buf.begin(), inner_buf.begin() + static_cast<std::ptrdiff_t>(inner_n));
    result.error = fault::code::success;

    trace::info("{} handshake completed successfully", HsTag);
    co_return result;
}
```

**行 625-631**：
- 设置返回结果：authenticated、加密传输层、内层预读数据
- 记录成功日志

---

## 回退逻辑

### fallback_to_dest（行 122-164）

```cpp
auto fallback_to_dest(psm::agent::session_context &session,
                      channel::transport::shared_transmission inbound,
                      const std::span<const std::uint8_t> raw_record)
    -> net::awaitable<fault::code>
{
    const auto &reality_cfg = session.server.config().stealth.reality;

    std::string dest_host;
    std::uint16_t dest_port = 443;
    if (!parse_dest(std::string_view(reality_cfg.dest.data(), reality_cfg.dest.size()), dest_host, dest_port))
    {
        trace::error("{} invalid dest config: {}", HsTag, reality_cfg.dest);
        co_return fault::code::reality_dest_unreachable;
    }
```

**行 122-135**：
- 解析 dest 配置获取目标 host 和 port
- 失败时返回 dest_unreachable

```cpp
    trace::info("{} falling back to {}:{}", HsTag, dest_host, dest_port);

    char dest_port_buf[8];
    const auto [dest_port_end, dest_port_ec] = std::to_chars(dest_port_buf, dest_port_buf + sizeof(dest_port_buf), dest_port);
    auto [connect_ec, dest_conn] = co_await session.worker.router.async_forward(dest_host, std::string_view(dest_port_buf, std::distance(dest_port_buf, dest_port_end)));
    if (fault::failed(connect_ec) || !dest_conn.valid())
    {
        trace::warn("{} connect to dest failed: {}", HsTag, fault::describe(connect_ec));
        co_return fault::code::reality_dest_unreachable;
    }
```

**行 137-146**：
- 通过 router 连接目标服务器
- 失败时返回 dest_unreachable

```cpp
    auto *dest_socket_raw = dest_conn.release();

    boost::system::error_code write_ec;
    co_await net::async_write(*dest_socket_raw, net::buffer(raw_record.data(), raw_record.size()),
                              net::redirect_error(net::use_awaitable, write_ec));
    if (write_ec)
    {
        trace::warn("{} write to dest failed: {}", HsTag, write_ec.message());
        co_return fault::code::reality_dest_unreachable;
    }
```

**行 148-157**：
- 将原始 TLS 记录转发给目标服务器
- 这是 ClientHello，目标服务器会继续 TLS 握手

```cpp
    auto dest_trans = channel::transport::make_reliable(std::move(*dest_socket_raw));
    co_await pipeline::primitives::tunnel(inbound, std::move(dest_trans), session);

    trace::debug("{} fallback tunnel completed", HsTag);
    co_return fault::code::success;
}
```

**行 159-164**：
- 创建传输层并开始双向隧道
- 透明代理完成

### fetch_dest_certificate（行 170-238）

```cpp
auto fetch_dest_certificate(const std::string_view host, const std::uint16_t port, resolve::router &router)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>
```

从 dest 服务器获取证书（用于 Reality 伪造）。

流程：
1. 连接目标服务器
2. 建立 TLS 连接（不验证证书）
3. 提取 peer certificate
4. 转换为 DER 格式
5. 返回证书数据

---

## 工具函数

### parse_dest（行 73-116）

解析 "host:port" 格式的 dest 配置。

支持格式：
- `host:port`（IPv4 或域名）
- `[ipv6]:port`（IPv6）

### read_exact（行 53-67）

```cpp
static auto read_exact(channel::transport::transmission &transport, std::span<std::byte> buf)
    -> net::awaitable<bool>
```

精确读取指定字节数，循环直到缓冲区满。

### format_hex_short（行 35-47）

格式化字节为 hex 字符串（最多 16 字节），用于诊断日志。

---

## 调用链

- [[scheme]] ← 方案类调用 handshake
- [[request]] ← ClientHello 解析
- [[auth]] ← Reality 认证
- [[keygen]] ← 密钥派生
- [[response]] ← ServerHello 生成
- [[seal]] ← 加密传输层
- [[constants]] ← TLS 常量
- [[crypto-x25519]] ← 密钥交换
- [[crypto-aead]] ← AEAD 加密
- [[crypto-hkdf]] ← 密钥派生
- [[crypto-base64]] ← 私钥解码
- [[transmission]] ← 传输层接口
- [[pipeline-primitives]] ← tunnel 透明代理

## 故障模式

### 五阶段失败条件速查表

| 阶段 | 失败条件 | 错误码 | 行为 |
|------|----------|--------|------|
| Stage 1 | TLS 记录读取失败 | io_error | 返回错误 |
| Stage 1 | ClientHello 解析失败 | parse_error | fallback_to_dest |
| Stage 1 | fallback dest 不可达 | reality_dest_unreachable | 返回错误 |
| Stage 2 | 私钥解码失败（长度 ≠ 32） | - | fallback_to_dest |
| Stage 2 | SNI 不匹配 | reality_sni_mismatch | not_reality, 交下一方案 |
| Stage 2 | X25519 密钥交换失败 | reality_key_exchange_failed | 返回错误 |
| Stage 2 | short_id 不匹配 | - | not_reality, 交下一方案 |
| Stage 3 | ServerHello 生成失败 | reality_key_schedule_error | 返回错误 |
| Stage 3 | 密钥派生失败（9 个 HKDF 步骤中任一） | reality_key_schedule_error | 返回错误 |
| Stage 4 | 发送握手记录失败 | io_error | 返回错误 |
| Stage 4 | CCS 循环无迭代限制 | - | 资源耗尽向量 |
| Stage 4 | 客户端 TLS ALERT | reality_handshake_failed | 握手被拒绝 |
| Stage 4 | 客户端 Finished 解密失败 | reality_handshake_failed | 返回错误 |
| Stage 5 | 应用密钥派生失败 | reality_key_schedule_error | 返回错误 |
| Stage 5 | 预读内层失败 | io_error | 返回错误 |

### 资源泄漏向量

- **fallback_to_dest 裸 socket 泄漏**：`dest_conn.release()` 后如果 `async_write` 失败，`dest_socket_raw` 已 release 无法自动回收（handshake.cpp 行 148-157）
- **consume_client_finished CCS 循环**：while 循环无最大迭代限制，恶意客户端可发送无限 CCS 记录（handshake.cpp 行 288-366）
- **read_encrypted_record 无长度上限**：`record_len` 最大 65535 字节，无额外检查（seal.cpp 行 153-156）

### short_id 认证要点

- 纯密码学比对（HKDF + AES-256-GCM），**不涉及时钟/时间窗口**
- short_id 无过期机制，需通过配置轮换管理
- 合成 Ed25519 证书有效期仅 1 小时

详见 [[dev/debugging/deep-dive/reality-handshake|Reality 握手深层故障分析]]