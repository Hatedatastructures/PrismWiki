---
name: prism-wiki-sync
description: PrismWiki 知识库同步检查 Skill
---

# prism-wiki-sync Skill

> 检查 PrismWiki 知识库与 Prism 源码的同步状态。
> 运行时机：每次 git push（普通检查）和发布前（深度检查）。

---

## 触发条件

### 普通检查（Push）

每次 `git push` 时自动运行：

```bash
# pre-push hook
git push → prism-wiki-sync --mode=quick
```

### 深度检查（Release）

发布版本前运行：

```bash
# 手动触发
prism-wiki-sync --mode=deep
```

---

## 检查模式

| 模式 | 检查内容 | 耗时 |
|------|----------|------|
| quick | 源码路径存在性、wikilink 有效 | < 1 min |
| deep | + 实现一致性、接口匹配、版本对齐 | 5-10 min |

---

## 检查项

详见 [[checks|检查项清单]]。

---

## 输出

详见 [[report-template|报告模板]]。

---

## 使用方式

### CLI 调用

```bash
# 普通检查
prism-wiki-sync

# 深度检查
prism-wiki-sync --deep

# 只检查某模块
prism-wiki-sync --module=stealth
```

### Claude Code 调用

```text
/run prism-wiki-sync
```

---

## 相关参考

- [[SCHEMA|知识库规范]]
- [[checks|检查项清单]]
- [[report-template|报告模板]]
- [[references/source-mapping|源码映射规则]]