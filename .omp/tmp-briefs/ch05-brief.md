# 第 5 章设计简报｜并发与内存模型：数据竞争的边界

目标文件：`tutorial/ch05-concurrency-and-memory-model.md`。对齐 Stages.md 第 1 节·阶段 5。

## 章元信息
- 章标题（H1 原文）：`# 第 5 章｜并发与内存模型：数据竞争的边界`
- 阶段学习目标（原文）：理解 C++ 内存模型划定的合法边界——什么构成数据竞争、标准原语如何安全通信；目标定为「能识别与治理竞态、默认顺序一致、必要时才降级」，不追求精通 lock-free。
- 检验标准（原文，学习目标与自测题 1–3 对位）：
  1. 能给出最小数据竞争示例并解释为何是未定义行为；
  2. 能口述 acquire/release 如何替代一把重锁完成配对同步；
  3. 修复前 TSan 报竞争、修复后零报告；
  4. 异步加载期间帧率不塌、完成后纹理正确上屏。

## 连续性契约
- 开篇必须兑现 ch04 结尾承诺（原文指向）：「下一章我们把并发请进来：数据竞争为什么是未定义行为、mutex 与条件变量怎么守住共享的任务队列——而你的第一个实战场景，就是给本章这个工程接上『主线程渲染 + 工作线程异步加载』的异步队列，让 ResourceManager 的加载真正离开主线程。」
- 回收点（正文必须兑现）：
  - ch02 2.7「引用计数原子性 ≠ 对象线程安全」→ 在 5.9 接口级竞争处回收（三层表述：控制块原子、各线程实例独立、同一实例并发读写要锁）。
  - ch02 的 RAII 世界观 → 5.4 锁的 RAII（early-return/异常不解锁）。
  - ch03 3.7 std::function 类型擦除 → 5.5 任务队列里装 std::function。
  - ch02 2.8 lambda 捕获 this 悬垂 / enable_shared_from_this → 5.2 线程函数捕获局部引用与 this 的悬垂坑。
  - ch04 mini_engine 工程（math/resman/demo + CTest）→ 实践任务 1 素材。
- 埋点：5.10 false sharing 埋第 6 章缓存层级（一句指针）；5.11 的 MSVC /analyze 提一句「第 7 章的 clang-tidy 接棒」；开篇可提一句帧预算算术归第 6 章。
- 收尾段钩子：false sharing 已摸到缓存行的边——下一章把整个缓存层级搬上台面（帧预算、火焰图、AoS/SoA、零分配）。

## 分节大纲（11 节）

### 5.1 为什么游戏需要并发：主线程的一帧与「卡一下」的加载
- 渲染主循环是串行事实：输入→update→render，一帧内必须完成。
- 同步加载的代价：主线程里读盘+解码几十 MB 纹理 = 卡顿掉帧（模拟数字：帧时间从 16ms 飙到数百 ms 的示意表）。
- 解法分工图（文字版）：主线程渲染，工作线程干重活（加载/解码/IO），结果回主线程消费。
- 章内地图：5.2 线程本身 → 5.3 什么是数据竞争 → 5.4–5.6 三大标准原语（锁/条件变量/一次性行动）→ 5.7–5.8 atomic 与内存序 → 5.9–5.10 两类高级坑（接口级竞争/false sharing）→ 5.11 工具。
- 一句指针：帧预算的完整算术归第 6 章。

### 5.2 std::thread 与 jthread：线程的生老病死
- thread 构造即启动；join 前不能析构（terminate）；detach 的危险（资源随 thread 对象析构而悬垂，尽量不用）。
- 参数传递：实参被 decay-copy 进线程自有存储——按值安全；传引用必须 std::ref/cref 且对象必须活得比线程久。
- 悬垂引用坑完整示例（关键示例，完整可编译）：lambda 按引用捕获局部变量 + 用 std::thread 或延迟 join → 数据竞争/悬垂；正确版对照。成员函数 + this 的坑（回收 ch02 2.8 词汇）。
- jthread（C++20）：析构自动 request_stop()+join()；stop_token 协作式取消示例（工作线程循环检查 stop）；为什么取消必须「协作」——线程不能被外部安全杀死。
- 完整可编译示例：N 个工作线程各打日志再 join；故意演示一个 detach 反例（注释警示）。

