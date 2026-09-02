# 第 2 章｜RAII 与所有权：从裸句柄到智能指针

欢迎来到第 2 章。第 1 章结束时我们说过：你在那里写过的每一个 `delete[]`，都会在这里找到更安全的归宿——现在兑付。上一章你建立了「对象有类型、有存储、有生命周期」的世界观，也亲手体会过裸资源的两难：`new` 出来的东西出了作用域还活着，忘了 `delete` 就泄漏，`delete` 早了就悬垂（1.11 节的 HUD 崩溃）。这一章我们把这个两难连根拔掉：让语言替你管资源。同时偿还第 1 章欠下的两笔账——1.2 节表格里「静态存储期：析构顺序陷阱（第 2 章展开）」，以及「管理堆对象的正确姿势是第 2 章的主题」。

这一章的终点是一条工程铁律：**渲染器里每个 GL 对象恰有一个 RAII 属主，全代码检索不到一个裸 `glDelete` 调用**——这也是你后面第 6 节「及时用 RAII 重构」能力的直接出处。

---

## 学习目标

对齐本领域阶段 2（RAII 与所有权）的学习目标。读完本章，你应该能够：

1. **把 RAII 内化为唯一资源管理模式**：理解「构造即获取、析构即释放」如何把第 1 章的确定性析构变成万能的资源管理机制；它不止管内存——文件、互斥锁、GL/Vulkan 句柄、一切「拿了就要还」的东西，都能包。
2. **为任意 C API 裸资源现场设计 RAII 包装**：给出 `glCreateTextures/glDeleteTextures` 这样的 create/destroy 对，你能立刻写出 `unique_ptr` + 自定义 deleter 或句柄类两种方案，并说清各自的取舍。
3. **吃透三大智能指针**：`unique_ptr`（独占、零开销、默认答案）、`shared_ptr`（共享、控制块、有代价、罕见）、`weak_ptr`（观察、破环）；讲到控制块与 `make_shared` 的内存布局级差异。
4. **避开所有权四大暗坑**：循环引用泄漏；「引用计数原子性 ≠ 对象本身线程安全」；`enable_shared_from_this` 误用开出第二张控制块；lambda 捕获 `this` 的悬垂回调。
5. **建立所有权词汇表**：拥有 / 借用（引用、`span`、`string_view` 借而不拥有）成为你设计接口时的本能——每个参数都说得清「谁拥有、谁借用、借用窗口多长」。
6. **零泄漏成为肌肉记忆**：会用 AddressSanitizer 与 MSVC CRT 调试堆给「中途提前 return」路径开泄漏报告；知道静态/全局对象析构顺序陷阱概念级的成因与工程缓解。

> **环境约定（全章适用）**：Windows + Visual Studio 2022（「使用 C++ 的桌面开发」工作负载），语言标准 `/std:c++20`，警告级别 `/W4`。本章源码含中文注释，命令行编译请加 `/utf-8`（否则 MSVC 默认按本地代码页读源文件，中文注释会被误读成一串莫名其妙的语法错误）：
>
> ```bat
> cl /utf-8 /std:c++20 /EHsc /W4 源文件.cpp
> ```
>
> GCC / Clang 等价命令：`g++ -std=c++20 -Wall -Wextra 源文件.cpp` 或 `clang++ -std=c++20 -Wall -Wextra 源文件.cpp`。本章全部「完整示例」均在上述两条工具链（MSVC 19.4x 工具集 + clang++ 18）下编译零警告、运行并逐行核对过输出；涉及 C++17/20 差异或引擎停留在 C++17 会受影响的场景，正文显式标注。

---

## 分节正文

### 2.1 RAII：把「释放」绑定到作用域——不止内存

RAII，Resource Acquisition Is Initialization，资源获取即初始化。这句话念起来拗口，机制却朴素得惊人：

> **在构造函数里获取资源，在析构函数里释放资源。** 于是资源的生命周期 = 对象的生命周期，而对象的生命周期由第 1 章 1.2 节的规则精确管辖：局部对象随作用域而生、随作用域而死，构造正序、析构逆序，完全确定。

你在第 1 章写 `Image` 类时已经无意识地干过一次 RAII：构造函数 `new[]` 一块像素内存，析构函数 `delete[]` 它——当时我们管这叫「五件套的手工时代」，因为拷贝、移动都得你亲自护送。本章要做的是把它推广成全场景的模式，并且把「护送」的负担也卸掉。

先想清楚 RAII 到底解决了什么。一个资源（内存、文件、句柄、锁）的一生有两个端点：**获取**和**释放**。C 风格代码把两个端点都交给流程里的人为语句：`fopen` 在前，`fclose` 在后，中间隔着几十行业务逻辑。而业务逻辑中间可能有五条 `return`、两个 `break`、一场异常——每一条都绕过了你的 `fclose`。RAII 把「释放」这个端点从流程里摘出来，焊死在析构函数上：**只要对象活着，资源就有人管；对象死（必然发生），资源必然被放**。释放逻辑只写一处，配对关系由类型系统保证。

RAII 的普适性值得逐项点名——它远不止管内存：

| 资源 | 获取 | 释放 | 游戏语境 |
| --- | --- | --- | --- |
| 堆内存 | `new` / `malloc` | `delete` / `free` | 顶点缓冲、像素数据 |
| 文件 | `fopen` / `CreateFile` | `fclose` / `CloseHandle` | 存档、日志、资产包 |
| 互斥锁 | `mutex.lock()` | `mutex.unlock()` | 加载队列、任务队列（第 5 章展开） |
| GL 句柄 | `glCreateTextures` | `glDeleteTextures` | 纹理、着色器、FBO |
| Vulkan 对象 | `vkCreateXxx` | `vkDestroyXxx` | 第 6 节全家 |
| 音频通道 | `Mix_OpenAudio` 系 | `Mix_CloseAudio` 系 | BGM/SFX |

它们形状完全一致：一对 C 风格的 create/destroy 函数（或方法）。所以包装手法也完全一致——这就是「现场设计 RAII 包装」能力的来源。

看第一个完整例子。我们挑最朴素的 C API 资源——文件，把三条退出路径一次性演完：

```cpp
// ch02_raii_file.cpp —— RAII 第一课：文件句柄的生与死绑定到作用域（完整可编译）
// cl     /utf-8 /std:c++20 /EHsc /W4 ch02_raii_file.cpp
// clang++ -std=c++20 -Wall -Wextra ch02_raii_file.cpp -o ch02_raii_file
#define _CRT_SECURE_NO_WARNINGS   // 教学演示用 fopen/fclose（POSIX 风格 C API），关掉 MSVC 的安全警告
#include <cstdio>
#include <stdexcept>
#include <string>
#include <utility>

// 一个最典型的 C API 资源：std::FILE*。「谁打开，谁关闭」——RAII 把这句话写进类型。
class LogFile {
public:
    // 构造 = 资源获取：在构造函数里把资源拿到手；拿不到就直接抛（构造失败 = 没有对象）
    explicit LogFile(const char* path)
        : file_(std::fopen(path, "w")), path_(path)
    {
        if (!file_) throw std::runtime_error("打不开 " + std::string(path));
        std::printf("  [构造] 打开 %s\n", path_);
    }

    // 析构 = 资源释放：无论函数从哪条路退出，析构一定执行
    ~LogFile()
    {
        if (file_) {
            std::printf("  [析构] 关闭 %s\n", path_);
            std::fclose(file_);
        }
    }

    // 独占资源的类：拷贝必须删掉，移动放行（第 1 章 rule of five 的「删减版」）
    LogFile(const LogFile&) = delete;
    LogFile& operator=(const LogFile&) = delete;
    LogFile(LogFile&& other) noexcept
        : file_(other.file_), path_(other.path_)
    {
        other.file_ = nullptr;      // 关键：让对方析构时无资源可放（第 1 章的「空壳」约定）
        other.path_ = "(已移交)";
    }
    LogFile& operator=(LogFile&&) = delete;   // 简化示例：本节不需要移动赋值

    void write(const char* line) { if (file_) std::fputs(line, file_); }

private:
    std::FILE*  file_ = nullptr;
    const char* path_ = "";
};

// 场景 A：正常路径
void recordNormal() {
    LogFile log("normal.log");
    log.write("frame 1 ok\n");
    std::printf("  recordNormal：一切正常，走函数尾\n");
}   // ← log 在这里析构，文件一定被关闭——没有任何手写清理代码

// 场景 B：中途提前 return（游戏代码里最常见的退出方式）
bool recordEarlyReturn(bool mapLoaded) {
    LogFile log("early.log");
    log.write("start\n");
    if (!mapLoaded) {
        std::printf("  recordEarlyReturn：地图没加载好，提前 return\n");
        return false;   // ← 就算从这里溜走，析构照样执行
    }
    log.write("done\n");
    return true;
}

// 场景 C：异常路径——栈展开会逆序析构所有已构造完毕的局部对象（第 1 章 1.2 的规则）
void recordWithException() {
    LogFile log("exception.log");
    log.write("start\n");
    std::printf("  recordWithException：即将抛异常\n");
    throw std::runtime_error("材质加载失败");   // ← 不会跳过 log 的析构
}

int main() {
    std::printf("== 场景 A：正常路径 ==\n");
    recordNormal();

    std::printf("== 场景 B：中途提前 return ==\n");
    (void)recordEarlyReturn(false);

    std::printf("== 场景 C：抛异常 ==\n");
    try {
        recordWithException();
    } catch (const std::exception& e) {
        std::printf("  main 捕获异常：%s\n", e.what());
    }
    std::printf("== 三条路径结束：没有一处手写 fclose，也没有一个文件没关 ==\n");
    return 0;
}
```

实测输出：

```text
== 场景 A：正常路径 ==
  [构造] 打开 normal.log
  recordNormal：一切正常，走函数尾
  [析构] 关闭 normal.log
== 场景 B：中途提前 return ==
  [构造] 打开 early.log
  recordEarlyReturn：地图没加载好，提前 return
  [析构] 关闭 early.log
== 场景 C：抛异常 ==
  [构造] 打开 exception.log
  recordWithException：即将抛异常
  [析构] 关闭 exception.log
  main 捕获异常：材质加载失败
== 三条路径结束：没有一处手写 fclose，也没有一个文件没关 ==
```

三个值得咀嚼的细节：

1. **构造失败 = 没有对象**。`LogFile` 拿不到文件时直接抛异常，此时析构函数**不会**被调用（对象从未构造完成）——所以析构里的 `if (file_)` 守护的是「移动后的空壳」，不是「构造失败」。那构造函数抛异常时已经拿到的资源怎么办？看下一条。
2. **部分构造时，已构造完毕的成员会逆序析构**。如果 `LogFile` 还有别的成员（比如一个已经构造好的 `std::string` 路径），构造函数中途抛出时，这些已完成构造的成员析构函数照样执行——所以**成员本身用 RAII 类型，部分构造也安全**。这就是为什么「资源成员用智能指针/容器管」是铁律：连构造函数的异常安全都白送。
3. **移动语义在这里的职责变了**。第 1 章 `Image` 的移动是为了性能（避免深拷贝）；这里的移动是**所有权移交**——「这份资源归谁管」从一个对象转移到另一个对象，移动后源对象变空壳。这个语义转折是本章的主线，2.3 节正式展开。

顺手补上「锁」的最小例子，证明 RAII 模式跨资源通用：

```cpp
// 节选：std::lock_guard——标准库自带的锁 RAII 包装
#include <mutex>

std::mutex loadMutex;
int loadedCount = 0;

void addLoaded() {
    std::lock_guard<std::mutex> guard(loadMutex);  // 构造时 lock()
    ++loadedCount;
}   // ← guard 在这里析构，自动 unlock()——提前 return / 抛异常都不会忘了解锁
```

`lock_guard` 是「锁的 RAII」：忘记 `unlock` 这种经典事故从类型上不可能发生。锁的完整故事（为什么要锁、锁什么、死锁）将在第 5 章展开，这里只需要看清形状：**它和你手写的 `LogFile` 是同一个模式**。

### 2.2 提前 return、异常与「人总会忘」：为什么 RAII 是唯一解

有同学会想：我小心一点，在每个 `return` 前手动释放，不也一样？我们用三面镜子照一照这个想法。

**镜子一：提前 return。** 游戏代码里函数多出口是常态——参数不合法 `return`，缓存命中 `return`，加载失败 `return`。手动管理要求你在**每一个**出口前释放**每一个**已获取资源：

```cpp
// 节选：手动管理的灾难现场（反面教材，勿模仿）
bool loadEffectBad(bool ok) {
    GLuint tex = 0;
    glCreateTextures(GL_TEXTURE_2D, 1, &tex);
    GLuint fbo = 0;
    glCreateFramebuffers(1, &fbo);
    if (!ok) {
        glDeleteFramebuffers(1, &fbo);   // ← 差点忘了 fbo
        glDeleteTextures(1, &tex);       // ← 差点忘了 tex
        return false;
    }
    // ……二十行业务逻辑，又一个 return……
    // ← 这里就真的忘了：两个 GL 对象泄漏
    return true;
}
```

资源数量 × 出口数量的配对矩阵，靠人脑维护注定漏格。而 RAII 版本里，`return` 就是普通语句——局部对象析构兜底，**出口越多越体现价值**。

**镜子二：异常。** 异常沿调用链向上传播时，途经的每一层作用域都会做**栈展开**：所有已构造完毕的局部对象逆序析构。这意味着 RAII 对象的释放逻辑在异常路径上**自动执行**，而手写的 `delete`（在 throw 之后的那几行）永远执行不到。引擎里禁用异常的团队很多（错误处理策略的取舍将在第 7 章展开），但要澄清一件事：**RAII 的价值不依赖异常**——提前 return 和正常作用域退出已经占了日常路径的大头，异常只是让 RAII 的账面更赚。

**镜子三：维护。** 今天函数只有一个出口，你配平了；下个月同事在中间加了一个「提前退出」的分支——他必须知道这个函数里有三处需要手动释放。RAII 把这份「约定知识」变成类型系统的强制：**释放逻辑写在类里一次，所有使用者自动继承正确性**。这就是「人总会忘」的工程解法——不靠记得，靠编译器。

### 2.3 unique_ptr：独占所有权的零开销抽象

RAII 包装类（如 2.1 的 `LogFile`）要写移动构造、删拷贝……每个资源类都手写一套就太累了。标准库把这套「拷贝删除 + 移动放行 + 析构释放」的通用形状做成了模板：`std::unique_ptr<T>`。

**语义：独占所有权。** 同一时刻，一个 `unique_ptr` 是资源的**唯一**属主。推论三条：

- **不可拷贝**（拷贝构造/赋值被 `= delete`）——「两个属主」从类型上就不存在；
- **可移动**（移交所有权）——移交后原属主变空（`get() == nullptr`，第 1 章的「空壳」）；
- **析构即释放**——默认用 `delete`（数组特化 `unique_ptr<T[]>` 用 `delete[]`）。

