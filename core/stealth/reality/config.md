---
layer: core
source: include/prism/stealth/reality/config.hpp
title: Reality Config
tags:
  - stealth
  - reality
  - config
---

# Reality Config

Reality TLS 伪装服务端配置结构体。

## 源码位置

- 头文件：`include/prism/stealth/reality/config.hpp`

## 结构体定义

```cpp
struct config
{
    memory::string dest;                         // 目标伪装网站
    memory::vector<memory::string> server_names; // 允许的 SNI 列表
    memory::string private_key;                  // X25519 静态私钥
    memory::vector<memory::string> short_ids;    // 短 ID 列表
};
```

## 字段说明

### dest

目标伪装网站（host:port 格式）。

- 用于回退时的透明代理
- 用于获取真实证书
- 示例：`"www.microsoft.com:443"`

### server_names

允许的 SNI 列表。

- 只有匹配的 ClientHello 才会尝试认证
- 支持多个域名
- 示例：`["www.microsoft.com", "www.apple.com"]`

### private_key

X25519 静态私钥。

- base64 编码
- 32 字节原始数据
- 服务端密钥交换使用

### short_ids

短 ID 列表。

- hex 编码
- 最长 16 字节
- 空字符串 `""` 表示接受任意 short_id

## 方法

### enabled

```cpp
[[nodiscard]] auto enabled() const noexcept -> bool
```

检查配置是否完整启用。

启用条件：
- `dest` 非空
- `private_key` 非空
- `server_names` 非空

## 设计决策（WHY）

### 为什么 `short_ids` 不参与 `enabled()` 检查

`enabled()` 仅检查 `dest`、`private_key`、`server_names` 三个字段。`short_ids` 为空是合法配置——表示接受任意 short_id。如果将 `short_ids` 非空作为启用条件，就无法支持"不验证 short_id"的宽松模式。

### 为什么 `dest` 是必需的

`dest` 有两个用途：
1. **fallback 透明代理**：非 Reality 客户端的 ClientHello 被转发到 dest 服务器，完成真实的 TLS 握手
2. **证书获取**（非认证客户端时）：从 dest 服务器获取真实 TLS 证书用于伪装

没有 `dest`，Reality 无法回退也无法伪装，因此是必需字段。

### 为什么使用 `memory::string` 而非 `std::string`

Reality config 嵌入在 `psm::config` 层级中，`psm::config` 在启动时通过 JSON 解析构建，使用 PMR 分配器。`memory::string` = `std::pmr::string`，与全局池集成，减少堆分配碎片。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `private_key` 必须是 base64 编码的 32 字节 | X25519 密钥长度 | 解码后长度不等于 32 会导致密钥交换失败 |
| `short_ids` hex 编码最长 16 字节（32 hex chars） | 协议限制 | 超过 16 字节的 short_id 会被截断 |
| `dest` 格式必须是 `host:port` | 解析器要求 | IPv6 需用 `[ipv6]:port` 格式 |
| `server_names` 为空时 `enabled()` 返回 false | `enabled()` 逻辑 | 无 SNI 白名单 = 未配置 |
| 配置与标准 TLS 证书互斥 | Reality 协议设计 | 启用 Reality 时不使用自身证书 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| `private_key` 格式错误 | base64 解码失败或长度不等于 32 | 握手阶段密钥交换失败 |
| `dest` 不可达 | dest 服务器宕机 | fallback 失败，非 Reality 客户端断连 |
| `server_names` 包含不匹配的域名 | 配置的 SNI 与 dest 网站不同 | 客户端 SNI 不匹配，走其他方案或 fallback |
| `short_ids` 全为空字符串 | 配置 `[""]` | 接受任意 short_id，安全性降低 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `reality::scheme` → `config` | 调用 | `active()` 调用 `enabled()`，`snis()` 读取 `server_names` |
| `reality::auth` → `config` | 调用 | 认证时使用 `private_key`、`server_names`、`short_ids` |
| `reality::handshake` → `config` | 调用 | 握手中使用 `dest` 进行 fallback |
| `config` → `psm::config` | 嵌入 | `cfg.stealth.reality` 路径 |
| `config` → `memory` | 依赖 | 所有字符串和向量使用 PMR 分配器 |

## 与 TLS 证书的关系

Reality 配置与标准 TLS 证书配置**互斥**：

- 启用 Reality 时不使用自身证书
- 使用目标网站的真实证书进行伪装
- 通过 [[handshake#fetch_dest_certificate]] 获取证书

## 配置示例

```yaml
stealth:
  reality:
    dest: "www.microsoft.com:443"
    server_names:
      - "www.microsoft.com"
      - "www.apple.com"
    private_key: "base64-encoded-32-byte-key"
    short_ids:
      - "0123456789abcdef"
      - ""  # 接受任意
```

## 调用链

- [[scheme]] ← 方案类检查配置
- [[auth]] ← 认证时使用 private_key 和 short_ids
- [[handshake]] ← 回退时使用 dest