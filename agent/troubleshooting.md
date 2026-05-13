---
title: Prism 故障排查
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [troubleshooting, proxy-server]
sources:
  - I:/code/Prism/docs/tutorial/troubleshooting.md
confidence: high
---

# Prism 故障排查

Prism 常见问题排查方法和解决方案。

## 常见问题分类

```
┌─────────────────────────────────────────────────────────────┐
│                Prism 常见问题分类                            │
│                                                             │
│  ① 连接问题                                                 │
│  ├── 程序启动失败                                           │
│  ├── 无法连接代理                                           │
│  ├── 连接超时                                               │
│  └── 连接被拒绝                                             │
│                                                             │
│  ② 协议问题                                                 │
│  ├── TLS 握手失败                                           │
│  ├── 证书验证错误                                           │
│  ├── 协议版本不匹配                                         │
│  └── 认证失败                                               │
│                                                             │
│  ③ 性能问题                                                 │
│  ├── 速度慢                                                 │
│  ├── 延迟高                                                 │
│  ├── 连接不稳定                                             │
│  └── 频繁断开                                               │
│                                                             │
│  ④ DNS 问题                                                 │
│  ├── DNS 解析失败                                           │
│  ├── DNS 泄漏                                               │
│  ├── 域名无法访问                                           │
│  └── fake-ip 异常                                           │
└─────────────────────────────────────────────────────────────┘
```

## 连接问题排查

### Q1：程序启动失败

**排查流程图**：

```
程序启动失败
    │
    ▼
检查运行环境 ──> 非 MinGW 环境 ──> 切换到 MinGW 编译环境
    │
    ├── 环境正常
    ▼
查看日志 ──> 配置文件错误 ──> 修正 JSON 配置
    │
    ├── 配置正常
    ▼
检查端口 ──> 端口被占用 ──> 修改端口号或停止占用进程
    │
    ├── 端口正常
    ▼
检查权限 ──> 权限不足 ──> 以管理员/root 身份运行
```

**排查步骤**：

1. **检查运行环境**：确认使用 MinGW 编译环境（程序静态链接，无外部依赖）
2. **查看日志**：检查配置中 `trace.path_name` 指定的日志目录
3. **端口占用**：修改配置中的端口号

**解决方案**：

```bash
# 检查端口占用
netstat -ano | findstr :8081

# Linux
sudo lsof -i :8081
```

### Q2：无法连接代理

**排查流程图**：

```
无法连接代理
    │
    ▼
确认服务运行 ──> 未运行 ──> 启动 Prism 服务
    │
    ├── 运行中
    ▼
测试本地连接 ──> 失败 ──> 检查监听地址和端口
    │
    ├── 本地成功
    ▼
检查防火墙 ──> 被拦截 ──> 添加防火墙规则
    │
    ├── 已放行
    ▼
检查网络 ──> 网络不通 ──> 检查网络配置
    │
    ├── 网络正常
    ▼
检查代理配置 ──> 配置错误 ──> 修正客户端代理设置
```

**排查步骤**：

1. **确认服务运行**：`tasklist | findstr Prism.exe`
2. **检查防火墙**：允许代理端口通过
3. **测试连接**：`telnet 127.0.0.1 8081`

**解决方案**：

- Windows 防火墙：控制面板 → 系统和安全 → Windows 防火墙 → 允许应用通过防火墙
- Linux 防火墙：`sudo ufw allow 8081/tcp`
- 检查 Prism 监听地址是否为 0.0.0.0（而非 127.0.0.1）

### Q3：连接超时

**排查流程图**：

```
连接超时
    │
    ▼
测试网络延迟 ──> 延迟过高 ──> 检查网络质量/更换线路
    │
    ├── 延迟正常
    ▼
检查服务器状态 ──> 服务器无响应 ──> 检查服务器日志
    │
    ├── 服务器正常
    ▼
检查协议配置 ──> 协议不匹配 ──> 核对客户端/服务器协议配置
    │
    ├── 配置匹配
    ▼
检查 DNS ──> DNS 解析失败 ──> 更换 DNS 服务器
```

## 协议问题排查

### Q4：TLS 握手失败

**排查流程图**：

