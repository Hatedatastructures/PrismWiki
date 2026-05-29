---
title: Gauge EMA 平滑瞬时值原语
layer: core
source: include/prism/stats/gauge.hpp
module: stats
tags: [stats, gauge, ema, performance]
created: 2026-05-23
updated: 2026-05-23
---

# Gauge EMA 平滑瞬时值原语

`gauge` 是 Stats 模块的底层原语之一，提供基于指数移动平均（Exponential Moving Average, EMA）的瞬时值平滑。与 [[core/stats/counter|counter]] 的累计计数不同，gauge 用于跟踪随时间波动的连续值（如延迟、负载百分比），过滤短期抖动，输出稳定的趋势值。

> 源码: `include/prism/stats/gauge.hpp`

## 类定义

```cpp
namespace psm::stats
{
    class gauge
    {
    public:
        explicit gauge(double alpha = 7.0 / 8.0) noexcept;
        void update(double sample) noexcept;
        [[nodiscard]] auto value() const noexcept -> double;
        void reset() noexcept;

    private:
        double value_{0.0};
        double alpha_;
    };
}
```

## 设计决策

### 非线程安全

`gauge` 故意设计为非线程安全，这是基于使用场景的决策：

- `gauge` 用于 [[core/stats/runtime|worker_load]] 的 `observe()` 协程中
- `observe()` 在单个 worker 的 `io_context` 上运行，天然是单线程的
- 无需原子操作开销，`double` 的读写直接使用普通 `mov` 指令

如果需要跨线程读取 gauge 值，应在单写者线程侧通过 `value()` 读取后，存入 `std::atomic` 或快照结构体。

### EMA 平滑系数 alpha = 7/8

默认 `alpha = 7/8 = 0.875`，这意味着：

```
平滑公式: value = alpha * value + (1 - alpha) * sample
         = 0.875 * value + 0.125 * sample

含义:
- 历史值权重: 87.5%
- 新采样权重: 12.5%
- 约 8 个采样周期到达 63% 的新值影响（1/e 时间常数 ≈ 8 次采样）
```

选择 7/8 的原因：
1. 与 [[core/stats/runtime|worker_load]] 的 `observe()` 协程中的延迟平滑算法保持一致
2. 与 Linux 内核 load average 的平滑策略类似
3. 可以用整数运算实现（`(value * 7 + sample) / 8`），避免浮点除法

### 为什么不用滑动窗口

| 方案 | 优点 | 缺点 |
|------|------|------|
| EMA | O(1) 内存，O(1) 计算，无窗口边界效应 | 无精确窗口语义 |
| 滑动窗口平均值 | 精确窗口统计 | O(n) 内存，需要存储历史样本 |
| 加权滑动窗口 | 近期样本权重更高 | O(n) 内存，计算复杂 |

EMA 是最适合实时监控场景的方案：极低的开销、平滑的输出、自动衰减旧值。

## 方法详解

### 构造函数

```cpp
explicit gauge(double alpha = 7.0 / 8.0) noexcept : alpha_(alpha) {}
```

**参数**:
- `alpha`: EMA 平滑系数，范围 (0, 1)
  - 接近 1: 强平滑，历史值权重高，输出变化缓慢
  - 接近 0: 弱平滑，新采样权重高，输出快速跟随输入

**注意**: `alpha` 在构造后不可修改，避免运行期参数变化导致统计不一致。

### update

```cpp
void update(double sample) noexcept
{
    value_ = alpha_ * value_ + (1.0 - alpha_) * sample;
}
```

**行为**: 输入新采样值，更新 EMA 平滑值。

**数学展开**:

```
value_new = alpha * value_old + (1 - alpha) * sample

展开递推:
value_n = (1-alpha) * sample_n
        + alpha * (1-alpha) * sample_{n-1}
        + alpha^2 * (1-alpha) * sample_{n-2}
        + ...
        + alpha^n * value_0

权重几何级数衰减，最近采样权重最大。
```

