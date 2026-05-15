---
title: "config — Restls 伪装方案配置"
source: "include/prism/stealth/restls/config.hpp"
module: "stealth"
submodule: "restls"
type: api
tags: [stealth, restls, config, 配置]
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/stealth/restls/config.hpp`
> 模块: [[stealth|stealth]] > [[stealth/restls|restls]]

## 概述

Restls 伪装方案配置。Restls 是一种 TLS 伪装协议，通过模拟真实 TLS 流量来隐藏代理特征。服务端通过 TLS 应用数据中的认证信息验证客户端身份。

**协议参考**: https://github.com/3andne/restls

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::string/vector PMR 容器 |
| 被依赖 | [[stealth/restls/scheme|scheme]] | 方案使用配置 |

## 命名空间

`psm::stealth::restls`

---

## 结构体: config

> 源码: `include/prism/stealth/restls/config.hpp:33`

### 概述

Restls 服务端配置。包含 TLS 后端目标、认证密码、版本提示和流量控制脚本。

### 成员变量

| 变量 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| server_names | memory::vector\<memory::string\> | SNI 白名单 | 空 |
| host | memory::string | TLS 后端目标 host:port | "" |
| password | memory::string | 认证密码 | "" |
| version_hint | memory::string | 版本提示: "tls12" 或 "tls13" | "" |
| restls_script | memory::string | 流量控制脚本 | "" |
| handshake_timeout_ms | std::uint32_t | 握手超时（毫秒） | 5000 |

### 设计意图

- **server_names**: SNI 白名单，只有匹配的 ClientHello 才会执行认证
- **host**: TLS 后端目标服务器（必须是 TLS 1.2 或 TLS 1.3 服务器）
- **password**: 认证密码
- **version_hint**: 版本提示，"tls12" 或 "tls13"
- **restls_script**: 流量控制脚本，用于隐藏代理特征

### Restls Script 语法

- `300?100`: 发送 300 字节，等待 100ms
- `400~100`: 等待 100ms 后发送 400 字节
- `<1`: 等待客户端数据

### 生命周期

1. **构造**: 由配置加载器从 `configuration.json` 解析
2. **使用**: 被 Restls 方案和握手函数读取
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 配置在启动时加载，运行时只读，线程安全

### 异常安全

- 不抛异常，配置解析失败时返回默认值

---

## 函数: enabled()

> 源码: `include/prism/stealth/restls/config.hpp:46`

### 功能

检查配置是否有效。如果 `server_names`、`host` 和 `password` 都非空，返回 `true`。

### 签名

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool;
```

### 返回值

`bool` — 配置完整返回 `true`，否则返回 `false`

### 调用链

**被调用**（向上）:
- [[stealth/restls/scheme|active()]] — 判断方案是否启用

---

## JSON 序列化

配置文件使用 glaze 库进行 JSON 序列化/反序列化。

### config 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| server_names | server_names | vector\<string\> |
| host | host | string |
| password | password | string |
| version_hint | version_hint | string |
| restls_script | restls_script | string |
| handshake_timeout_ms | handshake_timeout_ms | uint32_t |

### 配置示例

```json
{
  "stealth": {
    "restls": {
      "server_names": ["example.com", "www.example.com"],
      "host": "backend.example.com:443",
      "password": "your_password",
      "version_hint": "tls13",
      "restls_script": "300?100 400~100 <1",
      "handshake_timeout_ms": 5000
    }
  }
}
```

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — Restls 基于 TLS 1.3 协议伪装

## 相关链接

- [[stealth/restls/scheme|scheme]] — Restls 方案，使用此配置
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
