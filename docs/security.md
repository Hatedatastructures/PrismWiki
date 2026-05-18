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

提供认证、防火墙、日志监控、证书管理等安全最佳实践。

## 认证配置

```json
{
  "agent": {
    "authentication": {
      "users": [
        {
          "password": "your-strong-password",
          "uuid": "unique-uuid",
          "max_connections": 10
        }
      ]
    }
  }
}
```

- 使用强密码（>= 16 字符，混合大小写数字符号）
- 限制 `max_connections` 防止滥用
- 定期更换密码

## 防火墙

- 仅开放必要端口
- 限制来源 IP 范围（如仅允许特定网段）
- 启用日志记录

## 日志监控

- 生产环境使用 `info` 级别
- 配置日志轮转（`max_size`、`max_files`）
- 定期检查异常连接记录

## 证书管理

- 使用有效证书（Let's Encrypt 或商业证书）
- 设置证书过期提醒
- 保护私钥文件权限（`chmod 600`）

## 配置文件权限

```bash
chmod 600 /etc/prism/configuration.json
chown prism:prism /etc/prism/configuration.json
```

## 运行权限

- 非 root 用户运行
- systemd 服务指定 `User=prism`

## 常见问题

**Q: 发现可疑连接怎么办？**
A: 立即修改端口、查看日志追踪来源、防火墙屏蔽可疑 IP。

**Q: 如何防止端口扫描？**
A: 使用非标准端口、配置防火墙限速、启用 fail2ban。

## 相关页面

- [[docs/deployment|部署指南]]
- [[core/stealth/reality/handshake|Reality 伪装]]
- [[core/stealth/shadowtls|ShadowTLS 伪装]]
- [[log|日志系统]]