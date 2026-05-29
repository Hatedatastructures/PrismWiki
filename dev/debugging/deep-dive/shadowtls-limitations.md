---
title: ShadowTLS 深层限制与故障分析
source:
  - src/prism/stealth/shadowtls/handshake.cpp
  - src/prism/stealth/shadowtls/transport.cpp
  - src/prism/stealth/shadowtls/util/auth.cpp
module: debugging
type: deep-dive
tags: [debugging, shadowtls, tls, hmac, resource-exhaustion, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[dev/debugging/tls]]"
  - "[[core/stealth/shadowtls/handshake]]"
  - "[[core/stealth/shadowtls/auth]]"
---

# ShadowTLS 深层限制与故障分析

## 概述

ShadowTLS 是 Prism stealth 子系统中的伪装方案之一，其设计哲学与其他方案（如 Reality）有本质区别：**ShadowTLS 不是伪造 TLS，而是将流量代理到真实后端 TLS 服务器，同时通过 HMAC 累积验证实现客户端认证**。

这种设计使被动观测者看到的是完全正常的 TLS 流量（真实的证书、真实的 TLS 握手），但引入了独特的故障模式和资源管理限制。

### 核心机制

```
客户端连接到达
      |
      v
[1] ClientHello HMAC 验证
      |  提取 SessionID 最后 4 字节作为 HMAC 标签
      |  HMAC-SHA1(password, data)[:4] 与标签恒定时间比较
      |  多用户遍历匹配，第一个命中胜出
      |
      v
[2] 后端 TLS 连接
      |  连接 handshake_dest（真实 TLS 服务器）
      |  提取 ServerRandom 等关键字段
      |
      v
[3] 双工转发（握手阶段）
      |  后端 -> 客户端: 明文 payload（不加密）
      |  客户端 -> 服务端: XOR 加密 + HMAC 标签
      |
      v
[4] 首帧 HMAC 匹配
      |  客户端发送的首帧包含 HMAC 标签
      |  匹配成功 → 进入传输阶段
      |  匹配失败 → 连接终止
      |
      v
[5] 传输阶段
      正常的双工数据转发，累积 HMAC 贯穿全程
```

### 关键数字

| 指标 | 值 |
|------|-----|
| HMAC 标签长度 | 4 字节 |
| HMAC 算法 | HMAC-SHA1 |
| ClientHello 最小长度 | 76 字节 |
| SHA1 内部状态大小 | 20 字节（固定） |
| relay 超时等待 | 500ms |
| TLS 记录长度上限 | 65535 字节（2 字节字段） |

---

## 握手阶段故障

### ClientHello HMAC 验证

ClientHello HMAC 验证是 ShadowTLS 认证的第一道关卡。整个验证过程在 `auth.cpp` 中实现。

#### 验证流程

```
收到 ClientHello
      |
      v
长度检查: >= 76 字节？
      |  否 → 拒绝（长度不足）
      v
提取 SessionID 最后 4 字节 → HMAC 标签
      |
      v
遍历所有 users:
      password = users[i].password
      expected = HMAC-SHA1(password, data)[:4]
      |
      +-- CRYPTO_memcmp(tag, expected, 4) == 0？
      |       |
      |       是 → 认证成功，用户匹配
      |       |
      |       否 → 继续下一个 user
      |
      v
所有 users 均不匹配 → auth_failed
```

#### 技术细节

| 参数 | 值 | 说明 |
|------|-----|------|
| 最小 ClientHello 长度 | 76 字节 | 低于此值直接拒绝 |
| HMAC 标签位置 | SessionID 最后 4 字节 | 客户端在构造时嵌入 |
| HMAC 算法 | HMAC-SHA1 | 输出截断为前 4 字节 |
| HMAC 输入 | `password` + ClientHello `data` | data 为 ClientHello 关键字段 |
| 比较方式 | `CRYPTO_memcmp` | 恒定时间比较，防止时序攻击 |
| 多用户匹配 | 遍历所有 users | **第一个匹配胜出**（顺序敏感） |

#### 认证失败处理

```
auth_failed
      |
      v
executor rewind 尝试
      |
      v
snapshot.wrote_ == false ?
      |  是 → rewind 成功，尝试下一个方案
      |  否 → 无法 rewind（见下文"转发后无法 rewind"）
      v
下一个方案
```