### 5.3 数据竞争：定义与未定义行为的本质
- 定义（cppreference 口径）：两个线程访问同一内存位置，至少一个是写，且两个访问没有 happens-before 同步关系 → 未定义行为。内存位置（memory location）概念：独立标量对象或相邻位域组。
- 为什么不是「偶尔读到旧值」：三个自由被打破前的世界——编译器可把变量缓存在寄存器、可重排互不依赖的指令、非原子的宽类型写可撕裂（tearing，概念级）。
- UB 的真正杀伤：优化器以「无数据竞争」为前提。经典双变量示例（关键示例，完整可编译+两线程）：`payload` + `ready` 都用普通 bool/int，写线程先填 payload 再置 ready，读线程轮询 ready 后读 payload——正常编译看似能跑，开了优化可能读到「ready 已置位但 payload 是旧值」（重排/缓存所致）。输出标注示例输出，注明未定义行为的表现形式不限于列举。
- 计数器丢更新示例：两个线程对普通 int 各自自增 N 万次，结果 < 2N（++ 是 load/add/store 三步，交错丢更新）。
- 「能跑 ≠ 没竞争」：概率性 bug，换编译器/优化等级/机器就翻脸；正解是同步原语或重构数据所有权，并用 TSan 验证（对位 Stages 误区「用 sleep 或多跑几遍没崩对待竞态」）。

### 5.4 mutex 与锁的 RAII：lock_guard、unique_lock、scoped_lock
- mutex 保护的是「约定都拿锁的代码路径」，锁不认识数据——纪律是接口契约的一部分。
- 锁的 RAII（平移 ch02）：手动 lock/unlock 遇 early-return/异常必漏解锁 → lock_guard；为什么这是「锁版 rule of zero」。
- unique_lock：可延迟加锁、可中途 unlock、可移动——condition_variable 的搭档（5.5 用到）。
- scoped_lock（C++17）：一次锁多把，内部用 std::lock 死锁避免算法；两把锁次序相反的经典死锁示例（线程 1 先 A 后 B、线程 2 先 B 后 A）+ 两种修法（全局锁次序 / scoped_lock 一次拿全）。
- 死锁四必要条件概念级 + 工程解法清单（锁次序、一次拿全、缩小临界区、避免嵌套锁内调用未知代码）。
- 锁粒度两难：太粗并发归零，太碎复合不变量被撕裂——先正确后缩小。
- 完整可编译示例：多线程互斥自增计数器，结果精确。

### 5.5 condition_variable：等待、通知与虚假唤醒
- 忙等轮询的问题：烧 CPU 且加剧缓存行竞争（回收 5.3 计数器例）。
- cv 模型：wait(unique_lock) 原子地「解锁+挂起」，被通知后唤醒+重新加锁。
- **虚假唤醒（spurious wakeup）**：OS 底层（futex 语义）允许无通知唤醒——所以条件必须用循环重查；`wait(lock, pred)` 就是谓词循环的语法糖。
- 丢失唤醒：通知先于等待发生 → 若不持锁改条件就永久沉睡；规则：改共享条件必须持（同一把）锁，或依赖谓词版 wait 的双检。
- notify 时持锁还是通知前解锁的权衡（唤醒后立刻撞锁墙 vs 通知被延迟），结论：先用对，再谈微调。
- 完整可编译示例（关键示例，较大）：`BlockingQueue<T>`——mutex + condition_variable + std::deque + stop 标志；push 后 notify_one；pop 用谓词 wait；close() 置 stop + notify_all + 析构前 join 的关闭礼仪。注释标注「队列元素类型是 std::function<void()> 时即任务队列」（回收 ch03 3.7）。

