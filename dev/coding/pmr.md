---
layer: dev
title: PMR 内存策略
description: Prism 多态内存资源 (PMR) 策略，实现热路径零堆分配
source:
  - include/prism/memory/container.hpp
  - include/prism/memory/pool.hpp
  - benchmarks/MemoryBench.cpp
module: memory
type: concept
tags: [cpp, pmr, memory-management, performance, zero-allocation]
created: 2026-05-12
updated: 2026-05-18
related:
  - dev/cpp23-coroutines
  - dev/tcp
  - dev/agent
  - docs/prism/memory.md
confidence: high
layer: dev
---

# PMR 内存策略

Prism 使用 **PMR (Polymorphic Memory Resource)** 实现热路径零堆分配，确保高性能内存管理。

## 编码规范

### 核心概念

PMR (Polymorphic Memory Resource) 是 C++17 引入的多态内存资源抽象，允许容器在运行时选择内存分配策略，而无需修改容器类型。

### 为什么使用 PMR

- **零堆分配**: 热路径使用预分配内存池，避免动态内存分配
- **缓存友好**: 内存竞技场提供连续内存布局
- **确定性释放**: 批量释放内存，减少碎片

### 内存类型

#### 全局池字符串

`memory::string` 使用全局内存池：

```cpp
namespace psm::memory {
    using string = std::pmr::string;
}
```

特点：
- 进程生命周期
- 线程安全
- 适用于配置、域名等长生命周期字符串

#### 帧竞技场向量

`memory::vector<T>` 使用帧竞技场：

```cpp
namespace psm::memory {
    template<typename T>
    using vector = std::pmr::vector<T>;
}
```

特点：
- 请求生命周期
- 单线程使用
- 适用于临时缓冲区、数据包处理

### 内存池实现

#### 全局内存池

启动时必须调用 `memory::system::enable_global_pooling()` 初始化：

```cpp
// main.cpp
int main() {
    // 初始化全局内存池
    psm::memory::system::enable_global_pooling();

    // ... 其他初始化
}
```

#### 帧竞技场

每个请求或会话使用帧竞技场：

```cpp
auto handle_request(tcp::socket socket) -> net::awaitable<void> {
    // 创建帧竞技场
    psm::memory::arena arena;

    // 使用竞技场分配向量
    psm::memory::vector<std::byte> buffer(&arena);

    // 请求结束，竞技场自动释放所有内存
}
```

### Memory 模块结构

```
include/prism/memory/
├── container.hpp   // PMR 容器定义 (string, vector 等)
└── pool.hpp        // 内存池实现
```

### 使用指南

#### 字符串使用

```cpp
// 配置和长生命周期字符串
memory::string domain = "example.com";
memory::string config_value = some_config;

// 避免频繁转换
std::string std_str = "temp";
memory::string pmr_str(std_str);  // 有拷贝开销
```

#### 向量使用

```cpp
// 在协程中使用帧竞技场
auto process_data(buffer_span input) -> net::awaitable<void> {
    memory::arena arena;
    memory::vector<std::byte> buffer(&arena);

    buffer.reserve(input.size());
    std::copy(input.begin(), input.end(), std::back_inserter(buffer));

    // 处理数据...
}
```

#### 内存池选择

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 配置字符串 | `memory::string` | 长生命周期，全局共享 |
| 域名缓存 | `memory::string` | 长生命周期 |
| 临时缓冲区 | `memory::vector<T>` | 短生命周期，帧分配 |
| 数据包处理 | `memory::vector<std::byte>` | 批量释放高效 |

### 初始化流程

```cpp
// main.cpp 启动顺序
int main() {
    // 1. 初始化全局内存池
    psm::memory::system::enable_global_pooling();

    // 2. 注册伪装方案
    psm::stealth::register_all_schemes();

    // 3. 加载配置
    auto config = psm::loader::load("config.json");

    // 4. 初始化日志
    psm::trace::init(config.trace);

    // ...
}
```

### 性能考量

- **避免混合使用**: 不要在热路径中混合 PMR 和标准分配器
- **预分配**: 对于已知大小的容器，使用 `reserve()` 预分配
- **竞技场大小**: 根据实际负载调整帧竞技场初始大小

