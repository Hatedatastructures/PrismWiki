---
title: 安全最佳实践
layer: docs
tags: [docs, security, best-practices]
---

# 安全最佳实践

Prism 生产环境的安全配置指南。从密钥管理、TLS 配置、伪装方案选择到网络防护，涵盖部署全链路的安全建议。

## 安全模型概述

Prism 采用三层防御架构来保护代理流量：

1. **TLS 伪装层** — 将代理流量伪装为正常 HTTPS 访问，对抗深度包检测（DPI）。通过伪装方案（Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel）实现。
2. **协议认证层** — 代理协议自带认证机制，防止未授权访问。包括密码认证（Trojan）、UUID 认证（VLESS）、PSK 密钥认证（SS2022）等。
3. **自加密层** — SS2022 等协议在 TLS 之上额外应用 AEAD 加密，提供端到端密文保护。即使 TLS 被中间人破解，流量仍然安全。

配置文件中与安全相关的模块为 `agent.authentication`、`agent.certificate` 和 `stealth` 三部分。

## 密钥管理

### 凭据生成

所有密钥和凭据必须使用密码学安全的随机数生成器。禁止使用人工编写的密码。

**Trojan 密码**（至少 16 字符）：

```bash
openssl rand -base64 32
```

**VLESS UUID**：

```bash
uuidgen
# 或
python3 -c "import uuid; print(uuid.uuid4())"
```

**SS2022 PSK** — 密钥长度必须与加密方法匹配：

| 加密方法 | 密钥长度 | 生成命令 |
|----------|----------|----------|
| `2022-blake3-aes-128-gcm` | 16 字节 | `openssl rand -base64 16` |
| `2022-blake3-aes-256-gcm` | 32 字节 | `openssl rand -base64 32` |

**Reality X25519 密钥对**：

```bash
# Linux/macOS
bash scripts/GenRealityKeys.sh
# Windows PowerShell
powershell scripts/GenRealityKeys.ps1
```

Reality 配置需要 `private_key`（服务端）和对应的 `public_key`（客户端），通过 X25519 椭圆曲线 Diffie-Hellman 交换生成会话密钥。私钥为 Base64 编码的 32 字节原始数据。

### Reality short_id 配置

`short_ids` 是十六进制编码的认证标识符，最长 16 字节。安全建议：

- 至少配置一个非空的 short_id，避免空字符串（空字符串表示接受任意 short ID）
- 每个 short_id 应随机生成：`openssl rand -hex 8`（8 字节 = 16 字符）
- 定期轮换 short_id，废弃旧值

### 密钥轮换策略

| 凭据类型 | 轮换周期 | 说明 |
|----------|----------|------|
| 用户密码 | 30-90 天 | 按用户分配不同凭据，便于追踪和隔离 |
| SS2022 PSK | 60-90 天 | 轮换需同步更新所有客户端 |
| Reality 密钥对 | 90-180 天 | 轮换后需同步更新客户端配置中的 public_key |
| TLS 证书 | 按有效期 | Let's Encrypt 90 天自动续期 |
| Reality short_id | 60-90 天 | 可保留多个，渐进式轮换 |

### 密钥存储

密钥和配置文件必须严格保护：

```bash
# 配置文件权限
chmod 600 /etc/prism/configuration.json
chown prism:prism /etc/prism/configuration.json

# 私钥文件权限
chmod 600 /etc/prism/key.pem
chown prism:prism /etc/prism/key.pem
```

安全检查清单：

- 不要将配置文件提交到版本控制系统（`configuration.json`、`*.pem`、`*.key` 加入 `.gitignore`）
- 不要通过明文渠道（邮件、即时消息）传输密钥
- 使用密码管理器存储和分发凭据
- 为不同用户分配不同凭据，设置合理的 `max_connections`（建议 5-20）

## TLS 配置推荐

