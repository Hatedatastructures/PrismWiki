---
title: "ADR 004: 协议智能识别"
created: 2026-05-23
updated: 2026-05-23
layer: dev
tags: [dev, adr, architecture, recognition, protocol]
status: accepted
---

# ADR 004: 协议智能识别

## 状态

已接受 (Accepted)

## 背景

Prism 需要在单个监听端口上同时处理多种协议（HTTP、SOCKS5、Trojan、VLESS、SS2022），这些协议的流量特征各不相同：

- **HTTP**: 明文请求，以 HTTP 方法名开头（GET、POST、CONNECT 等）
- **SOCKS5**: 明文二进制，首字节 `0x05`
- **TLS**: 加密流量，前两字节 `0x16 0x03`，内层承载 Trojan/VLESS/HTTP 等
- **SS2022**: 无正特征，流量外观类似随机数据

此外，TLS 连接还需要进一步识别伪装方案（Reality、ShadowTLS、Restls 等），这在 [[dev/adr/003-stealth-scheme-pipeline|ADR 003]] 中已有描述。

### 方案 A: 端口分离

每种协议使用不同端口。

- **优势**: 实现最简单，无识别开销
- **劣势**: 多端口暴露增加被探测风险；端口数量多管理复杂；不符合生产环境单端口部署需求

### 方案 B: 协议前缀标记

在数据流前添加协议标识。

- **优势**: 准确率 100%
- **劣势**: 不兼容标准客户端；破坏协议标准；增加流量特征

### 方案 C: 三阶段识别流水线

预读少量数据 -> 协议检测 -> TLS 伪装方案识别，分层递进。

- **优势**: 单端口多协议；兼容标准客户端；无额外协议开销；支持协议伪装
- **劣势**: 检测有延迟（需要读取数据）；某些协议特征不明显（SS2022）需要排除法

## 决策

**选择方案 C: 三阶段识别流水线**

### 流水线架构

```
Phase 1: Probe（外层探测）
  |
  | 预读 24 字节
  | detect(): SOCKS5 -> TLS -> HTTP -> SS2022(fallback)
  |
  v
Phase 2: Identify（TLS 伪装识别）
  |
  | 仅当 Phase 1 检测为 TLS 时执行
  | 读取完整 ClientHello
  | 分层检测: Tier 0 -> Tier 1 -> Tier 2
  | 执行匹配的伪装方案握手
  |
  v
Phase 3: Execute（执行）
  |
  | 返回: { transport, detected, preread }
  | dispatch 根据 detected 获取 handler
  | handler 执行协议处理
```

### Phase 1: Probe（外层探测）

预读前 24 字节数据，通过魔术字节快速判断协议类型。检测采用排除法，按优先级依次匹配：

```cpp
detect(peek_data):
  if peek_data[0] == 0x05       -> socks5
  if peek_data[0:2] == 0x16 0x03 -> tls
  if starts_with_http_method     -> http
  if none matched                -> shadowsocks (fallback)
```

**SS2022 的特殊性**: SS2022 没有正特征，其首字节是随机 salt，任何值都可能出现（包括恰好为 `0x16`，约 1/256 概率）。因此 SS2022 使用排除法判定 -- 当数据不匹配 SOCKS5、TLS、HTTP 时，认为是 SS2022。

**数据复用**: 预读的 24 字节不会丢弃，而是传递给后续阶段作为 `preread` 数据。这避免了额外的网络读取，降低了延迟。

### Phase 2: Identify（TLS 伪装识别）

仅在 Phase 1 检测为 TLS 时执行。核心流程：

1. **Read**: 从传输层读取完整 TLS ClientHello（处理已有 24 字节 preread）
2. **Parse**: 解析 ClientHello 结构，提取 SNI、session_id、key_share 等特征
3. **Detect**: 执行分层检测管道（[[dev/adr/003-stealth-scheme-pipeline|ADR 003]]）
4. **Execute**: 按候选列表顺序执行伪装方案握手

### Phase 3: 内层协议检测

TLS 握手完成后，需要识别内层承载的应用层协议。`detect_tls()` 函数在 TLS 应用数据中检测：

```cpp
detect_tls(peek_data):
  // 阶段 1: HTTP（最少 4 字节）
  if starts_with_http_method -> http

  // 阶段 2: VLESS（最少 22 字节）
  if byte[0]==0x00 && byte[17]==0x00 && valid_cmd && valid_atyp -> vless

  // 阶段 3: Trojan（最少 60 字节）
  if 56 hex chars + CRLF + valid_cmd + valid_atyp -> trojan

  // 60+ 字节无法匹配 -> unknown
```

### 统一入口

`recognize()` 函数封装完整的三阶段流程：

```cpp
auto recognize(recognize_context ctx) -> net::awaitable<recognize_result>;
```

输入 `recognize_context` 包含传输层、配置、路由器、会话上下文和帧内存池。输出 `recognize_result` 包含最终传输层、检测到的协议类型、预读数据和错误码。

### 完整识别流程示例

以一个同时启用 SOCKS5、Trojan、VLESS + Reality、SS2022 + ShadowTLS 的 Prism 服务器为例：

