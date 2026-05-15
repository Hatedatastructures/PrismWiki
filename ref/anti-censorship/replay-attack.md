---
title: "重放攻击"
category: "anti-censorship"
type: ref
tags: [流量对抗, 重放攻击, 安全]
created: 2026-05-15
updated: 2026-05-15
---

# 重放攻击

**类别**: 流量对抗

## 概述

重放攻击是攻击者截获合法数据包并重新发送的攻击方式。在代理场景中，攻击者可能重放认证数据包来冒充合法用户。

## 原理

### 攻击方式

1. **截获**: 攻击者截获客户端发送的认证数据包
2. **重放**: 攻击者将截获的数据包重新发送给服务器
3. **冒充**: 服务器误认为攻击者是合法用户

### 防御机制

- **时间戳**: 在数据包中包含时间戳
- **序列号**: 使用递增的序列号
- **一次性令牌**: 使用一次性认证令牌
- **滑动窗口**: 使用滑动窗口验证序列号

## 在 Prism 中的应用

Prism 使用防重放窗口机制：
- **Shadowsocks**: 使用 replay 模块检测重放
- **时间窗口**: 维护已见数据包的时间窗口

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| replay | 防重放窗口 | [[protocol/shadowsocks/replay|replay]] |

## 参考资料

- [Replay Attack](https://en.wikipedia.org/wiki/Replay_attack)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/mitm-hijack|中间人劫持]] — MITM 攻击
