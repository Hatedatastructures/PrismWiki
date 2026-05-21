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
| 磁盘 | 100 MB | 500 MB（含日志） |
| 系统 | Windows 10+, Linux glibc 2.31+ | Ubuntu 22.04+, Debian 12+ |
| 网络 | 1 Mbps | 10 Mbps+ |

## Linux 部署

### 1. 编译与安装

```bash
# 编译
cd Prism
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 安装二进制和配置
sudo mkdir -p /etc/prism /var/log/prism
sudo cp build_release/src/Prism /usr/local/bin/prism
sudo cp src/configuration.json /etc/prism/configuration.json
```

### 2. 创建专用用户

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prism
sudo chown -R prism:prism /etc/prism /var/log/prism
sudo chmod 600 /etc/prism/configuration.json
```

### 3. 创建 systemd 服务

```bash
sudo tee /etc/systemd/system/prism.service > /dev/null << 'EOF'
[Unit]
Description=Prism Proxy Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prism
Group=prism
ExecStart=/usr/local/bin/prism --config /etc/prism/configuration.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

### 4. 启动与自启

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prism
sudo systemctl status prism
```

### 5. 查看日志

```bash
# 实时日志
sudo journalctl -u prism -f

# 最近 100 行
sudo journalctl -u prism -n 100

# 按级别过滤
sudo journalctl -u prism --priority=err
```

## Docker 部署

### Dockerfile

```dockerfile
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y cmake g++ git
COPY . /Prism
WORKDIR /Prism
RUN cmake -B build_release -DCMAKE_BUILD_TYPE=Release && \
    cmake --build build_release --config Release

FROM ubuntu:22.04
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /Prism/build_release/src/Prism /usr/local/bin/prism
COPY configuration.json /etc/prism/configuration.json
EXPOSE 8081
ENTRYPOINT ["/usr/local/bin/prism", "--config", "/etc/prism/configuration.json"]
```

### 构建与运行

```bash
docker build -t prism:latest .
docker run -d \
  --name prism \
  --restart unless-stopped \
  -p 8081:8081 \
  -v $(pwd)/configuration.json:/etc/prism/configuration.json:ro \
  -v $(pwd)/logs:/var/log/prism \
  --memory=512m \
  --cpus=2 \
  prism:latest
```

### Docker Compose

```yaml
version: "3.8"
services:
  prism:
    build: .
    container_name: prism
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - ./configuration.json:/etc/prism/configuration.json:ro
      - ./logs:/var/log/prism
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "2"
```

```bash
docker compose up -d
docker compose logs -f prism
```

## Windows 部署

### 1. 准备文件

```
C:\Prism\
├── Prism.exe        # 编译或下载的二进制
├── configuration.json  # 配置文件
└── logs\            # 日志目录
```

### 2. 注册为 Windows 服务

推荐使用 [NSSM](https://nssm.cc/)（Non-Sucking Service Manager）：

```powershell
# 下载 NSSM 后
nssm install Prism "C:\Prism\Prism.exe"
nssm set Prism AppDirectory "C:\Prism"
nssm set Prism AppStdout "C:\Prism\logs\prism.log"
nssm set Prism AppStderr "C:\Prism\logs\prism-err.log"
nssm set Prism Restart 1  # 自动重启

sc start Prism
```

### 3. 使用 sc 命令（简易方式）

```powershell
sc create Prism binPath= "C:\Prism\Prism.exe --config C:\Prism\configuration.json" start= auto
sc description Prism "Prism Proxy Service"
sc start Prism
```

## 防火墙配置

### Linux (ufw)

```bash
sudo ufw allow 8081/tcp
sudo ufw reload
sudo ufw status
```

### Linux (firewalld)

```bash
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload
```

### Linux (iptables)

```bash
sudo iptables -A INPUT -p tcp --dport 8081 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4
```

### Windows

```powershell
netsh advfirewall firewall add rule name="Prism Proxy" dir=in action=allow protocol=TCP localport=8081
```

## TLS 证书

### 自签名证书（测试用）

```bash
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=your.server.com"
```

### Let's Encrypt（生产用）

```bash
# 使用 certbot
sudo certbot certonly --standalone -d your.server.com
# 证书路径: /etc/letsencrypt/live/your.server.com/fullchain.pem
# 私钥路径: /etc/letsencrypt/live/your.server.com/privkey.pem
```

> **注意**: Let's Encrypt 证书有效期 90 天，需配置自动续期：`sudo certbot renew --dry-run`

## 日志和监控

### 日志轮转（Linux）

```bash
sudo tee /etc/logrotate.d/prism > /dev/null << 'EOF'
/var/log/prism/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 prism prism
    postrotate
        systemctl reload prism 2>/dev/null || true
    endscript
}
EOF
```

### 健康检查

```bash
#!/bin/bash
# health-check.sh - 可配合 cron 或 systemd timer 使用
if ! nc -z 127.0.0.1 8081 2>/dev/null; then
  echo "$(date): Prism is down" | logger
  systemctl restart prism
fi
```

### 资源监控

```bash
# 查看内存和 CPU 使用
systemctl status prism
top -p $(pgrep -o prism)

# Docker 资源统计
docker stats prism
```

## 常见问题

**Q: 如何配置防火墙？**
A: Linux: `sudo ufw allow 8081/tcp`；Windows: `netsh advfirewall firewall add rule name="Prism" dir=in action=allow protocol=TCP localport=8081`

**Q: TLS 证书怎么获取？**
A: 测试用自签名：`openssl req -x509 -newkey ec ...`；生产用 Let's Encrypt。详见上方 TLS 证书章节。

**Q: Docker 容器内如何指定自定义端口？**
A: 修改 `configuration.json` 中的 `agent.addressable.port`，并将 `docker run -p` 的端口映射对应修改。

**Q: 服务启动后自动退出？**
A: 检查日志（`journalctl -u prism -f`），常见原因是配置文件路径错误或端口被占用。

## 相关页面

- [[docs/getting-started|快速开始]]
- [[docs/security|安全注意事项]]
- [[dev/stress|压力测试]]
- [[docs/upgrade|升级指南]]
