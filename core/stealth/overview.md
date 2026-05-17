---
layer: core
source: include/prism/stealth/
---

# Stealth 模块总览

> 源码位置: `include/prism/stealth/`

## 模块职责

Stealth 是 TLS 伪装方案模块，负责：

- **方案抽象**: 定义 `stealth_scheme` 基类，统一伪装方案接口
- **分层检测**: Tier 0/1/2 三级检测架构，从快速到详细逐层筛选
- **方案执行**: 按优先级依次尝试方案，直到匹配成功
- **方案注册**: 单例注册表管理所有伪装方案
- **Native 兜底**: 标准 TLS 方案作为最终兜底

## 文件结构

### 头文件

```
include/prism/stealth/
├── scheme.hpp           # 方案基类定义
├── executor.hpp         # 方案执行器
├── registry.hpp         # 方案注册表
├── native.hpp           # Native 兜底方案
├── anytls/              # AnyTLS 伪装方案
├── ech/                 # ECH 伪装方案
├── reality/             # Reality 伪装方案
├── restls/              # Restls 伪装方案
├── shadowtls/           # ShadowTLS 伪装方案
└── trusttunnel/         # TrustTunnel 伪装方案
```

## 核心组件

| 组件 | 头文件 | 职责 |
|------|--------|------|
| [[scheme|方案基类]] | `scheme.hpp` | 抽象基类、检测结果、执行上下文 |
| [[executor|执行器]] | `executor.hpp` | 方案管道执行、候选筛选 |
| [[registry|注册表]] | `registry.hpp` | 单例方案注册、查询 |
| [[native|Native方案]] | `native.hpp` | 标准 TLS 兜底方案 |

## 分层检测架构

Stealth 模块采用三级检测架构，按成本从低到高逐层筛选：

```
入站 TLS 连接
       │
       ▼
┌─────────────────────────────────────┐
│  Tier 0: sniff() 零成本检测          │
│  - 字节比较，无 HMAC/解密            │
│  - 如 Reality session_id 标记       │
│  - 命中后可独占跳过其他方案           │
└─────────────────────────────────────┘
       │ 未命中
       ▼
┌─────────────────────────────────────┐
│  Tier 1: verify() 有成本验证         │
│  - HMAC 验证或解密                   │
│  - 如 ShadowTLS HMAC、AnyTLS ECH    │
│  - 返回评分 (0-1000)                 │
└─────────────────────────────────────┘
       │ 未命中
       ▼
┌─────────────────────────────────────┐
│  Tier 2: guess() 模糊匹配            │
│  - 依赖 SNI 路由                     │
│  - 如 Restls、TrustTunnel、Native   │
│  - 作为兜底方案                      │
└─────────────────────────────────────┘
```

## 执行流程

```
recognition 分析 ClientHello
       │
       ├── 特征位图提取
       ├── Tier 0 快速检测
       ├── Tier 1 详细检测
       ├── Tier 2 模糊匹配
       │
       ▼
生成候选方案列表
       │
       ▼
scheme_executor 执行
       │
       ├── 按候选顺序尝试
       ├── 每个方案执行 handshake()
       │       ├── 返回 TLS: 不是此方案，继续下一个
       │       └── 返回具体协议: 匹配成功
       │
       ├── 全部失败 → Native 兜底
       │
       ▼
返回 handshake_result
       ├── transport: 最终传输层
       ├── detected: 检测到的内层协议
       └── preread: 预读数据
```

## 方案列表

| 方案 | Tier | 独占 | 检测方式 | 说明 |
|------|------|------|----------|------|
| Reality | 0 | 是 | session_id 标记 | 最早期的 TLS 伪装方案 |
| ShadowTLS | 1 | 是 | HMAC 验证 | 真实 TLS 握手伪装 |
| AnyTLS | 1 | 是 | ECH 解密 | 加密 ClientHello |
| Restls | 2 | 否 | SNI 匹配 | 模糊匹配方案 |
| TrustTunnel | 2 | 否 | SNI 匹配 | 模糊匹配方案 |
| Native | 2 | 否 | 兜底 | 标准 TLS 处理 |

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[../channel/overview|Channel]] | 传输层抽象 |
| 依赖 | [[../protocol/overview|Protocol]] | 协议类型、TLS 特征 |
| 依赖 | [[../recognition/overview|Recognition]] | ClientHello 分析结果 |
| 依赖 | [[../fault/overview|Fault]] | 错误码 |
| 依赖 | [[../memory/overview|Memory]] | PMR 容器 |
| 被依赖 | [[../agent/overview|Agent]] | 会话处理使用伪装方案 |

## 相关文档

- [[scheme|方案基类详解]] — stealth_scheme 抽象类定义
- [[executor|执行器详解]] — scheme_executor 管道执行
- [[registry|注册表详解]] — scheme_registry 单例管理
- [[native|Native 方案]] — 标准 TLS 兜底方案
- [[../recognition/recognition|Recognition 模块]] — 协议识别与候选生成