认证失败后，executor 尝试 rewind 传输层到认证前的快照状态。如果此时 `wrote_` 标记仍为 false（即尚未向客户端写入任何数据），rewind 可以成功，executor 将尝试管道中的下一个伪装方案。

**多用户匹配的顺序敏感性**：当多个 users 的 HMAC 值碰巧都能匹配时（虽然概率极低，4 字节标签空间为 2^32），先列出的 user 总是胜出。这不是安全漏洞（碰撞概率约 1/4,294,967,296），但在调试时需注意用户列表的顺序。

---

### 后端连接失败

ShadowTLS 需要连接一个真实的后端 TLS 服务器（`handshake_dest`），将其 TLS 握手代理到客户端。这一步的失败模式较为直接。

#### 故障分类

| 故障点 | 错误码 | 日志特征 | 可恢复性 |
|--------|--------|----------|----------|
| `handshake_dest` 不可达 | `connection_refused` | `connect to handshake_dest failed: {ec}` | 可 rewind，尝试下一方案 |
| ServerRandom 提取失败 | `protocol_error` | `failed to extract ServerRandom` | 可 rewind |
| strict_mode + 后端不支持 TLS 1.3 | `protocol_error` | `backend does not support TLS 1.3 in strict mode` | 可 rewind |

#### 后端不可达诊断

```
[ShadowTLS] connect to handshake_dest failed: Connection refused
      |
      可能原因:
        1. handshake_dest 地址配置错误（IP/域名/端口）
        2. 后端服务器宕机
        3. 网络防火墙阻断
        4. DNS 解析失败（如果 dest 是域名）

      排查:
        curl -v https://{handshake_dest_host}:{handshake_dest_port}
        dig {handshake_dest_host}
        traceroute {handshake_dest_host}
```

#### strict_mode 详解

strict_mode 是一个安全增强选项。启用时，ShadowTLS 验证后端 TLS 服务器必须支持 TLS 1.3。

| 模式 | 行为 | 安全性 |
|------|------|--------|
| strict_mode = off | 接受后端任何 TLS 版本 | 较低，可能被指纹识别 |
| strict_mode = on | 后端必须支持 TLS 1.3 | 较高，与常见客户端行为一致 |

如果后端服务器仅支持 TLS 1.2（如旧版 Nginx 未升级），strict_mode 将导致握手失败。

---

### 转发 ClientHello 后无法 rewind

这是 ShadowTLS 最关键的架构限制之一，直接影响故障恢复能力。

#### 机制分析

ShadowTLS 在握手阶段需要将客户端发来的 ClientHello **原封不动地转发**到后端 TLS 服务器。这个转发操作是一次写入：

```
ensure_snapshot()              ← 保存传输层快照
      |
      v
读取 ClientHello               ← 从客户端读取
      |
      v
HMAC 验证                      ← 验证客户端身份
      |
      v
连接后端 handshake_dest         ← 建立 TCP 连接到真实 TLS 服务器
      |
      v
async_write(ClientHello)        ← 将 ClientHello 转发到后端
      |                              此时 snapshot.wrote_ = true
      v
后续步骤...
```

问题在于：`async_write(ClientHello)` 是一个**写操作**。一旦执行，snapshot 的 `wrote_` 标记变为 true。这意味着：

- 后续任何步骤失败都**无法 rewind** 到认证前的状态
- executor 无法尝试下一个伪装方案
- 传输层数据已被写入后端，状态被"污染"

#### 影响范围

```
ClientHello 转发到后端后，以下失败场景无法 rewind:

  [1] 后端握手成功，但首帧 HMAC 匹配失败
      → 传输层已被后端的 TLS 响应数据污染
      → 客户端已收到后端的真实 TLS ServerHello
      → 无法回退到其他方案

  [2] 后端连接建立但握手超时
      → 后端可能已经分配资源
      → 客户端等待超时

  [3] relay 协程启动后失败
      → relay 可能已经往客户端写了数据
      → 传输层状态不一致
```

#### 对比 Reality

| 特性 | ShadowTLS | Reality |
|------|-----------|---------|
| 转发 ClientHello | 是（到后端） | 否（本地处理） |
| 写操作时机 | 握手早期（转发 ClientHello） | 握手中期（发送 ServerHello） |
| rewind 窗口 | 极短（仅在 HMAC 验证前） | 较长（直到发送 ServerHello） |
| 后端依赖 | 强依赖（必须连接真实服务器） | 弱依赖（仅用于回退代理） |

