---
title: 故障排查
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 故障排查

Prism 常见问题的排查方法和解决方案。

## 功能简介

提供故障排查流程、日志分析方法、常见错误码解读、以及各模块问题的解决步骤。

## 故障排查流程

当遇到问题时，按照以下流程逐步定位：

```
1. Prism 是否正在运行？
   ├── 否 → 启动服务，查看启动错误
   └── 是 ↓

2. 端口是否正确监听？
   ├── 否 → 检查 agent.addressable.port 配置
   └── 是 ↓

3. 能否从本机连接？
   ├── 否 → 检查防火墙、端口占用
   └── 是 ↓

4. 能否从远程连接？
   ├── 否 → 检查 host=0.0.0.0、防火墙、路由器
   └── 是 ↓

5. 代理功能是否正常？
   ├── 否 → 检查协议是否启用、密码是否正确
   └── 是 → 性能问题，检查连接池和缓冲区配置
```

## 日志分析

### 开启调试日志

```json
{
  "trace": {
    "log_level": "debug",
    "path_name": "/var/log/prism/",
    "enable_console": true
  }
}
```

### 日志级别说明

| 级别 | 内容 | 适用场景 |
|------|------|----------|
| `trace` | 每帧数据、握手细节 | 协议开发调试 |
| `debug` | 连接建立/关闭、配置加载 | 排查连接问题 |
| `info` | 服务启动/停止、正常事件 | 生产环境推荐 |
| `warn` | 配置缺失使用默认值、重试 | 关注潜在问题 |
| `error` | 不可恢复的错误 | 必须处理 |

### 查看日志

```bash
# Linux systemd
sudo journalctl -u prism -f --since "10 minutes ago"

# Linux 文件
tail -f /var/log/prism/prism.log

# Windows PowerShell
Get-Content C:\Prism\logs\prism.log -Wait -Tail 50
```

### 关键日志模式

| 日志关键字 | 含义 |
|------------|------|
| `failed to bind` | 端口已被占用 |
| `certificate not found` | TLS 证书路径错误或文件不存在 |
| `authentication failed` | 密码/UUID 不匹配 |
| `connection refused` | 目标服务不可达 |
| `TLS handshake error` | TLS 协商失败，检查证书或 SNI |
| `pool exhausted` | 连接池耗尽，需增大 max_cache_per_endpoint |

## 启动失败

### 排查步骤

1. 确认运行环境（MinGW 编译，无外部依赖）
2. 检查配置文件 JSON 格式是否正确
3. 查看日志目录是否有写入权限
4. 检查端口是否被占用

### 端口占用检查

```bash
# Linux
sudo lsof -i :8081
sudo ss -tlnp | grep 8081

# Windows
netstat -ano | findstr :8081
tasklist | findstr <PID>
```

### 配置文件验证

```bash
# 检查 JSON 格式
python3 -m json.tool configuration.json > /dev/null && echo "JSON OK" || echo "JSON INVALID"

# 检查文件权限
ls -la /etc/prism/configuration.json
```

## 无法连接

### 排查步骤

1. 确认服务运行中
2. 检查防火墙规则
3. 测试端口连通性
4. 验证网络路由

```bash
# Windows: 确认服务进程
tasklist | findstr Prism.exe

# Linux: 确认服务状态
systemctl status prism

# 测试端口
telnet 127.0.0.1 8081
# 或
nc -zv 127.0.0.1 8081
```

### 防火墙配置

- **Windows**: 控制面板 > 防火墙 > 高级设置 > 入站规则 > 新建规则 > TCP > 端口 8081
- **Linux (ufw)**: `sudo ufw allow 8081/tcp`
- **Linux (firewalld)**: `sudo firewall-cmd --permanent --add-port=8081/tcp && sudo firewall-cmd --reload`
- **云平台**: 检查 AWS 安全组 / 阿里云安全组 / GCP 防火墙规则

## 速度慢

### 排查步骤

1. 测试直连速度（不经过代理）
2. 调整连接池参数
3. 检查 TCP 参数配置
4. 确认服务器网络状况

### 优化配置

```json
{
  "pool": {
    "max_cache_per_endpoint": 64,
    "max_idle_seconds": 120,
    "tcp_nodelay": true,
    "recv_buffer_size": 131072,
    "send_buffer_size": 131072
  },
  "multiplex": {
    "smux": {
      "enabled": true,
      "buffer_size": 32768,
      "max_streams": 128
    }
  }
}
```

### 网络诊断

```bash
# 测试延迟
ping your.server.com

# 测试带宽
curl -o /dev/null -w "%{time_total}" http://speedtest.server.com/100mb

# 跟踪路由
traceroute your.server.com
```

## 证书问题

### 排查步骤

1. 确认 PEM 格式正确
2. 检查文件读取权限
3. 验证证书有效期
4. 确认 SNI 域名匹配

### 验证证书

```bash
# 检查证书信息
openssl x509 -in cert.pem -text -noout

# 检查私钥
openssl rsa -in key.pem -check

# 测试 TLS 连接
openssl s_client -connect your.server.com:8081 -servername your.server.com
```

### 重新生成证书

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -nodes -subj "/CN=your.server.com"
```

## 常见错误码

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `failed to bind: Address already in use` | 端口被占用 | 更换端口或终止占用进程 |
| `configuration.json: parse error` | JSON 格式无效 | 用 json.tool 验证并修复 |
| `No such file or directory` (TLS) | 证书路径错误 | 检查路径，确保文件存在 |
| `Permission denied` (TLS) | 文件权限不足 | `chmod 644 cert.pem && chmod 600 key.pem` |
| `authentication failed` | 密码/UUID 不匹配 | 核对客户端配置与服务端 |
| `connection pool exhausted` | 并发超过池容量 | 增加 `max_cache_per_endpoint` |
| `TLS handshake failed` | 证书/SNI/协议版本不匹配 | 检查证书有效性和 SNI 配置 |
| `DNS resolution failed` | DNS 服务器不可达 | 更换 DNS 或检查网络 |
| `max_connections reached` | 用户连接数超限 | 增加 `max_connections` 或检查是否有滥用 |
| `buffer allocation failed` | 内存不足 | 降低 buffer_size 或增加系统内存 |

## 调试模式完整配置

```json
{
  "trace": {
    "log_level": "debug",
    "path_name": "./logs/",
    "max_size": 52428800,
    "max_files": 10,
    "enable_console": true
  }
}
```

调试模式下，所有连接建立、断开、数据传输细节都会输出到日志。完成排查后记得将 `log_level` 改回 `info`。

## 获取帮助

如果以上步骤无法解决问题：

1. 收集以下信息：
   - Prism 版本：`./Prism --version`
   - 操作系统和版本
   - 完整配置文件（脱敏后）
   - 错误日志（最近 100 行）
   - 复现步骤
2. 提交 Issue 到项目仓库

## 已知问题

Prism 存在一些已知的实现限制，可能导致排错困难。详见 [[docs/known-issues|已知问题清单]]。

关键已知问题：
- 无优雅关闭机制（只能强制终止）
- SOCKS5 UDP 不支持 IPv6 客户端
- VLESS 不支持 XTLS/Vision flow
- SS2022 两端时钟需同步（30s 窗口）

## 相关页面

- [[docs/faq|常见问题]]
- [[log|日志系统]]
- [[dev/testing/overview|测试框架]]
- [[docs/configuration|配置说明]]
