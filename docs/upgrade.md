---
title: 升级指南
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 升级指南

Prism 版本升级步骤和注意事项。

## 功能简介

提供版本升级、配置迁移、兼容性检查、回滚策略的完整指导流程。

## 升级前准备

### 1. 备份

升级前务必备份以下内容：

```bash
# Linux
sudo cp /etc/prism/configuration.json /etc/prism/configuration.json.bak.$(date +%Y%m%d)
sudo cp -r /var/log/prism /tmp/prism-logs-backup
sudo cp /usr/local/bin/prism /usr/local/bin/prism.bak.$(date +%Y%m%d)

# Windows PowerShell
Copy-Item C:\Prism\configuration.json "C:\Prism\configuration.json.bak.$(Get-Date -Format yyyyMMdd)"
Copy-Item C:\Prism\Prism.exe "C:\Prism\Prism.exe.bak.$(Get-Date -Format yyyyMMdd)"
```

### 2. 查看变更日志

在 [GitHub Releases](https://github.com/Hatedatastructures/Prism/releases) 查看新版本的变更日志，重点关注：

- **BREAKING CHANGES**: 破坏性变更
- **New Features**: 新增功能和配置项
- **Bug Fixes**: 修复的问题
- **Performance**: 性能改进

### 3. 兼容性检查

| 升级跨度 | 风险等级 | 建议操作 |
|----------|----------|----------|
| 补丁版本（1.0.0 → 1.0.1） | 低 | 直接升级，无需修改配置 |
| 次版本（1.0.x → 1.1.x） | 中 | 检查新增配置项，对比默认值 |
| 主版本（1.x.x → 2.x.x） | 高 | 完整阅读迁移指南，逐项检查配置 |

## 升级步骤

### Linux systemd

```bash
# 1. 停止服务
sudo systemctl stop prism

# 2. 编译新版本（或下载预编译二进制）
cd Prism && git pull
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 3. 替换二进制
sudo cp build_release/src/Prism /usr/local/bin/prism

# 4. 检查配置兼容性
# 对比新旧配置格式（见下方配置迁移章节）

# 5. 启动服务
sudo systemctl start prism

# 6. 验证
sudo systemctl status prism
sudo journalctl -u prism -n 50
```

### Docker

```bash
# 1. 拉取最新代码
git pull

# 2. 重建镜像
docker compose build

# 3. 重启容器
docker compose down
docker compose up -d

# 4. 验证
docker compose logs -f prism
```

### Windows

```powershell
# 1. 停止服务
sc stop Prism

# 2. 替换二进制
# 将新的 Prism.exe 复制到 C:\Prism\ 覆盖旧文件

# 3. 启动服务
sc start Prism

# 4. 验证
sc query Prism
Get-Content C:\Prism\logs\prism.log -Tail 20
```

## 配置迁移

### 检查配置变更

新版本可能涉及以下配置变化：

1. **新增配置项**: 如果新版本新增了 `buffer`、`stealth` 等模块，需要在配置中添加对应字段
2. **字段重命名**: 检查是否有字段名称变更（如 `max_cache` → `max_cache_per_endpoint`）
3. **默认值变更**: 某些参数的默认值可能调整，需确认是否影响你的场景
4. **废弃字段**: 旧版本支持的字段可能在新版本中移除

### 迁移步骤

```bash
# 1. 使用 diff 对比新旧配置
diff /etc/prism/configuration.json.bak /etc/prism/configuration.json

# 2. 在新版本源码中找到最新默认配置
cat src/configuration.json  # 或查看 Release 附件中的示例配置

# 3. 逐步迁移
# - 保留旧配置中已确认仍有效的核心参数
# - 补充新增字段，使用默认值或根据需求调整
# - 移除已废弃字段
```

### 示例：添加新模块

如果新版本新增了 `buffer` 模块：

```json
{
  "agent": { /* 保留原有配置 */ },
  "pool": { /* 保留原有配置 */ },
  "buffer": {
    "default_size": 16384,
    "max_size": 1048576
  },
  "protocol": { /* 保留原有配置 */ },
  "multiplex": { /* 保留原有配置 */ },
  "stealth": { /* 保留原有配置 */ },
  "dns": { /* 保留原有配置 */ },
  "trace": { /* 保留原有配置 */ }
}
```

## 升级后验证

### 基础验证

1. **服务状态**: `systemctl status prism` / `sc query Prism`
2. **端口监听**: `netstat -tlnp | grep 8081`
3. **日志检查**: 查看是否有 `error` 或 `warn` 级别的日志
4. **代理测试**: `curl -x http://127.0.0.1:8081 http://example.com`

### 功能验证

- [ ] 所有启用的协议工作正常
- [ ] TLS 连接正常
- [ ] 认证功能正常
- [ ] 多路复用工作正常（如启用）
- [ ] DNS 解析正常
- [ ] 连接池行为符合预期

### 性能验证

```bash
# 对比升级前后的性能指标
# 连接延迟
time curl -x http://127.0.0.1:8081 http://example.com

# 内存占用
ps aux | grep prism  # Linux
Get-Process Prism    # Windows

# 并发能力
# 使用 ab 或 wrk 进行压力测试
```

## 回滚

如果升级后出现问题，可以快速回滚：

### Linux

```bash
sudo systemctl stop prism
sudo cp /usr/local/bin/prism.bak.$(date +%Y%m%d) /usr/local/bin/prism
sudo cp /etc/prism/configuration.json.bak.$(date +%Y%m%d) /etc/prism/configuration.json
sudo systemctl start prism
```

### Docker

```bash
# 使用旧版本镜像标签
docker compose down
git checkout v1.0.0  # 或旧版本 tag
docker compose up -d
```

### Windows

```powershell
sc stop Prism
Copy-Item C:\Prism\Prism.exe.bak.$(Get-Date -Format yyyyMMdd) C:\Prism\Prism.exe -Force
Copy-Item C:\Prism\configuration.json.bak.$(Get-Date -Format yyyyMMdd) C:\Prism\configuration.json -Force
sc start Prism
```

## 常见问题

**Q: 升级后配置不兼容怎么办？**
A: 对比新旧配置格式，调整参数名称或结构。参考上方"配置迁移"章节的迁移步骤。如果不确定，可以先用新版本的默认配置文件启动，再将旧配置中的参数逐个添加回去。

**Q: 升级后连接失败？**
A: 检查协议参数是否变化，确保客户端配置同步更新。特别是 TLS 证书路径、认证密码/UUID、Reality 公钥等关键参数。

**Q: 如何回滚？**
A: 保留旧版本二进制和配置备份，必要时恢复。详见上方"回滚"章节。

**Q: 升级后性能变差了？**
A: 检查新版本是否有默认值变更。某些优化可能改变了默认配置，需要根据你的场景重新调整 `pool`、`buffer`、`multiplex` 等模块的参数。

## 相关页面

- [[docs/deployment|部署指南]]
- [[docs/troubleshooting|故障排查]]
- [[dev/modules|模块架构]]
- [[docs/configuration|配置说明]]
