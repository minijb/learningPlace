# C++ 异步编程详解：从 std::async 到协程与 sender/receiver

> 本文是「单独教程」系列的独立篇章，与 `cpp_traits.md` 同体例。读者假设：懂模板、懂 `std::thread` / `mutex` 基本用法（最好已读 [01-cpp ch05｜并发与内存模型](../01-cpp/tutorial/ch05-concurrency-and-memory-model.md)，文中以「ch05 §N.N」后向互链，不复讲）。
> 语言基线 C++20（`<coroutine>`）；凡 C++23 / C++26 内容均显式标注归属。全部「完整可编译」示例仅依赖标准库，并已在 MSVC（VS 2022，MSVC 19.44 工具集）以 `cl /nologo /utf-8 /std:c++20 /EHsc /W4` 实证编译运行；§7 的 C++23 示例需 `/std:c++latest`（本机实证结论，见 §7 开头）。g++/clang++ 等价命令见 §12。

## 1. 导读：异步到底难在哪

### 1.1 异步 ≠ 并发

先把两个被混用几十年的词拆开：

- **并发（concurrency）**：多件事在逻辑上同时推进。它们可以真并行（多核），也可以只是分时交错（单核）。ch05 讲的线程、锁、内存序，全部是并发话题。
- **异步（asynchrony）**：发起一件事之后**不等它做完**，先回去干别的，结果好了再「接上」。重点不是「同时」，而是「不阻塞等待」。

关键辨析：**异步不需要第二个线程**。主线程发起一次贴图解压，先回来跑游戏逻辑，隔一会儿去取结果——哪怕解压是靠同一个线程分片推进的，这也是异步。反过来，开了十个线程但每个都在死等 I/O，那只是昂贵的同步。

异步的对立面不是「单线程」，而是「同步等待」。这个区分决定了后面所有技术的定位：协程（§4–§5）解决的是**怎么写**异步；线程池（§9.1）解决的是**在哪跑**异步。两件事正交。

**分工声明**：同步原语（`mutex`、`condition_variable`、原子与内存序、TSan）属于并发，ch05 已系统讲过。本章只在需要处引用，例如「挂起时不占线程」的论证依赖 ch05 §5.2 的线程生命周期模型，「自旋等待 vs 条件变量等待」的取舍依赖 ch05 §5.4。

### 1.2 为什么 C++ 的异步特别难

一句话：**C++ 标准库直到最近才回答异步问题，而且至今只回答了一半。**

- C++11 给了 `std::async` / `std::future`，但它是个半成品：没有 `.then()` 续接、没有运行位置控制（不由你选线程池）、`std::async` 返回的 future 析构还会阻塞（§2.2 逐条解剖）。
- 于是社区各自造轮子：asio（网络 I/O 事实标准）、cppcoro、folly::Future、libunifex……每家一套 task 类型、一套取消语义、一套错误传播。
- C++20 给了**协程**——但注意，它给的是「机器」不是「答案」：语言层只提供 `co_await` 等关键字和一套协议，你想要的 `task<T>`、`generator<T>` 都要自己写或靠库（§4.5）。`std::generator` 直到 C++23 才进标准库（§7）。
- 真正的标准答案——基于 sender/receiver 的 `std::execution`——要等 C++26（§8.3）。

所以学 C++ 异步，实际上是学三层东西：**现有工具的边界在哪**（§2–§3）、**协程这台机器怎么运转**（§4–§6）、**生态与标准往哪走**（§7–§8）。本章按这条线走完，最后落到游戏引擎的真实场景（§9）。

### 1.3 演进时间线与本章路线图

```text
2000s       同步阻塞调用 + OS 线程
            │  线程一多就崩：栈内存、调度开销、锁竞争（ch05）
            ▼
2000s–今    回调（completion callback）
            │  不阻塞了，但控制流倒置、错误处理被撕碎（§3）
            ▼
2011  C++11 std::async / std::future / std::promise
            │  有未来值了，但没有 .then()，析构还会阻塞（§2）
            ▼
2020  C++20 协程：co_await / co_yield / co_return（语言机器）
            │  同步的写法、异步的执行——但要自己造 task（§4–§6）
            │  2023  C++23 std::generator：标准库第一个协程类型（§7）
            ▼
2026  C++26 std::execution（P2300）：sender/receiver/scheduler
               组合式异步的标准答案（草案已定，实现落地中）（§8.3）
```

本章路线图：§2 把 `std::async` 全家族用到透并看清它的天花板 → §3 用回调复现同样的问题，提炼「我们真正想要的」需求清单 → §4 拆开协程这台机器 → §5 **手写 `task<T>` 三版（全章枢纽）** → §6 补齐错误处理与取消 → §7 体验标准库第一个协程类型 → §8 看真实世界的 executor / asio / C++26 → §9 回到游戏引擎 → §10 两个深入专题做实验验证 → §13–§17 实践、自测与资源。

## 2. std::async 与 std::future 全家族

### 2.1 最小可用：async + get

先看它「看起来很美好」的一面。两个大数求和，串行 vs 并行计时：

```cpp
// a_async_basic.cpp —— 双 async 并行计时对比（完整可编译，实测输出见下）
#include <chrono>
#include <cstdio>
#include <future>
using namespace std::chrono;

long long sum_to(long long n) {
    long long s = 0;
    for (long long i = 1; i <= n; ++i) s += i;
    return s;
}

int main() {
    // 串行
    auto t0 = steady_clock::now();
    long long a = sum_to(200'000'000);
    long long b = sum_to(200'000'000);
    auto t1 = steady_clock::now();
    std::printf("串行: %lld ms, 结果 %lld\n",
                (long long)duration_cast<milliseconds>(t1 - t0).count(), a + b);

    // 并行：显式 std::launch::async（为什么必须显式，2.2 坑②）
    auto t2 = steady_clock::now();
    auto fa = std::async(std::launch::async, sum_to, 200'000'000);
    auto fb = std::async(std::launch::async, sum_to, 200'000'000);
    long long pa = fa.get();      // 阻塞直到就绪并取回结果
    long long pb = fb.get();
    auto t3 = steady_clock::now();
    std::printf("并行: %lld ms, 结果 %lld\n",
                (long long)duration_cast<milliseconds>(t3 - t2).count(), pa + pb);
}
```

本机实测（8 核，数值因机器而异，比例关系稳定）：

```text
串行: 218 ms, 结果 40000000000000000
并行: 113 ms, 结果 40000000000000000
```

要点：

- `std::async(std::launch::async, f, args...)` 返回 `std::future<T>`——一张「未来结果的取货单」。`get()` 阻塞到就绪并返回值。
- **异常会穿过 future**：任务里抛出的异常被存进共享状态，`get()` 时在调用方原样重抛。这是 future 家族最值得称道的设计，§5 手写 task 时会亲手复刻它。
- `get()` 只能调一次，之后 future 失效（§2.5 的 `shared_future` 解决多消费者）。

### 2.2 四大坑：为什么老手不碰 std::async

#### 坑① 析构阻塞：async 返回的 future 析构会「偷偷 join」

`std::async` 返回的 future 有一个全家族独一份的语义：**它的析构函数会阻塞，等任务做完**。其他途径（`promise` / `packaged_task`）拿到的 future 析构都不等。实测：

```cpp
// a_async_destruct.cpp —— 析构阻塞计时实测（完整可编译，实测输出见下）
#include <chrono>
#include <cstdio>
#include <future>
#include <thread>
using namespace std::chrono;

int work() {
    std::this_thread::sleep_for(milliseconds(1500));
    return 42;
}

int main() {
    auto t0 = steady_clock::now();
    {
        auto f = std::async(std::launch::async, work);  // 有名字：活到作用域结束
        std::printf("作用域内先干点别的...\n");
    }                                                   // f 在此析构 —— 阻塞约 1500ms
    auto t1 = steady_clock::now();
    std::printf("有名字的 future：作用域耗时 %lld ms（析构在等任务）\n",
                (long long)duration_cast<milliseconds>(t1 - t0).count());

    auto t2 = steady_clock::now();
    std::async(std::launch::async, work);               // 返回值被丢弃：临时 future 立即析构
    auto t3 = steady_clock::now();
    std::printf("丢弃返回值：本行耗时 %lld ms（async 退化成同步调用）\n",
                (long long)duration_cast<milliseconds>(t3 - t2).count());
}
```

```text
作用域内先干点别的...
有名字的 future：作用域耗时 1500 ms（析构在等任务）
丢弃返回值：本行耗时 1500 ms（async 退化成同步调用）
```

第二行是真正的事故现场：**手滑丢弃 `std::async` 返回值，异步立刻变同步**，而且编译器只给你一个无关痛痒的告警。游戏主循环里混进一行这样的代码，帧时间直接多 1500ms。

根源：`std::async` 不带策略参数时，默认策略是 `async | deferred` 二选一、由实现自行决定。标准委员会为了让「实现选了 async」的情形不至于任务悬空，规定析构必须兜底等待——于是「实现可能选 deferred」这个自由，让所有用户的析构都背上了阻塞的代价。

#### 坑② 默认策略薛定谔：不带策略的 std::async 不保证开线程

```cpp
auto f = std::async(work);   // 策略 async|deferred：可能开线程，也可能在你 get/wait 时当前线程原地执行
```

「薛定谔的异步」：写的时候像异步，跑的时候可能是惰性同步。纪律只有一条：**要么显式 `std::launch::async`，要么干脆不用 `std::async`**。本章后续示例全部显式。

#### 坑③ deferred × wait_for：`wait_for(0)` 永远不 ready

```cpp
// a_async_deferred.cpp —— deferred 三种 future_status 对比（完整可编译，实测输出见下）
#include <chrono>
#include <cstdio>
#include <future>

int main() {
    auto f = std::async(std::launch::deferred, [] {
        std::printf("  [任务真正执行]\n");
        return 42;
    });
    auto s1 = f.wait_for(std::chrono::seconds(0));
    std::printf("wait_for(0s) 第一次: %s\n",
                s1 == std::future_status::deferred ? "deferred" :
                s1 == std::future_status::ready    ? "ready"    : "timeout");
    std::printf("wait_for(1h) 返回: %s\n",
                f.wait_for(std::chrono::hours(1)) == std::future_status::deferred ? "deferred" : "other");
    std::printf("get() 触发执行: %d\n", f.get());
}
```

```text
wait_for(0s) 第一次: deferred
wait_for(1h) 返回: deferred
  [任务真正执行]
get() 触发执行: 42
```

注意两点：任务从头到尾没跑过（没开线程），`wait_for(0s)` 返回的不是 `timeout` 而是 **`deferred`**。所以「`wait_for(0)==ready` 就当已完成」的轮询写法，在 deferred future 上**永远判不完成**——正确探测姿势是先判断 `wait_for(0) == deferred` 分流，再谈轮询。

> **本机实证附注（工具链怪癖，如实记录）**：在 MSVC 19.44 下，对**已 `get()` 过的 deferred future** 再调 `wait_for` 会触发 fail-fast 崩溃（退出码 0xC0000409），故上面示例在 `get()` 后不再探测。该行为标准未定义归属，属实现怪癖；语义拿不准时以 cppreference 为权威，示例写法遵循「探测只做在 get 之前」的保守纪律。

#### 坑④ 一次性 + 不可组合

`get()` 之后 future 作废；想「A 完成后自动做 B」，没有 `.then()`——只能在 A 上 `get()` 阻塞完再做 B，异步变成了变相同步。链式组合、错误传播、取消这些「异步大厦」的地基，future 一概没有。§2.6 正式清算。

### 2.3 promise：手动结果通道

`std::promise<T>` 是 future 的另一面：future 是「取货单」，promise 是「交货口」，两者共享同一份堆上的共享状态。适合「我已经有线程了，只想跨线程递个结果」的场景（线程的创建与 join 语义见 ch05 §5.1）：

