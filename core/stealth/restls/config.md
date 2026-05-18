---
layer: core
source: I:/code/Prism/include/prism/stealth/restls/config.hpp
title: Restls 伪装方案配置
---

# Restls 伪装方案配置

> 源码位置: `I:/code/Prism/include/prism/stealth/restls/config.hpp`

## 概述

Restls 是一种 TLS 伪装协议，通过模拟真实 TLS 流量来隐藏代理特征。服务端通过 TLS 应用数据中的认证信息验证客户端身份。

协议参考: https://github.com/3andne/restls

## 命名空间

```cpp
namespace psm::stealth::restls
```

## 结构定义

### config

Restls 服务端配置。

```cpp
struct config
{
    memory::vector<memory::string> server_names;     ///< SNI 白名单
    memory::string host;                             ///< TLS 后端目标 host:port
    memory::string password;                         ///< 认证密码
    memory::string version_hint;                     ///< 版本提示: "tls12" 或 "tls13"
    memory::string restls_script;                    ///< 流量控制脚本
    std::uint32_t handshake_timeout_ms{5000};        ///< 握手超时（毫秒）
};
```

## 配置项说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `server_names` | vector<string> | SNI 白名单，只有匹配的 ClientHello 才会执行认证 |
| `host` | string | TLS 后端目标服务器（必须是 TLS 1.2 或 TLS 1.3 服务器） |
| `password` | string | 认证密码 |
| `version_hint` | string | 版本提示，"tls12" 或 "tls13" |
| `restls_script` | string | 流量控制脚本，用于隐藏代理特征 |
| `handshake_timeout_ms` | uint32_t | 握手超时时间，默认 5000ms |

## Restls Script 语法

流量控制脚本用于精确控制数据发送时机和大小，以模拟真实 TLS 流量模式：

| 语法 | 说明 | 示例 |
|------|------|------|
| `N?T` | 发送 N 字节，等待 T ms | `300?100` |
| `N~T` | 等待 T ms 后发送 N 字节 | `400~100` |
| `<N` | 等待客户端 N 个数据包 | `<1` |

**设计目的**:
- 打破 TLS 流量特征指纹
- 模拟正常 HTTPS 应用的流量模式
- 防止基于流量时序分析的检测

## 配置方法

### enabled

检查配置是否有效。

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool
{
    return !server_names.empty() && !host.empty() && !password.empty();
}
```

**要求**: `server_names` + `host` + `password` 均非空

## JSON 序列化

使用 Glaze 库进行 JSON 序列化：

```json
{
    "server_names": ["restls.example.com"],
    "host": "www.example.com:443",
    "password": "your_password",
    "version_hint": "tls13",
    "restls_script": "300?100<1",
    "handshake_timeout_ms": 5000
}
```

## 版本提示说明

| 值 | 后端 TLS 版本 | 说明 |
|----|---------------|------|
| `tls12` | TLS 1.2 | 兼容旧服务器 |
| `tls13` | TLS 1.3 | 推荐，更好的安全性 |

## 与 ShadowTLS 的区别

| 特性 | ShadowTLS | Restls |
|------|-----------|--------|
| 认证位置 | SessionID | 应用数据 |
| 流量控制 | 无 | Script 控制 |
| 后端要求 | TLS 1.3 | TLS 1.2/1.3 |
| 多用户 | 支持 (v3) | 单密码 |

## 调用链

```
config::enabled() <- restls::scheme::active()
config::server_names <- restls::scheme::snis()
config <- restls::scheme::handshake()
```

## 依赖

- [[core/stealth/restls/scheme|Restls 方案]] - 使用此配置