---

## 原理参考

> 以下内容为 PMR 内存池的深入原理、详细使用示例、性能分析与排障指南。

### PMR 基础

#### std::pmr::memory_resource

`std::pmr::memory_resource` 是 PMR 的核心抽象基类，定义了三个虚函数：

```cpp
class memory_resource {
public:
    virtual ~memory_resource();

    // 分配 n 字节，对齐到 alignment
    void* allocate(size_t n, size_t alignment = alignof(max_align_t));

    // 释放之前 allocate 返回的指针
    void deallocate(void* p, size_t n, size_t alignment = alignof(max_align_t));

    // 比较 two memory_resources 是否相等
    bool is_equal(const memory_resource& other) const noexcept;

private:
    virtual void* do_allocate(size_t n, size_t alignment) = 0;
    virtual void  do_deallocate(void* p, size_t n, size_t alignment) = 0;
    virtual bool  do_is_equal(const memory_resource& other) const noexcept = 0;
};
```

`allocate` 和 `deallocate` 是公共接口，内部调用 `do_allocate` 和 `do_deallocate` 虚函数。这种设计允许运行时策略切换，同时保持类型安全。

#### 标准库提供的 memory_resource

##### new_delete_resource

最基础的实现，直接调用全局 `::operator new` 和 `::operator delete`：

```cpp
auto* mr = std::pmr::new_delete_resource();
void* p = mr->allocate(256);  // 等价于 ::operator new(256)
mr->deallocate(p, 256);       // 等价于 ::operator delete(p)
```

用途：作为其他池的上游资源，或作为默认资源。

##### null_resource

拒绝所有分配请求的"空"资源，分配时抛出 `std::bad_alloc`：

```cpp
auto* mr = std::pmr::null_resource();
mr->allocate(1);  // 抛出 std::bad_alloc
```

用途：测试、调试分配来源，防止意外使用默认资源。

##### monotonic_buffer_resource

单调递增缓冲区资源，内存只从预分配缓冲区前端分配，不单独释放每个分配。整体销毁或调用 `release()` 时一次性回收：

```cpp
// 使用栈缓冲区
char buffer[4096];
std::pmr::monotonic_buffer_resource mr(buffer, sizeof(buffer));

// 使用上游资源作为后备
std::pmr::monotonic_buffer_resource mr2(4096, std::pmr::new_delete_resource());

// 分配不单独释放
void* p1 = mr.allocate(100);
void* p2 = mr.allocate(200);
mr.deallocate(p1, 100);  // 无效，内存不回收
mr.release();            // 一次性回收 p1 和 p2
```

特点：
- 分配速度极快（只增加指针偏移）
- 不支持单独释放（只支持 `release()` 整体回收）
- 适合请求级临时对象

##### synchronized_pool_resource

线程安全的内存池资源，内部使用互斥锁保护：

```cpp
std::pmr::pool_options opts;
opts.largest_required_pool_block = 16384;  // 最大池化块大小
opts.max_blocks_per_chunk = 256;           // 每个 chunk 最大块数

std::pmr::synchronized_pool_resource mr(opts, upstream);
```

特点：
- 内部使用锁，多线程安全
- 按大小分桶池化
- 适合全局共享池

##### unsynchronized_pool_resource

非线程安全的内存池资源，不使用任何锁：

```cpp
std::pmr::unsynchronized_pool_resource mr(opts, upstream);
```

特点：
- 无锁，分配释放最快
- 仅限单线程使用
- 适合线程独占池

#### polymorphic_allocator

`std::pmr::polymorphic_allocator<T>` 是 PMR 的分配器模板，持有 `memory_resource*` 指针：

```cpp
template<typename T>
class polymorphic_allocator {
public:
    using value_type = T;

    polymorphic_allocator(memory_resource* r) noexcept;

    T* allocate(size_t n);
    void deallocate(T* p, size_t n);

    // 用于嵌套容器（如 vector<string>）
    polymorphic_allocator select_on_container_copy_construction() const;

private:
    memory_resource* resource_;
};
```

关键特性：

1. **运行时绑定**：构造时传入 `memory_resource*`，后续分配使用该资源
2. **传播语义**：嵌套容器自动继承父容器的分配器
3. **无状态**：分配器本身无状态，只有资源指针

