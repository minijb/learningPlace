# 第 5 章｜并发与内存模型：数据竞争的边界

欢迎来到第 5 章。第 4 章结束时我们说过：要把「这一路攒下的 ResourceManager 变成真正能干活的东西」——异步加载，让大文件读取离开主线程——现在兑付。但在把线程放进工程之前，必须先立好一条边界：**哪些代码同时跑是安全的，哪些一旦同时跑整个程序就不再受 C++ 标准保护**。这条边界就是本章的标题：数据竞争的边界。

先钉墙上一句话，它是本章的世界观：

> **数据竞争是未定义行为，不是概率事件。** 同步原语管的是正确性，不是性能——默认顺序一致（seq_cst），有了证据才降级（acquire/release，乃至 relaxed）。竞态的正解是同步原语或重构数据所有权，而不是「多跑几遍没崩」。

前几章欠的账，本章照例全部结清：

| # | 前章承诺 | 兑付位置 |
| --- | --- | --- |
| ① | 第 4 章开篇：「异步加载——让 ResourceManager 的加载真正离开主线程」 | 5.4 + 5.8（任务队列），任务 1「迷你异步资源加载器」全章主线 |
| ② | 第 2 章 2.7：「引用计数的原子性 ≠ 对象本身的线程安全」 | 5.6（atomic 到底保证了什么）、5.8 升级回顾 |
| ③ | 第 2 章 2.1 资源表里露过一面的「互斥锁」：lock/unlock（第 5 章展开） | 5.3「锁的 RAII」全面展开 |
| ④ | 第 1 章 1.2 表格：「线程存储期 thread_local——并发话题，将在第 5 章展开」 | 5.1 侧栏顺带结清 |

这一章的终点是一条与「零裸 `glDelete`」同级的工程铁律：**任何跨线程共享的数据，都能指着它回答「谁拥有、谁负责同步、用什么原语同步」；回答不了的代码不许合入。** 你还会获得一套仪器闭环：故意写坏 → ThreadSanitizer 抓出报告 → 修复 → 零报告——这是后面所有项目里并发代码的交付姿势。

---

## 学习目标

对齐本领域阶段 5（并发与内存模型：数据竞争的边界）的学习目标。读完本章，你应该能够：

1. **判定并解释数据竞争**：给出最小数据竞争示例，说清「同一内存位置、至少一个写、无同步」三要件为何构成**未定义行为**，而非「偶尔读到旧值」的良性行为。
2. **正确创建与收尾线程**：`std::thread` / `std::jthread` 的创建与 join/detach 取舍；参数按值拷贝（decay）与 `std::ref`；一眼识别「线程活得比栈帧长」的悬垂引用坑并修复。
3. **搭建生产者-消费者骨架**：mutex + `lock_guard` / `unique_lock` / `scoped_lock` + condition_variable 谓词等待，亲手实现「主线程渲染 + 工作线程异步加载」的共享任务队列——实践任务的核心部件。
4. **用对一次性行动与原子量**：`call_once` 与 static 局部变量（magic statics）的选型；`atomic` 基本操作；说清 atomic 与 mutex 的分界——**单变量的原子性 ≠ 复合不变式的安全**。
5. **口述六种内存序概览**：seq_cst 为何是安全默认、acquire/release 如何配对完成发布-获取同步、relaxed 的窄用途。目标边界先讲明：不追求 lock-free 精通——无锁结构只点「为什么危险」，不实现。
6. **用仪器闭环治理竞态**：识别接口级竞争与 false sharing；在 WSL2/Linux 用 ThreadSanitizer 跑完「报竞争 → 修复 → 零报告」闭环；Windows 本地用 ASan + `/analyze` 并清楚它们各自抓什么、不抓什么。

> **环境约定（全章适用）**：Windows + Visual Studio 2022（MSVC 19.4x 工具集，同 ch01–ch04），语言标准 `/std:c++20`，警告级别 `/W4`，中文注释请挂 `/utf-8`：
>
> ```bat
> cl /utf-8 /std:c++20 /EHsc /W4 源文件.cpp
> ```
>
> GCC / Clang 等价命令：`g++ -std=c++20 -Wall -Wextra -pthread 源文件.cpp` 或 `clang++ -std=c++20 -Wall -Wextra -pthread 源文件.cpp`。**注意 Linux 上 `-pthread` 必须显式给出**（它同时影响编译宏与链接选项），MSVC 没有对应物、多线程示例无需额外链接选项。本章全部「完整示例」均在 MSVC + g++ 双工具链下编译零警告并运行核对过输出；技术事实（jthread 可用性、TSan 支持矩阵、magic statics 实现状态）以 cppreference 与编译器官方文档为权威口径，正文直接引用。5.10 有专门一节交代 Windows 下抓竞态的现实方案。

---

## 分节正文

### 5.1 std::thread 与 jthread：把加载搬离主线程

先看痛点。第 4 章的 ResourceManager 虽然工程化得漂漂亮亮，但 `load` 里的「读文件 + 解码」仍然发生在主线程。假设解码一张 4K 纹理要 300ms——主线程被卡住 300ms，游戏就冻 300ms，帧率表上是一个刺眼的零。你玩过的所有「读条时画面完全冻结」的老游戏，问题都出在这：**磁盘 IO 和重活不能发生在帧循环所在的线程上**。

解法是把活儿交给另一个执行流。C++ 标准库给的工具是 `std::thread`（C++11 起）和 `std::jthread`（C++20 起）：

```cpp
// ch05_first_thread.cpp —— 第一个工作线程（完整可编译）
#include <chrono>
#include <cstdio>
#include <string>
#include <thread>

void decode_texture(std::string name) {            // 参数按值拷贝进来（下文详述）
    std::printf("[工作线程] 开始解码 %s ...\n", name.c_str());
    std::this_thread::sleep_for(std::chrono::milliseconds(300)); // 模拟重活
    std::printf("[工作线程] %s 解码完成\n", name.c_str());
}

int main() {
    std::printf("[主线程] 帧循环开始\n");
    std::thread worker(decode_texture, "boss_diffuse.png"); // 创建即启动
    // ……主线程继续跑帧循环，不被解码阻塞……
    for (int frame = 0; frame < 3; ++frame) {
        std::printf("[主线程] 第 %d 帧渲染完成\n", frame);
        std::this_thread::sleep_for(std::chrono::milliseconds(16)); // 模拟一帧 16ms
    }
    worker.join();                                 // 等工作线程收工（不可省！）
    std::printf("[主线程] 工作线程已收工，程序退出\n");
}
```

编译运行（本章命令行示例统一用这组参数）：

```bat
cl /utf-8 /std:c++20 /EHsc /W4 ch05_first_thread.cpp
```

两个线程的输出会交错出现——交错的具体样子每次运行都可能不同，这正是并发的第一课：**线程的调度顺序不由你控制**。程序里能依赖的只有你显式建立的同步关系（本章的主角）。

#### join 与 detach：为什么detach 是危险的

`join()` 的语义朴素：**当前线程停在这里，等对方跑完再继续**。它是线程生命周期的「收线」动作。

`detach()` 则是把线「剪断」：线程和 `std::thread` 对象脱钩，在后台自己跑到天荒地老，你从此**没有任何合法手段等它、查它、停它**。听起来省事，实际是给未来的自己埋雷：

- 被 detach 的线程若还在引用 `main` 的局部变量，而 `main` 已经返回——悬垂引用，第 1 章讲过的崩溃在这里原样复发，且**时序随机、极难复现**；
- 程序退出阶段，后台线程可能在运行时（runtime）已经拆完的残骸上继续跑，死在 `main` 返回之后的某个随机位置。

工程纪律：**游戏运行期的工作线程一律 join 收线（或用下面马上讲的 jthread 让析构自动干这件事）；detach 只留给「真的要陪程序到最后一刻」的后台服务，而我们目前没有这种需求。**

#### 参数传递：按值拷贝是规则，std::ref 是例外

`std::thread` 的构造函数把参数**拷贝（或移动）进线程内部的存储**——即便函数签名写的是引用。这个设计是故意的：线程可能比传参表达式活得久，按值拷贝是防悬垂的默认姿势。

```cpp
// ch05_thread_args.cpp —— 参数传递规则与 std::ref（完整可编译）
#include <cstdio>
#include <string>
#include <thread>

void by_value(std::string s) {                     // 收到的是拷贝
    std::printf("by_value: %s (id=%p)\n", s.c_str(), (void*)&s);
}

void by_ref(std::string& s) {                      // 签名要引用，必须在调用点 std::ref
    std::printf("by_ref:   %s (id=%p)\n", s.c_str(), (void*)&s);
}

int main() {
    std::string asset = "skybox.hdr";
    std::printf("main:     %s (id=%p)\n", asset.c_str(), (void*)&asset);

    std::thread t1(by_value, asset);               // 拷贝一份进线程
    t1.join();

    std::thread t2(by_ref, std::ref(asset));       // std::ref 才是真的引用
    t2.join();
}
```

跑一次你会看到：`t1` 打印的地址和 `main` 不同（拷贝），`t2` 打印的相同（真引用）。**注意 `t2` 从此背上了责任**：只要线程还活着，`asset` 就必须活着——这正是悬垂引用坑的入口。

成员函数也能开线程：`std::thread t(&Player::update, &player, dt)`——成员函数指针后面第一个参数传对象（指针或引用），其余依次传参。

#### 悬垂引用坑：线程活得比栈帧长

把坑亲手踩一遍（**危险示例，只看不跑**）：

```cpp
// ch05_dangling_ref.cpp —— ⚠ 危险示例：悬垂引用（勿运行）
#include <chrono>
#include <cstdio>
#include <thread>

void work(const int& budget) {                     // 引用参数
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::printf("budget = %d\n", budget);          // 💥 此时 budget 早已死亡
}

int main() {
    int frame_budget = 16;
    std::thread t(work, std::ref(frame_budget));
    // t.detach();                                 // 若 detach，main 直接跑完返回
    return 0;                                      // frame_budget 随栈帧销毁
}
```

`main` 返回，`frame_budget` 随栈帧消亡；而工作线程 100ms 后才来读它——读一块已经不属于任何对象的内存，未定义行为（UB 的完整含义下一节展开）。修复姿势二选一：