**零开销抽象**，意思是：不用它你手写也一样快，用它编译器生成的代码不比裸指针慢——`sizeof(unique_ptr<T>) == sizeof(T*)`（默认 deleter 无状态时），解引用不引入额外间接，析构内联后与手写 `if (p) delete p;` 同一条机器码。这是一句有边界的表扬（「零开销抽象」也有Hidden成本的时候，章末延伸资源里 Chandler Carruth 的讲座会给你泼一盆理性的冷水），但对 `unique_ptr` 而言基本成立：**你用类型表达意图，运行期一分钱不付**。

看完整示例——注意 `Mesh` 类本身：资源成员全部交给 RAII 类型后，它自己一个特殊成员函数都不用写（第 1 章 rule of zero 的兑现），拷贝被自动禁掉、移动被自动放行，连 `noexcept` 都会随成员自动联动：

```cpp
// ch02_unique_ptr.cpp —— unique_ptr：独占所有权的零开销抽象（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_unique_ptr.cpp -o ch02_unique_ptr
#include <cassert>
#include <cstdio>
#include <memory>
#include <string>
#include <utility>
#include <vector>

class Mesh {  // 一个「重」资源：构造时分配顶点缓冲，析构时释放
public:
    explicit Mesh(std::string name, std::size_t vertexCount)
        : name_(std::move(name)),
          vertices_(std::make_unique<float[]>(vertexCount)),  // 堆内存也交给 RAII（数组版 unique_ptr）
          vertexCount_(vertexCount)
    {
        std::printf("  [构造] Mesh(%s, %zu 顶点)\n", name_.c_str(), vertexCount_);
    }
    ~Mesh() { std::printf("  [析构] Mesh(%s)\n", name_.c_str()); }

    // Mesh 自己只拥有堆内存，而内存由 unique_ptr 管理 → rule of zero（第 1 章 1.6）：
    // 五件套一个都不写——拷贝被自动禁掉（unique_ptr 不可拷贝），移动自动放行。
    const std::string& name() const { return name_; }
    std::size_t vertexCount() const { return vertexCount_; }

private:
    std::string name_;
    std::unique_ptr<float[]> vertices_;   // 拥有：数组版 unique_ptr
    std::size_t vertexCount_ = 0;
};

// 工厂函数：返回 unique_ptr —— 「谁调用，谁拥有」，所有权随返回值移交
std::unique_ptr<Mesh> loadMesh(const std::string& path) {
    std::printf("  loadMesh(%s)\n", path.c_str());
    return std::make_unique<Mesh>(path, 1024);   // 优先 make_unique，而不是 unique_ptr<Mesh>(new Mesh(...))
}

// 借用：只想「用一下」Mesh 的函数，收 const 引用/裸指针，而不是 unique_ptr 的值
std::size_t totalVertices(const std::vector<std::unique_ptr<Mesh>>& meshes) {
    std::size_t sum = 0;
    for (const auto& mesh : meshes) sum += mesh->vertexCount();
    return sum;
}

int main() {
    std::printf("== 1. sizeof：无状态 deleter 的 unique_ptr 不比裸指针多一个字节 ==\n");
    std::printf("  sizeof(Mesh*)               = %zu\n", sizeof(Mesh*));
    std::printf("  sizeof(unique_ptr<Mesh>)    = %zu\n", sizeof(std::unique_ptr<Mesh>));

    std::printf("== 2. 工厂函数：所有权随返回值移交 ==\n");
    auto hero = loadMesh("hero.gltf");       // hero 是唯一属主
    assert(hero && hero->vertexCount() == 1024);

    std::printf("== 3. 移动：独占所有权只能「移交」，不能「复制」==\n");
    std::vector<std::unique_ptr<Mesh>> scene;
    scene.push_back(std::move(hero));        // 所有权移交进容器
    assert(hero == nullptr);                 // 原属主交出后变空（第 1 章的空壳）
    scene.push_back(loadMesh("slime.gltf")); // prvalue 临时 → 直接移动进容器

    std::printf("== 4. 借用：只读访问不需要交出所有权 ==\n");
    std::printf("  场景合计顶点数 = %zu\n", totalVertices(scene));

    std::printf("== 5. release 与 reset 的区别 ==\n");
    auto tmp = std::make_unique<Mesh>("tmp.gltf", 8);
    Mesh* raw = tmp.release();               // release：交出所有权，但【不】释放！
    assert(!tmp && raw != nullptr);
    std::printf("  release 交出所有权后由我接管：重新装回 unique_ptr\n");
    tmp.reset(raw);                          // reset：接管新指针（旧的若有则先析构）

    std::printf("== 6. reset(nullptr)：立即析构旧对象 ==\n");
    tmp.reset();                             // tmp.gltf 在这一行析构
    std::printf("== 程序结束：scene 里的 Mesh 随容器逆序析构 ==\n");
    return 0;
}
```

实测输出：

```text
== 1. sizeof：无状态 deleter 的 unique_ptr 不比裸指针多一个字节 ==
  sizeof(Mesh*)               = 8
  sizeof(unique_ptr<Mesh>)    = 8
== 2. 工厂函数：所有权随返回值移交 ==
  loadMesh(hero.gltf)
  [构造] Mesh(hero.gltf, 1024 顶点)
== 3. 移动：独占所有权只能「移交」，不能「复制」==
  loadMesh(slime.gltf)
  [构造] Mesh(slime.gltf, 1024 顶点)
== 4. 借用：只读访问不需要交出所有权 ==
  场景合计顶点数 = 2048
== 5. release 与 reset 的区别 ==
  [构造] Mesh(tmp.gltf, 8 顶点)
  release 交出所有权后由我接管：重新装回 unique_ptr
== 6. reset(nullptr)：立即析构旧对象 ==
  [析构] Mesh(tmp.gltf)
== 程序结束：scene 里的 Mesh 随容器逆序析构 ==
  [析构] Mesh(hero.gltf)
  [析构] Mesh(slime.gltf)
```

四条使用纪律，请直接抄进自己的编码习惯：

1. **创建永远用 `std::make_unique<T>(...)`**（C++14 起），不要写 `unique_ptr<T>(new T(...))`：少一次裸 `new`、异常安全更好、类型写一遍（`make_unique` 的深理由见深入专题）。C++14 之前的引擎只能用后者。
2. **`release()` 交出所有权但不释放，`reset()` 放手并立即释放**——两者极易混。日常代码里 `release` 只该出现在「把所有权交给旧式 C API」的边界处，出现一次审一次。
3. **传参表达借用，不传 `unique_ptr` 的值**：只读访问收 `const Mesh&` 或 `const Mesh*`（2.9 节词汇表）；想移交所有权才按值收 `unique_ptr<Mesh>`。函数参数里出现 `unique_ptr` 按值，就等于宣告「我要拿走你的所有权」——这是个郑重承诺，别滥用。
4. **容器装重资源，装 `unique_ptr`**：`std::vector<std::unique_ptr<Mesh>>` 里元素不可拷贝但可移动，扩容时只交接指针（第 1 章深入专题的移动语义在这里全部生效），重排/删除元素时自动析构对应资源。

还有一个「白送」的语义收益值得点名：`unique_ptr` 可空、可比较 `== nullptr`，所以它天然替代了「空指针表示没有对象」的旧习——`Mesh* mesh = nullptr;` 加一句注释「可能为空，不拥有」的时代过去了，`std::unique_ptr<Mesh>` 把这些话写进了类型。

### 2.4 自定义 deleter：为任意 C API 句柄现场造 RAII 包装（GL 实战）

现在面对本章的主场任务：GL 句柄。`GLuint` 只是驱动侧某个对象在当前上下文里的**编号**，真正的东西（显存、状态）在驱动里。创建用 `glCreateTextures`，释放必须配对调用 `glDeleteTextures`——漏了就泄漏驱动侧对象（ASan 还看不见它，后面 2.11 节细说），重复调就是未定义行为。这套「编号 + 显式配对释放」的形状，GL、Vulkan、SDL、stb、PhysX 全都一个样，所以下面练成的方法一通百通。

**教学桩**：为了让不依赖任何图形环境的示例也能编译运行（也为你的实践任务打底），我们实现一套模拟 GL 的 C API——接口签名与真实 OpenGL 4.5 一致，实现是纯 CPU 假货，并内置一张「存活对象登记表」：谁忘了释放，退出时报账；谁重复释放，当场抓获。

**写法 A：`unique_ptr` + 自定义 deleter。** `unique_ptr<T>` 默认拿 `delete` 当释放手段；模板的第二个参数 **deleter** 允许你换掉它——传入任何「能以 `d(ptr)` 形式调用」的东西：函数指针、函数对象、无捕获 lambda。GL 句柄是整数不是指针，于是惯用法是「把句柄装进一个堆上的小盒子，让 deleter 在销毁盒子时顺手释放句柄」：

```cpp
// ch02_gl_raii.cpp —— 教学桩 GL + RAII 包装 + 中途提前 return 零泄漏实证（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_gl_raii.cpp -o ch02_gl_raii
#include <cstdio>
#include <cstdlib>
#include <memory>
#include <stdexcept>
#include <unordered_map>
#include <utility>

// ====================================================================
// 第 0 部分：教学桩「GL」——模拟 glCreateXxx / glDeleteXxx 的 C API。
// 接口形状与真实 OpenGL 4.5（DSA）完全一致；实现是本文件内的纯 CPU 假货，
// 内置一张「存活对象登记表」：谁忘了释放，退出时当场报账；谁重复释放，当场抓获。
// （P1 衔接：真实渲染器里把包装类里的桩调用换成 glad 的真实调用即可，类本身一行不改。）
// ====================================================================
using GLuint  = unsigned int;
using GLsizei = int;
constexpr unsigned int GL_TEXTURE_2D      = 0x0DE1;
constexpr unsigned int GL_VERTEX_SHADER   = 0x8B31;
constexpr unsigned int GL_FRAGMENT_SHADER = 0x8B30;

namespace glstub {
    inline std::unordered_map<GLuint, const char*>& live() {
        static std::unordered_map<GLuint, const char*> table;
        return table;
    }
    inline GLuint& nextId() { static GLuint id = 1; return id; }

    inline GLuint create(const char* typeName) {
        GLuint id = nextId()++;
        live()[id] = typeName;
        std::printf("    [gl] 创建 %s -> %u\n", typeName, id);
        return id;
    }
    inline void destroy(GLuint id, const char* typeName) {
        auto it = live().find(id);
        if (it == live().end()) {   // 不存在或已释放：真实驱动多半静默，教学桩当场揭发
            std::printf("!! [gl] 非法释放 %s %u（不存在或已释放）——双重释放被当场抓获\n", typeName, id);
            std::abort();
        }
        std::printf("    [gl] 释放 %s %u\n", typeName, id);
        live().erase(it);
    }
    inline std::size_t liveCount() { return live().size(); }
    inline void report() {
        if (live().empty()) {
            std::printf("  [gl] 登记表为空：GL 对象零泄漏\n");
            return;
        }
        std::printf("!! [gl] 泄漏报告：%zu 个对象未释放：\n", live().size());
        for (const auto& [id, name] : live())
            std::printf("      - %s %u\n", name, id);
    }
}  // namespace glstub

// ---- 桩 API：两种真实 GL 里都存在的签名形状 ----
// 形状一：「target + 数量 + 出参数组」，失败时写回 0（真实 glCreateTextures / glCreateFramebuffers 就长这样）
void glCreateTextures(unsigned int target, GLsizei n, GLuint* textures) {
    (void)target;   // 教学桩不区分纹理种类；真实 GL 里这里是 GL_TEXTURE_2D 等
    for (GLsizei i = 0; i < n; ++i) textures[i] = glstub::create("纹理");
}
void glCreateFramebuffers(GLsizei n, GLuint* framebuffers) {
    for (GLsizei i = 0; i < n; ++i) framebuffers[i] = glstub::create("帧缓冲");
}
void glDeleteTextures(GLsizei n, const GLuint* textures) {
    for (GLsizei i = 0; i < n; ++i) glstub::destroy(textures[i], "纹理");
}
void glDeleteFramebuffers(GLsizei n, const GLuint* framebuffers) {
    for (GLsizei i = 0; i < n; ++i) glstub::destroy(framebuffers[i], "帧缓冲");
}
// 形状二：「返回 id」，失败时返回 0（真实 glCreateShader / glDeleteShader 就长这样）
GLuint glCreateShader(unsigned int type) {
    return glstub::create(type == GL_VERTEX_SHADER    ? "顶点着色器"
                          : type == GL_FRAGMENT_SHADER ? "片元着色器"
                                                       : "着色器");
}
void glDeleteShader(GLuint shader) { glstub::destroy(shader, "着色器"); }

// ====================================================================
// 第 1 部分：RAII 包装——两种写法，都保证「每个 GL 对象恰有一个属主」
// ====================================================================

// ---- 写法 A：unique_ptr + 自定义 deleter（无状态 → sizeof 与裸指针相同）----
// deleter 是一个「怎么释放」的函数对象；它参与 TexturePtr 的类型签名（下文细讲）。
struct TextureDeleter {
    void operator()(GLuint* tex) const noexcept {
        if (tex != nullptr && *tex != 0) glDeleteTextures(1, tex);
        delete tex;                      // 装句柄的小盒子也要还
    }
};
using TexturePtr = std::unique_ptr<GLuint, TextureDeleter>;

TexturePtr makeTexture() {
    GLuint id = 0;
    glCreateTextures(GL_TEXTURE_2D, 1, &id);
    if (id == 0) return nullptr;         // 创建失败：返回空，没有资源可管
    return TexturePtr(new GLuint(id), TextureDeleter{});
}

// ---- 写法 B：句柄类（域类型带语义时更舒服；拷贝删除 + 移动放行 + 析构释放）----
class Shader {
public:
    Shader(unsigned int type, const char* label) : id_(glCreateShader(type)), label_(label) {
        if (id_ == 0) throw std::runtime_error("着色器创建失败");   // 构造失败 = 没有对象
    }
    ~Shader() {
        if (id_ != 0) glDeleteShader(id_);          // 析构 = 唯一的释放点
    }
    Shader(const Shader&) = delete;                 // 句柄不可复制：杜绝「一个对象两个属主」
    Shader& operator=(const Shader&) = delete;
    Shader(Shader&& other) noexcept                 // 可移交：所有权转移
        : id_(other.id_), label_(other.label_)
    {
        other.id_ = 0;                              // 空壳化（第 1 章的约定）
        other.label_ = "(已移交)";
    }
    Shader& operator=(Shader&& other) noexcept {
        if (this != &other) {
            if (id_ != 0) glDeleteShader(id_);      // 先放掉自己手里的旧资源
            id_ = other.id_;
            label_ = other.label_;
            other.id_ = 0;
            other.label_ = "(已移交)";
        }
        return *this;
    }
    GLuint get() const { return id_; }              // 借出裸句柄给 gl* 函数用：只借不管
    const char* label() const { return label_; }

private:
    GLuint      id_ = 0;
    const char* label_ = "";
};

// ---- 用两种包装搭一个「加载特效」流程，看三条退出路径各自的结果 ----
bool loadEffect(bool ok) {
    TexturePtr albedo = makeTexture();              // ① RAII 资源：纹理
    if (!albedo) { std::printf("  纹理创建失败\n"); return false; }
    Shader vs(GL_VERTEX_SHADER, "effect.vert");     // ② RAII 资源：着色器

    GLuint fbo = 0;                                 // ③ 对照组：最后一个裸句柄，手动管理
    glCreateFramebuffers(1, &fbo);

    if (!ok) {
        std::printf("  特效配置非法，中途提前 return——albedo/vs 自动释放，fbo 忘在了半路\n");
        return false;                               // ①② 由 RAII 兜底；③ 泄漏
    }
    glDeleteFramebuffers(1, &fbo);                  // 手动释放：忘一次就漏一次
    std::printf("  特效加载成功\n");
    return true;
}

int main() {
    std::printf("== 1. sizeof：无状态 deleter 不增加任何开销 ==\n");
    std::printf("  sizeof(GLuint*)     = %zu\n", sizeof(GLuint*));
    std::printf("  sizeof(TexturePtr)  = %zu\n", sizeof(TexturePtr));

    std::printf("== 2. 正常路径：全部资源各归各位 ==\n");
    (void)loadEffect(true);

    std::printf("== 3. 中途提前 return：RAII 兜底，只漏手动的那个 fbo ==\n");
    (void)loadEffect(false);

    std::printf("== 4. 登记表对账：应该只剩 1 个泄漏的帧缓冲 ==\n");
    glstub::report();

    std::printf("== 5. 双重释放演示（已注释）：打开注释即被桩当场抓获 ==\n");
    // GLuint dup = 0;
    // glCreateTextures(GL_TEXTURE_2D, 1, &dup);
    // glDeleteTextures(1, &dup);
    // glDeleteTextures(1, &dup);   // ← 去掉本行注释：桩打印「非法释放」并 abort
    return 0;
}
```

