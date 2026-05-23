---
title: sing-box 配置指南
created: 2026-05-23
updated: 2026-05-23
layer: docs
tags: [docs, client, sing-box, guide]
---

# sing-box 配置指南

sing-box 是新一代通用代理平台，支持多种协议。本文提供 sing-box 连接 Prism 服务器的完整配置指南。

## 安装 sing-box

### Linux

```bash
# Debian/Ubuntu
curl -Lo sing-box.deb https://github.com/SagerNet/sing-box/releases/latest/download/sing-box_1.11.0_linux_amd64.deb
sudo dpkg -i sing-box.deb

# 通用二进制
curl -Lo sing-box.tar.gz https://github.com/SagerNet/sing-box/releases/latest/download/sing-box-1.11.0-linux-amd64.tar.gz
tar -xzf sing-box.tar.gz
chmod +x sing-box-1.11.0-linux-amd64/sing-box
```

### macOS

```bash
brew install sing-box
```

### Windows

从 [GitHub Releases](https://github.com/SagerNet/sing-box/releases) 下载 `sing-box-1.11.0-windows-amd64.zip`。

### Android

使用 SFA (sing-box for Android)，从 GitHub Releases 下载。

## 基础配置结构

sing-box 使用 JSON 格式配置文件：

```json
{
  "log": { ... },
  "dns": { ... },
  "inbounds": [ ... ],
  "outbounds": [ ... ],
  "route": { ... }
}
```

## 完整配置模板

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },

  "dns": {
    "servers": [
      {
        "tag": "google",
        "address": "tls://8.8.8.8"
      },
      {
        "tag": "local",
        "address": "https://223.5.5.5/dns-query",
        "detour": "direct"
      }
    ],
    "rules": [
      {
        "outbound": "any",
        "server": "local"
      }
    ],
    "strategy": "ipv4_only"
  },

  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 7890
    },
    {
      "type": "tun",
      "tag": "tun-in",
      "inet4_address": "172.19.0.1/30",
      "auto_route": true,
      "strict_route": false,
      "sniff": true,
      "sniff_override_destination": true
    }
  ],

  "outbounds": [
    {
      "type": "vless",
      "tag": "vless-reality",
      "server": "your.server.com",
      "server_port": 8081,
      "uuid": "123e4567-e89b-12d3-a456-426614174000",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "your-reality-public-key",
          "short_id": "your-short-id"
        }
      }
    },
    {
      "type": "shadowsocks",
      "tag": "ss2022",
      "server": "your.server.com",
      "server_port": 8081,
      "method": "2022-blake3-aes-128-gcm",
      "password": "your-base64-psk"
    },
    {
      "type": "trojan",
      "tag": "trojan",
      "server": "your.server.com",
      "server_port": 8081,
      "password": "your-trojan-password",
      "tls": {
        "enabled": true,
        "server_name": "your.server.com",
        "insecure": true
      }
    },
    {
      "type": "socks",
      "tag": "socks5",
      "server": "your.server.com",
      "server_port": 8081,
      "version": "5",
      "username": "admin",
      "password": "your-password"
    },
    {
      "type": "http",
      "tag": "http",
      "server": "your.server.com",
      "server_port": 8081,
      "username": "username",
      "password": "your-password"
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],

  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "ip_is_private": true,
        "outbound": "direct"
      }
    ],
    "final": "vless-reality"
  }
}
```

## 协议配置详解

### VLESS + Reality

```json
{
  "type": "vless",
  "tag": "vless-reality",
  "server": "your.server.com",
  "server_port": 8081,
  "uuid": "123e4567-e89b-12d3-a456-426614174000",
  "flow": "",
  "tls": {
    "enabled": true,
    "server_name": "www.microsoft.com",
    "utls": {
      "enabled": true,
      "fingerprint": "chrome"
    },
    "reality": {
      "enabled": true,
      "public_key": "your-reality-public-key-base64",
      "short_id": "your-short-id-hex"
    }
  }
}
```

**配置要点**:
- `server_name` 必须与服务端 `reality.server_names` 一致
- `public_key` 是公钥（不是私钥），使用 `GenRealityKeys.sh` 生成
- `short_id` 必须与服务端 `reality.short_ids` 列表中的某一项匹配
- `utls.fingerprint` 推荐 `chrome`，模拟 Chrome TLS 指纹
- `flow` 留空（Prism 不使用 XTLS flow）

### VLESS + Reality + 多路复用

```json
{
  "type": "vless",
  "tag": "vless-reality-mux",
  "server": "your.server.com",
  "server_port": 8081,
  "uuid": "123e4567-e89b-12d3-a456-426614174000",
  "tls": {
    "enabled": true,
    "server_name": "www.microsoft.com",
    "utls": {
      "enabled": true,
      "fingerprint": "chrome"
    },
    "reality": {
      "enabled": true,
      "public_key": "your-reality-public-key",
      "short_id": "your-short-id"
    }
  },
  "multiplex": {
    "enabled": true,
    "protocol": "smux",
    "max_connections": 4,
    "min_streams": 4,
    "max_streams": 64
  }
}
```

**Yamux 多路复用**:

```json
{
  "multiplex": {
    "enabled": true,
    "protocol": "yamux",
    "max_connections": 4,
    "max_streams": 64
  }
}
```

### Trojan

```json
{
  "type": "trojan",
  "tag": "trojan",
  "server": "your.server.com",
  "server_port": 8081,
  "password": "your-trojan-password",
  "tls": {
    "enabled": true,
    "server_name": "your.server.com",
    "insecure": true
  },
  "udp_over_tcp": false
}
```

**Trojan + 多路复用**:

```json
{
  "type": "trojan",
  "tag": "trojan-mux",
  "server": "your.server.com",
  "server_port": 8081,
  "password": "your-trojan-password",
  "tls": {
    "enabled": true,
    "server_name": "your.server.com",
    "insecure": true
  },
  "multiplex": {
    "enabled": true,
    "protocol": "smux",
    "max_connections": 4,
    "max_streams": 64
  }
}
```

### SS2022 (Shadowsocks 2022)

```json
{
  "type": "shadowsocks",
  "tag": "ss2022",
  "server": "your.server.com",
  "server_port": 8081,
  "method": "2022-blake3-aes-128-gcm",
  "password": "your-base64-psk"
}
```

**加密方法对应关系**:

| PSK 长度 | method |
|----------|--------|
| 16 字节 (base64) | `2022-blake3-aes-128-gcm` |
| 32 字节 (base64) | `2022-blake3-aes-256-gcm` |

### SS2022 + ShadowTLS

sing-box 不原生支持 ShadowTLS plugin。如果需要 ShadowTLS 伪装，建议使用 [[docs/client-setup/clash-verge|Clash Verge]] 客户端。

作为替代方案，sing-box 可以通过自定义传输层实现类似效果：

```json
{
  "type": "shadowsocks",
  "tag": "ss2022",
  "server": "your.server.com",
  "server_port": 8081,
  "method": "2022-blake3-aes-128-gcm",
  "password": "your-base64-psk",
  "tls": {
    "enabled": true,
    "server_name": "your.server.com",
    "insecure": true
  }
}
```

### SOCKS5

```json
{
  "type": "socks",
  "tag": "socks5",
  "server": "your.server.com",
  "server_port": 8081,
  "version": "5",
  "username": "admin",
  "password": "your-password",
  "udp_timeout": "5m"
}
```

### HTTP

```json
{
  "type": "http",
  "tag": "http-proxy",
  "server": "your.server.com",
  "server_port": 8081,
  "username": "username",
  "password": "your-password",
  "tls": {
    "enabled": true,
    "server_name": "your.server.com",
    "insecure": true
  }
}
```

## 启动与使用

### 命令行启动

```bash
# 基本启动
sing-box run -c config.json

