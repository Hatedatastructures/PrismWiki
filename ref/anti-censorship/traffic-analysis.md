---
title: "流量分析"
category: "anti-censorship"
type: ref
tags: [流量对抗, 流量分析, 统计]
created: 2026-05-15
updated: 2026-05-15
---

# 流量分析

**类别**: 流量对抗

## 概述

流量分析通过分析网络流量的统计特征（包大小、时序、方向等）来识别代理流量，即使流量被加密也能检测。

## 原理

### 分析维度

| 维度 | 说明 |
|------|------|
| 包大小分布 | 代理流量的包大小分布与普通流量不同 |
| 包时序 | 代理流量的响应时间模式 |
| 流量方向 | 上行/下行流量比例 |
| 连接模式 | 连接建立和关闭的模式 |

### 检测技术

1. **统计分析**: 计算流量的统计特征
2. **机器学习**: 使用 ML 模型分类
3. **深度学习**: 使用神经网络识别

## 在 Prism 中的应用

Prism 通过以下方式对抗流量分析：
- **多路复用**: 混合多个流的流量
- **padding**: 填充隐藏真实包大小
- **流量伪装**: 模拟正常流量模式

## 参考资料

- [Website Fingerprinting](https://en.wikipedia.org/wiki/Traffic_analysis)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — TLS 指纹