实测输出（重点看第 3、4 段的对照）：

```text
== 1. sizeof：无状态 deleter 不增加任何开销 ==
  sizeof(GLuint*)     = 8
  sizeof(TexturePtr)  = 8
== 2. 正常路径：全部资源各归各位 ==
    [gl] 创建 纹理 -> 1
    [gl] 创建 顶点着色器 -> 2
    [gl] 创建 帧缓冲 -> 3
    [gl] 释放 帧缓冲 3
  特效加载成功
    [gl] 释放 着色器 2
    [gl] 释放 纹理 1
== 3. 中途提前 return：RAII 兜底，只漏手动的那个 fbo ==
    [gl] 创建 纹理 -> 4
    [gl] 创建 顶点着色器 -> 5
    [gl] 创建 帧缓冲 -> 6
  特效配置非法，中途提前 return——albedo/vs 自动释放，fbo 忘在了半路
    [gl] 释放 着色器 5
    [gl] 释放 纹理 4
== 4. 登记表对账：应该只剩 1 个泄漏的帧缓冲 ==
!! [gl] 泄漏报告：1 个对象未释放：
      - 帧缓冲 6
== 5. 双重释放演示（已注释）：打开注释即被桩当场抓获 ==
```

现在把两种写法的关键差异说透。

**deleter 参与 `unique_ptr` 的类型签名。** `TexturePtr` 的完整类型是 `std::unique_ptr<GLuint, TextureDeleter>`——deleter 是类型的一部分。三个直接后果：

1. `unique_ptr<GLuint, TextureDeleter>` 和 `unique_ptr<GLuint, FileCloser>`（管理 `FILE*` 的）是**两个不同的、不可互换的类型**——你不能把纹理句柄当成文件句柄还给它，编译器替你把「张冠李戴的释放」拦在编译期。
2. **无状态 deleter 不占字节**：`TextureDeleter` 没有数据成员，`unique_ptr` 用「空基类优化」把它叠进指针里，所以 `sizeof(TexturePtr) == 8`，零开销成立。换成**有状态** deleter（比如要捕获 `device`、`allocator` 参数的），每个实例就要多存一份状态，`sizeof` 相应变大——Vulkan 包装时你会遇到这个抉择（实践任务 3）。
3. 跨类型转换不自由：想从一种 deleter 换成另一种要显式构造——工程上这反而是优点，「换释放方式」必须过一道编译器的眼。

对比 `FILE*` 的经典包装感受一下通用性（这是「包装 C API」的最小完整套路，两分钟一个）：

```cpp
// 节选：FILE* 的 unique_ptr 包装——任意 C API 套同款
struct FileCloser {
    void operator()(std::FILE* f) const noexcept {
        if (f) std::fclose(f);
    }
};
using FilePtr = std::unique_ptr<std::FILE, FileCloser>;

FilePtr log = FilePtr(std::fopen("game.log", "w"), FileCloser{});
if (log) std::fputs("hello\n", log.get());   // get() 借出裸指针给 C API
// 作用域结束：fclose 自动执行
```

**写法 B：句柄类。** `Shader` 类演示了另一种选择：不套 `unique_ptr`，直接写一个小的域类型——构造函数拿句柄、析构函数还句柄、拷贝删除、移动放行（注意移动赋值要先释放自己手里的旧句柄再接管，这正是第 1 章 1.6 节五件套纪律的「删减版」：五件里拷贝两件 `= delete`，剩下三件手写）。选型口诀：

- **一次性、无附加语义的句柄** → `unique_ptr` + deleter，两分钟搞定；
- **有域语义的类型**（要带 label、尺寸、绑定接口、日志）→ 句柄类，它就是你自己项目里的 `Texture2D`/`ShaderProgram`。

**本章铁律（也是阶段 2 检验标准的第一条）**：每个 GL 对象恰有一个 RAII 属主；裸 `GLuint` 只允许出现在两个地方——包装类内部，以及「刚创建、还没交给属主」的那一行。自检方法三级火箭：**类型级**（包装类拷贝被删，编译器拦下第二属主）、**检索级**（全代码 `grep "glDelete"`，只允许命中桩/包装类实现文件）、**运行级**（登记表对账 + 2.11 节的仪器）。

### 2.5 shared_ptr：共享所有权与控制块

`unique_ptr` 是默认答案，但有些资源确实说不清唯一属主：一段 BGM 同时被游戏世界、音频系统、暂停界面引用，谁也不该先死。这种「多个属主，最后一个负责收尾」的需求，标准库的答案是 `std::shared_ptr<T>`。

**控制块：`shared_ptr` 的记账本。** 每个 `shared_ptr` 管理的对象背后有一张**控制块**，至少装着两个计数器：**强计数**（还有几个 `shared_ptr` 拥有对象）和**弱计数**（还有几个 `weak_ptr` 观察着，加一），另存 deleter 和分配器（如果指定了）。规则精确而简单：

- 拷贝 `shared_ptr` → 强计数 +1；析构 → −1；
- **强计数归零 → 立即析构被管对象**；
- **弱计数也归零 → 释放控制块本身**（深入专题会看到「对象已析构但内存还没释放」的中间态）。

`shared_ptr` 的代价也摆上台面：它本身是**两个指针**（对象指针 + 控制块指针）；控制块要一次堆分配；计数是**原子操作**（有成本，但换来多线程下拷贝/析构的安全，2.7 节细讲）；访问对象多一跳间接。所以「万物 `shared_ptr`」是错的（章末误区一），正确姿势是：**默认 `unique_ptr`，确有共享所有权才 `shared_ptr`**。

**deleter 的类型擦除——与 `unique_ptr` 的决定性差异。** `shared_ptr` 也支持自定义 deleter，但方式完全不同：deleter 作为**构造函数参数**传入，不进类型：

```cpp
// 节选：shared_ptr 的 deleter 在构造时给，不在类型里
{
    std::shared_ptr<std::FILE> file(std::fopen("shared_log.txt", "w"), &std::fclose);
    // shared_ptr<std::FILE> 的类型不含 fclose——换个 deleter，类型不变，可以混存同一个容器
}   // 引用计数归零：fclose 被调用
```

两相对照，把这张表吃透（这是面试和 code review 的常客）：

| | `unique_ptr<T, D>` | `shared_ptr<T>` |
| --- | --- | --- |
| deleter 位置 | **类型签名**（D 是模板参数） | **构造函数参数**（类型擦除，存进控制块） |
| deleter 有状态时的大小 | 每实例多存一份状态 | 大小恒为两个指针，状态进控制块 |
| 不同 deleter 的实例能否混装容器 | 不能（类型不同） | 能（类型相同） |
| 运行成本 | 无（无状态时零开销） | 控制块 + 原子计数 + 一次间接 |
| 释放时机 | `unique_ptr` 析构即释放 | 强计数归零才析构对象 |

完整示例把控制块的行为和 `sizeof` 全家福一次性演完：

```cpp
// ch02_shared_ptr.cpp —— shared_ptr：控制块、引用计数与类型擦除的 deleter（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_shared_ptr.cpp -o ch02_shared_ptr
#define _CRT_SECURE_NO_WARNINGS   // 教学演示用 fopen/fclose，关掉 MSVC 的安全警告
#include <cstdio>
#include <memory>
#include <string>
#include <utility>

class AudioBuffer {
public:
    explicit AudioBuffer(std::string name) : name_(std::move(name)) {
        std::printf("  [构造] AudioBuffer(%s)\n", name_.c_str());
    }
    ~AudioBuffer() { std::printf("  [析构] AudioBuffer(%s)\n", name_.c_str()); }
    const std::string& name() const { return name_; }
private:
    std::string name_;
};

int main() {
    std::printf("== 1. 控制块：计数跟着拷贝走，最后一个属主负责销毁 ==\n");
    auto bgm = std::make_shared<AudioBuffer>("bgm.wav");
    std::printf("  刚造出来：use_count = %ld\n", static_cast<long>(bgm.use_count()));
    {
        auto sfx = std::make_shared<AudioBuffer>("sfx.wav");
        std::shared_ptr<AudioBuffer> p1 = bgm;      // 拷贝：控制块强计数 +1
        std::shared_ptr<AudioBuffer> p2 = bgm;
        std::printf("  两位借用者进场后：bgm.use_count = %ld\n", static_cast<long>(bgm.use_count()));
        std::printf("  p1 与 bgm 指向同一对象：%s\n", bgm == p1 ? "是" : "否");
        std::printf("  （sfx 有自己的控制块：use_count = %ld）\n", static_cast<long>(sfx.use_count()));
    }   // p1/p2/sfx 析构：各自计数递减；bgm 不受影响
    std::printf("  借用者离场后：bgm.use_count = %ld，对象还活着\n",
                static_cast<long>(bgm.use_count()));

    std::printf("== 2. 最后一个属主放手，对象才析构 ==\n");
    auto temp = std::make_shared<AudioBuffer>("temp.wav");
    temp = nullptr;   // 放手：引用计数 1 → 0，析构发生在这一行
    std::printf("  （析构日志在上面一行：reset 当场触发，不是等出作用域）\n");

    std::printf("== 3. deleter 类型擦除：fclose 不出现在类型里 ==\n");
    {
        // shared_ptr 允许「构造时给释放办法」：类型不变，大小不变
        std::shared_ptr<std::FILE> file(std::fopen("shared_log.txt", "w"), &std::fclose);
        if (file) {
            std::fputs("shared_ptr 管理的文件\n", file.get());
            std::printf("  sizeof(shared_ptr<FILE>) = %zu\n", sizeof(file));
        }
    }   // 引用计数归零：fclose 被自动调用（析构日志不显示，文件已关）
    std::printf("  离开作用域：fclose 已随计数归零自动执行\n");

    std::printf("== 4. sizeof 全家福：谁进类型签名，谁被擦除 ==\n");
    struct NopDeleter { void operator()(int*) const noexcept {} };
    std::printf("  sizeof(unique_ptr<int>)                 = %zu\n", sizeof(std::unique_ptr<int>));
    std::printf("  sizeof(unique_ptr<int, NopDeleter>)     = %zu（无状态 deleter：与裸指针同大）\n",
                sizeof(std::unique_ptr<int, NopDeleter>));
    std::printf("  sizeof(unique_ptr<int, void(*)(int*)>)  = %zu（deleter 有状态：多一个函数指针）\n",
                sizeof(std::unique_ptr<int, void (*)(int*)>));
    std::printf("  sizeof(shared_ptr<int>)                 = %zu（deleter 被擦除：永远两个指针）\n",
                sizeof(std::shared_ptr<int>));
    return 0;
}
```

实测输出：

```text
== 1. 控制块：计数跟着拷贝走，最后一个属主负责销毁 ==
  [构造] AudioBuffer(bgm.wav)
  刚造出来：use_count = 1
  [构造] AudioBuffer(sfx.wav)
  两位借用者进场后：bgm.use_count = 3
  p1 与 bgm 指向同一对象：是
  （sfx 有自己的控制块：use_count = 1）
  [析构] AudioBuffer(sfx.wav)
  借用者离场后：bgm.use_count = 1，对象还活着
== 2. 最后一个属主放手，对象才析构 ==
  [构造] AudioBuffer(temp.wav)
  [析构] AudioBuffer(temp.wav)
  （析构日志在上面一行：reset 当场触发，不是等出作用域）
== 3. deleter 类型擦除：fclose 不出现在类型里 ==
  sizeof(shared_ptr<FILE>) = 16
  离开作用域：fclose 已随计数归零自动执行
== 4. sizeof 全家福：谁进类型签名，谁被擦除 ==
  sizeof(unique_ptr<int>)                 = 8
  sizeof(unique_ptr<int, NopDeleter>)     = 8（无状态 deleter：与裸指针同大）
  sizeof(unique_ptr<int, void(*)(int*)>)  = 16（deleter 有状态：多一个函数指针）
  sizeof(shared_ptr<int>)                 = 16（deleter 被擦除：永远两个指针）
  [析构] AudioBuffer(bgm.wav)
```

三个使用要点：

1. **创建优先 `std::make_shared<T>(...)`**：一次分配同时装下对象和控制块，还有缓存局部性红利——为什么、代价是什么，留给本章深入专题用实验拆解。
2. **`use_count()` 只用于调试与日志**，别用它写业务逻辑（多线程下读到的是瞬间值）；「是否还有人用」的正确问法是 `weak_ptr` + `lock()`（下一节）。
3. **数组别用 `shared_ptr<T[]>`**（C++17 起才有支持）：数组要共享所有权本身就是设计问号，通常 `std::vector<T>` 或 `std::vector<std::unique_ptr<T>>` 更诚实。

### 2.6 循环引用与 weak_ptr：破环、观察与安全回访

