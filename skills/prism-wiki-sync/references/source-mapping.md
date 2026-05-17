---
title: Source Mapping Rules
created: 2026-05-17
updated: 2026-05-17
layer: skill
tags: [skill, sync, mapping]
---

# 源码映射规则

> Prism 源码目录与 Wiki 页面的映射规则。
> 最后更新：2026-05-17

---

## 基本映射

### 目录映射

| 源码目录 | Wiki 目录 | 层级 |
|----------|-----------|------|
| `I:/code/Prism/include/prism/<module>/` | `H:/PrismWiki/core/<module>/` | core |
| `I:/code/Prism/src/prism/<module>/` | `H:/PrismWiki/core/<module>/` | core |
| `I:/code/Prism/logs/mihomo-Meta/` | `H:/PrismWiki/ref/mihomo/` | ref |

### 文件映射

| 源码文件类型 | Wiki 页面类型 |
|--------------|---------------|
| `<module>.hpp` | `<module>/overview.md` |
| `<submodule>.hpp` | `<module>/<submodule>.md` |
| `<submodule>.cpp` | `<module>/<submodule>.md`（实现部分）|

---

## 模块映射表

### Core 模块

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `include/prism/agent/` | `core/agent/` |
| `include/prism/channel/` | `core/channel/` |
| `include/prism/pipeline/` | `core/pipeline/` |
| `include/prism/protocol/` | `core/protocol/` |
| `include/prism/stealth/` | `core/stealth/` |
| `include/prism/recognition/` | `core/recognition/` |
| `include/prism/resolve/` | `core/resolve/` |
| `include/prism/multiplex/` | `core/multiplex/` |
| `include/prism/crypto/` | `core/crypto/` |
| `include/prism/outbound/` | `core/outbound/` |
| `include/prism/memory/` | `core/memory/` |
| `include/prism/fault/` | `core/fault/` |
| `include/prism/exception/` | `core/exception/` |
| `include/prism/trace/` | `core/trace/` |
| `include/prism/transformer/` | `core/transformer/` |
| `include/prism/loader/` | `core/loader/` |

### Stealth 子模块

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `include/prism/stealth/reality/` | `core/stealth/reality/` |
| `include/prism/stealth/shadowtls/` | `core/stealth/shadowtls/` |
| `include/prism/stealth/restls/` | `core/stealth/restls/` |
| `include/prism/stealth/anytls/` | `core/stealth/anytls/` |
| `include/prism/stealth/trusttunnel/` | `core/stealth/trusttunnel/` |
| `include/prism/stealth/ech/` | `core/stealth/ech/` |

### Protocol 子模块

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `include/prism/protocol/common/` | `core/protocol/common/` |
| `include/prism/protocol/http/` | `core/protocol/http/` |
| `include/prism/protocol/socks5/` | `core/protocol/socks5/` |
| `include/prism/protocol/trojan/` | `core/protocol/trojan/` |
| `include/prism/protocol/vless/` | `core/protocol/vless/` |
| `include/prism/protocol/shadowsocks/` | `core/protocol/shadowsocks/` |

### Multiplex 子模块

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `include/prism/multiplex/core/` | `core/multiplex/` |
| `include/prism/multiplex/smux/` | `core/multiplex/smux/` |
| `include/prism/multiplex/yamux/` | `core/multiplex/yamux/` |

---

## Mihomo 映射

### 协议映射

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `mihomo-Meta/adapter/outbound/socks.go` | `ref/mihomo/protocols/socks5.md` |
| `mihomo-Meta/adapter/outbound/http.go` | `ref/mihomo/protocols/http.md` |
| `mihomo-Meta/adapter/outbound/trojan.go` | `ref/mihomo/protocols/trojan.md` |
| `mihomo-Meta/adapter/outbound/vless.go` | `ref/mihomo/protocols/vless.md` |
| `mihomo-Meta/adapter/outbound/vmess.go` | `ref/mihomo/protocols/vmess.md` |
| `mihomo-Meta/adapter/outbound/shadowsocks.go` | `ref/mihomo/protocols/shadowsocks.md` |
| `mihomo-Meta/adapter/outbound/hysteria2.go` | `ref/mihomo/protocols/hysteria2.md` |
| `mihomo-Meta/adapter/outbound/tuic.go` | `ref/mihomo/protocols/tuic.md` |
| `mihomo-Meta/adapter/outbound/wireguard.go` | `ref/mihomo/protocols/wireguard.md` |
| ... | ... |

### 组件映射

| 源码路径 | Wiki 路径 |
|----------|-----------|
| `mihomo-Meta/component/dialer/` | `ref/mihomo/implementation/dialer.md` |
| `mihomo-Meta/component/resolver/` | `ref/mihomo/dns/` |
| `mihomo-Meta/rules/` | `ref/mihomo/rules/` |
| `mihomo-Meta/proxygroups/` | `ref/mihomo/proxy-groups/` |

---

## 标注格式

### core/ 层页面标注

```markdown
> 源码: include/prism/module/file.hpp:XX | 实现: src/prism/module/file.cpp:XX
```

### 函数标注

```markdown
**调用（向下）**: `callee()` — 源码: include/prism/module/file.hpp:45
**被调用（向上）**: [[caller/module|caller]] 的 `caller_func()`
```

---

## 检查脚本示例

```bash
# 检查源码路径存在性
for page in core/**/*.md; do
    source=$(grep "源码:" "$page" | sed 's/源码: //')
    if [ -n "$source" ]; then
        if [ ! -f "I:/code/Prism/$source" ]; then
            echo "FAIL: $page → $source (not found)"
        fi
    fi
done

# 检查 wikilink 有效性
for page in **/*.md; do
    links=$(grep -o '\[\[[^]]*\]' "$page" | sed 's/\[\[\([^|]*\).*/\1/')
    for link in $links; do
        target="H:/PrismWiki/$link.md"
        if [ ! -f "$target" ]; then
            echo "WARN: $page → $link (wikilink broken)"
        fi
    done
done
```

---

## 相关参考

- [[SKILL|Skill 主页]]
- [[checks|检查项清单]]