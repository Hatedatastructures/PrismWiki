---
title: 安全注意事项
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide, security]
---

# 安全注意事项

Prism 生产环境安全配置要点。

## 功能简介

提供认证、防火墙、日志监控、证书管理、密码安全、运行权限等安全最佳实践。

## 认证配置

```json
{
  "agent": {
    "authentication": {
      "users": [
        {
          "password": "your-strong-password-here-at-least-16-chars",
          "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
          "max_connections": 10
        }
      ]
    }
  }
}
```

### 认证安全最佳实践

- **密码强度**: 至少 16 字符，混合大小写字母、数字、符号。避免使用字典词汇、生日、常用密码
- **UUID 安全**: 使用 `uuidgen` 或 `/dev/urandom` 生成真正的随机 UUID
- **连接限制**: 设置合理的 `max_connections`（5-20），防止单用户耗尽资源
- **定期轮换**: 每 30-90 天更换一次认证凭据
- **最小权限**: 仅给需要的用户分配凭据，不使用共享凭据

### 生成强凭据

```bash
# 生成强密码
openssl rand -base64 32

# 生成 UUID
uuidgen
# 或
python3 -c "import uuid; print(uuid.uuid4())"
```

## 网络安全

### 防火墙规则

```bash
# 仅开放必要端口
sudo ufw allow 8081/tcp

# 限制来源 IP（可选，仅允许特定网段访问）
sudo ufw allow from 192.168.1.0/24 to any port 8081

# 拒绝其他所有入站连接
sudo ufw default deny incoming
```

### 非标准端口

使用非标准端口可降低被自动化扫描发现的概率：

```json
{
  "agent": {
    "addressable": {
      "host": "0.0.0.0",
      "port": 44312
    }
  }
}
```

> **注意**: 使用 443 端口可伪装为 HTTPS 流量，但需注意不要与其他 HTTPS 服务冲突。

### Fail2ban 集成（Linux）

```bash
sudo tee /etc/fail2ban/filter.d/prism.conf > /dev/null << 'EOF'
[Definition]
failregex = ^.*authentication failed.*<HOST>.*$
            ^.*connection refused.*<HOST>.*$
EOF

sudo tee -a /etc/fail2ban/jail.local > /dev/null << 'EOF'
[prism]
enabled = true
filter = prism
logpath = /var/log/prism/*.log
maxretry = 5
bantime = 3600
findtime = 600
EOF

sudo systemctl restart fail2ban
```

## 证书管理

### 证书类型选择

| 类型 | 适用场景 | 有效期 | 成本 |
|------|----------|--------|------|
| Let's Encrypt | 生产环境，需要浏览器信任 | 90 天 | 免费 |
| 商业证书 | 企业生产环境 | 1 年 | 付费 |
| 自签名 | 测试/内网环境 | 自定义 | 免费 |

### 证书部署

```bash
# 生成 Let's Encrypt 证书
sudo certbot certonly --standalone -d your.server.com

# 配置证书路径
# cert_path: /etc/letsencrypt/live/your.server.com/fullchain.pem
# key_path: /etc/letsencrypt/live/your.server.com/privkey.pem
```

### 证书自动续期

```bash
# 测试续期
sudo certbot renew --dry-run

# 配置 cron 自动续期
echo "0 3 * * * root certbot renew --quiet && systemctl restart prism" | sudo tee /etc/cron.d/certbot-renew-prism
```

### 私钥保护

```bash
# 私钥文件权限必须严格限制
chmod 600 /etc/prism/key.pem
chmod 644 /etc/prism/cert.pem
chown prism:prism /etc/prism/key.pem
```

> **重要**: 私钥泄露等同于服务被完全控制。绝不将私钥提交到版本控制系统或共享给他人。

### 证书检查清单

- [ ] 证书有效期充足
- [ ] CN/SAN 包含服务的域名
- [ ] 私钥权限为 600
- [ ] 已配置自动续期（如使用 Let's Encrypt）
- [ ] 备份了证书和私钥

## 密码安全

### 密码存储

Prism 的密码存储在配置文件中，因此配置文件本身需要保护：

