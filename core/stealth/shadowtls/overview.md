---
title: ShadowTLS 协议
created: 2026-05-27
updated: 2026-05-29
layer: core
source:
  - include/prism/stealth/facade/shadowtls/scheme.hpp
  - include/prism/stealth/facade/shadowtls/handshake.hpp
  - include/prism/stealth/facade/shadowtls/transport.hpp
  - include/prism/stealth/facade/shadowtls/util/auth.hpp
  - include/prism/stealth/facade/shadowtls/util/constants.hpp
  - include/prism/stealth/facade/shadowtls/config.hpp
  - src/prism/stealth/facade/shadowtls/handshake.cpp
tags: [stealth, shadowtls, overview, tier-1, hmac-sha1, tls-proxy]
---

# ShadowTLS 协议

> 源码: `include/prism/stealth/facade/shadowtls/` | 实现: `src/prism/stealth/facade/shadowtls/`

ShadowTLS 是 Stealth 模块中 Tier 1 级 TLS 伪装方案，通过代理真实 TLS 服务器的完整握手过程来伪装代理流量。客户端与真实后端之间建立 TLS 隧道，使用 HMAC-SHA1 累积认证区分合法客户端与探测者。

## 模块架构

```
ClientHello 到达
       |
       v
+----------------------------------------------------------+
|  scheme::sniff() -- Tier 0 快速检测                       |
|  检查 TLS 特征位 -> 候选方案                               |
+----------------------------------------------------------+
       | 候选
       v
+----------------------------------------------------------+
|  scheme::verify() -- Tier 1 HMAC 验证                     |
|  verify_client_hello() 检查 SessionID 尾部 4 字节 HMAC   |
|  命中 -> 进入握手                                         |
+----------------------------------------------------------+
       | 认证通过
       v
+----------------------------------------------------------+
|  handshake() -- 三阶段流水线                                |
|                                                          |
|  1. verify_client                                        |
|     v3: 遍历 users 逐个验证 SessionID HMAC              |
|     v2: 用单一 password 验证                             |
|                                                          |
|  2. connect_backend                                      |
|     解析 handshake_dest -> DNS -> TCP 连接后端             |
|     转发 ClientHello -> 读取 ServerHello -> 转发给客户端   |
|                                                          |
|  3. run_relay（并行双向中继）                             |
|     +- relay_modified -> 后端帧 XOR 加密 + HMAC 标签     |
|     +- read_hmac_match -> 读取客户端 HMAC 认证帧         |
|     500ms 超时取消中继                                   |
+----------------------------------------------------------+
       |
       v
  handshake_result + handshake_detail
       |
       v
+----------------------------------------------------------+
|  shadowtls_transport -- 累积 HMAC 传输层                   |
|  继承 transport::transmission                            |
|  读写操作维护双向累积 HMAC_CTX                            |
|  每帧: HMAC 标签(4B) + XOR 加密载荷 + TLS 记录封装      |
+----------------------------------------------------------+
```

### 核心组件

| 组件 | 头文件 | 职责 |
|------|--------|------|
| [[core/stealth/shadowtls/scheme\|scheme]] | facade/shadowtls/scheme.hpp | 方案注册、Tier 1 sniff/verify、handshake 入口 |
| [[core/stealth/shadowtls/handshake\|handshake]] | facade/shadowtls/handshake.hpp | 三阶段握手流水线、后端连接、并行中继 |
| [[core/stealth/shadowtls/transport\|transport]] | facade/shadowtls/transport.hpp | 累积 HMAC 传输层，继承 transmission 基类 |
| [[core/stealth/shadowtls/auth\|auth]] | facade/shadowtls/util/auth.hpp | HMAC-SHA1 验证、XOR 密钥派生 |
| [[core/stealth/shadowtls/constants\|constants]] | facade/shadowtls/util/constants.hpp | 协议常量（TLS 头大小、HMAC 长度等） |
| [[core/stealth/shadowtls/config\|config]] | facade/shadowtls/config.hpp | 配置结构体（v2/v3、用户列表、后端目标） |

## 子页面

