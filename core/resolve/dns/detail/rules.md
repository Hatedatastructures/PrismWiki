---
title: "rules_engine -- DNS 域名规则引擎"
layer: core
source: "include/prism/resolve/dns/detail/rules.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, rules, trie, wildcard, adblock]
created: 2026-05-17
updated: 2026-05-28
related:
  - core/resolve/dns/config
  - core/resolve/dns/dns
---

# rules_engine -- DNS 域名规则引擎

> 源码位置: `include/prism/resolve/dns/detail/rules.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

`rules_engine` 提供基于反转域名基数树（trie）的高效域名规则匹配系统。支持精确匹配、通配符匹配和后缀匹配三种模式，用于实现静态地址解析、广告屏蔽和 CNAME 重定向。

## rule_result 结构

| 字段 | 类型 | 规则类型 | 说明 |
|------|------|----------|------|
| `addresses` | `memory::vector<ip::address>` | 地址规则 | 静态 IP 映射 |
| `cname` | `memory::string` | CNAME 规则 | 重定向目标域名 |
| `negative` | `bool` | 否定规则 | 返回 NXDOMAIN |
| `blocked` | `bool` | 否定规则 | 标记被拦截 |
| `no_cache` | `bool` | 特殊规则 | 跳过缓存 |

## domain_trie -- 反转域名基数树

### 设计原理

域名标签按反转顺序存储：`"www.example.com"` 变为 `["com", "example", "www"]`。

**为什么反转存储**: 后缀匹配等价于前缀遍历。`*.example.com` 的通配符只需在 `"example"` 节点标记一次。多个子域名（`www.example.com`, `api.example.com`）共享 `root -> "com" -> "example"` 路径，通配符检查只需回溯祖先节点。

### node 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `children` | `unordered_map<string, unique_ptr<node>>` | 子标签映射 |
| `value` | `std::any` | 规则关联值 |
| `is_end` | `bool` | 是否为规则终点 |
| `wildcard` | `bool` | 是否为通配符标记 |

### 接口

| 方法 | 说明 |
|------|------|
| `insert(domain, value)` | 去末尾点号 -> 检测 `"*."` 前缀设置 wildcard -> 按点分割标签 -> 反转 -> 逐级创建节点 -> 终点设 `is_end=true` |
| `search(domain) -> optional<any>` | 分割反转标签 -> 沿树精确匹配 -> 不完整则回溯检查 `wildcard` -> 返回 value 或 nullopt |
| `match(domain) -> bool` | 简化版 search，只返回是否命中 |
| `split_labels(domain) -> vector<string>` | `"www.example.com"` -> `["com", "example", "www"]` |
| `to_lower(s) -> string` | 域名统一转小写 |

### 匹配优先级

精确匹配（路径完整且 `is_end=true`） > 通配符匹配（路径不完整，回溯检查 `wildcard`，从最深节点向上） > 无匹配。

### 通配符规则

`*.example.com` 匹配 `www.example.com`、`api.example.com`、`www.api.example.com`，但**不匹配** `example.com`（通配符要求至少匹配一级子域）。

实现方式：去掉 `"*"` 前缀，在 `"example"` 节点标记 `wildcard=true`。搜索时若精确路径不存在，回溯祖先节点检查 `wildcard` 标志。

## rules_engine 类

### 双树架构

维护两棵独立的基数树：
- `address_trie_` -- 地址规则 + 否定规则（广告屏蔽）
- `cname_trie_` -- CNAME 重定向规则

### 接口

| 方法 | 说明 |
|------|------|
| `add_addr_rule(domain, ips)` | 将地址映射插入 address_trie_ |
| `add_neg_rule(domain)` | 将否定规则插入 address_trie_，设置特殊标记 |
| `add_cname(domain, target)` | 将 CNAME 重定向插入 cname_trie_ |
| `match(domain) -> optional<rule_result>` | 地址树优先查找 -> CNAME 树查找 -> 合并结果 |

### 统一匹配优先级

地址规则命中 > CNAME 规则命中。地址命中时直接返回 IPs，跳过 DNS 查询。仅 CNAME 命中时递归 resolve 目标域名。否定规则命中时返回 NXDOMAIN。

## 匹配示例

| 规则 | 查询 | 结果 |
|------|------|------|
| `"example.com"` -> 1.2.3.4 | `"example.com"` | 命中：1.2.3.4（精确） |
| `"example.com"` -> 1.2.3.4 | `"www.example.com"` | 未命中 |
| `"*.example.com"` -> 5.6.7.8 | `"example.com"` | 未命中（通配符需至少一级子域） |
| `"*.example.com"` -> 5.6.7.8 | `"www.example.com"` | 命中：5.6.7.8（通配符） |
| `"*.ad.com"` -> negative | `"track.ad.com"` | 命中：`{negative=true, blocked=true}` |
| `"alias.example.com"` -> CNAME "real.example.com" | `"alias.example.com"` | 命中：`{cname="real.example.com"}` |

## 性能特性

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| insert | O(L) | L = 域名标签数 |
| search | O(L) | 两棵树各 O(L)，最多 L 次哈希查找 |
| match | O(L) | 两棵树各搜一次 |

透明哈希查找：使用 `string_view` 查找避免构造临时 `string`，零分配。

## 调用链

```
resolver::resolve(host)
  |
  +-> rules_engine::match(host)
  |     +-> address_trie::search(host) -> 精确/通配符 -> 返回 addresses 或 negative
  |     +-> cname_trie::search(host) -> 返回 cname 目标
  +-> 合并结果 -> rule_result
      地址命中 -> 直接返回 IPs
      CNAME 命中 -> 重定向解析
      否定命中 -> 返回 NXDOMAIN
```

## 规则加载流程

```
config 加载
    -> 遍历 address_rules: negative=true -> add_neg_rule / negative=false -> add_addr_rule
    -> 遍历 cname_rules: add_cname
    -> 两棵树构建完成，后续所有 resolve 调用共享规则引擎
```

## 参见

- [[core/resolve/dns/config|address_rule]] -- 地址规则配置
- [[core/resolve/dns/config|cname_rule]] -- CNAME 规则配置
- [[core/resolve/dns/dns|resolver]] -- DNS 解析器接口
