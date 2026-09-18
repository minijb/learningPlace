# c++ 基础+深度理解

本文件是根纲领 [Topic.md](../Topic.md) 第 1 节「c++ 基础+深度理解」的领域纲领，限定 `01-cpp/` 文件夹内一切知识的范围：现代 C++（17/20）语言功底、资源管理与所有权、标准库与泛型、工具链工程化、并发与内存模型、实时性能意识、大型工程阅读与重构素养。本领域的主要学习材料为 `tutorial/` 下分章生成的中文教程（主教材，定位与写作规则见 [Context.md](./Context.md)）；各阶段的详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 资源 / 误区）以根 [Stages.md](../Stages.md) 第 1 节为权威基线。

边界声明：C++ 语言功底归本领域；图形数学与渲染理论归第 2 节、图形 API 实操归第 3/6 节、引擎架构归第 4 节、GPU 并行归第 5 节；实践项目的完整实现归跨章节项目 P1/P3（见根 [Projects.md](../Projects.md)），教程只承载讲解与练习。

## 阶段 1｜现代 C++ 基石：对象模型、值语义与移动语义

- 建立「对象有类型、有存储、有生命周期」的底座世界观，吃透值语义与拷贝/移动机制，三/五/零法则成为默认取舍。
- 掌握现代语法基本面（auto、范围 for、结构化绑定、enum class 等）与初始化陷阱，const 正确性从第一天养成。
- 搭好开发环境，从第一天起会用调试器单步、看调用栈、定位崩溃。
- 教程：[ch01-object-model-and-move-semantics](./tutorial/ch01-object-model-and-move-semantics.md)（已定稿）｜联动：双线先行——本阶段及阶段 2 可与第 2 节阶段 1 并行起步。

## 阶段 2｜RAII 与所有权：从裸句柄到智能指针

- 把 RAII 内化为唯一资源管理模式，能为任意 C API 裸资源（GL/Vulkan 句柄、文件、锁）现场设计包装。
- 吃透 unique_ptr/shared_ptr/weak_ptr 语义：控制块、循环引用、自定义 deleter、拥有/借用词汇表。
- 以 ASan/CRT 调试堆验证零泄漏为肌肉记忆。
- 教程：[ch02-raii-and-ownership](./tutorial/ch02-raii-and-ownership.md)（已定稿）｜联动：P1「RAII 化」里程碑落点。

## 阶段 3｜标准库与泛型编程：从用模板到写模板

- 容器选型有依据、迭代器失效有戒心，算法库优先于手写循环。
- 跨过「只会用模板」到「能写带约束的模板」：实例化模型、concepts、constexpr、type traits、CRTP 概念级。
- 吃透可调用对象全景（lambda/std::function/std::bind/std::invoke）与词汇类型（optional/variant/any/span）的用法与选型边界。
- 教程：[ch03-standard-library-and-generic-programming](./tutorial/ch03-standard-library-and-generic-programming.md)（已定稿）｜联动：为 P1 沉淀泛型 ResourceManager。

## 阶段 4｜工具链工程化：CMake 构建体系

- 摆脱 IDE 一键工程黑盒：target 属性传递、FetchContent/find_package 接库、CTest 测试。
- 产出可交接工程：新环境克隆后 configure→build→test 三步从零构建。
- 教程：[ch04-cmake-build-system](./tutorial/ch04-cmake-build-system.md)（已定稿）｜联动：P1「CMake 工程化」里程碑落点；第 6 节直接复用此工程。

## 阶段 5｜并发与内存模型：数据竞争的边界

- 理解数据竞争即未定义行为，mutex/condition_variable/atomic 与六种内存序的安全默认和降级路径。
- 目标是「能识别与治理竞态、默认顺序一致、必要时才降级」，不追求精通 lock-free；用 TSan 验证。
- 教程：[ch05-concurrency-and-memory-model](./tutorial/ch05-concurrency-and-memory-model.md)（已定稿）｜联动：P1「主线程渲染 + 工作线程异步加载」实践。

## 阶段 6｜实时性能意识：帧预算、缓存友好与热路径零分配

- 建立「每帧都是一次 16.6ms 预算考试」世界观：会用剖析器定位帧耗时、解读火焰图。
- 理解缓存层级对数据布局的支配（AoS vs SoA），热路径零分配炼成肌肉记忆。
- 教程：[ch06-real-time-performance](./tutorial/ch06-real-time-performance.md)（已定稿）｜联动：第 4 节 ECS 数据导向设计的性能出处。

## 阶段 7｜大型工程阅读与重构：以 UE 源码与 Core Guidelines 为标尺

- 掌握百万行级代码库定点阅读方法论：调用链追踪 + 调试器断点验证，不通读。
- 以 UE 源码与 C++ Core Guidelines 为标尺形成工程审美，clang-tidy 护航小步重构。
- 教程：[ch07-large-codebase-reading-and-refactoring](./tutorial/ch07-large-codebase-reading-and-refactoring.md)（已定稿）｜联动：UE 源码对应根纲领推荐资源第 14 条。
