---
layer: dev
---

# Bug 报告模板

提交 Prism Bug 报告时，请按以下模板填写，确保提供足够信息以便快速定位和修复问题。

## Bug 报告模板

```markdown
## Bug 标题

[Bug 简要描述]

## 环境信息

| 项目 | 信息 |
|------|------|
| Prism 版本 | [如 v1.2.0 或 commit hash] |
| 操作系统 | [Windows 11 / Ubuntu 22.04 / macOS 14] |
| 编译器 | [GCC 15.2 / Clang 18 / MSVC 2022] |
| 构建类型 | [Release / Debug] |
| AES-NI | [已启用 / 未启用] |

## Bug 描述

[详细描述 Bug 的现象和影响]

## 复现步骤

1. [步骤 1]
2. [步骤 2]
3. [步骤 3]
...

## 期望行为

[描述正确行为应该是什么]

## 实际行为

[描述实际发生了什么]

## 日志输出

```
[粘贴相关日志，包含错误信息]
```

## 配置文件

```json
{
  [粘贴相关配置片段]
}
```

## 复现代码（如有）

```cpp
[粘贴最小复现代码]
```

## 附加信息

- [是否影响性能]
- [是否影响稳定性]
- [是否有 Crash]
- [相关 Issue 或 PR]

## 检查清单

- [ ] 已搜索现有 Issue，确认非重复报告
- [ ] 已尝试最新版本，确认 Bug 仍存在
- [ ] 已提供完整复现步骤
- [ ] 已提供相关日志和配置
```

---

## Bug 分类

### 性能问题

- 吞吞吐量异常（低于预期）
- 延迟抖动（P99 异常）
- 内存泄漏（RSS 持续增长）
- CPU 占用异常

**附加信息要求**:
- 运行 `CryptoBench.exe` / `MemoryBench.exe` / `IOBench.exe` 基准输出
- 性能对比数据（与预期或历史版本）

### 连接问题

- 连接建立失败
- 连接中断
- 认证失败
- 协议解析错误

**附加信息要求**:
- 客户端类型和版本
- 服务端配置
- 网络环境（ISP、防火墙）

### Crash 问题

- 进程崩溃
- 断言失败
- 异常未捕获

**附加信息要求**:
- Crash 日志或 stack trace
- Windows: Event Viewer 日志
- Linux: core dump 或 dmesg

### 兼容性问题

- 客户端不兼容
- 配置解析错误
- 协议实现偏差

**附加信息要求**:
- 不兼容的客户端名称和版本
- 对比标准协议规范

---

## 常见问题快速排查

### AES-NI 未启用

**症状**: AES-256-GCM 吞吐 <500 Mi/s

**排查**:
```bash
# 运行加密基准
build_release/benchmarks/CryptoBench.exe

# 检查输出，AES-256-GCM 应 >10 Gi/s
# 若 <500 Mi/s，说明 AES-NI 未启用
```

**解决方案**: 
- 检查 CPU 是否支持 AES-NI
- 检查编译选项是否启用 AES-NI
- BoringSSL MinGW patch 是否正确应用

### 内存池性能退化

**症状**: 多线程下内存分配性能骤降

**排查**:
```bash
# 运行内存基准
build_release/benchmarks/MemoryBench.exe

# 检查 Global Pool 4T 是否 >3000 ns
# 正常应 <150 ns
```

**解决方案**: 使用分片池 (`sharded_pool`) 替代 `synchronized_pool`

### 连接延迟 P99 抖动

**症状**: P99 延迟 >300 us，P50 正常

**排查**:
```bash
# 运行延迟基准
build_release/benchmarks/LatencyBench.exe

# 检查 P99/P50 比值
```

**解决方案**: 
- 快速路径跳过健康检查
- 后台预创建连接

---

## 提交方式

- GitHub Issue: [项目 Issue 页面]
- 邮件: [维护者邮箱]
- IRC/Discord: [社区频道]

---

## 相关页面

- [[performance/tuning]] — 性能调优指南
- [[performance/profiling]] — 性能分析方法
- [[testing]] — 测试方法