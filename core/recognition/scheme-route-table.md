---
tags: [recognition, scheme-route-table]
layer: core
module: recognition
source: include/prism/recognition/routes.hpp
title: route_table
---

# route_table

SNI 路由表，根据 ClientHello SNI 快速路由到对应伪装方案。

## 设计决策

### 为什么用独立路由表而非在每个 scheme 中检查 SNI？

分层检测管道的 Tier 2 需要一次查找匹配所有方案的 SNI。如果每个 scheme 独立检查，需要遍历所有 scheme 实例逐一调用 `snis()` 方法，O(n) 复杂度。路由表预构建 SNI → 方案映射，查找 O(log n)。同时路由表支持多方案共享同一 SNI，返回候选列表。

**后果**: 配置热加载时需重建路由表（`build()` 是静态工厂方法）。

## 核心方法

| 方法 | 说明 |
|------|------|
| `build(cfg)` | 从配置构建路由表（静态工厂） |
| `lookup(sni)` | 根据 SNI 查找匹配方案，返回候选列表 |
| `matches_any(sni)` | 检查 SNI 是否匹配任意方案 |
| `registered_snis()` | 获取所有已注册的 SNI 列表（调试用） |
| `empty()` | 检查路由表是否为空 |

## 约束

### SNI 精确匹配

**类型**: 调用顺序

**规则**: `lookup()` 使用精确匹配（`memory::map` 的 `find()`），不支持通配符（如 `*.example.com`）。

**违反后果**: 通配符 SNI 配置不会匹配任何请求。必须在配置中列出每个具体域名。

**源码依据**: `routes.hpp:87`

### 空 SNI 返回空列表

**类型**: 状态前置

**规则**: 空 `string_view` 的 SNI 查找返回空列表，不会匹配任何方案。

**源码依据**: `routes.hpp:60`

## 引用关系

### 依赖

- [[core/instance/config|psm::config]]：全局配置

### 被引用

- [[core/recognition/layered_pipeline|layered_pipeline]]：Tier 2 检测使用
- [[core/recognition/recognition|recognition]]：识别入口引用