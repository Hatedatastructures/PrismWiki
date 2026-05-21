---
title: 常见问题
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 常见问题

Prism 使用过程中的高频问题速答。

## 功能简介

收集安装运行、配置使用、网络连接、性能安全等常见问题及解答。

## 安装运行

### Q1: 双击程序没反应？

A: Prism 是命令行程序，需要在终端/命令行中启动。以管理员身份运行命令行，确保配置文件 `configuration.json` 在同目录下或通过 `--config` 指定路径。

### Q2: 如何查看错误信息？

A: 直接在命令行运行 Prism 可以看到实时输出。如果配置了 `trace.path_name`，日志文件也会写入指定目录。Windows 可添加 `trace.enable_console = true`。

### Q3: 编译时提示 C++23 特性不支持？

A: 需要 GCC >= 13 或 Clang >= 16。检查编译器版本：`gcc --version`。如果使用较旧的系统，可能需要安装更新的编译器（如通过 apt 安装 `gcc-13`）。

### Q4: CMake 找不到编译器？

A: 确认已安装 C++ 编译工具链：
- Ubuntu/Debian: `sudo apt install build-essential cmake`
- Windows: 安装 Visual Studio Build Tools 或 MinGW-w64

## 配置使用

### Q5: 配置文件在哪里？

A: 默认在程序同目录下的 `configuration.json`。可通过 `--config <path>` 自定义路径。建议生产环境使用 `/etc/prism/configuration.json`。

### Q6: 如何修改代理端口？

A: 修改 `agent.addressable.port` 字段。修改后需要重启服务生效。

### Q7: 如何启用调试日志？

A: 设置 `trace.log_level = "debug"` 或 `trace.log_level = "trace"`。生产环境建议使用 `info` 级别以避免日志过多影响性能。

### Q8: 修改配置后必须重启吗？

A: 是的，当前版本不支持配置热更新。修改配置后需要停止并重新启动服务。详见 [[docs/configuration|配置说明]]。

### Q9: 可以不配置 TLS 吗？

A: 可以不配置，但数据将以明文传输，不推荐在生产环境中使用。建议至少配置自签名证书用于测试。

## 网络连接

### Q10: 浏览器提示无法连接代理服务器？

A: 按以下步骤排查：
1. 确认 Prism 服务正在运行
2. 检查防火墙是否放行了对应端口
3. 测试端口连通性：`telnet 127.0.0.1 <端口>`
4. 确认代理配置中地址和端口正确

### Q11: 其他电脑无法连接？

A: 将 `agent.addressable.host` 改为 `0.0.0.0`（而非 `127.0.0.1`），检查服务器防火墙和路由器端口转发规则。

### Q12: 连接经常断开？

A: 调整连接池参数：
- 增加 `pool.max_idle_seconds`（默认 60 秒，可设为 120-300）
- 增加 `pool.max_cache_per_endpoint`（默认 32，可设为 64-128）
- 启用多路复用减少连接数

### Q13: DNS 解析失败？

A: 检查 `dns.servers` 配置是否包含可用的 DNS 服务器。推荐配置 `["8.8.8.8", "1.1.1.1", "9.9.9.9"]`。增加 `dns.timeout_ms` 应对网络延迟。

## 性能安全

### Q14: 使用代理后网速变慢？

A: 可能原因及解决方案：
- **连接池过小**: 增加 `pool.max_cache_per_endpoint`
- **缓冲区太小**: 增加 `pool.recv_buffer_size` 和 `pool.send_buffer_size` 至 128KB+
- **Nagle 算法**: 开启 `pool.tcp_nodelay = true`
- **多路复用**: 启用 Smux 减少连接开销
- 详见 [[docs/performance-tips|性能建议]]

### Q15: 如何保护代理不被滥用？

A: 多层防护：
1. **认证**: 配置 `agent.authentication` 强密码
2. **防火墙**: 限制来源 IP 范围
3. **限流**: 配置 `max_connections` 限制每个用户连接数
4. **监控**: 定期检查日志中的异常连接
5. **更新**: 定期更换密码和更新软件

### Q16: 内存占用过高怎么办？

A: 降低以下参数：
- `pool.max_cache_per_endpoint` 减少缓存连接数
- `dns.cache_size` 减少 DNS 缓存条目
- `multiplex.smux.buffer_size` 减少复用缓冲区大小
- `buffer.max_size` 限制最大缓冲区

### Q17: Prism 支持多少并发连接？

A: 取决于系统资源和配置。在 4 核 2GB 服务器上，典型配置下可支持 1000+ 并发连接。关键限制因素：
- 文件描述符限制（`ulimit -n`）
- 内存大小（每个连接约占用 64-256 KB）
- 网络带宽

### Q18: 日志文件太大怎么处理？

A: 在 `trace` 模块配置日志轮转参数：
```json
{
  "trace": {
    "max_size": 10485760,
    "max_files": 5
  }
}
```
或在 Linux 中配置 `logrotate`。详见 [[docs/deployment|部署指南]] 中的日志轮转章节。

## 协议相关

### Q19: 哪个协议最安全？

A: Reality + VLESS 或 ShadowTLS v3 + SS2022 提供最强的反检测能力。Trojan 适合基础场景。选择取决于你的安全需求和网络环境。

### Q20: 多个协议可以同时启用吗？

A: 可以。在 `protocol` 模块中将需要的协议设为 `enabled: true`，客户端分别配置对应节点即可。Prism 会自动识别并路由到正确的协议处理模块。

## 相关页面

- [[docs/troubleshooting|故障排查]]
- [[docs/security|安全注意事项]]
- [[docs/configuration|配置说明]]
- [[docs/performance-tips|性能建议]]
- [[docs/deployment|部署指南]]
