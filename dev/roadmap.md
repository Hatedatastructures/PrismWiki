---
layer: dev
title: Prism 开发路线图
tags: [dev, roadmap]
---

# Prism 开发路线图

> 编制日期: 2026/05/01
> 修订日期: 2026/05/06
> 时间跨度: 2026/05/01 — 2026/06/30 (共 8 周)

---

## 一、项目现状总览

### 已完成（生产就绪）

| 模块 | 状态 | 关键文件 |
|------|------|---------|
| HTTP 代理 | 完成 | `src/prism/pipeline/protocols/http.cpp` |
| SOCKS5 (TCP+UDP) | 完成 | `src/prism/pipeline/protocols/socks5.cpp` |
| Trojan (TCP+UDP+MUX) | 完成 | `src/prism/pipeline/protocols/trojan.cpp` |
| VLESS (TCP+UDP+MUX) | 完成 | `src/prism/pipeline/protocols/vless.cpp` |
| SS2022 AEAD (TCP+UDP) | 完成 | `src/prism/pipeline/protocols/shadowsocks.cpp` |
| Reality TLS 伪装 | 完成 | `src/prism/stealth/reality/` |
| ShadowTLS v2 | 完成 | `src/prism/stealth/shadowtls/` |
| smux v1 多路复用 | 完成 | `src/prism/multiplex/smux/` |
| yamux 多路复用 | 完成 | `src/prism/multiplex/yamux/` |
| 七阶段 DNS 解析 | 完成 | `src/prism/resolve/dns/` |
| Happy Eyeballs | 完成 | `include/prism/channel/eyeball/` |
| 连接池 + 健康检查 | 完成 | `src/prism/channel/connection/pool.cpp` |
| PMR 内存管理 | 完成 | `include/prism/memory/` |

### 未完成 / TODO

| # | TODO 位置 | 问题描述 | 优先级 |
|---|----------|---------|--------|
| 1 | `restls/scheme.cpp:68` | Restls 握手为 TODO 桩 | CRITICAL |
| 2 | `router.cpp:52` | 正向代理返回 `not_supported` | HIGH |
| 3 | `reality/handshake.cpp:429` | 类型统一 TODO | MEDIUM |

---

## 二、性能目标

| 指标 | Before | 第 8 周目标 | 业界对标 |
|------|--------|------------|---------|
| AES-256-GCM | 205 Mi/s | **13.7 Gi/s ✅** | shadowsocks-rust 1-2 Gi/s |
| X25519 交换 | 81.5 us | **21.4 us ✅** | WireGuard ~40 us |
| 全局池 4T | 3530 ns | <100 ns | — |
| 连接 P50 | 95 us | 85 us | Envoy ~100 us |
| 连接 P99 | 336 us | <200 us | — |
| TCP 64B | 20.1 us | 10 us | — |
| TCP 128KB | 7.11 Gi/s | 7.11 Gi/s | HAProxy ~10 Gbps |

---

## 三、第一阶段 (第 1-2 周): 关键性能修复

### 已完成

| 任务 | 提交 | 结果 |
|------|------|------|
| AES-NI 启用 | `a3f3799` | AES-256-GCM 67x 提升 |
| X25519 优化 | `a3f3799` | X25519 3.8x 提升 |

### 待完成

| 任务 | 涉及文件 | 目标 |
|------|---------|------|
| 全局内存池分片化 | `pool.hpp` | 4T 下 <150 ns |
| P99 尾延迟优化 | `pool.hpp`, `racer.hpp` | P99 <250 us |
| 小报文 TCP 优化 | `reliable.hpp` | 64B TCP 12-15 us |
| 性能回归基准 | `RegressionBench.cpp` | 自动回归检测 |

---

## 四、第二阶段 (第 3-4 周): 补全未完成功能

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| Restls 完整握手 | `restls/scheme.cpp`, 新建 `handshake.cpp` | 替换 TODO 桩 |
| 类型统一 | `reality/handshake.cpp:429` | 消除 `client_hello_info` 冗余 |
| ECH/AnyTLS 方案 | 新建 `stealth/ech/`, `stealth/anytls/` | 新架构下重新接入 |
| 消除 dynamic_cast | `transmission.hpp`, 4 处 | 使用 `raw_socket()` 虚方法 |
| 正向代理模式 | `router.cpp:52` | HTTP CONNECT 实现 |
| Happy Eyeballs 重构 | `racer.hpp` | 配置化 + 取消逻辑完善 |

