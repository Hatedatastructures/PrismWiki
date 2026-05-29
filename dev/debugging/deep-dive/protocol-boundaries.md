---
title: 代理协议实现边界与认证深层分析
source:
  - src/prism/protocol/shadowsocks/conn.cpp
  - src/prism/protocol/trojan/conn.cpp
  - src/prism/protocol/vless/conn.cpp
  - src/prism/protocol/socks5/conn.cpp
  - src/prism/crypto/aead.cpp
module: debugging
type: deep-dive
tags: [debugging, protocol, authentication, ss2022, trojan, vless, socks5, failure-analysis, implementation-boundary]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[dev/debugging/protocol]]"
  - "[[core/protocol/shadowsocks/relay]]"
  - "[[core/protocol/trojan/format]]"
  - "[[core/protocol/vless/relay]]"
  - "[[core/protocol/socks5/stream]]"
  - "[[core/fault/code]]"
---

# 代理协议实现边界与认证深层分析

## 概述

协议规范（RFC、草案、社区标准）描述的是"理想行为"，而代理软件的实际实现存在大量边界条件、半初始化状态、资源泄漏路径和静默失败场景。**规范与实现之间的差异**正是深层排错的关键切入点。

本篇从四个维度展开分析：

| 维度 | 说明 | 典型表现 |
|------|------|----------|
| **时间维度** | 时钟偏移、timestamp 窗口 | SS2022 `timestamp_expired`，握手成功但连接立即断开 |
| **状态维度** | 对象半初始化、空指针解引用 | PSK 解码失败后 `decrypt_ctx_` 为空，跳过 handshake 直接读数据 |
| **缓冲区边界维度** | 精确分配、零余量、截断 | Trojan 域名长度 255 时 `required_total` 恰好等于缓冲区大小 |
| **资源维度** | 内存线性增长、session 泄漏 | SS2022 `salt_pool` 无上限、UDP session_tracker 无效条目累积 |

错误码到根因的映射往往不是一对一的关系。例如 `crypto_error` 可能源于：

- nonce 不同步（TCP 流中一端递增，另一端未同步）
- 密钥错误（PSK 配置不匹配）
- 数据损坏（网络传输中间层篡改或截断）
- AEAD authentication tag 不匹配（数据被篡改或偏移量错误）

---

## SS2022 (Shadowsocks 2022) 深层分析

### timestamp_window 窄窗口

SS2022 协议在 `read_fixed_header` 阶段对客户端携带的 timestamp 进行校验：

```
diff = abs(client_ts - now)
if diff > config_.timestamp_window:
    → timestamp_expired
```

默认 `timestamp_window` 为 **30 秒**。

**窗口过窄（如 1s）的后果：**

- 客户端与服务端之间存在时钟漂移时，大量合法连接被拒绝
- 跨洲际链路中网络延迟叠加两端时钟差，极易超出 1s 窗口
- 表现为间歇性连接失败，日志中大量 `timestamp_expired`

**窗口过宽（如 3600s）的后果：**

- 重放攻击窗口变大，但 `salt_pool` 兜底（同一 salt 不可重用）
- 安全性依赖 salt 的唯一性而非 timestamp 窄窗口

**日志模板：**

```
[SS2022.Relay] timestamp expired: client_ts=1716259200, server_ts=1716259235, diff=35s
[SS2022.Relay] timestamp window check failed, dropping connection from 203.0.113.5:54321
```

**排障步骤：**

```bash
# 检查两端时间同步状态
timedatectl status
ntpstat

# 手动对比两端时间戳（误差应小于 timestamp_window）
date +%s    # 客户端
date +%s    # 服务端
```

### PSK 半初始化风险

在 `conn` 构造函数中，PSK（Pre-Shared Key）从 base64 解码后存储。如果解码失败：

- 只打印日志，**不抛出异常**
- `decrypt_ctx_` 保持为空的 `unique_ptr`
- 正常路径不会触发此问题——`handshake` 函数会先检查 `psk_.empty()` 并返回错误

**但存在防御性编程缺陷：**

