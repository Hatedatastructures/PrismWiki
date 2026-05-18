---
title: Reject
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, reject]
---
# Reject 拒绝协议

Reject 用于拒绝或丢弃连接请求。

## 功能概述

Reject 类型：
- `REJECT` - 返回拒绝响应
- `REJECT-DROP` - 丢弃连接（静默）
- `PASS` - 放行（调试用）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/reject.go` | Reject 适配器 |

## YAML 配置示例

```yaml
proxies:
  - name: "REJECT"
    type: reject

  - name: "REJECT-DROP"
    type: reject

  - name: "PASS"
    type: reject
```

## 类型对比

| 类型 | TCP | UDP | 说明 |
|------|-----|-----|------|
| `REJECT` | 返回 EOF | 返回 EOF | 明确拒绝 |
| `REJECT-DROP` | 静默丢弃 | 静默丢弃 | 不响应 |
| `PASS` | 放行 | 放行 | 调试用 |

## 使用场景

| 场景 | 推荐类型 |
|------|----------|
| 广告拦截 | `REJECT-DROP` |
| 恶意域名 | `REJECT-DROP` |
| 调试测试 | `REJECT` 或 `PASS` |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Reject 实现 |
| Pipeline | 完全兼容 | 虚拟连接 |
| Rules | 完全兼容 | 规则匹配 |

## 相关文档

- [[direct]] - Direct 协议
- [[ref/mihomo/rules|rules]] - 规则配置