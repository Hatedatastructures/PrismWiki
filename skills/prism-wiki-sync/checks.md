---
title: Wiki Sync Checks
created: 2026-05-17
updated: 2026-05-17
layer: skill
tags: [skill, sync, check]
---

# 检查项清单

> prism-wiki-sync 的具体检查项。
> 最后更新：2026-05-17

---

## Quick 检查（Push）

### 1. 源码路径存在性

**检查**: core/ 层页面标注的源码路径是否存在

```yaml
# 示例
core/stealth/reality/handshake.md:
  源码标注: include/prism/stealth/reality/handshake.hpp
  检查: ls I:/code/Prism/include/prism/stealth/reality/handshake.hpp
  期望: 文件存在
```

**失败处理**: 
- 源码已删除 → 删除对应页面或更新标注
- 源码路径变更 → 更新标注

### 2. wikilink 有效性

**检查**: 页面中的 wikilink 目标是否存在

```yaml
# 示例
[[core/stealth/overview|Stealth 模块]]
  检查: ls H:/PrismWiki/core/stealth/overview.md
  期望: 文件存在
```

**失败处理**:
- 目标不存在 → 创建页面或修正链接
- 循环链接 → 警告

### 3. Frontmatter 完整性

**检查**: 每个页面是否包含必要 frontmatter

```yaml
必要字段:
  - title
  - layer (core|dev|docs|ref)
  
core 层额外:
  - source (源码路径)
```

**失败处理**: 补充缺失字段

### 4. 目录结构合规

**检查**: 文件是否在正确层级目录

```yaml
core/: 详细实现文档
dev/: 开发排障文档  
docs/: 用户指南
ref/: 参考知识
skills/: Skill 定义
```

**失败处理**: 移动到正确目录

---

## Deep 检查（Release）

### 5. 函数签名一致性

**检查**: 页面描述的函数签名与源码是否匹配

```yaml
# 示例
页面描述:
  函数: handshake_reality()
  参数: (socket, config)
  返回: awaitable<void>

源码检查:
  grep -A 5 "awaitable<void> handshake_reality" I:/code/Prism/include/prism/stealth/reality/handshake.hpp
  对比签名
```

**失败处理**: 更新页面描述

### 6. 常量值一致性

**检查**: 页面描述的常量值与源码是否匹配

```yaml
# 示例
页面:
  SOCKS5_VERSION = 0x05

源码检查:
  grep "SOCKS5_VERSION" I:/code/Prism/include/prism/protocol/socks5.hpp
  对比值
```

**失败处理**: 更新页面值

### 7. 调用链完整性

**检查**: 页面标注的调用链是否可验证

```yaml
# 示例
页面标注:
  调用 [[core/stealth/reality/handshake|handshake]]

验证:
  grep "handshake" I:/code/Prism/src/prism/stealth/reality/*.cpp
  检查是否实际调用
```

**失败处理**: 更新或删除标注

### 8. 模块覆盖完整性

**检查**: 所有源码模块是否有对应页面

```yaml
# 扫描源码目录
ls I:/code/Prism/include/prism/
  
# 对应检查
对于每个模块目录:
  ls H:/PrismWiki/core/<module>/overview.md
```

**失败处理**: 创建缺失页面

### 9. 版本对齐

**检查**: 知识库版本与源码版本是否对齐

```yaml
源码版本:
  git describe --tags I:/code/Prism

知识库版本:
  grep "version" H:/PrismWiki/log.md
  
对比:
  知识库应反映源码最新版本
```

**失败处理**: 更新知识库内容

### 10. mihomo 协议覆盖

**检查**: mihomo 所有协议是否有对应页面

```yaml
# 扫描 mihomo 源码
ls I:/code/Prism/logs/mihomo-Meta/adapter/outbound/

# 对应检查
对于每个协议文件:
  ls H:/PrismWiki/ref/mihomo/protocols/<protocol>.md
```

**失败处理**: 创建缺失页面

---

## 检查优先级

| 级别 | 检查项 | 错误时 |
|------|--------|--------|
| P0 | 源码路径存在性 | 阻止 push |
| P0 | wikilink 有效性 | 阻止 push |
| P1 | Frontmatter 完整性 | 警告 |
| P1 | 目录结构合规 | 警告 |
| P2 | 函数签名一致性 | 警告（deep）|
| P2 | 常量值一致性 | 警告（deep）|
| P2 | 调用链完整性 | 警告（deep）|
| P3 | 模块覆盖完整性 | 信息（deep）|
| P3 | 版本对齐 | 信息（deep）|
| P3 | mihomo 协议覆盖 | 信息（deep）|

---

## 相关参考

- [[SKILL|Skill 主页]]
- [[report-template|报告模板]]
- [[references/source-mapping|源码映射规则]]