---
title: 常见问题汇总
source:
  - I:/code/Prism/include/prism/fault/
  - I:/code/Prism/tests/
module: debugging
type: reference
tags: [faq, troubleshooting, common-issues, errors, solutions]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[dev/debugging/connection]]"
  - "[[dev/debugging/protocol]]"
  - "[[dev/debugging/memory]]"
  - "[[dev/debugging/performance]]"
  - "[[dev/debugging/tls]]"
  - "[[fault]]"
---

# 常见问题汇总

本文档汇总 Prism 运行中的常见问题、错误码、症状和解决方案。

## 概述

### 问题分类索引

| 问题类别 | 参考文档 | 常见问题数 |
|----------|----------|------------|
| 启动问题 | 本文档 | 8 |
| 连接问题 | [[dev/debugging/connection]] | 12 |
| 协议问题 | [[dev/debugging/protocol]] | 10 |
| TLS 问题 | [[dev/debugging/tls]] | 8 |
| 内存问题 | [[dev/debugging/memory]] | 6 |
| 性能问题 | [[dev/debugging/performance]] | 8 |
| 配置问题 | 本文档 | 10 |

### 错误码快速索引

| 错误码 | 含义 | 常见原因 |
|--------|------|----------|
| `network_error` | 网络错误 | 连接失败、网络中断 |
| `timeout` | 超时 | 操作超时、响应延迟 |
| `connection_refused` | 连接拒绝 | 端口未监听、服务未启动 |
| `dns_failure` | DNS 解析失败 | DNS 服务器错误、域名不存在 |
| `ssl_error` | SSL/TLS 错误 | 证书问题、握手失败 |
| `protocol_error` | 协议错误 | 格式错误、版本不兼容 |
| `authentication_failed` | 认证失败 | 凭据错误、用户不存在 |
| `out_of_memory` | 内存不足 | 内存泄漏、池溢出 |
| `resource_exhausted` | 资源耗尽 | 连接数超限、池满 |
| `internal_error` | 内部错误 | 程序逻辑错误 |

---

## 启动问题

### Q1: 配置文件找不到

**症状**:
```
Error: Configuration file not found
```

**原因**: 配置文件路径不正确

**解决方案**:
```bash
# 确认配置文件存在
ls configuration.json

# 使用命令行指定路径
prism --config /path/to/config.json

# 或将配置文件放在可执行文件同目录
```

---

### Q2: 配置文件格式错误

**症状**:
```
Error: Invalid JSON format in configuration
```

**原因**: JSON 格式不正确

**解决方案**:
```bash
# 验证 JSON 格式
python -m json.tool configuration.json

# 或使用 jq
jq . configuration.json
```

常见 JSON 错误：
- 缺少逗号或多余逗号
- 字符串未用双引号
- 字段名拼写错误

---

### Q3: 端口已被占用

**症状**:
```
Error: Address already in use: port=443
```

**原因**: 端口被其他程序占用

**解决方案**:
```bash
# 查看端口占用
netstat -tlnp | grep 443
ss -tlnp | grep 443

# 释放端口
kill $(lsof -t -i:443)

# 或更改端口配置
{
    "agent": {
        "addressable": {"port": 8443}
    }
}
```

---

### Q4: 证书文件找不到

**症状**:
```
Error: Certificate file not found: key.pem
```

**原因**: 证书路径不正确

**解决方案**:
```bash
# 确认证书文件存在
ls /etc/ssl/certs/server.crt
ls /etc/ssl/private/server.key

# 检查路径配置
{
    "agent": {
        "certificate": {
            "key": "/correct/path/to/key.pem",
            "cert": "/correct/path/to/cert.pem"
        }
    }
}
```

---

### Q5: 权限不足

**症状**:
```
Error: Permission denied: cannot bind to port 443
```

**原因**: 非 root 用户无法绑定低端口（<1024）

**解决方案**:
```bash
# 使用 root 运行
sudo prism

# 或使用高端口
{
    "agent": {
        "addressable": {"port": 8081}
    }
}

# 或设置 capability
sudo setcap 'cap_net_bind_service=+ep' prism
```

---

### Q6: 依赖库缺失

**症状**:
```
Error: Cannot load library: libssl.so
```

**原因**: 运行时依赖库缺失

**解决方案**:
```bash
# Linux: 安装依赖
apt-get install libssl-dev

# Windows: 确认 DLL 在 PATH 中

# 或使用静态链接版本
```

---

### Q7: 内存分配失败

**症状**:
```
Error: Memory allocation failed at startup
```

**原因**: 系统内存不足

