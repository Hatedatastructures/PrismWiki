---
title: 协议问题排查指南
source:
  - I:/code/Prism/include/prism/protocol/
  - I:/code/Prism/include/prism/pipeline/
  - I:/code/Prism/src/prism/protocol/
module: debugging
type: reference
tags: [protocol, parsing, encoding, authentication, troubleshooting]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[core/protocol/overview|protocol]]"
  - "[[core/pipeline/overview|pipeline]]"
  - "[[recognition]]"
---

# 协议问题排查指南

协议问题涉及协议识别、解析错误、认证失败等。本文档提供协议层面的排障方法。

## 概述

### 协议问题分类

| 问题类型 | 典型症状 | 影响范围 |
|----------|----------|----------|
| 识别失败 | 无法判断协议类型 | 入口连接 |
| 解析错误 | 请求格式不符合规范 | 数据处理 |
| 认证失败 | 凭据验证不通过 | 用户访问 |
| 编码错误 | 输出格式异常 | 响应数据 |
| 命令不支持 | 未实现的协议命令 | 功能限制 |

### 协议处理流水线

```
+--------------------------------------------------+
|                协议处理流水线                      |
+--------------------------------------------------+
                       |
         +-------------+-------------+
         |             |             |
    +---------+   +---------+   +---------+
    | Probe   |   | Identify|   | Execute |
    | 首字节  |   | 协议识别|   | 执行处理|
    +---------+   +---------+   +---------+
         |             |             |
         v             v             v
    +---------+   +---------+   +---------+
    | 24B peek|   | 特征匹配|   | 协议流水线|
    +---------+   +---------+   +---------+
```

问题可能发生在任何阶段：

- Probe：首字节特征提取失败
- Identify：协议特征匹配失败
- Execute：协议处理错误

---

## 详解

### 协议识别失败排查

#### 症状特征

```
日志示例:
[2026-05-17 10:30:45.123] [prism] [warn] Protocol not recognized: from=192.168.1.100
[2026-05-17 10:30:45.456] [prism] [error] Unknown protocol: confidence=low
```

#### 排查流程

```
识别失败
    |
    v
+------------------------+
| 1. 检查客户端协议       |
| - 使用正确的协议       |
| - 版本兼容性           |
+------------------------+
    |
    v
+------------------------+
| 2. 检查入口配置         |
| - protocol 配置        |
| - stealth 配置         |
+------------------------+
    |
    v
+------------------------+
| 3. 检查识别特征         |
| - feature_bitmap       |
| - 探测数据             |
+------------------------+
    |
    v
+------------------------+
| 4. 检查 TLS 层          |
| - SNI 匹配             |
| - 伪装路径             |
+------------------------+
```

#### 各协议特征

| 协议 | 入口特征 | 识别方法 |
|------|----------|----------|
| HTTP | `GET/POST/CONNECT` | 首字节文本匹配 |
| SOCKS5 | `0x05` | 版本字节匹配 |
| Trojan | SHA224 密码 + CRLF | TLS 内首行解析 |
| VLESS | `0x00` + UUID | TLS 内首字节 |
| SS2022 | 随机 salt + AEAD | 加密数据特征 |

### 协议解析错误排查

#### 症状特征

```
日志示例:
[2026-05-17 10:35:00.123] [prism] [error] Invalid SOCKS5 request: expected=0x05, got=0x04
[2026-05-17 10:35:00.456] [prism] [error] HTTP parse failed: invalid header format
[2026-05-17 10:35:00.789] [prism] [error] VLESS version mismatch: expected=0, got=1
```

#### 各协议解析要点

**HTTP 协议**

```
解析检查点:
1. 方法：GET/POST/CONNECT/HEAD
2. URL 格式：绝对 URL 或相对路径
3. Host 头：必需
4. HTTP 版本：HTTP/1.1

常见错误:
- 缺少 Host 头
- URL 格式错误
- HTTP 版本不支持
```

**SOCKS5 协议**