Prism 的 Worker TLS 模块（`worker::tls`）在初始化阶段为每个 Worker 创建 TLS 上下文。配置流程加载证书链和私钥，启用 GREASE 扩展增加 TLS 指纹随机性，并设置 ALPN 协议列表支持 HTTP/2 和 HTTP/1.1 协商。

### 证书选择

| 场景 | 推荐证书 | 说明 |
|------|----------|------|
| Reality 方案 | 无需自备证书 | 使用目标网站的真实证书，这是 Reality 的核心优势 |
| AnyTLS / TrustTunnel | Let's Encrypt 或商业证书 | 需要受信任的证书，Let's Encrypt 免费且可自动续期 |
| 测试/内网 | 自签名证书 | 仅用于开发测试，生产环境不推荐 |

### 证书部署与续期

```bash
# 申请 Let's Encrypt 证书
sudo certbot certonly --standalone -d your.server.com

# 配置自动续期
echo "0 3 * * * root certbot renew --quiet && systemctl restart prism" \
  | sudo tee /etc/cron.d/certbot-renew-prism
```

证书检查清单：

- 证书有效期充足（Let's Encrypt 建议在到期前 30 天续期）
- CN/SAN 包含服务的域名
- 私钥权限为 600
- 已配置自动续期
- 备份了证书和私钥

### TLS 版本与加密套件

Prism 基于 BoringSSL 构建，支持现代 TLS 参数。推荐配置：

- **最低版本**：TLS 1.2（建议 TLS 1.3）
- **加密套件优先级**：AES-256-GCM > ChaCha20-Poly1305 > AES-128-GCM
- **密钥交换**：ECDHE（X25519 优先）

ShadowTLS 的 `strict_mode` 默认为 `true`，仅接受 TLS 1.3 连接。生产环境应保持此设置。

## 伪装方案安全对比

Prism 支持五种 TLS 伪装方案，安全强度从高到低排列：

### Reality（推荐）

**安全等级：最高**

- 无需自备 TLS 证书，使用目标网站的真实证书完成握手
- 基于 X25519 密钥交换 + HKDF 派生会话密钥
- AEAD (AES-GCM / ChaCha20-Poly1305) 加密封装
- 被动探测时回退到真实目标网站， DPI 无法区分
- short_id 提供第一层过滤，X25519 认证提供第二层

配置要点：
- `dest` 选择高流量、稳定的知名网站（如 `www.microsoft.com:443`）
- `server_names` 精确匹配目标域名，不要使用通配符
- `private_key` 使用 X25519 密钥对，32 字节
- `short_ids` 避免使用空字符串

### ShadowTLS v3

**安全等级：高**

- 通过代理真实 TLS 握手来伪装流量指纹
- v3 版本在 SessionID 中嵌入 HMAC 验证，支持多用户
- `strict_mode` 仅允许 TLS 1.3
- 注意：传输阶段写入方向发送明文 payload（依赖外层 TLS 加密）

配置要点：
- 始终使用 v3 版本，不要使用 v2
- `strict_mode` 保持 `true`
- `handshake_dest` 选择稳定可靠的目标服务器

### Restls

**安全等级：中高**

- 通过模拟真实 TLS 流量特征来隐藏代理行为
- 支持 TLS 1.2 / TLS 1.3 版本模拟
- `restls_script` 控制流量长度分布，可自定义以匹配目标网站流量模式
- 认证信息嵌入 TLS 应用数据中

配置要点：
- `restls_script` 应针对目标网站调整，避免使用默认值
- `version_hint` 与目标网站实际 TLS 版本一致

### AnyTLS

**安全等级：中高**

- 使用标准 TLS 证书 + 应用层用户认证
- 内部支持多路复用（mux），单连接多流传输
- 可叠加 ECH 加密 ClientHello 中的 SNI
- `padding_scheme` 隐藏流量特征

配置要点：
- 需要自备 TLS 证书（Let's Encrypt 即可）
- 启用 `padding_scheme` 增强流量混淆
- 如有条件，配置 `ech_key` 加密 SNI

### TrustTunnel

**安全等级：中等**