### 5.6 一次性行动：call_once 与 magic statics
- 懒初始化在多线程下天然是竞争（if (!inited) { init(); } 两步）。
- magic statics：C++11 起局部 static 初始化线程安全，编译器自动加同步——单例正确形态 `static T& inst() { static T t; return t; }`；注意这只保护「初始化一次」，不保护后续使用。
- call_once + once_flag：适合更复杂的初始化协议（带参数、跨多个入口、失败重试语义）。
- 两者对比选择表；「线程安全初始化 ≠ 线程安全使用」粗体结论（接 5.9）。

### 5.7 atomic：它真正解决的问题
- atomic<T> 三件事：不可撕裂的原子访问、与其他原子建立同步关系（内存序）、禁止编译器跨它重排/缓存。
- 适用判据（粗体）：单变量、无复合不变量。「balance 与 log 必须一起改」不是 atomic 能表达的——那是锁的辖区。
- is_lock_free() 概念；atomic<bool> 停机标志（5.2 stop_token 的底层亲戚）；fetch_add 无锁统计计数器（每帧命中数）；exchange 设标志并取旧值。
- compare_exchange_weak/strong 概念级一句话（CAS 是 lock-free 的基石——本章刻意不深入，对位「不追求 lock-free」；想深入去章末资源）。
- 完整可编译示例：atomic 计数器 vs 5.3 的普通 int 计数器对照（结果精确）。

### 5.8 六种内存序：为什么默认 seq_cst、何时才降级
- 安全声明先行（粗体）：全部用默认 seq_cst 在绝大多数场景正确且够快；降级是测量驱动的高级操作，不是日常操作。对位「默认顺序一致、必要时才降级」。
- 概览表：seq_cst / acquire / release / acq_rel / relaxed（consume 实践中被实现为 acquire，一句话带过）。
- seq_cst 直觉模型：所有 seq_cst 操作构成一个全局一致的全序，各线程看到的交错都兼容它——「仿佛线程真的在交错执行」的直觉在 seq_cst 下成立。
- acquire/release 配对 = 主要降级路径：release-store 之前的所有写，对成功 acquire-load 同一变量的线程之后的读可见——「发布-订阅」；用它修好 5.3 的 ready/payload 双变量竞争（关键示例，完整可编译：payload 用普通类型、ready 用 atomic，store(release)/load(acquire)）。
- relaxed：只保原子性不建同步——纯计数、统计（用完即弃的数字）；给「读完就打印、不再用于决策」的例子。
- 决策树（列表/文字图）：需要「A 之前的写对 B 之后可见」吗？不需要 → relaxed；需要单变量发布 → release/acquire；涉及多变量组合一致性或不想操心 → seq_cst。
- 警示案例概念级：两个原子变量各自 acquire/release 配对，多个观察者交错时仍可能看到不一致顺序（IRIW 类）——多原子+多观察者的一致全序只有 seq_cst 保证。深读去 C++ Concurrency in Action 第 2 版。
- happens-before 一句话定位：本节的语言学基础，完整理论不在本章展开。

### 5.9 接口级竞争：线程安全容器并不存在
- 「给 vector 加把锁就是线程安全容器？」——单接口锁保护不了接口组合：`if (!q.empty()) x = q.front();` 两步之间容器被改（TOCTOU：check-then-act）。
- 正确路线二选一：把「检查+取用」折叠成一个原子接口（如 BlockingQueue::pop 整体弹出，处理逻辑留在锁外）；或干脆不共享——消息传递/每线程私有 + 集中交换。
- STL 容器线程安全官方口径（cppreference）：同一容器的并发 const 访问 OK；任一线程做写则所有并发访问都必须同步——标准库不提供线程安全容器。
- 回收 ch02 2.7 三层表述（控制块原子 / 每线程各自的 shared_ptr 实例独立 / 同一实例并发读写要外部同步）——它就是「数据级 vs 实例级」的官方范例；shared_ptr 的 atomic 重载存在但默认避用。