**解决方案**:
```bash
# 检查系统内存
free -h

# 减小池配置
{
    "pool": {
        "max_cache_per_endpoint": 32
    }
}

# 或增加系统内存
```

---

### Q8: DNS 解析失败

**症状**:
```
Error: DNS resolution failed for upstream
```

**原因**: DNS 服务器配置问题

**解决方案**:
```json
{
    "dns": {
        "servers": [
            {"address": "8.8.8.8", "protocol": "udp"}
        ]
    }
}
```

---

## 连接问题

### Q9: 所有连接超时

**症状**: 任何连接都超时

**原因**: 网络不通或防火墙阻断

**解决方案**:
```bash
# 检查网络
ping target_host

# 检查防火墙
iptables -L -n

# 增加超时
{
    "pool": {"connect_timeout_ms": 5000}
}
```

**参考**: [[dev/debugging/connection]]

---

### Q10: 连接随机断开

**症状**: 连接不定时断开

**原因**: 网络不稳定或心跳缺失

**解决方案**:
```json
{
    "pool": {
        "keep_alive": true
    }
}
```

**参考**: [[dev/debugging/connection]]

---

### Q11: 连接池满

**症状**: 无法获取新连接

**原因**: 连接池容量不足

**解决方案**:
```json
{
    "pool": {
        "max_cache_per_endpoint": 128
    }
}
```

**参考**: [[dev/debugging/connection]]

---

### Q12: 大量 TIME_WAIT

**症状**: 大量连接处于 TIME_WAIT

**原因**: 连接关闭频繁

**解决方案**:
```bash
# Linux 系统优化
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
```

**参考**: [[dev/debugging/connection]]

---

## 协议问题

### Q13: 协议识别失败

**症状**: 协议显示为 unknown

**原因**: 客户端协议不匹配

**解决方案**:
- 确认客户端使用正确协议
- 检查 TLS 是否正确建立
- 查看日志中的首字节特征

**参考**: [[dev/debugging/protocol]]

---

### Q14: Trojan 认证失败

**症状**: Trojan 连接认证失败

**原因**: 密码哈希不匹配

**解决方案**:
```bash
# 生成正确的 SHA224 哈希
echo -n "password" | sha224sum | cut -d' ' -f1

# 确保客户端和服务器密码一致
```

**参考**: [[dev/debugging/protocol]]

---

### Q15: VLESS UUID 格式错误

**症状**: UUID 验证失败

**原因**: UUID 格式不正确

**解决方案**:
```bash
# 使用标准 UUID
uuidgen

# 格式: 8-4-4-4-12
# 例如: 550e8400-e29b-41d4-a716-446655440000
```

**参考**: [[dev/debugging/protocol]]

---

### Q16: SOCKS5 认证失败

**症状**: SOCKS5 认证不通过

**原因**: 认证方法不匹配

**解决方案**:
```json
{
    "protocol": {
        "socks5": {
            "enable_auth": true
        }
    }
}
```

**参考**: [[dev/debugging/protocol]]

---

## TLS 问题

### Q17: TLS 握手失败

**症状**: SSL/TLS 连接建立失败

**原因**: 版本或密码套件不兼容

**解决方案**:
```bash
# 测试 TLS 连接
openssl s_client -connect server:443 -tls1_3

# 检查密码套件支持
nmap --script ssl-enum-ciphers -p 443 server
```

**参考**: [[dev/debugging/tls]]

---

### Q18: 证书验证失败

**症状**: 证书无效错误

**原因**: 证书过期或域名不匹配

**解决方案**:
```bash
# 检查证书有效期
openssl x509 -in cert.pem -enddate -noout

# 检查域名匹配
openssl x509 -in cert.pem -text -noout | grep DNS
```

**参考**: [[dev/debugging/tls]]

---

### Q19: Reality 认证失败

**症状**: Reality 连接失败

**原因**: 密钥或 Short ID 不匹配

**解决方案**:
- 确认公钥私钥匹配
- 确认 Short ID 两端一致
- 检查伪装目标可达

**参考**: [[dev/debugging/tls]]

---

### Q20: ShadowTLS 失败

**症状**: ShadowTLS 握手失败

**原因**: 版本或密码不匹配

**解决方案**:
```json
{
    "stealth": {
        "shadowtls": {
            "version": 3,
            "password": "same_on_both_ends"
        }
    }
}
```

**参考**: [[dev/debugging/tls]]

---

## 内存问题

### Q21: 内存持续增长

**症状**: RSS 内存持续上升

**原因**: 内存泄漏

