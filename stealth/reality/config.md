---
title: "config — Reality TLS 伪装配置"
source: "include/prism/stealth/reality/config.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, config, 配置]
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/stealth/reality/config.hpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality TLS 伪装配置。定义 Reality TLS 伪装的服务端配置结构体，包含目标伪装网站、允许的 SNI 列表、X25519 静态私钥和短 ID 列表。Reality 配置与标准 TLS 证书配置互斥：启用 Reality 时不使用自身证书，而是使用目标网站的真实证书。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::string/vector PMR 容器 |
| 被依赖 | [[stealth/reality/scheme|scheme]] | 方案使用配置 |
| 被依赖 | [[stealth/reality/auth|auth]] | 认证使用配置 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手使用配置 |

## 命名空间

`psm::stealth::reality`

---

## 结构体: config

> 源码: `include/prism/stealth/reality/config.hpp:25`

### 概述

Reality TLS 伪装服务端配置。配置 Reality 协议所需的所有参数。`dest` 为目标伪装网站，用于回退时的透明代理和证书获取；`server_names` 为允许的 SNI 列表，只有匹配的 ClientHello 才会尝试认证；`private_key` 为服务端 X25519 静态私钥（base64 编码）；`short_ids` 为客户端 short ID 列表（hex 编码），空字符串表示接受任意。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| dest | memory::string | 目标伪装网站（host:port 格式），如 "www.microsoft.com:443" |
| server_names | memory::vector\<memory::string\> | 允许的 SNI 列表，如 ["www.microsoft.com", "www.apple.com"] |
| private_key | memory::string | X25519 静态私钥（base64 编码，32 字节原始数据） |
| short_ids | memory::vector\<memory::string\> | 短 ID 列表（hex 编码，最长 16 字节），空字符串 "" 表示接受任意 short ID |

### 设计意图

- **dest**: 目标伪装网站，用于回退时的透明代理和证书获取
- **server_names**: 允许的 SNI 列表，只有匹配的 ClientHello 才会尝试认证
- **private_key**: 服务端 X25519 静态私钥，用于密钥交换
- **short_ids**: 客户端 short ID 列表，用于认证

### 生命周期

1. **构造**: 由配置加载器从 `configuration.json` 解析
2. **使用**: 被 Reality 方案和握手函数读取
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 配置在启动时加载，运行时只读，线程安全

### 异常安全

- 不抛异常，配置解析失败时返回默认值

---

## 函数: enabled()

> 源码: `include/prism/stealth/reality/config.hpp:36`

### 功能

检查 Reality 是否已启用。需要 `dest`、`private_key`、`server_names` 都非空。

### 签名

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool;
```

### 返回值

`bool` — 配置完整返回 `true`，否则返回 `false`

### 调用链

**被调用**（向上）:
- [[stealth/reality/scheme|active()]] — 判断方案是否启用

### 注意事项

- `dest` 必须非空
- `private_key` 必须非空
- `server_names` 必须非空

---

## JSON 序列化

配置文件使用 glaze 库进行 JSON 序列化/反序列化。

### config 结构体字段映射

| JSON 字段 | C++ 字段 | 类型 |
|-----------|----------|------|
| dest | dest | string |
| server_names | server_names | vector\<string\> |
| private_key | private_key | string |
| short_ids | short_ids | vector\<string\> |

### 配置示例

```json
{
  "stealth": {
    "reality": {
      "dest": "www.microsoft.com:443",
      "server_names": ["www.microsoft.com", "www.apple.com"],
      "private_key": "base64_encoded_private_key",
      "short_ids": ["01020304", "05060708"]
    }
  }
}
```

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — Reality 基于 TLS 1.3 协议伪装
- [[ref/crypto/x25519|X25519]] — Reality 使用 X25519 密钥交换

## 相关链接

- [[stealth/reality/scheme|scheme]] — Reality 方案，使用此配置
- [[stealth/reality/auth|auth]] — Reality 认证，使用此配置
- [[stealth/reality/handshake|handshake]] — Reality 握手，使用此配置
