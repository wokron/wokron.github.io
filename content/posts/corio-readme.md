+++
title = "Corio - 基于 Asio 的 C++20 协程框架"
tags = ["Coroutine", "Asio", "C++"]
categories = ["网络"]
date = 2025-01-21T17:37:45+08:00
+++

最近研究了一下协程和网络编程，结合 C++20 的 Coroutine 和 Asio 编写了一个轻量级的协程框架 Corio。Corio 与 Asio 无缝集成，提供了多线程运行时支持和灵活的协程控制接口。

项目仓库：[wokron/corio](https://github.com/wokron/corio)。

## 安装

corio 是一个 Header-only Library。因此只需要将本仓库 `include/` 路径下的一系列头文件放在你的项目中指定位置，并将该位置添加到编译器的包含路径中即可。

以 CMake 为例，为了保持项目的模块化，可以选择将本项目作为 git 子模块。

```shell
git submodule add https://github.com/wokron/corio.git ./your/path/to/corio
```

随后在 `CMakeLists.txt` 中引入 corio。

```cmake
add_library(corio INTERFACE)
target_include_directories(corio INTERFACE ./your/path/to/corio/include)
target_link_libraries(corio INTERFACE ${ASIO_LIBRARY})
```

> 如前所述，Corio 依赖了 Asio。因此为了使用 Corio，还需安装 Asio（非 Boost 版本）。

最后，在你的代码中包含 `corio.hpp` 头文件以引入 corio。corio 的所有功能均位于 `corio::` 命名空间下。

```cpp
#include <corio.hpp>
```

## 使用

### 协程类型

#### lazy

`corio::Lazy<T>` 是 corio 库的核心。通过将函数的返回值设定为 `Lazy<T>`，我们将该函数定义为协程。在 `Lazy<T>` 所定义的协程中，不再使用 `return`，而是使用 `co_return` 以便从协程中返回。其中 `T` 为该协程的返回值类型。

```cpp
corio::Lazy<void> f1() {
    co_return;
}

auto f2 = []() -> corio::Lazy<int> {
    co_return 1;
}
```

`Lazy` 是**惰性**的，这意味着当我们 “调用” `Lazy` 所修饰的函数时，该函数并不会立即执行，而是会返回 `Lazy` 类型的实例。当对实例使用 `co_await` 运算符的时候，该协程才会运行。此时类似于一般的函数调用。

```cpp
corio::Lazy<int> f();

corio::Lazy<void> f2() {
    int r1 = co_await f();
    
    auto lazy = f();
    int r2 = co_await lazy;
}
```

> 只有在协程中才能使用 `co_await` 调用另一协程。

#### generator

迭代器 `corio::Generator<T>` 是 corio 中的另一种协程类型。在 `Generator<T>` 中可以使用 `co_yield` 关键字返回一 `T` 类型的值。返回之后该 `Generator` 并不销毁，而是可以从上一次返回处重新运行，直到遇到 `co_return`。

和 `Lazy` 类似，`Generator` 也是惰性的。我们需要使用 `co_await` 来使 `Generator` 恢复执行。此时 `co_await` 的返回值类型为 `bool`，表示 “此次执行是否遇到了新的 `co_yield`”。如果遇到了新的 `co_yield`，则可以从 `gen.current()` 方法中获取最新的 `co_yield` 返回值。

很容易想到 `Generator` 的遍历方法。

```cpp
corio::Generator<int> func_gen();

auto gen = func_gen();
while (co_await gen) {
    int v = gen.current();
}
```

corio 中额外提供了宏 `CORIO_ASYNC_FOR()` 和协程函数 `async_for_each()` 以简化迭代器的遍历。

```cpp
CORIO_ASYNC_FOR(int v, func_gen()) {
    // ...
}

co_await corio::async_for_each(func_gen(), [](int v) {
    // ...
});
```

### 协程入口

#### run

协程只能由协程调用，因此在所有协程之上需要协程入口以分离同步程序和异步程序。`corio::run()` 提供了开箱即用的协程入口。该函数接受并运行一 `Lazy` 类型的协程，阻塞直到该协程运行结束。如果协程有返回值，`run()` 函数也会将其返回。

`run()` 函数内部默认会根据当前 CPU 的核心数启动**多个线程**，由该多线程运行时处理异步程序的执行。但也可通过设置 `multi_thread = false` 以使用单线程运行时。

```cpp
corio::Lazy<int> f();

int r = corio::run(f());
// int r = corio::run(f(), /*multi_thread=*/false);
```

#### block_on

如果希望自行设定运行时，可以使用 `corio::block_on()` 函数。该函数的第一个参数应当传入一个可以转换为 `asio::any_io_executor` 的 `executor`。用户应当保证该 `executor` 上程序的执行是**串行**的。一些可能的 `executor` 包括：

- 在单线程上运行的 `asio::io_context` 的 `executor`。
- 线程数为 1 的 `asio::thread_pool` 的 `executor`。
- 被包装在 `asio::strand` 中的在多线程上运行的 `asio::io_context` 的 `executor`。
- 被包装在 `asio::strand` 中的线程数大于 1 的 `asio::thread_pool` 的 `executor`。
- 被包装在 `asio::any_io_executor` 当中的上述 `executor`。

```cpp
corio::Lazy<void> f();

asio::io_context io_context;
std::jthread t([&]() { io_context.run(); });
corio::block_on(io_context.get_executor(), f()); // ok

asio::thread_pool pool(1);
corio::block_on(pool.get_executor(), f()); // ok

asio::thread_pool pool(4);
corio::block_on(asio::make_strand(pool.get_executor()), f()); // ok

asio::thread_pool pool(4);
corio::block_on(pool.get_executor(), f()); // wrong
```

> 如果你的 `executor` 是一个被包装在 `strand` 中的多线程 `executor`。最好不要将其包装在 `any_io_executor` 之后再传入 `block_on()`，否则协程运行时将退化为单线程。

### 并发

#### spawn

`corio::spawn()` 和 `corio::spawn_backgroud()` 用于创建并运行一个**任务**。任务是一组顺序执行的协程。在协程中，任务的角色类似于同步程序中的线程。不同任务的执行是并发的（对于 corio 来说甚至是并行的！！）。

`spawn()` 和 `spawn_backgroud()` 的不同之处在于 `spawn()` 返回了一个不可忽略的 `corio::Task<T>` 实例。该实例可用于控制 `spawn()` 所创建的任务。具体内容见下一小节。

`spawn()` 和 `spawn_backgroud()` 分别提供了非协程版本和协程版本。非协程版本的函数签名和 `block_on()` 相同。不同之处在于调用 `spawn()` 和 `spawn_backgroud()` 时不会阻塞代码直到任务结束。

```cpp
corio::Lazy<void> f();
corio::Lazy<void> g();

asio::thread_pool pool(4);

auto t = corio::spawn(asio::make_strand(pool.get_executor()), f());
corio::spawn_background(asio::make_strand(pool.get_executor()), g());
// Here f() and g() are running in parallel
pool.join();
```

当需要在协程内创建并发任务时，请使用协程版本的 `spawn()` 和 `spawn_backgroud()`。这一版本不再需要传入 `executor`，corio 会根据当前协程所处的运行时自动决定新任务的运行时。

```cpp
corio::Lazy<void> f();
corio::Lazy<void> g();

corio::Lazy<void> h() {
    auto t = co_await corio::spawn(f());
    co_await corio::spawn_background(g());
    // Here f(), g() and h() are running in parallel
    // ...
    co_return;
}

corio::run(h());
```

#### task

如前所述，调用 `spawn()` 后会返回 `corio::Task<T>` 实例。其中 `T` 与协程的返回值保持一致。可以使用 `co_await` 等待 `Task` 对象。此时当前协程会挂起，直到 `Task` 对象所对应的协程完成。

```cpp
corio::Lazy<int> f();

corio::Lazy<void> g() {
    corio::Task<int> task = co_await corio::spawn(f());
    int r = co_await task;
    // ...
}
```

调用 `task.abort()` 函数可以取消对应任务。调用该函数后对应任务将在下一次挂起时取消。所有协程栈上的对象都将析构。

调用 `task.get_abort_handle()` 函数将返回一 `corio::AbortHandle<T>` 实例。调用该实例的 `abort()` 方法同样可以取消对应任务。如果此时 `task` 正在被等待，则会抛出 `corio::CancellationError` 异常。

```cpp
corio::Lazy<int> f();

corio::Lazy<void> g(corio::AbortHandle<T> h) {
    h.abort();
    co_return;
}

corio::Lazy<void> h() {
    corio::Task<int> task = co_await corio::spawn(f());
    co_await corio::spawn_background(g(task.get_abort_handle()));
    try {
        int r = co_await task;
    } catch (const corio::CancellationError& e) {
        // ...
    }
    // ...
}
```

> 需要注意，当 `Task<T>` 实例析构的时候，其将**默认取消对应任务**。因此如果不希望任务被取消，应当使用 `spawn_background()`。或者可以通过 `task.detach()` 方法放弃对此任务的控制权。

### 可等待对象

#### awaitable

**可等待对象**是满足 `awaitable` concept 的对象。在 corio 中，大部分可以应用 `co_await` 的对象都是可等待对象。

```cpp
template <typename Awaiter, typename Promise = void>
concept awaiter =
    requires(Awaiter awaiter, std::coroutine_handle<Promise> handle) {
        { awaiter.await_ready() };
        { awaiter.await_suspend(handle) };
        { awaiter.await_resume() };
    };

template <typename Awaitable, typename Promise = void>
concept awaitable = awaiter<Awaitable, Promise>
    || requires(Awaitable awaitable) {
        { awaitable.operator co_await() };
    }
    || requires(Awaitable awaitable) {
        { operator co_await(awaitable) };
    };
```

#### any_awaitable

`corio::AnyAwaitable<Ts...>` 可以接受任意可等待对象，只需保证其应用 `co_await` 后的返回值为类型 `Ts...` 中之一。

如果 `Ts...` 中类型多于一个，则 `AnyAwaitable` 应用 `co_await` 后的类型为 `std::variant`。若 `Ts...` 中包含 `void`，则其被替换为 `std::monostate`；若 `Ts...` 中同一类型出现了多次，则只保留第一次出现的。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();

corio::AnyAwaitable<void, int> aw1 = f();
auto r1 = std::get<int>(co_await aw1);

corio::AnyAwaitable<void, int> aw2 = g();
auto r2 = std::get<std::monostate>(co_await aw2);

corio::AnyAwaitable<void, int> aw3 = co_await corio::spawn(f());
auto r3 = std::get<int>(co_await aw3);
```

### 协程控制

#### executor

可以通过对 `corio::this_coro::executor` 应用 `co_await` 来获取当前协程所运行的 `executor`。返回值类型为 `asio::any_io_executor`。返回的 `executor` 可以传递给 Asio 的 IO 对象。

```cpp
corio::Lazy<void> f() {
    auto ex = co_await corio::this_coro::executor;
    asio::steady_timer timer(ex, 100us);
    // ...
}
```

> 请不要将此处的 `executor` 传递给 `spawn()` 或 `spawn_backgroud()`。对于多线程运行时，这将导致新创建的任务和当前任务运行在同一个 `strand` 中。

#### yield

可以通过对 `corio::this_coro::yield` 应用 `co_await` 来使当前任务主动放弃运行权。此外也可以通过调用 `corio::this_coro::do_yield()` 函数实现相同功能。`yield` 的效率较高，但是并不是可等待对象；后者则会返回一可等待对象。如无明确需要，请使用 `yield`。

```cpp
corio::Lazy<void> f() {
    co_await corio::this_coro::yield;

    auto aw = corio::this_coro::do_yield();
    co_await aw;
}
```

#### sleep

可以对 `std::chrono::duration` 或 `std::chrono::time_point` 对象应用 `co_await` 来等待特定时间。此外也可以通过调用 `corio::this_coro::sleep_for()` 和 `corio::this_coro::sleep_until()` 函数实现相同功能。其区别与 `yield` 类似。

```cpp
using namespace std::chrono_literals;

corio::Lazy<void> f() {
    co_await 1s;
    co_await (std::chrono::steady_clock::now() + 1s);

    auto aw1 = corio::this_coro::sleep_for(1s);
    co_await aw1;
    auto aw2 = corio::this_coro::sleep_until(std::chrono::steady_clock::now() + 1s);
    co_await aw2;
}
```

另外也可以对 Asio 中的各类计时器（如 `asio::steady_timer`）应用 `co_await` 操作。

```cpp
using namespace std::chrono_literals;

corio::Lazy<void> f() {
    auto ex = co_await corio::this_coro::executor;
    asio::steady_timer timer(ex, 1s);
    co_await timer; // sleep 1s
    timer.expires_after(1s);
    co_await timer; // sleep 1s
}
```

#### roam

可以对 `executor` 对象应用 `co_await` 来使当前协程任务切换到新的运行时上。对 `executor` 的要求和 `block_on()` 函数相同。这有助于为协程任务的 IO-Bound 和 CPU-Bound 部分选择更加合适的 `executor`。此外也可以通过调用 `corio::this_coro::roam_to()` 函数实现相同功能。其区别与 `yield` 类似。

```cpp
corio::Lazy<void> io_bound_func();
void cpu_bound_func();

asio::thread_pool p1(1);
asio::thread_pool p2(16);

auto f = [&]() -> corio::Lazy<void> {
    co_await io_bound_func();

    co_await asio::make_strand(p2.get_executor());

    cpu_bound_func(); // Run cpu-bound code in multi-thread runtime

    auto aw = corio::this_coro::roam_to(p1.get_executor());
    co_await aw;

    co_await io_bound_func();
}

corio::block_on(p1.get_executor(), f());
```

#### future

可以对 `std::future` 对象应用 `co_await` 异步等待 `future` 的完成。这并不会阻塞当前运行时的任何线程。`co_await` 的返回值等同于 `future.get()` 的返回值。

```cpp
corio::Lazy<void> f() {
    std::future<int> fut = std::async(std::launch::async, []() -> int { 
        std::this_thread::sleep_for(10s);
        return 42;
    });
    int r = co_await fut;
    // ...
}
```

> `co_await future` 不会阻塞当前运行时内的线程，因为阻塞被转移到了 `asio::system_executor` 的线程之中。

### 同步

#### gather

`corio::gather()` 函数可以并行等待多个可等待对象。当所有可等待对象均完成后（包括成功返回或产生异常）返回。`gather()` 包含两个重载，分别接受可变数量的可等待对象或一包含可等待对象的**可迭代**对象。

`gather()` 的返回值是 `std::tuple<corio::Result<Ts>...>` 或 `std::vector<corio::Result<T>>`。其中 `Ts...` 和 `T` 为所传入的可等待对象应用 `co_await` 后的返回值。

> `corio::Result<T>` 是一个容器，类似于 Rust 的 `Result`，其中可能包含 `T` 类型的取值或一个异常。可以通过 `result.exception() != nullptr` 来判断 `Result` 中所包含的是否为异常。如果不存在异常，则 `result.result()` 将返回 `Result` 中所存储的值的引用（若 `T = void` 则无返回值）；如果存在异常，则 `result.result()` 会将该异常抛出。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();
corio::Lazy<int> h();

// std::tuple<Result<int>, Result<void>>
auto [r1, r2] = co_await corio::gather(
    co_await corio::spawn(f()),
    co_await corio::spawn(g()));


std::vector<corio::Task<int>> tasks;
tasks.push_back(co_await corio::spawn(f()));
tasks.push_back(co_await corio::spawn(h()));

// std::vector<Result<int>>
auto r3 = co_await corio::gather(tasks);
```

另外对于可变参数版本的 `gather()`，corio 提供了对 `&` 的运算符重载。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();

using corio::awaitable_operators::operator&;

auto [r1, r2] = co_await (corio::spawn(f()) & corio::spawn(g()));
```

#### try_gather

和 `gather()` 类似，`corio::try_gather()` 同样可以并行等待多个可等待对象，也包含了两个重载。不同之处在于，当某一个可等待对象第一个抛出异常时，`try_gather()` 将提前返回。若提前返回，剩下还未完成的 `co_await` 操作将会被取消。`try_gather()` 的返回值类型是 `std::tuple<Ts...>` 或 `std::vector<T>`。若 `T = void`，则 `T` 被替换为 `std::monostate`。

> 若采用移动语义将可等待对象移动到 `try_gather` 当中，则不仅 `co_await` 操作将会被取消，可等待对象本身也会被析构。对于 `Task<T>`，需要注意传入 `task` 和 `std::move(task)` 的区别。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();
corio::Lazy<int> h();

// int, std::monostate
auto [r1, r2] = co_await corio::try_gather(
    co_await corio::spawn(f()),
    co_await corio::spawn(g()));


std::vector<corio::Task<int>> tasks;
tasks.push_back(co_await corio::spawn(f()));
tasks.push_back(co_await corio::spawn(h()));

// std::vector<int>
auto r3 = co_await corio::gather(tasks);
```

对于可变参数版本的 `try_gather()`，corio 提供了对 `&&` 的运算符重载。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();

using corio::awaitable_operators::operator&&;

auto [r1, r2] = co_await (corio::spawn(f()) && corio::spawn(g()));
```

#### select

`corio::select()` 并行等待多个可等待对象，当可等待对象第一个完成时（包括成功返回或产生异常）返回，并取消剩下还未完成的 `co_await` 操作。`select()` 同样包含了两个重载。`select()` 的返回值类型为 `std::variant<Ts...>` 或 `std::pair<std::size_t, T>`。当 `T = void` 时，`T` 被替换为 `std::monostate`。

> 返回值类型为 `std::variant<Ts...>` 时，可以通过 `var.index()` 判断首先完成的可等待对象为第几个传入的参数。
> 返回值类型为 `std::pair<std::size_t, T>` 时，`std::size_t` 指示了可迭代容器中的第几个元素首先完成。

```cpp
corio::Lazy<int> f();
corio::Lazy<void> g();
corio::Lazy<void> h();

// std::variant<std::monostate, int, std::monostate>
auto r = co_await corio::select(
    corio::this_coro::sleep_for(1ms),
    co_await corio::spawn(f()),
    co_await corio::spawn(g()),
);
if (r.index() == 0) {
    // timeout
} else {
    // not timeout
}

std::vector<corio::Task<void>> tasks;
tasks.push_back(co_await corio::spawn(g()));
tasks.push_back(co_await corio::spawn(h()));
// std::pair<std::size_t, std::monostate>
auto [index, _] = co_await corio::select(tasks);
```

> 和 `try_gather()` 类似，也请注意 `task` 和 `std::move(task)` 的区别。

对于可变参数版本的 `select()`，corio 提供了对 `||` 的运算符重载。

```cpp
using corio::awaitable_operators::operator||;

corio::Lazy<int> f();
corio::Lazy<void> g();

auto r = co_await (
    corio::this_coro::sleep_for(1ms)
    || corio::spawn(f())
    || corio::spawn(g())
);
```

### 与 Asio 结合

Asio 提供了丰富的异步 IO 接口。corio 提供了 Completion Token `corio::use_corio` 实现了与 Asio 的适配。

以 `asio::steady_timer` 为例。经典的基于回调的异步代码如下：

```cpp
asio::steady_timer timer(io_context, 500us);
timer.async_wait([]() {
    // do something...
});
// ...
io_context.run();
// ...
```

而在 corio 中，我们使用 `use_corio` 使 `async_wait` 返回一个可等待对象 `corio::Operation<...>`。

```cpp
corio::Lazy<void> f() {
    auto ex = co_await corio::this_coro::executor;
    asio::steady_timer timer(ex, 500us);
    auto op = timer.async_wait(corio::use_corio);
    co_await op;
    // do something...
}
```

我们也可以使用 `use_corio_t::as_default_on_t` 或 `use_corio.as_default_on` 将 `corio::use_corio` 作为异步操作的默认 Complete Token。

```cpp
namespace corio {
    using steady_timer = use_corio_t::as_default_on_t<asio::steady_timer>;
}

corio::Lazy<void> f() {
    auto ex = co_await corio::this_coro::executor;
    // corio:: instead of asio::
    corio::steady_timer timer(ex, 500us);
    co_await timer.async_wait();
    // do something...
}

corio::Lazy<void> g() {
    auto ex = co_await corio::this_coro::executor;
    auto timer = corio::use_corio.as_default_on(asio::steady_timer(ex, 500us));
    co_await timer.async_wait();
    // do something...
}
```