#### PMR 容器

C++17 为所有标准容器提供了 PMR 别名：

```cpp
namespace std::pmr {
    using string = basic_string<char, char_traits<char>, polymorphic_allocator<char>>;

    template<typename T>
    using vector = std::vector<T, polymorphic_allocator<T>>;

    template<typename T>
    using list = std::list<T, polymorphic_allocator<T>>;

    template<typename K, typename V>
    using map = std::map<K, V, less<K>, polymorphic_allocator<pair<const K, V>>>;

    template<typename K, typename V>
    using unordered_map = std::unordered_map<K, V, hash<K>, equal_to<K>,
                                              polymorphic_allocator<pair<const K, V>>>;
}
```

PMR 容器的关键特性：

1. **构造时传入资源**：`pmr::vector<int> vec(mr)` 使用 `mr` 分配
2. **默认资源**：不传入资源时使用 `std::pmr::get_default_resource()`
3. **传播语义**：`pmr::vector<pmr::string>` 中 string 自动继承 vector 的资源

#### 默认资源机制

`std::pmr::set_default_resource()` 设置全局默认资源：

```cpp
// 设置默认资源
std::pmr::set_default_resource(&my_pool);

// 之后所有不带显式资源的 PMR 容器使用 my_pool
std::pmr::string s;  // 使用 my_pool
std::pmr::vector<int> v;  // 使用 my_pool
```

程序启动时默认资源是 `new_delete_resource()`。Prism 在启动时调用 `memory::system::enable_global_pooling()` 将默认资源切换到全局池。

### Prism 内存系统

#### 容器别名定义

Prism 在 `include/prism/memory/container.hpp` 定义了 PMR 容器别名：

```cpp
namespace psm::memory
{
    // 核心类型
    using resource = std::pmr::memory_resource;
    using resource_pointer = std::add_pointer_t<resource>;

    // 分配器
    template<typename Type>
    using allocator = std::pmr::polymorphic_allocator<Type>;

    // 池资源
    using synchronized_pool = std::pmr::synchronized_pool_resource;
    using unsynchronized_pool = std::pmr::unsynchronized_pool_resource;
    using monotonic_buffer = std::pmr::monotonic_buffer_resource;

    // 容器
    using string = std::pmr::string;

    template<typename Value>
    using vector = std::pmr::vector<Value>;

    template<typename Value>
    using list = std::pmr::list<Value>;

    template<typename Key, typename Value, typename Compare = std::less<Key>>
    using map = std::pmr::map<Key, Value, Compare>;

    template<typename Key, typename Value>
    using unordered_map = std::pmr::unordered_map<Key, Value>;

    template<typename Key>
    using unordered_set = std::pmr::unordered_set<Key>;

    // 辅助函数
    inline auto current_resource() -> resource_pointer {
        return std::pmr::get_default_resource();
    }
}
```

使用这些别名，Prism 代码中使用 `memory::string` 和 `memory::vector<T>` 而不是 `std::pmr::string` 和 `std::pmr::vector<T>`。

#### 内存策略配置

`policy` 结构体定义池化策略参数：

```cpp
struct policy
{
    // 每个 Chunk 包含的最大块数
    static constexpr std::size_t max_blocks = 256;

    // 最大池化阈值，16KB 足以覆盖 HTTP Header 等典型对象
    static constexpr std::size_t max_pool_size = 16384;
};
```

`max_blocks_per_chunk = 256`：每个 chunk 最多包含 256 个同大小块。较小的值降低内存峰值，较大的值减少 chunk 分配次数。

`largest_required_pool_block = 16384`：超过 16KB 的分配直通系统堆。典型代理场景中，HTTP Header、DNS 响应、协议头都在 16KB 以内，大缓冲区（如 64KB 传输缓冲区）使用堆分配更合理。

#### 全局内存系统

`system` 类提供全局和线程局部内存池的访问接口：