# 后台运行
nohup sing-box run -c config.json > sing-box.log 2>&1 &

# 检查配置有效性
sing-box check -c config.json
```

### systemd 服务

```bash
sudo tee /etc/systemd/system/sing-box.service > /dev/null << 'EOF'
[Unit]
Description=sing-box Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/sing-box run -c /etc/sing-box/config.json
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable sing-box
sudo systemctl start sing-box
```

### 系统代理设置

启动 sing-box 后，配置系统代理：

```bash
# Linux: 设置环境变量
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5://127.0.0.1:7890

# 或使用 TUN 模式自动全局代理（需要 root）
# 在配置中启用 tun inbound 即可
```

## 常见问题

**Q: VLESS Reality 连接失败？**
A: 1) 检查 `public_key` 是否正确（不是 private_key）。2) 检查 `short_id` 与服务端匹配。3) 确认 `server_name` 与服务端 `server_names` 一致。

**Q: SS2022 认证失败？**
A: 确认 `password` 是 base64 编码的 PSK，与服务端 `psk` 字段完全一致。`method` 必须与服务端 `method` 匹配。

**Q: 多路复用连接不稳定？**
A: 1) `max_streams` 不超过服务端配置。2) 检查网络延迟，高延迟下适当降低 `max_connections`。3) Smux 适合低延迟场景，Yamux 适合高延迟高吞吐场景。

**Q: sing-box 不支持 ShadowTLS 怎么办？**
A: sing-box 目前不原生支持 ShadowTLS plugin。推荐使用 [[docs/client-setup/clash-verge|Clash Verge]] 作为替代客户端，或在 sing-box 中使用 VLESS + Reality 作为高安全性方案。

**Q: TUN 模式无法工作？**
A: 需要管理员/root 权限运行 sing-box。Linux 上还需要 `tun` 内核模块。检查 `auto_route` 和 `strict_route` 配置。

## 相关页面

- [[docs/client-setup|客户端配置总览]]
- [[docs/client-setup/clash-verge|Clash Verge 配置指南]]
- [[docs/protocol-matrix|协议能力矩阵]]
- [[docs/configuration|配置说明]]
- [[docs/troubleshooting|故障排查]]