```
解析检查点:
1. 版本字节：必须是 0x05
2. 方法列表：至少一个认证方法
3. 认证流程：用户名/密码格式
4. 命令字节：CONNECT/BIND/UDP
5. 地址类型：IPv4/域名/IPv6

常见错误:
- 版本不是 0x05
- 地址类型无效
- 端口格式错误
```

**Trojan 协议**

```
解析检查点:
1. 密码行：56 字符 SHA224 + CRLF
2. 命令字节：CONNECT/UDP
3. 地址格式：ATYP + 地址 + 端口
4. CRLF 结尾

常见错误:
- 密码长度不正确
- 密码未 SHA224 编码
- CRLF 缺失
```

**VLESS 协议**

```
解析检查点:
1. 版本字节：0x00
2. UUID：16 字节
3. 增体字节：0x00 或 0x01
4. 增体长度：正确计算
5. 命令：CONNECT/UDP

常见错误:
- UUID 格式错误
- 增体解析失败
- 命令字节无效
```

### 认证失败排查

#### 症状特征

```
日志示例:
[2026-05-17 10:40:00.123] [prism] [warn] Authentication failed: user=unknown
[2026-05-17 10:40:00.456] [prism] [error] Invalid credential: type=trojan
[2026-05-17 10:40:00.789] [prism] [error] User not found: uuid=xxx-xxx-xxx
```

#### 排查流程

```
认证失败
    |
    v
+------------------------+
| 1. 检查用户配置         |
| - users 列表           |
| - password/uuid        |
+------------------------+
    |
    v
+------------------------+
| 2. 检查凭据格式         |
| - Trojan: SHA224       |
| - VLESS: UUID          |
| - SOCKS5: 原始密码     |
+------------------------+
    |
    v
+------------------------+
| 3. 检查认证流程         |
| - 协议匹配             |
| - 认证方法             |
+------------------------+
```

#### 各协议认证方式

| 协议 | 认证方式 | 凭据格式 |
|------|----------|----------|
| HTTP | Basic Auth | base64(user:pass) |
| SOCKS5 | Method 0x00/0x02 | 用户名 + 密码 |
| Trojan | SHA224 哈希 | 56 字符 hex |
| VLESS | UUID | 36 字符 (8-4-4-4-12) |
| SS2022 | PSK | Base64 16B/32B |

#### 凭据生成示例

```bash
# Trojan 密码 SHA224
echo -n "your_password" | sha224sum | cut -d' ' -f1

# VLESS UUID
uuidgen

# SS2022 PSK
openssl rand -base64 16  # AES-128
openssl rand -base64 32  # AES-256
```

---

## 日志分析示例

### 正常协议识别

```
[10:30:00.100] [debug] Probe: first_bytes=47455420 ("GET ")
[10:30:00.150] [info] Protocol identified: type=http, confidence=high
[10:30:00.200] [info] Pipeline started: protocol=http
```

### 协议识别失败

```
[10:30:00.100] [debug] Probe: first_bytes=unknown
[10:30:00.150] [warn] Protocol not recognized: confidence=low
[10:30:00.200] [error] Falling back to default handler
```

### 认证成功

```
[10:30:00.100] [debug] Credential received: type=trojan
[10:30:00.150] [info] Authentication success: user=user1
[10:30:00.200] [info] Session authenticated: protocol=trojan
```

### 认证失败

```
[10:30:00.100] [debug] Credential received: hash=abc123...
[10:30:00.150] [warn] User not found in directory
[10:30:00.200] [error] Authentication failed: code=auth_failed
```

---

## 使用示例

### 协议测试脚本

```bash
#!/bin/bash
# test_protocol.sh

# HTTP 代理测试
curl -x http://user:pass@127.0.0.1:8081 http://example.com

# SOCKS5 代理测试
curl -x socks5://user:pass@127.0.0.1:8081 http://example.com

# Trojan 协议测试（需要客户端）
# mihomo 配置后测试

# VLESS 协议测试（需要客户端）
# mihomo 配置后测试
```

### 凭据验证脚本