```cpp
class system
{
public:
    // 获取全局线程安全池
    static synchronized_pool* global_pool() {
        static auto* pool = []() {
            std::pmr::pool_options opts;
            opts.largest_required_pool_block = policy::max_pool_size;
            opts.max_blocks_per_chunk = policy::max_blocks;
            return new synchronized_pool(opts, std::pmr::new_delete_resource());
        }();
        return pool;
    }

    // 获取线程局部无锁池
    static unsynchronized_pool* thread_local_pool() {
        thread_local auto* pool = []() {
            std::pmr::pool_options opts;
            opts.largest_required_pool_block = policy::max_pool_size;
            opts.max_blocks_per_chunk = policy::max_blocks;
            return new unsynchronized_pool(opts, std::pmr::new_delete_resource());
        }();
        return pool;
    }

    // 热路径池（语义别名）
    static unsynchronized_pool* hot_path_pool() {
        return thread_local_pool();
    }

    // 启用全局池化
    static void enable_global_pooling() {
        std::pmr::set_default_resource(global_pool());
    }
};
```

设计要点：

1. **单例模式**：使用 static/thread_local 保证全局唯一实例
2. **延迟初始化**：首次调用时才创建池
3. **new 分配**：全局池使用 `new` 分配，避免静态析构顺序问题
4. **上游资源**：使用 `new_delete_resource()` 作为后备

#### 池类型选择

`pool_type` 枚举定义两种池策略：

```cpp
enum class pool_type
{
    global, // 全局线程安全池
    local,  // 线程局部无锁池
};
```

- `global`：适用于跨线程传递、生命周期不确定的对象
- `local`：适用于热路径单线程对象（默认）

#### pooled_object 基类

`pooled_object<T, Type>` 模板通过 CRTP 惯用法使继承类自动使用内存池分配：

```cpp
template<typename T, pool_type Type = pool_type::local>
class pooled_object
{
public:
    static resource_pointer get_target_pool() {
        if (Type == pool_type::global) {
            return system::global_pool();
        }
        return system::thread_local_pool();
    }

    void* operator new(std::size_t count) {
        if (count <= policy::max_pool_size) {
            return get_target_pool()->allocate(count);
        }
        return ::operator new(count);  // 大对象直通系统堆
    }

    void operator delete(void* ptr, std::size_t count) {
        if (count <= policy::max_pool_size) {
            get_target_pool()->deallocate(ptr, count);
        } else {
            ::operator delete(ptr);
        }
    }

    // 数组版本
    void* operator new[](std::size_t count);
    void operator delete[](void* ptr, std::size_t count);
};
```

使用方式：

```cpp
class Connection : public pooled_object<Connection, pool_type::local> {
    // Connection 对象自动从线程局部池分配
};

class Config : public pooled_object<Config, pool_type::global> {
    // Config 对象从全局池分配，可跨线程传递
};
```

#### frame_arena 帧分配器

`frame_arena` 是短生命周期临时对象的专用分配器：

```cpp
class frame_arena
{
    std::byte buffer_[512];  // 栈上预分配缓冲区
    monotonic_buffer resource_;

public:
    frame_arena()
        : resource_(buffer_, sizeof(buffer_), system::thread_local_pool())
    {}

    // 禁止拷贝
    frame_arena(const frame_arena&) = delete;
    frame_arena& operator=(const frame_arena&) = delete;

    // 获取内存资源
    resource_pointer get() { return &resource_; }

    // 重置（一次性回收所有分配）
    void reset() { resource_.release(); }
};
```

设计要点：

1. **栈缓冲区**：512 字节栈缓冲区覆盖典型 mux 地址头
2. **上游资源**：超过栈缓冲区后从 `thread_local_pool` 分配
3. **无锁**：`monotonic_buffer_resource` 本身无锁
4. **批量回收**：`reset()` 一次性回收，避免逐个释放

典型用途：

```cpp
class session {
    memory::frame_arena frame_arena_;

    auto process_request() -> net::awaitable<void> {
        // 在帧上创建临时对象
        memory::vector<std::byte> header(frame_arena_.get());
        parse_header(header);

        // 处理完毕后重置
        frame_arena_.reset();  // 所有分配一次性回收
    }
};
```

#### 内存池初始化流程

Prism 启动时的内存初始化序列：