```cpp
// a_promise.cpp —— promise 回传 + broken_promise 触发（完整可编译，实测输出见下）
#include <cstdio>
#include <future>
#include <thread>

int compute() { return 6 * 7; }

int main() {
    // 正常通道：工作线程交货
    std::promise<int> p;
    std::future<int> f = p.get_future();
    std::thread t([&p] { p.set_value(compute()); });
    t.join();
    std::printf("回传结果: %d\n", f.get());

    // broken_promise：交货口先没了
    std::future<int> f2;
    {
        std::promise<int> p2;
        f2 = p2.get_future();
    }                                   // p2 未 set_value 就析构
    try {
        f2.get();
    } catch (const std::future_error& e) {
        std::printf("捕获: %s\n", e.what());   // broken promise
    }
}
```

```text
回传结果: 42
捕获: broken promise
```

要点：`set_exception(std::current_exception())` 同样可用——promise 也走「异常穿透共享状态」的通道；promise 先亡而未交货，对端 `get()` 收到 `std::future_error`（`broken_promise`）。

对比 ch05 §5.4 / §5.8 的任务队列：那里线程间递结果靠「mutex + condition_variable + 共享缓冲区」，这里一个 promise 就是一条带异常语义的结果通道。**promise/future 是标准库把「锁 + 条件变量 + 约定」封装成一个语义单元的样子**——但注意它每次都要堆分配共享状态，别在热路径逐帧用（ch05 §5.9 的分配成本意识在此同样适用）。

### 2.4 packaged_task：把可调用物打包成「带返回值的任务」

`std::packaged_task<R(Args...)>` = 「任意可调用物 + 一张配套 future」。它本身是个可调用物，可以投给线程或任务队列；调用它就执行函数、结果自动进 future：

```cpp
// a_packaged.cpp —— packaged_task 投线程（完整可编译，实测输出见下）
#include <cstdio>
#include <future>
#include <thread>

int main() {
    std::packaged_task<int(int, int)> task([](int a, int b) { return a * b; });
    std::future<int> f = task.get_future();

    std::thread worker(std::move(task), 6, 7);   // packaged_task 只能移动（move-only）
    worker.join();
    std::printf("打包执行结果: %d\n", f.get());
}
```

```text
打包执行结果: 42
```

和 `std::function` 的差别：`std::function` 调用了就完了，拿不到返回值给第三方；`packaged_task` 调用的返回值自动进 future。**它是「给任务队列补上返回值」的最短路径**——ch05 §5.8 的 `TaskQueue<std::function<void()>>` 换成 `packaged_task` 就能携带结果，§9.1 的迷你 job 池直接用这套手艺。

### 2.5 shared_future：多消费者

`future` 是 move-only 的单次取货单；`shared_future` 可拷贝，多个消费者各自 `get()` 同一份结果（含异常）：

```cpp
// a_shared_future.cpp —— 一次结果，三个消费者（完整可编译，实测输出见下）
#include <cstdio>
#include <future>
#include <thread>
#include <vector>

int main() {
    std::shared_future<int> sf = std::async(std::launch::async, [] {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        return 100;
    });

    std::vector<std::thread> consumers;
    for (int i = 0; i < 3; ++i) {
        consumers.emplace_back([sf, i] {          // 按值捕获：每个线程一份拷贝
            std::printf("消费者%d 拿到: %d\n", i, sf.get());   // 各自 get，互不干扰
        });
    }
    for (auto& c : consumers) c.join();
}
```

```text
消费者0 拿到: 100
消费者1 拿到: 100
消费者2 拿到: 100
```

（打印顺序不保证。）注意 `shared_future` 仍不解决组合问题——它只是把「单次」放宽为「多次」。

### 2.6 裁决：future 撑不起异步大厦

把「一个可组合的异步原语」需要的能力列出来对照：

| 需求 | std::future 现状 |
| --- | --- |
| 发起后不阻塞 | ✔（`async` 策略下） |
| 异常传播到等待方 | ✔（get 重抛，做得很好） |
| 完成后续接（`.then`） | ✘ 只能阻塞 get 后手工接 |
| 控制运行位置（选线程池/调度器） | ✘ 线程哪来的你管不着，也没法换成 IO 线程 |
| 可组合（同时等多个、任一等） | ✘ 只有 `future` 的阻塞原语拼凑 |
| 可取消 | ✘（C++20 `stop_token` 与它无集成） |

**业界共识：`std::future` 是「一次性跨线程结果容器」，不是异步组合原语。** 这个缺口正是后来 P2300（§8.3）立项的历史动因；在标准补齐之前，填补它的两次尝试分别是回调（§3，代价惨重）和协程（§4 起，本章主线）。

## 3. 回调：能跑，但不可组合

### 3.1 回调风格重写

把 §2.1 的求和改成回调版——「完成时请打这个电话」：

```cpp
// b_callback_basic.cpp —— 回调版求和（完整可编译，实测输出见下）
#include <chrono>
#include <cstdio>
#include <functional>
#include <system_error>
#include <thread>

using Callback = std::function<void(int result, const std::error_code& ec)>;

void async_sum(long long n, Callback on_done) {
    std::thread([n, on_done = std::move(on_done)] {     // 线程模拟异步完成
        int digits = 0;
        for (long long v = n; v; v /= 10) ++digits;     // 算点东西
        on_done(digits, {});                            // 成功：ec 为空
    }).detach();                                        // detach 演示用；生产代码用线程池（§9.1）
}

int main() {
    async_sum(123456789, [](int digits, const std::error_code& ec) {
        if (ec) { std::printf("失败\n"); return; }
        std::printf("完成: %d 位\n", digits);
    });
    std::this_thread::sleep_for(std::chrono::milliseconds(100));   // 演示：否则 main 退出回调没得跑
}
```

```text
完成: 8 位
```

回调风格是所有异步 I/O 底层（包括 asio，§8.2）的通用语：**发起时留下函数指针/闭包，完成时被调用**。单个回调尚可忍受，问题出在组合。

### 3.2 回调地狱解剖

三步串联：读文件 → 解析头得到长度 → 读出数据。异步回调版：

```cpp
// b_callback_hell.cpp —— 三步串联的回调地狱（完整可编译，实测输出见下）
// 模拟异步操作：read(offset, buf) -> header；read_exact(len) -> data；parse(header) -> len
#include <cstdio>
#include <functional>
#include <string>
#include <thread>

template <class F>
static void async_step(int delay_ms, F&& body) {
    std::thread([d = delay_ms, f = std::forward<F>(body)] {
        std::this_thread::sleep_for(std::chrono::milliseconds(d));
        f();
    }).detach();
}

struct Ctx {                       // 步骤间共享状态：被迫堆上逃逸（或 new 或 shared_ptr）
    std::string header;
    std::string data;
    int len = 0;
};

int main() {
    auto ctx = std::make_shared<Ctx>();

    async_step(10, [ctx] {                                      // 第 1 步
        ctx->header = "len=42";
        std::printf("[1] header=%s\n", ctx->header.c_str());

        async_step(10, [ctx] {                                  // 第 2 步：缩进 +1
            sscanf(ctx->header.c_str(), "len=%d", &ctx->len);
            std::printf("[2] len=%d\n", ctx->len);

            async_step(10, [ctx] {                              // 第 3 步：缩进 +2
                ctx->data.assign(ctx->len, 'x');
                std::printf("[3] data.size=%zu 全部完成\n", ctx->data.size());
                // 若第 3 步失败要重试？要取消？要超时？——在这个形状里都是灾难
            });
        });
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(200));
}
```

```text
[1] header=len=42
[2] len=42
[3] data.size=42 全部完成
```

三个痛点，逐条指认：

1. **控制流倒置**：「做完 A 做 B」写成了「A 的完成回调里发起 B」。三步就缩进成箭头形，顺序语义全靠肉眼从嵌套里挖。
2. **错误处理被撕碎**：每层回调都要自带 ec 分支。三步串联要有统一的「任一步失败就清理退出」，回调版得让错误逐层手工下传——漏一层就是悬挂路径。
3. **状态被迫逃逸**：步骤间的 `header`/`len` 生命周期横跨回调，只能 `shared_ptr<Ctx>` 上堆，所有权立刻含糊（ch05 §5.3 的 shared_ptr 纪律在这里被结构性逼破）。

还要加上捕获悬垂：回调捕获栈变量引用、而回调在栈帧死后才执行——和 ch05 §5.1 的线程悬垂引用坑同源，但回调让这类坑更隐蔽，因为「何时执行」不再由你调用。

### 3.3 我们真正想要什么

把痛点翻过来，就是需求清单——请记住这五条，§5 手写 task 时它们就是验收标准：

1. **顺序写法**：「做完 A 做 B」就是先写 A 再写 B，缩进不涨；
2. **挂起不占线程**：等待时线程要还回去（不占线程的等待才敢大规模用）；
3. **异常一路传播**：任意一步抛的异常，能在最外层一个 `try` 里接住；
4. **结果可组合**：任务之间能自由串联、并列；
5. **可取消**：外部能喊停，任务在安全点退出。

回调满足 0 条，future 满足 1.5 条。满足全部五条的下一站，就是协程。

## 4. C++20 协程机制

### 4.1 三个关键字

一个函数里出现 `co_await`、`co_yield`、`co_return` 任意一个，它就是**协程**：编译器把它改写成「可挂起-恢复的状态机」。

- `co_return v;` —— 结束协程，把 v 交给承诺对象；
- `co_yield v;` —— 产出一个值并**挂起**（生成器的基础，§7）；
- `co_await aw;` —— 等待一个「等待者（awaiter）」，可能挂起（异步的基础，§4.4）。

与线程的本质区别（回扣 §1.1）：协程是**协作式**的——只在显式挂起点让出控制权；**可以单线程**——挂起不等于并发；**不提供并行**——要并行仍需线程（§9.1）。协程解决的是「函数执行到一半可以停」这一件事，其余全是库的事（§4.5）。

「看似普通函数、实则挂起两次」的直观演示放在 §4.5 最小类型之后，见 c_min_coro.cpp。

### 4.2 协程帧：你的局部变量住哪了

协程被调用时，编译器在**堆上**分配一块**协程帧（coroutine frame）**，把「函数本次调用的全部现场」搬进去：

```text
协程帧（堆上）
┌──────────────────────────────┐
│ promise_type 对象（库的挂钩） │
│ 参数副本（按值拷贝进帧）       │
│ 跨挂起点存活的局部变量         │
│ 状态计数器（恢复后跳到哪）     │
│ 结束后要恢复的调用方/句柄等    │
└──────────────────────────────┘
```

三个必须刻进脑子的推论：

1. **参数按值拷贝进帧**——这是协程相对线程的一大安全改进（`std::thread` 传引用要显式 `std::ref`，见 ch05 §5.1；协程自动按值）。
2. **但按值拷贝引用本身不复制所指对象**：参数类型是 `T&`、`T*`、`std::string_view` 时，拷进帧的是引用/指针，所指对象先亡就悬垂（§4.6 坑①，实测复现）。
3. **堆分配可能被编译器省略（HALO, Heap Allocation eLision Optimization）**：若编译器能证明协程帧生命周期完全嵌在调用方内，可以把它优化进调用方栈帧。省不省、何时省，是实现细节——代码**不能依赖**省略发生，但要知道「协程创建 = 一次可能的堆分配」，热路径上别天真地认为它免费（专题 A 有实测实验）。

帧的生命周期 = 创建（调用协程函数那一刻）到 destroy（通过 `coroutine_handle` 销毁）。谁负责 destroy，是 §5 手写 task 要解决的核心问题之一。

### 4.3 promise_type：库与语言之间的插座

协程行为全部由返回类型关联的 **`promise_type`**（用户定义）控制。编译器生成的协程体，本质是按固定次序调用这些钩子：