### 5.10 false sharing：缓存行上的隐形撞车
- 场景：两个线程各自累加**不同的**变量，逻辑上毫无竞争，性能却塌——两变量挤在同一条 64 字节**缓存行（cache line）**上，缓存一致性协议让两核之间缓存行乒乓。
- 定性：不是 UB、不是数据竞争，是**正确但慢**——同步正确的代码仍可能被缓存惩罚。
- 完整对照实验（关键示例）：N 线程各自累加自己计数器，紧凑 `std::array` 布局 vs `alignas(64)`（或 `std::hardware_destructive_interference_size`，注其可用性以实现为准）隔离，计时倍差（只报数量级，标注以本机实测为准）。
- 一句埋点：缓存行只是缓存层级的原子单位——L1/L2/L3 的容量与延迟量级、预取、AoS/SoA 全在下一章。

### 5.11 ThreadSanitizer 与静态分析：让竞态现形
- 肉眼与 code review 抓不住概率 bug——需要仪器。
- TSan 原理一句话：影子内存记录访问历史，报告「无 happens-before 关系的冲突访问对」。
- 平台现实：clang/gcc `-fsanitize=thread` 是成熟路线；MSVC 有 `/fsanitize:tsan`（VS2022 17.8 起，实验性、x64，以官方文档为准）；Windows 备选 clang-cl 或 WSL。TSan 与 ASan 不能同开；给 sanitizer 单独一个构建配置（回收 ch04 多配置世界观：Debug/Release 之外加 sanitizer 配置，不污染 Release）。
- 读一份 TSan 报告的解剖：两个线程、两次冲突访问、各自栈、对象创建栈——从报告到修复的路径（加同步 or 重构所有权）。
- 静态分析：MSVC `/analyze`（含并发相关 C26xxx 警告族）与 Core Guidelines checker 插件；一句前向指针：跨平台统一的主力是第 7 章的 clang-tidy。
- 修复循环纪律（对位检验标准③）：写竞争版 → TSan 报告 → 修复 → 零报告；「修复后跑一次绿 ≠ 无竞争」仍需在 CI/例行跑。

## 深入专题：解剖一条「主线程渲染 + 工作线程异步加载」管线
把 5.2/5.4/5.5/5.9 组装成完整可编译程序（本章压轴示例，约 200–260 行含代码）：
- `AsyncAssetLoader`：请求队列 `BlockingQueue<LoadRequest>`（5.5 成品复用）+ 结果回收队列（mutex 保护的 `std::vector<LoadResult>`，主线程每帧 drain——**不**用条件变量等结果：主线程不该为工作线程挂起）。
- 游戏语境模拟：主线程循环 N 帧，打印帧号与帧耗时；帧 0/1 提交若干「纹理加载任务」（用忙碌循环或 std::this_thread::sleep 模拟解码耗时，产出像素数据字符串）；工作线程完成后把 `LoadResult{key, data}` 推入结果队列；主线程每帧 drain 后「上屏」（打印）。
- 关闭礼仪：析构里 close 队列 + join（或用 jthread）。
- 代码注释逐处标注同步决策：哪把锁、为什么谓词 wait、哪个 atomic、内存序为什么用默认 seq_cst。
- 示例输出（示意）：帧号连续滚动、加载期间帧时间不塌、结果在后续帧上屏。
- 讨论：为什么不让工作线程直接回调主线程对象（悬垂 this + 接口级竞争，回收 ch02 2.8 与 5.9）——「数据向主线程流动，而不是代码向主线程伸手」粗体结论。