如果由于某种代码路径跳过了 `handshake` 阶段，直接调用 `async_read_some`，则会对空的 `decrypt_ctx_` 进行解引用，导致 **segfault（段错误）** 而非优雅的错误返回。

```
// 危险路径示意：
conn(nullptr, config)  // PSK 解码失败，decrypt_ctx_ = nullptr
  → async_read_some()  // 跳过 handshake
    → decrypt_ctx_->process()  // NULL dereference → SIGSEGV
```

**诊断特征：**

| 现象 | 根因 | 日志关键词 |
|------|------|------------|
| 进程 crash，SIGSEGV | `decrypt_ctx_` 空指针解引用 | 无日志（crash 前无输出） |
| handshake 阶段返回错误 | `psk_.empty()` 检查生效 | `empty PSK` / `invalid PSK` |
| base64 解码失败 | PSK 配置格式错误 | `base64 decode failed` |

### salt_pool 内存线性增长

SS2022 使用 `salt_pool` 防止 salt 重放攻击，数据结构为：

```cpp
std::unordered_map<std::string, std::chrono::steady_clock::time_point> salt_pool_;
```

**清理机制：** 每秒清理一次过期条目（超过 `timestamp_window` 的条目）。

**问题分析：**

- 每个新 TCP 连接产生一个 salt，清理周期内累积 `连接速率 x timestamp_window` 个条目
- 1000 连接/秒 + 30s 窗口 = 约 **30,000 条目**
- **无上限保护**——如果连接速率突然飙升，内存消耗线性增长
- 极端场景：10,000 连接/秒 + 30s 窗口 = 约 **300,000 条目**

| 连接速率 | timestamp_window | 稳态条目数 | 内存占用（估算） |
|----------|------------------|------------|------------------|
| 100/s | 30s | ~3,000 | ~300 KB |
| 1,000/s | 30s | ~30,000 | ~3 MB |
| 10,000/s | 30s | ~300,000 | ~30 MB |
| 100,000/s | 30s | ~3,000,000 | ~300 MB |

### "乐观响应"问题

在 `process.cpp` 中，`acknowledge()` 函数在**拨号成功后**才向客户端发送响应。这意味着：

1. 握手成功（密码验证通过）
2. 服务端尝试拨号到目标地址
3. 如果拨号失败（目标不可达），客户端已经看到"握手成功"
4. 客户端认为连接已建立，但随后立即收到断开

**日志表现：**

```
[SS2022.Process] handshake succeeded from 203.0.113.5:54321
[SS2022.Process] dial failed: connection refused to 192.0.2.1:80
[SS2022.Process] acknowledge sent, then immediately closing
```

**客户端视角：**

```
连接建立 → 无数据 → 连接断开（超时或 RST）
```

**根因：** 协议设计中握手确认与目标拨号的时序问题，非 bug 但影响排障体验。

### UDP session_tracker 资源耗尽

在 `datagram.cpp` 中，`get_or_create` 在 **AEAD 解密前** 被调用：

```cpp
auto& session = session_tracker_.get_or_create(session_id);
// session 已创建，即使后续解密失败
auto result = aead_decrypt(session, payload);
if (!result) {
    // 解密失败，但 session 已经存在于 tracker 中
    return error;
}
```

**攻击向量：**

- 攻击者发送大量携带不同 `SessionID` 的 UDP 数据包
- 每个包都会在 `session_tracker` 中创建一个条目
- 即使 AEAD 解密失败（密钥不匹配），session 条目已经存在
- 无效 session 累积，最终耗尽内存

**诊断特征：**

| 现象 | 说明 |
|------|------|
| UDP 转发性能逐步下降 | session_tracker 查找变慢（哈希冲突增多） |
| 内存持续增长 | 无效 session 不被清理 |
| TCP 连接正常 | 仅影响 UDP |

### nonce 递增问题

SS2022 的 AEAD nonce 使用小端序递增（little-endian increment），理论溢出点为 `2^96`（AES-128-GCM 的 nonce 长度）。

**实际问题不在溢出，而在同步：**

