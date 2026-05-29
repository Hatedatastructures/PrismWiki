---
title: 全局配置聚合
layer: core
source:
  - include/prism/config.hpp
  - include/prism/instance/config.hpp
  - include/prism/instance/serialize.hpp
  - include/prism/stealth/serialize.hpp
  - include/prism/multiplex/serialize.hpp
  - include/prism/resolve/dns/serialize.hpp
tags: [config, loader, glaze, serialization]
---

# 全局配置聚合

> 源码: `include/prism/config.hpp` | 序列化: 各模块 `serialize.hpp`

`psm::config` 是全局配置聚合结构体，将各子系统的独立配置组合为统一顶层配置，通过 glaze JSON 库实现序列化/反序列化。

## 配置层级

```
psm::config
├── instance::config          // 代理服务核心
│   ├── limits                // 连接限制
│   ├── positive              // 正向代理端点
│   ├── addressable           // 监听端点
│   ├── cert                  // TLS 证书
│   ├── auth                  // 身份认证（用户列表）
│   ├── camouflage            // 伪装路径
│   └── reverse_map           // 反向代理路由表
├── connect::config (pool)    // 连接池
├── instance::buffer          // 缓冲区配置
├── protocol::config          // 协议配置
│   ├── socks5
│   ├── trojan
│   ├── vless
│   └── shadowsocks
├── multiplex::config         // 多路复用
├── stealth::config           // 伪装配置
│   ├── reality
│   ├── shadowtls
│   ├── restls
│   ├── anytls
│   ├── trusttunnel
│   └── native_tls
├── dns::config               // DNS 解析器
└── trace::config             // 日志追踪
```

## JSON 字段映射

glaze 元数据定义了 JSON 键名到结构体字段的映射：

| JSON 键 | C++ 字段 | 说明 |
|---------|---------|------|
| `"agent"` | `instance` | 代理服务核心配置（历史命名） |
| `"pool"` | `pool` | 连接池配置 |
| `"buffer"` | `buffer` | 缓冲区配置 |
| `"protocol"` | `protocol` | 协议配置 |
| `"multiplex"` | `mux` | 多路复用配置 |
| `"stealth"` | `stealth` | 伪装配置 |
| `"dns"` | `dns` | DNS 配置 |
| `"trace"` | `trace` | 日志配置 |

## 设计决策

### 为什么顶层配置用 `"agent"` 而非 `"instance"`？

**问题**: JSON 配置文件中使用 `"agent"` 作为代理服务配置的键名，但源码中对应的结构体是 `instance::config`。

**选择**: 保留 `"agent"` 作为 JSON 键名，因为这是用户可见的配置文件格式，变更会破坏所有现有配置文件的兼容性。源码中使用 `instance` 是因为内部重构已将 agent 统一为 instance。

**后果**: `glz::meta<psm::config>` 中 `"agent"` 映射到 `&T::instance`，用户配置文件和源码命名不一致但不影响功能。

### 为什么各模块的序列化分散在各自的 `serialize.hpp` 中？

**问题**: glaze 的 `glz::meta<T>` 特化必须在全局命名空间中定义，且必须在包含 glaze 头文件之后。

**选择**: 每个模块在自己的 `serialize.hpp` 中定义 glaze 映射，`config.hpp` 在最后统一包含所有 `serialize.hpp`。这避免了单一巨型序列化文件，各模块可以独立维护自己的映射。

**后果**: 包含 `config.hpp` 会传递引入所有子模块的头文件。如果只需要部分配置，应直接包含对应模块的 `config.hpp`。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| [[core/loader/load\|loader]] → `config` | 调用 | loader::load() 解析 JSON 文件填充 `psm::config` |
| `config` → 各模块 | 依赖 | 各模块通过 `cfg.stealth.reality`、`cfg.protocol.socks5` 等访问配置 |
| `config` → glaze | 依赖 | 序列化/反序列化依赖 glaze 库 |

## 变更敏感度

| 变更 | 影响 |
|------|------|
| 新增顶层配置字段 | 需在 `psm::config` 中添加字段 + 更新 `glz::meta<psm::config>` |
| 修改 JSON 键名 | 破坏所有现有配置文件兼容性 |
| 修改子模块配置结构 | 需同步更新对应的 `serialize.hpp` |
| glaze 库升级 | 可能影响 `glz::meta` 特化语法 |

## 相关文档

- [[core/loader/load|配置加载]] — loader::load() 解析流程
- [[core/instance/config|Instance 配置]] — 代理服务核心配置详解
- [[core/stealth/overview|Stealth 配置]] — 各伪装方案配置