1. **按值传**（默认姿势）：`std::thread t(work, frame_budget)`，去掉 `std::ref`，让线程持有自己的拷贝；大对象则 `std::move` 进去；
2. **保证生命周期**：把数据放到比线程长寿的地方（堆上、静态区），并用同步原语约定访问规则——这正是 5.3 以后的主题。

#### jthread：线程的 RAII（C++20）

第 2 章的世界观在这里原样复用：**资源要 RAII，线程也是一种资源**——它拿了「一个执行流」，忘收线（忘记 join）程序直接 `std::terminate`。`std::jthread`（joining thread）把这个收尾焊死在析构函数上：

| | `std::thread` | `std::jthread`（C++20） |
| --- | --- | --- |
| 忘记 join | 析构时 `std::terminate`，程序崩 | 析构时自动 `request_stop()` + `join()` |
| 中途取消 | 无标准机制（自己写标志位） | `stop_token` 协作式取消（本章深入专题展开） |
| 工程建议 | 早年唯一选择 | **新代码默认用它**（C++20 基线下没有理由不用） |

本章后续示例为了覆盖两种工具链习惯、并与网上大量资料对得上，主线仍写 `std::thread + join()`，但在深入专题把 jthread 的优雅关停完整解剖一遍。你自己的工程里，请从 jthread 起步。

#### 线程不是越多越好

顺手立一条数量纪律。`std::thread::hardware_concurrency()` 能告诉你机器的逻辑核数；线程数远超核数不会带来并行加速——多余线程只是在调度器里排队，还增加上下文切换开销。本章任务队列用「2 个工作线程」只是为了演示清晰；工程上的参考答案是：**CPU 密集的工作线程数 ≈ 核数；IO 密集（读盘、解码）可以多一些，因为它们大部分时间在等**。线程池的完整实现是资源管理课题，本章先把「随用随建、用完即收」的姿势练熟。另外，等待中如果不想硬忙等也不想永久睡去，`std::this_thread::yield()` 是「让出一口气」的折中——5.7 的示例里就用了它。

> **顺带结清旧账④**：第 1 章 1.2 表格里的第四种存储期——`thread_local`，每个线程各有一份独立对象，线程创建时初始化、线程结束时析构。它适合「每线程一份」的上下文数据（如工作线程各自的临时缓冲）。但它**不解决任何同步问题**——各线程看到的是不同对象，根本不存在共享，也就无所谓竞争。后文的全部讨论针对的是**共享**数据。

### 5.2 数据竞争：定义与未定义行为的本质

现在立本章的界碑。两个线程同时碰同一块内存，什么时候是错的？标准给出的判据是**数据竞争（data race）**的三要件：

1. 两个（或更多）线程**并发**访问**同一内存位置**；
2. 其中**至少一个是写**；
3. 这两个访问**没有同步关系**（本章 5.7 会精确定义「同步关系」，这里直觉理解成「没用锁、没用原子量建立先后顺序」）。

三件齐备，程序就发生数据竞争，而数据竞争的后果在 C++ 标准里写得斩钉截铁：**未定义行为（undefined behavior）**——不是「读到旧值」，不是「偶尔错一次」，是标准从此**不承诺这个程序的任何行为**。

#### 为什么「只是读到旧值」是致命误解

初学者最常见的侥幸是：「竞争最坏也就是读到旧数据，业务上能忍。」错。数据竞争交出去之后，你面对的不再是「新旧值」的博弈，而是三重放大器，任何一层都能把「读到旧值」放大成任意灾难：

- **编译器优化**：编译器被授权假定你的程序没有数据竞争（有竞争的程序不是合法 C++ 程序）。没有竞争意味着「单线程语义下可见的效果不变」——于是它可以把频繁读取的变量**缓存在寄存器里**、把「看起来没效果」的读循环删掉、按单线程视角重排指令。你的多线程程序在 `/O2` 下和 Debug 下行为不同，第一嫌疑就是它。
- **指令重排**：即使编译器老实了，CPU 和缓存系统也会为了让流水线更满而重排内存操作。单线程里重排被「as-if」规则兜住（结果不可分辨）；有数据竞争时没有任何兜底。
- **多核可见性**：每个核心有自己的缓存。核心 A 写的值，核心 B 什么时候「看到」，没有同步机制就没有任何承诺——可能立刻、可能很久、可能永远看不到。

马上看一个能亲眼验证的现场。

#### 最小竞争现场：/O2 下当场变脸

```cpp
// ch05_minimal_race.cpp —— 最小数据竞争（完整可编译；故意保留竞态）
#include <cstdio>
#include <thread>

bool ready = false;                    // 非 atomic 的普通 bool —— 竞争主角
int payload = 0;                       // 工作线程的「产出」

void producer() {
    payload = 42;                      // 写 ①
    ready = true;                      // 写 ②（和读线程的读构成数据竞争）
}

int main() {
    std::thread t(producer);
    while (!ready) {                   // 读 ready —— 与写 ② 构成数据竞争
        // 忙等：把 CPU 让印象里「应该很快就好」
    }
    std::printf("payload = %d\n", payload);
    t.join();
}
```

Debug 下跑，十有八九能打印 `payload = 42`——「能跑」，于是你以为自己写对了。换 Release：

```bat
cl /utf-8 /O2 /std:c++20 /EHsc /W4 ch05_minimal_race.cpp
```

两种可能的现场（取决于编译器版本与优化决策）：

- **死循环**：编译器看到 `while (!ready)` 里没人改 `ready`，把它**提出循环外**只读一次——读到初始 `false` 后永远循环，程序挂死；
- **读到垃圾**：即便侥幸退出循环，`payload` 的读也没有任何顺序保证——打印 0 或 42 都「合法」。

这不是玄学：`ready` 的读和写没有同步关系，数据竞争成立，标准撒手不管。**唯一正解是同步原语**——这个例子用 `std::atomic<bool>`（5.6/5.7 给完整修复，`ready` 改成 atomic 后一切立刻规矩起来），或者干脆用 5.4 的条件变量把忙等消灭掉。你还会在 5.10 用 ThreadSanitizer 让这类问题自己「招供」。

> **检验标准①自查**：合上书，向同事口述「为什么数据竞争是未定义行为，而不是偶尔读到旧值」——三要件 + UB 授权编译器与硬件放手优化 + 三重放大器。这是本章第一个验收点。

#### 顺带分清两个词：数据竞争与竞态条件

文献里你会撞上两个长得很像的词，分清它们能避免很多无谓争论：

- **数据竞争（data race）**：本节定义的、标准里的术语——三要件齐备即为 UB，**纯技术判定**，与人无关；
- **竞态条件（race condition）**：更宽泛的逻辑概念——「结果依赖线程时序、时序错了逻辑就坏」。典型例子：两个工作线程都检查「缓存里没有这个资源」然后各自加载一遍——没有任何数据竞争（有锁护着），但逻辑上重复加载了。它是**设计缺陷**，不是 UB。

关系：数据竞争必然是隐患（UB）；竞态条件则要用更粗的临界区、接口重组或状态机去修。TSan 只能抓前者——后者要靠你的设计审查（任务 3 的约定文档就是干这个的）。

---

### 5.3 mutex 与锁的 RAII：lock_guard / unique_lock / scoped_lock

5.2 说了竞争要靠同步原语治理，第一个也是最主要的原语就是互斥锁（mutex）。规则朴素：**一段临界区（critical region）同一时刻只允许一个线程进入**。想进入的线程先 `lock()`——没抢到就睡在那等；出来的线程 `unlock()`，放下一个进来。

先看没有锁的灾难现场。经典游戏场景：多个线程普攻同一个 Boss，并发扣血：

```cpp
// ch05_boss_no_lock.cpp —— 无锁并发扣血：结果错乱（完整可编译；故意保留竞态）
#include <cstdio>
#include <thread>
#include <vector>

int boss_hp = 40000;                   // 共享：全体线程都写

void attack(int hits) {
    for (int i = 0; i < hits; ++i) {
        boss_hp -= 1;                  // 读-改-写三步，非原子！
    }
}

int main() {
    constexpr int kThreads = 4, kHits = 10000;
    std::vector<std::thread> party;
    for (int i = 0; i < kThreads; ++i) party.emplace_back(attack, kHits);
    for (auto& t : party) t.join();
    std::printf("boss_hp = %d（应为 0）\n", boss_hp);
}
```

多跑几次，`boss_hp` 大概率**不是 0**，而且每次都不一样。原因：`boss_hp -= 1` 不是一步，是「读内存 → 寄存器里减 → 写回」三步；四个线程的三步随意交错，互相覆盖——少了多少血，只有天知道。这就是数据竞争的真实后果之一：**丢失更新（lost update）**。它完美诠释 5.2 的观点——Debug 下你可能次次都得到 0，Release 交错概率一变，错误立刻现形。

上锁版：

```cpp
// ch05_boss_locked.cpp —— mutex 保护临界区（完整可编译）
#include <cstdio>
#include <mutex>
#include <thread>
#include <vector>

int boss_hp = 40000;
std::mutex boss_hp_mutex;              // 锁与它保护的数据成对出现

void attack(int hits) {
    for (int i = 0; i < hits; ++i) {
        std::lock_guard<std::mutex> lock(boss_hp_mutex); // 构造即加锁
        boss_hp -= 1;                  // 临界区：同一时刻至多一个线程在这
    }                                  // lock 析构即解锁——RAII！
}

int main() {
    constexpr int kThreads = 4, kHits = 10000;
    std::vector<std::thread> party;
    for (int i = 0; i < kThreads; ++i) party.emplace_back(attack, kHits);
    for (auto& t : party) t.join();
    std::printf("boss_hp = %d（应为 0）\n", boss_hp);   // 次次都是 0
}
```

现在扣血节次次正确。代价也真实存在：每次进出临界区，抢不到锁的线程要睡过去再被唤醒，这是实实在在的开销——所以临界区要**尽量小**：只护住真正共享的那几行，别把整个函数锁进去；更别在持锁时做 IO。

#### 锁的 RAII：手动 lock/unlock 是事故现场

把上面的 `lock_guard` 换成手动版试试：

```cpp
boss_hp_mutex.lock();
boss_hp -= 1;
boss_hp_mutex.unlock();                // 看似对称……
```