```
TLS 握手失败
    │
    ▼
检查证书文件 ──> 文件不存在/权限不足 ──> 修正证书路径和权限
    │
    ├── 文件正常
    ▼
检查证书有效性 ──> 过期/格式错误 ──> 重新生成或续期证书
    │
    ├── 证书有效
    ▼
检查 SNI 匹配 ──> SNI 不匹配 ──> 核对客户端 SNI 配置
    │
    ├── SNI 匹配
    ▼
检查 TLS 版本 ──> 版本不兼容 ──> 统一 TLS 1.3
    │
    ├── 版本兼容
    ▼
检查密码套件 ──> 无共同密码套件 ──> 使用兼容的密码套件
```

### Q5：证书问题

**排查步骤**：

1. **生成证书**：使用 OpenSSL 生成有效证书
2. **检查路径**：确保证书文件存在且有读取权限
3. **格式验证**：确保证书为 PEM 格式

**生成自签名证书**：

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# 验证证书
openssl x509 -in cert.pem -text -noout

# 验证证书链
openssl verify -CAfile ca.pem cert.pem
```

### Q6：认证失败

| 错误现象 | 可能原因 | 解决方案 |
|---------|---------|---------|
| 401 Unauthorized | 密码/UUID 不匹配 | 核对客户端和服务器配置 |
| 握手后立即断开 | 认证超时 | 检查时间同步（NTP） |
| 间歇性认证失败 | 负载均衡配置不一致 | 统一多服务器配置 |

## 性能问题排查

### Q7：速度慢

**排查流程图**：

```
速度慢
    │
    ▼
测试直连速度 ──> 直连也慢 ──> 网络带宽问题
    │
    ├── 直连正常
    ▼
检查服务器带宽 ──> 带宽不足 ──> 升级服务器带宽
    │
    ├── 带宽充足
    ▼
调整连接池 ──> 缓存不足 ──> 增加缓存数量
    │
    ├── 连接池正常
    ▼
检查协议开销 ──> 协议开销大 ──> 使用更高效的协议
    │
    ├── 协议正常
    ▼
检查 TCP 参数 ──> 参数不合理 ──> 调整 TCP 缓冲区/拥塞控制
```

**解决方案**：

```json
{
  "pool": {
    "max_cache_per_endpoint": 64,
    "max_idle_seconds": 30
  }
}
```

### Q8：连接不稳定

| 现象 | 可能原因 | 解决方案 |
|------|---------|---------|
| 频繁断开 | TCP 超时设置过短 | 增加 keepalive 超时 |
| 间歇性丢包 | 网络质量差 | 使用 QUIC 协议 |
| 大文件传输中断 | 缓冲区溢出 | 调整 TCP 缓冲区大小 |
| 高峰期卡顿 | 服务器过载 | 增加服务器/使用负载均衡 |

## DNS 问题排查

### Q9：DNS 解析失败

**排查流程图**：

```
DNS 解析失败
    │
    ▼
检查 DNS 配置 ──> DNS 服务器不可达 ──> 更换 DNS 服务器
    │
    ├── DNS 可达
    ▼
检查 fake-ip 模式 ──> fake-ip 异常 ──> 检查 fake-ip 过滤列表
    │
    ├── fake-ip 正常
    ▼
检查 DNS 规则 ──> 规则配置错误 ──> 修正 DNS 路由规则
    │
    ├── 规则正常
    ▼
检查上游 DNS ──> 上游 DNS 被封 ──> 更换 DNS 或使用 DoH/DoT
```

### Q10：DNS 泄漏

| 泄漏类型 | 检测方法 | 解决方案 |
|---------|---------|---------|
| 系统 DNS 泄漏 | dnsleaktest.com | 使用 fake-ip 模式 |
| IPv6 DNS 泄漏 | ipleak.net | 禁用 IPv6 或配置 IPv6 代理 |
| 应用 DNS 泄漏 | Wireshark 抓包 | 使用 TUN 模式 |
| mDNS 泄漏 | 网络监控 | 过滤 .local 域名 |

## 与 mihomo 客户端的配合排查

### mihomo + Prism 联合排查

```
┌─────────────────────────────────────────────────────────────┐
│                mihomo + Prism 联合排查流程                   │
│                                                             │
│  ① 确认 Prism 服务器正常                                    │
│     ├── 检查 Prism 日志                                     │
│     ├── 测试 Prism 端口连通性                               │
│     └── 验证 Prism 配置正确性                               │
│                                                             │
│  ② 确认 mihomo 客户端配置                                   │
│     ├── 检查代理节点配置（地址/端口/密码）                  │
│     ├── 检查 DNS 配置（fake-ip/规则）                       │
│     └── 检查路由规则（是否正确匹配）                        │
│                                                             │
│  ③ 联合测试                                                 │
│     ├── mihomo 日志查看连接状态                             │
│     ├── Prism 日志查看接收状态                              │
│     └── 对比两端日志定位问题                                │
│                                                             │
│  ④ 常见联合问题                                             │
│     ├── TLS 版本/密码套件不匹配                             │
│     ├── 证书/SNI 配置不一致                                 │
│     ├── 超时设置不合理                                      │
│     └── DNS 解析策略冲突                                    │
└─────────────────────────────────────────────────────────────┘
```

### mihomo 配置检查清单

```yaml
# mihomo 配置检查要点
proxies:
  - name: "prism-proxy"
    type: vless              # 协议类型与 Prism 匹配
    server: your-server.com  # Prism 服务器地址
    port: 443                # Prism 监听端口
    uuid: your-uuid          # 与 Prism 配置一致
    tls: true                # 启用 TLS
    servername: your-sni     # SNI 与 Prism 证书匹配

