---
title: Restls
created: 2026-05-13
updated: 2026-05-13
type: protocol
tags:
  - stealth
  - tls
  - tier2
related:
  - "[[shadowtls]]"
  - "[[anytls]]"
  - "[[trusttunnel]]"
  - "[[ech]]"
---

# Restls

Restls 是 Prism stealth 模块中的 **Tier 2** 伪装方案，通过模拟真实 TLS 流量来隐藏代理特征。
与 ShadowTLS 不同，Restls 没有 ClientHello 独占特征，完全依赖 SNI 匹配进行路由。

协议参考: https://github.com/3andne/restls

## 工作原理

Restls 的核心思路是**完全模仿真实 TLS 流量模式**：

1. 读取客户端 TLS ClientHello
2. 建立到后端 TLS 服务器的真实连接
3. 完成标准 TLS 握手（与后端建立完整 TLS 会话）
4. 在 TLS 应用数据层中验证客户端身份
5. 认证成功后，使用 `restls-script` 控制流量模式

关键区别：Restls 的认证发生在 **TLS 应用数据层**，而非 ClientHello 阶段。
这使得 Restls 的 TLS 握手与正常 TLS 完全一致，无法通过 ClientHello 分析检测。

## 与 Reality 的区别

| 特性 | Restls | Reality |
|------|--------|---------|
| Tier | 2（无独占特征） | 1（有独占特征） |
| 认证阶段 | TLS 应用数据 | ClientHello 扩展 |
| ClientHello | 完全标准 | 有特殊扩展 |
| 流量控制 | restls-script 脚本 | 无 |
| 证书来源 | 真实后端证书 | 真实后端证书 |
| DPI 抗性 | 高（无 ClientHello 特征） | 中（有可检测扩展） |

## Detection

Restls 在 Prism 检测流水线中的位置：

- **Tier 0 sniff**：无（无 ClientHello 快速特征）
- **Tier 1 verify**：无（不实现 verify 接口）
- **Tier 2 guess**：返回基础分 100，依赖 SNI 路由阶段的过滤

由于 Restls 没有任何 ClientHello 独占特征，它的检测完全依赖 SNI 白名单匹配。
这意味着即使配置了 Restls，只有 SNI 匹配的连接才会尝试 Restls 握手。

## 配置项

```json
{
  "server_names": ["example.com"],
  "host": "example.com:443",
  "password": "my-secret-password",
  "version_hint": "tls13",
  "restls_script": "300?100,400~100,<1",
  "handshake_timeout_ms": 5000
}
```

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `server_names` | array | 是 | SNI 白名单 |
| `host` | string | 是 | TLS 后端目标 host:port |
| `password` | string | 是 | 认证密码 |
| `version_hint` | string | 否 | 版本提示：`tls12` 或 `tls13` |
| `restls_script` | string | 否 | 流量控制脚本 |
| `handshake_timeout_ms` | uint32 | 否 | 握手超时（默认 5000ms） |

## Restls Script 语法

流量控制脚本用于隐藏代理特征，控制数据发送的大小和时机：

| 语法 | 含义 |
|------|------|
| `300?100` | 发送 300 字节，等待 100ms |
| `400~100` | 等待 100ms 后发送 400 字节 |
| `<1` | 等待客户端数据 |

脚本使流量模式看起来像正常的 HTTPS 浏览行为，而非代理隧道。

## 实现状态

当前实现状态：**基础框架已实现，认证逻辑待完善**。

已完成：
- 方案注册和生命周期管理
- SNI 白名单匹配
- Tier 2 guess 接口
- 可靠传输层穿透（`find_reliable`）

待实现：
- 完整的 TLS 应用数据层认证
- restls-script 流量控制引擎
- 后端 TLS 会话管理

## 相关源码

- `include/prism/stealth/restls/scheme.hpp` — 方案类定义
- `include/prism/stealth/restls/config.hpp` — 配置结构
- `src/prism/stealth/restls/scheme.cpp` — 方案实现
