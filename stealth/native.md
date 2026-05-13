---
title: "Native TLS 方案"
created: 2026-05-13
updated: 2026-05-13
type: stealth
tags: [native, tls, fallback, stealth, tier-2]
related: ["[[stealth]]", "[[reality]]", "[[shadowtls]]", "[[restls]]", "[[pipeline]]"]
---

# Native TLS 方案

> 源码：`include/prism/stealth/native.hpp` · `src/prism/stealth/native.cpp`
> 相关：[[stealth]] | [[reality]] | [[shadowtls]] | [[restls]] | [[pipeline]]

## 1. 概述

Native 是 Prism 的**原生 TLS 伪装方案**，属于 Tier 2 级别（最低优先级）。它不做任何伪装，直接使用服务器的真实证书与客户端进行标准 TLS 握手。当所有其他伪装方案（Reality、ShadowTLS、Restls 等）均未匹配时，Native 作为兜底 fallback 处理连接。

## 2. Tier 分级与优先级

Native 在三阶分层检测架构中的位置：

| 层级 | 方案 | 特征 |
|------|------|------|
| Tier 0 | Reality | session_id[0:3] 独占字节标记 |
| Tier 1 | ShadowTLS | HMAC 验证 |
| Tier 2 | Restls / **Native** | 模糊匹配 / SNI 路由 |

Native 的 `guess()` 返回固定 score=50（最低），确保所有其他方案优先匹配。当所有方案的 `sniff()`、`verify()`、`guess()` 均未命中时，Native 兜底接管。

关键属性：

```cpp
auto tier() const noexcept -> std::uint8_t override { return 2; }      // Tier 2
auto unique() const noexcept -> bool override { return false; }         // 无独占特征
auto active(const psm::config &cfg) const noexcept -> bool override {   // 始终启用
    return true;
}
auto weight() const noexcept -> std::uint16_t override { return 50; }   // 最低权重
```

## 3. 握手流程

`native::handshake()` 的执行步骤：

1. **TLS 握手** — 调用 [[pipeline]] 原语 `primitives::ssl_handshake()`，使用服务器真实 SSL 上下文完成标准 TLS 服务端握手
2. **包装加密传输** — 将 TLS 流包装为 `channel::transport::encrypted` 传输对象
3. **内层协议探测** — 从 TLS 明文层预读最多 128 字节，使用 `protocol::analysis::detect_tls()` 检测内层协议类型（Trojan/VLESS/HTTP 等）
4. **返回结果** — 返回 `handshake_result`，包含加密传输层、检测到的内层协议类型和预读数据

```
客户端 ──TLS(真实证书)──▶ Native 握手 ──▶ 内层协议探测 ──▶ dispatch 分发
```

## 4. 与 Reality / ShadowTLS 的区别

| 特性 | Native | Reality | ShadowTLS |
|------|--------|---------|-----------|
| 证书来源 | 服务器真实证书 | 伪装目标站点证书 | 第三方 TLS 代理 |
| 抗检测能力 | 无 | 强（证书不可伪造） | 中（依赖代理可信度） |
| 配置复杂度 | 零配置 | 需密钥对 + SNI 白名单 | 需代理地址 |
| 优先级 | Tier 2（最低） | Tier 0（最高） | Tier 1 |
| 适用场景 | 测试、无需抗检测 | 生产环境首选 | 备选方案 |

## 5. 使用场景

- **测试与调试环境** — 不需要 DPI 规避时，直接使用原生 TLS 简化部署
- **内网代理** — 流量不经过审查的网络环境
- **开发阶段** — 验证协议处理逻辑，排除伪装方案的干扰
- **兜底保障** — 确保所有 TLS 连接都能被处理，不会因方案不匹配而丢弃连接

## 6. 设计考量

Native 始终返回 `active(cfg) = true`，这是有意为之的设计：作为兜底方案，它必须在任何配置下都可用。如果禁用 Native，一旦所有伪装方案都未匹配，连接将被直接丢弃，导致服务不可用。

内层协议探测的阈值为 60 字节（`trojan_min`），与 Trojan 协议最小报文长度一致。探测上限为 128 字节，足够覆盖所有支持的协议特征。