- TCP 流版本中 nonce 自动递增
- 如果连接中断后重连，或中间层重传导致数据重复/丢失
- 两端 nonce 不同步时，**后续所有操作都会失败**
- 错误信息只有通用的 `crypto_error`，无法区分是 nonce 错、密钥错还是数据损坏

**日志模板：**

```
[SS2022.AEAD] crypto_error: nonce=0x000000000000000000000001, tag mismatch
[SS2022.AEAD] decrypt failed at offset 0, connection will be terminated
```

---

## Trojan 深层分析

### SHA224 明文传输

Trojan 协议的认证方式是将密码做 SHA224 哈希后以 **56 字节 hex 字符串** 形式传输：

```
Trojan 请求格式:
  [56-byte hex SHA224] [CRLF] [cmd] [atyp] [addr] [port] [CRLF] [payload]
```

**安全分析：**

| 场景 | 安全性 | 说明 |
|------|--------|------|
| 正常 TLS 保护 | 安全 | SHA224 在 TLS 加密层内传输，即使哈希也无法反推 |
| TLS 被剥掉（配置错误） | 危险 | 56 字节 hex 明文暴露，可被中间人截获 |
| 与 VLESS UUID 对比 | Trojan 更大 | VLESS UUID 仅 16 字节二进制，Trojan 56 字节 hex |

**TLS 被剥掉的典型配置错误：**

- `tls.enabled: false` 但 `trojan.enabled: true`
- TLS SNI 配置错误导致 TLS 握手失败，降级为明文
- 中间层代理（如 CDN）剥离 TLS

**验证步骤：**

```bash
# 生成正确的 SHA224 哈希
echo -n "your-password" | sha224sum

# 对比配置文件中的哈希值
grep -i password /path/to/config.yaml
```

### 缓冲区精确边界

Trojan 请求的最小缓冲区需求：

```
required_total = 56(hex) + 2(CRLF) + 1(cmd) + 1(atyp) + N(addr) + 2(port) + 2(CRLF)
```

**各地址类型下的 `required_total`：**

| 地址类型 | addr 长度 | required_total | 缓冲区大小 | 余量 |
|----------|-----------|----------------|------------|------|
| IPv4 | 4 | 68 | 320 | 252 |
| IPv6 | 16 | 80 | 320 | 240 |
| 域名（最大 255 字节） | 255 | 319 | 320 | **1** |
| 域名（255 + 1 长度前缀） | 256 | 320 | 320 | **0** |

**关键发现：** 域名长度达到 255 字节时，`required_total` 几乎等于固定缓冲区大小 320 字节，**零余量**。

这意味着：

- 域名长度恰好 255 字节时可以正常工作
- 但任何额外的填充、扩展或未来协议变更都无法容纳
- 如果客户端发送超过 255 字节的域名，行为取决于 `async_read` 的缓冲区处理——可能截断或断连

### 日志不区分错误类型

Trojan 的认证失败只输出一条通用日志：

```
[Trojan] credential verification failed
```

**不区分以下场景：**

| 实际根因 | 日志输出 | 排障难度 |
|----------|----------|----------|
| 密码哈希长度不等于 56 | `credential verification failed` | 需手动检查长度 |
| 密码哈希包含非 hex 字符 | `credential verification failed` | 需手动验证格式 |
| 密码正确但大小写错误 | `credential verification failed` | hex 不区分大小写，此项不存在 |
| 密码完全错误 | `credential verification failed` | 需 `sha224sum` 对比 |

**排障建议：**

```bash
# 1. 验证密码哈希长度
echo -n "your-password" | sha224sum | awk '{print length($1)}'
# 应输出: 56

# 2. 验证哈希值
echo -n "your-password" | sha224sum
# 对比配置文件中的值（hex 不区分大小写）

# 3. 检查是否有特殊字符
echo -n "your-password" | xxd | head
# 确认没有意外的换行或空格
```

---

## VLESS 深层分析

### XTLS/Vision flow 不支持

Prism 的 VLESS 实现要求 `addnl_info_len` 必须为 **0**：

```cpp
if (addnl_info_len != 0) {
    return error::bad_message;
}
```

**影响：**

