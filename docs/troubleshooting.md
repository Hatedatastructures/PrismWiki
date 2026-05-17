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

提供启动失败、连接问题、速度慢、证书问题的排查步骤和解决方案。

## 启动失败

**排查**:
1. 检查运行环境（MinGW 编译，无外部依赖）
2. 查看日志目录
3. 检查端口占用

**解决**:

```powershell
netstat -ano | findstr :8081
```

## 无法连接

**排查**:
1. 确认服务运行：`tasklist | findstr Prism.exe`
2. 检查防火墙
3. 测试端口：`telnet 127.0.0.1 8081`

**解决**:
- Windows: 控制面板 > 防火墙 > 允许应用通过
- Linux: `sudo ufw allow 8081/tcp`

## 速度慢

**排查**:
1. 调整连接池参数
2. 测试直连速度
3. 调整 TCP 参数

**解决**:

```json
{
  "pool": {
    "max_cache_per_endpoint": 64,
    "max_idle_seconds": 30
  }
}
```

## 证书问题

**排查**:
1. 使用 OpenSSL 生成有效证书
2. 检查路径和读取权限
3. 确认 PEM 格式

**解决**:

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

## 调试模式

```json
{
  "trace": {
    "log_level": "debug",
    "enable_console": true
  }
}
```

## 相关页面

- [[docs/faq|常见问题]]
- [[log|日志系统]]
- [[dev/testing|测试框架]]