---

## 传输阶段故障

### 写入方向发 plain payload

ShadowTLS 传输阶段的数据流存在一个容易被误解的设计：**写入方向（服务端 -> 客户端）发送的是明文 payload，不做 XOR 加密**。

#### 方向定义

```
                        ShadowTLS 服务端
                              |
              +---------------+---------------+
              |                               |
     读取方向（客户端 -> 服务端）      写入方向（服务端 -> 客户端）
              |                               |
     握手阶段: XOR 加密               握手阶段: 明文转发
     传输阶段: 明文                   传输阶段: 明文
```

#### 安全性分析

| 方向 | 加密方式 | 安全性依赖 |
|------|----------|-----------|
| 客户端 -> 服务端 | XOR（握手阶段） | XOR 密钥的保密性 |
| 服务端 -> 客户端 | **无加密** | 外层 TLS 或网络加密 |

这意味着在握手阶段：
- 客户端发送到服务端的数据经过 XOR 加密（基于 HMAC 派生的密钥流）
- 服务端发送到客户端的数据**直接转发**后端的明文 payload

**安全性依赖外层**：如果 ShadowTLS 运行在已有加密的传输层之上（如外层 TLS），这不会引入安全问题。但如果外层不加密，服务端 -> 客户端方向的数据对中间人完全可见。

#### 排错注意事项

这一设计选择与 sing-shadowtls 的行为一致（ShadowTLS 的参考实现），但在排障时需要注意：

```
[场景] 抓包分析 ShadowTLS 流量

观察方向:
  服务端 -> 客户端: 看到 TLS ServerHello、Certificate 等明文
  客户端 -> 服务端: 看到加密后的数据（XOR）

误判风险:
  如果只抓取服务端 -> 客户端方向，可能误认为 ShadowTLS 无加密
  实际上这是一个有意的设计选择，安全性由外层保证
```

---

### relay 协程 use-after-close 竞态

这是 ShadowTLS 传输层中最复杂的并发安全问题。

#### 机制分析

握手阶段，ShadowTLS 启动一个 relay 协程将后端 TLS 数据转发到客户端：

```
handshake()
      |
      v
连接后端成功
      |
      v
co_spawn(relay_coroutine, detached)     ← 启动 relay 协程
      |                                      relay: 后端 -> 客户端
      |
      v
等待首帧 HMAC 匹配
      |
      +-- 匹配成功 → 进入传输阶段
      |
      +-- 匹配失败 或 超时 → cancellation_signal.emit()
                                    |
                                    v
                              等待 relay 退出（500ms 超时）
                                    |
                                    +-- relay 在 500ms 内退出 → 正常
                                    |
                                    +-- relay 未退出 → 日志警告
                                        "relay coroutine did not exit within
                                         timeout, socket may be corrupted"
```

#### 竞态条件详解

```
时间线（最坏情况）:

T+0ms    handshake 启动 relay 协程 (co_spawn, detached)
              relay 开始从 backend_sock 读取数据

T+10ms   handshake 等待首帧 HMAC，超时
              cancellation_signal.emit()

T+10ms   handshake 等待 relay 退出
              relay 可能:
                - 阻塞在 backend_sock.async_read() → 未被调度
                - 正在往 client_sock.async_write() → 数据传输中

T+510ms  500ms 超时到达
              relay 仍未退出
              handshake 日志: "relay coroutine did not exit within timeout"

T+510ms  handshake 继续执行后续代码
              可能关闭 backend_sock
              可能继续使用 client_sock

T+???ms  relay 协程被调度
              relay 尝试访问已关闭的 backend_sock → use-after-close
              或 relay 尝试往 client_sock 写数据 → 数据污染
```

#### 影响评估

| 场景 | 概率 | 影响 |
|------|------|------|
| relay 在 500ms 内正常退出 | 高 | 无影响 |
| relay 阻塞在 read，500ms 后取消生效 | 中 | 无影响（取消信号传播到 async_read） |
| relay 正在 write，500ms 内未完成 | 低 | **数据污染**：relay 往 client_sock 写入意外数据 |
| relay 的 async_read 持有 backend_sock 引用 | 低 | **use-after-close**：后续代码关闭 backend_sock 后 relay 仍尝试读取 |

#### 日志特征