**解决方案**:
```bash
# 使用 Valgrind 检测
valgrind --leak-check=full prism

# 检查日志中的内存增长
grep "Memory usage" prism.log
```

**参考**: [[dev/debugging/memory]]

---

### Q22: 内存分配失败

**症状**: 分配失败但内存足够

**原因**: 内存碎片

**解决方案**:
- 使用 PMR Arena 批量分配
- 减少大小差异大的分配
- 定期 reset Arena

**参考**: [[dev/debugging/memory]]

---

### Q23: 僵尸会话

**症状**: 会话计数不减少

**原因**: 会话未正确释放

**解决方案**:
- 检查会话关闭路径
- 确认异常处理释放资源
- 检查协程是否正常结束

**参考**: [[dev/debugging/memory]]

---

## 性能问题

### Q24: 响应延迟高

**症状**: 请求响应慢

**原因**: 网络延迟或处理瓶颈

**解决方案**:
```bash
# 检查各阶段延迟
ping target
traceroute target

# 优化配置
{
    "buffer": {"size": 524288},
    "pool": {"max_cache_per_endpoint": 128}
}
```

**参考**: [[dev/debugging/performance]]

---

### Q25: 吞吐达不到预期

**症状**: 性能测试不达标

**原因**: 资源瓶颈

**解决方案**:
```bash
# 定位瓶颈
top -p $(pidof prism)
perf top -p $(pidof prism)
```

**参考**: [[dev/debugging/performance]]

---

### Q26: CPU 使用率高

**症状**: CPU 持续高负载

**原因**: 加密或日志开销

**解决方案**:
```json
{
    "trace": {
        "log_level": "info"  // 避免 trace/debug
    }
}
```

**参考**: [[dev/debugging/performance]]

---

## 配置问题

### Q27: 用户认证不生效

**症状**: 配置了用户但无法认证

**原因**: 配置格式错误

**解决方案**:
```json
{
    "agent": {
        "authentication": {
            "users": [
                {
                    "password": "correct_password",
                    "uuid": "valid-uuid-format"
                }
            ]
        }
    }
}
```

---

### Q28: 多路复用不工作

**症状**: 多路复用功能不启用

**原因**: 未启用多路复用

**解决方案**:
```json
{
    "multiplex": {
        "enabled": true
    }
}
```

---

### Q29: DNS 解析异常

**症状**: DNS 解析结果不正确

**原因**: DNS 配置问题

**解决方案**:
```json
{
    "dns": {
        "servers": [
            {"address": "dns.google", "protocol": "https"}
        ],
        "mode": "fastest",
        "cache_enabled": true
    }
}
```

---

### Q30: UDP 不工作

**症状**: UDP 代理失败

**原因**: UDP 未启用

**解决方案**:
```json
{
    "protocol": {
        "socks5": {"enable_udp": true},
        "trojan": {"enable_udp": true}
    }
}
```

---

## 快速诊断表

### 症状 → 错误码 → 解决方案

| 症状 | 错误码 | 解决方案 |
|------|--------|----------|
| 无法启动 | internal_error | 检查配置文件 |
| 连接失败 | network_error | 检查网络连通性 |
| 连接超时 | timeout | 增加超时时间 |
| 认证失败 | authentication_failed | 检查凭据配置 |
| TLS 错误 | ssl_error | 检查证书 |
| 内存不足 | out_of_memory | 检查内存泄漏 |
| 连接拒绝 | connection_refused | 检查服务状态 |
| DNS 失败 | dns_failure | 检查 DNS 配置 |

---

## 排障流程图

```
发现问题
    |
    v
+------------------------+
| 1. 查看日志             |
| grep error prism.log   |
+------------------------+
    |
    v
+------------------------+
| 2. 识别错误码           |
| 确定问题类别           |
+------------------------+
    |
    v
+------------------------+
| 3. 查找解决方案         |
| 参考本文档或详细文档    |
+------------------------+
    |
    v
+------------------------+
| 4. 应用修复             |
| 配置调整或代码修复      |
+------------------------+
    |
    v
+------------------------+
| 5. 验证修复             |
| 重启并观察日志         |
+------------------------+
```

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[dev/debugging/connection]] — 连接问题排查
- [[dev/debugging/protocol]] — 协议问题排查
- [[dev/debugging/memory]] — 内存问题排查
- [[dev/debugging/performance]] — 性能问题排查
- [[dev/debugging/tls]] — TLS 问题排查
- [[dev/debugging/log-analysis]] — 日志分析方法
- [[fault]] — Fault 错误码体系