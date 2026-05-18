---
title: 生命周期安全
description: Prism 协程生命周期安全指南，包括 co_spawn 捕获和迭代器失效处理
---

# 生命周期安全

协程的挂起和恢复特性带来了独特的生命周期挑战。本文档定义 Prism 项目中的生命周期安全策略。

## co_spawn 捕获规则

### 按 `self` 捕获

`co_spawn` 的 lambda 必须按值捕获 `self`（shared_ptr）保持对象存活：

```cpp
// 正确示例
void session::start() {
    auto self = shared_from_this();
    net::co_spawn(
        io_context_,
        [self]() -> net::awaitable<void> {
            co_await self->handle_session();
        },
        net::detached
    );
}
```

### 错误示例

```cpp
// 错误：按引用捕获 this
void session::start_bad() {
    net::co_spawn(
        io_context_,
        [this]() -> net::awaitable<void> {
            co_await this->handle_session();  // 危险！this 可能已失效
        },
        net::detached
    );
}

// 错误：裸指针捕获
void session::start_bad2() {
    net::co_spawn(
        io_context_,
        [this]() -> net::awaitable<void> {
            co_await handle_session();  // 危险！
        },
        net::detached
    );
}
```

### 原因

协程挂起期间，原始调用栈可能已销毁：
1. `co_spawn` 将协程投递到 io_context
2. 原函数返回，局部变量销毁
3. `this` 指针可能指向已销毁对象
4. 协程恢复时访问 `this` 导致未定义行为

按值捕获 `shared_ptr` 会增加引用计数，确保对象在协程执行期间保持存活。

## 挂起恢复后的指针失效

### 问题

`co_await` 挂起恢复后，之前获取的裸指针/迭代器/引用可能失效：

```cpp
// 危险示例
auto handle_data() -> net::awaitable<void> {
    auto& container = get_container();
    auto* ptr = &container.front();  // 获取指针

    co_await async_read(socket, buffer);  // 挂起

    // 恢复后 ptr 可能失效！
    // 其他协程可能修改了 container
    use_ptr(ptr);  // 危险！
}
```

### 解决方案

在 `co_await` 恢复后重新获取：

```cpp
// 安全示例
auto handle_data() -> net::awaitable<void> {
    auto& container = get_container();

    co_await async_read(socket, buffer);

    // 恢复后重新获取
    auto& container_after = get_container();
    auto* ptr = &container_after.front();
    use_ptr(ptr);  // 安全
}
```

### 使用智能指针

对于跨协程共享的对象，使用智能指针：

```cpp
auto process_connection(connection_ptr conn) -> net::awaitable<void> {
    // shared_ptr 保持连接存活
    co_await conn->read_some(buffer);
    co_await conn->write_some(data);
    // conn 自动释放
}
```

## 迭代器失效处理

### erase 后更新迭代器

`erase()` 后必须使用返回值更新迭代器：

```cpp
// 正确示例
auto handle_remove() -> void {
    auto it = container.begin();
    while (it != container.end()) {
        if (should_remove(*it)) {
            it = container.erase(it);  // 使用返回值
        } else {
            ++it;
        }
    }
}
```

### 错误示例

```cpp
// 错误：erase 后迭代器失效
auto handle_remove_bad() -> void {
    for (auto it = container.begin(); it != container.end(); ++it) {
        if (should_remove(*it)) {
            container.erase(it);  // it 失效！
            // ++it 未定义行为
        }
    }
}
```

### 在协程中特别注意

```cpp
// 危险：挂起期间容器被修改
auto process_items() -> net::awaitable<void> {
    auto& items = get_items();
    for (auto it = items.begin(); it != items.end(); ++it) {
        co_await process_item(*it);  // 挂起

        // 恢复后 it 可能失效！
        // 其他协程可能插入/删除了元素
    }
}

// 安全：使用索引或重新获取
auto process_items_safe() -> net::awaitable<void> {
    for (size_t i = 0; i < get_items().size(); ) {
        co_await process_item(get_items()[i]);
        // 重新检查大小
        if (i < get_items().size()) {
            ++i;
        }
    }
}
```

## 引用失效

### 问题

协程挂起期间，引用指向的对象可能被修改或销毁：

```cpp
auto bad_reference() -> net::awaitable<void> {
    std::string local = "data";
    std::string_view view = local;  // 指向 local

    co_await async_operation();  // 挂起

    // local 可能已失效（如果协程迁移到其他线程）
    use_view(view);  // 危险！
}
```

### 解决方案

1. **拷贝数据**：将需要跨挂起点的数据拷贝到堆上
2. **使用智能指针**：通过 shared_ptr 共享所有权
3. **避免引用**：在挂起点之前完成所有引用访问

```cpp
auto safe_reference() -> net::awaitable<void> {
    auto data = std::make_shared<std::string>("data");

    co_await async_operation();

    use_data(*data);  // 安全
}
```

## 生命周期检查清单

在编写协程代码时，检查以下事项：

- [ ] `co_spawn` lambda 按值捕获 `self`
- [ ] `co_await` 恢复后重新获取指针/迭代器
- [ ] `erase()` 使用返回值更新迭代器
- [ ] 跨协程共享对象使用 `shared_ptr`
- [ ] 避免引用栈上的变量跨挂起点

## 相关文档

- [[overview]] - 编码规范概览
- [[coroutine]] - 协程约定
- [[dev/coding/coroutine|coroutines]] - C++23 协程详解