```
# 正常情况：relay 在超时内退出
[ShadowTLS] first frame HMAC mismatch, cancelling relay
[ShadowTLS.Relay] relay coroutine exited normally

# 异常情况：relay 超时未退出
[ShadowTLS] first frame HMAC mismatch, cancelling relay
[ShadowTLS] relay coroutine did not exit within timeout, socket may be corrupted
[ShadowTLS] continuing with potentially corrupted state
```

#### 风险缓解方向

当前实现选择 500ms 等待是一种工程权衡。完全消除竞态需要：

1. 使用 `co_await` 等待 relay 协程完成（但 relay 可能永不返回，导致握手也阻塞）
2. 引入 shared_ptr + weak_ptr 管理 socket 生命周期
3. 使用 `async_completion` 确保 relay 的所有异步操作在关闭前完成

---

### 累积 HMAC 无重置

ShadowTLS 使用两个 HMAC 上下文（`hmac_write_ctx` 和 `hmac_read_ctx`）分别追踪两个方向的数据完整性。这些上下文使用 `HMAC_Update` 累积所有 payload，且**永远不会重置**。

#### 机制分析

```
传输阶段开始:
      hmac_write_ctx = HMAC_Init(password)
      hmac_read_ctx  = HMAC_Init(password)

每收到一个 payload (读取方向):
      HMAC_Update(hmac_read_ctx, payload)

每发送一个 payload (写入方向):
      HMAC_Update(hmac_write_ctx, payload)

... 永不调用 HMAC_Reset 或 HMAC_Init ...
```

#### 内存影响

| 因素 | 分析 |
|------|------|
| SHA1 内部状态大小 | 固定 20 字节 |
| HMAC 上下文总大小 | 固定（不随数据量增长） |
| 内存泄漏风险 | **无** |

SHA1 的内部状态固定为 20 字节（160 位），不随输入数据量增长。`HMAC_Update` 只是更新这个固定大小的状态，不会累积历史数据。因此**不存在内存无限增长的问题**。

#### CPU 影响

这是真正的性能瓶颈所在：

```
HMAC_Update 的时间复杂度: O(block_size)
  - SHA1 block_size = 64 字节
  - 每次调用处理一个 block

总 CPU 开销 = 所有 payload 的 HMAC_Update 调用总和
            = O(总传输数据量)

单次 HMAC_Update 调用开销:
  - ~100-200ns (现代 CPU)
  - 对于 1KB payload ≈ 16 次 block 处理 ≈ 1.6-3.2μs

长连接场景 (传输 10GB):
  - HMAC_Update 调用次数 ≈ 10GB / 64B ≈ 1.6 亿次
  - 累计 CPU 时间 ≈ 16-32 秒
```

#### 长连接性能退化

| 传输数据量 | HMAC 调用次数 | 累计 CPU 时间 | 影响程度 |
|-----------|--------------|--------------|----------|
| 100 MB | ~160 万次 | ~0.16-0.32s | 可忽略 |
| 1 GB | ~1600 万次 | ~1.6-3.2s | 轻微 |
| 10 GB | ~1.6 亿次 | ~16-32s | **显著** |
| 100 GB | ~16 亿次 | ~160-320s | **严重** |

在长连接场景（如持续运行的数据流、持久隧道）中，累积 HMAC 的 CPU 开销随传输数据量线性增长。由于 HMAC 上下文永远不重置，唯一的缓解方式是**断开连接并重新建立**。

#### 缓解措施

```
# 当前: 无法缓解，只能断开重连

# 潜在改进方向:
  1. 定期重置 HMAC 上下文（需要协议层协商）
  2. 基于时间/数据量的 HMAC 重置机制
  3. 使用更高效的哈希算法（如 HMAC-SHA256 的硬件加速）
  4. 限制单连接最大传输数据量，超限自动重连
```

---

### TLS frame 长度无上限检查

ShadowTLS 在读取 TLS 记录时，直接从记录头提取长度字段并分配对应大小的缓冲区，没有额外的上限检查。

#### 漏洞代码模式

```
read_tls_frame():
      |
      v
读取 5 字节 TLS 记录头
      raw[0]    = content_type
      raw[1:3]  = TLS version
      raw[3:4]  = record_length (2 字节, 大端序)
      |
      v
record_length = (raw[3] << 8) | raw[4]
      |
      v
payload.resize(record_length)        ← 分配 record_length 字节缓冲区
      |
      v
async_read(payload)                  ← 读取 record_length 字节数据
```