```
main.cpp:
    memory::system::enable_global_pooling()
           |
           v
    std::pmr::set_default_resource(global_pool())
           |
           v
    后续所有 memory::string/memory::vector 默认使用全局池
           |
           v
    worker 线程创建时各拥有独立的 thread_local_pool
           |
           v
    session 创建时各拥有独立的 frame_arena
```

详细流程：

1. `enable_global_pooling()` 设置默认资源为全局同步池
2. 全局配置、账户目录等长期对象从全局池分配
3. 每个 worker 线程创建时，`thread_local_pool()` 首次调用创建线程独占池
4. 每个会话创建时，`frame_arena` 初始化，栈缓冲区 + 线程池作为上游
5. 会话处理期间，临时对象从 `frame_arena` 分配
6. 会话结束时，`frame_arena.reset()` 一次性回收，池对象归还线程池

### 使用示例

#### 基本容器使用

```cpp
#include <prism/memory.hpp>

// 使用默认资源（全局池）
memory::string host("example.com");
memory::vector<std::byte> buffer;

// 使用显式资源
auto* mr = memory::system::thread_local_pool();
memory::string path(mr);
memory::vector<std::uint8_t> data(mr);

// 嵌套容器自动继承分配器
memory::vector<memory::string> headers(mr);
headers.push_back("Host: example.com");  // string 自动使用 mr
```

#### 协程中使用帧分配器

```cpp
auto handle_connection(session_context& ctx) -> net::awaitable<void>
{
    // 从帧竞技场分配临时对象
    memory::vector<std::byte> request(ctx.frame_arena.get());

    // 预读数据
    co_await ctx.inbound->async_read_some(request);

    // 解析协议头（临时对象）
    memory::string method(ctx.frame_arena.get());
    parse_method(request, method);

    // 处理完毕，重置帧竞技场
    ctx.frame_arena.reset();

    // 后续处理...
}
```

#### pooled_object 类继承

```cpp
// 高频连接对象，使用线程局部池
class Connection : public memory::pooled_object<Connection, memory::pool_type::local>
{
    memory::vector<std::byte> buffer_;

public:
    Connection() : buffer_(memory::system::thread_local_pool()) {}
};

// 全局配置对象，使用全局池
class RuntimeConfig : public memory::pooled_object<RuntimeConfig, memory::pool_type::global>
{
    memory::unordered_map<memory::string, memory::string> entries_;

public:
    RuntimeConfig() : entries_(memory::system::global_pool()) {}
};

// 使用
auto conn = new Connection();  // 从线程池分配
auto cfg = new RuntimeConfig();  // 从全局池分配
```

#### DNS 缓存使用 PMR

```cpp
class DnsCache
{
    memory::unordered_map<memory::string, DnsEntry> cache_;

public:
    DnsCache() : cache_(memory::system::global_pool()) {}

    auto lookup(const std::string_view domain) -> std::optional<DnsEntry>
    {
        memory::string key(domain, cache_.get_allocator().resource());
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            return it->second;
        }
        return std::nullopt;
    }

    void store(const std::string_view domain, const DnsEntry& entry)
    {
        memory::string key(domain, cache_.get_allocator().resource());
        cache_.emplace(std::move(key), entry);
    }
};
```

#### 协议解析批量分配

```cpp
auto parse_headers(std::span<const std::byte> data) -> memory::vector<memory::string>
{
    memory::frame_arena arena;
    auto mr = arena.get();

    memory::vector<memory::string> headers(mr);

    // 批量分配解析结果
    for (std::size_t i = 0; i < 100; ++i) {
        memory::string header(mr);
        // 解析单个 header
        headers.push_back(std::move(header));
    }

    // 返回前复制到全局池（或使用 frame_arena 作为上游）
    // 注意：arena.reset() 后返回的 vector 内的 string 会失效
    // 需要将结果移动到上游池管理的容器

    return headers;  // 注意生命周期！
}
```

正确做法：

```cpp
auto parse_headers(memory::resource_pointer result_mr, std::span<const std::byte> data)
    -> memory::vector<memory::string>
{
    memory::frame_arena arena;
    auto parse_mr = arena.get();

    // 解析时使用帧竞技场
    memory::vector<memory::string> temp(parse_mr);
    // ... 解析逻辑

    // 结果复制到目标资源
    memory::vector<memory::string> result(result_mr);
    for (auto& h : temp) {
        result.push_back(memory::string(h, result_mr));
    }

    arena.reset();  // 安全回收临时对象
    return result;
}
```

