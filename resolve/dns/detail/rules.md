---
title: "rules.hpp — DNS 域名规则引擎"
source: "include/prism/resolve/dns/detail/rules.hpp"
module: "resolve"
type: api
tags: [resolve, dns, rules, 规则引擎, 基数树, trie, 通配符]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - resolve/dns/config
  - memory/container
---

# rules.hpp

> 源码: `include/prism/resolve/dns/detail/rules.hpp` + `src/prism/resolve/dns/detail/rules.cpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

提供基于反转域名基数树（trie）的高效域名规则匹配系统。支持精确匹配、通配符匹配和后缀匹配三种模式，用于实现静态地址解析、广告屏蔽（否定规则）和 CNAME 重定向等 DNS 规则功能。

设计原理：
- **反转存储**：域名标签按 `"com" -> "example" -> "www"` 顺序存储，使后缀匹配等价于前缀遍历，大幅提升匹配效率
- **通配符折叠**：`*.example.com` 在 `"example"` 节点标记 `wildcard`，搜索时回溯检查即可，无需枚举子域
- **通配符要求**：至少匹配一级子域，即 `*.example.com` 匹配 `www.example.com` 但不匹配 `example.com`
- **PMR 内存管理**：所有容器使用 `memory::` 命名空间类型，支持运行时内存资源切换

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver_impl 使用规则引擎 |

## 命名空间

`psm::resolve::dns::detail`

---

## 结构体: rule_result

### 概述

域名规则匹配结果，封装一次规则匹配的全部产出信息。根据命中的规则类型，各字段具有不同的语义：地址规则填充 `addresses`，否定规则设置 `negative` 和 `blocked` 标记，CNAME 规则填充 `cname` 字段。未命中的字段保持默认值。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::vector<net::ip::address>` | `addresses` | 静态地址列表 |
| `memory::string` | `cname` | CNAME 目标域名 |
| `bool` | `negative` | 否定规则（广告屏蔽） |
| `bool` | `no_cache` | 跳过缓存 |
| `bool` | `blocked` | 被拦截 |

---

### 函数: rule_result::rule_result()

- **功能说明**: 构造规则匹配结果，使用指定内存资源初始化所有 PMR 容器成员。
- **签名**:
  ```cpp
  explicit rule_result(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源，用于内部容器分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `rules_engine::match()` 内部构造结果
- **知识域**: [[memory/container|PMR 容器]]

---

## 类: domain_trie

### 概述

反转域名基数树。使用反转域名标签作为存储结构，将域名后缀匹配转化为树的前缀遍历问题。例如 `"www.example.com"` 被拆分为标签 `["com", "example", "www"]`，沿树逐级查找。

### 内部结构体: node

每个节点存储一组子标签映射和可选的规则值。

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::unordered_map<memory::string, std::unique_ptr<node>>` | `children` | 子标签映射 |
| `std::any` | `value` | 规则关联值 |
| `bool` | `is_end` | 是否为规则终点 |
| `bool` | `wildcard` | 是否为通配符标记 |

---

### 函数: domain_trie::domain_trie()

- **功能说明**: 构造反转域名基数树，创建根节点并初始化内存资源。
- **签名**:
  ```cpp
  explicit domain_trie(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源，用于内部节点和标签分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `rules_engine` 构造函数中初始化 `address_trie_` 和 `cname_trie_`
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

### 函数: domain_trie::insert()

- **功能说明**: 插入域名规则。对域名进行标签分割和反转后沿树逐级创建节点。若域名以 `"*."` 开头，去掉 `"*"` 后在倒数第二个标签节点标记 `wildcard=true`，表示该节点下的所有子域均匹配此规则。域名自动转为小写。
- **签名**:
  ```cpp
  void insert(std::string_view domain, const std::any &value);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名字符串，支持通配符前缀 `"*."` |
  | `value` | `const std::any &` | 规则关联的任意类型数据 |

- **返回值**: 无
- **调用（向下）**: `split_labels()` → 遍历创建节点 → 设置 `wildcard` / `is_end` / `value`
- **被调用（向上）**: `rules_engine::add_address_rule()`、`add_negative_rule()`、`add_cname_rule()` 调用
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

### 函数: domain_trie::search()

- **功能说明**: 精确 + 通配符匹配域名。首先尝试沿反转标签路径精确匹配到终点节点；若精确路径不完整，回溯检查沿途节点的 `wildcard` 标志。通配符要求查询域名至少比通配符域名多一级标签。
- **签名**:
  ```cpp
  [[nodiscard]] auto search(std::string_view domain) const -> std::optional<std::any>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 待匹配的域名 |

- **返回值**: `std::optional<std::any>` — 匹配成功时返回关联值，否则返回 `std::nullopt`
- **调用（向下）**: `split_labels()` → 沿树遍历 → 精确匹配检查 → wildcard 回溯
- **被调用（向上）**: `rules_engine::match()` 内部调用；`domain_trie::match()` 委托调用
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

### 函数: domain_trie::match()

- **功能说明**: 检查域名是否命中任何规则。委托给 `search()` 检查是否有匹配的规则。
- **签名**:
  ```cpp
  [[nodiscard]] auto match(std::string_view domain) const -> bool;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 待检查的域名 |

- **返回值**: `bool` — 命中返回 `true`，否则返回 `false`
- **调用（向下）**: `search(domain).has_value()`
- **被调用（向上）**: 外部快速检查场景使用
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

### 函数: domain_trie::split_labels() [private, static]

