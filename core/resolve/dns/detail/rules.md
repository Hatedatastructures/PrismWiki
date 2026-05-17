---
title: "rules_engine — DNS 域名规则引擎"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/rules.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, rules, trie, wildcard, adblock]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/config
  - core/resolve/dns/dns
---

# rules_engine — DNS 域名规则引擎

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/rules.hpp`
> 模块: [[resolve|resolve]] / [[resolve/dns|dns]] / detail

## 组件定位

`rules_engine` 提供基于反转域名基数树（trie）的高效域名规则匹配系统。支持精确匹配、通配符匹配和后缀匹配三种模式，用于实现静态地址解析、广告屏蔽和 CNAME 重定向。

## rule_result 结构

```cpp
struct rule_result {
    memory::vector<net::ip::address> addresses; // 静态地址列表
    memory::string cname;                       // CNAME 目标域名
    bool negative{false};                       // 否定规则（广告屏蔽）
    bool no_cache{false};                       // 跳过缓存
    bool blocked{false};                        // 被拦截
};
```

各字段语义：
| 字段 | 规则类型 | 说明 |
|------|----------|------|
| `addresses` | 地址规则 | 静态 IP 映射 |
| `cname` | CNAME 规则 | 重定向目标域名 |
| `negative` | 否定规则 | 返回 NXDOMAIN |
| `blocked` | 否定规则 | 标记被拦截 |
| `no_cache` | 特殊规则 | 跳过缓存 |

## domain_trie — 反转域名基数树

### 设计原理

**反转存储**: 域名标签按 `"com → example → www"` 顺序存储。

优势：
- 后缀匹配等价于前缀遍历
- `*.example.com` 匹配所有子域名
- 遍历效率高于正向存储

### node 结构

```cpp
struct node {
    memory::unordered_map<memory::string, std::unique_ptr<node>> children; // 子标签映射
    std::any value;                                                        // 规则关联值
    bool is_end{false};                                                    // 是否为规则终点
    bool wildcard{false};                                                  // 是否为通配符标记
};
```

### 通配符处理

`*.example.com` 的处理：
1. 去掉 `"*"` 前缀
2. 在 `"example"` 节点标记 `wildcard=true`
3. 搜索时回溯检查 `wildcard` 标志

**通配符规则**: `*.example.com` 匹配 `www.example.com` 但不匹配 `example.com`（至少匹配一级子域）。

### 核心方法

#### insert

```cpp
void insert(std::string_view domain, const std::any &value);
```

流程：
1. 去末尾点号
2. 检测 `"*."` 前缀 → 设置 `wildcard=true`
3. 按 `.` 分割标签
4. 反转标签顺序
5. 沿树逐级创建节点
6. 终点节点设置 `is_end=true` 和 `value`

#### search

```cpp
auto search(std::string_view domain) const -> std::optional<std::any>;
```

流程：
1. 分割并反转标签
2. 沿树精确匹配
3. 若路径不完整 → 回溯检查 `wildcard`
4. 返回命中节点的 `value` 或 `nullopt`

**匹配优先级**: 精确匹配 > 通配符匹配

#### match

```cpp
auto match(std::string_view domain) const -> bool;
```

简化版 `search()`，只返回是否命中。

### 标签处理辅助函数

```cpp
static auto split_labels(std::string_view domain) -> memory::vector<memory::string>;
static auto to_lower(std::string_view s) -> memory::string;
```

- `split_labels`: `"www.example.com"` → `["com", "example", "www"]`
- `to_lower`: 域名统一转小写

## rules_engine 类

### 构造函数

```cpp
explicit rules_engine(memory::resource_pointer mr = memory::current_resource())
    : address_trie_(mr), cname_trie_(mr), mr_(mr) {}
```

维护两棵独立的基数树：
- `address_trie_` — 地址规则 + 否定规则
- `cname_trie_` — CNAME 重定向规则

### 规则添加方法

#### add_address_rule

```cpp
void add_address_rule(std::string_view domain,
                      const memory::vector<net::ip::address> &ips);
```

将地址映射插入 `address_trie_`。

#### add_negative_rule

```cpp
void add_negative_rule(std::string_view domain);
```

将否定规则插入 `address_trie_`，值设为特殊标记。

#### add_cname_rule

```cpp
void add_cname_rule(std::string_view domain, std::string_view target);
```

将 CNAME 重定向插入 `cname_trie_`。

### match — 统一匹配接口

```cpp
auto match(std::string_view domain) const -> std::optional<rule_result>;
```

流程：
1. 在 `address_trie_` 查找
2. 在 `cname_trie_` 查找
3. 合并两者结果

**优先级**: 地址规则 > CNAME 规则

## 匹配示例

### 精确匹配

```
规则: "example.com" → 1.2.3.4
查询: "example.com" → 命中，返回 1.2.3.4
查询: "www.example.com" → 未命中
```

### 通配符匹配

```
规则: "*.example.com" → 5.6.7.8
查询: "example.com" → 未命中（通配符需至少一级子域）
查询: "www.example.com" → 命中，返回 5.6.7.8
查询: "api.example.com" → 命中，返回 5.6.7.8
```

### 否定规则（广告屏蔽）

```
规则: "*.ad.com" → negative
查询: "track.ad.com" → 命中，返回 {negative=true, blocked=true}
```

### CNAME 重定向

```
规则: "alias.example.com" → CNAME "real.example.com"
查询: "alias.example.com" → 命中，返回 {cname="real.example.com"}
```

## 搜索算法详解

```
search("www.example.com")
  │
  ├─→ split_labels → ["com", "example", "www"]
  │
  ├─→ 沿树查找:
  │     root → "com" → "example" → "www" ?
  │
  ├─→ 精确路径完整 → 检查 is_end
  │     ├─→ is_end=true → 返回 value
  │     └─→ is_end=false → 回溯
  │
  └─→ 路径不完整 → 回溯检查 wildcard
        ├─→ "example" wildcard=true → 返回 value
        └─→ 无 wildcard → 返回 nullopt
```

## 调用链

```
resolver::resolve(host)
  │
  ├─→ rules_engine::match(host)
  │     ├─→ address_trie::search(host)
  │     │     ├─→ 精确匹配 → 返回 addresses 或 negative
  │     │     └─→ 通配符匹配 → 返回 wildcard 节点值
  │     │
  │     └─→ cname_trie::search(host)
  │           → 返回 cname 目标
  │
  ├─→ 合并结果 → rule_result
  │
  └→ 地址规则命中 → 直接返回 IPs
      CNAME 规则命中 → 重定向解析
      否定规则命中 → 返回 NXDOMAIN
```

## 参见

- [[core/resolve/dns/config|address_rule]] — 地址规则配置
- [[core/resolve/dns/config|cname_rule]] — CNAME 规则配置
- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口