`shared_ptr` 的记账本有个盲区：**计数只减不放过环**。看一个游戏里最常见的形状——玩家拥有任务，任务回指属主（要显示「谁的任务」）：

```cpp
// 节选：成环的形状（完整实验与输出见下方完整示例）
class Player {
    std::vector<std::shared_ptr<QuestBad>> quests;   // 拥有：我接的任务
};
class QuestBad {
    std::shared_ptr<Player> owner;                    // ⚠ 回指属主：也「拥有」玩家
};
// hero->quests 里存着 quest，quest->owner 又攥着 hero：
// hero 的强计数 ≥ 2，quest 的强计数 ≥ 2。作用域结束，各只剩 1——永不归零，永不析构。
```

栈帧撤了，两个对象还手拉手站在堆上，谁也等不到析构——**内存泄漏，而且如果对象里还有 GL 句柄，驱动侧对象一起漏**。这就是「循环引用」：`shared_ptr` 表达的是「我们共同保证你活着」，两个对象互相这样保证，就构成了逻辑上的死锁。

**破环工具：`std::weak_ptr<T>`**——指向对象但**不增加强计数**的观察者。语义三件套：

- `wp.expired()`：对象死了没？（强计数归零没？）
- `wp.lock()`：尝试把观察升级为临时的 `shared_ptr`——活着就返回一个能安全使用的强属主（计数临时 +1），死了返回空；
- 用之前必须 `lock()`——**weak 不保活，只报信**。

工程口诀：**拥有沿一个方向走（`shared_ptr`/`unique_ptr`），回指与观察走反方向（`weak_ptr`）**。玩家拥有任务没问题；任务回看属主，改用 `weak_ptr`。完整对照实验：

```cpp
// ch02_cyclic_weak.cpp —— 循环引用的泄漏实证与 weak_ptr 破环（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_cyclic_weak.cpp -o ch02_cyclic_weak
#include <cstdio>
#include <memory>
#include <string>
#include <vector>

class Player;   // 前置声明：两个类互相引用时的标准姿势

class Player {
public:
    explicit Player(std::string name) : name_(std::move(name)) {
        std::printf("  [构造] Player(%s)\n", name_.c_str());
    }
    ~Player() { std::printf("  [析构] Player(%s)\n", name_.c_str()); }
    const std::string& name() const { return name_; }
    std::vector<std::shared_ptr<class QuestBad>>  badQuests;    // 拥有：我接的任务
    std::vector<std::shared_ptr<class QuestGood>> goodQuests;
private:
    std::string name_;
};

// ---- 反面版本：任务也用 shared_ptr 回指玩家 → 成环，谁也死不了 ----
class QuestBad {
public:
    explicit QuestBad(std::string title) : title_(std::move(title)) {
        std::printf("  [构造] QuestBad(%s)\n", title_.c_str());
    }
    ~QuestBad() { std::printf("  [析构] QuestBad(%s)\n", title_.c_str()); }
    std::shared_ptr<Player> owner;                // ⚠ 回指属主，shared_ptr 成环
private:
    std::string title_;
};

// ---- 正面版本：回指改 weak_ptr——「看着你，但不替你续命」----
class QuestGood {
public:
    explicit QuestGood(std::string title) : title_(std::move(title)) {
        std::printf("  [构造] QuestGood(%s)\n", title_.c_str());
    }
    ~QuestGood() { std::printf("  [析构] QuestGood(%s)\n", title_.c_str()); }
    std::weak_ptr<Player> owner;                  // 观察者：不加计数
    std::string ownerNameSafe() const {
        if (std::shared_ptr<Player> p = owner.lock())   // 用之前 lock()：还活着才用
            return p->name();
        return "(属主已下线)";
    }
private:
    std::string title_;
};

int main() {
    std::printf("== A. 反面：shared_ptr 成环，main 结束也等不来析构 ==\n");
    {
        auto hero  = std::make_shared<Player>("勇者");
        auto quest = std::make_shared<QuestBad>("屠龙");
        hero->badQuests.push_back(quest);
        quest->owner = hero;
        std::printf("  hero.use_count = %ld, quest.use_count = %ld（互相各加了一票）\n",
                    static_cast<long>(hero.use_count()), static_cast<long>(quest.use_count()));
    }
    // 作用域结束：hero 计数 2→1（quest->owner 还攥着），不归零 → 两个析构都没发生
    std::printf("  作用域已结束——注意上面没有任何析构日志（实证泄漏！）\n");

    std::printf("== B. 正面：回指改 weak_ptr，环被剪断 ==\n");
    {
        auto hero  = std::make_shared<Player>("勇者");
        auto quest = std::make_shared<QuestGood>("屠龙");
        hero->goodQuests.push_back(quest);
        quest->owner = hero;                      // weak_ptr 赋值：不加计数
        std::printf("  hero.use_count = %ld（quest 的回指没加分）\n", static_cast<long>(hero.use_count()));
        std::printf("  quest 查询属主：%s\n", quest->ownerNameSafe().c_str());
    }
    // 作用域结束：hero 计数 1→0 → Player 析构；quest 计数 1→0 → QuestGood 析构
    std::printf("== C. 属主先死时，weak_ptr 摸得到「已失效」 ==\n");
    std::weak_ptr<Player> spy;
    {
        auto ghost = std::make_shared<Player>("幽灵");
        spy = ghost;
        std::printf("  活着时：spy.expired() = %s\n", spy.expired() ? "true" : "false");
    }
    std::printf("  属主死后：spy.expired() = %s\n", spy.expired() ? "true" : "false");
    std::printf("== 程序结束 ==\n");
    return 0;
}
```

实测输出（A 段的两行「构造」之后，你等不到任何析构）：

```text
== A. 反面：shared_ptr 成环，main 结束也等不来析构 ==
  [构造] Player(勇者)
  [构造] QuestBad(屠龙)
  hero.use_count = 2, quest.use_count = 2（互相各加了一票）
  作用域已结束——注意上面没有任何析构日志（实证泄漏！）
== B. 正面：回指改 weak_ptr，环被剪断 ==
  [构造] Player(勇者)
  [构造] QuestGood(屠龙)
  hero.use_count = 1（quest 的回指没加分）
  quest 查询属主：勇者
  [析构] Player(勇者)
  [析构] QuestGood(屠龙)
== C. 属主先死时，weak_ptr 摸得到「已失效」 ==
  [构造] Player(幽灵)
  活着时：spy.expired() = false
  [析构] Player(幽灵)
  属主死后：spy.expired() = true
== 程序结束 ==
```

自查循环引用的土办法，好用得惊人：**给可疑类加上析构日志（第 1 章 Tracer 的老手艺），盯着析构日志有没有出现**。析构迟迟不来而程序逻辑上「早该死了」，八成是环。`weak_ptr` 的另两个惯用场景——资源缓存（缓存持有 weak，正在用的系统持有 strong，没人用时自动卸载）与观察者回调（下一节）——会在你渲染器和事件系统里反复出现（事件系统与观察者生命周期的工程治理还将在第 4 节系统展开）。

### 2.7 「引用计数原子性」≠「对象本身线程安全」

`shared_ptr` 的控制块计数是**原子操作**，这句话常被读成「`shared_ptr` 是线程安全的」——错，而且错得危险。精确表述分三层：

1. **控制块计数是原子的** ✅：多个线程同时拷贝/析构**各自持有的** `shared_ptr` 实例（指向同一对象），计数不会错——这是「引用计数原子性」的准确含义。
2. **被管的 `T` 对象没有任何保护** ❌：四个线程同时 `sp->masterVolume = sp->masterVolume + 1`，是教科书级**数据竞争**——未定义行为，不是「偶尔丢几次更新」。`shared_ptr` 管的只是「谁负责销毁」，对象的成员同步它一概不管，该上锁上锁（第 5 章展开）。
3. **同一个 `shared_ptr` 实例被多线程同时读写也是竞争** ❌：线程 A 执行 `sp = other;`（写实例本身）而线程 B 执行 `auto local = sp;`（读实例本身），竞争的是这个 `shared_ptr` 变量，不是控制块。要么每线程各持自己的 `shared_ptr` 副本（第 1 层的安全只覆盖这种），要么用互斥锁保护（标准还提供 `std::atomic<std::shared_ptr<T>>` 特化，C++20 起，何时需要它是第 5 章的话题）。

看实测：A 组四线程各拷贝/析构十万次，计数精确回到 1；B 组对同一对象的普通 `int` 并发自增，期望 400000，实际跑三次三次不同——丢失更新触目惊心：

```cpp
// ch02_atomic_vs_safe.cpp —— 引用计数的原子性 ≠ 对象本身的线程安全（完整可编译）
// clang++ -std=c++20 -Wall -Wextra -pthread ch02_atomic_vs_safe.cpp -o ch02_atomic_vs_safe
// 案子 B 故意制造数据竞争（未定义行为）作教学演示；工程上请用第 5 章的同步手段。
#include <cstdio>
#include <memory>
#include <thread>
#include <vector>

class AudioMixer {
public:
    int masterVolume = 0;   // 演示用：普通 int，没有任何保护
};

int main() {
    constexpr int kThreads    = 4;
    constexpr int kIterations = 100000;

    std::printf("== A. 控制块计数是原子的：多线程拷贝/销毁不丢计数 ==\n");
    auto mixer = std::make_shared<AudioMixer>();
    {
        std::vector<std::thread> threads;
        for (int t = 0; t < kThreads; ++t)
            threads.emplace_back([&mixer] {
                for (int i = 0; i < kIterations; ++i) {
                    std::shared_ptr<AudioMixer> local = mixer;  // 原子 ++
                    // ……假装在用混音器……
                }   // local 析构：原子 --
            });
        for (auto& th : threads) th.join();
    }
    std::printf("  %d 个线程各拷贝/销毁 %d 次后：use_count = %ld（精确回到 1）\n",
                kThreads, kIterations, static_cast<long>(mixer.use_count()));

    std::printf("== B. 对象本身毫无保护：并发写普通成员 = 数据竞争 ==\n");
    {
        std::vector<std::thread> writers;
        for (int t = 0; t < kThreads; ++t)
            writers.emplace_back([&mixer] {
                for (int i = 0; i < kIterations; ++i)
                    mixer->masterVolume = mixer->masterVolume + 1;  // 读-改-写，非原子
            });
        for (auto& th : writers) th.join();
    }
    std::printf("  期望 masterVolume = %d，实际 = %d（丢失更新）\n",
                kThreads * kIterations, mixer->masterVolume);
    std::printf("  结论：shared_ptr 管的只是「谁负责销毁」；对象成员的同步它一概不管\n");
    std::printf("        （数据竞争的本质与治理将在第 5 章展开）\n");
    return 0;
}
```

实测输出（B 组数字每次运行都不同，这本身就是数据竞争的病征；A 组稳定）：

```text
== A. 控制块计数是原子的：多线程拷贝/销毁不丢计数 ==
  4 个线程各拷贝/销毁 100000 次后：use_count = 1（精确回到 1）
== B. 对象本身毫无保护：并发写普通成员 = 数据竞争 ==
  期望 masterVolume = 400000，实际 = 185511（丢失更新）
  结论：shared_ptr 管的只是「谁负责销毁」；对象成员的同步它一概不管
        （数据竞争的本质与治理将在第 5 章展开）
```

（B 组是**故意**的未定义行为演示，跑三次实际值分别为 185511、130376、174569——「能跑出数」不等于「没竞争」，这正是第 5 章要建立的概念。）还有一条游戏专属提醒：`make_shared` 把对象和控制块放进同一块内存（深入专题），于是「一个线程写对象成员」和「另一个线程拷贝 `shared_ptr`（读写控制块计数）」可能落在同一缓存行上互相拖累——性能层面的伪共享，概念记下，第 5、6 章展开。

### 2.8 enable_shared_from_this 与 lambda 捕获 this：两类「回身取自己」的坑

**坑一：在成员函数里给自己开 `shared_ptr`。** 异步系统常见需求：`Fireball` 的成员函数要把「自己」交给调度器/音频系统暂存。新手的写法 `std::shared_ptr<Fireball>(this)` 看似顺理成章，实则**给同一个对象开出了第二张控制块**——新旧两批属主各记各的账，最后各销各的，**双重释放**。正确姿势：类继承 `std::enable_shared_from_this<Fireball>`，成员函数里调 `shared_from_this()`——它复用**出生时预埋**的那张控制块，只是计数 +1。前提要记牢：**对象必须已经被 `shared_ptr` 接管**（通常是刚 `make_shared` 出来的），否则 C++17 起 `shared_from_this()` 抛 `std::bad_weak_ptr`。

**坑二：lambda 捕获 `this`。** `[this]` 捕获的是裸指针——它**不会**让对象多活一纳秒。回调登记时对象还活着，执行时对象可能早死了：这就是第 1 章 1.11 节 HUD 悬垂崩溃的「异步回调」翻版，而且更隐蔽，因为崩溃点在回调执行处，离注册处隔着一整个事件循环。修复套路与坑一同源：**类继承 `enable_shared_from_this`，回调里捕获 `weak_from_this()`，执行时 `lock()`**——活着就干，死了安全跳过。lambda 捕获的完整机制（按值/按引用/init capture）将在第 3 章展开，这里先掌握这个保命组合拳。

完整示例一次演完三个场景：