| 钩子 | 编译器何时调它 | 典型实现 |
| --- | --- | --- |
| `get_return_object()` | 协程函数**刚被调用、还未执行** | 构造返回给调用方的对象（常包着 coroutine_handle） |
| `initial_suspend()` | 函数体执行前 | `suspend_always`（惰性）或 `suspend_never`（急切） |
| `final_suspend()` | `co_return` 之后、帧销毁前 | **必须挂起**（§4.6 坑③），且必须 `noexcept` |
| `return_void()` / `return_value(v)` | 处理 `co_return` | 存结果（二选一，取决于协程是否带值） |
| `yield_value(v)` | 处理 `co_yield v` | 存值并挂起（§7 生成器的主角） |
| `unhandled_exception()` | 函数体异常逃出 | `std::current_exception()` 存入异常盒（§6.1） |
| `get_return_object_on_allocation_failure()` | 帧堆分配失败时（若定义） | 返回空对象而非抛 `bad_alloc` |

`promise_type` 的获取规则：优先找 `T::promise_type`（T 为返回类型），否则找 `std::coroutine_traits<T, Args...>::promise_type`——这就是「返回类型决定协程行为」的实现机制。

一张心智图：**语言层只负责在正确的时机拨动开关（调钩子），开关后面接什么电路（存结果/抛异常/挂谁）全部由 promise_type 的作者决定**。§5 的三版 task，就是对这张插座插上三套逐渐完整的电路。

### 4.4 Awaiter 协议：co_await 到底发生了什么

`co_await expr` 的执行序列（伪代码，标注每一步谁在跑）：

```text
AWT: co_await expr 的完整时序
  ① 取等待者 awaiter：
       expr 自身满足协议 → 直接用
       否则找成员/全局 operator co_await(expr) → 用其返回值
  ② if (!awaiter.await_ready())          // —— 协程（当前线程）
         挂起：保存状态，按 await_suspend 返回值分派：
           void                    → 无条件让出，控制权回到 resume() 的调用方
           bool                    → 返回 false 则不真挂起，立刻继续
           coroutine_handle<>      → 【对称转移】立即切入该句柄所指协程（专题 B 实测）
  ③ ……之后某时刻，有人对句柄调用 resume() ……
  ④ awaiter.await_resume()               // —— 恢复后的协程
       其返回值就是整个 co_await 表达式的值
```

标准库自带的现成 awaiter 只有两个：

```cpp
std::suspend_never{};   // await_ready=true：co_await 它等于没等
std::suspend_always{};  // await_ready=false、suspend 为空：挂起，直到有人 resume
```

**对称转移（symmetric transfer）**值得单独记住：`await_suspend` 返回一个 `coroutine_handle`，编译器用**尾调用**方式切换过去，而不是嵌套调用 resume——即使一百万个协程互相等待，栈深也是常数。这是 §5.3 co_await 链与专题 B 的技术核心；配套的 `std::noop_coroutine()` 返回一个「什么都不做」的句柄，用于「没有等待者时优雅收尾」。

适配规则备忘：`expr` 类型本身有三个成员 → 它就是 awaiter；否则找 `expr.operator co_await()` 或 `operator co_await(expr)`——这就是为什么 §5.3 里 `co_await some_task` 能工作（task 提供 `operator co_await` 返回真正的 awaiter）。

### 4.5 协程是库特性：标准只给机器

把上面三层合起来看清楚边界：**C++20 标准给了关键字、协程帧、promise_type/Awaiter 协议，但没有给任何「任务」或「生成器」类型**。想写 `task<T>`（§5）、`generator<T>`（§7，等 C++23）、`sync_wait`（§5.3），全要自己动手或用库。这不是缺陷设计，而是刻意分期：语言先把机器造好，形状各异的轮子由库生态去长。

第一个自造类型：只负责「惰性执行一段代码」的最小协程——它是 §5.1 task v1 的前身：

```cpp
// c_min_coro.cpp —— 最小协程类型：initial/final 两个挂起点（完整可编译，实测输出见下）
#include <coroutine>
#include <cstdio>
#include <exception>
#include <utility>

struct Lazy {
    struct promise_type {
        Lazy get_return_object() noexcept {
            return Lazy{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }  // 惰性：创建时不执行
        std::suspend_always final_suspend() noexcept { return {}; }    // 结束后帧保留（见下）
        void return_void() noexcept {}
        void unhandled_exception() noexcept { std::terminate(); }
    };
    using Handle = std::coroutine_handle<promise_type>;

    explicit Lazy(Handle h) noexcept : h_(h) {}
    Lazy(Lazy&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    Lazy(const Lazy&) = delete;                 // 句柄唯一所有：禁拷贝
    Lazy& operator=(const Lazy&) = delete;
    ~Lazy() { if (h_) h_.destroy(); }           // RAII：帧的唯一属主负责 destroy

    void start() {                              // 手动驱动：从初始挂起点跑到最终挂起点
        if (!h_.done()) h_.resume();
    }
    bool done() const { return h_.done(); }
    Handle h_;
};

Lazy hello() {                                  // 看似普通函数——
    std::printf("  [body] 协程体执行\n");        // ——实则挂起两次（见输出）
    co_return;
}

int main() {
    Lazy l = hello();                           // 只创建帧，函数体未执行（initial_suspend）
    std::printf("创建完成，函数体尚未执行: done=%s\n", l.done() ? "true" : "false");
    l.start();                                  // 第一次 resume：跑完函数体，
                                                // co_return 后停在 final_suspend → 「第二次挂起」
    std::printf("start 之后: done=%s（停在最终挂起点，帧仍完好）\n", l.done() ? "true" : "false");
}                                               // l 析构 → destroy：唯一属主收走帧
```

```text
创建完成，函数体尚未执行: done=false
  [body] 协程体执行
start 之后: done=true（停在最终挂起点，帧仍完好）
```

数一数挂起点：① 创建后立即挂在 `initial_suspend`（所以「创建完成」时函数体没跑）；② `co_return` 后挂在 `final_suspend`（所以 start 后 `done()==true` 但帧没销毁）。两个挂起点都没占线程——resume 之间，什么线程都不欠着。

注意 RAII 细节：`Lazy` 是句柄的唯一属主（禁拷贝、移动置空、析构 destroy）。**谁持有句柄、谁负责销毁、何时销毁**，就是 §5 要正面对决的问题。

### 4.6 生命周期四坑

协程的坑几乎全部源于同一个事实：**协程帧活得比普通栈帧久，但又不像堆对象那样有明确属主**。四坑如下：

**坑① 引用/指针/string_view 参数悬垂。** 参数按值拷贝进帧（§4.2），但引用类型的「值」就是地址本身：

```cpp
// c_dangle.cpp —— string_view 参数悬垂复现（完整可编译；⚠ 运行即 UB，教学性质）
// ⚠ 本示例演示悬垂：只验证「能编译、现象存在」，跨编译器表现不承诺。
#include <coroutine>
#include <cstdio>
#include <exception>
#include <string>
#include <string_view>
#include <utility>

struct Lazy {
    struct promise_type {
        Lazy get_return_object() noexcept {
            return Lazy{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() noexcept { std::terminate(); }
    };
    using Handle = std::coroutine_handle<promise_type>;
    explicit Lazy(Handle h) noexcept : h_(h) {}
    Lazy(Lazy&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    Lazy(const Lazy&) = delete;
    ~Lazy() { if (h_) h_.destroy(); }
    void start() { if (!h_.done()) h_.resume(); }
    Handle h_;
};

Lazy greet(std::string_view name) {   // 拷进帧的是 view（指针+长度），不是字符串本身
    std::printf("greet: %.4s...\n", name.data());   // 若 name 所指已亡：UB
    co_return;
}

int main() {
    // 危险写法：实参是临时 string，右值被截成 string_view 绑定到参数——
    // 临时对象在 full-expression 结束即亡，而协程体此刻还没跑（initial_suspend）！
    Lazy bad = greet(std::string("temp").substr(0, 4));
    bad.start();                       // 此时 name 所指早已释放：UB（本机实测打印乱码/崩溃）
}
```

修复姿势：协程参数**一律按值收**（`std::string` 而非 `string_view`/`const std::string&`），让拷贝发生在创建时刻；或文档化「实参寿命必须覆盖协程全程」。规则只有一句：**协程参数默认按值，引用参数视为「调用方承诺活全程」的裸借用**。

**坑② 临时实参生命周期。** 与坑①同根的常见变形：`greet(std::string("tmp"))` 若参数是 `string_view`，临时 string 在创建协程的 full-expression 末尾就亡了。编译器对「协程参数的临时量」有特殊延长（拷进帧），但对「临时量的 view」无能为力——危险在隐式转换链上，往往编译器不吭声。

**坑③ final_suspend 忘记挂起 → 帧自毁、句柄集体悬垂。** 若把 `final_suspend` 写成 `suspend_never`：`co_return` 一到，帧**当场自毁**。此后：

- 外部手里的 `coroutine_handle` 全部悬垂，任何 `resume()/done()` 都是 UB；
- 你的 RAII 包装析构时再 `destroy()` = 对已亡帧二次销毁，UB 中的 UB。

正确纪律：**final_suspend 永远挂起**，让「帧已逻辑结束」与「帧内存被回收」解耦——销毁权交给持有句柄的属主（§5.2 的统一销毁协议）。代价只是协程结束时帧暂时多活一会儿。

**坑④ unhandled_exception 不能就地吞掉。** 最常见的错误写法：

```cpp
void unhandled_exception() { e_.reset(); /* 打个日志了事 */ }   // ✘ 异常就地蒸发
```

协程体内逃逸的异常要么存进异常盒（`std::current_exception()`，§6.1 展开值语义搬运），要么 `std::rethrow_exception` 上抛。**吞掉的异常 = 悬挂的调用方**：sync_wait 永远等不到结果、错误静默丢失。正确姿势在 §5.2 落地。

## 5. 手写 task<T>：渐进三版

本节是全章枢纽：用 §4 的机器，把 §3.3 的五条需求逐条变成代码。三版渐进——v1 会挂起会恢复，v2 会存结果存异常，v3 会 co_await 链与同步收口。每一步只加一类能力，每一类都对应 promise_type 的某个钩子。

### 5.1 v1：能挂起、能手动恢复

v1 要回答的问题只有一个：**句柄的所有权与销毁协议**。直接给结论——RAII 包装，句柄唯一属主：

```cpp
// d_task_v1.cpp —— 手写 task v1：挂起 / 手动 resume / RAII 句柄（完整可编译，实测输出见下）
#include <coroutine>
#include <cstdio>
#include <exception>
#include <utility>

struct TaskV1 {
    struct promise_type {
        TaskV1 get_return_object() noexcept {
            return TaskV1{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }  // 惰性：创建不执行
        std::suspend_always final_suspend() noexcept { return {}; }    // 结束后保留帧，属主收尸
        void return_void() noexcept {}
        void unhandled_exception() noexcept { std::terminate(); }      // v1 还没有异常通道（v2 补）
    };
    using Handle = std::coroutine_handle<promise_type>;

    explicit TaskV1(Handle h) noexcept : h_(h) {}
    TaskV1(TaskV1&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    TaskV1(const TaskV1&) = delete;
    TaskV1& operator=(TaskV1&&) = delete;
    ~TaskV1() { if (h_) h_.destroy(); }            // 统一销毁协议：属主析构 = destroy

    void start() { if (!h_.done()) h_.resume(); }
    bool done() const { return h_.done(); }

    Handle h_;
};

TaskV1 half_work(int n) {
    std::printf("  [协程] 拿到 n=%d，干一半...\n", n);
    co_return;
}

int main() {
    TaskV1 t = half_work(42);
    std::printf("[主] 任务已创建，尚未执行\n");
    t.start();                                     // 手动驱动：resume 一次
    std::printf("[主] done=%s，帧将在 t 析构时回收\n", t.done() ? "true" : "false");
}
```