- 客户端配置使用 XTLS/Vision flow 时，`addnl_info_len` 非零
- 连接在 handshake 阶段**立即失败**
- 错误码 `bad_message` (6) 与其他 `bad_message` 场景**无法区分**

**日志模板：**

```
[Vless] handshake failed: bad_message
[Vless] rejected connection from 203.0.113.5:54321, addnl_info_len=12
```

**问题：** 仅凭 `bad_message` 错误码无法判断是 Vision flow 不支持还是其他格式错误。

**常见触发场景：**

| 客户端配置 | addnl_info_len | 结果 |
|------------|----------------|------|
| `flow: ""` (空) | 0 | 正常连接 |
| `flow: "xtls-rprx-vision"` | 非零 | `bad_message`，连接立即失败 |
| 无 flow 字段 | 0 | 正常连接 |

### 固定 320 字节缓冲区

VLESS 请求的最大理论大小：

```
1(version) + 16(UUID) + 1(addnl_info_len) + 1(cmd) + 2(port) + 1(atyp) + 1(len) + 255(domain)
= 278 字节
```

| 参数 | 值 |
|------|-----|
| 最大请求大小 | 278 字节 |
| 固定缓冲区大小 | 320 字节 |
| 余量 | 42 字节 |

与 Trojan 相比，VLESS 的 42 字节余量相对充裕，但仍需注意：

- 如果协议扩展增加字段，可能耗尽余量
- 实际上 VLESS 协议扩展通过 `addnl_info` 字段进行，但 Prism 已拒绝非零值

---

## SOCKS5 深层分析

### UDP ASSOCIATE 仅绑定 IPv4

在 SOCKS5 的 UDP ASSOCIATE 实现中，`bind_datagram_port` 使用了硬编码的 `udp::v4()`：

```cpp
// 问题代码示意
udp::socket sock(io_context, udp::v4());  // 硬编码 IPv4
sock.bind(udp::endpoint(udp::v4(), 0));
```

**后果：**

| 客户端地址类型 | UDP ASSOCIATE 行为 |
|----------------|-------------------|
| IPv4 客户端 | 正常工作 |
| IPv6 客户端 | **静默失败**，无响应 |
| 双栈客户端（IPv6 优先） | 可能失败，取决于 DNS 解析顺序 |

**最严重的问题：完全没有任何日志输出。** 只能通过代码审计发现此 bug。

**UDP ASSOCIATE 流程对比：**

```
正常流程 (IPv4):
  Client → SOCKS5 CONNECT/UDP ASSOCIATE → Server binds UDP::v4
  Client → UDP data to bound port → Server forwards → 成功

失败流程 (IPv6):
  Client → SOCKS5 CONNECT/UDP ASSOCIATE → Server binds UDP::v4
  Client → UDP data to [IPv6 address]:port → Server 无法接收 → 静默失败
  （无日志、无错误返回）
```

### nmethods 边界处理

SOCKS5 握手阶段客户端发送 `VER | NMETHODS | METHODS`，其中 `NMETHODS` 为 1 字节（0-255）。

**边界情况：**

| nmethods 值 | 处理方式 | 结果 |
|-------------|----------|------|
| 0 | 无可用认证方法 | 返回 `0xFF`（无可用方法） |
| 1 | 正常读取 1 个方法 | 标准流程 |
| 255 | 读取 255 个方法字节 | 正常处理 |
| 256 | 不可能（1 字节最大 255） | N/A |

`nmethods=0` 是一个合法但不常见的边界情况。根据 RFC 1928，如果服务端不支持客户端提供的任何方法，应返回 `0xFF`。Prism 的实现在 `nmethods=0` 时直接返回 `0xFF`，行为正确。

---

## 通用加密错误诊断

### crypto_error 反推决策树

当出现 `crypto_error` 时，按以下决策树逐步缩小根因范围：