#### 攻击向量

| 攻击方式 | 构造 | 影响 |
|---------|------|------|
| 声称大记录 | TLS 记录头声称 65535 字节，实际发送很少 | 分配 64KB 缓冲区后等待超时 |
| 持续发送大记录头 | 多个连接同时声称最大长度 | 高并发下内存放大效应 |
| 非对称攻击 | 发送 5 字节记录头 + 不发送数据 | 每连接占住 64KB 缓冲区，极低成本 |

#### 内存影响估算

```
假设:
  恶意连接数 = 1000
  每连接声称 record_length = 65535

理论内存 = 1000 * 65535 ≈ 64 MB

对比:
  正常 TLS 记录 (16KB) * 1000 ≈ 16 MB

放大倍数 ≈ 4x
```

虽然单个 64KB 分配在现代系统中不构成威胁，但在高并发恶意连接场景下存在内存放大效应。结合 Session 无 idle timeout（见 [[dev/debugging/deep-dive/system-risks]]），攻击者可以低成本消耗大量内存。

---

## 排障方法

### 日志标签速查表

| 日志标签 | 模块 | 输出内容 |
|----------|------|----------|
| `[ShadowTLS]` | handshake.cpp | 握手各阶段进度和错误 |
| `[ShadowTLS.Relay]` | transport.cpp | relay 协程生命周期和错误 |
| `[ShadowTLS.Transport]` | transport.cpp | 传输阶段错误和 HMAC 状态 |

### 故障场景速查

| 故障现象 | 可能原因 | 日志关键词 | 排查方向 |
|---------|----------|-----------|----------|
| 连接立即断开 | ClientHello HMAC 验证失败 | `auth_failed` | 检查密码配置、SessionID 构造 |
| 连接超时 | 后端不可达 | `connection_refused` | 测试后端可达性 |
| 连接建立但无数据 | relay 协程挂起 | 无后续日志 | 检查后端 TLS 版本 |
| 首帧后断开 | HMAC 不匹配 | `HMAC mismatch` | 检查累积 HMAC 初始状态 |
| 延迟突然增大 | HMAC CPU 开销 | 无直接日志 | 检查连接传输数据量 |
| socket 状态异常 | relay use-after-close | `socket may be corrupted` | 检查 relay 超时频率 |

### 常见故障场景详解

#### 场景 1: auth 失败

```
日志:
  [ShadowTLS] ClientHello received (128 bytes)
  [ShadowTLS] HMAC verification failed for all users
  [ShadowTLS] auth_failed, attempting rewind

可能原因:
  1. 客户端密码与服务端不匹配
  2. SessionID 最后 4 字节未正确嵌入 HMAC 标签
  3. ClientHello 被中间设备修改（如 TLS proxy 修改 SessionID）
  4. HMAC 计算的 data 字段与客户端不一致

排查:
  1. 确认客户端和服务端使用相同的 password
  2. 检查客户端是否正确构造 SessionID（最后 4 字节为 HMAC 标签）
  3. 确认网络路径上没有 TLS 中间件修改 ClientHello
  4. 对比客户端和服务端的 HMAC 输入数据
```

#### 场景 2: 后端不可达

```
日志:
  [ShadowTLS] connecting to handshake_dest: example.com:443
  [ShadowTLS] connect to handshake_dest failed: Connection refused
  [ShadowTLS] connection_refused, rewinding

可能原因:
  1. handshake_dest 地址或端口错误
  2. 后端服务器宕机
  3. 网络防火墙阻断
  4. DNS 解析失败

排查:
  1. curl -v https://example.com:443
  2. dig example.com
  3. telnet example.com 443
  4. 检查防火墙出站规则
```

#### 场景 3: relay 超时

```
日志:
  [ShadowTLS] first frame HMAC mismatch
  [ShadowTLS] cancelling relay coroutine
  [ShadowTLS.Relay] relay coroutine did not exit within timeout
  [ShadowTLS] socket may be corrupted

可能原因:
  1. 后端响应缓慢，relay 阻塞在读取操作
  2. 网络延迟导致 relay 的写操作未在 500ms 内完成
  3. 首帧 HMAC 计算错误（客户端和服务端不一致）

排查:
  1. 检查后端响应延迟: curl -o /dev/null -w "%{time_total}" https://example.com
  2. 检查网络延迟: ping example.com
  3. 确认首帧 HMAC 的计算方式在客户端和服务端一致
  4. 如果频繁出现，考虑增加 relay 超时时间
```