```bash
# 配置文件权限
chmod 600 /etc/prism/configuration.json
chown prism:prism /etc/prism/configuration.json
```

### 密码管理

- 不要在代码仓库中存储配置文件
- 不要通过明文渠道（邮件、聊天）传输密码
- 使用密码管理器生成和存储凭据
- 定期审计和轮换密码

### 多个凭据管理

为不同用户分配不同凭据，便于追踪和隔离：

```json
{
  "agent": {
    "authentication": {
      "users": [
        { "password": "user-a-password", "uuid": "uuid-a", "max_connections": 5 },
        { "password": "user-b-password", "uuid": "uuid-b", "max_connections": 5 }
      ]
    }
  }
}
```

## 配置文件安全

```bash
# 文件权限
chmod 600 /etc/prism/configuration.json
chown prism:prism /etc/prism/configuration.json

# 验证权限
ls -la /etc/prism/configuration.json
# 应显示: -rw------- 1 prism prism
```

### 版本控制排除

确保配置文件不在版本控制范围内：

```bash
# .gitignore
echo "configuration.json" >> .gitignore
echo "*.pem" >> .gitignore
echo "*.key" >> .gitignore
```

## 运行权限

### 非 root 运行

```bash
# 创建专用用户
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prism

# systemd 服务配置
[Service]
User=prism
Group=prism
# 如果需要监听 1024 以下端口，添加以下能力
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### 最小权限原则

- 不使用 root/Administrator 运行
- 限制文件系统访问范围
- 限制网络访问范围
- 禁用不必要的系统调用（可通过 seccomp / AppArmor）

## 日志监控

### 生产环境日志配置

```json
{
  "trace": {
    "log_level": "info",
    "path_name": "/var/log/prism/",
    "max_size": 10485760,
    "max_files": 7,
    "enable_console": false
  }
}
```

### 异常检测

定期检查以下模式：

| 模式 | 含义 | 响应 |
|------|------|------|
| 大量认证失败 | 暴力破解攻击 | 屏蔽来源 IP，检查凭据 |
| 异常高频连接 | 资源耗尽攻击 | 限制 max_connections |
| 未知 IP 连接 | 未授权访问 | 检查防火墙和认证配置 |
| 大量 TLS 握手失败 | 扫描或攻击 | 检查来源，考虑屏蔽 |

### 日志审计脚本

```bash
#!/bin/bash
# 检查最近 1 小时的认证失败次数
LOG_DIR="/var/log/prism/"
FAILURES=$(grep -c "authentication failed" "$LOG_DIR"/*.log 2>/dev/null || echo 0)
if [ "$FAILURES" -gt 50 ]; then
  echo "WARNING: $FAILURES authentication failures detected"
fi
```

## 应急响应

### 发现可疑连接

1. 立即修改端口和认证凭据
2. 查看日志追踪来源 IP
3. 防火墙屏蔽可疑 IP
4. 评估是否有数据泄露

### 服务被入侵

1. 停止服务
2. 保留日志用于取证
3. 更换所有认证凭据
4. 检查系统完整性
5. 从干净状态重新启动

## 常见问题

**Q: 发现可疑连接怎么办？**
A: 立即修改端口、查看日志追踪来源、防火墙屏蔽可疑 IP。如果怀疑凭据已泄露，立即更换所有密码和 UUID。

**Q: 如何防止端口扫描？**
A: 使用非标准端口、配置防火墙限速、启用 fail2ban。但请注意，没有任何方法可以完全阻止端口扫描，重点在于认证和监控。

**Q: 配置文件被泄露了怎么办？**
A: 立即更换所有认证凭据（密码和 UUID），检查日志确认是否有未授权访问，修复泄露原因。

**Q: 是否需要定期更新 Prism？**
A: 是的，定期检查 [GitHub Releases](https://github.com/Hatedatastructures/Prism/releases)，及时应用安全修复。详见 [[docs/upgrade|升级指南]]。

## 相关页面

- [[docs/deployment|部署指南]]
- [[core/stealth/reality/handshake|Reality 伪装]]
- [[core/stealth/shadowtls/overview|ShadowTLS 伪装]]
- [[log|日志系统]]
- [[docs/upgrade|升级指南]]