```
客户端连接到达 -> 预读 24 字节:

Case 1: SOCKS5 客户端
  peek_data[0] = 0x05
  detect() -> socks5
  不需要 Phase 2（非 TLS）
  结果: { detected: socks5, transport: 原始 socket }

Case 2: VLESS + Reality 客户端
  peek_data[0:2] = 0x16 0x03 (TLS)
  detect() -> tls
  进入 Phase 2:
    读取 ClientHello -> SNI = www.microsoft.com
    Tier 0 sniff(): session_id[0:3] == [0x01, 0x08, 0x02] -> 命 Reality
    执行 Reality 握手 -> 成功
  detect_tls(): VLESS 特征字节 -> vless
  结果: { detected: vless, transport: Reality 加密传输 }

Case 3: SS2022 + ShadowTLS 客户端
  peek_data[0:2] = 0x16 0x03 (TLS)
  detect() -> tls
  进入 Phase 2:
    读取 ClientHello -> SNI = www.apple.com
    Tier 0 sniff(): 不命中
    Tier 1 verify(): ShadowTLS HMAC 验证 -> 命中
    执行 ShadowTLS 握手 -> 成功，获得内层数据
  detect_tls(): 不匹配 HTTP/VLESS/Trojan -> unknown
  fallback: SS2022（排除法）
  结果: { detected: shadowsocks, transport: ShadowTLS 传输 }

Case 4: 主动探测（扫描器）
  peek_data[0:2] = 0x16 0x03 (TLS)
  detect() -> tls
  进入 Phase 2:
    Tier 0/1/2 全部未命中
    Fallback: Native TLS
  结果: 表现为标准 HTTPS 服务器，返回伪装页面
```

### 识别延迟分析

| 协议 | Phase 1 | Phase 2 | 总读取 | 额外延迟 |
|------|---------|---------|--------|----------|
| SOCKS5 | 1 字节 | - | 1 字节 | ~0 |
| HTTP | 4 字节 | - | 4 字节 | ~0 |
| TLS (Reality) | 24 字节 | ClientHello | ~300 字节 | 1 RTT |
| TLS (ShadowTLS) | 24 字节 | ClientHello + HMAC | ~300 字节 | 1-2 RTT |
| SS2022 (裸) | 24 字节 | - | 24 字节 | ~0 |

> 参考: [[core/recognition/layered_pipeline|识别管道]]

## 后果

### 优势

1. **单端口多协议**: 单个监听端口同时支持所有协议，降低了部署复杂度和被探测的风险。通过 SNI 路由可以进一步区分不同伪装方案。

2. **低延迟检测**:
   - Phase 1 仅需读取 24 字节，一次 `async_read_some` 即可完成
   - Phase 2 的 ClientHello 通常在数百字节内，读取成本低
   - Tier 0 方案（Reality）零成本检测，无额外延迟

3. **兼容性**: 完全兼容标准客户端，不需要修改客户端协议实现。客户端无需知道服务端支持多协议。

4. **安全降级**:
   - 主动探测者发送非 TLS 流量时，Phase 1 正确识别协议类型
   - 发送 TLS 流量但无有效特征时，fallback 到 Native TLS，表现为标准 HTTPS 服务
   - SS2022 的排除法判定不会将合法 TLS 流量误判为 SS2022

5. **数据复用**: 预读数据通过 `preread` 传递给后续阶段，避免重复读取。整个识别过程最多读取一次 ClientHello。

### 劣势

1. **SS2022 识别的不确定性**:
   - SS2022 无正特征，使用排除法判定。理论上存在将未知协议误判为 SS2022 的风险
   - SS2022 的首字节有约 1/256 概率恰好为 `0x16`，可能与 TLS 混淆。通过检查第二字节 `0x03` 来降低误判率
   - 某些新协议的流量可能被误判为 SS2022

2. **识别延迟**:
   - Phase 1 需要等待至少 24 字节数据到达，引入最小一个 RTT 的延迟
   - Phase 2 需要读取完整 ClientHello，对于大 ClientHello（如携带多个扩展）可能需要多次读取
   - Trojan 识别需要 60 字节，数据不足时无法判定

3. **VLESS 识别的脆弱性**:
   - VLESS 的特征字节检测（byte[0]==0x00, byte[17]==0x00, 合法命令, 合法地址类型）可能被巧合匹配
   - 未来 VLESS 协议变更可能导致识别规则失效

4. **协议优先级固定**:
   - SOCKS5 总是优先于 TLS 检测，如果 SS2022 首字节恰好为 `0x05`，会被误判为 SOCKS5
   - 但 SS2022 首字节为 salt 的一部分，只有 `0x05` 一个值会被误判为 SOCKS5，概率约 1/256

### 缓解措施

- 通过 [[dev/testing/overview|测试]] 覆盖所有协议的识别场景
- 详细的日志记录每阶段检测的结果和原因
- 协议识别支持 `unknown` 状态，允许上层做进一步处理
- SS2022 的 1/256 误判概率在实际使用中可接受

## 相关页面

- [[dev/adr/003-stealth-scheme-pipeline|ADR 003: 伪装方案管道]]
- [[core/recognition/layered_pipeline|识别管道]]
- [[docs/protocol-matrix|协议能力矩阵]]
- [[dev/modules|模块架构]]
- [[core/protocol/overview|协议模块]]