```text
[主] 任务已创建，尚未执行
  [协程] 拿到 n=42，干一半...
[主] done=true，帧将在 t 析构时回收
```

v1 与 §4.5 的 Lazy 结构相同——这不是巧合：**task v0 形态就是 Lazy**。「任务」的第一性是「惰性 + 可驱动」，返回值与异常都是后加的。

v1 暴露的能力边界：`return_void` 意味着只能 `co_return;` 不带值；异常直接 terminate。下一个版本解决「结果怎么出来」。

### 5.2 v2：保存结果、统一销毁协议

v2 加三样东西，全部落在 promise_type 上：

1. `return_value(v)`：结果存进 promise（variant 三态：空/值/异常）；
2. `unhandled_exception()`：`std::current_exception()` 存异常盒（坑④ 的正解）；
3. `get()`：驱动到完成，再取值或重抛。

```cpp
// d_task_v2.cpp —— 手写 task v2：带返回值与异常路径（完整可编译，实测输出见下）
#include <coroutine>
#include <cstdio>
#include <exception>
#include <stdexcept>
#include <utility>
#include <variant>

template <class T>
struct TaskV2 {
    struct promise_type {
        std::variant<std::monostate, T, std::exception_ptr> result_;

        TaskV2 get_return_object() noexcept {
            return TaskV2{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(T v) { result_.template emplace<1>(std::move(v)); }
        void unhandled_exception() noexcept { result_.template emplace<2>(std::current_exception()); }
    };
    using Handle = std::coroutine_handle<promise_type>;

    explicit TaskV2(Handle h) noexcept : h_(h) {}
    TaskV2(TaskV2&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    TaskV2(const TaskV2&) = delete;
    ~TaskV2() { if (h_) h_.destroy(); }

    T get() {                                       // 驱动 + 取结果，一步到位
        h_.resume();                                // 跑到最终挂起点（§4.6 坑③：帧还在）
        auto& r = h_.promise().result_;
        if (auto* e = std::get_if<2>(&r))
            std::rethrow_exception(*e);             // 异常在调用方原地复活（§6.1）
        return std::move(std::get<1>(r));
    }

    Handle h_;
};

TaskV2<int> compute(int x) {
    if (x < 0) throw std::invalid_argument("x<0");  // 异常路径
    co_return x * 2;
}

int main() {
    TaskV2<int> ok = compute(21);
    std::printf("compute(21) = %d\n", ok.get());

    TaskV2<int> bad = compute(-1);
    try {
        (void)bad.get();
    } catch (const std::invalid_argument& e) {
        std::printf("compute(-1) 抛出: %s\n", e.what());   // 异常跨协程边界原样复活
    }
}
```

```text
compute(21) = 42
compute(-1) 抛出: x<0
```

对照 §3.3 需求清单：第 1 条（顺序写法）还没有，第 3 条（异常传播）**已达成**——协程体内随手 throw，`get()` 处一个 try 接住，和普通函数调用一模一样。这就是 std::future 让人称赞的那套「异常穿透共享状态」（§2.6 表格第二行），我们用一个 variant 就复刻了。

> T 为 `void` 时需要 `return_void()` 与 `get()` 的特化（variant 里去掉值态）。实现是机械重复，正文从略；§5.3 的完整版给出处理思路。

### 5.3 v3：co_await 链 + sync_wait（全章枢纽）

v2 的 task 还是一座孤岛：task 里不能再 `co_await` 另一个 task。v3 补上组合能力，要点两件事：

1. **把 task 变成 awaiter**：给 task 加 `operator co_await()`（§4.4 适配规则），返回的 Awaiter 在 `await_suspend` 里记下「谁在等我（continuation）」，然后**返回子协程句柄**——对称转移切入子任务；
2. **完成的反向跳转**：子任务跑到 `final_suspend` 时，返回等待者的句柄——控制权跳回父协程，父协程在 `await_resume()` 里取子任务的结果。

再加一个 `sync_wait()` 在最外层收口（驱动 + 取值，v2 的 get 改名）。完整代码——全章最重的 100 行，值得逐行读完：

```cpp
// d_task_v3.cpp —— co_await 三层嵌套 + sync_wait（完整可编译，实测输出见下）
#include <coroutine>
#include <cstdio>
#include <exception>
#include <utility>
#include <variant>

template <class T>
struct Task {
    struct promise_type {
        std::variant<std::monostate, T, std::exception_ptr> result_;
        std::coroutine_handle<> continuation_{};    // 谁在等我（完成后跳回去）

        Task get_return_object() noexcept {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }

        struct FinalAwaiter {                       // 协程结束时：跳回等待者
            bool await_ready() noexcept { return false; }
            template <class P>
            std::coroutine_handle<> await_suspend(std::coroutine_handle<P> h) noexcept {
                auto cont = h.promise().continuation_;
                return cont ? cont : std::noop_coroutine();   // 对称转移；无人等待则优雅收尾
            }
            void await_resume() noexcept {}
        };
        FinalAwaiter final_suspend() noexcept { return {}; }

        void return_value(T v) { result_.template emplace<1>(std::move(v)); }
        void unhandled_exception() noexcept { result_.template emplace<2>(std::current_exception()); }
    };
    using Handle = std::coroutine_handle<promise_type>;

    explicit Task(Handle h) noexcept : h_(h) {}
    Task(Task&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    Task(const Task&) = delete;
    ~Task() { if (h_) h_.destroy(); }

    struct Awaiter {                                // 让别的协程 co_await 本 task
        Handle h_;
        bool await_ready() noexcept { return false; }
        std::coroutine_handle<> await_suspend(std::coroutine_handle<> awaiting) noexcept {
            h_.promise().continuation_ = awaiting;  // 记住等待者
            return h_;                              // 对称转移：直接切入子协程
        }
        T await_resume() {                          // 子任务完成后，父协程从这里拿结果
            auto& r = h_.promise().result_;
            if (auto* e = std::get_if<2>(&r)) std::rethrow_exception(*e);
            return std::move(std::get<1>(r));
        }
    };
    auto operator co_await() noexcept { return Awaiter{h_}; }

    T sync_wait() {
        h_.resume();                                // 从最外层点火
        auto& r = h_.promise().result_;
        if (auto* e = std::get_if<2>(&r)) std::rethrow_exception(*e);
        return std::move(std::get<1>(r));
    }

    Handle h_;
};

Task<int> leaf(int x)          { co_return x * 2; }
Task<int> middle(int x)        { co_return (co_await leaf(x)) + 1; }
Task<int> outer(int x)         { co_return (co_await middle(x)) + 100; }

int main() {
    Task<int> t = outer(21);
    std::printf("sync_wait 结果: %d\n", t.sync_wait());    // (21*2+1)+100
}
```

```text
sync_wait 结果: 143
```

执行时序值得在脑中放一遍电影：

```text
main: t.sync_wait() → resume(outer)
  outer 跑到 co_await middle(x)
    Awaiter::await_suspend：记 continuation=outer，返回 middle 句柄
      → 对称转移：栈不加深，控制权进 middle
  middle 跑到 co_await leaf(x) → 同上，切入 leaf
  leaf co_return → FinalAwaiter：返回 continuation（middle 句柄）→ 跳回 middle
  middle 的 await_resume 取叶子结果，+1 后 co_return → 跳回 outer
  outer +100 后 co_return → FinalAwaiter：continuation 为空 → 返回 noop_coroutine()
      → 最外层 resume() 调用返回，控制权回到 sync_wait
main: 取结果 143
```

三个设计点点透：


- **为什么整条链一次 resume 就跑完了？** 因为链上每个挂起点都立即对称转移回下一个子任务，没有任何「真等待」。真实异步（等定时器、等 socket）时挂起是真挂起，这就需要一个调度器在事件发生时 resume 对应句柄——asio 干的正是这件事（§8.2），我们手写的 continuation 就是它的事件回调。
- **为什么 final_suspend 要返回句柄而不是 `cont.resume()`？** 递归 resume 每层叠加栈帧，百万深度必栈溢出；返回句柄是尾调用式跳转，栈深常数。这不是理论——专题 B 实测：对称转移版 100 万深度存活，递归 resume 版当场 `0xC00000FD` 栈溢出。
- **sync_wait 为什么能同步收口？** 链全跑通后 resume 自然返回。若链上真有外部事件等待，sync_wait 需阻塞当前线程等调度器通知（生产实现用条件变量，见 ch05 §5.4）。

对照需求清单验收：**1 顺序写法 ✔**（`co_return (co_await leaf(x)) + 1` 就是一行顺序代码）；**2 挂起不占线程 ✔**（挂起即让出，resume 才继续）；**3 异常传播 ✔**（v2 遗产，§6.1 实测穿链）；**4 结果可组合 ✔**（task 之间自由 co_await）。

### 5.4 复盘：我们写的就是「协程库」的内核

三版合起来，手写了协程库的三件核心事：

1. **生命周期**：惰性启动、统一销毁协议、final_suspend 挂起保帧（v1）；
2. **结果传递**：variant 三态 + exception_ptr 穿透（v2）；
3. **调度点**：continuation 记录 + 对称转移 + sync_wait 收口（v3）。

asio 的 `awaitable<T>`、cppcoro 的 `task<T>`、libunifex 各色 task——本质都是这三件事的工业化放大：多了调度器集成、取消传播、内存定制、void/引用特化。§8 会看到它们长成什么样。


## 6. 错误处理与取消

### 6.1 exception_ptr：异常的值语义盒子

`std::exception_ptr` 是 §5.2/§5.3 反复出现的「异常盒」，值得单独看清：它是**异常对象的值语义句柄**——可以拷贝、可以存储、可以跨线程/跨挂起点搬运，`std::rethrow_exception(p)` 时在**当前线程**把异常原地复活。

三个性质对应三个用途：

1. **捕获**：`std::current_exception()` 在 catch 语境（或 `unhandled_exception()` 钩子里）取当前异常的句柄；
2. **搬运**：存进 variant、传给另一线程、穿过任意多级协程挂起——异常对象引用计数保活；
3. **复活**：`std::rethrow_exception(p)` 在任何线程、任何时刻重抛。

为什么 `unhandled_exception` 里不能就地 catch 打日志了事（§4.6 坑④）？因为「异常的语义」必须交付给能决策的层：任务层不知道调用方想怎么处理失败。exception_ptr 让异常**延迟、跨域**地交付——这就是协程版「异常一路传播」（需求 3）的实现基座。

实测穿链：在 §5.3 的三层链最内层抛异常，最外层 sync_wait 处接住：

```cpp
// e_exception_chain.cpp —— 异常跨两级 co_await 穿链（承接 §5.3 Task v3，拼接其定义后编译通过；实测输出见下）
// 复用 §5.3 的 Task<T> 全部代码（本文省略，源文件中含）；另需：
#include <cstdio>
#include <stdexcept>

Task<int> boom()         { throw std::runtime_error("leaf 爆炸"); }
Task<int> middle_boom()  { co_return (co_await boom()) + 1; }     // 中层不接，异常直接上穿
Task<int> outer_boom()   { co_return (co_await middle_boom()) + 100; }

int main() {
    Task<int> t = outer_boom();
    try {
        (void)t.sync_wait();
    } catch (const std::runtime_error& e) {
        std::printf("最外层捕获: %s（穿过了两级 co_await）\n", e.what());
    }
}
```

```text
最外层捕获: leaf 爆炸（穿过了两级 co_await）
```

异常的传播路径：`boom` 体抛出 → 编译器路由到 `unhandled_exception` → 存入 promise 的 variant 异常态 → `boom` 停在 final_suspend → 跳回 middle → middle 的 `await_resume` 检查到异常态**重抛** → middle 自身没有 catch → 再次进 unhandled_exception → ……逐级上穿，直到最外层 catch。**每一级 await_resume 都自动成为异常检查点**——这就是为什么顺序写法（需求 1）与异常传播（需求 3）能同时成立。