- 基于 HTTP/2 CONNECT 代理 + Basic Auth 认证
- 支持 TCP（HTTP/2）和 UDP（HTTP/3 / QUIC）双栈传输
- 默认使用 BBR 拥塞控制算法
- 使用标准 TLS 证书

配置要点：
- 需要自备 TLS 证书
- `network` 选择 `both` 可同时支持 TCP 和 UDP
- 用户密码应足够复杂

### 方案选择建议

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 最高安全性需求 | Reality | 无需证书，被动探测无特征 |
| 兼容性优先 | ShadowTLS v3 | 广泛客户端支持 |
| 流量特征敏感 | Restls | 自定义 script 精细控制流量模式 |
| 需要多路复用 | AnyTLS | 内置 mux + 可选 ECH |
| 需要 UDP/QUIC | TrustTunnel | 原生 HTTP/3 支持 |

## 协议安全对比

Prism 支持的代理协议安全强度从高到低：

### SS2022（Shadowsocks 2022）— 推荐

- 基于 AEAD 的端到端加密，提供独立的密文保护层
- Prism 支持 AES-128-GCM、AES-256-GCM、ChaCha20-Poly1305、XChaCha20-Poly1305
- PSK 密钥认证 + 时间戳窗口防重放（30 秒窗口）
- 即使 TLS 被中间人攻破，SS2022 的 AEAD 加密仍然保护流量

推荐加密方法：`2022-blake3-aes-256-gcm`（256 位密钥，最高安全强度）。

### Trojan

- 密码认证 + TLS 加密传输
- 密码在 TLS 握手完成后以明文形式发送（依赖 TLS 保护）
- 安全性完全取决于 TLS 层的完整性
- 建议搭配 Reality 或 ShadowTLS 使用

### VLESS

- UUID 认证 + TLS 加密传输
- 不支持 XTLS/Vision flow（Prism 中 `addnl_info != 0` 直接拒绝）
- 安全性与 Trojan 类似，依赖 TLS 层保护

### SOCKS5

- 支持用户名/密码认证（`enable_auth: true`）
- 明文传输认证信息和数据（需依赖外层 TLS）
- 仅建议在可信网络或搭配 TLS 伪装方案使用

### HTTP

- 无认证机制
- 明文协议，最弱的安全性
- 仅用于开发测试或可信内网环境

### 协议安全等级总结

| 协议 | 认证方式 | 自加密 | 安全等级 | 生产建议 |
|------|----------|--------|----------|----------|
| SS2022 | PSK | AEAD | 最高 | 首选推荐 |
| Trojan | 密码 | 无（依赖 TLS） | 高 | 搭配伪装方案使用 |
| VLESS | UUID | 无（依赖 TLS） | 高 | 搭配伪装方案使用 |
| SOCKS5 | 用户名/密码 | 无 | 低 | 需配合 TLS |
| HTTP | 无 | 无 | 最低 | 仅测试环境 |

## 防火墙规则建议

### 最小化暴露面

仅开放代理服务监听的端口，拒绝其他所有入站连接：