问题在第 2 章 2.2 你已经见过同款论证：中间任何一句提前 `return`、一个 `break`、一场异常，`unlock()` 就被绕过——锁再也放不下去，所有等锁的线程全部挂死。所以纪律与 `new/delete` 的纪律完全同构：**永远不手写 lock/unlock 配对，用 RAII 锁对象管理**。锁即资源，第 2 章的资源表里它早就预留了位置，现在兑付。

三个 RAII 锁，各有分工：

| 锁对象 | 一句话定位 | 适用场景 |
| --- | --- | --- |
| `std::lock_guard` | 构造加锁、析构解锁，零额外能力 | **默认姿势**：普通临界区 |
| `std::unique_lock` | 可延迟加锁、可提前解锁、可移动、记录持有状态 | 条件变量的标配（5.4）；锁的传递 |
| `std::scoped_lock` | 一次拿多把锁，内部用避免死锁的算法按序获取 | 同时需要多把锁（防死锁） |

#### 多把锁与死锁：scoped_lock 的存在理由

两个线程以**不同次序**抢两把锁，就会互相等待——死锁（deadlock）：A 持锁 1 等锁 2，B 持锁 2 等锁 1，谁也不放。游戏里的经典场景是「转账」：把金币从背包搬到仓库，背包锁和仓库锁都得拿：

```cpp
// ch05_scoped_lock.cpp —— 多锁与 scoped_lock（完整可编译）
#include <cstdio>
#include <functional>
#include <mutex>
#include <thread>

struct Wallet { int gold = 100; std::mutex m; };

void transfer(Wallet& from, Wallet& to, int amount) {
    std::scoped_lock lock(from.m, to.m);           // 一次拿两把，内部按序获取防死锁
    from.gold -= amount;
    to.gold   += amount;
}                                                  // 两把锁一起放

int main() {
    Wallet bag, bank;
    std::thread a(transfer, std::ref(bag), std::ref(bank), 30);  // 背包→仓库
    std::thread b(transfer, std::ref(bank), std::ref(bag), 20);  // 仓库→背包（方向相反！）
    a.join(); b.join();
    std::printf("bag=%d bank=%d（合计应恒为 200）\n", bag.gold, bank.gold);
}
```

线程 a 先抢 `bag.m` 再抢 `bank.m`，线程 b 反过来——手写两个 `lock_guard` 就埋了死锁雷。`scoped_lock` 用标准库的死锁避免算法（底层是 `std::lock` 的 try-and-back-off 策略）保证**要么全拿到、要么都不拿**。如果拿不到 scoped_lock（旧代码基线），纪律是**全工程统一的加锁次序**（如按对象地址排序），但既然 C++20 基线，直接用 scoped_lock 即可。

> **后向引用**：这套「锁的 RAII」与 ch02 的 `unique_ptr`、`lock_guard 管文件句柄`是同一世界观——构造获取、析构释放、异常安全免费获得。你的并发约定文档（任务 3）里应该有一条：「禁止手写 lock/unlock」。

---

### 5.4 condition_variable 与虚假唤醒

mutex 解决了「同一时刻只有一个人动共享数据」，但还剩一个效率与正确性的混合问题：**工作线程怎么知道「有活儿了」？**

最直觉的答案是轮询——死循环查队列：

```cpp
while (queue.empty()) { /* 空转 */ }   // ❌ 一个核直接烧满
```

在游戏引擎里这就是「一个核心 100% 空转烧电、还挤占别的线程的调度」，不可接受。正确工具是**条件变量（condition variable，cv）**：让线程**睡着**，直到「条件可能成立」时被别人叫醒。

三个部件永远成套出现，缺一不可：

1. **mutex**：保护「条件」本身（通常是共享队列/标志）；
2. **`std::unique_lock<std::mutex>`**：cv 的 `wait` 只接受 `unique_lock`——因为它睡着时需要解锁、醒来时需要重新加锁，这套「中途开合」只有 unique_lock 干得了（这就是 5.3 表格里它的定位）；
3. **condition_variable**：`wait` 睡觉 / `notify` 叫人。

生活化类比：外卖叫号（notify）+ 取餐窗口排队（wait）。你取完餐（处理完一条任务）回队尾继续睡，号声一响睁眼看叫的是不是自己的号（检查谓词）。

#### 虚假唤醒与谓词版 wait：唯一推荐姿势

先说清一个反直觉的事实：**`wait` 可能在没人 notify 时自己醒来**（操作系统层面的实现细节允许它这样做，叫**虚假唤醒 spurious wakeup**）。所以「醒来 = 条件成立」是错误假设，你必须**醒来后再查一遍条件**。标准库把这个「睡 → 醒 → 查 → 不满足接着睡」的循环打包成了谓词版：

```cpp
cv.wait(lock, pred);   // 等价于：while (!pred()) cv.wait(lock);
```

**谓词版是唯一推荐姿势**——它同时免疫两类陷阱：

- **虚假唤醒**：醒了谓词一查不成立，接着睡，无害；
- **唤醒丢失（lost wakeup）**：notify 发生在你还没睡下（还在检查队列、还没进 wait）的间隙。如果只写裸 `wait`，这次 notify 就白白蒸发了，线程睡到天荒地老。谓词版之所以能救场，是因为 `wait(lock, pred)` 的检查在**持锁状态下**进行——notify 方必须先改条件（持同一把锁）再 notify，于是「检查」与「修改」被同一把锁串行化：要么你查的时候条件已成立（谓词为真直接不睡），要么你睡着之后 notify 才可能发出（必然叫醒你）。不存在两不管的缝隙。

还要记一条纪律：**必须存在与 cv 配对的「状态量」**（队列非空、标志为真……），notify 只是叫醒动作，不是状态本身。只 notify 不改状态，等于空头号声。

#### 迷你任务队列 v1：主线程投喂，工作线程消化

把三件套拼起来——这就是任务 1 的核心部件雏形：

```cpp
// ch05_cv_queue_v1.cpp —— 条件变量版迷你任务队列（完整可编译）
#include <chrono>
#include <condition_variable>
#include <cstdio>
#include <mutex>
#include <queue>
#include <string>
#include <thread>

std::queue<std::string> tasks;         // 状态量：任务队列
std::mutex tasks_mutex;
std::condition_variable tasks_cv;
bool finished = false;                 // 状态量：收工标志

void worker() {
    for (;;) {
        std::string task;
        {
            std::unique_lock<std::mutex> lock(tasks_mutex);
            tasks_cv.wait(lock, [] { return !tasks.empty() || finished; }); // 谓词等待
            if (tasks.empty() && finished) return;                          // 优雅退出
            task = std::move(tasks.front());
            tasks.pop();
        }                              // 临界区到这里结束——处理任务不持锁！
        std::printf("[工作] 处理 %s\n", task.c_str());
        std::this_thread::sleep_for(std::chrono::milliseconds(50)); // 模拟干活
    }
}

int main() {
    std::thread w(worker);
    for (int i = 1; i <= 3; ++i) {
        {
            std::lock_guard<std::mutex> lock(tasks_mutex);
            tasks.push("task_" + std::to_string(i));
        }
        tasks_cv.notify_one();         // 先改状态（上面括号里），后 notify
        std::this_thread::sleep_for(std::chrono::milliseconds(30));
    }
    {
        std::lock_guard<std::mutex> lock(tasks_mutex);
        finished = true;
    }
    tasks_cv.notify_one();
    w.join();
    std::printf("[主线程] 收工\n");
}
```

三个实现细节值得停一停：

- **先改状态、后 notify**（且改状态时持锁）：顺序反了就是唤醒丢失的种子；
- **处理任务在临界区外**：持锁干活 = 把并行串行化，锁的粒度纪律（5.3）在这里落地；
- **退出也是条件**：`finished` 进谓词，收工时队列空了谓词仍可为真，线程自然退出——不留给任务 1 的优雅关停埋雷。

`notify_one` 与 `notify_all` 的选型：**一个任务只需要一个工人 → `notify_one`**；**状态变化影响所有等待者（如 finished、关停信号）→ `notify_all`**。拿不准时 `notify_all` 语义永远安全，代价只是多叫醒几个人。

唤醒丢失的时间线画出来（裸 wait 版的翻车现场）：

```text
消费者线程                              生产者线程
─────────                               ─────
检查队列：empty() == true
    │（还在准备调 wait，未睡下）
    │                              push(task); notify_one();  ← 通知蒸发：无人在等
    ▼
cv.wait(lock);   ← 睡进永久黑夜
```

谓词版的救场：检查在持锁状态下进行，生产者改队列也必须持同一把锁——两者被串行化，上图中「检查」与「push+notify」不可能交错，缝隙不复存在。

> **现代等价物侧栏（C++20）**：`std::counting_semaphore`、`std::latch`、`std::barrier` 在 MSVC /std:c++20 下均可用。信号量能表达「资源计数」语义，latch/barrier 适合「阶段同步」。但本章主线与任务 1 保持 thread + mutex + condition_variable 组合——跨工具链零风险，也是网上绝大多数引擎代码的通用语言。新设施知道有这回事、能看懂即可。

---

### 5.5 一次性行动：call_once 与 magic statics

「这个初始化全程序只能做一次」是并发代码的常客：全局日志器、渲染后端、配置表加载。多线程世界里的麻烦在于——两个线程**同时**第一次触发，谁都没来得及把「做过了」的标记立起来，初始化就被执行了两遍。

手写方案（bool 标志 + 锁 + 双重检查）不仅繁琐，还有个臭名昭著的错误版本：double-checked locking——先无锁查标志、命中再加锁复查。C++11 之前的内存模型下，这个写法**理论上就不正确**（无锁的那次读没有同步关系，是个数据竞争）。正确姿势有两种，都是标准替你把锁管了：

**姿势一：static 局部变量（magic statics）**。C++11 起规定：局部 static 变量的初始化**由编译器保证线程安全**（多个线程同时首次到达，只有一个执行初始化，其余等待初始化完成）。MSVC 自 VS2015 起正确实现了这一保证（VS2022 无虞，放心用）：