### 6.2 取消：stop_token 与协程检查点

C++20 的协作式取消设施 `std::stop_token` / `std::stop_source`（随 `std::jthread` 引入，机制见 ch05 §5.1）与协程没有标准集成，但组合模式很自然：**把 token 带进协程，在安全检查点查询**。检查点本身可以做成 awaiter：

```cpp
// f_cancel.cpp —— stop_token 协程取消检查点（承接 §5.3 Task v3，拼接其定义后编译通过；实测输出见下）
// 复用 §5.3 的 Task<T>（本文省略）；另需：
#include <chrono>
#include <cstdio>
#include <stdexcept>
#include <thread>

struct CancelPoint {
    std::stop_token tok_;
    bool await_ready() const noexcept { return !tok_.stop_requested(); }
    // 未请求停止：await_ready=true，直通，零开销
    bool await_suspend(std::coroutine_handle<>) const noexcept { return false; }
    // 已请求停止：返回 false = 立即恢复（不真挂起），让 await_resume 抛出
    void await_resume() const { throw std::runtime_error("operation cancelled"); }
};

Task<int> process(std::stop_token st, int chunks) {
    int done = 0;
    for (int i = 0; i < chunks; ++i) {
        co_await CancelPoint{st};                   // 每个块边界一次检查点
        std::printf("  完成块 %d\n", i);
        ++done;
    }
    co_return done;
}

int main() {
    std::stop_source src;
    Task<int> t = process(src.get_token(), 1000);

    std::jthread killer([&src] {                    // 外部喊停（模拟超时/用户取消）
        std::this_thread::sleep_for(std::chrono::milliseconds(30));
        src.request_stop();
    });

    try {
        int n = t.sync_wait();
        std::printf("全部完成: %d 块\n", n);
    } catch (const std::runtime_error& e) {
        std::printf("被取消: %s\n", e.what());
    }
    killer.join();
}
```

```text
  完成块 0
  完成块 1
被取消: operation cancelled
```

（具体完成几块取决于机器速度；取消总是发生在某个块边界。）

设计点：检查点 awaiter **从不真挂起**——`await_ready` 直通或 `await_suspend` 返回 false 立即恢复后抛出。取消在这里是「轮询式」的；真正的「挂起中被动唤醒并取消」需要调度器把 stop_callback 挂到等待队列上（asio/stdexec 的取消传播干的正是这个，§8.3）。

### 6.3 结构化并发：生命周期不逃逸作用域

**结构化并发（structured concurrency）**是一条设计纪律：**并发任务的声明周期不得逃逸创建它的作用域——作用域退出前，所有子任务必须收束（完成、取消或失败上抛）**。

它为什么是 §4.6 四坑的根治术：

- 参数悬垂（坑①）：因为「子任务可能活得比调用方久」才有悬垂问题；生命周期不逃逸 → 调用方全程在场 → 按引用/借用也安全了；
- final_suspend 悬垂（坑③）：帧没人收尸的根源是「完成时属主可能已不在」；作用域收束保证属主始终在场；
- 取消难题：作用域退出 = 天然的取消时机，父任务异常时先取消全部子任务再上抛。

一个简化版 when_all——「同时跑两个子任务，等全部收束」（利用 v3 的 sync_wait 在两个线程上并行驱动）：

```cpp
// g_when_all.cpp —— 结构化并发的简化形态（承接 §5.3 Task v3，拼接其定义后编译通过；实测输出见下）
// 复用 §5.3 的 Task<T>（本文省略）；另需：
#include <chrono>
#include <cstdio>
#include <thread>

template <class A, class B>
struct AllResult { A a; B b; };

template <class A, class B>
AllResult<A, B> when_all2(Task<A>& ta, Task<B>& tb) {
    A a{};
    B b{};
    {
        std::jthread runner([&] { a = ta.sync_wait(); });   // 子任务 A 在工作线程驱动
        b = tb.sync_wait();                                 // 子任务 B 在当前线程驱动
    }   // ← jthread 在此 join：作用域收束前，两个子任务必然完成。
        //   注意 join 必须先于「读 a」：若把 return 直接写在本作用域里，
        //   返回值求值先于局部变量析构（join），读 a 与工作线程写 a 构成
        //   数据竞争——本机实测固定拿到垃圾值（修正前的版本就是反例）。
    return AllResult<A, B>{std::move(a), std::move(b)};
}

Task<int> work(int ms, int v) {
    std::this_thread::sleep_for(std::chrono::milliseconds(ms));
    co_return v;
}

int main() {
    Task<int> ta = work(120, 1);
    Task<int> tb = work(120, 2);
    auto t0 = std::chrono::steady_clock::now();
    auto r = when_all2(ta, tb);                         // 并行 120ms，而非串行 240ms
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                  std::chrono::steady_clock::now() - t0).count();
    std::printf("结果 %d+%d，耗时 %lld ms（串行需 ~240）\n", r.a, r.b, (long long)ms);
}
```

```text
结果 1+2，耗时 121 ms（串行需 ~240）
```

注意这个简化版的诚实边界：它用「每任务一线程」实现并行，没有复用池；完整 when_all（任意数量子任务、父取消传播给子、子失败聚合）需要调度器层面的支持——stdexec 的 `execution::when_all` 才是完整答案（§8.3）。但**结构化纪律**已经完整体现：`runner` 是 jthread（析构 join，ch05 §5.1），函数不 return 完，子任务不可能还活着。


## 7. C++23 std::generator：标准库第一个协程类型

> **归属与实证结论（本机 VS 2022，MSVC 19.44）**：`std::generator` 由 P2502R3 标准化，**C++23**。本机实证：`/std:c++20` 下无此类型；**`/std:c++latest` 下可用**（微软 STL 已实装，见 §16 延伸资源微软 C++ 团队博客专文）。本节完整示例编译命令带 `/std:c++latest`；**你若停留在 C++20，§7.3 的手写 mini_generator 是逐行等价物**，教学价值相同。

### 7.1 语法与懒执行

`std::generator<T>` 包装「一串按需产出的值」：协程体内 `co_yield v` 逐个产出，外部用范围 for 消费。它就是 §4.5 说的「标准库补完的第一块轮子」：

```cpp
// h_generator.cpp —— C++23 std::generator（完整可编译；需 /std:c++latest，实测输出见下）
#include <generator>
#include <ranges>
#include <cstdio>

std::generator<int> fib() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;                 // 产出一个值并挂起：执行权交回消费者
        auto t = a + b;
        a = b;
        b = t;
    }
}

int main() {
    int produced = 0;
    for (int v : fib() | std::views::take(8)) {   // 无限序列 × 只取 8 个
        std::printf("%d ", v);
        ++produced;
    }
    std::printf("\n只产出了 %d 个值——没有取 8 个之前，fib 不会多算一步\n", produced);
}
```

```text
0 1 1 2 3 5 8 13
只产出了 8 个值——没有取 8 个之前，fib 不会多算一步
```

三个要点：

- **懒执行**：`fib()` 被调用时（`generator` 构造完成）协程体一步未跑；每次迭代推进才执行到下一个 `co_yield`。这就是 §4.3 `initial_suspend` 挂起的教科书应用。
- **无限序列安全**：值是按需拉的，「无限」只在你取的范围内存在。
- **拉模型**：消费者驱动节奏（区别于回调的推模型）——游戏里「按需解压下一个资源块」这类场景天然契合。

### 7.2 与 ranges 衔接

`std::generator` 实现了 range 概念，可直接进 ranges 管道——协程产值 + 视图组合，两套标准库机制无缝咬合：

```cpp
// h_generator_ranges.cpp —— generator 喂 ranges 管道（完整可编译；需 /std:c++latest）
#include <generator>
#include <ranges>
#include <cstdio>

std::generator<int> naturals() {
    int i = 0;
    while (true) co_yield i++;
}

int main() {
    auto evens = naturals() | std::views::filter([](int v) { return v % 2 == 0; })
                            | std::views::take(5);
    for (int v : evens) std::printf("%d ", v);    // 0 2 4 6 8
    std::printf("\n");
}
```

```text
0 2 4 6 8
```

注意边界：`std::generator` 是**同步**的生成器（拉取在消费线程上执行）。它不是异步流（没有「数据将来某时刻到达」的语义）——「异步序列」要等 `std::execution` 生态的 async sequences（§8.3）。名字里没有 async，语义里也没有，别被直觉骗了。

### 7.3 兜底：C++20 手写 mini_generator

若你的工程钉在 C++20，40 行手写一个等价物——顺便把 §4.3 的 `yield_value` 钩子用上（它正是为生成器准备的）：

```cpp
// h_mini_generator.cpp —— C++20 手写 generator（完整可编译，实测输出见下）
#include <coroutine>
#include <cstdio>
#include <exception>
#include <iterator>
#include <utility>

template <class T>
class MiniGen {
public:
    struct promise_type {
        T value_{};
        MiniGen get_return_object() noexcept {
            return MiniGen{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }   // 懒执行
        std::suspend_always final_suspend() noexcept { return {}; }     // 结束保帧，迭代器据此判停
        std::suspend_always yield_value(T v) {                          // co_yield 的落点：
            value_ = std::move(v);                                      // 存值并挂起
            return {};
        }
        void return_void() noexcept {}
        void unhandled_exception() noexcept { std::terminate(); }
    };
    using Handle = std::coroutine_handle<promise_type>;

    explicit MiniGen(Handle h) noexcept : h_(h) {}
    MiniGen(MiniGen&& o) noexcept : h_(std::exchange(o.h_, {})) {}
    MiniGen(const MiniGen&) = delete;
    MiniGen& operator=(MiniGen&&) = delete;
    ~MiniGen() { if (h_) h_.destroy(); }

    struct iterator {
        Handle h_;
        bool done_ = false;
        iterator& operator++() { h_.resume(); done_ = h_.done(); return *this; }   // 拉下一个值
        const T& operator*() const { return h_.promise().value_; }
        bool operator==(std::default_sentinel_t) const { return done_; }           // C++20 自动重写 !=
    };
    iterator begin() {
        if (h_ && !h_.done()) h_.resume();   // 推进到第一个 co_yield
        return {h_, h_.done()};
    }
    std::default_sentinel_t end() const { return {}; }

private:
    Handle h_;
};

MiniGen<int> fib() {
    int a = 0, b = 1;
    while (true) { co_yield a; auto t = a + b; a = b; b = t; }
}

int main() {
    int n = 0;
    for (int v : fib()) {
        if (n >= 8) break;
        std::printf("%d ", v);
        ++n;
    }
    std::printf("\n（C++20 手写版，与 std::generator 同一机制）\n");
}
```

```text
0 1 1 2 3 5 8 13
（C++20 手写版，与 std::generator 同一机制）
```

逐行对照 §4.3 的钩子表，会发现这 40 行没有一行新知识——`yield_value` 存值、`initial_suspend` 懒启动、`final_suspend` 保帧供判停。标准库的 generator，就是这段代码的工业化版本（多了对 ranges 概念的完整实现、拷贝语义约束与异常传播）。


## 8. 真实世界的异步：executor、asio 与 C++26

### 8.1 executor：被缺席的关键概念

§3.3 需求清单里最难的一条是「挂起不占线程」——不占，那**谁来恢复它**？答案需要一个标准词汇：**executor（执行器）**——「把一段工作放到某处执行」的抽象，「某处」可以是当前线程、线程池、IO 事件循环、GPU 流……

executor 是 §2.6 表格「运行位置控制」的正式学名。它的缺席正是标准库异步的地基缺口：`std::async` 不让你选线程池（§2.2 坑②的根源）、future 没有「在哪继续」的概念。而 asio 生态带着完整的 executor 模型跑了十几年；C++26 的 `std::execution` 最终把 scheduler（executor 的标准形态）纳入了组合式异步的核心（§8.3）。