```bash
#!/bin/bash
# verify_credential.sh

TYPE=$1
VALUE=$2

case $TYPE in
    "trojan")
        # SHA224 应该是 56 字符
        if [ ${#VALUE} -eq 56 ]; then
            echo "Valid Trojan password hash length"
        else
            echo "Invalid: expected 56 chars, got ${#VALUE}"
        fi
        ;;
    "vless")
        # UUID 格式检查
        if [[ $VALUE =~ ^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$ ]]; then
            echo "Valid VLESS UUID format"
        else
            echo "Invalid UUID format"
        fi
        ;;
    *)
        echo "Unknown credential type"
        ;;
esac
```

---

## 最佳实践

### 协议配置建议

| 场景 | 协议选择 | 建议 |
|------|----------|------|
| 简单代理 | HTTP/SOCKS5 | 易于调试 |
| TLS 伪装 | Trojan/VLESS | 抗审查 |
| 高性能 | SS2022 | AEAD 加密 |

### 认证配置建议

| 参数 | 建议 | 原因 |
|------|------|------|
| password | 强密码 | 防止破解 |
| max_connections | 设置限制 | 防止滥用 |
| 用户数量 | 最小化 | 减少管理负担 |

### 协议识别配置建议

```json
{
    "protocol": {
        "socks5": {
            "enable_tcp": true,
            "enable_udp": true,
            "enable_auth": true
        },
        "trojan": {
            "enable_tcp": true,
            "enable_udp": false
        }
    }
}
```

---

## 常见问题

### Q1: 协议识别为 unknown 怎么办？

**A**: 
1. 确认客户端使用正确的协议
2. 检查 TLS 是否正确建立
3. 查看日志中的 first_bytes 内容

### Q2: Trojan 认证总是失败怎么办？

**A**: 
1. 确认密码已转换为 SHA224
2. 检查密码长度是否为 56 字符
3. 确认客户端和服务器密码一致

### Q3: VLESS UUID 格式错误怎么办？

**A**: 
1. 使用 `uuidgen` 生成标准 UUID
2. 检查格式为 `8-4-4-4-12`
3. 确认是小写字母和数字

### Q4: SOCKS5 认证失败怎么办？

**A**: 
1. 检查 `enable_auth` 是否开启
2. 确认用户名密码配置正确
3. 查看认证方法协商日志

---

## 排障指南

### 问题：HTTP CONNECT 请求失败

**症状**: CONNECT 请求返回错误

**排查步骤**:

1. 检查请求格式
   ```
   CONNECT example.com:443 HTTP/1.1
   Host: example.com
   ```

2. 检查 Host 头是否存在

3. 检查认证配置
   ```json
   "authentication": {
       "users": [{"password": "..."}]
   }
   ```

---

### 问题：SOCKS5 握手失败

**症状**: SOCKS5 连接无法建立

**排查步骤**:

1. 检查版本字节
   - 客户端必须发送 0x05

2. 检查方法协商
   ```
   Client: 05 02 00 02  // 版本5，2方法，无认证+用户名密码
   Server: 05 02        // 版本5，选择用户名密码
   ```

3. 检查认证流程
   ```
   Client: 01 04 user 04 pass  // 版本1，用户名4字节，密码4字节
   Server: 01 00              // 版本1，成功
   ```

---

### 问题：TLS 内协议无法识别

**症状**: TLS 连接建立但协议未知

**排查步骤**:

1. 确认 TLS 已正确握手

2. 检查协议配置
   ```json
   "protocol": {
       "trojan": {"enable_tcp": true}
   }
   ```

3. 检查伪装配置
   ```json
   "stealth": {
       "reality": {...}
   }
   ```

4. 查看内部数据日志（debug 级别）

---

## 深层故障分析

当基本排障无法定位问题时，参考以下深层分析文档：

- [[dev/debugging/deep-dive/protocol-boundaries|协议实现边界与认证深层分析]] — SS2022 timestamp_window、PSK 半初始化、Trojan 缓冲区边界、VLESS Vision 不支持、SOCKS5 IPv6 限制

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[protocol]] — Protocol 协议模块
- [[pipeline]] — Pipeline 管道模块
- [[recognition]] — Recognition 识别模块
- [[dev/debugging/tls]] — TLS 问题排查