**调用时机**: 在 [[core/stats/runtime|worker_load]] 的 `observe()` 协程中，每次采样事件循环延迟后调用。

**被调用（向上）**: [[core/stats/runtime|worker_load::observe()]]

### value

```cpp
[[nodiscard]] auto value() const noexcept -> double
{
    return value_;
}
```

**行为**: 返回当前 EMA 平滑值。不修改内部状态。

### reset

```cpp
void reset() noexcept
{
    value_ = 0.0;
}
```

**行为**: 将平滑值重置为零。通常用于测试或管理端重置。

## EMA 收敛特性

### 阶跃响应

假设输入从 0 突变到 100，alpha = 7/8：

```
采样序号  |  输入  |  gauge.value()
---------|--------|--------
    0    |   0    |   0.00
    1    |  100   |  12.50
    2    |  100   |  23.44
    3    |  100   |  33.01
    4    |  100   |  41.38
    5    |  100   |  48.71
    6    |  100   |  55.12
    7    |  100   |  60.73
    8    |  100   |  65.64
   16    |  100   |  87.63
   24    |  100   |  95.54
   32    |  100   |  98.43
```

达到 90% 目标值约需 16 次采样。在 [[core/stats/runtime|worker_load]] 的 `observe()` 中（250ms 采样间隔），约 4 秒收敛到 90%。

### 脉冲响应

单个异常采样 (100) 输入到稳定值为 0 的 gauge：

```
采样序号  |  输入  |  gauge.value()
---------|--------|--------
   -1    |   0    |   0.00
    0    |  100   |  12.50  ← 脉冲
    1    |   0    |  10.94
    2    |   0    |   9.57
    3    |   0    |   8.37
    4    |   0    |   7.33
    ...
    8    |   0    |   3.45
   16    |   0    |   0.28
```

单个脉冲在 16 次采样后衰减至 2.8%，32 次采样后基本消失。这就是 EMA 的天然脉冲抑制能力。

## 在 observe() 中的使用

[[core/stats/runtime|worker_load]] 的 `observe()` 协程使用了类似 EMA 的整数版本，而非直接使用 `gauge` 类。原因是 observe 中需要整数运算以匹配 `event_loop_lag_us_` 的 `atomic<uint64_t>` 类型：

```cpp
// observe() 中的 EMA（整数版，与 gauge 等价）
smoothed_lag_us = (smoothed_lag_us * 7ULL + effective_lag_us) / 8ULL;
```

这等价于 `gauge(7.0/8.0).update(effective_lag_us)` 的整数近似。

`gauge` 类本身提供给需要浮点精度的场景（如负载百分比、延迟百分位数等）使用。

## 与相关组件的对比

| 特性 | [[core/stats/gauge\|gauge]] | [[core/stats/counter\|counter]] |
|------|------|------|
| 数值类型 | `double` | `uint64_t` |
| 方向 | 可增可减（跟踪连续值） | 仅增或仅减（累计计数） |
| 平滑 | EMA（过滤抖动） | 无平滑（精确累计） |
| 线程安全 | 否（单写者） | 是（原子操作） |
| 缓存行对齐 | 否 | 是（alignas(64)） |
| 典型场景 | 延迟、负载百分比 | 连接数、字节数 |

## 潜在改进方向

1. **整数 EMA 特化**: 为 [[core/stats/runtime|worker_load]] 提供整数版本的 gauge，统一 EMA 实现
2. **配置化 alpha**: 允许通过配置文件调整平滑系数
3. **多级 EMA**: 支持 1min/5min/15min 多级平滑（类似 Linux load average）
4. **预热策略**: 构造时提供初始值，避免冷启动阶段的零值偏差

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/counter|counter]] -- 互补的原子计数器原语
- [[core/stats/runtime|runtime]] -- 在 observe() 中使用 EMA 的运行时统计
- [[core/stats/snapshot|snapshot]] -- gauge 值通过快照对外暴露
- [[core/instance/worker/stats|Worker 负载统计]] -- 旧版 observe() 中的 EMA 实现