```cpp
// ch02_shared_from_this.cpp —— enable_shared_from_this 与 lambda 捕获 this 的悬垂（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_shared_from_this.cpp -o ch02_shared_from_this
#include <cassert>
#include <cstdio>
#include <functional>
#include <memory>
#include <string>
#include <utility>
#include <vector>

// 极简「延时回调」调度器：收集 lambda，等「下一帧」再统一执行
class Scheduler {
public:
    void post(std::function<void()> job) { jobs_.push_back(std::move(job)); }
    void runAll() {
        for (auto& job : jobs_) job();
        jobs_.clear();
    }
private:
    std::vector<std::function<void()>> jobs_;
};

// 正面教材：继承 enable_shared_from_this，回调里攥 weak_ptr 而不是裸 this
class Fireball : public std::enable_shared_from_this<Fireball> {
public:
    explicit Fireball(std::string caster) : caster_(std::move(caster)) {
        std::printf("  [构造] Fireball(%s 施放)\n", caster_.c_str());
    }
    ~Fireball() { std::printf("  [析构] Fireball(%s 施放)\n", caster_.c_str()); }

    // 场景一：成员函数内部要把「自己的 shared_ptr」交给异步系统（常见需求）
    std::shared_ptr<Fireball> self() {
        // 错误姿势：return std::shared_ptr<Fireball>(this);
        //   —— 给同一个对象开出第二张控制块，两批属主互不知情 → 双重释放
        return shared_from_this();   // ✅ 复用同一张控制块，只是计数 +1
    }

    // 场景二：把回调挂到调度器。回调晚于对象销毁执行，是游戏异步代码的常态
    void scheduleReport(Scheduler& sched) {
        std::weak_ptr<Fireball> weakSelf = weak_from_this();   // C++17 起
        sched.post([weakSelf] {
            if (std::shared_ptr<Fireball> self = weakSelf.lock()) {
                std::printf("  [回调] 火球还活着：%s 的火球命中\n", self->caster_.c_str());
            } else {
                std::printf("  [回调] 火球已销毁，安全跳过（没有悬垂访问）\n");
            }
        });
    }

private:
    std::string caster_;
};

// 反面教材：直接捕获 this —— 对象一死，回调里就是悬垂指针（第 1 章 HUD 崩溃的翻版）
class Trap {
public:
    void scheduleReport(Scheduler& sched) {
        sched.post([this] {   // ⚠ 捕获裸 this：它不会让对象多活一纳秒
            (void)this;   // 演示刻意不碰成员（避免未定义行为直接炸掉示例）；真实代码已在访问已释放内存
            std::printf("  [回调] Trap 还在（本例没碰成员所以没崩；真实代码已在使用已释放内存）\n");
        });
    }
};

int main() {
    std::printf("== 1. shared_from_this：成员函数里拿「自己的」shared_ptr ==\n");
    std::shared_ptr<Fireball> fire = std::make_shared<Fireball>("法师");
    std::shared_ptr<Fireball> alsoFire = fire->self();
    assert(fire == alsoFire);
    std::printf("  use_count = %ld（同一对象、同一控制块，多了一个属主）\n",
                static_cast<long>(fire.use_count()));

    std::printf("== 2. 回调晚于销毁：weak_from_this 版安全跳过 ==\n");
    {
        Scheduler sched;
        {
            auto temp = std::make_shared<Fireball>("学徒");
            temp->scheduleReport(sched);
        }                  // 学徒的火球在这里析构
        sched.runAll();    // 「下一帧」回调才执行
    }

    std::printf("== 3. 对照：捕获 this 的版本（对象已死，回调还攥着 this）==\n");
    {
        Scheduler sched;
        {
            auto trap = std::make_unique<Trap>();
            trap->scheduleReport(sched);
        }
        sched.runAll();    // this 已悬垂——真实工程中这里就是第 1 章 1.11 的崩溃现场
    }
    std::printf("== 程序结束 ==\n");
    return 0;
}
```

实测输出（注意最后：`fire` 的析构在 main 末尾，晚于「程序结束」打印）：

```text
== 1. shared_from_this：成员函数里拿「自己的」shared_ptr ==
  [构造] Fireball(法师 施放)
  use_count = 2（同一对象、同一控制块，多了一个属主）
== 2. 回调晚于销毁：weak_from_this 版安全跳过 ==
  [构造] Fireball(学徒 施放)
  [析构] Fireball(学徒 施放)
  [回调] 火球已销毁，安全跳过（没有悬垂访问）
== 3. 对照：捕获 this 的版本（对象已死，回调还攥着 this）==
  [回调] Trap 还在（本例没碰成员所以没崩；真实代码已在使用已释放内存）
== 程序结束 ==
  [析构] Fireball(法师 施放)
```

一句工程忠告：**异步回调的默认姿势就是「捕获 weak，lock 再用」**。凡是你把成员函数（或 lambda）登记给队列、定时器、事件总线、网络层的地方，都值得问一句：「执行的时候，this 还活着吗？谁保证？」

### 2.9 所有权词汇表：拥有、借用与生存期责任

现在把全章的零件拧成一张接口设计的词汇表。C++ 没有 GC，于是「谁拥有、谁借用、借用多久」必须**显式地写在类型和接口里**——这不是负担，是 C++ 给你的表达能力。

**拥有（ownership）**：对资源的生死负责，负责释放。表达方式：值成员、`std::unique_ptr`（独占）、`std::shared_ptr`（共享）、容器（拥有其元素）。

**借用（borrowing）**：临时访问，不负责生死，**绝不释放**。表达方式：`T&` / `const T&`（非空借用）、`T*` / `const T*`（可空借用，仅作观察者，**永不 `delete`**）、`std::span<const T>`（借一段连续数据）、`std::string_view`（借一段字符串）。后两者是 C++20 / C++17 的词汇类型，借而不拥有是它们存在的全部意义（选型细节将在第 3 章展开；引擎停留在 C++17 时，`span` 可用「指针 + 长度」参数或 gsl::span 平替）。

| 意图 | 接口形状 | 生存期责任 |
| --- | --- | --- |
| 「给我，归你管」 | `void install(std::unique_ptr<Mesh> m)` | 调用方移交，被调方拥有并负责释放 |
| 「我们一起保它活着」 | `void bind(std::shared_ptr<Audio> a)` | 共享所有权，最后一个属主收尾 |
| 「我用一下，别销毁」 | `void draw(const Mesh& m)` | 调用窗口内有效，双方心照不宣 |
| 「看一眼，可能没有」 | `const Mesh* find(...)` | 同上，且可为空；永不 delete |
| 「借这一段数据」 | `void update(std::span<const float> uv)` | 语句级借用，不存储 |
| 「借这个名字」 | `void rename(std::string_view n)` | 同上；要保存就拷进自己的 `string` |

借用的全部风险浓缩成一句话：**借用不延长生存期**。`string_view` 绑了临时 `string`，语句结束就是悬垂（第 1 章 HUD 教训的所有权版）；`span` 绑了临时 `vector`，同理。规则：借用只活在「当前调用」或文档明确标注的窗口内；要过夜，请拥有（拷贝 / 移入容器 / 智能指针）：

```cpp
// ch02_borrow.cpp —— 拥有与借用：接口里写清「生存期责任」（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_borrow.cpp -o ch02_borrow
#include <cassert>
#include <cstdio>
#include <span>
#include <string>
#include <string_view>
#include <vector>

// 借用者：只读一串 UV 数据——span 不拥有数据，也绝不释放
float maxU(std::span<const float> uvs) {
    float best = 0.f;
    for (float v : uvs) best = v > best ? v : best;
    return best;
}

// 借用者：string_view 借一段名字——不拷贝、不拥有（starts_with 是 C++20；C++17 用 rfind("boss_", 0) == 0）
bool isBoss(std::string_view name) { return name.starts_with("boss_"); }

// 拥有者：把名字存下来（拷贝进自己的 string），这才谈得上「保存」
class Enemy {
public:
    explicit Enemy(std::string_view name) : name_(name) {}   // 参数借用，成员拥有
    const std::string& name() const { return name_; }
private:
    std::string name_;   // 拥有：一份独立拷贝
};

int main() {
    std::vector<float> uv = { 0.f, 0.5f, 1.f };
    float m = maxU(uv);                       // vector → span：隐式借用，零拷贝
    assert(m == 1.f);

    Enemy e("boss_dragon");                   // 构造时拷贝一份，此后自给自足
    assert(isBoss(e.name()));                 // string → string_view：只借一眼

    // ⚠ 反面教材（不执行，只说明「借用不延长生存期」）：
    // std::string_view bad = std::string("临时");   // 临时 string 本行结束即析构，bad 悬垂
    // auto keep = std::span<const float>(std::vector<float>{1.f, 2.f, 3.f});
    //                                              // 临时 vector 语句末析构，keep 悬垂
    std::printf("拥有者持有数据；借用只在调用窗口内有效，越过窗口即悬垂（第 1 章 HUD 教训的所有权版）。\n");
    return 0;
}
```

实测输出：

```text
拥有者持有数据；借用只在调用窗口内有效，越过窗口即悬垂（第 1 章 HUD 教训的所有权版）。
```

把这套词汇用起来的自查清单（每个接口过一遍，十秒钟）：

1. 这个参数**谁拥有**？函数结束后资源在哪、谁释放？
2. 这个参数是不是只是**借用**？那为什么不用 `const&` / `span` / `string_view`，而要让调用方交出智能指针？
3. 我存下的这个引用 / 指针 / view，**过夜了吗**？它的主人在我之前死怎么办？

### 2.10 静态/全局对象的析构顺序陷阱（概念级）

第 1 章 1.2 节存储期表格里那句「静态存储期：析构顺序陷阱（第 2 章展开）」，现在兑现。规则两条：

1. **同一翻译单元内**：静态/全局对象按定义顺序构造，按逆序析构——确定、安全，和局部对象一个道理。
2. **跨翻译单元**：标准**不规定**不同 `.cpp` 的全局对象初始化先后（著名的 static initialization order fiasco）——于是析构顺序也无从谈起。A.cpp 的全局 `TextureCache` 析构函数里调用了 B.cpp 的全局 `Logger`，而链接器恰好让 `Logger` 先死：你的析构就在调用一具尸体上的方法——未定义行为。它在崩溃转储里的典型形状：`main` 已经正常返回，进程却在静态析构段崩溃。

工程缓解三招，按推荐排序：

1. **别依赖全局对象的构造/析构做资源管理**——引擎惯例是显式的 `Engine::Init()` / `Engine::Shutdown()` 序列，把生死交给你能看见的代码（这与本章主线一致：资源生命周期应当是设计出来的，不是全局变量碰巧活到最后的副产品）。
2. **必须用全局单例时，用函数局部静态**（Meyers singleton）：`static` 局部变量首次执行到声明处才构造（C++11 起保证线程安全地只构造一次——并发语义第 5 章展开），析构仍在 `main` 后按构造逆序，但「用到才存在」大幅缩小了跨 TU 顺序的暴露面。
3. 全局对象的析构函数里**只做无依赖的收尾**（比如释放纯自己的内存），不调用任何别的全局/单例。

完整示例演示「同 TU 逆序安全」与 Meyers 单例写法：

```cpp
// ch02_static_order.cpp —— 静态/全局对象的析构顺序陷阱与工程缓解（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_static_order.cpp -o ch02_static_order
#include <cstdio>

class Logger {
public:
    void write(const char* line) { std::printf("  [logger] %s\n", line); }
};

// ---------- 同一翻译单元内：构造正序、析构逆序，天然安全（第 1 章 1.2 的规则）----------
Logger g_logger;    // 先声明 → 先构造 → 后析构

class TextureCache {
public:
    ~TextureCache() {
        // 析构里使用另一个全局对象：同一 TU 内 TextureCache 后构造 → 先析构，g_logger 还活着
        g_logger.write("TextureCache 析构：32 张纹理写回磁盘");
    }
};
TextureCache g_cache;   // 声明在 g_logger 之后 ✔

// ---------- 跨翻译单元：标准【不规定】不同 .cpp 的全局初始化顺序 ----------
// 假如 g_logger 定义在另一个 .cpp：完全可能「cache 先构造、logger 后构造」，
// 析构逆序时 logger 先死，cache 的析构就踩在一具尸体上——未定义行为。
// 这类事故在崩溃转储里的形状：main 结束后、进程退出前的静态析构段崩溃。

// ---------- 工程缓解：函数局部静态（Meyers 单例）——首次使用才构造 ----------
Logger& logger() {
    static Logger instance;   // 首次调用那行才构造；析构仍按构造逆序，但「用到才存在」大幅缩小暴露面
    return instance;
}

class SoundBank {
public:
    ~SoundBank() { logger().write("SoundBank 析构：释放 8 MB 音频"); }
};

int main() {
    logger().write("main 开始");
    {
        SoundBank bank;   // 局部对象：生死完全确定
        logger().write("一局游戏进行中");
    }
    logger().write("main 结束（全局/静态对象的析构在此之后按构造逆序发生）");
    return 0;
}
```

实测输出（注意最后两行的次序：局部对象 → main 结束 → 全局逆序析构，`TextureCache` 析构时 `g_logger` 还活着）：

```text
  [logger] main 开始
  [logger] 一局游戏进行中
  [logger] SoundBank 析构：释放 8 MB 音频
  [logger] main 结束（全局/静态对象的析构在此之后按构造逆序发生）
  [logger] TextureCache 析构：32 张纹理写回磁盘
```

### 2.11 泄漏与悬垂检测：AddressSanitizer 与 MSVC CRT 调试堆

第 1 章 1.11 节你见过调试堆的毒化模式（`0xCC`/`0xCD`/`0xDD`），本章把它升级成两件正经仪器，加上教学桩的登记表，凑齐「零泄漏」的完整证据链。先摆清分工，再给操作路径：

| 工具 | 抓什么 | 抓不住什么 | 何处可用 |
| --- | --- | --- | --- |
| **AddressSanitizer (ASan)** | 释放后使用、越界、双重释放（当场中断，带分配/释放两侧调用栈） | **Windows 版不查堆泄漏**（LSan 未随 Windows 端提供，实测确认）；**驱动侧 GL 对象泄漏完全不可见** | MSVC `/fsanitize=address`（x64，Debug/Release 皆可）；clang/gcc `-fsanitize=address`（Linux/macOS 自带 LSan，退出时顺带报泄漏） |
| **MSVC CRT 调试堆** | 堆内存泄漏（退出总报告 + 区间检测 + 定位到分配序号） | Release 配置无效；句柄/文件/GL 对象泄漏 | MSVC Debug 运行库（`/MDd` `/MTd`），免编译器插桩 |
| **教学桩登记表 / 真实工具** | GL 对象泄漏、非法释放 | — | 你自己的桩（2.4）；真实渲染器用 RenderDoc 资源列表核对（第 3 节主线工具） |

三者互为补集：ASan 抓「用错了」，CRT 抓「堆上漏了」，登记表抓「驱动侧漏了」。一个 GL 纹理泄漏可以三层全绿也照样漏——因为它漏在驱动进程的显存里，你的堆上只有一个 4 字节的 `GLuint`。

**ASan 操作路径（MSVC）**：Visual Studio 里，项目属性 → 配置属性 → C/C++ → 常规 → 「启用地址清除程序」选「是」（等效于命令行 `cl /fsanitize=address`，建议 x64 配置；它由编译器插桩 + 运行时库组成，与「编辑并继续」等特性互斥）。跑起来后，第一处非法访问当场中断，报告形如本节示例的输出（含 `freed by thread T0 here:` 的完整两侧调用栈）。

**CRT 调试堆操作路径（MSVC）**：用 Debug 配置（`/MDd`）编译；包含 `<crtdbg.h>`；三件套代码：

```cpp
// 节选：CRT 调试堆三件套（完整可运行示例见下）
#include <crtdbg.h>
// ① 全局开关：退出时自动查泄漏；报告重定向到 stderr（默认只发给调试器，控制台看不见）
_CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_FILE);
_CrtSetReportFile(_CRT_WARN, _CRTDBG_FILE_STDERR);
_CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
// ② 区间检测：只查「这段代码」的净增量——正好对上「中途提前 return」场景
_CrtMemState before = {}, after = {}, diff = {};
_CrtMemCheckpoint(&before);
/* ……可疑代码…… */
_CrtMemCheckpoint(&after);
if (_CrtMemDifference(&diff, &before, &after)) _CrtMemDumpAllObjectsSince(&before);
// ③ 定位：报告里的 {140} 是分配序号，_CrtSetBreakAlloc(140) 可让第 140 次分配直接断下来
```