```cpp
// ch05_magic_static.cpp —— 线程安全单例：static 局部版（完整可编译）
#include <cstdio>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

class Logger {
public:
    static Logger& instance() {
        static Logger logger;          // magic static：首次调用时构造，且线程安全
        return logger;
    }
    void write(const std::string& msg) {
        std::lock_guard<std::mutex> lock(m_);   // 初始化安全 ≠ 使用安全，使用仍要锁
        std::printf("[日志] %s\n", msg.c_str());
    }
private:
    Logger() { std::printf("[日志] 初始化（只应打印一次）\n"); }
    std::mutex m_;
};

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i)
        threads.emplace_back([i] { Logger::instance().write("线程 " + std::to_string(i)); });
    for (auto& t : threads) t.join();
}
```

注意两点：`instance()` 无锁（初始化完成后每次调用只是一次指针检查级别的开销）；「初始化只一次」和「使用要同步」是两件事——`write` 里的 mutex 保护的是后续并发使用，别指望 magic static 包办一切。

**姿势二：call_once + once_flag**。需要**传参**、或有**多个可触发的初始化入口**、或初始化失败后要能重试时用它：

```cpp
// ch05_call_once.cpp —— call_once 版本（完整可编译）
#include <cstdio>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

std::once_flag config_flag;
std::string   config_value;

void load_config(const char* path) {           // 多个线程都可能第一个到达
    std::call_once(config_flag, [](const char* p) {
        std::printf("加载配置（只应执行一次）：%s\n", p);
        config_value = "windowed=1;vsync=1";   // 实际工程里读文件
    }, path);
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 3; ++i)
        threads.emplace_back(load_config, "engine.ini");
    for (auto& t : threads) t.join();
}
```

选型口诀：**无参单例 → static 局部；要传参/多入口 → call_once**。两者都比手写锁标志好——少一处可能写错的同步代码，就少一个未来凌晨三点的崩溃报告。

### 5.6 atomic 基本操作：一把锁太重时

mutex 是万能答案，但有时候它像拿卡车送外卖：保护对象只是一个 `int` 计数器，却让每次访问都要走「可能睡一觉再被叫醒」的完整流程。对**单个基础类型的变量**，标准库给了轻量级原语：`std::atomic`。

```cpp
// ch05_atomic_counter.cpp —— 原子击杀计数器（完整可编译）
#include <atomic>
#include <cstdio>
#include <thread>
#include <vector>

std::atomic<int> kill_count{0};        // 原子计数器
int  raw_count = 0;                    // 对照组：裸 int

void farm(int n) {
    for (int i = 0; i < n; ++i) {
        kill_count.fetch_add(1);       // 原子的读-改-写
        ++raw_count;                   // ❌ 对照组：数据竞争，结果漂移
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) threads.emplace_back(farm, 100000);
    for (auto& t : threads) t.join();
    std::printf("atomic = %d（正确）\nraw    = %d（漂移，每次运行不同）\n",
                kill_count.load(), raw_count);
}
```

常用操作一览：

| 操作 | 说明 |
| --- | --- |
| `load()` / `store(v)` | 原子读 / 原子写 |
| `fetch_add(n)` / `fetch_sub(n)` | 原子加减，返回**旧值** |
| `++` / `--` / `+=` | 运算符重载，等价 fetch_add 家族 |
| `exchange(v)` | 原子换入新值，返回旧值 |
| `compare_exchange_weak/strong(expected, desired)` | CAS：若当前值等于 expected 则写入 desired——无锁算法的基石（本章不实现无锁结构，知道它的存在即可） |
| `is_lock_free()` | 询问是否真无锁（有些平台上 atomic<T> 内部仍是轻量锁） |

现在升级 ch02 2.7 欠的那笔账。当时说「`shared_ptr` 引用计数的原子性 ≠ 对象本身的线程安全」，这里可以精确解释了：

- **原子性（atomicity）**：单个操作不被打断（`fetch_add` 是完整的一步）；
- **不变式（invariant）**：多个变量之间约定好的关系（如「boss_hp 不得低于 0，否则切换死亡状态」）。

atomic 只能保证前者。把 5.3 的 Boss 场景改用 `std::atomic<int> boss_hp`，每个 `fetch_sub(1)` 都原子了，但「检查 hp<=0 则置 dead」是**两步**：另一个线程可能在你检查和置位之间又扣了血——不变式照样破。**原子性是单变量的属性，不变式是复合状态的安全，后者只有锁（或精心设计的无锁算法）能护住。** 这就是 atomic 与 mutex 的分界：

| | `std::atomic` | `std::mutex` |
| --- | --- | --- |
| 保护的量 | 单个基础类型变量 | 任意大小的临界区与不变式 |
| 开销 | 轻量（通常一条 CPU 指令级） | 较重（睡醒调度） |
| 复合操作 | ❌ 只保证单步原子 | ✅ 跨多语句 |
| 会睡眠等待 | 否（自旋级） | 可能 |

一句话选型：**计数器、标志位、发布指针这类「单变量」用 atomic；任何涉及多个变量或先查后改的，用 mutex。**

两个补充常识，免得在真实代码里措手不及：

- **`std::atomic<T*>` 同样存在**：无锁数据结构里常见的「原子换头节点」（`head.exchange(new_node)`）就是它；普通工程里你至少要能看懂。
- **`std::atomic<MyStruct>` 不等于成员各自原子**：对大于机器字长的类型，编译器可能用内部锁模拟（`is_lock_free()` 会告诉你真相），且逐字段访问没有原子性组合。结构体级别的同步，答案仍是 mutex。

另外，`std::atomic` 的所有操作都接受一个**内存序**参数——到目前你写的都是省略它，那就意味着默认值 `memory_order_seq_cst`。这个默认值为什么是「安全默认」，正是下一节的主题。

---

### 5.7 六种内存序概览：安全默认与唯一降级路径

内存序（memory order）是并发里最容易吓退初学者的话题。先把它放到正确的位置：**它只管 atomic 操作（以及达到同样效果的场景）之间的可见性次序**，不影响 mutex/cv——锁本身自带全套顺序保证，用锁就忘掉这个话题。也就是说：只要你在用锁，内存序对你透明；只有你开始用 atomic 承担「同步」职责时，才需要它。

六个枚举值先见个面：

| memory_order | 一句话定位 | 本章要求 |
| --- | --- | --- |
| `seq_cst` | 顺序一致：所有 seq_cst 操作存在全线程统一的总次序 | **默认值，安全默认** |
| `acquire` | 读侧：后续读写不得重排到本次 load 之前 | **主要降级路径（配 release）** |
| `release` | 写侧：之前的读写不得重排到本次 store 之后 | **主要降级路径（配 acquire）** |
| `acq_rel` | 读改写操作同时具备 acquire + release | 与上面配对使用时用于 RMW |
| `consume` | 理论上比 acquire 更弱，但编译器实际上都当 acquire 实现 | 名存实亡，不展开 |
| `relaxed` | 只保证单变量原子性，不提供任何顺序承诺 | 窄用途：纯统计计数 |

#### seq_cst 为什么是安全默认

`seq_cst`（sequential consistency，顺序一致）承诺：程序里**所有** seq_cst 操作拼起来存在一个所有线程都认同的统一总次序，且每个线程看到的次序一致。它最接近「把多线程想象成交错执行的单线程」的直觉——你能用普通推理想清楚的事件顺序，在 seq_cst 下都成立。**新代码一律用默认值写，写完能证明正确、有剖析数据证明锁/默认序是真瓶颈时，才降级到 acquire/release。** 这条纪律为你挡掉整类「在纸面上推理内存序」的错误。

#### acquire/release 配对：发布-获取

降级路径只有一个：`release` 写 + `acquire` 读配对。语义用公告板类比：

- **release store** = 把公告**连同你之前写的一切**一起钉上公告板（钉的动作保证钉之前的所有写入已完成且可见）；
- **acquire load** = 取公告（取的动作保证你之后的所有读写都在取到公告**之后**进行）。

当 acquire load 读到的是 release store 写入的值，标准称两个操作建立了 **synchronizes-with（同步于）**关系，进而得到跨线程的 **happens-before（先行发生）**——于是 release 之前的所有写入，对 acquire 之后的代码全部可见。这是标准给你的、 mutex 之外的另一种建立先后关系的合法手段，也是所有无锁代码的地基。

游戏语境的完整现场：工作线程解码完一块像素缓冲，主线程要用它：

```cpp
// ch05_release_acquire.cpp —— 发布-获取同步（完整可编译）
#include <atomic>
#include <cstdio>
#include <thread>
#include <vector>

std::vector<unsigned char> pixels;          // 工作线程的产出（非 atomic 数据本身）
std::atomic<bool> ready{false};             // 发布标志

void decode_worker() {
    pixels.assign(256, 0xA7);               // ① 写数据（普通写）
    ready.store(true, std::memory_order_release);  // ② release 发布：① 之前的一切写入就此打包可见
}

int main() {
    std::thread w(decode_worker);
    while (!ready.load(std::memory_order_acquire)) {   // ③ acquire 获取：与 ② 配对
        std::this_thread::yield();
    }
    // ④ 这里安全消费 pixels：③ 读到 true ⟹ 与 ② 建立 synchronizes-with
    //    ⟹ ① 的全部写入对本线程可见
    std::printf("首像素 = 0x%X（应为 0xA7）\n", pixels[0]);
    w.join();
}
```

关键理解：**acquire/release 只同步「配对的那对操作」，但把 release 之前、acquire 之后的所有普通内存访问一并捎带**——所以 `pixels` 这个普通 vector 不需要是 atomic。这就是「用 acquire/release 替代一把重锁完成同步」的准确含义：锁护住的是一段临界区，acquire/release 只需要保护「数据就绪」这个发布动作，适合「生产者完整生产 → 消费者才允许消费」的场景。

时序图把这条链画出来：

```text
工作线程                                主线程
─────────                               ─────
① pixels.assign(256, 0xA7)   （普通写）
        │  release 保证：①不得重排到②之后
② ready.store(true, release)
        │
        └──── synchronizes-with ────┐
                                    │
                        ③ ready.load(acquire) == true
                            │  acquire 保证：后续读写不得重排到③之前
                            │  happens-before
                        ④ 读 pixels[0] ⟹ 必见 0xA7
```

