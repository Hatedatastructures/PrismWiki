---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/config.hpp
title: Reality Config
---

# Reality Config

Reality TLS 伪装服务端配置结构体。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/config.hpp`

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