```
crypto_error
├── 是否仅 SS2022？
│   ├── 是 → 检查 timestamp_window
│   │   ├── client_ts 与 server_ts 差值 > window → timestamp_expired
│   │   └── 差值正常 → 检查 salt
│   │       ├── salt 重复 → salt replay（密钥泄露或重放攻击）
│   │       └── salt 唯一 → 检查 PSK
│   │           ├── PSK 长度/格式错误 → PSK 配置错误
│   │           └── PSK 正确 → nonce 同步问题
│   └── 否 → 通用 AEAD 诊断
│       ├── 首次解密就失败 → 密钥错误
│       ├── 前几个包成功，后续失败 → nonce 不同步
│       ├── 间歇性失败 → 数据损坏（网络层）
│       └── 所有包都失败 → AEAD tag 不匹配（密钥或算法不匹配）
└── 特定协议？
    ├── Trojan → SHA224 哈希不匹配
    ├── VLESS → UUID 不匹配（通常非 crypto_error）
    └── SOCKS5 → 无加密层，不应出现 crypto_error
```

### 错误码速查表

#### SS2022 错误码

| 错误码 | 触发条件 | 日志关键词 | 严重程度 |
|--------|----------|------------|----------|
| `timestamp_expired` | `abs(client_ts - now) > timestamp_window` | `timestamp expired`, `diff=` | 高（连接被拒绝） |
| `salt_reused` | 同一 salt 在窗口内重复出现 | `salt replay`, `duplicate salt` | 高（疑似重放攻击） |
| `crypto_error` | AEAD 解密失败（密钥/nonce/数据损坏） | `crypto_error`, `decrypt failed` | 高（连接断开） |
| `invalid_psk` | PSK 格式错误或解码失败 | `invalid PSK`, `base64 decode failed` | 严重（无法建立任何连接） |
| `buffer_overflow` | 固定头部长度超出预期 | `buffer overflow`, `header too large` | 中（ malformed input） |

#### Trojan 错误码

| 错误码 | 触发条件 | 日志关键词 | 严重程度 |
|--------|----------|------------|----------|
| `credential_failed` | SHA224 哈希不匹配 | `credential verification failed` | 高（认证失败） |
| `bad_message` | 请求格式错误（长度不足、CRLF 缺失） | `bad message`, `invalid format` | 高（连接断开） |
| `buffer_overflow` | 请求超过 320 字节缓冲区 | `buffer overflow` | 中 |
| `unsupported_cmd` | cmd 字段非 1 (CONNECT) 或 3 (UDP) | `unsupported command` | 低 |

#### VLESS 错误码

| 错误码 | 触发条件 | 日志关键词 | 严重程度 |
|--------|----------|------------|----------|
| `bad_message` (6) | addnl_info_len != 0（Vision flow）或格式错误 | `bad_message`, `handshake failed` | 高 |
| `invalid_version` | 版本号非 0 | `invalid version` | 高 |
| `uuid_mismatch` | UUID 不在允许列表中 | `uuid mismatch`, `unknown UUID` | 高（认证失败） |

#### SOCKS5 错误码

| 错误码 | 触发条件 | 日志关键词 | 严重程度 |
|--------|----------|------------|----------|
| `0xFF` (no acceptable methods) | nmethods=0 或无匹配方法 | `no acceptable method` | 中 |
| `0x01` (general failure) | 一般性 SOCKS 错误 | `general SOCKS server failure` | 中 |
| `0x05` (connection refused) | 拨号目标拒绝连接 | `connection refused` | 低 |
| `0x08` (address type not supported) | atyp 不支持 | `address type not supported` | 中 |

### 通用排障日志模板

#### SS2022 排障日志模板

```log
=== SS2022 连接排障 ===
[时间检查]
  client_ts:       ___________
  server_ts:       ___________
  diff:            ___________s
  timestamp_window: ___________s
  结果:            [PASS/FAIL]

[PSK 检查]
  PSK 长度:        ___________ bytes
  base64 解码:     [PASS/FAIL]
  PSK 是否为空:    [YES/NO]

[Salt 检查]
  salt 值:         ___________
  是否重复:        [YES/NO]
  salt_pool 大小:  ___________

[AEAD 检查]
  nonce 当前值:    ___________
  解密结果:        [PASS/FAIL]
  错误详情:        ___________
```

#### Trojan 排障日志模板

