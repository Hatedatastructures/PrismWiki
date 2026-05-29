---
layer: core
source: src/prism/stealth/reality/handshake.cpp
title: Reality Handshake — 握手状态机
module: stealth/reality
tags: [stealth, reality, handshake, state-machine]
updated: 2026-05-27
---

# Reality Handshake — 握手状态机

Reality 协议核心握手状态机，协调 ClientHello 解析、认证、TLS 1.3 密钥派生、握手消息交换和应用密钥建立。

## 核心接口

| 函数 | 说明 |
|------|------|
| `handshake(inbound, cfg, session)` → `awaitable<handshake_result>` | 主入口：五阶段握手 |
| `fallback_dest(session, inbound, raw_record)` → `awaitable<fault::code>` | 连接 dest 服务器透明代理 |
| `parse_dest(dest, host, port)` → `bool` | 解析 "host:port" 格式 |
| `fetch_dest_cert(host, port, router)` → `awaitable<pair<code, vector<uint8_t>>>` | 获取 dest 证书（DER） |

### 握手结果

| handshake_result_type | 含义 | 后续 |
|-----------------------|------|------|
| `authenticated` | Reality 认证成功 | encrypted_transport + inner_preread 有效 |
| `not_reality` | SNI 不匹配或认证失败 | raw_tls_record 保留，交下一方案 |
| `fallback` | 回退到 dest 完成 | 已建立透明代理 |
| `failed` | 错误 | error 字段包含错误码 |

## 五阶段流程

| 阶段 | 核心操作 | 失败处理 |
|------|----------|----------|
| Stage 1 | 读取 TLS 记录 → 解析 ClientHello → 提取 SNI/session_id/X25519 公钥 | 解析失败 → fallback_to_dest |
| Stage 2 | base64 解码私钥 → X25519 密钥交换 → short_id 验证 | SNI 不匹配 → not_reality；私钥无效 → fallback |
| Stage 3 | X25519 临时密钥交换 → 9 步 HKDF 派生握手密钥 → 生成 ServerHello | 密钥派生失败 → reality_key_schedule_error |
| Stage 4 | scatter-gather 发送 ServerHello + CCS + 加密握手记录 → 接收客户端 Finished | 客户端 ALERT → reality_handshake_failed |
| Stage 5 | transcript hash → 派生应用密钥 → 创建 seal 加密传输层 → 预读内层数据 | 预读失败 → io_error |

## 设计决策

### 为什么认证客户端使用合成 Ed25519 证书？

Reality 客户端已通过 X25519 密钥交换验证身份，无需再从 dest 服务器获取真实证书。`generate_reality_certificate()` 生成的合成 Ed25519 证书避免了一次网络往返。

**后果**: 合成证书有效期仅 1 小时，不适用于长寿命会话。

### 为什么 Stage 4 使用 scatter-gather 写入？

ServerHello + CCS + 加密握手记录是三个 TLS 记录，必须一次性发送。`async_write_scatter` 将三个 buffer 合并为一次系统调用。分三次写可能因 Nagle 算法导致客户端收到不完整响应，触发超时。

### 为什么 derive_and_encrypt_finished 需要重算？

`generate_server_hello` 在 Stage 3 传入 `dummy_keys`（空密钥材料），因为 ServerHello 结构构造和 Finished 计算存在循环依赖——Finished 需要 transcript hash，而 transcript 包含 ServerHello 本身。先用空密钥生成结构，再用正确密钥重算 Finished。

### 为什么 fallback 是透明代理而非直接拒绝？

Reality 的目标是**在被动探测者看来就是一个正常的网站**。拒绝非 Reality 客户端会暴露代理特征。透明代理到 dest 使服务器在探测者看来就是 dest 网站本身。

## 约束

### 握手阶段发送数据后不可 rewind

**类型**: 状态前置

**规则**: Stage 4 的 `async_write_scatter` 发送后 `polluted=true`，执行器无法回退到发送前状态。

**违反后果**: rewind 后读到不一致的传输层状态。

### consume_client_finished CCS 循环无迭代上限

**类型**: 资源上限

**规则**: while 循环处理 CCS 记录无最大迭代限制。

**违反后果**: 恶意客户端可发送无限 CCS 记录消耗 CPU。

**源码依据**: `handshake.cpp:288-366`

### read_encrypted_record 记录长度最大 65535

**类型**: 资源上限

**规则**: TLS 记录头 2 字节限定最大长度 65535，无额外上限检查。

**违反后果**: 单条记录最多 64KB 内存分配。

### transcript hash 必须包含所有握手消息

**类型**: 数据完整性

**规则**: TLS 1.3 规范要求 transcript hash 包含完整握手消息序列。

**违反后果**: 缺失任何消息导致密钥不一致，握手失败。

## 故障场景

| 阶段 | 场景 | 行为 |
|------|------|------|
| Stage 1 | ClientHello 读取失败 | 返回 `io_error` |
| Stage 1 | ClientHello 解析失败 | fallback 到 dest |
| Stage 1 | fallback dest 不可达 | 返回 `reality_dest_unreachable` |
| Stage 2 | 私钥长度 ≠ 32 | fallback 到 dest |
| Stage 2 | SNI 不匹配 | 返回 `not_reality`，交下一方案 |
| Stage 2 | short_id 不匹配 | 返回 `not_reality`，交下一方案 |
| Stage 3 | 密钥派生失败 | 返回 `reality_key_schedule_error` |
| Stage 4 | 客户端 TLS ALERT | 返回 `reality_handshake_failed` |
| Stage 4 | Finished 解密失败 | 返回 `reality_handshake_failed` |
| Stage 5 | 应用密钥派生失败 | 返回 `reality_key_schedule_error` |
| Stage 5 | 预读内层失败 | 返回 `io_error` |

### 资源泄漏向量

- **fallback 裸 socket 泄漏**: `dest_conn.release()` 后如果 `async_write` 失败，已 release 的 socket 无法自动回收
- **CCS 循环**: 无迭代限制的 while 循环，恶意客户端可发送无限 CCS 记录

### short_id 认证要点

- 纯密码学比对（HKDF + AES-256-GCM），不涉及时钟/时间窗口
- short_id 无过期机制，需通过配置轮换管理

## 跨模块契约

| 模块 | 契约 |
|------|------|
| handshake → request | 调用 `read_tls_record` + `parse_client_hello` |
| handshake → auth | Stage 2 认证 |
| handshake → keygen | Stage 3 派生握手密钥，Stage 5 派生应用密钥 |
| handshake → response | `generate_server_hello` 构造 ServerHello |
| handshake → seal | Stage 5 创建加密传输层 |
| handshake → tunnel | fallback 透明代理 |
| handshake → router | fallback 连接 dest 服务器 |
