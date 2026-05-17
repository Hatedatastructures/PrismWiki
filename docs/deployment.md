---
title: 部署指南
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 部署指南

生产环境部署 Prism 的完整步骤，包含 systemd、Docker、Windows 服务。

## 功能简介

提供系统要求、编译安装、服务配置、防火墙、TLS 证书等部署全流程。

## 系统要求

| 资源 | 最低 | 推荐 |
|------|------|------|
| CPU | 2 核 | 4 核 |
| 内存 | 512 MB | 2 GB |
| 系统 | Windows 10+, Linux glibc 2.31+ | |

## 如何启用

### Linux systemd

```bash
sudo cp build_release/src/Prism /usr/local/bin/prism
sudo cp src/configuration.json /etc/prism/
sudo systemctl enable --now prism
```

### Docker

```bash
docker build -t prism .
docker run -d -p 8081:8081 -v ./config:/etc/prism:ro prism
```

### Windows 服务

```powershell
sc create Prism binPath= "C:\Prism\Prism.exe" start= auto
sc start Prism
```

## 常见问题

**Q: 如何配置防火墙？**
A: Linux: `sudo ufw allow 8081/tcp`；Windows: `netsh advfirewall firewall add rule name="Prism" dir=in action=allow protocol=TCP localport=8081`

**Q: TLS 证书怎么获取？**
A: 测试用自签名：`openssl req -x509 -newkey ec ...`；生产用 Let's Encrypt。

## 相关页面

- [[docs/getting-started|快速开始]]
- [[docs/security|安全注意事项]]
- [[dev/stress|压力测试]]