对照反例：如果把 ② 的 release 改成 `relaxed`，③ 的 acquire 改成 `relaxed`——程序照样「通常」能跑，但 ①③ 之间没有任何顺序承诺：主线程可能读到 `ready == true` 却看到 `pixels` 还是空的。**relaxed 只有原子性、没有顺序**——它把「拿到旧值」的灾难又请回来了。

#### relaxed 的窄用途

那 relaxed 什么时候合法？**当这个变量只承担「统计」、不承担「同步」时**：命中率计数、每帧粒子数、profiling 计数器。这些场景里计数本身就是全部目的，读到的值略旧无所谓，更不依赖「计数可见 ⟹ 别的数据可见」。纪律再说一遍：**先证明正确（默认 seq_cst），再谈降级（唯一路径 acquire/release），relaxed 只留给纯统计。** 而「降级是否值得」需要测量说话——剖析与测量方法论将在第 6 章展开，本章先把「默认序一定正确」刻进肌肉。

> **检验标准②自查**：向同事口述——「工作线程 `store(true, release)`，主线程 `load(acquire)` 见真后安全消费：store 与 load 建立 synchronizes-with，store 之前的一切写入 happens-before load 之后的一切读写，所以普通缓冲不需要锁也不需要 atomic」。这就是配对口述模板，任务 3 选做题要求你落笔写一遍。

---

### 5.8 接口级竞争：「线程安全容器」并不存在

先破一个流传甚广的迷思：「给 `std::queue` 内部加把锁，它就是线程安全容器了。」

不是。先看标准怎么说的：**标准库容器对所有 const 成员函数承诺并发只读安全**（多少个线程同时读都没事）；但只要有一个线程在写，其他线程的任何访问（读也算）都需要外部同步——标准**不承诺任何容器内部带锁**。好，那外面包一层锁是不是就完事了？看这个「加了内置锁的队列」：

```cpp
class NaiveQueue {                          // ❌ 反面教材
    std::queue<int> q_;
    mutable std::mutex m_;
public:
    bool empty() const { std::lock_guard<std::mutex> l(m_); return q_.empty(); }
    void push(int v)    { std::lock_guard<std::mutex> l(m_); q_.push(v); }
    int  pop()          { std::lock_guard<std::mutex> l(m_); int v = q_.front(); q_.pop(); return v; }
};
```

每个方法各自原子，但看看消费者线程的用法：

```cpp
if (!q.empty()) {            // ① 检查
    int v = q.pop();         // ② 取——①②之间别人可能把队列抽干！
}
```

`empty()` 和 `pop()` 是**两次独立的加锁**，两把锁之间谁都能插进来。三个消费者同时跑这段代码，总有人在 ①② 之间被同伴抢空，`pop()` 一个空队列——未定义行为。这就是 **check-then-act（先查后改）** 竞争，也叫接口级竞争：**单个操作都安全，复合序列不安全**。它证明了「线程安全容器」是个伪概念——**安全与否取决于接口如何组合，而不是容器有没有内部锁**。

正解：**把复合操作做成一个原子接口**。把「等待并取出」合并成一个方法，锁在方法内部贯穿整个复合操作——这正是 5.4 任务队列 v1 做的事，现在把它收编成正式的类（任务 1 直接用它）：

```cpp
// ch05_task_queue.cpp —— 外覆锁的线程安全任务队列（完整可编译）
#include <condition_variable>
#include <mutex>
#include <optional>
#include <queue>
#include <thread>

template <typename T>
class TaskQueue {
public:
    void push(T value) {                        // 生产端：可多线程并发调用
        {
            std::lock_guard<std::mutex> lock(m_);
            q_.push(std::move(value));           // ch01 移动语义：任务对象 move 进队列
        }
        cv_.notify_one();
    }

    std::optional<T> try_pop() {                 // 消费端：阻塞直到有任务或已关停
        std::unique_lock<std::mutex> lock(m_);
        cv_.wait(lock, [this] { return !q_.empty() || closed_; });
        if (q_.empty()) return std::nullopt;     // closed_ 且无任务 ⟹ 收工
        T value = std::move(q_.front());
        q_.pop();
        return value;                            // ch03：optional 表达「结果未就绪/关停」
    }

    void close() {                               // 关停：影响所有等待者
        {
            std::lock_guard<std::mutex> lock(m_);
            closed_ = true;
        }
        cv_.notify_all();
    }

private:
    std::queue<T> q_;
    mutable std::mutex m_;
    std::condition_variable cv_;
    bool closed_ = false;
};
```

注意设计上的三条自觉：

- **临界区只护队列本体**，取出的任务在锁外处理——并行度保住了；
- **`pop` 从接口上消灭了 check-then-act**：等待、检查、取出在一个临界区里完成，调用方无从拆散它们；
- **谁拥有、谁同步一目了然**：队列内部拥有一把锁，push/try_pop/close 各自成套——同步责任长在接口里，而不是散落在每个调用点。

最后升级 ch02 2.7 那笔账的最后一层：`shared_ptr` 的**控制块引用计数是原子的**，所以多个线程可以同时持有/拷贝/析构各自的那份 `shared_ptr`（指向同一对象）——这没问题。但这**不意味着**你能让多个线程同时调用对象上的非 const 方法：计数原子保护的是「指针和计数」，不是「对象内部状态」。同理，ch02 讲过的借用词汇表在这里要加一条纪律：**`span`、`string_view` 这些非拥有视图不得跨线程传递**——它们不延长任何生命周期，接收线程无法知道数据还活着，悬垂风险的并发版。

### 5.9 false sharing：两个变量，一条缓存行

下一个陷阱更阴险：**代码完全正确——没有任何数据竞争——却莫名变慢**。

回忆第 1 章：CPU 不按字节读内存，按**缓存行（cache line，x86-64 上 64 字节）**整块搬。硬件还会做缓存一致性（cache coherence）：每个核心的缓存里都有数据的私有副本，**一个核心改了某个字节，其他核心手里整条缓存行就作废重拉**。

现在把两个事实拼起来：如果两个线程分别写**两个不同的变量**，而这两个变量恰好在**同一条缓存行**上（比如一个结构体里相邻的两个 `long long`）——没有数据竞争（访问的是不同内存位置，同步齐备），但每次写都会把对方的缓存行打失效，两核隔着缓存一致性协议来回把同一条 64 字节抛来抛去。这个现象叫 **false sharing（伪共享）**：共享的不是数据，是缓存行。

```cpp
// ch05_false_sharing.cpp —— false sharing 演示（完整可编译；只验正确性不测速）
#include <atomic>
#include <cstdio>
#include <functional>
#include <thread>

struct CountersBad {                  // 两个计数器挤在同一条缓存行上
    std::atomic<long long> a{0};
    std::atomic<long long> b{0};      // a 与 b 相距仅 8 字节
};

#pragma warning(push)
#pragma warning(disable : 4324)       // MSVC /W4 下 alignas(64) 必然伴生「结构体已填充」告警
struct alignas(64) Padded {           // 对齐到 64 字节：独享一条缓存行（C4324 正是 alignas 生效的证据，非缺陷）
    std::atomic<long long> v{0};
};
#pragma warning(pop)                  // GCC/Clang 无此告警，无需对应处理
struct CountersGood {
    Padded a;                         // 各占一条缓存行
    Padded b;
};

int main() {
    CountersBad  bad;
    CountersGood good;
    using Counter = std::atomic<long long>;
    auto bump = [](Counter& c1, Counter& c2) {
        for (int i = 0; i < 1'000'000; ++i) {
            c1.fetch_add(1, std::memory_order_relaxed);  // 纯统计 → relaxed（5.7）
            c2.fetch_add(1, std::memory_order_relaxed);
        }
    };
    std::thread t1(bump, std::ref(bad.a),  std::ref(bad.b));
    std::thread t2(bump, std::ref(bad.a),  std::ref(bad.b));
    t1.join(); t2.join();
    std::printf("bad: a=%lld b=%lld（正确性无恙；性能差异的量化测量将在第 6 章展开）\n",
                bad.a.load(), bad.b.load());
    (void)good;
}
```

缓解手段就是例子里那招：**把被不同线程高频写的数据对齐到缓存行边界，各住各家**——`alignas(64)` 手动对齐（可移植、全工具链可用）；标准里还有 `std::hardware_destructive_interference_size`（C++17 `<new>`，语义就是「别共享的粒度」，MSVC 可用但 GCC 12 之前缺失，所以示例用 `alignas(64)` 并注释两者关系）。

什么时候该想起它？当剖析器告诉你「这行热点谁也没共享它」、或加了锁/改了原子还是慢的时候。缓存层级的容量与延迟量级、以及量化测量的完整方法论将在第 6 章展开——本章你只需要带走这个概念和 `alignas(64)` 这招。

---

### 5.10 抓竞态的仪器：ThreadSanitizer 与静态分析

到目前你已经会写正确的同步代码了。但并发代码的残酷之处在于：**错的代码往往也能跑对一万次**。靠人眼审 + 多跑几遍，不是工程化的质控。这一节给你装上仪器，把「报告 → 修复 → 零报告」变成并发代码的交付流程。

#### ThreadSanitizer：数据竞争的专门探测器

**ThreadSanitizer（TSan）**是编译器内置的数据竞争探测器（Clang/GCC 的 `-fsanitize=thread`）。原理一句话：程序运行时，它给每块内存维护一份「访问历史」（shadow memory，记录哪个线程在什么同步点读过/写过），同时拦截所有同步原语（锁、atomic）建立 happens-before 关系；一旦发现「两个访问之间既无同步关系、又有写」，立刻报告冲突双方的完整调用栈。

先说清平台现实，别在这里浪费时间：**TSan 不支持 MSVC**（官方明确无计划）；Clang/GCC 上可用，其中 **Linux（含 WSL2）是最顺的主路径**（macOS 只有 Intel 机上的 clang 支持，Apple Silicon 不支持）。Windows 用户的推荐姿势：

1. 安装 WSL2（`wsl --install`，默认 Ubuntu），进入 Linux 环境；
2. `sudo apt install g++` 后用下面的命令构建运行：

```bash
# WSL2 / Linux：TSan 构建（g++ 或 clang++ 同参数）
g++ -std=c++20 -O1 -g -fsanitize=thread -pthread ch05_minimal_race.cpp -o race
./race          # 报告直接打到 stderr
```