```bash
# 开放代理端口（示例使用 443）
sudo ufw allow 443/tcp

# 如果使用 TrustTunnel 的 UDP 模式
sudo ufw allow 443/udp

# 拒绝其他所有入站
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

### 来源 IP 限制（可选）

如果客户端 IP 范围已知，可进一步限制来源：

```bash
# 仅允许特定网段访问
sudo ufw allow from 203.0.113.0/24 to any port 443
```

### 端口选择建议

| 端口 | 优势 | 劣势 |
|------|------|------|
| 443 | 伪装为 HTTPS 流量，DPI 难以识别 | 可能与现有 Web 服务冲突 |
| 8443 | 替代 HTTPS 端口，冲突概率低 | 不如 443 常见 |
| 自定义高端口 | 降低被扫描发现的概率 | 流量特征异常，容易被标记 |

## 日志安全

### 日志级别选择

| 环境 | 推荐 `log_level` | 说明 |
|------|-------------------|------|
| 生产环境 | `info` | 记录连接和错误信息，不输出调试细节 |
| 调试排查 | `debug` | 临时启用，排查完成后恢复 `info` |
| 安全审计 | `warn` | 仅记录警告和错误 |

### 禁止记录的内容

Prism 的日志系统不应记录以下敏感信息：

- 用户密码和 PSK 密钥
- TLS 私钥内容
- Reality X25519 私钥
- 完整的请求/响应 payload
- 客户端认证凭据的明文

配置文件权限必须限制为仅服务账户可读：

```bash
chmod 600 /etc/prism/configuration.json
```

### 日志轮转与保留

推荐配置：

- `max_size`：单个日志文件 64MB（默认值已合理）
- `max_files`：保留 7-8 个历史文件
- `enable_console`：生产环境设为 `false`，避免控制台输出泄露敏感信息
- 日志目录权限：`chmod 750 /var/log/prism/`

## DDoS 防护

### 连接限制

通过 `max_connections` 参数限制单用户的最大并发连接数：

```json
{
  "agent": {
    "authentication": {
      "users": [
        {
          "password": "strong-password",
          "uuid": "uuid-value",
          "max_connections": 10
        }
      ]
    }
  }
}
```

建议值 5-20，防止单用户耗尽服务器资源。

### 连接池保护

连接池配置可间接缓解资源耗尽攻击：

- `max_cache_per_endpoint`：每个目标最大缓存 32 连接（默认值已合理）
- `max_idle_seconds`：空闲连接 240 秒超时自动回收
- `connect_timeout_ms`：连接建立超时 300ms，快速释放失败连接

### Fail2ban 集成

监控认证失败日志，自动屏蔽恶意 IP：

```bash
# 创建过滤器
sudo tee /etc/fail2ban/filter.d/prism.conf > /dev/null << 'EOF'
[Definition]
failregex = ^.*authentication failed.*<HOST>.*$
            ^.*connection refused.*<HOST>.*$
EOF

# 配置 jail
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

### 异常流量监控

定期检查以下日志模式：

| 模式 | 含义 | 响应 |
|------|------|------|
| 大量认证失败 | 暴力破解 | 屏蔽来源 IP，检查凭据强度 |
| 异常高频连接 | 资源耗尽攻击 | 降低 `max_connections`，启用 Fail2ban |
| 大量 TLS 握手失败 | 主动探测/扫描 | 检查来源，防火墙屏蔽 |
| 未知 IP 大量连接 | 凭据泄露 | 立即更换凭据 |

## 运行权限

### 非 root 运行

```bash
# 创建专用系统用户
sudo useradd --system --no-create-home --shell /usr/sbin/nologin prism

# systemd 服务配置
[Service]
User=prism
Group=prism
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

使用 `AmbientCapabilities` 代替 root 运行，仅在需要绑定 1024 以下端口时授予 `CAP_NET_BIND_SERVICE` 能力。

## 应急响应

### 凭据泄露

1. 立即停止服务
2. 更换所有认证凭据（密码、UUID、PSK、Reality 密钥对）
3. 检查日志确认泄露期间是否有未授权访问
4. 从干净状态重新启动服务
5. 修复泄露原因（配置文件权限、版本控制排除等）

### 服务被入侵

1. 停止服务，保留日志用于取证
2. 检查系统完整性（`rpm -Va` / `debsums`）
3. 更换服务器 IP 或迁移到新服务器
4. 审计所有配置，确认无后门
5. 重新部署

## 相关文档

- [[docs/security|安全注意事项]] — 基础安全配置清单
- [[docs/configuration|配置说明]] — 完整配置字段说明
- [[docs/deployment|部署指南]] — 生产环境部署步骤
- [[docs/protocol-matrix|协议矩阵]] — 各协议功能对比
- [[docs/performance-tips|性能建议]] — 性能与安全的平衡
- [[docs/upgrade|升级指南]] — 安全更新与版本升级