完整实验室——三个案子喂给两种工具：

```cpp
// ch02_leak_lab.cpp —— 泄漏实验室：给 ASan / CRT 调试堆喂三个案子（完整可编译）
//
// 编译与运行（选一种）：
//   ① MSVC + ASan（Debug/Release 皆可，建议 x64）：
//        cl /utf-8 /std:c++20 /EHsc /fsanitize=address ch02_leak_lab.cpp
//   ② MSVC + CRT 调试堆（仅 Debug 运行库 /MDd 或 /MTd）：
//        cl /utf-8 /std:c++20 /EHsc /MDd ch02_leak_lab.cpp
//   ③ Clang/GCC + ASan：
//        clang++ -std=c++20 -Wall -Wextra -g -fsanitize=address ch02_leak_lab.cpp
#define _CRT_SECURE_NO_WARNINGS
#include <cstdio>
#include <cstring>

#if defined(_MSC_VER) && defined(_DEBUG)
#include <crtdbg.h>   // CRT 调试堆：只在 Debug 运行库下有效
#endif

namespace {
    // 案子 A：干净区——构造即释放，工具应零报告（对照组）
    void caseA_clean() {
        char* p = new char[64];
        std::memset(p, 0, 64);
        delete[] p;
        std::printf("案子A：干净路径跑完\n");
    }

    // 案子 B：中途提前 return，把 new 出来的帧缓冲忘在半路（泄漏）
    bool caseB_earlyReturn(bool ok) {
        char* frame = new char[1024];   // 裸 new：本章主角登场前的「事故现场」
        if (!ok) {
            std::printf("案子B：提前 return——frame 的 1024 字节忘了 delete\n");
            (void)frame;                // 它还指着那块没人管的内存
            return false;               // ⚠ 泄漏
        }
        delete[] frame;
        return true;
    }

    // 案子 C：释放后使用（悬垂）——ASan 当场中断；不开 ASan 时就是第 1 章的 0xDD 剧本
    void caseC_useAfterFree() {
        int* hp = new int(100);
        delete hp;
        std::printf("案子C：读取已释放内存 → %d（开 ASan 时上一行就该被工具拦截）\n", *hp);  // ⚠
    }
}  // namespace

int main() {
#if defined(_MSC_VER) && defined(_DEBUG)
    // CRT 调试堆开关：泄漏报告写到 stderr（默认只写给调试器，控制台看不见）
    _CrtSetReportMode(_CRT_WARN, _CRTDBG_MODE_FILE);
    _CrtSetReportFile(_CRT_WARN, _CRTDBG_FILE_STDERR);
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
#endif
    caseA_clean();

#if defined(_MSC_VER) && defined(_DEBUG)
    {
        // 区间检测：只查「这段代码」漏没漏——正好对上「中途提前 return」场景
        _CrtMemState before = {}, after = {}, diff = {};
        _CrtMemCheckpoint(&before);                 // 拍一张分配现场快照
        (void)caseB_earlyReturn(false);
        _CrtMemCheckpoint(&after);
        if (_CrtMemDifference(&diff, &before, &after) != 0) {
            std::printf("案子B：CRT 区间检测到 %zu 次分配未释放、共 %zu 字节\n",
                        diff.lCounts[_NORMAL_BLOCK], diff.lSizes[_NORMAL_BLOCK]);
            _CrtMemDumpAllObjectsSince(&before);    // 把未释放对象逐个列出来
        }
    }
#else
    (void)caseB_earlyReturn(false);
#endif

    caseC_useAfterFree();
    std::printf("main 结束（退出时的泄漏总报告由工具给出）\n");
    return 0;
}
```

**不开任何工具**（Release CRT）的实测输出——注意案子 C 每次跑出的垃圾值都不同，这就是「没崩 ≠ 没事」：

```text
案子A：干净路径跑完
案子B：提前 return——frame 的 1024 字节忘了 delete
案子C：读取已释放内存 → -1820433440（开 ASan 时上一行就该被工具拦截）
main 结束（退出时的泄漏总报告由工具给出）
```

**MSVC Debug（/MDd）+ CRT 调试堆**的实测输出——区间检测精确到 1 次 1024 字节；案子 C 读出的 `-572662307` 正是 `0xDDDDDDDD` 毒化模式的十进制真身（第 1 章 1.11 的理论在这里闭环）；退出时给出带分配序号 `{140}` 的总报告：

```text
案子A：干净路径跑完
案子B：提前 return——frame 的 1024 字节忘了 delete
案子B：CRT 区间检测到 1 次分配未释放、共 1024 字节
案子C：读取已释放内存 → -572662307（开 ASan 时上一行就该被工具拦截）
main 结束（退出时的泄漏总报告由工具给出）
Detected memory leaks!
Dumping objects ->
{140} normal block at 0x0000026978F384F0, 1024 bytes long.
 Data: <                > CD CD CD CD CD CD CD CD CD CD CD CD CD CD CD CD
Object dump complete.
```

**clang + ASan** 的实测报告（节选）——案发即中断，分配与释放两侧调用栈俱全，直接定位到源文件行号：

```text
==12532==ERROR: AddressSanitizer: heap-use-after-free on address 0x1295588200d0 ...
READ of size 4 at 0x1295588200d0 thread T0
    #0 ... in caseC_useAfterFree D:\tmp\ch02lab\ch02_leak_lab.cpp:41
    #1 ... in main D:\tmp\ch02lab\ch02_leak_lab.cpp:71
freed by thread T0 here:
    #1 ... in caseC_useAfterFree D:\tmp\ch02lab\ch02_leak_lab.cpp:40
previously allocated by thread T0 here:
    #1 ... in caseC_useAfterFree D:\tmp\ch02lab\ch02_leak_lab.cpp:39
```

实践任务 2 会要求你把这套流程跑成肌肉记忆：**改完资源管理代码 → 开仪器跑一遍提前 return 路径 → 零报告才算完**。

---

## 深入专题：make_shared 的控制块——一次分配的内存布局解剖

2.5 节说 `make_shared` 优先，理由是「一次分配 + 缓存局部性」；2.6 节又说 weak_ptr「看着不续命」。这两件事在内存层面交汇成一个反直觉的事实：**对象的析构和对象内存的释放是两个独立时刻**。本专题用一台「分配计数显微镜」把它看穿。

### 控制块里装着什么：两种布局的内存图

`shared_ptr<T>(new T)` 与 `make_shared<T>()` 功能等价，内存布局截然不同：

```text
写法一：shared_ptr<T>(new T)  ——  两次分配，两处住
  ┌─────────────────┐        ┌──────────────────────────┐
  │ T 对象           │        │ 控制块                    │
  │  (你的数据)      │◀──ptr──│ 强计数 │ 弱计数 │ deleter │
  └─────────────────┘        └──────────────────────────┘
  分配 #1（new T）              分配 #2（控制块）
  强计数归零：T 析构 + #1 释放
  弱计数归零：#2 释放

写法二：make_shared<T>()  ——  一次分配，一块住
  ┌───────────────────────────────────┐
  │ 控制块            │  T 对象         │
  │ 强计数 │ 弱计数 … │  (紧挨控制块)    │
  └───────────────────────────────────┘
  分配 #1（合并块：控制块 + 对象）
  强计数归零：T 析构（内存不还！弱计数还钉着）
  弱计数归零：整块 #1 释放
```

`make_shared` 的红利一目了然：省一次堆分配（分配是有成本的，成本细账第 6 章展开），且对象与控制块相邻，拷贝 `shared_ptr` 要摸的控制块大概率还在缓存里——缓存局部性红利。

### 计数实验：分配与释放的完整对照

我们重载全局 `operator new/delete` 当计数器，对两种写法各数一遍「分配几次、释放几次」，并安排一个 `weak_ptr` 观察者：

```cpp
// ch02_make_shared_lab.cpp —— 深入专题实验：数一数分配，看对象死与内存放的时机（完整可编译）
// clang++ -std=c++20 -Wall -Wextra ch02_make_shared_lab.cpp -o ch02_make_shared_lab
#include <cstdio>
#include <cstdlib>
#include <memory>
#include <new>
#include <string>
#include <utility>

// ---- 全局分配计数器：重载全局 operator new/delete，数「分配几次、释放几次」----
// 注意：计数器内绝不 printf（printf 内部可能分配内存，会造成递归）。
namespace allocwatch {
    inline int& allocs() { static int n = 0; return n; }
    inline int& frees()  { static int n = 0; return n; }
}  // namespace allocwatch

void* operator new(std::size_t n) {
    allocwatch::allocs()++;
    return std::malloc(n);
}
void operator delete(void* p) noexcept { allocwatch::frees()++; std::free(p); }
void operator delete(void* p, std::size_t) noexcept { allocwatch::frees()++; std::free(p); }

class BossAlien {
public:
    explicit BossAlien(std::string tag) : tag_(std::move(tag)) {
        std::printf("    BossAlien 构造\n");
    }
    ~BossAlien() { std::printf("    BossAlien 析构\n"); }
    std::string tag_;
};

static void report(const char* when, int a0, int f0) {
    std::printf("    [%s] 相对基线：分配 %d 次 / 释放 %d 次\n",
                when, allocwatch::allocs() - a0, allocwatch::frees() - f0);
}

int main() {
    // 预热：让 stdio 建好缓冲区，避免它污染计数（首个 printf 内部可能分配一次）
    std::printf("== 深入专题实验：shared_ptr<T>(new T) vs make_shared<T> ==\n");

    std::printf("== A. 写法一：shared_ptr<BossAlien>(new BossAlien)——对象与控制块分两处住 ==\n");
    {
        int a0 = allocwatch::allocs(), f0 = allocwatch::frees();
        {
            std::weak_ptr<BossAlien> spy;                 // 模拟「缓存里还惦记着它」的观察者
            {
                std::shared_ptr<BossAlien> p(new BossAlien("A"));
                report("构造后", a0, f0);                 // 期望：分配 2（对象 + 控制块）
                spy = p;
                p.reset();                                // 最后一个强属主放手
                std::printf("    强属主放手：对象析构（见上），但控制块还活着\n");
                report("放手后", a0, f0);                 // 期望：释放 1（对象本体）
            }
            std::printf("    现在 spy 离开作用域：控制块内存这时才归还\n");
        }   // ← spy 在这里析构
        report("观察者离场后", a0, f0);                   // 期望：释放 2（控制块）
    }

    std::printf("== B. 写法二：make_shared<BossAlien>——一块内存把对象和控制块一起住 ==\n");
    {
        int a0 = allocwatch::allocs(), f0 = allocwatch::frees();
        {
            std::weak_ptr<BossAlien> spy;
            {
                std::shared_ptr<BossAlien> p = std::make_shared<BossAlien>("B");
                report("构造后", a0, f0);                 // 期望：分配 1（合并块）
                spy = p;
                p.reset();
                std::printf("    强属主放手：对象析构了，但内存【没有】归还——weak 还钉着这块内存\n");
                report("放手后", a0, f0);                 // 期望：释放 0！
            }
            std::printf("    现在 spy 离开作用域：合并块（对象内存+控制块）一起归还\n");
        }   // ← spy 在这里析构
        report("观察者离场后", a0, f0);                   // 期望：释放 1
    }
    std::printf("== 结论：对象析构 ≠ 内存释放；make_shared 省一次分配，代价是 weak 能钉住大对象 ==\n");
    return 0;
}
```

实测输出（MSVC STL，与上图的预测逐行吻合）：

```text
== 深入专题实验：shared_ptr<T>(new T) vs make_shared<T> ==
== A. 写法一：shared_ptr<BossAlien>(new BossAlien)——对象与控制块分两处住 ==
    BossAlien 构造
    [构造后] 相对基线：分配 2 次 / 释放 0 次
    BossAlien 析构
    强属主放手：对象析构（见上），但控制块还活着
    [放手后] 相对基线：分配 2 次 / 释放 1 次
    现在 spy 离开作用域：控制块内存这时才归还
    [观察者离场后] 相对基线：分配 2 次 / 释放 2 次
== B. 写法二：make_shared<BossAlien>——一块内存把对象和控制块一起住 ==
    BossAlien 构造
    [构造后] 相对基线：分配 1 次 / 释放 0 次
    BossAlien 析构
    强属主放手：对象析构了，但内存【没有】归还——weak 还钉着这块内存
    [放手后] 相对基线：分配 1 次 / 释放 0 次
    现在 spy 离开作用域：合并块（对象内存+控制块）一起归还
    [观察者离场后] 相对基线：分配 1 次 / 释放 1 次
== 结论：对象析构 ≠ 内存释放；make_shared 省一次分配，代价是 weak 能钉住大对象 ==
```

把两段生命周期整理成表（这张表值得默写）：

| 事件 | `shared_ptr<T>(new T)` | `make_shared<T>()` |
| --- | --- | --- |
| 构造 | 分配 2 次（对象 + 控制块） | 分配 1 次（合并块） |
| 强计数归零 | T 析构，对象内存**立即**释放 | T 析构，内存**不释放** |
| 弱计数归零 | 控制块释放 | 整个合并块释放 |

### 工程边界：make_shared 不是免费午餐

1. **weak 钉住大对象**。B 段实验已经演示：只要还有一个 `weak_ptr` 活着（哪怕它只是「想看看对象还在不在」），合并块——包括对象本身的全部内存——就一天不还。给一张 4K 纹理（几十 MB）用 `make_shared`，而缓存/观察者系统里躺着几个长命 `weak_ptr`，就等于给这张大内存上了无期徒刑。**大对象用 `shared_ptr<T>(new T)` 拆开住，或改用 `unique_ptr`，是真实的工程抉择**，不是抠门。
2. **不能指定 deleter**。`make_shared` 没有带 deleter 的重载——需要自定义 deleter 时只能 `shared_ptr<T>(raw, d)`（2.5 节的 `FILE*` 例子因此没法改写成 make_shared）。
3. **私有构造函数不受用**。`make_shared` 需要在库内部 `new` 你的对象，私有构造函数会让它编译失败（工厂模式常用私有构造），需要 friend 或「passkey」技巧绕过——知道有这回事即可。
4. **C++17/20 差异标注**：老教科书说「必须用 `make_shared`，否则 `f(shared_ptr<T>(new T), g())` 若 `g()` 抛异常就泄漏」——这条在 **C++17 起已失效**：语言收紧了函数实参的求值顺序规则，同一个表达式中「分配」与「另一参数求值」不再可能交错。今天坚持 `make_shared` 的理由只剩两条干净的：少一次分配、缓存局部性。C++17 之前的引擎维护旧代码时，那条历史教训依然有效。

---

## 实践任务

四个任务对齐阶段 2 的实践要求。按 E3 约定：原版任务要求「把第 3 节渲染器的 GL 资源（Texture/Shader/Mesh/FBO）重构为 RAII 包装」，若你的渲染器（P1）尚未启动，**直接在 2.4 节的教学桩上完成同款重构**，验证手段与检验标准完全一致；P1 启动后把包装类搬过去即可（可选衔接，见任务末尾）。