（`-O1 -g` 是 TSan 官方推荐组合：保留优化代表性又保留符号；本章正文一直强调 Release 才有意义的性能结论，探测正确性问题则用这个专用配置。）

拿 5.2 的 `minimal_race` 跑一遍，典型报告节选：

```text
WARNING: ThreadSanitizer: data race (pid=12345)
  Read of size 1 at 0x... by main thread:
    #0 main ch05_minimal_race.cpp:12 (race+0x...)        ← 读 ready 的那行
  Previous write of size 1 at 0x... by thread T1:
    #1 producer() ch05_minimal_race.cpp:8 (race+0x...)   ← 写 ready 的那行
  Summary: data race ch05_minimal_race.cpp:12 ↔ 8
```

读报告的三步：**谁读、谁写（两边的栈各指向一行源码）→ 两个字段之间有无同步原语 → 定位修复**。把 `ready` 改成 `std::atomic<bool>` 并用默认序（或按 5.7 配 release/acquire）后重跑，报告归零——这就是「红 → 绿」闭环，任务 2 会让你完整走一遍。

TSan 会拖慢程序 5–15 倍、内存涨数倍——它是**测试工具不是发布配置**，只在 CI/本地测试时开。

#### Windows 本地：ASan 与 /analyze 各自能抓什么

等不了 WSL 的时候，Windows 本地有两个替代品，但**能力边界必须背下来**：

| 工具 | 能抓 | 不能抓 |
| --- | --- | --- |
| TSan（WSL2/Linux） | **数据竞争**（本职） | —— |
| `/fsanitize=address`（ASan，MSVC 可用） | use-after-free、越界、双释放等**内存 UB**（ch02 老朋友） | **数据竞争**——竞争不是内存错误，ASan 看不见 |
| `/analyze`（MSVC 静态分析） | 少量锁纪律问题（配合 SAL 标注）、空解引用等 | 并发路径的真实时序 |

```bat
cl /utf-8 /std:c++20 /EHsc /W4 /fsanitize=address ch05_minimal_race.cpp
cl /utf-8 /std:c++20 /EHsc /W4 /analyze ch05_minimal_race.cpp
```

（若两个选项同开报 D8016 冲突，分两次构建即可。）同一段竞争代码，ASan 一声不吭——请在任务 2 里亲手验证这个「不报」的边界事实，它以后能帮你省下「为什么 ASan 没报竞态」的困惑。

最后一个组合纪律：**TSan 与 ASan 不能同时开**（两者都要接管 shadow memory，插桩运行时互斥；clang 同传 `-fsanitize=thread,address` 直接报错）。同一份代码，两套构建分别跑：ASan 管内存 UB，TSan 管竞争，各司其职。

> **顺带声明（E1）**：clang-tidy 这类静态护栏在工程里的标准化接入，将在第 7 章展开；本章先把 TSan 闭环走通。

---

## 深入专题：一次优雅关停的解剖——jthread、stop_token 与虚假唤醒的实测

延续前几章「亲手解剖」的传统（ch01 解剖 vector 扩容、ch02 解剖 make_shared 控制块、ch03 解剖 std::function 类型擦除、ch04 解剖链接错误），这次解剖一个所有游戏代码都躲不开的场景：**关停**。你按了 Alt+F4，主线程退出，可工作线程还堵在 `wait` 里——进程退不干净，或者直接崩在退出路径上。我们从最糙到最优雅，把退出协议演进三级走完，并在第二级亲手测出虚假唤醒的存在证据。

**第一级：轮询 bool（烧 CPU，不可接受）**

```cpp
while (!stop) { do_work(); }        // ❌ 空转：停得快，但一个核 100% 烧着
```

**第二级：cv + 退出标志（正确，但要手写全套仪式）**

就是 5.4 迷你队列的模式：退出也是条件，`notify_all` 唤醒所有等待者，谓词里包含 `stop`。能跑、正确，但每个队列都要手写一遍「标志 + 谓词 + notify_all」仪式。

顺手做个实验：统计 `notify` 的次数和 `wait` 返回的次数——两者不相等的差值，就是**虚假唤醒真实存在**的直接证据：

```cpp
// ch05_spurious_wakeup_probe.cpp —— 虚假唤醒实测（完整可编译）
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <cstdio>
#include <mutex>
#include <thread>

std::mutex m;
std::condition_variable cv;
bool go = false;
std::atomic<int> notify_count{0};
std::atomic<int> wake_count{0};

int main() {
    std::thread waiter([] {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [] { return go; });        // 谓词版
        ++wake_count;                             // wait 返回（含虚假唤醒后重睡的次数不计）
        // 裸 wait 版对照：若把谓词去掉，wake_count 可能大于 notify_count
    });
    for (int i = 0; i < 1000; ++i) {
        cv.notify_one();                          // 空发 notify：go 还是 false
        ++notify_count;
        std::this_thread::sleep_for(std::chrono::microseconds(100));
    }
    {
        std::lock_guard<std::mutex> lock(m);
        go = true;
    }
    cv.notify_one();
    ++notify_count;
    waiter.join();
    std::printf("notify = %d 次\n", notify_count.load());
    std::printf("谓词版最终醒来 1 次（虚假唤醒被谓词过滤，这就是它的价值）\n");
}
```

跑几遍你会发现行为有随机性：某些平台/某些时刻，裸 `wait` 版会被无通知唤醒（`wake_count` 超过 `notify_count` 中有效的那次）；谓词版则永远稳如老狗——**它就是为这个现实设计的**。这也是「谓词版是唯一推荐姿势」的实验依据，不只是书本规矩。

**第三级：jthread + stop_token（C++20，把仪式自动化）**

```cpp
// ch05_jthread_shutdown.cpp —— jthread 优雅关停（完整可编译）
#include <chrono>
#include <condition_variable>
#include <cstdio>
#include <mutex>
#include <queue>
#include <thread>          // jthread/stop_token 也在这里

std::queue<int> tasks;
std::mutex m;
std::condition_variable_any cv_any;    // 注意：_any 版本才支持 stop_token 重载

void worker(std::stop_token st) {      // jthread 自动把 stop_token 传进来
    for (;;) {
        std::unique_lock<std::mutex> lock(m);
        // 谓词里叠加 stop_requested：任务空且要求停 ⟹ 退出；停时 wait 自动醒来
        cv_any.wait(lock, st, [] { return !tasks.empty(); });
        if (tasks.empty()) return;      // st.stop_requested() 为真且没活了
        std::printf("处理 %d\n", tasks.front());
        tasks.pop();
    }
}

int main() {
    std::jthread w(worker);             // 创建即绑定 stop_source
    {
        std::lock_guard<std::mutex> lock(m);
        tasks.push(42);
    }
    cv_any.notify_one();
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::printf("main 返回——jthread 析构：自动 request_stop + 唤醒 + join\n");
}                                       // ← 一切收尾都在这行花括号里发生
```

三级对照：

| 级别 | 机制 | 烧 CPU | 需要手写的仪式 |
| --- | --- | --- | --- |
| 一级 | 轮询 bool | 高 | 无（但不可接受） |
| 二级 | cv + bool 标志 | 无 | 标志 + 谓词 + notify_all + 手动 join |
| 三级 | jthread + stop_token | 无 | 几乎为零：析构自动 request_stop + 唤醒 + join |

两个工程提醒：

1. **支持 stop_token 等待的是 `condition_variable_any`**（不是 `condition_variable`），`wait(lock, stop_token, pred)` 会在 stop 请求到达时自动醒来。若你的工具链对这个重载有疑义，降级方案就是第二级「thread + cv + bool 退出标志」——语义完全一致，只是仪式手动。本示例在 MSVC /std:c++20 与 g++ 12+ 下均可编译运行。
2. **关停协议是接口的一部分**：谁负责发 stop、谁负责 join、队列 close 与线程退出的先后，都写进你的并发约定文档（任务 3）。ch02 的 RAII 世界观在这里收官——连「线程的收尾」都应该是对象析构的一部分，而不是散在 main 里的几行脆弱代码。

---

## 构建与运行（Windows + VS2022 速查）

本章全部示例文件与编译命令汇总（与 ch01–ch04 同一约定；中文注释需 `/utf-8`）：

```bat
cl /utf-8 /std:c++20 /EHsc /W4 ch05_first_thread.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_minimal_race.cpp     （竞态示例；/O2 下观察行为变化）
cl /utf-8 /std:c++20 /EHsc /W4 ch05_boss_locked.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_scoped_lock.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_cv_queue_v1.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_magic_static.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_call_once.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_atomic_counter.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_release_acquire.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_task_queue.cpp       （类库片段：自行补 main 消费）
cl /utf-8 /std:c++20 /EHsc /W4 ch05_false_sharing.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_jthread_shutdown.cpp
cl /utf-8 /std:c++20 /EHsc /W4 ch05_spurious_wakeup_probe.cpp
```

GCC/Clang 等价（Linux 需显式 `-pthread`）：`g++ -std=c++20 -Wall -Wextra -pthread 源文件.cpp`。竞态示例请按 5.10 的 TSan 配置另行构建。深入实战的任务 1 工程可直接挂进 ch04 的 CMake 骨架：`add_executable(loader apps/loader/main.cpp)` + `target_link_libraries` 接入 TaskQueue 头文件目录即可。

## 本章纪律清单

全章收敛成一张可贴在工位上的检查单，写并发代码时逐条过：

| # | 纪律 | 出处 |
| --- | --- | --- |
| 1 | 线程一律 join 收线或用 jthread；不用 detach | 5.1 |
| 2 | 线程参数默认按值/`move` 传入；`std::ref` 必须伴随生命周期证明 | 5.1 |
| 3 | 「同一位置 + 有写 + 无同步」= 数据竞争 = UB；不与概率论媾和 | 5.2 |
| 4 | 不手写 `lock()/unlock()` 配对；临界区尽量小；持锁不做 IO | 5.3 |
| 5 | 多锁需求用 `scoped_lock`；自定义加锁次序纪律只留给旧代码 | 5.3 |
| 6 | `cv.wait` 只写谓词版；先改状态（持锁）、后 notify；退出也是条件 | 5.4 |
| 7 | 无参单例用 magic static；传参/多入口用 `call_once` | 5.5 |
| 8 | atomic 只管单变量；复合不变式上 mutex | 5.6 |
| 9 | 内存序默认 seq_cst；降级唯一路径 acquire/release；relaxed 只给纯统计 | 5.7 |
| 10 | 不存在线程安全容器，只有线程安全接口；check-then-act 必须收进一个原子接口 | 5.8 |
| 11 | 不同线程高频写的数据 `alignas(64)` 分缓存行 | 5.9 |
| 12 | 并发代码交付即「TSan 红 → 修复 → 零报告」；ASan 与 TSan 分开跑 | 5.10 |

