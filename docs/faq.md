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

**Q: 双击程序没反应？**
A: 以管理员身份运行命令行启动，或检查配置文件路径。

**Q: 如何查看错误信息？**
A: 命令行运行或查看 `trace.path_name` 指定的日志目录。

## 配置使用

**Q: 配置文件在哪里？**
A: `src/configuration.json`，修改后需重启程序。

**Q: 如何修改代理端口？**
A: 修改 `agent.addressable.port` 字段。

**Q: 如何启用调试日志？**
A: 设置 `trace.log_level = "debug"`。

## 网络连接

**Q: 浏览器提示无法连接代理服务器？**
A: 确认服务运行中，检查防火墙，测试 `telnet 127.0.0.1 端口`。

**Q: 其他电脑无法连接？**
A: 将 `host` 改为 `0.0.0.0`，检查防火墙和路由器。

**Q: 连接经常断开？**
A: 调整 `pool.max_idle_seconds`，增加 `max_cache_per_endpoint`。

## 性能安全

**Q: 使用代理后网速变慢？**
A: 优化连接池配置，调整 TCP 参数，检查网络状况。

**Q: 如何保护代理不被滥用？**
A: 使用防火墙限制 IP、配置认证、监控日志、定期更新。

## 相关页面

- [[docs/troubleshooting|故障排查]]
- [[docs/security|安全注意事项]]
- [[docs/configuration|配置说明]]