### 性能分析

#### 基准测试结果

Prism 的 `MemoryBench.cpp` 测试了各种分配策略的性能：

| 测试场景 | 分配器 | 128B 分配延迟 | 512B 分配延迟 |
|---------|--------|--------------|--------------|
| std::string | new/delete | ~150ns | ~180ns |
| memory::string (默认) | global_pool | ~25ns | ~30ns |
| memory::string (thread_local) | unsynchronized_pool | ~12ns | ~15ns |
| memory::string (frame_arena) | monotonic_buffer | ~8ns | ~10ns |

关键发现：

1. **线程局部池比全局池快 2x**：无锁开销
2. **帧竞技场比线程池快 1.5x**：单调增长，无释放逻辑
3. **PMR 比标准分配器快 10-20x**：池化 + 无锁

#### 多线程竞争分析

```
线程数 vs 分配吞吐量 (ops/sec)

 线程数 | global_pool | thread_local_pool | new_delete
   1    |   800K      |      1.2M         |    200K
   2    |   600K      |      2.4M         |    180K
   4    |   400K      |      4.8M         |    150K
   8    |   280K      |      9.6M         |    100K

 观察：thread_local_pool 随线程数线性增长
       global_pool 随线程数下降（锁竞争）
       new_delete 随线程数严重下降
```

`thread_local_pool` 的吞吐量随线程数线性增长，完美扩展。`global_pool` 因内部锁导致吞吐量下降。`new_delete` 在高线程数下竞争最严重。

#### 内存占用分析

| 分配策略 | 1 万个 256B 对象 | 内存峰值 | 利用率 |
|---------|------------------|---------|-------|
| new/delete | 256KB + 元数据 | ~300KB | ~85% |
| global_pool | 256KB + chunk | ~280KB | ~91% |
| thread_local_pool x 4 | 256KB x 4 | ~1.12MB | ~91% |
| frame_arena | 栈缓冲区 512B | ~512B | 100% |

线程池的代价是每个线程独立占用内存。对于 4 线程引擎，内存占用增加约 4 倍，但换来无锁性能。

#### 热路径零分配验证

使用性能分析工具验证热路径无分配：

```
session::diversion() 调用链：
    probe::detect()       -> 无分配（使用预分配 buffer）
    handshake::execute()  -> 使用 frame_arena，批量回收
    handler::process()    -> 使用 session 内存池
    tunnel::forward()     -> 无分配（使用传输层 buffer）

验证方法：
1. 运行 build_release/src/Prism.exe
2. 使用 perf/VTune 分析 malloc 调用
3. 数据转发期间 malloc 计数应为 0
```

### 最佳实践

#### 选择合适的内存资源

| 场景 | 推荐资源 | 原因 |
|------|---------|------|
| 全局配置/缓存 | `global_pool()` | 跨线程访问，需线程安全 |
| Worker 独占对象 | `thread_local_pool()` | 无锁性能最大化 |
| 请求级临时对象 | `frame_arena` | 批量回收，极快分配 |
| 大对象 (>16KB) | 直通堆 | 超过池化阈值 |
| 协程帧 | `thread_local_pool()` | 协程在单 io_context 执行 |

#### 容器传播语义

嵌套容器自动继承父容器分配器：

```cpp
// 正确：外层指定资源，内层自动继承
memory::vector<memory::string> vec(mr);
vec.push_back("text");  // string 使用 mr

// 错误：内层单独指定不同资源
memory::vector<memory::string> vec(mr);
vec.push_back(memory::string(other_mr));  // 混合资源，潜在问题
```

#### 生命周期管理

1. **frame_arena 对象不得跨越 reset()**
2. **thread_local_pool 对象不得跨线程传递**
3. **global_pool 对象可跨线程传递**

```cpp
// 错误：跨 reset() 访问
memory::frame_arena arena;
memory::string* p = new memory::string(arena.get());
arena.reset();
p->append("data");  // 未定义行为，内存已回收

// 正确：reset 前完成所有操作
memory::string result(arena.get());
result.append("data");
process(result);
arena.reset();
```