```log
=== Trojan 连接排障 ===
[凭据检查]
  收到哈希:        ___________ (长度: ___)
  期望哈希:        ___________ (长度: ___)
  哈希匹配:       [PASS/FAIL]
  格式检查 (hex):  [PASS/FAIL]

[请求解析]
  总长度:          ___________ bytes
  CRLF 位置:       ___________
  cmd 字段:        ___________
  atyp 字段:       ___________
  目标地址:        ___________
  目标端口:        ___________
  缓冲区余量:      ___________ bytes
```

#### VLESS 排障日志模板

```log
=== VLESS 连接排障 ===
[版本检查]
  协议版本:        ___________
  结果:            [PASS/FAIL]

[UUID 检查]
  收到 UUID:       ___________
  是否匹配:       [PASS/FAIL]

[Flow 检查]
  addnl_info_len:  ___________
  flow 支持:       [SUPPORTED/REJECTED]
  注: Prism 不支持 Vision flow

[请求解析]
  cmd 字段:        ___________
  atyp 字段:       ___________
  目标地址:        ___________
  目标端口:        ___________
  请求总大小:      ___________ bytes (缓冲区: 320)
```

#### SOCKS5 排障日志模板

```log
=== SOCKS5 连接排障 ===
[握手阶段]
  VER:             ___________ (期望: 0x05)
  NMETHODS:        ___________
  METHODS:         ___________
  选择方法:        ___________

[请求阶段]
  VER:             ___________
  CMD:             ___________ (1=CONNECT, 2=BIND, 3=UDP)
  ATYP:            ___________ (1=IPv4, 3=域名, 4=IPv6)
  目标地址:        ___________
  目标端口:        ___________

[UDP ASSOCIATE]
  绑定地址类型:    ___________ (实际: IPv4 only)
  客户端地址类型:  ___________
  兼容性:          [PASS/FAIL - IPv6 静默失败]
```

---

## 总结：协议实现边界问题一览

| 协议 | 问题 | 类别 | 影响范围 | 可观测性 |
|------|------|------|----------|----------|
| SS2022 | timestamp_window 窄窗口 | 时间维度 | 时钟不同步的客户端 | 有日志，易诊断 |
| SS2022 | PSK 半初始化 | 状态维度 | 异常代码路径触发 segfault | 无日志，难诊断 |
| SS2022 | salt_pool 无上限 | 资源维度 | 高连接速率场景 | 需内存监控 |
| SS2022 | 乐观响应 | 时序维度 | 目标不可达场景 | 有日志，需理解时序 |
| SS2022 | UDP session_tracker 泄漏 | 资源维度 | UDP 攻击场景 | 无直接日志 |
| SS2022 | nonce 递增不同步 | 状态维度 | 连接中断/重传 | 日志不精确 |
| Trojan | SHA224 明文暴露 | 安全维度 | TLS 配置错误 | 需流量分析 |
| Trojan | 缓冲区零余量 | 缓冲区维度 | 域名长度 255 字节 | 正常时无表现 |
| Trojan | 错误日志不区分 | 可观测性 | 所有认证失败场景 | 需手动验证 |
| VLESS | Vision flow 不支持 | 功能维度 | 使用 Vision 的客户端 | 有日志但模糊 |
| VLESS | 固定 320 字节缓冲区 | 缓冲区维度 | 协议扩展 | 正常时无影响 |
| SOCKS5 | UDP 仅 IPv4 | 功能维度 | IPv6 客户端 | 无日志，最难发现 |
| SOCKS5 | nmethods=0 | 边界维度 | 非标准客户端 | 符合 RFC |

**发现优先级建议：**

1. **SOCKS5 UDP IPv6 静默失败** -- 零可观测性，用户无任何反馈
2. **SS2022 PSK 半初始化 segfault** -- 崩溃无日志，难以定位
3. **SS2022 UDP session_tracker 泄漏** -- 可被攻击利用，无防护
4. **SS2022 nonce 递增诊断不足** -- 所有后续操作失败但无法定位
5. **Trojan/VLESS 错误日志模糊** -- 增加排障时间但不影响功能