---

## 五、第三阶段 (第 5-6 周): 代码质量与架构优化

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| 提取协议处理器共享模式 | 新建 `handler_utility.hpp` | 减少 ~150 行重复 |
| HTTP 处理器纳入重构 | `http.cpp` | 使用共享辅助函数 |
| 补充 6 个测试文件 | `tests/` | Eyeball, Health, Tunnel 等 |
| RTTI 移除 | `CMakeLists.txt` | `-fno-rtti` 编译选项 |
| 性能回归 CI | `benchmark_check.sh` | 自动回归检查 |

---

## 六、第四阶段 (第 7-8 周): 新功能与生产加固

| 任务 | 涉及文件 | 说明 |
|------|---------|------|
| ShadowTLS v3 | `shadowtls/handshake.cpp` | 多用户支持 |
| WebSocket 传输 | 新建 `websocket.hpp/cpp` | RFC 6455 实现 |
| 监控指标系统 | 新建 `trace/metrics.hpp` | Prometheus/JSON 导出 |
| 压力测试升级 | `stresses/` | 并发、稳定性、模糊测试 |
| 文档更新 | `README.md`, `docs/` | 特性矩阵、性能数字 |

---

## 七、依赖关系

```
第 1 周 (性能修复):
  ├─ AES-NI ✅
  ├─ X25519 ✅
  ├─ 全局池分片化 ─── 可并行
  └─ P99 尾延迟 ──── 可并行

第 2 周 (验证):
  ├─ 小报文 TCP 优化
  ├─ 性能回归基准
  └─ 第一阶段整体验证

第 3 周 (补全功能):
  ├─ Restls 握手 ───── 可并行
  ├─ 类型统一 ──────── 可并行
  ├─ ECH/AnyTLS ────── 可并行
  └─ 消除 dynamic_cast

第 4 周 (路由/DNS):
  ├─ 正向代理 ──────── 可并行
  ├─ Happy Eyeballs ── 可并行
  ├─ DNS 截断测试
  └─ Yamux 窗口测试

第 5 周 (代码质量):
  ├─ 提取共享处理器
  └─ HTTP 重构 ──── 依赖上一项

第 6 周 (测试/RTTI):
  ├─ 补充 6 个测试 ── 可并行
  ├─ RTTI 移除 ────── 依赖 dynamic_cast 消除
  └─ 性能回归 CI ─── 可并行

第 7 周 (新功能):
  ├─ ShadowTLS v3 ─── 可并行
  └─ WebSocket 传输 ── 可并行

第 8 周 (生产加固):
  ├─ 指标系统 ──────── 可并行
  ├─ 压力测试 ──────── 可并行
  ├─ 文档更新 ──────── 可并行
  └─ 最终验证
```

---

## 八、上层架构计划

本 8 周计划聚焦于性能修复 + TODO 清零 + 代码质量 + WebSocket/ShadowTLS v3。

完成后可启动的更大规模架构：

| 上层功能 | 与本计划关系 |
|----------|-------------|
| Outbound 出站代理抽象 | 独立，可后续启动 |
| Traffic Controller | 独立，可后续启动 |
| Rule 规则引擎 | 独立，可后续启动 |
| Proxy Group | 依赖 Outbound 抽象 |
| QUIC / Hysteria2 / TUIC | 独立大工程 |
| REST API | 指标系统为其打基础 |
| TUN / WireGuard / VMess | 独立大工程 |

---

## 相关页面

- [[dev/performance/overview|performance/overview]] — 性能架构概览
- [[dev/performance/report|performance/report]] — 完整基准报告
- [[dev/extending/overview|extending/overview]] — 扩展开发概览
- [[dev/bugs/template|bugs/template]] — Bug 报告模板