- [[core/stealth/shadowtls/scheme|Scheme]] — 方案基类实现
- [[core/stealth/shadowtls/config|Config]] — 配置参数
- [[core/stealth/shadowtls/constants|Constants]] — 常量定义
- [[core/stealth/shadowtls/auth|Auth]] — HMAC-SHA1 累积认证
- [[core/stealth/shadowtls/handshake|Handshake]] — 握手流程
- [[core/stealth/shadowtls/transport|Transport]] — 传输层包装

## 设计决策（WHY）

### 为什么 Tier 1 而非 Tier 0（独占检测）？

**问题**: ShadowTLS 无法仅凭 ClientHello 字节特征可靠识别，需要 HMAC 计算。

**选择**: Tier 0 sniff() 做快速特征匹配（候选），Tier 1 verify() 做 HMAC-SHA1 验证（确认）。两阶段检测避免了对每个连接都做 HMAC 计算的开销。

**后果**: ShadowTLS 检测需要两步，比 Reality（Tier 0 独占）多一次验证。但 unique()==false 允许多个 ShadowTLS 实例共存。

### 为什么用 HMAC-SHA1 累积认证而非单次认证？

**问题**: 单次认证只能验证握手阶段，后续数据传输无法防篡改。

**选择**: 使用累积 HMAC-SHA1，每个方向的 HMAC_CTX 覆盖从握手结束到当前的所有帧载荷。写入方向初始化为 password + ServerRandom + "S"，读取方向初始化为 password + ServerRandom + "C" + 首帧载荷 + 首帧HMAC[:4]。每帧更新累积状态，任何帧的篡改都会导致后续所有帧的 HMAC 校验失败。

**后果**: HMAC_CTX 是 shared_ptr 在握手和传输层之间传递，不可复制（OpenSSL 内部状态）。截断为 4 字节的 HMAC 标签提供了足够的安全性（2^32 暴力破解成本在 TLS 连接超时内不可行）。

### 为什么连接真实 TLS 后端？

**问题**: DPI 可以通过 ServerHello 的证书、密码套件等特征判断是否为真实 TLS 服务器。

**选择**: connect_backend() 建立到配置的 handshake_dest（如 www.microsoft.com:443）的真实 TCP 连接，转发客户端的 ClientHello 并中继真实 ServerHello。客户端看到的 TLS 握手完全来自真实服务器。

**后果**: 握手依赖后端可用性。后端不可达时握手失败。strict_mode（默认开启）要求后端必须协商 TLS 1.3，否则拒绝。

### 为什么用 XOR 而非 AEAD 加密载荷？

**问题**: 后端到客户端的帧载荷需要加密，否则代理数据暴露在 TLS 记录中。

**选择**: 使用 SHA256(password + ServerRandom) 派生 XOR 密钥，对 ApplicationData 帧载荷做逐字节 XOR。XOR 加密简单高效，且每个会话的 ServerRandom 不同，密钥不重复。

**后果**: XOR 不是认证加密，但每个帧已有 HMAC-SHA1 标签保护完整性。XOR 的安全性依赖于 HMAC 的完整性保护。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| SessionID 必须恰好 32 字节 | ShadowTLS 将 HMAC 嵌入 SessionID 尾部 | 非 32 字节 SessionID 的 ClientHello 直接跳过 | constants.hpp:19 |
| HMAC 标签固定 4 字节 | 累积 HMAC-SHA1 截断为前 4 字节 | 标签长度变更破坏与客户端的兼容性 | constants.hpp:20 |
| v2 无多用户支持 | v2 使用单一 password 字段 | 无法区分不同用户，所有客户端共享同一密码 | config.hpp:39 |
| strict_mode 默认开启 | 后端必须协商 TLS 1.3 | TLS 1.2 后端导致握手失败 | config.hpp:43 |
| 握手超时 5 秒 | hs_timeout 默认 5000ms | 超时后连接中断 | config.hpp:44 |
| 中继取消宽限期 500ms | run_relay() 取消中继后等待 500ms | 宽限期内未退出可能导致 socket 状态不一致 | handshake.cpp:600 |
| polluted=true | 握手修改了传输流，不可 rewind | 其他方案无法接管该连接 | handshake.cpp:657 |
| HMAC_CTX 不可复制 | 使用 shared_ptr 传递 | 复制会导致两个传输层共享同一 HMAC 状态 | handshake.hpp:33 |