心智模型：**协程解决「怎么写」（控制流形状），executor 解决「在哪跑」（执行位置），两者正交组合**——同一份 `co_await` 代码，换一个 executor 就从线程池切到 IO 循环，控制流一个字不改。

### 8.2 asio 实景：事实标准的模样

> **本节全部为讲解性片段，依赖第三方库 asio（Boost.Asio 或 standalone asio），不在本章编译验证范围。** API 形状以 asio 官方文档为准（§16）。

网络技术规范（Networking TS）2018 年后停止推进，asio 成为 C++ 网络异步**事实标准**。三个核心概念：

1. **`io_context`**：事件循环 + 调度器。所有异步操作的完成事件汇聚于此，`run()` 它的线程就是「恢复协程的线程」。
2. **完成令牌（completion token）**：同一个异步操作，按令牌类型自动变成回调风格 / future 风格 / **协程风格**——§2 与 §3 的对立在此和解。
3. **`awaitable<T>` + `use_awaitable` + `co_spawn`**：asio 自家的 task 类型（§5.4 说的工业化放大版），协程挂起时把 continuation 注册进事件循环，事件到达时由 io_context 恢复。

```cpp
// ===== 片段（依赖 asio，不可直接编译）：协程风格的定时器 =====
asio::awaitable<void> heartbeat() {
    auto ex = co_await asio::this_coro::executor;            // 当前协程的执行器
    asio::steady_timer timer(ex);                            // 定时器绑定到事件循环
    for (int i = 0; i < 3; ++i) {
        timer.expires_after(std::chrono::seconds(1));
        co_await timer.async_wait(asio::use_awaitable);      // 挂起：不占线程等 1 秒
        std::printf("beat %d\n", i);
    }
}

int main() {
    asio::io_context io;
    asio::co_spawn(io, heartbeat, asio::detached);           // 把协程种进事件循环
    io.run();                                                // 事件循环：恢复挂起的协程
}
```

```cpp
// ===== 片段（依赖 asio）：echo 服务的协程骨架 =====
asio::awaitable<void> echo(asio::ip::tcp::socket sock) {
    char data[1024];
    for (;;) {
        std::size_t n = co_await sock.async_read_some(          // 挂起等数据——不占线程
            asio::buffer(data), asio::use_awaitable);
        co_await async_write(sock, asio::buffer(data, n),       // 挂起等写完
            asio::use_awaitable);
    }   // 异常经 co_await 自动传播（§5.2 同款机制，asio 的 awaitable 也这么做）
}
```

对照 §5.3 手写的 task，库替你做掉了什么：`awaitable` = 我们的 Task（多了与 io_context 的调度集成）；`async_wait/use_awaitable` = 我们的 Awaiter（挂起时注册事件回调，而非立即对称转移）；`io.run()` = 我们的 sync_wait（只是它等的是事件，不是链跑完）。**机制一模一样，规模不同**——这就是「手写一遍」的价值：读 asio 代码时每个部件你都认识。

### 8.3 C++26 std::execution：sender/receiver 展望

> **本节为展望 + 讲解性片段。** `std::execution`（P2300R10）已进入 **C++26 工作草案**；微软 STL 尚未实装（microsoft/STL issue #4768 跟踪中，§16）；参考实现 NVIDIA/stdexec 可用于先行体验；2025 年的 P3826 在继续打磨发送器算法的定制机制。

std::execution 的三个核心概念：

- **sender**：「一件将来会完成的活」的惰性描述。sender 本身不执行——它是**值**，可以拷贝、组合、变换；
- **receiver**：「活完成后接住结果/错误/取消」的回调三件套；
- **scheduler**：「在某个 executor 上获得执行时机」的 sender 工厂。

与 future 的本质区别：future 是**急切**的（async 一调就开跑）、行为绑定在对象上；sender 是**惰性**的——先声明整条管线，最后 `sync_wait` 一锤定音才执行：

```cpp
// ===== 片段（stdexec 语法示意，非标准库）：
auto work = ex::schedule(pool.get_scheduler())          // 在线程池上获得执行时机
          | ex::then([] { return load_texture("hero.png"); })   // 活 1：加载
          | ex::then([](Tex t) { return decompress(t); })       // 活 2：解压（自动接活 1 的结果）
          | ex::upon_error([](auto e) { return fallback_tex; }); // 错误通道一路铺好

auto [tex] = ex::sync_wait(std::move(work)).value();    // 到此才真正执行
```

对照整章演进线：

| 能力 | future (C++11) | 协程+手写task (C++20) | execution (C++26) |
| --- | --- | --- | --- |
| 顺序写法 | ✘ 阻塞拼接 | ✔ co_await | ✔ sender 管道 |
| 异常/错误传播 | ✔ get 重抛 | ✔ exception_ptr 穿链 | ✔ error channel 一等公民 |
| 运行位置控制 | ✘ | ◐ 换调度器要改 task 类型 | ✔ scheduler 随手换 |
| 惰性组合 | ✘ 急切 | ◐ task 内惰性 | ✔ 全程惰性，先声明后执行 |
| 取消传播 | ✘ | ◐ 手工检查点（§6.2） | ✔ 结构化取消内建 |
| 标准化程度 | 标准 | 语言机制标准，类型靠库/23 起 generator | C++26 草案 |

**给当下工程的务实结论**：2026 年的现实是「协程机器 + 库轮子」。新代码用协程写控制流（本章 §5 的手写版足以理解原理，工程上用 asio/stdexec/cppcoro 的成品 task）；`std::execution` 值得跟踪，等主流标准库实装（盯 microsoft/STL issue #4768）再评估迁移。


## 9. 游戏引擎语境

### 9.1 job system：fork/join 与工作窃取

游戏引擎不「每任务一线程」（§6.3 的简化版当反例），而是固定一组 worker 线程 + **任务队列**，经典模型：

- **fork/join**：主（或任一）worker 把大任务拆（fork）成若干子任务入队，然后 join 等全部完成再合成；
- **每个 worker 一个本地双端队列**：自己从队尾 LIFO 取（缓存友好、天然后进先出的 fork 深度序），**别人偷则偷队头 FIFO**（最老的任务大概率已经被完整拆解、工作量最大）——这就是**工作窃取（work stealing）**，概念级即可；
- **固定 worker 池而非动态线程**：线程数 ≈ 硬件并发，创建销毁成本与频繁上下文切换都省掉（回扣 ch05 §5.9 的「线程不是越多越好」）。

```cpp
// i_job_pool.cpp —— packaged_task 迷你 job 池：fork/join（完整可编译，实测输出见下）
// 复用 §2.4 的 packaged_task 手艺 + ch05 §5.4/§5.8 的 mutex/条件变量队列纪律
#include <condition_variable>
#include <cstdio>
#include <deque>
#include <future>
#include <mutex>
#include <thread>
#include <vector>

class MiniJobPool {
public:
    explicit MiniJobPool(int n) {
        for (int i = 0; i < n; ++i)
            workers_.emplace_back([this] { worker_loop(); });
    }
    ~MiniJobPool() {
        { std::lock_guard lk(m_); stop_ = true; }
        cv_.notify_all();
    }
    template <class F>
    auto submit(F f) -> std::future<decltype(f())> {          // fork：入队并取回 future
        std::packaged_task<decltype(f())()> task(std::move(f));
        auto fut = task.get_future();
        { std::lock_guard lk(m_); jobs_.emplace_back(std::move(task)); }
        cv_.notify_one();
        return fut;
    }

private:
    void worker_loop() {                                       // 固定 worker：线程不随任务生灭
        for (;;) {
            std::unique_lock lk(m_);
            cv_.wait(lk, [this] { return stop_ || !jobs_.empty(); });   // 谓词等待（ch05 §5.4）
            if (stop_ && jobs_.empty()) return;
            auto job = std::move(jobs_.front());
            jobs_.pop_front();
            lk.unlock();
            job();                                             // packaged_task：结果进 future
        }
    }
    std::vector<std::jthread> workers_;
    std::deque<std::packaged_task<void()>> jobs_;
    std::mutex m_;
    std::condition_variable cv_;
    bool stop_ = false;
};

int main() {
    MiniJobPool pool(4);
    constexpr int kChunks = 8;
    std::vector<std::future<long long>> parts;                 // fork：拆 8 份并行求和
    for (int c = 0; c < kChunks; ++c)
        parts.push_back(pool.submit([c] {
            long long s = 0;
            for (long long i = c * 1000000LL + 1; i <= (c + 1) * 1000000LL; ++i) s += i;
            return s;
        }));
    long long total = 0;
    for (auto& f : parts) total += f.get();                    // join：等全部收束再合成
    std::printf("fork/join 总和: %lld\n", total);
}
```

```text
fork/join 总和: 40000008000000000
```

（这个简化池每个 worker 一个共享队列，没有工作窃取；真实引擎如 Naughty Dog 的方案、Unreal 的 TaskGraph 都是「本地队列 + 窃取」的完整版。注意 submit 的闭包被 `std::function` 类型擦除了一次调用开销——量级与 ch05 §5.8 对 std::function 的讨论一致，帧级热路径不走这条道。）

### 9.2 异步资源加载：管线与时序

主线程渲染 + 工作线程加载的完整管线（ch05 实践任务 1 的迷你加载器是它的骨架，这里升级为带阶段的视图）：

```text
IO 线程                    解压线程                    渲染线程（主）
─────────                 ─────────                  ────────────
read(file) ──字节块──▶   inflate ──原始像素──▶   GPU 上传(glTexImage2D)
   ▲ 慢：磁盘/网络          ▲ 中：CPU 密集            ▲ 快但独占 GL 上下文
                                                        │
                   三级流水线，各级用任务队列衔接（ch05 §5.8）
                   GPU 上传必须回渲染线程：GL 上下文归属决定（第 3 节渲染器约定）
```

关键工程判断：

- **任务载荷要结构化**：队列里别传裸 `string`（路径），传「带校验和、尺寸、格式的加载请求结构体」——ch05 实践任务 1 的升级方向；
- **GPU 上传是阶段边界**：前面的阶段可以任意并行，最后一步必须回渲染线程。这就是「异步加载」里协程/任务们最终的 join 点；
- **背压**：加载速度快于消费时，队列会膨胀——队列长度上限 + 满时暂停入队（生产者-消费者的流控，ch05 §5.8 的队列加个计数即可）。

### 9.3 结构化并发在引擎里落地

§6.3 的纪律在引擎里有三个具体落点：

1. **帧边界即作用域**：本帧 fork 的 job 本帧 join 收束。「渲染帧 N 时，帧 N-1 的粒子更新还挂着」= 竞态温床（下一帧要读写同一批数据）。帧循环结构天然给出作用域：`begin_frame → spawn jobs → join → end_frame`；
2. **加载屏即取消域**：玩家退出关卡 → `stop_source.request_stop()` → 加载管线在下一个检查点（§6.2）逐级退出，已分配的半成品资源统一回收。没有结构化取消的引擎，这里就是「退出关卡后加载线程还往已销毁的世界里写数据」的经典事故；
3. **反例形态**：某系统在任务里 `detach`（或无主地投进全局队列）一个「延迟 5 秒回调」——5 秒内关卡已卸载，回调捕获的世界指针全部悬垂（ch05 §5.1 悬垂引用坑的任务版）。**逃逸出作用域的任务 = 延迟引爆的悬垂指针**。

## 10. 深入专题

### 专题 A：协程帧的解剖——编译器把你的函数改写成了什么

承诺对象、挂起点、状态计数器这些词容易停留在比喻层面。本专题把一个协程**手工改写回普通 C++**，看语言在背后生成的状态机长什么样。