dns:
  enable: true
  enhanced-mode: fake-ip     # 推荐 fake-ip
  nameserver:
    - https://dns.cloudflare.com/dns-query

rules:
  - GEOIP,private,DIRECT,no-resolve
  - MATCH,Prism              # 默认走 Prism 代理
```

## 日志分析方法

### Prism 日志级别

| 级别 | 内容 | 关注场景 |
|------|------|---------|
| error | 错误信息 | 连接失败、配置错误 |
| warn | 警告信息 | 性能问题、潜在风险 |
| info | 一般信息 | 连接状态、流量统计 |
| debug | 调试信息 | 详细排查、协议分析 |

### 日志分析技巧

**按级别过滤**：重点关注 `error` 和 `warn` 级别日志。

**按时间搜索**：查找特定时间段的日志，定位问题发生时刻。

**关键词搜索**：搜索特定 IP、域名或错误代码。

**模式识别**：寻找重复出现的错误模式，识别系统性问题。

### 常见错误码和解决方案

| 错误码/信息 | 含义 | 解决方案 |
|------------|------|---------|
| `connection refused` | 连接被拒绝 | 检查服务是否运行、端口是否正确 |
| `connection timeout` | 连接超时 | 检查网络、防火墙、服务器状态 |
| `TLS handshake error` | TLS 握手失败 | 检查证书、SNI、TLS 版本 |
| `certificate verify failed` | 证书验证失败 | 检查证书有效性、信任链 |
| `authentication failed` | 认证失败 | 核对密码/UUID |
| `address already in use` | 端口被占用 | 更换端口或停止占用进程 |
| `too many open files` | 文件描述符不足 | 增加系统限制（ulimit） |
| `DNS resolve failed` | DNS 解析失败 | 检查 DNS 配置 |

### 启用详细日志

```json
{
  "trace": {
    "log_level": "debug",
    "enable_console": true,
    "path_name": "./logs",
    "file_name": "prism.log"
  }
}
```

### mihomo 日志分析

```bash
# mihomo 日志位置
tail -f ~/.config/mihomo/mihomo.log

# 搜索错误
grep -i "error\|fail\|timeout" ~/.config/mihomo/mihomo.log

# 搜索特定代理节点
grep "prism-proxy" ~/.config/mihomo/mihomo.log
```

## 快速排查检查清单

| 序号 | 检查项 | 命令/方法 |
|------|--------|---------|
| 1 | Prism 进程运行 | `tasklist \| findstr Prism` |
| 2 | 端口监听 | `netstat -ano \| findstr :443` |
| 3 | 防火墙放行 | `netsh advfirewall firewall show rule name=all` |
| 4 | 网络连通 | `ping your-server.com` |
| 5 | 端口连通 | `telnet your-server.com 443` |
| 6 | TLS 握手 | `openssl s_client -connect your-server.com:443` |
| 7 | DNS 解析 | `nslookup your-server.com` |
| 8 | 证书状态 | `echo \| openssl s_client -connect your-server.com:443 2>/dev/null \| openssl x509 -text -noout` |

## 相关页面

- [[agent]] — 项目主页
- [[agent/configuration]] — 配置详解
- [[agent/deployment]] — 部署指南
- [[client/mihomo-meta]] — mihomo 客户端
- [[stealth/proxy-detection]] — 代理检测技术
- [[stealth/pki-certificates]] — PKI 证书体系
- [[client/mihomo-dns]] — DNS 相关问题