### 任务 1：GL 资源全家桶 RAII 化——消灭散落的裸 glDelete

以 2.4 节的教学桩为地基，扩桩并补全包装：

1. **扩桩**（照抄 `glCreateTextures` 的实现模式）：`glCreateBuffers/glDeleteBuffers`、`glCreateVertexArrays/glDeleteVertexArrays`、`glCreateRenderbuffers/glDeleteRenderbuffers`（形状一）；桩的登记表与 `report()` 原样复用。
2. **包装 Texture 与 FBO**（写法 A：`unique_ptr` + 无状态 deleter）、包装 Shader（写法 B 句柄类，2.4 已给出可参照实现）。
3. **实现 `Mesh` 聚合类**：一个 `Mesh` 拥有一个 VAO + 一个 VBO + 一个 EBO 三个 GL 对象（成员用任务 2 的包装类型），拷贝删除、移动放行（`noexcept`）、析构自动清空三个子资源——验证「聚合资源的属主只有一个」。
4. **验收要点**：
   - `grep -n "glDelete" *.cpp` 只命中桩实现与包装类内部，业务代码零命中（检验标准 ①）；
   - 刻意写一行「把 `Mesh` 拷贝进两个容器」的代码 → **编译失败**（类型系统拦截第二属主），截图注释后删除；
   - 桩登记表对账：任意路径跑完 `report()` 为空；
   - 把 2.4 示例里那个手动管理的 `fbo` 也收编进 RAII，`loadEffect` 三条路径全部零泄漏。

### 任务 2：中途提前 return / 异常路径零泄漏——用仪器说话

1. 在任务 1 的代码里构造两条路径：`(a)` 加载中途提前 `return`；`(b)` 构造函数抛异常（桩可加一个「下一次创建必失败」的开关，模拟显存耗尽）。
2. **ASan 路径**：`cl /utf-8 /std:c++20 /EHsc /fsanitize=address 任务1.cpp`（VS 里：项目属性 → C/C++ → 常规 → 启用地址清除程序）。跑通全流程，确认没有 use-after-free / double-free 中断（检验标准 ② 的一半——Windows 版 ASan 不查堆泄漏，另一半靠下面）。
3. **CRT 调试堆路径**：Debug（`/MDd`）编译，按 2.11 节三件套接入；重点用**区间检测**（`_CrtMemCheckpoint` / `_CrtMemDifference`）包住提前 return 路径，验收输出「区间零增量」。
4. **验收要点**：两条路径、两种工具的报告文本各存档一份（全空/零增量）；再故意注释掉一个包装类的析构释放语句复跑，亲眼看到报告出现——**见过抓到长的样子，才认得出没抓到**。

### 任务 3：现场为 Vulkan 风格 create/destroy C API 设计 RAII 包装

不查资料，现场给下面这对真实形状的 API 写包装（检验标准 ③）。先造个两行的桩，再写包装类：

```cpp
// Vulkan 风格签名（示意桩）：句柄是不透明指针，create 返回 VkResult 错误码，destroy 显式传 device
using VkDevice = struct VkDevice_T*;
using VkShaderModule = struct VkShaderModule_T*;
constexpr int VK_SUCCESS = 0, VK_ERROR_OUT_OF_DEVICE_MEMORY = -4;
VkResult vkCreateShaderModule(VkDevice device, const void* pCreateInfo,
                              const void* pAllocator, VkShaderModule* pShaderModule);
void vkDestroyShaderModule(VkDevice device, VkShaderModule shaderModule, const void* pAllocator);
```

设计题要点（先想再写）：它和 GL 的三点不同——句柄是不透明指针而非整数；失败通过**返回值**报（不写回 0）；destroy 需要 `device` 与 `pAllocator` 两个额外参数。你的 deleter 必须「记住」device/allocator——**有状态 deleter**。请各给一版：`unique_ptr` + 有状态 deleter（观察 `sizeof` 变大了几字节、类型签名长什么样），以及句柄类（构造函数收 `device`，成员存下来供析构用）。验收要点：创建失败（`VK_SUCCESS` 以外）不产生半成品对象；两种写法都通过「拷贝必须编译失败」检查；能口述「有状态 deleter 进类型签名」带来的类型膨胀问题与句柄类如何回避。

### 任务 4：抓一个「shared_ptr 是偷懒」的反例并改造

写（或从旧代码里挑）一个这样的场景：`Renderer` 独占持有若干 `Mesh`，`ParticleSystem` 独占一份粒子纹理——生命周期清晰唯一，却通篇 `shared_ptr`（检验标准 ④）。改造为 `unique_ptr`，并写三行注释回答：改之前「谁能删它」说得清吗？控制块/原子计数/两次间接白花了多少？这段代码有没有潜力演化成循环引用？验收要点：改造后全工程编译通过、行为不变；画一张「谁拥有（unique/shared/值）、谁借用（引用/裸指针/view）」的两栏清单，覆盖这些类型。

**（可选）P1 衔接点**：第 3 节渲染器启动后，把任务 1–3 的包装类原样搬入，桩调用替换为 glad 的真实 GL 调用，登记表换成了如 RenderDoc 的资源列表核对——「每个 GL 对象恰有一个 RAII 属主、全代码检索不到裸 glDelete」就是渲染器「RAII 化」里程碑的验收线。

---

## 自测题

先自己作答（口头或写在纸上），再对照章末参考答案。第 1–4 题对应阶段 2 的四条检验标准，务必都能独立完成。

**第 1 题（达成路径，检验标准 ①）**：接手一个散落着裸 `glCreateTextures`/`glDeleteTextures` 的渲染器。请给出把工程推进到「每个 GL 对象恰有一个 RAII 属主、全代码检索不到裸 glDelete」的具体路径：包装形态怎么选？如何从类型系统上杜绝「一个句柄两个属主」？如何分三级自检（类型级/检索级/运行级）验证达成？

**第 2 题（验证方法，检验标准 ②）**：你刚把一个加载函数改成一堆提前 return 的结构。如何证明这条路径零泄漏？给出至少两条相互独立的验证手段（工具名 + 操作要点 + 报告看什么），并回答：为什么「析构日志条数配平」不足以作为最终证据？

**第 3 题（现场设计，检验标准 ③）**：给出 Vulkan 风格签名 `VkResult vkCreateShaderModule(VkDevice, const void* pCreateInfo, const void* pAllocator, VkShaderModule* out)` 与 `void vkDestroyShaderModule(VkDevice, VkShaderModule, const void* pAllocator)`。口述（或写出骨架）：包装成 `unique_ptr` 时 deleter 需要携带什么状态？这会怎样影响类型签名与大小？改用句柄类时怎么组织？创建失败如何表达？

**第 4 题（偷懒反例，检验标准 ④）**：举出一个「所有权本可唯一却用了 `shared_ptr`」的真实场景，说明它的三宗罪（所有权叙事、运行期成本、演化风险），并给出 `unique_ptr` + 借用参数的改造形状。

**第 5 题（循环引用）**：下面代码退出 `main` 时两个析构日志都不出现。指出环在哪里、为什么计数不归零、给出最小修复（改哪个成员、为什么是它）：

```cpp
class Camera;   // 已有完整定义，析构打日志
class RenderTarget {
public:
    std::shared_ptr<Camera> viewCamera;    // 渲染目标记着「谁在看」
    ~RenderTarget() { std::puts("~RenderTarget"); }
};
class Camera {
public:
    std::vector<std::shared_ptr<RenderTarget>> targets;   // 相机拥有它画的目标
    ~Camera() { std::puts("~Camera"); }
};
// main：auto cam = make_shared<Camera>(); auto rt = make_shared<RenderTarget>();
//       cam->targets.push_back(rt); rt->viewCamera = cam;
```

**第 6 题（deleter 差异）**：同样自定义 deleter，`unique_ptr<T, D>` 与 `shared_ptr<T>` 在**类型系统与内存**上各如何处理它？至少答出：写在签名里还是构造时传入、有状态 deleter 对 `sizeof` 的影响、两种 deleter 不同的对象能否混装同一容器，各附一句场景。

**第 7 题（原子性 ≠ 线程安全）**：同事说「`shared_ptr` 的引用计数是原子的，所以我可以放心在多线程里用它指向的配置对象」。指出这句话里的错误，并给每条错误配一个最小修正。追问：两个线程分别持有指向同一对象的不同 `shared_ptr` 实例，一边析构、一边拷贝，安全吗？那同一个 `shared_ptr` 变量呢？

**第 8 题（shared_from_this 与捕获 this）**：(a) 在 `Effect` 的成员函数里写 `std::shared_ptr<Effect>(this)` 交给异步系统，后果是什么？运行期什么时候爆？正确姿势与前提条件？(b) 一段 UI 代码把 `[this]` 的 lambda 登记到事件总线，界面关闭后总线还在派发——症状与修复各是什么？

**第 9 题（make_shared 与控制块）**：`make_shared<T>()` 与 `shared_ptr<T>(new T)` 至少说出三个差异（分配次数、缓存局部性、可否指定 deleter 任选三方面）。追问：`make_shared` 出来的对象、全部 `shared_ptr` 已销毁、但还有一个 `weak_ptr` 活着——此时对象析构了吗？内存释放了吗？换成 `shared_ptr<T>(new T)` 呢？

---

## 常见误区

**误区一：万物 `shared_ptr`。** 症状：拿不准所有权就上 `shared_ptr`，工程里一半的堆对象是共享的。它有两个子误区，都得点名：
- **子误区 A（所有权含糊化 + 循环引用）**：既然人人持有，就没人说得清谁负责生死；对象之间的关系悄悄成环（互指、lambda 捕 `shared_ptr`、观察者注册、缓存回填都可能成环），析构永不来，泄漏无声无息。纠偏：默认 `unique_ptr`；确有共享才 `shared_ptr`；回指/观察一律 `weak_ptr`；给可疑类挂析构日志盯梢。
- **子误区 B（误把引用计数线程安全当对象线程安全）**：「计数是原子的」被读成「随便哪个线程都能并发读写对象成员」——数据竞争是未定义行为，不是「偶尔丢更新」（2.7 节实测三次三个数）。纠偏：`shared_ptr` 只保证「记账不出错」，对象同步靠第 5 章的手段。

**误区二：把裸句柄塞进多个容器/包装共用。** 症状：`GLuint tex;` 创建后既存进 `Texture` 类、又抄一份进渲染列表、再抄一份进 UI 引用表——三处都是「值拷贝的句柄」，一处析构 `glDeleteTextures`，另外两处全成悬垂句柄，下一帧绑定或重复删除都是未定义行为。纠偏：裸句柄只活在属主对象内部；别人要引用，借 `Texture&`/`const Texture*`（或句柄对象的指针），**永远不借裸 `GLuint` 出门**。

**误区三：lambda 捕获 `this` 的异步回调。** 症状：回调注册时一切正常，对象销毁后某一帧崩溃或读到垃圾——崩溃点离注册处隔着一整个事件循环，栈上根本找不到凶手。纠偏：异步回调默认「捕获 `weak_from_this()`，执行时 `lock()`」（2.8 节）；捕获 `[&]` 引用局部变量的回调同理（局部变量活不过当前函数）。

**误区四：`release` / `reset` / `get` 混用三连。** 症状：以为 `release()` 会释放（它只交所有权）；`get()` 出来的裸指针存进成员长期使用（借出去的又想拥有）；对 `get()` 的返回值手动 `delete`（属主析构时双重释放）。纠偏：`release` 只在向 C API 移交所有权的边界出现；`get` 的返回值只在「当下这个调用」内有效；删除永远是属主（智能指针/容器/包装类）的事。

**误区五：对智能指针的成本没有账。** 症状：或以为 `unique_ptr` 有运行开销（无状态 deleter 时零开销，`sizeof` 就是裸指针），或以为 `shared_ptr` 只是「略贵」（两个指针 + 控制块一次分配 + 每次拷贝析构的原子操作 + 访问一跳间接 + 引入循环引用风险）。纠偏：用 2.3/2.5 的 `sizeof` 实验建立量级感；热路径（每帧万次）上的共享计数要过脑子——量化的账第 6 章算。

**误区六：只在「明显互指」时防循环引用。** 症状：代码里搜不到两个类互相声明 `shared_ptr` 成员，就以为没有环——实际环藏在 lambda 捕获（回调闭包持有 `shared_ptr`，对象又持有回调列表）、事件订阅表、对象-缓存互持里。纠偏：环是一个**运行期可达的引用图**问题，不是成员声明问题；自查靠析构日志 + `use_count` 不归零的信号。

**误区七：全局/静态对象的析构里依赖别的全局。** 症状：`main` 正常返回后、进程退出前崩溃，或日志/存档在最后关头写坏——静态析构段踩了已析构的邻居（2.10 节）。纠偏：显式 `Init/Shutdown` 序列管资源；全局对象析构只做无依赖收尾；单例用函数局部静态。

**误区八：只查内存泄漏，不查句柄泄漏；只信一种工具。** 症状：ASan 全绿就宣布「零泄漏」——但 GL 对象漏在驱动侧，你的堆上只有 4 字节编号，ASan（Windows 版甚至不查堆泄漏）和 CRT 调试堆都看不见。纠偏：证据链三层——ASan 抓用错、CRT 调试堆抓堆泄漏、登记表/RenderDoc 抓句柄（2.11 节分工表），三层全绿才算零泄漏。

---

## 延伸资源

**learncpp.com**（免费、持续更新，本阶段配合主教材查漏；章节编号以站内目录为准）：

- 「Resource Management」相关章节（RAII 与 `std::unique_ptr`/`std::shared_ptr`/`std::weak_ptr` 的系统讲解，与本章 2.1–2.6 互为印证）；
- 「std::span」与「std::string_view」小节——2.9 节借用词汇类型的细则；
- 「Move semantics and smart pointers」专题下关于按值传递智能指针与借用传参的讨论——对应 2.3 的传参纪律。

**《Effective Modern C++》（Scott Meyers）**，与本章直接相关的条目：

- 条目 18「让 std::unique_ptr 成为独占资源的默认选择」——2.3 节的完整论证；
- 条目 19「对共享资源使用 std::shared_ptr」——控制块与计数语义；
- 条目 20「对于类似 shared_ptr 但有可能空悬的场合使用 weak_ptr」——2.6 节；
- 条目 21「优先选用 std::make_unique 和 std::make_shared，而非直接 new」——深入专题的出处（注意其中异常安全论证基于 C++11/14 求值顺序，C++17 后读法见本章标注）；
- 条目 22「使用 Pimpl 惯用法时把特殊成员函数定义放实现文件」——句柄类/`unique_ptr` 组合的工程进阶；
- 条目 31「避免使用默认捕获模式」、条目 32「使用初始化捕获将对象移入闭包」——2.8 节 lambda 捕获坑的权威展开（init capture 属第 3 章剧透，可先读一半）。