#### 场景 4: HMAC 不匹配

```
日志:
  [ShadowTLS] first frame received
  [ShadowTLS] first frame HMAC mismatch
  [ShadowTLS] expected: 0xaabbccdd, got: 0x11223344

可能原因:
  1. 客户端和服务端 password 不一致
  2. 累积 HMAC 的初始状态不一致（如一方从握手阶段开始累积，另一方从传输阶段开始）
  3. 首帧数据被中间设备修改
  4. HMAC 计算的输入数据包含额外或遗漏的字段

排查:
  1. 确认 password 配置完全一致
  2. 确认两端的 HMAC 累积起点一致
  3. 检查是否有 TLS 中间件修改数据
  4. 开启 debug 日志查看 HMAC 中间值
```

### 故障传播路径

```
客户端连接 ShadowTLS
      |
      +-- auth_failed
      |       |
      |       +-- rewind 成功 → executor 尝试下一个方案
      |       |
      |       +-- rewind 失败 → 返回错误（已转发 ClientHello 到后端）
      |
      +-- connection_refused (后端不可达)
      |       |
      |       +-- rewind 成功 → executor 尝试下一个方案
      |
      +-- relay 超时
              |
              +-- relay 退出 → 正常关闭
              |
              +-- relay 未退出 → socket 状态可能损坏
                      |
                      +-- 后续操作可能返回 io_error
                      +-- 客户端可能收到意外的数据
```

### 诊断命令集

```bash
#!/bin/bash
# shadowtls-quick-diag.sh -- ShadowTLS 快速诊断

echo "=== ShadowTLS 诊断 ==="

echo -e "\n[1] 后端可达性检查"
echo "检查 handshake_dest 是否可达:"
# 替换为实际配置的 handshake_dest
curl -v --connect-timeout 5 https://example.com:443 2>&1 | head -20

echo -e "\n[2] TLS 版本检查"
echo "后端 TLS 版本支持:"
curl -v --tlsv1.3 --tls-max 1.3 https://example.com:443 2>&1 | grep "TLSv"

echo -e "\n[3] ShadowTLS 相关日志"
echo "最近的 ShadowTLS 错误:"
grep "\[ShadowTLS" /var/log/prism.log | grep -i "fail\|error\|mismatch\|corrupted" | tail -20

echo -e "\n[4] relay 超时频率"
echo "relay 超时事件:"
grep -c "did not exit within timeout" /var/log/prism.log

echo -e "\n[5] auth 失败频率"
echo "认证失败事件:"
grep -c "auth_failed\|HMAC verification failed" /var/log/prism.log

echo -e "\n=== 诊断完成 ==="
```

---

## 已知限制汇总

| 限制 | 严重度 | 影响范围 | 缓解方式 |
|------|--------|----------|----------|
| ClientHello 转发后无法 rewind | 高 | 故障恢复 | 无（架构限制） |
| relay 协程 use-after-close 竞态 | 高 | 数据完整性 | 增加超时时间（部分缓解） |
| 累积 HMAC CPU 开销线性增长 | 中 | 长连接性能 | 断开重连 |
| 写入方向无加密 | 中 | 数据安全 | 确保外层加密 |
| TLS frame 长度无上限检查 | 中 | 内存安全 | 限制并发连接数 |
| 多用户匹配顺序敏感 | 低 | 多用户场景 | 注意用户列表顺序 |
| strict_mode 限制后端 TLS 版本 | 低 | 后端兼容性 | 升级后端或关闭 strict_mode |

---

## 相关文档

- [[dev/debugging/tls]] -- TLS 协议参考与问题排查指南
- [[core/stealth/shadowtls/handshake]] -- ShadowTLS 握手状态机源码分析
- [[core/stealth/shadowtls/auth]] -- ShadowTLS 认证逻辑
- [[dev/debugging/deep-dive/system-risks]] -- 系统级风险与资源耗尽分析
- [[dev/debugging/deep-dive/reality-handshake]] -- Reality 握手深层故障分析（对比参考）
- [[core/stealth/executor]] -- 伪装方案执行器
- [[core/fault/code]] -- 全局错误码枚举
