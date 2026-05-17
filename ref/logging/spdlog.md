---
title: spdlog 日志库
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [logging, spdlog, cpp]
---

# spdlog 日志库

> spdlog 是高性能 C++ 日志库，Prism 使用其作为日志后端。

---

## 概述

spdlog 特点：
- 高性能异步日志
- 多种输出目标（文件、控制台、网络）
- 格式化支持
- 线程安全

---

## 在 Prism 中的应用

Prism 使用 spdlog 实现 trace 模块：
- 异步日志队列
- 多线程写入
- 文件滚动

详见 [[core/trace/overview|Trace 模块]]。

---

## 相关参考

- spdlog 官方文档: https://github.com/gabime/spdlog
- [[core/trace/overview|Trace 模块实现]]