#### 避免混合资源

不同资源管理的对象不应混合：

```cpp
// 错误：混合资源
memory::vector<int> vec_a(mr_a);
memory::vector<int> vec_b(mr_b);
vec_a.swap(vec_b);  // 元素在不同资源间移动

// 正确：统一资源
memory::vector<int> vec_a(mr);
memory::vector<int> vec_b(mr);
vec_a.swap(vec_b);  // 同一资源，安全
```

#### 对象池继承

高频分配的类应继承 `pooled_object`：

```cpp
// 正确：高频类使用池
class Packet : public memory::pooled_object<Packet>
{
    // operator new/delete 自动从池分配
};

// 错误：高频类使用堆
class Packet  // 每次 new 调用 malloc
{
};
```

#### 异常安全

PMR 分配可能抛出 `std::bad_alloc`：

```cpp
// 正确：捕获异常
try {
    memory::vector<std::byte> buffer(mr);
    buffer.resize(huge_size);
} catch (const std::bad_alloc& e) {
    log_error("allocation failed");
    return fault::code::out_of_memory;
}

// 或使用 noexcept 构造
auto buffer = memory::vector<std::byte>(mr);
if (buffer.capacity() < needed) {
    return fault::code::buffer_overflow;
}
```

### 常见问题

#### Q1: 为什么不直接使用 std::pmr？

Prism 定义 `memory::string` 等别名而非直接使用 `std::pmr::string` 的原因：

1. **命名空间一致性**：`psm::memory::` 前缀与项目风格统一
2. **扩展便利性**：可添加项目特定的容器或辅助函数
3. **文档清晰**：明确标识使用 Prism 内存策略
4. **默认资源控制**：`enable_global_pooling()` 统一控制默认资源

#### Q2: thread_local_pool 能跨线程吗？

**不能**。`unsynchronized_pool_resource` 内部无锁，跨线程访问会导致数据竞争。

跨线程传递对象的正确做法：

```cpp
// 错误：跨线程传递 thread_local 池对象
void thread_a() {
    auto obj = new ObjectFromLocalPool();
    queue.push(obj);  // 传给 thread_b
}

void thread_b() {
    auto obj = queue.pop();
    delete obj;  // 错误：归还到 thread_b 的池
}

// 正确：使用 global_pool 跨线程对象
class GlobalObject : public memory::pooled_object<GlobalObject, memory::pool_type::global>
{
};
```

#### Q3: frame_arena 的栈缓冲区大小如何确定？

Prism 默认 512 字节栈缓冲区，覆盖典型 mux 地址头解析需求。调整建议：

| 场景 | 建议大小 |
|------|---------|
| HTTP Header 解析 | 1024-4096B |
| DNS 响应解析 | 512B（已足够） |
| TLS ClientHello | 2048-4096B |
| mux 数据帧头 | 256B |

过大的栈缓冲区增加栈压力，可能导致栈溢出。建议不超过 4096B。

#### Q4: 如何调试内存问题？

使用 `null_resource` 检测意外分配：

```cpp
// 设置 null_resource，任何分配都会抛出异常
std::pmr::set_default_resource(std::pmr::null_resource());

try {
    memory::string s;  // 应该从显式资源分配
    s = "test";
} catch (const std::bad_alloc&) {
    // 检测到意外使用默认资源
}
```

#### Q5: PMR 与智能指针兼容吗？

`std::shared_ptr` 和 `std::unique_ptr` 不使用 PMR 分配器。指针管理的对象需要手动池化：

```cpp
// 正确：使用 pooled_object
class Session : public memory::pooled_object<Session>
{
};
auto s = std::shared_ptr<Session>(new Session());  // 从池分配

// 或使用 allocate_shared（需要自定义分配器）
auto s = std::allocate_shared<Session>(memory::allocator<Session>(mr));
```

#### Q6: 16KB 池化阈值够用吗？

典型代理场景中：

- HTTP Header：平均 500B，最大 8KB
- DNS 响应：平均 200B，最大 4KB
- TLS ClientHello：平均 512B，最大 16KB
- 协议头：平均 64B，最大 256B