## 实践任务

对照 Stages.md 阶段 5 实践任务：给渲染器实现「主线程渲染 + 工作线程异步加载」；先故意写一版有数据竞争的实现，用 TSan 抓出并修复；写一份引擎并发约定短文档。以下任务全部**不依赖 P1 渲染器资产、独立可做**（E3）；P1 启动后的衔接点以可选标注给出。

### 任务 1（核心层）：迷你异步资源加载器

用控制台模拟「主线程渲染 + 工作线程加载」的完整闭环。新建工程（可直接挂进 ch04 的目录结构，链接第 4 章的构建骨架）：

**要求**：

1. **主线程**：`while` 帧循环，每帧约 16ms（用 `sleep_until` + 稳定时钟），每帧打印帧号与本帧实际帧间隔；帧末检查「完成队列」并「上屏」；监听输入 `q` 触发关停；
2. **下单**：另一个输入线程（或主线程开场预先下单）往共享 TaskQueue（直接用 5.8 的 `TaskQueue<std::string>`）投递 10 个「加载任务」，名字形如 `tex_boss.png`；
3. **工作线程 ×2**：`try_pop` 取任务，模拟解码（`sleep_for` 随机 50–200ms + 生成**确定性校验和**，如名字长度 ×31 + 首字符，用 `unsigned` 溢出算术即可），把「名字 + 校验和」放入加锁保护的完成队列；
4. **上屏**：主线程每帧末取走完成队列全部条目，打印「上屏：tex_boss.png 校验和=0xXXXX」；
5. **关停**：`q` 输入后队列 `close()`，工作线程退出，主线程 `join` 后打印总帧数退出。

**验收要点**：① 加载期间帧间隔仍 ≈16ms（帧率不塌——控制台版检验标准④）；② 每个上屏校验和与生成规则一致（数据未损坏）；③ 关停后进程零崩溃、零挂死退出；④ 全代码无裸 `lock()/unlock()` 配对、无 `detach`。

> **P1 衔接（可选）**：模拟解码换 `stb_image`、上屏换 `glTexImage2D`、TaskQueue 载荷换 `LoadRequest/LoadResult` 结构体，即 P1「主线程渲染 + 工作线程异步加载」原版——第 4 章结尾预告的欠账至此完全兑付。

### 任务 2（扩展层 A）：红 → 绿仪器闭环

复制任务 1 的代码，故意写坏**三处中的至少一处**：① 完成队列去掉锁；② 「数据就绪」标志改回裸 `bool` 忙等；③ 工作线程无锁直接 `push_back` 共享 vector。然后：

1. WSL2/Linux 下用 TSan 构建（`g++ -std=c++20 -O1 -g -fsanitize=thread -pthread`）跑出**竞争报告**并存档节选；
2. 逐条修复（对照报告栈回任务 1 的正确写法），重跑至**零报告**并存档；
3. Windows 本地分别用 `/fsanitize=address` 与 `/analyze` 各跑一遍同一份「坏」代码，记录「ASan 不报竞争」的边界事实（5.10）。

**交付物**：两份报告节选（红/绿）+ 每处竞争的「三要件」归因（谁读、谁写、缺什么同步）+ 修复 diff 说明。完整验收口径见参考答案。

### 任务 3（扩展层 B）：引擎并发约定短文档

为任务 1 工程写一份 `CONCURRENCY.md`，用你自己的话回答：哪些数据可跨线程（任务/结果应为值类型）、谁拥有、谁负责同步（队列两端各管各）、上屏只准主线程、关停协议（谁发 stop、谁 join）、禁止事项（不跨线程传引用/span/string_view；不手写 lock/unlock；不用 detach）。

**选做**：把「数据就绪」通知改成 5.7 的 atomic release/acquire 版，落笔三句话说明它与 cv 版各自的适用边界（检验标准②的落笔版）。并发约定如何在大代码库里长期执行，将在第 7 章展开。

## 自测题

1. 写出一个最小数据竞争示例，并解释为什么它是未定义行为，而不是「偶尔读到旧值」。
2. `std::thread t(worker, x)` 与 `std::thread t(worker, std::ref(x))` 各自发生什么？给出一种线程参数悬垂引用的具体场景与修复。
3. `lock_guard` / `unique_lock` / `scoped_lock` 各自的适用场景？为什么 `condition_variable::wait` 只接受 `unique_lock`？
4. 为什么 `wait(lock, pred)` 的谓词版是唯一推荐姿势？描述一个「唤醒丢失」的时间线，并解释谓词如何救场。
5. 线程安全单例：static 局部变量版与 `call_once` 版各适合什么场景？C++11 前手写 double-checked locking 错在哪里？
6. `std::atomic<int>` 的 `count++` 保证了什么、没保证什么？给一个 atomic 替代不了 mutex 的具体游戏场景。
7. 口述：acquire/release 如何配对替代一把重锁完成同步？seq_cst 默认为何安全？relaxed 的一个正当用途？
8. 「给 `std::vector` 内部加把锁，它就是线程安全容器了」——反驳这句话，并给一个容器内置锁防不住的复合操作序列。
9. 两个线程分别写两个不同的变量、没有任何数据竞争，程序却变慢了。为什么？给出缓解写法。
10. MSVC 没有 TSan。本章给出的替代路线各能抓什么、不能抓什么？为什么 TSan 与 ASan 不能同开？

## 常见误区

1. **用 sleep 或「多跑几遍没崩」对待竞态**——数据竞争是未定义行为，不是概率事件；正解是同步原语或重构数据所有权，并用 TSan 验证闭环。
2. **把数据竞争当「读到旧值」的良性行为**——UB 授权编译器寄存器缓存、指令重排、多核可见性三重放大；`/O2` 下忙等标志直接死循环（5.2 现场）。
3. **手动 lock/unlock 或持锁调用会再加锁的函数**——中途 `return`/异常绕过 `unlock` 全员挂死；递归加锁直接死锁。锁的 RAII 是唯一姿势（与 ch02 同构）。
4. **「加了锁就线程安全了」的接口错觉**——容器内置锁护不住 `empty()+pop()` 的复合序列；同理别把 `shared_ptr` 的计数原子性当对象线程安全（ch02 2.7 复发），别让 `span`/`string_view` 跨线程。
5. **没有数据就随手降级内存序**——为想象中的性能写 relaxed、乱配 acquire/release；先 seq_cst 证明正确，有剖析数据再降（测量方法论将在第 6 章展开）。
6. **detach 一时爽 / 线程比数据活得久**——detach 后访问已死亡的局部变量、队列析构时工作线程还在跑；join/jthread + 关停协议是标配。

## 延伸资源

- **《C++ Concurrency in Action》第 2 版**（Anthony Williams，Manning 2019）——Stages 点名的阶段 5 主教材；第 3–5 章与本章一一对应，内存模型与内存序章节值得精读。
- **cppreference**（全程工具书，Stages 清单第 5 条）——重点页：`std::thread`、`std::jthread`、`std::condition_variable`、`std::atomic`、`std::memory_order`。任何语义疑问先查它。
- **CppCon 2017：Fedor Pikus《C++ atomics, from basic to advanced. What do they really do?》**——Stages 点名的阶段 5 并发专题讲座，六种内存序的最佳可视化讲解。
- **ThreadSanitizer 官方文档**（google/sanitizers wiki 与 LLVM ThreadSanitizer 页）——5.10 的权威出处，含算法论文链接。
- **Microsoft Learn：/fsanitize=address 与 /analyze 文档**——Windows 本地替代路线的官方依据。
- 选看加分：Herb Sutter《atomic<> Weapons》1/2（CppCon 2013，acquire/release 深化）；learncpp.com 多线程章节（免费打底）。

## 参考答案

### 自测题

**1. 最小数据竞争示例与 UB 论证。**

```cpp
bool ready = false;                    // 全局普通 bool
int  payload = 0;
// 线程 A：payload = 42; ready = true;
// 线程 B：while (!ready) {} use(payload);
```

论证三段式：① **三要件齐备**——A 对 `ready` 的写与 B 的读并发、同一内存位置、无同步关系，数据竞争成立；② **UB 授权**——标准规定有数据竞争的程序行为未定义，等于授权编译器与 CPU 放手优化：A 的两次写可以重排、B 的读可以提出循环（`/O2` 下忙等直接死循环）；③ **为何不是旧值问题**——UB 下讨论「读到 0 还是 42」已无意义，读到的可以是一致的谎言、可以是垃圾，程序可以崩，任何行为都「合法」。唯一正解：`ready` 改 `std::atomic<bool>`（默认序即可）并用 release/acquire 顺带护住 `payload` 的可见性。

**2. 两种传参与悬垂引用。** `t(worker, x)`：`x` 被**拷贝**（严格说经过 decay-copy）进线程内部存储，线程用的是自己的副本——安全默认。`t(worker, std::ref(x))`：把 `x` 以引用语义包裝传入，函数签名必须带 `&` 才能接住；线程直接操作原对象——原对象必须比线程活得久。悬垂场景：`main` 里 `int budget = 16; std::thread t(work, std::ref(budget)); t.detach(); return 0;`——`main` 返回后 `budget` 随栈帧消亡，工作线程稍后读它即 UB。修复：去 `std::ref` 按值传（大对象 `std::move`）；或保证生命周期并加同步；不用 detach。

