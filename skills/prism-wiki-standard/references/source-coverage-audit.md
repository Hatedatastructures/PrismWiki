---
title: source-coverage-audit
created: 2026-05-18
updated: 2026-05-18
layer: skills
tags: [skill, reference]
---

# PrismWiki 源码覆盖率审计脚本

对比 wiki 页面与 hpp 文件的覆盖情况，生成优先级报告。

## 环境变量

```bash
WIKI="H:/PrismWiki"
SRC="/i/code/Prism/include/prism"
EXCLUDE="-not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*'"
```

## Phase 1: 文件覆盖对比

```bash
# 列出所有 hpp（去掉 include/prism/ 前缀和 .hpp 后缀）
find "$SRC" -name "*.hpp" | sed "s|${SRC}/||;s|\\.hpp$||" | sort > /tmp/hpp_list.txt

# 列出所有 wiki API 文档（排除 ref/dev/client/docs/performance）
find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" \
  | sed "s|${WIKI}/||;s|\\.md$||" | sort > /tmp/wiki_list.txt

# 找没有 wiki 页面的 hpp
while IFS= read -r hpp; do
  grep -qx "$hpp" /tmp/wiki_list.txt && continue
  dir=$(dirname "$hpp")
  [ "$dir" != "." ] && grep -qx "${dir}/${dir##*/}" /tmp/wiki_list.txt && continue
  echo "MISSING: $hpp"
done < /tmp/hpp_list.txt
```

## Phase 2: 质量分布

```bash
find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" \
  -print0 | xargs -0 wc -l | grep -v total | awk '{
    if($1<50)thin++
    else if($1<150)med++
    else rich++
  }END{
    print "B档 (<50行):  ", thin
    print "A档 (50-150): ", med
    print "S档 (>150):   ", rich
    print "总计:         ", thin+med+rich
  }'
```

## Phase 3: 描述准确性

```bash
# 对比 wiki 概述 vs hpp @brief
for f in $(find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" -print0 | xargs -0 wc -l | sort -n | awk '$1>0 && $1<60{print $2}'); do
  src=$(head -15 "$f" | grep -m1 '^source:' | sed 's/^source: *"*//;s/"*$//')
  [ -z "$src" ] && continue
  hpp="${SRC}/../${src}"
  [ ! -f "$hpp" ] && continue
  hpp_brief=$(grep -m1 '@brief' "$hpp" | sed 's/.*@brief *//')
  wiki_title=$(head -5 "$f" | grep -m1 '^title:' | sed 's/^title: *"*//;s/"*$//')
  rel=$(echo "$f" | sed "s|${WIKI}/||;s|\\.md$||")
  [ -n "$hpp_brief" ] && echo "  $rel  wiki=\"$wiki_title\"  hpp=\"$hpp_brief\""
done
```

## Phase 4: Module 字段校验

```bash
find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" -print0 | while IFS= read -r -d '' f; do
  rel=$(echo "$f" | sed "s|${WIKI}/||;s|\\.md$||")
  wiki_top=$(echo "$rel" | cut -d'/' -f1)
  mod=$(head -15 "$f" | grep -m1 '^module:' | sed 's/^module: *"*//;s/"*$//')
  [ -n "$mod" ] && [ "$mod" != "$wiki_top" ] && echo "MISMATCH: $rel module=$mod (应为 $wiki_top)"
done
```

## Phase 5: Wiki-to-HPP 行数比

```bash
for f in $(find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" -print0 | xargs -0 wc -l | sort -n | awk '$1>0 && $1<60{print $2}'); do
  src=$(head -15 "$f" | grep -m1 '^source:' | sed 's/^source: *"*//;s/"*$//')
  [ -z "$src" ] && continue
  hpp="${SRC}/../${src}"
  [ ! -f "$hpp" ] && continue
  wiki_lines=$(wc -l < "$f")
  hpp_lines=$(wc -l < "$hpp")
  ratio=$((wiki_lines * 100 / hpp_lines))
  echo "STUB: $(echo $f|sed "s|${WIKI}/||;s|\\.md$||")  wiki=${wiki_lines}L hpp=${hpp_lines}L ratio=${ratio}%"
done | sort -t= -k4 -n
```

## Phase 6: 段落完整性

```bash
# 检查 API 文档缺少必有段落的情况
for f in $(find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*"); do
  type=$(head -15 "$f" | grep -m1 '^type:' | sed 's/^type: *//')
  [ "$type" != "api" ] && continue
  rel=$(echo "$f" | sed "s|${WIKI}/||;s|\\.md$||")
  lines=$(wc -l < "$f")
  issues=""
  grep -q '## 概述' "$f" || issues="${issues} 概述"
  grep -q '## 命名空间' "$f" || issues="${issues} 命名空间"
  grep -q '## 依赖关系' "$f" || issues="${issues} 依赖关系"
  grep -qE '## 类与结构体|## 类型别名' "$f" || issues="${issues} 类/结构体"
  grep -qE '## 函数签名表|## 函数:' "$f" || issues="${issues} 函数签名表"
  [ -n "$issues" ] && echo "MISSING($lines行): $rel → 缺少:${issues}"
done
```

## Phase 7: 链接健康

```bash
# 找死链
grep -roh '\[\[[^]]*\]\]' "$WIKI" --include="*.md" | \
  sed 's/\[\[//;s/\]\]//' | sed 's/|.*//' | sed 's/#.*//' | \
  sort -u | while read target; do
    [ ! -f "${WIKI}/${target}.md" ] && echo "DEAD: $target"
  done

# 找孤儿页（不在 index.md 中出现的页面）
index_links=$(grep -oP '\[\[\K[^|\]]+' "$WIKI/index.md" | sed 's/#.*//' | sort -u)
find "$WIKI" -name "*.md" \
  -not -path "*/.git/*" -not -path "*/.obsidian/*" \
  -not -name "index.md" -not -name "SCHEMA.md" \
  -not -name "log.md" -not -name "README.md" \
  -not -path "*/.plan/*" -not -path "*/skills/*" \
  -not -path "*/ref/*" -not -path "*/dev/*" \
  -not -path "*/client/*" -not -path "*/docs/*" \
  -not -path "*/performance/*" | while read f; do
  rel=$(echo "$f" | sed "s|${WIKI}/||;s|\\.md$||")
  echo "$index_links" | grep -qx "$rel" && continue
  echo "ORPHAN: $rel"
done
```

## 报告模板

```
================================================================
       PrismWiki 覆盖率审计报告 (YYYY-MM-DD)
================================================================

## 总体统计
  项目 hpp 文件数:  N
  Wiki API 页面数:  M
  Wiki 总页面数:    T
  总行数:           L

## 质量分布
  B档 (<50行):    X (Y%)
  A档 (50-150行):  X (Y%)
  S档 (>150行):    X (Y%)

## P0 — 描述错误（误导读者）
  - module/file.md: wiki 说 "X" 但 hpp 说 "Y"

## P1 — 空壳页面（需要重写）
  - module/file.md: 23行 vs hpp 176行 (13%)

## P2 — 字段问题
  - module/file.md: module 字段 "X::Y" 应为 "X"

## P3 — 缺失覆盖
  - module/header: 无 wiki 页面

## 缺失段落
  - module/file.md: 缺少 概述 命名空间 函数签名表

## 死链
  - [[dead/link]] — 目标文件不存在

## 孤儿页
  - module/orphan.md — 未被 index.md 链接

## 建议
  优先级: P0 → P1 → P2 → P3
```