16KB 阈值覆盖 99% 的场景。对于罕见大对象（如大证书），直通堆更合理。

### 排障指南

#### 内存泄漏排查

PMR 池不会自动释放未归还的内存：

```cpp
// 潜在泄漏：忘记归还
void* p = pool->allocate(256);
// ... 使用 p
// 忘记 pool->deallocate(p, 256)
// 池会保留这块内存，无法用于其他分配
```

排查方法：

1. 记录分配/释放计数
2. 使用 `pool_options` 控制 chunk 数量
3. 定期调用 `release()` 清理未使用内存（会破坏池）

#### 性能下降诊断

分配性能突然下降的可能原因：

| 现象 | 可能原因 | 解决方案 |
|------|---------|---------|
| 全局池延迟增加 | 线程数增加导致锁竞争 | 迁移到 thread_local_pool |
| thread_local_pool 延迟增加 | chunk 耗尽需向上游分配 | 增加 max_blocks_per_chunk |
| frame_arena 延迟增加 | 超过栈缓冲区 | 增大栈缓冲区或预分配 |

#### 崩溃诊断

PMR 相关崩溃的常见原因：

1. **跨线程访问 thread_local_pool**
2. **frame_arena.reset() 后访问已回收对象**
3. **deallocate 到错误的池**
4. **对齐不匹配**

诊断步骤：

```cpp
// 1. 添加资源标记
class DebugResource : public std::pmr::memory_resource
{
    std::atomic<std::size_t> alloc_count_{0};
    std::atomic<std::size_t> dealloc_count_{0};

    void* do_allocate(std::size_t n, std::size_t align) override {
        alloc_count_++;
        return upstream_->allocate(n, align);
    }

    void do_deallocate(void* p, std::size_t n, std::size_t align) override {
        dealloc_count_++;
        upstream_->deallocate(p, n, align);
    }

    void dump() {
        std::cout << "allocs: " << alloc_count_ << " deallocs: " << dealloc_count_ << std::endl;
    }
};

// 2. 检查线程 ID
thread_local std::size_t owning_thread = get_current_thread_id();
void check_thread() {
    assert(get_current_thread_id() == owning_thread);
}
```

#### 内存峰值控制

线程池导致内存峰值增加：

```
解决方案：
1. 降低 max_blocks_per_chunk（减少每个 chunk 内存）
2. 使用共享池而非线程池（权衡性能）
3. 定期释放不活跃池
4. 监控池内存使用
```

#### 配置调优参数

根据负载调整 `policy` 参数：

```cpp
// 高并发场景（10 万连接）
struct policy {
    static constexpr std::size_t max_blocks = 128;  // 降低峰值
    static constexpr std::size_t max_pool_size = 8192;  // 小对象池化
};

// 低延迟场景（P99 优化）
struct policy {
    static constexpr std::size_t max_blocks = 512;  // 减少 chunk 分配
    static constexpr std::size_t max_pool_size = 32768;  // 大对象池化
};

// 内存受限场景
struct policy {
    static constexpr std::size_t max_blocks = 64;  // 最小峰值
    static constexpr std::size_t max_pool_size = 4096;  // 仅小对象池化
};
```

#### 池内存统计

实现池内存使用统计：

```cpp
class StatsResource : public std::pmr::memory_resource
{
    std::pmr::memory_resource* upstream_;
    std::atomic<std::size_t> total_allocated_{0};
    std::atomic<std::size_t> total_freed_{0};
    std::atomic<std::size_t> current_used_{0};

    void* do_allocate(std::size_t n, std::size_t align) override {
        total_allocated_ += n;
        current_used_ += n;
        return upstream_->allocate(n, align);
    }

    void do_deallocate(void* p, std::size_t n, std::size_t align) override {
        total_freed_ += n;
        current_used_ -= n;
        upstream_->deallocate(p, n, align);
    }

public:
    auto stats() const {
        return std::tuple{total_allocated_.load(), total_freed_.load(), current_used_.load()};
    }
};
```

## 相关页面

- [[overview]] - 编码规范概览
- [[dev/testing/benchmark|benchmark]] - 性能基准测试