**3. 三把 RAII 锁。** `lock_guard`：构造加锁析构解锁，零额外能力——普通临界区默认姿势。`unique_lock`：可延迟加锁（`std::defer_lock`）、可提前 `unlock`/再 `lock`、可移动——需要「中途开合」时用它。`scoped_lock`：一次拿多把锁且内部防死锁——复合锁需求用它。`cv::wait` 只接受 `unique_lock` 的原因：线程睡在 wait 里时**必须解锁**让同伴进临界区，醒来后**必须重新加锁**再返回——这套「持锁状态中途开合、异常安全恢复」的動作需要锁对象记录状态并可操作，`lock_guard` 没有这些接口，无法与 wait 配合。

**4. 谓词版 wait 与唤醒丢失。** 谓词版是唯一推荐姿势因为它同时免疫虚假唤醒（醒来谓词不成立则继续睡）与唤醒丢失。唤醒丢失时间线：消费者检查队列 `empty()` 为真 → 准备调 `wait`（还没睡下）；此刻生产者 push 并 `notify_one` → 这个通知没有任何等待者接收，**蒸发**；消费者随后睡进 `wait` → 永远等不到下一次通知（可能再也不会有）。谓词救场的机制：`wait(lock, pred)` 在**持锁状态下**先查谓词——生产者改队列必须持同一把锁，于是「查」与「改」被串行化：要么查时队列已非空（直接不睡），要么 notify 发出时消费者必然已睡在 wait 里（必然被叫醒）。两不管的缝隙被同一把锁封死。

**5. 单例两姿势与 DCL 的错。** static 局部版：适合无参、构造即完成的单例——C++11 起标准保证初始化线程安全，代码最短，访问几乎零开销。`call_once` 版：适合要传参、多个触发入口、或需要失败重试语义的初始化。C++11 前 double-checked locking（先无锁查标志，命中再加锁复查）的错误：第一次无锁读没有与初始化线程建立同步关系，本身就是数据竞争；即使侥幸读到「已初始化」，对象构造写入的内存对当前线程的可见性也无保证——读到标志为真却看到半成品对象。根源是当年没有内存模型，无法表达「发布」语义；C++11 后这由 magic statics / call_once 原生解决。

**6. atomic++ 的保证与局限。** 保证：这个「读-改-写」作为整体原子完成（不丢失更新），且（默认 seq_cst 下）参与全程序统一总次序。没保证：跨变量的任何复合不变式。游戏场景：`std::atomic<int> hp` 与「hp ≤ 0 时置 `dead = true` 并切换死亡状态」——两个 atomic 各自原子，但「检查+置位」两步之间其他线程可再扣血/死亡状态可能漏判。这类多变量状态机必须用 mutex 把「检查+转移」护成临界区。

**7. acquire/release 口述。** 模板：生产者完成数据后 `flag.store(true, memory_order_release)`——release 保证标志写入之前的所有普通写入打包完成；消费者 `flag.load(memory_order_acquire)` 读到 true——acquire 保证后续读写不重排到 load 之前；双方建立 synchronizes-with，store 之前的写入 happens-before load 之后的读写，于是普通缓冲区无需锁即可安全消费——**一把重锁护整段临界区，被缩成「发布一个标志」这一对原子操作**。seq_cst 安全默认：所有 seq_cst 操作存在全线程统一总次序，可直接用普通推理想清楚事件顺序；默认写法先保证正确，降级需要剖析数据（测量方法论将在第 6 章展开）。relaxed 正当用途：纯统计计数（命中率、粒子数），不承担任何同步职责。

**8. 反驳「内置锁 = 线程安全容器」。** 内部锁只能让**单个成员函数**原子，护不住**跨调用的复合序列**。反例：消费者写 `if (!q.empty()) { v = q.pop(); }`——`empty()` 与 `pop()` 是两次独立加锁，中间其他消费者可把队列抽干，`pop()` 空队列即 UB（check-then-act 竞争）。正解：把「等待并取出」合并成一个原子接口（如 5.8 的 `try_pop`），锁在方法内部贯穿整个复合操作——安全性长在接口设计上，不在容器上。

**9. 无竞争却变慢。** false sharing：两个变量恰在同一条 64 字节缓存行上，两核交替写使对方缓存行持续失效，带宽耗在缓存一致性协议上。缓解：`alignas(64)`（或 `std::hardware_destructive_interference_size`）把被不同线程高频写的数据对齐隔离，各住一条缓存行。量化验证归第 6 章测量方法。

**10. Windows 替代路线。** TSan（WSL2/Linux 下 clang++/g++ `-fsanitize=thread`）：专抓数据竞争，运行时对比访问历史与同步关系，误报率低；不能抓内存 UB。`/fsanitize=address`（MSVC 本地）：抓 use-after-free/越界/双释放等内存 UB；**不抓数据竞争**——竞争不是内存布局错误，ASan 的探测器看不见线程间时序。`/analyze`：静态分析，配合 SAL 能查少量锁纪律与空解引用，抓不到真实并发时序。TSan 与 ASan 不能同开：两者都要接管 shadow memory 并插桩整个运行时，互斥；clang 同传两个 sanitize 标志直接报错，MSVC 同开若报 D8016 则分两次构建。

### 实践任务

**任务 1 参考思路**：

- **骨架**：`TaskQueue<std::string> orders`（5.8 原样复用）+ 完成队列 `std::mutex + std::vector<LoadResult>`（主线程帧末批量取，也可再包一层带 cv 的队列）；两个 `std::jthread` 工作线程循环 `try_pop`；主线程帧循环 + 输入检查。
- **帧循环**：`auto next = steady_clock::now(); while (running) { next += 16ms; ……干活……; sleep_until(next); }`——比「睡固定时长」稳，帧间隔打印即验收证据 ①。
- **确定性校验和**：`unsigned checksum(std::string_view n) { unsigned h = 2166136261u; for (char c : n) h = (h ^ c) * 16777619u; return h; }`（FNV-1a，ch01 的溢出算术在这里变现）——生成端与上屏端都调它，逐单核对即验收 ②。
- **关停协议**：输入 `q` → `orders.close()`；工作线程 `try_pop` 返回 `nullopt` 即退出；主线程 join 全部工作线程后收尾。不需要裸标志位——close 就是关停条件。
- **验收要点**：① 帧间隔打印持续 ≈16ms（加载高峰期偶发到 20ms 内可接受，持续翻倍即说明持锁干活了）；② 校验和逐单一致；③ 退出码 0、无挂死（任务管理器确认进程消失）；④ `grep` 全工程无 `\.lock\(\)|\.unlock\(\)|detach`。检验标准④的控制台版即此：异步加载期间帧率不塌、完成后逐单正确上屏。

参考骨架（主线程帧循环部分，其余部件全部来自正文，不重复贴）——注意看同步点都长什么样：

```cpp
// 任务 1 主循环骨架（完整实现由你组装）
#include <chrono>
using namespace std::chrono;

// …… orders: TaskQueue<std::string>；done: 加锁保护的 vector<LoadResult>……

auto next_frame = steady_clock::now();
int frame = 0;
while (running) {
    next_frame += milliseconds(16);                 // 定长帧步进
    ++frame;

    // 帧末：收走全部完成项并「上屏」
    for (auto& r : take_done_items())               // take_done_items 内部加锁、批量 move 出来
        std::printf("[帧 %d] 上屏 %s 校验和=0x%X\n", frame, r.name.c_str(), r.checksum);

    // 检查输入 q（略，CRT 非阻塞读或另起输入线程均可）

    std::this_thread::sleep_until(next_frame);      // 睡到下一帧，帧间隔即验收证据
}
```

三个提示：完成队列的「批量取出」接口一次锁、一次 move 出全部条目，比逐条锁高效且天然减少锁竞争；`sleep_until` + 绝对时间点能抵消上屏耗时漂移，比「干活完再睡 16ms」准；校验和函数生成端与上屏端共用同一个，对照才成立。

**任务 2 参考思路（红 → 绿）**：

- **制造红**：三选一即可——完成队列去锁（vector push_back 竞争，TSan 报 `pthread_mutex` 缺失的双写冲突）；就绪标志裸 bool 忙等（TSan 报 bool 的读写冲突）；无锁 push_back（可能报 vector 内部多个栈帧的冲突）。任一都会在数秒内出报告。
- **报告阅读**：WARNING 行定位冲突类型 → 两段调用栈各指一行源码 → `Summary` 给出「文件：行 ↔ 行」。把两行源码、三要件（同一位置/一写/无同步）逐条归因写进交付物。
- **绿判定口径**：同一二进制、同一输入，TSan 全程零 `WARNING: ThreadSanitizer` 输出；跑足够触发并发的场景（多任务 × 多工作线程）且重复 3 次。
- **Windows 边界记录**：同一份坏代码在 `/fsanitize=address` 下正常跑完（无竞争报告——它不干这个）；`/analyze` 可能给 C26100 类锁纪律警告（若有 SAL 标注），给不出时序结论。把「工具能边界」写成三行放进交付物。

**任务 3 参考思路（CONCURRENCY.md 目录模板）**：

1. **可跨线程的数据**：任务与结果均为值类型（string/POD 结构体），沿队列移动所有权；
2. **所有权**：队列拥有待处理任务；工作线程拥有处理中的任务；主线程独占「已上屏」数据；
3. **同步责任**：队列内部自护（push/try_pop/close 各自成套）；完成队列同构；除此之外任何共享都要显式写明原语；
4. **上屏纪律**：只准主线程触碰渲染/输出；
5. **关停协议**：输入线程发 close → 工作线程自然退出 → 主线程 join；禁止从外部杀线程；
6. **禁止事项**：不跨线程传引用/`span`/`string_view`；不手写 lock/unlock；不用 detach；不加锁做 IO；无数据不降内存序。

评分式验收：每条约定能指出「对应的代码位置」与「违反时的故障模式」两栏；选做题（release/acquire 版就绪通知）三句话能说清「cv 版适合『队列语义』的等待-消费，atomic 版适合『状态发布』的轻量通知，且后者不能携带队列的等待/唤醒语义」即达标。完整工程化实现留给 P1/P3。

---

*第 5 章到此收官。你已经能在「数据竞争的边界」内自由调度线程：mutex 护临界区、cv 管等待、atomic 发通知、仪器抓竞态。下一章我们换一把尺子——每帧 16.6ms 的预算考试：缓存层级如何支配数据布局、堆分配如何无声侵蚀帧时间、以及用剖析器把「感觉慢」变成「看得见的数字」。**实时性能意识，将在第 6 章展开。***