## 实践任务（3 个，全部核心层；E3 声明放开头）
- 任务 1：把 ch04 mini_engine 的 resman 升级为异步加载版（AsyncResourceManager：请求 BlockingQueue + 主线程 drain 结果 + 去重）；demo 主循环模拟渲染帧打印帧号与帧耗时，加载在工作线程完成回主线程「上屏」。素材：ch04 任务 1 的 resource_manager.hpp / memory_loader.hpp + 本章深入专题骨架。验收：①加载期间主循环帧时间不塌（日志证明，对位检验标准④）②同 key 去重仍成立（ch03 任务 1 语义不丢）③关闭礼仪：正常退出无崩溃、stop+join 有日志。可选 P1 衔接：P1·M2 之后的「主线程渲染 + 工作线程异步加载」增量落点。
- 任务 2：竞争解剖与修复（对位检验标准①③）：先故意写竞争版（共享 payload+ready 无同步），正常构建与 TSan 构建各跑一次，贴报告；再用 atomic release/acquire 修复（对位检验标准②的配对同步）贴零报告；再写一版 mutex 对照（复合不变量场景锁更通用）。验收：修复前后报告对照 + 一段「为什么正常构建看起来能跑」的解释。
- 任务 3：写《并发约定》一页文档：线程清单与职责、哪些数据可跨线程、谁负责同步（每把锁的属主）、关闭次序、禁止事项（渲染数据只在主线程碰等）。验收：一个没读过代码的同事能据此判断「新功能该在哪个线程做」。

## 自测题（8 题，第 1–3 题对位检验标准）
①口述：最小数据竞争示例 + 为何是 UB（对位检验标准①）。②口述：acquire/release 如何替代一把重锁完成 payload 发布的配对同步（对位检验标准②，对位 Stages 检验原文）。③报告解剖：给一段 TSan 报告文本节选，问两个访问各来自哪、缺什么同步、给两种修法（对位检验标准③）。④改错：thread 参数传递三坑（按值搬移误解 / std::ref 悬垂 / 成员函数 this）。⑤概念对比：jthread vs thread 析构行为 + 为什么取消必须协作式。⑥场景判断四连：普通 int 双线程读/atomic 计数/两个不同变量同行/BloCkingQueue 两步 pop——哪些是数据竞争、哪些是接口级竞争、哪些是 false sharing、哪些没事。⑦内存序选型三场景：帧统计计数器 / 单 flag 发布 payload / 多变量不变量——选 relaxed/acq-rel/seq_cst 并说理由。⑧对比：magic statics vs call_once + 「线程安全初始化 ≠ 线程安全使用」。

## 常见误区（8 条，症状/病根/纠偏）
①「多跑几遍没崩」当验证（对位 Stages 误区原文）。②detach 撒手不管。③忙等轮询代替 cv / 用了 cv 但裸 wait 不用谓词循环（撞虚假唤醒+丢失唤醒）。④锁粒度两个极端（一把大锁并发归零 / 锁太碎不变量撕裂）。⑤万物 atomic（复合不变量表达不了）。⑥未测量先降内存序（盲目 relaxed 追性能）。⑦「加了锁的 STL 容器=线程安全容器」（TOCTOU）。⑧没有关闭礼仪（不 join/不 stop，退出时线程还在跑）。

## 延伸资源（7–9 条，带确定链接）
- 《C++ Concurrency in Action》第 2 版（Anthony Williams）——本章主教材（Stages 指定）。
- cppreference：C++ 并发页面族（std::thread / mutex / condition_variable / atomic / 内存序），工具书。
- ThreadSanitizer 官方文档（github.com/google/sanitizers）。
- MSVC sanitizer 文档（learn.microsoft.com，检索 /fsanitize）。
- Herb Sutter「atomic<> Weapons」讲座（检索词，内存序深读经典）。
- CppCon Fedor Pikus 并发讲座（检索词，Stages 指定）。
- C++ Core Guidelines CP 并发规则族（isocpp.github.io/CppCoreGuidelines，第 7 章接棒细读）。

## 覆盖清单（写作自查，逐条对应 Stages 阶段 5 知识点）
1. std::thread/jthread、参数传递与悬垂引用坑 → 5.2
2. 数据竞争定义与 UB 本质 → 5.3
3. mutex + lock_guard/scoped_lock（锁的 RAII）→ 5.4
4. condition_variable 与虚假唤醒 → 5.5
5. call_once、static 局部变量初始化 → 5.6
6. atomic 基本操作 → 5.7
7. 六种内存序概览（seq_cst 默认/acq-rel 降级/relaxed 窄用途）→ 5.8
8. 接口级竞争、线程安全容器不存在 → 5.9
9. false sharing 概念 → 5.10
10. TSan/静态分析 → 5.11