- **功能说明**: 将域名分割为标签并反转顺序。按 `"."` 分割域名，跳过前导和末尾点号，转为小写后反转标签顺序。例如 `"www.example.com"` 分割为 `["com", "example", "www"]`。
- **签名**:
  ```cpp
  [[nodiscard]] static auto split_labels(std::string_view domain) -> memory::vector<memory::string>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 原始域名，如 `"www.example.com"` |

- **返回值**: `memory::vector<memory::string>` — 反转后的标签列表
- **调用（向下）**: `to_lower()` → 按 `'.'` 分割 → `std::reverse()`
- **被调用（向上）**: `insert()` 和 `search()` 内部使用
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

### 函数: domain_trie::to_lower() [private, static]

- **功能说明**: 将字符串转换为小写。逐字符调用 `std::tolower` 转换，返回新的小写字符串。使用默认内存资源（静态函数无法访问实例 `mr_`）。
- **签名**:
  ```cpp
  [[nodiscard]] static auto to_lower(std::string_view s) -> memory::string;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `s` | `std::string_view` | 输入字符串视图 |

- **返回值**: `memory::string` — 小写字符串
- **调用（向下）**: 无（`std::transform` + `std::tolower`）
- **被调用（向上）**: `split_labels()` 内部对每个标签调用
- **知识域**: [[resolve/dns/detail/rules|基数树]]

---

## 类: rules_engine

### 概述

DNS 域名规则引擎，整合地址规则和 CNAME 规则两棵独立的 [[#类-domain_trie|基数树]]，提供统一的域名匹配接口。匹配时先查找地址规则，再查找 CNAME 规则，两者结果合并后返回。地址规则优先级高于 CNAME 规则。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `domain_trie` | `address_trie_` | 地址规则基数树 |
| `domain_trie` | `cname_trie_` | CNAME 规则基数树 |
| `memory::resource_pointer` | `mr_` | 内存资源 |

---

### 函数: rules_engine::rules_engine()

- **功能说明**: 构造规则引擎，使用指定内存资源初始化地址规则树和 CNAME 规则树。
- **签名**:
  ```cpp
  explicit rules_engine(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源，用于内部树结构和结果分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 构造 `address_trie_(mr)` 和 `cname_trie_(mr)`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中初始化
- **知识域**: [[resolve/dns/detail/rules|规则引擎]]

---

### 函数: rules_engine::add_address_rule()

- **功能说明**: 添加静态地址规则。将域名和对应的 IP 地址列表插入地址规则基数树。域名支持通配符前缀 `"*."`。
- **签名**:
  ```cpp
  void add_address_rule(std::string_view domain, const memory::vector<net::ip::address> &ips);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 匹配的域名，支持通配符 |
  | `ips` | `const memory::vector<net::ip::address> &` | 该域名映射的静态 IP 地址列表 |

- **返回值**: 无
- **调用（向下）**: `address_trie_.insert(domain, std::any(ips))`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中加载地址规则配置时调用
- **知识域**: [[resolve/dns/detail/rules|规则引擎]]

---

### 函数: rules_engine::add_negative_rule()

- **功能说明**: 添加否定规则（广告屏蔽）。匹配到否定规则时，`rule_result` 的 `negative` 和 `blocked` 标记将设置为 `true`。内部使用 `std::any(true)` 存储标记。
- **签名**:
  ```cpp
  void add_negative_rule(std::string_view domain);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 需要屏蔽的域名，支持通配符 |

- **返回值**: 无
- **调用（向下）**: `address_trie_.insert(domain, std::any(true))`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中加载否定规则配置时调用
- **知识域**: [[resolve/dns/detail/rules|规则引擎]]

---

### 函数: rules_engine::add_cname_rule()

- **功能说明**: 添加 CNAME 重定向规则。将源域名和目标域名插入 CNAME 规则基数树。
- **签名**:
  ```cpp
  void add_cname_rule(std::string_view domain, std::string_view target);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 源域名，支持通配符 |
  | `target` | `std::string_view` | CNAME 目标域名 |

- **返回值**: 无
- **调用（向下）**: `cname_trie_.insert(domain, std::any(target_str))`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中加载 CNAME 规则配置时调用
- **知识域**: [[resolve/dns/detail/rules|规则引擎]]

---

### 函数: rules_engine::match()

- **功能说明**: 匹配域名并返回合并的规则结果。依次在地址树和 CNAME 树中查找域名，合并两者的结果。地址规则优先级高于 CNAME 规则。否定规则使用 `std::any_cast<bool>` 检测；地址规则使用 `std::any_cast<memory::vector<net::ip::address>>` 提取；CNAME 规则使用 `std::any_cast<memory::string>` 提取。
- **签名**:
  ```cpp
  [[nodiscard]] auto match(std::string_view domain) const -> std::optional<rule_result>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 待匹配的域名 |

- **返回值**: `std::optional<rule_result>` — 命中规则时返回结果，未命中返回 `std::nullopt`
- **调用（向下）**: `address_trie_.search(domain)` + `cname_trie_.search(domain)` → 合并结果
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在规则匹配阶段调用
- **知识域**: [[resolve/dns/detail/rules|规则引擎]]

---

## 调用链总览

```
[[resolve/dns/dns|resolver_impl 构造函数]]
  ├── rules_engine::add_address_rule()   → address_trie_.insert()
  ├── rules_engine::add_negative_rule()  → address_trie_.insert()
  └── rules_engine::add_cname_rule()     → cname_trie_.insert()

[[resolve/dns/dns|resolver_impl::query_pipeline()]]
  └── rules_engine::match()
      ├── address_trie_.search() → split_labels() → 沿树遍历 + wildcard 回溯
      └── cname_trie_.search()   → split_labels() → 沿树遍历 + wildcard 回溯
```

---

## 知识域

- [[resolve/dns/detail/rules|DNS 规则]]
- [[resolve/dns/detail/rules|基数树]]
- [[resolve/dns/config|DNS 配置]]