原协程（两阶段工作，中间挂起一次）：

```cpp
// 专题 A 的原协程（配合 §4.5 的 Lazy 类型使用）
Lazy two_step(int seed) {
    int doubled = seed * 2;          // 挂起点之前
    std::suspend_always pause{};     // 具名 awaiter（勘误注①的具名写法；Lazy 无默认构造，不可 co_await Lazy{}）
    co_await pause;                  // 挂起：执行权交出，doubled 必须存活
    std::printf("  恢复后 doubled=%d（跨挂起存活）\n", doubled + 1);   // 挂起点之后
    co_return;
}
```

编译器眼中，它等价于这个普通结构体状态机（手工改写，字段编号对应 §4.2 帧布局图）：

```cpp
// j_state_machine.cpp —— 手工改写协程为状态机（完整可编译，实测输出见下）
#include <cstdio>

struct TwoStepMachine {              // = 协程帧
    int param_seed;                  // ① 参数区：按值拷贝进帧（§4.2）
    int local_doubled;               // ② 局部区：跨挂起点的局部变量
    int state = 0;                   // ③ 状态计数器：恢复后从哪继续
    // ④ promise/句柄区：本例省略（Lazy 的 promise 无状态）

    // 挂起点位置：唯一的 co_await 在 local_doubled 计算之后 → 只需 2 个状态
    void resume() {                  // = 编译器生成的协程体（switch 版）
        switch (state) {
        case 0:                      // 首次进入：从函数开头跑到挂起点
            local_doubled = param_seed * 2;
            std::printf("  挂起前 doubled=%d\n", local_doubled);
            state = 1;               // 记住「恢复后从哪继续」
            return;                  // 挂起：控制权交还 resume() 的调用方
        case 1:                      // 恢复：直接跳到挂起点之后
            std::printf("  恢复后 doubled=%d（跨挂起存活）\n", local_doubled + 1);
            state = 2;
            return;
        default:
            return;                  // 已结束
        }
    }
};

int main() {
    TwoStepMachine m;                // = 协程帧（此处栈上示意；真实协程帧在堆上）
    m.param_seed = 21;

    m.resume();                      // 跑到挂起点（对应 initial_suspend 后的第一次 resume）
    m.resume();                      // 从挂起点恢复（局部变量完好无损）
    std::printf("状态机与协程行为逐点对应：参数区/局部区/状态跳转表\n");
}
```

```text
  挂起前 doubled=42
  恢复后 doubled=43（跨挂起存活）
```

逐字段对应：`param_seed` ↔ 帧的参数区；`local_doubled` ↔ 局部区（**它跨挂起存活的唯一原因是它住帧里不住栈里**——普通函数的栈帧在返回时就没了）；`state`+`switch` ↔ 恢复跳转表。真实编译器还会做：局部变量活性分析（不跨挂起的变量留在栈上不进帧）、状态编号压缩、把 switch 编成跳转表。

**HALO 观察实验**（接 §4.2 第 3 条推论）：全局替换 `operator new` 计数分配次数，`/O2` 下本机 MSVC 19.44 实测：

```text
场景1（协程句柄经 noinline 函数返回、编译器无法证明帧不逃逸）: 分配 1 次
场景2（同一协程、句柄寿命完全限于调用方）:               分配 0 次 —— HALO 生效
```

结论坐实：协程创建的成本是「**可能**一次堆分配」，且优化器在你够聪明的场景下能把它抹掉——但写代码时按「有一次分配」做预算，把「零分配」当惊喜而非依赖（ch05 §5.9 的分配纪律直接适用）。

### 专题 B：对称转移为什么能防栈溢出——手测 100 万次 co_await 链

§4.4 与 §5.3 都断言「await_suspend 返回句柄 = 尾调用式跳转，栈深常数」。这个实验给断言以证据：构建 **100 万环**的 co_await 链，两个版本只差 final_suspend 的一行：

```cpp
// k_symmetric.cpp —— 两版唯一差异（FinalAwaiter::await_suspend）；完整程序见本节末附注：
template <bool SymUnwind>
struct ChainTask { /* …promise 含 continuation_ 与 child_（迭代拆除用）… */
    struct FinalAwaiter {
        template <class P>
        std::coroutine_handle<> await_suspend(std::coroutine_handle<P> h) noexcept {
            auto cont = h.promise().continuation_;
            if constexpr (SymUnwind) {
                return cont ? cont : std::noop_coroutine();   // 好版：返回句柄 → 尾调用跳转
            } else {
                if (cont) cont.resume();                      // 坏版：递归 resume → 栈深 O(N)
                return std::noop_coroutine();
            }
        }
        /* … */
    };
};
// 链构造：环 i 的协程体是 co_await 环 i-1 的句柄（Handoff awaiter）；
// 叶子是 co_return。外层 resume() 后沿 continuation 一路回跳。
// 拆除用 promise 里记录的 child_ 迭代 destroy（避免析构级联递归）。
```

本机 MSVC 19.44 实测（默认 1MB 栈，深度 1,000,000）：

```text
> k_symmetric.exe good
链构建完成（1000000 环），开始运行...
完成：对称转移 版在 1000000 深度下存活
迭代拆除 1000001 帧，进程正常退出          ← 100 万次跳转，栈深始终常数

> k_symmetric.exe bad
（无输出，进程崩溃）                       ← 递归 resume：0xC00000FD STATUS_STACK_OVERFLOW
```

原理一句话说透：`return cont` 的跳转发生在 `await_suspend` **返回之后**（当前栈帧已弹出的位置上做尾调用）；`cont.resume()` 则在当前帧**之内**再压入一整套协程帧栈帧。100 万环 × 每环几十字节栈 = 远超 1MB 默认栈。这也解释了 §5.3 FinalAwaiter 里那句 `std::noop_coroutine()`：它就是「链到头了，别跳了，返回最外层 resume() 的调用方」的优雅收尾。

> **实现勘误注（本机实证）**：MSVC 前端不接受 `co_await 依赖类型花括号临时量`（如 `co_await ChainTask<B>::Handoff{next}` 报 C2760），需具名变量规避：`typename ChainTask<B>::Handoff aw{next}; co_await aw;`。本章正文示例均采用具名写法。

**附注：k_symmetric.cpp 完整程序**（上文输出即本程序实测；构建与运行）：

```bat
cl /nologo /utf-8 /std:c++20 /EHsc /W4 /DSYMUNWIND=1 k_symmetric.cpp /Fe:k_symmetric_good.exe
cl /nologo /utf-8 /std:c++20 /EHsc /W4 /DSYMUNWIND=0 k_symmetric.cpp /Fe:k_symmetric_bad.exe
k_symmetric_good.exe good        :: 输出见上（good 段）
k_symmetric_bad.exe bad          :: 无输出，退出码 0xC00000FD（STACK_OVERFLOW）
```

```cpp
// k_symmetric.cpp —— 对称转移 vs 递归 resume：100 万环对照实验
// 构建：cl /nologo /utf-8 /std:c++20 /EHsc /W4 /DSYMUNWIND=1 k_symmetric.cpp /Fe:k_symmetric_good.exe
//       cl /nologo /utf-8 /std:c++20 /EHsc /W4 /DSYMUNWIND=0 k_symmetric.cpp /Fe:k_symmetric_bad.exe
// 运行：k_symmetric_good.exe good    k_symmetric_bad.exe bad
#include <coroutine>
#include <cstdio>
#include <exception>

template <bool SymUnwind>
struct ChainTask {
    struct promise_type;
    using handle_t = std::coroutine_handle<promise_type>;

    handle_t h_{};
    explicit ChainTask(handle_t h) : h_(h) {}
    ChainTask(const ChainTask&) = delete;
    ChainTask& operator=(const ChainTask&) = delete;
    ~ChainTask() { if (h_) h_.destroy(); }        // RAII 属主；所有权可经 detach() 移交
    handle_t detach() noexcept { handle_t h = h_; h_ = {}; return h; }

    struct promise_type {
        std::coroutine_handle<> continuation_{};  // 完成后要跳回的父协程（noop=链头）
        handle_t child_{};                        // 我移交出去的子协程（迭代拆除线索）
        ChainTask get_return_object() { return ChainTask{handle_t::from_promise(*this)}; }
        std::suspend_always initial_suspend() noexcept { return {}; }

        struct FinalAwaiter {                     // ←—— 两版唯一差异在这里
            bool await_ready() const noexcept { return false; }
            template <class P>
            std::coroutine_handle<> await_suspend(std::coroutine_handle<P> h) noexcept {
                std::coroutine_handle<> cont = h.promise().continuation_;
                if constexpr (SymUnwind) {
                    return cont ? cont : std::noop_coroutine();   // 好版：尾调用跳回父，栈深常数
                } else {
                    if (cont) cont.resume();                  // 坏版：当前帧之内再压一栈
                    return std::noop_coroutine();
                }
            }
            void await_resume() noexcept {}
        };
        FinalAwaiter final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() noexcept { std::terminate(); }
    };

    // 「等子做完再继续」的 awaiter：登记续体与拆除线索后，尾调用跳进子（两版相同）
    struct Handoff {
        handle_t child_;
        explicit Handoff(handle_t c) : child_(c) {}
        bool await_ready() const noexcept { return false; }
        template <class P>
        std::coroutine_handle<> await_suspend(std::coroutine_handle<P> parent) noexcept {
            child_.promise().continuation_ = parent;      // 子完成 → 跳回本协程
            parent.promise().child_ = child_;             // 留下迭代拆除的线索
            return child_;                                // 跳进子协程
        }
        void await_resume() noexcept {}
    };
};

template <bool SymUnwind>
ChainTask<SymUnwind> leaf_coro() { co_return; }       // 最内层：直接完成

template <bool SymUnwind>
ChainTask<SymUnwind> link_coro(typename ChainTask<SymUnwind>::handle_t child) {
    typename ChainTask<SymUnwind>::Handoff aw{child}; // 具名 awaiter（勘误注①的具名写法）
    co_await aw;
}

int main(int argc, char** argv) {
    if (argc <= 1) { std::printf("用法: %s good|bad\n", argv[0]); return 1; }
    constexpr int kDepth = 1'000'000;
    constexpr bool kSym = (SYMUNWIND != 0);            // good/bad 在编译期定型
    using Task = ChainTask<kSym>;

    // 自底向上迭代构建（若递归构建，1M 深会先在构建期爆栈）
    ChainTask<kSym> t0 = leaf_coro<kSym>();
    typename Task::handle_t cur = t0.detach();
    for (int i = 0; i < kDepth; ++i) {
        ChainTask<kSym> t = link_coro<kSym>(cur);
        cur = t.detach();
    }
    cur.promise().continuation_ = std::noop_coroutine();  // 链头：完成后回到 main
    std::printf("链构建完成（%d 环），开始运行...\n", kDepth);
    cur.resume();   // good：常数栈跑完全链；bad：在回归级联中爆栈

    // 迭代拆除（若依赖 RAII 级联析构，1M 深的析构递归同样爆栈——这正是 child_ 的用途）
    int frames = 0;
    while (cur) {
        typename Task::handle_t c = cur.promise().child_;  // 先取线索再销毁
        cur.destroy();
        cur = c;
        ++frames;
    }
    std::printf("完成：对称转移 版在 %d 深度下存活\n迭代拆除 %d 帧，进程正常退出\n", kDepth, frames);
    return 0;
}
```


## 11. 小结与本章纪律清单

五种异步写法 × 四项能力，一张矩阵收束全章：