## 故障场景

### 1. 客户端 SessionID HMAC 不匹配

**触发条件**: ClientHello 的 SessionID 尾部 4 字节与 HMAC-SHA1 计算值不匹配

**传播路径**: verify_client() -> verify_client_hello() 返回 false -> 遍历所有用户均失败 -> 返回 nullopt -> handshake() 返回 auth_failed

**外部表现**: 连接关闭，非 ShadowTLS 客户端不受影响

**日志关键字**: "auth failed"

### 2. 后端 TLS 目标不可达

**触发条件**: handshake_dest 配置的域名无法解析或 TCP 连接被拒绝

**传播路径**: connect_backend() -> DNS 解析失败或 TCP 连接超时 -> 返回 connection_refused

**外部表现**: 连接关闭

**日志关键字**: "connect to dest failed"

### 3. 后端协商非 TLS 1.3（strict_mode）

**触发条件**: strict_mode=true（默认）且后端 ServerHello 不包含 TLS 1.3 supported_versions

**传播路径**: is_tls13_hello() 返回 false -> run_relay() 返回 nullopt -> handshake() 返回 protocol_error

**外部表现**: 连接关闭

**恢复机制**: 设置 strict_mode=false 可允许 TLS 1.2 后端，但降低安全性

### 4. HMAC 匹配帧未在超时内到达

**触发条件**: 客户端未在合理时间内发送包含正确 HMAC 的 TLS 记录

**传播路径**: read_hmac_match() 持续读取非匹配帧 -> 超时或连接断开 -> 返回 nullopt -> protocol_error

**外部表现**: 连接关闭

### 5. 中继协程未在宽限期内退出

**触发条件**: relay_modified() 在收到取消信号后 500ms 内未完成

**传播路径**: 宽限期到期 -> 日志警告 -> socket 可能处于不一致状态

**外部表现**: 后续读写可能失败

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| [[core/recognition/overview\|recognition]] -> shadowtls | 调用 | recognition 解析 ClientHello 生成特征位图，sniff() 据此判断候选 |
| shadowtls -> [[core/crypto/overview\|crypto]] | 依赖 | HMAC-SHA1 累积认证、SHA256 XOR 密钥派生 |
| shadowtls -> [[core/connect/dial/dial\|connect]] | 依赖 | connect_backend() 通过 DNS 解析和 TCP 连接建立到后端的通道 |
| shadowtls -> [[core/transport/transmission\|transport]] | 依赖 | shadowtls_transport 继承 transport::transmission，作为 session 的传输层 |
| [[core/instance/overview\|instance]] <- shadowtls | 被依赖 | session 通过 scheme::handshake() 调用 ShadowTLS 握手 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| password/users 变更 | 所有客户端 | v2 密码变更或 v3 用户列表变更导致旧客户端认证失败 |
| handshake_dest 变更 | 握手行为 | DPI 看到的 TLS 握手指纹变化 |
| strict_mode 关闭 | 安全性 | 允许 TLS 1.2 后端，可能被 DPI 通过 TLS 版本识别 |
| HMAC 标签长度变更 | 所有客户端 | 破坏与 sing-shadowtls 客户端的兼容性 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| recognition hello_features 字段变更 | scheme::sniff() 的特征位检测 | 所有 bitmap 检查逻辑 |
| BoringSSL HMAC API 变更 | HMAC_CTX 生命周期管理 | hmac_ctx_deleter 和所有 HMAC 调用 |
| transport::transmission 接口变更 | shadowtls_transport 的 read/write 实现 | async_read_some / async_write_some 签名 |
| connect::dial DNS 解析行为变更 | connect_backend() 的后端连接建立 | 错误处理和超时逻辑 |

## 相关文档

- [[core/stealth/overview|Stealth 模块总览]] — 三级检测架构和方案执行器
- [[core/stealth/reality/overview|Reality 协议]] — Tier 0 伪装方案对比
- [[core/crypto/overview|Crypto 模块]] — HMAC-SHA1、SHA256 原语
- [[core/transport/transmission|Transport]] — 传输层抽象接口
- [[core/connect/dial/dial|Dial]] — 后端连接建立
