---
title: 已知问题与限制
created: 2026-05-21
updated: 2026-05-21
layer: docs
tags: [docs, known-issues, limitations]
---

# 已知问题与限制

本页列出 Prism 当前版本中已知的限制和尚未解决的问题，以及对应的规避方法。

## 严重问题

以下问题可能导致服务不稳定或中断，部署时请务必关注。

### 无优雅关闭

Prism 目前不支持优雅关闭（graceful shutdown）。终止进程时，所有活跃连接会被立即中断，无法等待正在进行的请求完成。

**规避方法**：使用 systemd 管理服务时，配置 `TimeoutStopSec` 与 `KillSignal`：

```ini
[Service]
TimeoutStopSec=10
KillSignal=SIGKILL
```

这样 systemd 会先发送 SIGKILL，在 10 秒后强制结束残留进程。请在业务低峰期进行重启操作。

### Session 无超时

异常或恶意的客户端连接建立后不会自动释放，Session 没有超时机制。长时间运行可能导致文件描述符（fd）持续增长，最终耗尽系统资源。

**规避方法**：

- 监控进程的 fd 使用量：`lsof -p <pid> | wc -l`
- 设置告警阈值，当 fd 数量接近 `ulimit` 上限时触发通知
- 定期重启服务（建议结合 systemd timer 或 cron）

### 连接池无总容量上限

连接池（connection pool）缺少全局容量上限配置。在高并发场景下，连接数可能持续增长，最终耗尽系统 fd 资源。

**规避方法**：

- 调整系统 fd 上限：`ulimit -n 65535`
- 持久化配置可在 `/etc/security/limits.conf` 中设置
- 监控 `/proc/sys/fs/file-nr` 观察系统级 fd 使用情况

---

## 协议限制

各协议实现中的已知限制和兼容性问题。

### SOCKS5 UDP 不支持 IPv6

IPv6 客户端通过 SOCKS5 发起 UDP ASSOCIATE 请求时无法正常工作。如果客户端使用 IPv6 地址进行 UDP 关联，数据包将无法正确转发。

**规避方法**：对于需要 UDP 的场景，请使用 IPv4 地址连接，或改用其他支持 IPv6 UDP 的代理协议。

### VLESS 不支持 XTLS/Vision

VLESS 协议当前不支持 XTLS 的 Vision flow。客户端如果配置了 `flow: xtls-rprx-vision`，连接将直接失败，服务端日志显示：

```
handshake failed: bad_message
```

**规避方法**：客户端配置中不要使用 Vision flow，将 flow 字段留空或删除。

### SS2022 时钟同步要求

Shadowsocks 2022 (SS2022) 协议对两端系统时钟有严格要求。如果客户端与服务端的时钟偏移超过 30 秒，认证将失败并返回：

```
timestamp_expired
```

**规避方法**：

- 确保所有节点运行 NTP 时间同步服务
- 定期检查时钟偏差：`ntpstat` 或 `chronyc tracking`
- 在容器化部署中，确保宿主机与容器的时钟一致

### Trojan 密码明文传输

Trojan 协议的密码以 SHA224 hex 格式在 TLS 隧道内层传输。虽然 TLS 提供了加密保护，但如果 TLS 配置存在问题（如使用了弱密码套件或证书验证被跳过），密码存在暴露风险。

**规避方法**：

- 确保 TLS 配置正确，使用强密码套件
- 开启证书验证，不要跳过 `skip-cert-verify`
- 定期更换密码

---

## 伪装方案限制

流量伪装（camouflage）相关功能的已知限制。

### Restls / AnyTLS / TrustTunnel 不生效

配置了 Restls、AnyTLS 或 TrustTunnel 伪装方案后，流量实际上不会经过伪装处理，而是直接走 native 兜底逻辑。这意味着伪装功能形同虚设，流量特征不会被隐藏。

**规避方法**：如需流量伪装，暂时使用其他已验证可用的伪装方案，或等待后续版本修复。

### ECH 解密未实现

虽然配置文件中支持 `ech_key` 字段，但 Encrypted Client Hello (ECH) 的解密逻辑尚未实现。配置后不会产生错误提示，但 ECH 功能实际不生效。

**规避方法**：暂时不依赖 ECH 功能，关注后续版本更新。

### ShadowTLS 写入方向不加密

ShadowTLS 在 TLS 握手完成后的传输阶段，服务端到客户端方向的数据没有加密保护，仅依赖外层（如 CDN 或前置代理）来保障数据安全。

**规避方法**：在 ShadowTLS 前层部署 TLS 终结（如 CDN 或 reverse proxy），确保整体链路有加密保护。

---

## 性能限制

### ShadowTLS HMAC 累积

ShadowTLS 的 HMAC 计算会随连接持续时间线性增长。长时间运行的连接会导致 CPU 开销逐渐上升，可能影响整体吞吐量。

**规避方法**：

- 设置连接最大持续时间，定期断开重建
- 监控 CPU 使用率，发现异常时及时重启服务

### DNS 负缓存 300 秒

DNS 查询失败后，负缓存（negative cache）默认保留 300 秒（5 分钟）。这意味着即使上游 DNS 已恢复正常，Prism 仍会在缓存期内返回失败结果。

**规避方法**：

- 减小 `negative_ttl` 配置值（如设为 60 秒）
- 在 DNS 临时故障后，手动清除缓存或重启服务

### 不支持配置热更新

修改配置文件后需要重启 Prism 才能生效，当前不支持配置热加载（hot reload）。重启会中断所有活跃连接。

**规避方法**：

- 在业务低峰期执行配置变更和重启
- 结合负载均衡，实现滚动重启以减少中断时间

---

## 规避建议速查表

| 问题 | 规避方法 |
|------|---------|
| 无优雅关闭 | systemd `TimeoutStopSec=10` + `KillSignal=SIGKILL` |
| Session 无超时 | 监控 `lsof -p <pid> \| wc -l`，设置告警并定期重启 |
| fd 耗尽 | `ulimit -n 65535`，监控 `/proc/sys/fs/file-nr` |
| SS2022 时间窗口 | 确保 NTP 同步持续运行 |
| DNS 负缓存过长 | 减小 `negative_ttl` 配置值 |
| Vision 不支持 | 客户端不使用 Vision flow，删除 flow 字段 |
| 伪装方案不生效 | 暂用其他可用伪装方案，关注版本更新 |
| ECH 未实现 | 不依赖 ECH，等待后续版本 |
| ShadowTLS 单向加密 | 前层部署 TLS 终结保护 |
| HMAC 累积 | 限制连接最大持续时间，监控 CPU |
| 配置无法热更新 | 低峰期重启或使用负载均衡滚动更新 |

---

## 相关页面

- [[docs/troubleshooting|故障排查]] — 遇到问题时的排查流程和日志分析方法
- [[docs/faq|常见问题]] — 高频使用问题的快速解答
- [[docs/performance-tips|性能优化]] — 提升 Prism 运行性能的建议
- [[docs/security|安全配置]] — 加固 Prism 安全性的配置指南