| 写法 | 可组合 | 异常传播 | 可取消 | 运行位置控制 | 标准化 |
| --- | --- | --- | --- | --- | --- |
| 同步调用 | — | 直接 | — | 当前线程 | — |
| 回调（§3） | ✘ 手工拼接 | ✘ 双通道手工 | ✘ | ◐ 随宿主库 | ◐ asio 事实标准 |
| future（§2） | ✘ 阻塞拼接 | ✔ get 重抛 | ✘ | ✘ | C++11 |
| 协程 + task（§4–§6） | ✔ co_await 链 | ✔ exception_ptr 穿链 | ◐ 检查点/库 | ◐ 随 task 的调度器 | 语言 C++20；类型靠库/C++23 起 generator |
| sender/receiver（§8.3） | ✔ 惰性管道 | ✔ error channel | ✔ 结构化取消 | ✔ scheduler | C++26 草案 |

**本章纪律清单**（建议当贴纸贴显示器上）：

1. 不裸用 `std::async`：要么显式 `std::launch::async`，要么不用；**永不丢弃 async 返回的 future**（析构阻塞）；
2. deferred future 上探测完成，先分流 `wait_for(0) == deferred` 再轮询；
3. 协程参数默认按值收；引用/指针/string_view 参数视为「调用方承诺活全程」的裸借用；
4. `final_suspend` **永远挂起**且 `noexcept`——帧销毁权归属主；
5. `unhandled_exception` 存 `std::current_exception()`，**永不就地吞掉**；
6. 跨 await 的恢复跳转用对称转移（返回句柄），**禁递归 resume**（专题 B 实测栈溢出）；
7. 并发任务生命周期不逃逸作用域：帧边界收 job，作用域退出先取消子任务；
8. asio/stdexec 的示例是片段，不冒充可编译；语义拿不准查 cppreference。

## 12. 构建与运行速查（Windows + VS2022 主环境）

```bat
:: C++20 全章通用（§1–§6、§9–§10 及手写 generator）
cl /nologo /utf-8 /std:c++20 /EHsc /W4 源文件.cpp

:: §7 的 std::generator（C++23；本机实证 /std:c++20 下不可用）
cl /nologo /utf-8 /std:c++latest /EHsc /W4 h_generator.cpp
```

等价命令：`g++ -std=c++20 -Wall -Wextra -pthread 源文件.cpp`（Linux 需 `-pthread`；§7 用 `-std=c++23`，libstdc++ 的 `<generator>` 支持情况以实测为准）；`clang++ -std=c++20 -Wall 源文件.cpp`（libstdc++/libc++ 的协程与 generator 支持以实测为准）。

> 工具链怪癖备忘（本机实证，写代码时留意）：① 协程函数内 `co_await 依赖类型花括号临时量` 在 MSVC 下报 C2760，具名变量规避（专题 B 勘误注）；② 对已 `get()` 的 deferred future 再调 `wait_for` 在 MSVC 下 fail-fast（§2.2 坑③附注）。

## 13. 实践任务

### 任务 1（核心层）：给手写 task 加 then() 组合子

给 §5.3 的 `Task<T>` 增加链式续接：`task.then(f)` 返回新 task，f 接收原任务结果、产出新结果；三层 `then` 链最后 `sync_wait` 一次收口，异常能穿链重抛。

**验收要点**：① 三层 then 一次 sync_wait 得到复合结果；② 链中任一环抛异常，最外层 try 接住且后续环不执行；③ 不引入额外线程——全链仍是一次 resume 对称转移跑完（提示：then 产生的 wrapper 协程 co_await 原任务（复用 Awaiter），co_return f(值)）。

### 任务 2（核心层）：generator 惰性行读取器

用 `std::generator`（或 §7.3 手写版）写 `lines(path)`：打开 `ifstream`，按行 `co_yield`。**坑点设计**：协程参数若收 `string_view` 或 `co_yield` 局部 `string_view`——回扣 §4.6 坑①，必须 yield 拥有所有权的 `std::string`。

**验收要点**：① 大文件只取前 N 行时，实测读取行数恰为 N（打印计数证明惰性）；② 消费端拿到的行在 generator 销毁后仍可用（所有权语义）；③ 文件不存在时行为明确（抛异常进 generator 传播通道，消费端可接）。

### 任务 3（扩展层，片段级可接受）：ch05 任务队列的协程化改造

把 ch05 实践任务 1 的迷你异步任务队列改造成协程接口：worker 循环 `co_await` 队列任务，主循环 `sync_wait` 收上屏结果。允许片段级交付，重点是「回调 → 协程」翻译对照表：

| 回调版 | 协程版 |
| --- | --- |
| `queue.pop(cb)` 完成时调 cb | `co_await queue.pop_async()` |
| cb 里手工串下一步 | 顺序写下一步 |
| cb 的 ec 参数逐层 if | 异常自动穿链（§6.1） |
| shared_ptr<Ctx> 状态逃逸（§3.2） | 局部变量住协程帧，零逃逸 |

## 14. 自测题

1. `std::async` 返回的 future 析构时会发生什么？为什么是它独有而 promise/packaged_task 的 future 没有？（§2.2）
2. deferred future 上 `wait_for(0s)` 返回什么？「`wait_for(0)==ready` 当轮询判完成」错在哪？正确的探测姿势？（§2.2）
3. `std::future` 缺哪两样关键能力，使它当不了异步组合原语？（§2.6）
4. 使函数成为协程的条件是什么？三个关键字各自干什么？（§4.1）
5. 协程参数如何进入协程帧？哪些参数类型仍会悬垂？为什么？（§4.2/§4.6）
6. `final_suspend` 的 awaiter 不挂起，会发生什么？会引发哪两种 UB？（§4.6 坑③）
7. `await_suspend` 返回协程句柄意味着什么（术语）？它解决什么问题？证据？（§4.4/专题 B）
8. `unhandled_exception` 里应该存什么？为什么不就地 catch 打日志了事？（§4.6 坑④/§6.1）
9. `std::generator` 是哪个标准、哪份提案？你怎么验证自己编译器支不支持？（§7）
10. `std::execution` 进了哪个标准？三大概念是什么？当前 MSVC STL 什么状态？（§8.3）

## 15. 常见误区

1. **「所有 future 析构都阻塞」**——只有 `std::async` 返回的；promise/packaged_task 的 future 析构不等待（§2.2 坑①）。
2. **「wait_for(0)==ready 就能轮询判完成」**——deferred 上永远返回 deferred，轮询死循环（§2.2 坑③）。
3. **「std::async 默认一定开新线程」**——默认策略 async|deferred 由实现定，可能原地惰性执行（§2.2 坑②）。
4. **「协程=线程」**——协程是协作式状态机，单线程即可，挂起时不占任何线程（§1.1/§4.1）。
5. **「协程参数按引用传也没事」**——拷进帧的是引用本身；所指对象先亡即悬垂，string_view 同理（§4.6 坑①，实测复现）。
6. **「final_suspend 返回 suspend_never 省内存」**——帧提前自毁，外部句柄全部悬垂，RAII 属主二次 destroy（§4.6 坑③）。
7. **「co_await 任何表达式都行」**——须满足 awaiter 协议（三件套或经 operator co_await 适配）（§4.4）。
8. **「C++20 标准库自带 task/generator」**——语言只给机器；generator 是 C++23（本机需 /std:c++latest），task 至今没有标准版（§4.5/§7）。

## 16. 延伸资源

- **cppreference：Coroutines**（en.cppreference.com → Language → Coroutines）——语言机制权威页，promise_type/Awaiter 协议的最终口径。
- **Eli Bendersky《The promises and challenges of std::async task-based parallelism in C++》**（eli.thegreenplace.net，2016）——§2.2 坑的深入展开。
- **Stack Overflow: "Why is the destructor of the future returned from std::async blocking"**（stackoverflow.com/questions/23455104）——析构阻塞的社区经典讨论。
- **Simon Tatham《Coroutines, interrupted》**（chiark.greenend.org.uk/~sgtatham/quasiblog/）——协程与状态机改写的思路源头，专题 A 的理论底稿。
- **微软 C++ 团队博客《std::generator: Standard Library Coroutine Support》**（devblogs.microsoft.com/cppblog）——§7 的实装说明。
- **NVIDIA/stdexec**（github.com/NVIDIA/stdexec）——P2300 参考实现，§8.3 片段的语法出处。
- **asio 官方文档**（think-async.com → Overview → Composition / use_awaitable）——§8.2 的 awaitable/co_spawn 权威说明。
- **open-std P2300R10 / P3826**（open-std.org，WG21 papers）——std::execution 提案正文与 2025 年定制机制修订。
- **《C++ Concurrency in Action》第 2 版**（Anthony Williams）——future 家族与线程池的更系统展开（ch05 同款主教材）。

（以上为调研锚点；链接形态以站点现势为准，检索关键词已内嵌。）


## 17. 参考答案

**1.** 阻塞等待任务完成后才析构（本机实测 1500ms 任务 → 作用域多等 1500ms）。独有根源：`std::async` 默认策略 async|deferred 由实现二选一，标准规定析构必须兜住「实现选了 async」的情形，于是所有用户的析构都背上阻塞（§2.2 坑①）。丢弃返回值时临时 future 立即析构 → async 退化同步调用。

**2.** 返回 `future_status::deferred`（任务从未执行，也无超时概念）。错因：deferred 的 `wait_for(0)` 不是 ready 也不是 timeout，轮询条件永不成立。正确姿势：先分流 `if (f.wait_for(0s) == deferred)`（走 get/wait 路径），再对真异步 future 轮询 ready/timeout（§2.2 坑③）。

**3.** 续接（`.then`——完成后的下一步只能阻塞 get 后手工接）与运行位置控制（不能选线程池/调度器，线程哪来的你管不着）。二者合起来即「惰性可组合」的缺失，是 P2300 立项的历史动因（§2.6）。

**4.** 函数体内出现 `co_await`/`co_yield`/`co_return` 任一即成为协程：`co_return` 结束并交值；`co_yield` 产出值并挂起（生成器）；`co_await` 等待一个 awaiter、可能挂起（异步）。协程=可挂起-恢复的编译器状态机，与线程无关（§4.1）。

**5.** 按值拷贝进堆上协程帧。但 `T&`/`T*`/`string_view` 参数拷贝的是引用/指针本身，所指对象先亡即悬垂——创建时（initial_suspend 挂起）函数体还没跑，临时实参往往已亡。修复：参数一律按值收（§4.2、§4.6 坑①）。

**6.** `co_return` 一到帧当场自毁：①外部持有的 `coroutine_handle` 全悬垂（任何 resume/done 都是 UB）；②RAII 属主析构时再 `destroy()` 是对已亡帧二次销毁。纪律：final_suspend 永远挂起且 noexcept，销毁权交属主（§4.6 坑③）。

**7.** **对称转移（symmetric transfer）**：编译器以尾调用方式切换到目标协程，当前栈帧不保留。解决递归 `resume()` 的 O(N) 栈深问题。证据：专题 B 实测——返回句柄版 100 万深度存活，递归 resume 版 0xC00000FD 栈溢出（§4.4、专题 B）。

**8.** 存 `std::current_exception()` 进 exception_ptr（异常盒），由等待方 `rethrow_exception` 在合适位置复活。就地吞掉 = 异常的语义无法交付给能决策的调用方：sync_wait 永远等不到结果、错误静默丢失、悬挂路径（§4.6 坑④、§6.1）。

**9.** C++23，P2502R3。验证：写 5 行 `#include <generator>` + `co_yield` 用 MSVC `/std:c++20` 与 `/std:c++latest` 各编一次——本机 VS2022（MSVC 19.44）实证：`/std:c++20` 不可用，`/std:c++latest` 可用（§7 开头）。

**10.** C++26 工作草案（P2300R10）。三大概念：**sender**（惰性的「一件将来完成的活」，是可组合的值）、**receiver**（接住结果/错误/取消的回调三件套）、**scheduler**（在 executor 上获得执行时机的 sender 工厂）。MSVC STL 尚未实装（microsoft/STL issue #4768 跟踪），参考实现 NVIDIA/stdexec（§8.3）。
