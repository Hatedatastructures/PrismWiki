---
title: Wiki Sync Report Template
created: 2026-05-17
updated: 2026-05-17
layer: skill
tags: [skill, sync, report]
---

# 报告模板

> prism-wiki-sync 检查结果输出格式。
> 最后更新：2026-05-17

---

## 报告结构

```markdown
# PrismWiki 同步检查报告

> 检查时间: YYYY-MM-DD HH:MM:SS
> 检查模式: quick | deep
> 源码版本: vX.Y.Z
> 知识库版本: last-update-YYYY-MM-DD

---

## 检查结果摘要

| 检查项 | 状态 | 数量 |
|--------|------|------|
| 源码路径存在性 | ✅ PASS | 120/120 |
| wikilink 有效性 | ✅ PASS | 450/450 |
| Frontmatter 完整性 | ⚠️ WARN | 3 缺失 |
| 目录结构合规 | ✅ PASS | — |
| ... | ... | ... |

**总体状态**: ✅ PASS | ⚠️ WARN | ❌ FAIL

---

## 详细问题列表

### P0 问题（阻止 push）

> 无

### P1 问题（警告）

#### Frontmatter 缺失

| 文件 | 缺失字段 | 建议 |
|------|----------|------|
| core/memory/pool.md | source | 补充: include/prism/memory/pool.hpp |
| ref/crypto/sha256.md | layer | 补充: ref |

### P2 问题（deep 检查）

> （deep 模式才显示）

#### 函数签名不一致

| 页面 | 页面描述 | 源码实际 | 建议 |
|------|----------|----------|------|
| core/stealth/reality/handshake.md | handshake(socket, config) | handshake(socket, config, timeout) | 更新签名 |

#### 常量值不一致

| 页面 | 页面值 | 源码值 | 建议 |
|------|--------|--------|------|
| core/protocol/socks5.md | SOCKS5_CONNECT = 0x01 | SOCKS5_CMD_CONNECT = 0x01 | 更新名称 |

### P3 问题（覆盖检查）

#### 缺失模块页面

| 源码模块 | 缺失页面 | 建议 |
|----------|----------|------|
| I:/code/Prism/include/prism/new_module/ | core/new_module/overview.md | 创建页面 |

---

## 统计

| 类别 | 总数 | 检查数 | 通过数 | 失败数 |
|------|------|--------|--------|--------|
| core 页面 | 120 | 120 | 118 | 2 |
| dev 页面 | 45 | 45 | 45 | 0 |
| docs 页面 | 15 | 15 | 15 | 0 |
| ref 页面 | 180 | 180 | 178 | 2 |
| wikilinks | 450 | 450 | 450 | 0 |

---

## 建议操作

```bash
# 修复 Frontmatter
echo "source: include/prism/memory/pool.hpp" >> core/memory/pool.md

# 更新常量名
sed -i 's/SOCKS5_CONNECT/SOCKS5_CMD_CONNECT/' core/protocol/socks5.md

# 创建缺失页面
touch core/new_module/overview.md
```

---

## 下次检查建议

- 源码版本变更后运行 deep 检查
- 新增模块后检查覆盖完整性
```

---

## 状态码

| 状态 | 符号 | 说明 |
|------|------|------|
| PASS | ✅ | 检查通过 |
| WARN | ⚠️ | 有警告但不阻止 |
| FAIL | ❌ | 检查失败，阻止 push |
| SKIP | ⊘ | 未检查（如 deep 模式未运行）|

---

## 输出位置

- Push 检查: 输出到 stdout，阻止 push 时返回非零
- Deep 检查: 生成 `H:/PrismWiki/.sync-report.md`

---

## 相关参考

- [[SKILL|Skill 主页]]
- [[checks|检查项清单]]