**cppreference（全程工具书，语义疑问先查它）**：

- <https://en.cppreference.com/w/cpp/memory/unique_ptr>　｜　<https://en.cppreference.com/w/cpp/memory/shared_ptr>　｜　<https://en.cppreference.com/w/cpp/memory/weak_ptr>
- <https://en.cppreference.com/w/cpp/memory/enable_shared_from_this>　｜　<https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared>
- RAII 概览：<https://en.cppreference.com/w/cpp/language/raii>
- deleter 与分配计数：自定义 deleter 页 <https://en.cppreference.com/w/cpp/memory/shared_ptr/shared_ptr>（重载说明）；全局 `operator new/delete` <https://en.cppreference.com/w/cpp/memory/new/operator_new>
- 借用词汇类型：<https://en.cppreference.com/w/cpp/container/span>　｜　<https://en.cppreference.com/w/cpp/string/basic_string_view>
- 工具：AddressSanitizer（MSVC 文档「/fsanitize=address」）与 CRT 调试堆（MSVC 文档「CRT 调试堆详细信息」）

**CppCon 讲座**：

- CppCon 2019, Chandler Carruth, *There Are No Zero-Cost Abstractions*——对「零开销」保持理性：抽象有价，选对地方付（阶段 2 指定讲座）；
- CppCon 2017, Kostya Serebryany, *Code Sanitization in C/C++*——ASan/TSan/MSan 一家子的原理与用法（TSan 部分第 5 章再回看）。

---

## 参考答案

### 自测题

**第 1 题**。参考路径：① 盘点资源与配对函数（`glCreateXxx`/`glDeleteXxx` 每对登记在册）；② 按域语义选包装形态——无附加语义的句柄用 `unique_ptr` + 无状态 deleter（2.4 写法 A），有域语义（label、参数、接口）的写句柄类（写法 B），聚合资源（Mesh = VAO+VBO+EBO）写一个聚合类持多个子包装，保证聚合体是唯一属主；③ 用类型系统杜绝第二属主：包装的拷贝构造/赋值 `= delete`，移动放行且 `noexcept`——「把句柄拷贝给别人」从能编译变成编译错误；裸 `GLuint` 只允许存在于包装类内部和「创建后交给属主的那一行」；④ 三级自检：类型级（把包装类塞进两个容器/拷贝 → 编译失败）、检索级（`grep -rn "glDelete"`，只允许命中包装类与 C API 桩实现，业务代码零命中）、运行级（桩登记表 `report()` 对账为空 + ASan/CRT 仪器报告零异常）。达成「全代码检索不到裸 glDelete」即阶段 2 检验标准 ①。

**第 2 题**。至少两条独立证据链：① **CRT 调试堆区间检测**：Debug（/MDd）编译，`_CrtMemCheckpoint` 包住提前 return 路径前后，`_CrtMemDifference` 为 0（零次未释放分配）即通过；全局层面再加 `_CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF)` 看退出总报告无 `Detected memory leaks!`。② **ASan**：`/fsanitize=address`（或 clang `-fsanitize=address`）跑全流程，确认无 heap-use-after-free / double-free 中断——它管「用错」不保证报「漏」（Windows 版不查泄漏，这是实测结论，所以必须与 ① 互补）。③ 补充运行级证据：GL 桩登记表对账为空。为什么日志配平不够：日志只证明「析构函数跑了几次」，不证明「跑的那次真的释放了资源」（析构里可能早放了/放漏了），也不覆盖驱动侧与第三方库的分配；仪器直接对堆/分配器记账，且能抓到「析构根本没跑」的路径——日志是旁证，仪器是实证。

**第 3 题**。参考骨架（思路版）：deleter 需要**捕获 `device`（和 allocator）**——这是有状态 deleter。`unique_ptr` 版：

```cpp
// 节选：设计要点示意
struct ShaderModuleDeleter {
    VkDevice device;
    const void* allocator;
    void operator()(VkShaderModule m) const noexcept {
        if (m) vkDestroyShaderModule(device, m, allocator);
    }
};
using ShaderModulePtr = std::unique_ptr<
    std::remove_pointer_t<VkShaderModule>, ShaderModuleDeleter>;
// 创建：VkShaderModule h; if (vkCreateShaderModule(dev, &ci, nullptr, &h) != VK_SUCCESS)
//                          return {};                 // 失败：不产生半成品对象
//       return ShaderModulePtr(h, ShaderModuleDeleter{dev, nullptr});
```

要点：① 有状态 deleter **进入类型签名**——`ShaderModulePtr` 携带 `device` 状态，每个实例多存两个指针（`sizeof` 从 8 变 24）；不同 device 的指针类型相同（状态在对象里不在类型里），但语义上仍不该混用，可用句柄类更强约束。② 句柄类版：构造函数收 `device` 存为成员，拷贝删除、移动放行、析构用存的 `device` 调 destroy——回避了 deleter 状态进类型的问题，接口也更 domain 化。③ 失败表达：`vkCreateShaderModule` 返回非 `VK_SUCCESS` 时不构造包装（返回空 `unique_ptr` 或抛异常/返回 `expected`——错误处理策略取舍第 7 章展开）；要点是**失败路径上不存在需要销毁的半成品**。答出「deleter 有状态 → 进 unique_ptr 签名 → 大小膨胀 → 句柄类回避」这一主线即达标（检验标准 ③）。

**第 4 题**。参考反例：`Renderer` 独占所有 `Mesh`（加载、卸载、绘制全由它做），`ParticleSystem` 独占一份粒子纹理——除了属主没有任何人有理由决定它们的生死，却写成 `std::vector<std::shared_ptr<Mesh>>` + `shared_ptr<const Texture> particlesTex_`。三宗罪：① 所有权叙事崩坏——读者无法从类型判断「谁能删它」，最后一块拼图靠口口相传；② 白花销——控制块堆分配、每次传递的原子加减、`shared_ptr` 双指针、访问经控制块间接，换不来任何「共享」的实益；③ 演化风险——`shared_ptr` 人人可持，下一个需求「把纹理借给 UI」就顺手再发一个 `shared_ptr`，环与悬垂的地基就此打下。改造：属主侧 `std::vector<std::unique_ptr<Mesh>>`；借用侧一律 `const Mesh&` / `const Mesh*` / `std::span`（绘制函数收借用参数）。验收：改造后行为不变、编译通过；「谁拥有/谁借用」清单能覆盖全部涉及类型（检验标准 ④）。

**第 5 题**。环在 `RenderTarget::viewCamera`（`shared_ptr<Camera>`）与 `Camera::targets`（`shared_ptr<RenderTarget>`）之间：`cam` 强计数 = 1（main 的 `shared_ptr`）+ 1（`rt->viewCamera`）= 2；`rt` 强计数同理 = 2。退出 `main` 时各自 −1 剩 1，永不归零——两个析构都不会发生。最小修复：把回指改观察——`std::weak_ptr<Camera> viewCamera;`，使用处 `if (auto cam = viewCamera.lock()) ...`（渲染前本来就该检查相机还活着）。为什么是它而不是 `targets`：「相机拥有它画的目标」是合理的所有权方向（相机决定画什么），「渲染目标拥有相机」不是——修复应剪断不合理的方向，保留拥有链。若两个方向都「该拥有」，那才需要重新审视设计（通常说明职责切分有误）。

**第 6 题**。`unique_ptr<T, D>`：deleter 是模板参数，**写在类型签名里**；`unique_ptr<T, D1>` 与 `unique_ptr<T, D2>` 是不同类型、不可互换；无状态 D 时 `sizeof` 不变（空基类优化，= 裸指针），有状态 D 时每实例多存一份状态（函数指针版 +8 字节）。适合「释放方式是类型固有属性」的场景（如 GL 句柄包装），让编译器拦下「用错释放函数」。`shared_ptr<T>`：deleter 是**构造函数实参**，被**类型擦除**存进控制块；类型永远是 `shared_ptr<T>`，大小恒为两个指针；不同 deleter 的实例可以混装同一容器（都是 `shared_ptr<FILE>`），代价是控制块多存一个（可能堆分配的）deleter 且调用多一跳间接。适合「同一类型不同释放法」的场景（同一容器里装 `fclose` 关的文件和 `CloseHandle` 关的句柄——先包成同一签名再装）。一句话：`unique_ptr` 用类型换零开销与强约束，`shared_ptr` 用擦除换统一类型与灵活性。

**第 7 题**。错误一：把「控制块计数原子」当成「对象线程安全」——`shared_ptr` 不给 `T` 的成员加任何保护，并发写普通成员是数据竞争（未定义行为）；修正：共享可变状态用 `std::mutex`/`atomic`（第 5 章），或重构为每线程独享/只读共享。错误二（隐含）：认为「随便从哪个线程拷贝同一个 `shared_ptr` 变量」安全——对**同一个实例**的并发读写仍是竞争；修正：各线程持各自的 `shared_ptr` 副本（在单线程边界上分发），或用锁/`std::atomic<std::shared_ptr<T>>`（C++20）保护该变量。追问：不同实例（各自是独立 `shared_ptr` 对象、指向同一控制块）一边析构一边拷贝——**安全**，这正是原子计数的保证范围；同一个 `shared_ptr` 变量——**不安全**，竞争的是这个变量本身。

**第 8 题**。(a) 后果：`shared_ptr<Effect>(this)` 给对象开出**第二张控制块**（强计数从 1 开始的新账本）——异步系统那本账销毁对象时，原属主账本毫不知情，随后原属主析构即**双重释放**；也可能当下就因两本账「各自复活」而延迟崩溃，时序难复现。正确姿势：`class Effect : public std::enable_shared_from_this<Effect>`，成员函数里 `shared_from_this()`（或交出弱引用用 `weak_from_this()`）；前提：**对象必须已被某个 `shared_ptr` 接管**（通常刚 `make_shared`），否则 C++17 起 `shared_from_this()` 抛 `std::bad_weak_ptr`。(b) 症状：界面关闭后事件总线每次派发都撞悬垂 `this`——轻则读到垃圾数据（数据坏了程序没崩，比崩更危险），重则访问冲突崩溃，且崩溃点在总线派发处、栈上找不到凶手。修复：lambda 改捕 `std::weak_ptr<UI>`（`weak_from_this()`），派发时 `lock()` 失败即跳过；或由 UI 的析构（RAII 反注册）从总线注销——前者容忍「来晚的回调」，后者要求「不死就一定在场」，按语义选。

**第 9 题**。三个差异：① 分配次数——`shared_ptr<T>(new T)` 两次（对象 + 控制块），`make_shared` 一次（合并块）；② 缓存局部性——`make_shared` 的对象与控制块相邻，拷贝/判活要摸的控制块大概率在缓存里，拆开住则隔着一次跳转；③ 可否指定 deleter——`make_shared` 无 deleter 重载，需要自定义释放（如 `FILE*` + `fclose`）只能 `shared_ptr<T>(raw, deleter)`（另一差异：`make_shared` 不能调用私有构造函数）。追问：`make_shared` 版——对象**已析构**（强计数归零立刻析构），但内存**未释放**（对象与控制块同住一块，弱计数还钉着整块）；`shared_ptr<T>(new T)` 版——对象析构且**对象内存已释放**，控制块自己的小块等弱计数归零后才释放。这也是「大对象慎用 `make_shared` + 长命 weak」的依据。

### 实践任务

**任务 1（GL 资源全家桶 RAII 化）**。参考思路：扩桩直接复制 `glCreateTextures` 实现改名字与类型名；`Buffer`/`VertexArray` 包装照抄 2.4 写法 A（换 deleter 里的释放函数即可）；`Shader` 用写法 B 对照实现；`Mesh` 聚合类成员写三个包装对象（`VertexArray vao_; Buffer vbo_; Buffer ebo_;`），**不写任何特殊成员函数**——成员不可拷贝自动让 `Mesh` 不可拷贝、成员可移动自动让 `Mesh` 可移动（这次连「删减版五件套」都不用手写，rule of zero 的复利）。验收要点：见任务描述四条；特别核对 `grep "glDelete"` 输出——业务文件零命中才算达成检验标准 ①；「拷贝 `Mesh` 编译失败」的报错信息里应能看到 `deleted function` 字样（把证据注释留进代码）。

**任务 2（提前 return / 异常零泄漏）**。参考思路：路径 (a) 在加载流程中段按条件 `return false`；路径 (b) 给桩加 `glStubSetNextCreateFails(true)` 之类的故障注入开关，让某个包装的构造函数抛异常。仪器接入照抄 2.11 三件套与 ASan 编译行；CRT 区间检测放在「进入加载 → 提前 return 返回」的外层。验收要点：两条路径 × 两种工具，四份报告文本归档；「故意弄漏再修好」的对照报告各一份；口述结论——ASan 抓用错、CRT 抓堆漏、登记表抓句柄漏（检验标准 ② 达成）。常见坑：CRT 报告默认只写给调试器（记得 `_CrtSetReportMode` 重定向 stderr）；Release 配置下 CRT 调试堆不生效（用 /MDd）。

**任务 3（Vulkan 风格现场设计）**。参考思路与验收要点：先造桩（一个 `std::unordered_map` 记存活 module 即可复用登记表思路），再按参考答案第 3 题的主线实现两版；重点验收三问——创建失败是否零残留（故障注入一测便知）、「拷贝包装」是否编译失败、能否口述「有状态 deleter 使 `unique_ptr` 的 `sizeof` 从 8 → 24 且类型签名携带 device/allocator，句柄类用成员回避但多写三个成员函数」。加分项：给句柄类补 `get()` 并解释「为什么借出裸 `VkShaderModule` 给 vk 函数使用是安全的」（答案：借用窗口内属主活着；与 2.9 借用词汇表同一条规矩）。

**任务 4（shared_ptr 偷懒反例）**。参考思路：反例代码不必长——`Renderer` 持 `vector<shared_ptr<Mesh>>`、`ParticleSystem` 持 `shared_ptr<const Texture>` 就够；改造只有两步：属主侧换 `unique_ptr`、消费侧改借用参数（`const Mesh&`/`const Texture&`/`span`）。验收要点：全工程编译通过、输出逐字节一致；两栏清单（拥有：`unique_ptr`/值/容器；借用：`const&`/裸指针观察者/view）覆盖全部涉及类型；三条注释能对上「叙事/成本/演化」三宗罪（检验标准 ④ 达成）。可选 P1 衔接：渲染器启动后把包装类原样迁移，登记表对账换成 RenderDoc 资源列表核对。

---

*本章完。你已经能把任何 C API 裸资源包进 RAII，也能在「独占 / 共享 / 借用」之间为每个对象选对户口。下一章我们离开资源管理，进入标准库与泛型编程：容器选型、迭代器失效、算法库，以及从「用模板」到「写模板」的那一步——本章借出去的 `span` 与 `string_view`，会在那里正式转正。*
