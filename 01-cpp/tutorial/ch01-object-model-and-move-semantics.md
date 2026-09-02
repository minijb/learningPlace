# 第 1 章｜现代 C++ 基石：对象模型、值语义与移动语义

欢迎来到第 1 章。你已经是一个能独立交付功能的游戏开发者，缺的不是编程能力，而是一层「C++ 机器模型」的底片：对象到底放在哪、活多久、什么时候死、传值时发生了什么。这一章我们把这层底片补上——它决定了你后面读引擎代码、写渲染器、优化帧率时，是「看懂每一行」还是「背语法猜行为」。

---

## 学习目标

对齐本领域阶段 1（现代 C++ 基石）的学习目标。读完本章，你应该能够：

1. **建立底座世界观**：对任何一个 C++ 对象，能回答三个问题——它是什么类型、它的存储在哪（栈/堆/静态区）、它的生命周期从哪开始到哪结束（「对象有类型、有存储、有生命周期」）。
2. **吃透值语义与拷贝/移动**：能顺着一段日志讲清拷贝构造/赋值与移动构造/赋值分别在什么时机被调用、各自做了哪些事；能向同事口述「**std::move 不移动任何东西，真正的移动发生在移动构造函数里**」。
3. **三/五/零法则成为默认取舍**：拿到任何一个类，能立刻判断它该遵循 rule of zero 还是 rule of five，并说出理由。
4. **现代语法基本面与初始化陷阱**：熟练使用 auto、范围 for、结构化绑定、if constexpr、enum class、nullptr；能识别并避开成员初始化顺序陷阱与统一初始化的收窄（narrowing）。
5. **const 正确性从第一天养成**：写出「编译器可验证的只读承诺」。
6. **调试器是基本功**：搭好 Windows + Visual Studio 2022 + `/std:c++20` 的环境，会下断点、单步、监视变量、读调用栈，能独立定位一次悬垂引用类崩溃。

> **环境约定（全章适用）**：Windows + Visual Studio 2022（安装「使用 C++ 的桌面开发」工作负载），语言标准 `/std:c++20`，警告级别 `/W4`。命令行编译在「x64 Native Tools Command Prompt for VS 2022」中执行：
>
> ```bat
> cl /std:c++20 /EHsc /W4 源文件.cpp
> ```
>
> GCC / Clang 等价命令：`g++ -std=c++20 -Wall -Wextra 源文件.cpp` 或 `clang++ -std=c++20 -Wall -Wextra 源文件.cpp`。本章全部「完整示例」均在 C++20 标准下编译运行验证过；若你的引擎还停留在 C++17，本章代码全部适用——涉及 C++17/20 行为差异的地方会显式标注（本章语法本身没有用到 C++20 独有特性）。

---

## 分节正文

### 1.1 从源码到程序：预处理、编译、链接与翻译单元

我们先建立一张「源代码是怎么变成 exe 的」地图。你是写游戏的人，不是写脚本的人，这张图必须长在脑子里，因为后面一半的编译错误和链接错误都靠它定位。

C++ 把一个 `.cpp` 变成可执行文件，粗分为三步：

```
foo.cpp ──① 预处理──▶ foo.i(展开后的文本) ──② 编译──▶ foo.obj ──┐
bar.cpp ──① 预处理──▶ bar.i                  ──② 编译──▶ bar.obj ──┼──③ 链接──▶ 游戏.exe
        ──② 编译──▶ bar.obj                 ◀── 头文件被①文本展开 ─┘
```

**① 预处理（preprocess）**。预处理器是个纯粹的文本工具，只干三件事：把 `#include` 的头文件**逐字粘贴**进来、展开宏、按 `#if/#ifdef` 裁剪代码。注意 `#include` 不是「导入模块」，就是复制粘贴——这也是为什么头文件要有防重复包含的保护：

```cpp
#pragma once          // 现代写法（非标准但全平台支持）
// 或者传统写法：头文件守卫
#ifndef GAME_IMAGE_H
#define GAME_IMAGE_H
// ……头文件内容……
#endif
```

预处理之后得到的完整文本叫一个**翻译单元（translation unit, TU）**——通常说「一个 .cpp 连同它递归包含的所有头文件」。编译器每次只看一个翻译单元，互相不知道对方存在。

**② 编译（compile）**。每个翻译单元被独立编译成一个**目标文件**（MSVC 是 `.obj`，GCC/Clang 是 `.o`）。目标文件里有机器码，还有一张**符号表**：我这个 TU 定义了哪些函数/全局变量（供别人用），又引用了哪些自己没定义的名字（找别人要）。这一步的错误叫编译错误，通常是语法、类型不匹配。

**③ 链接（link）**。链接器把所有目标文件（和库）拼在一起，做符号配对：A.obj 要的符号在 B.obj 里有吗？游戏工程师最该认识的两个链接错误：

- **LNK2019 / unresolved external symbol（无法解析的外部符号）**：声明了但没人定义——比如你声明了 `void tick();` 却没写函数体，或者忘了把某个 .cpp 加进工程。
- **LNK2005 / already defined（符号重复定义）**：同一个名字被定义了两次。典型现场：你把函数**定义**（带函数体）写在了头文件里，而这个头文件被两个 .cpp 包含——两个翻译单元各有一份定义，链接时撞车。

第二类错误正好引出本章第一个「概念级」规则——**ODR（One Definition Rule，单一定义规则）**：

- 同一个非 inline 函数或全局变量，在整个程序里**只能有一个定义**；
- 类的**定义**可以在多个翻译单元里各出现一次，但每一份必须逐 token 相同——这就是类定义放头文件、成员函数定义放 .cpp（或写成类内 inline）的根本原因。

这里我们只区分两个词：**声明**（declaration，告诉编译器「有这么个东西」，不产生代码）与**定义**（definition，给出实体本身）。ODR 管的是定义。

**为什么游戏工程师要在乎这些**：读懂构建报错能少走一半弯路；理解「改一个头文件全工程重编」的代价，会让你从一开始就克制头文件的包含关系（构建系统工程化将在第 4 章展开，这里只需要地图）。想亲眼看编译器各阶段产物的同学，章末延伸资源里 Godbolt 的 CppCon 讲座值得一听。

### 1.2 对象的一生：栈与堆、存储期与析构次序

现在进入正题：对象有类型、有存储、有生命周期。C++ 用**存储期（storage duration）**给每个对象规定生命周期，游戏工程师最常用的是三种：

| 存储期 | 典型形态 | 生命周期 | 你需要操心的事 |
| --- | --- | --- | --- |
| 自动存储期 | 函数里的局部对象 | 从**声明处**构造，到**所在作用域结束**析构 | 基本不用操心——这就是 C++ 的超能力 |
| 静态存储期 | 全局变量、`static` 变量 | 程序启动时（`main` 之前）构造，程序退出时析构 | 构造/析构顺序陷阱（第 2 章展开） |
| 动态存储期 | `new` 出来的对象 | 从 `new` 到配对的 `delete` | 全程靠人——泄漏、悬垂、双重释放都源于此 |

（还有第四种「线程存储期」`thread_local`，并发话题，将在第 5 章展开。）

**栈：自动存储期的家。** 函数调用时，一块「栈帧」被压入调用栈，局部对象就在栈帧里，按**声明顺序**构造；函数返回或作用域退出时，栈帧弹出，对象按**声明顺序的逆序**析构。这个「构造正序、析构逆序」是完全确定、可预测的——记住它，它是 C++ 一切资源管理机制的基石（为什么是基石，将在第 2 章 RAII 一章展开）。

**堆：动态存储期的家。** `new` 在堆上找一块空闲内存并构造对象，`delete` 析构并归还。堆给了你「跨作用域、运行期决定大小」的自由，也把生命周期的管理责任整个压到了你头上。在本章我们先学会「看见」堆对象的一生；管理它的正确姿势（RAII、智能指针）是第 2 章的主题，这里不展开。

看一段完整可编译的程序，把构造/析构次序打印出来：

```cpp
// ch01_scope_order.cpp —— 构造正序、析构逆序（完整可编译）
#include <cstdio>

struct Tracer {
    const char* name;
    explicit Tracer(const char* n) : name(n) { std::printf("构造 %s\n", name); }
    ~Tracer() { std::printf("析构 %s\n", name); }
};

static Tracer g_static("全局对象");   // 静态存储期：main 之前构造

int main() {
    std::printf("--- main 开始 ---\n");
    Tracer frame("帧缓冲");           // ① 先构造，最后析构
    {
        Tracer pass("渲染通道");       // ② 后构造
        Tracer mesh("网格数据");       // ③ 最后构造，最先析构
        std::printf("……一帧的渲染工作……\n");
    }                                  // 退出内层作用域：逆序析构 ③ → ②
    std::printf("--- main 结束前 ---\n");
    return 0;
}                                      // 退出 main：析构 ①；全局对象在 main 之后析构
```

实测输出（逐行对照，验证逆序）：

```text
构造 全局对象
--- main 开始 ---
构造 帧缓冲
构造 渲染通道
构造 网格数据
……一帧的渲染工作……
析构 网格数据
析构 渲染通道
--- main 结束前 ---
析构 帧缓冲
析构 全局对象
```

三个必须内化的观察：

1. **作用域就是生命周期**。`{}` 一对花括号就是一个完整的生灭循环——所以游戏代码里大量临时对象（一个 pass 的中间缓冲、一层循环里的局部状态）写在最小作用域里，是免费的好习惯。
2. **析构严格逆序**。因为后构造的对象可能依赖先构造的对象（先有 `pass` 才有 `mesh`），所以拆家必须从最里面拆起。
3. **循环体也是作用域**。`for` 每一轮迭代里声明的对象，每轮构造、每轮析构——帧循环里每帧构造几百个临时 `std::string` 是新手常见的隐形开销（热路径上的分配纪律将在第 6 章展开）。

最后区分两个容易混的词：**作用域（scope）**是名字的可见范围（词法概念），**存储期（duration）**是对象活多久（运行时概念）。局部对象两者重合，但堆对象存储期远大于作用域——`new` 出来的东西出了 `{}` 依然活着，直到 `delete`。这个错位正是悬垂指针与内存泄漏的总根源，1.11 节调试器一节我们会亲手抓一个。

### 1.3 值类别：lvalue 与 rvalue，以及三种引用的重载选择

接下来是 C++ 最独特的一块地基：**值类别（value category）**。请注意，它是**表达式的属性**，不是类型的属性——每个表达式都有类型，同时也都有一个值类别。

C++11 起值类别分五类（lvalue、xvalue、prvalue，加上组合概念 glvalue/rvalue），初学阶段我们按「二分 + 一个特殊角色」来建立直觉：

- **lvalue（左值）**：有身份的对象——有名字、能取地址、能反复使用。`hero`、`players[0]`、`*ptr` 都是。
- **prvalue（纯右值）**：临时的值——字面量 `42`、`a + b` 的结果、按值返回的函数调用结果。它们没有稳定地址，用完即逝。
- **xvalue（将亡值）**：一个**本来是 lvalue 的对象**，被标记为「可以搬走它」。唯一常见的产生方式：`std::move(x)`。它既要有身份（是个真对象），又允许被掏空（将亡）——所以我们把它同时算进「有身份的 glvalue」和「可搬走的 rvalue」两族。

对照表（背下前四行，第五行是易错点）：

| 表达式 | 值类别 | 直觉 |
| --- | --- | --- |
| `hero`（具名变量） | lvalue | 有名字、有地址、可反复用 |
| `42`、`a + b`、按值返回的 `makeImage()` | prvalue | 临时值，用完即逝 |
| `std::move(hero)` | xvalue | 「这个 lvalue 可以被搬走」的标记 |
| `getImageRef()`（返回 `Image&`） | lvalue | 返回引用 = 既有对象本身 |
| `"brick.png"`（字符串字面量） | **lvalue** | 它是常量字符数组对象，有地址——很多人栽在这 |

**为什么要关心值类别？因为引用重载。** C++ 有三种引用形参，它们能「接住」的值类别不同：

| 形参类型 | 能绑定 | 语义 |
| --- | --- | --- |
| `T&` | 仅非 const 的 lvalue | 「我要就地修改你」 |
| `const T&` | 全部：lvalue、rvalue、const 对象 | 「只读访问」的万能兜底 |
| `T&&`（右值引用） | 仅右值（prvalue 和 xvalue） | 「我要搬走你的资源」——移动语义的入口 |

当 `const T&` 和 `T&&` 两个重载并存时，**左值实参选 `const T&`，右值实参选 `T&&`**——这就是移动语义的调度台。看完整可编译的演示：

```cpp
// ch01_value_category.cpp —— 引用重载如何按值类别调度（完整可编译）
#include <cstdio>
#include <string>
#include <utility>

void feed(const std::string& s) { std::printf("  [const T& 版本] 只读访问：%s\n", s.c_str()); }
void feed(std::string&& s)      { std::printf("  [T&& 版本] 可以窃取内部资源：%s\n", s.c_str()); }

// 一个「转发者」：它收到了一个右值引用参数，想把它继续递给下一棒
void relay(std::string&& s) {
    feed(s);             // 注意！s 有名字 → 在这里它是 lvalue → 选中 const T& 版本
    feed(std::move(s));  // 想保持「可搬走」属性，必须再 move 一次 → 选中 T&& 版本
}

int main() {
    std::string name = "goblin";
    std::printf("feed(name)（lvalue）：\n");
    feed(name);                    // lvalue → const T&
    std::printf("feed(std::string(\"troll\"))（prvalue 临时对象）：\n");
    feed(std::string("troll"));    // prvalue → T&&
    std::printf("feed(std::move(name))（xvalue）：\n");
    feed(std::move(name));         // xvalue → T&&
    std::printf("relay 里：具名的右值引用参数 s 本身是 lvalue：\n");
    relay(std::string("orc"));
    std::printf("relay 没有动 name：%s\n", name.c_str());
    return 0;
}
```

实测输出：

```text
feed(name)（lvalue）：
  [const T& 版本] 只读访问：goblin
feed(std::string("troll"))（prvalue 临时对象）：
  [T&& 版本] 可以窃取内部资源：troll
feed(std::move(name))（xvalue）：
  [T&& 版本] 可以窃取内部资源：goblin
relay 里：具名的右值引用参数 s 本身是 lvalue：
  [const T& 版本] 只读访问：orc
  [T&& 版本] 可以窃取内部资源：orc
relay 没有动 name：goblin
```

三个结论，请反复咀嚼直到觉得理所当然：

1. **值类别看的是表达式，不是对象**。`name` 是 lvalue，`std::move(name)` 是 xvalue——同一个对象，两个不同类别的表达式。`std::move` 没有对 `name` 做任何事（下一节细讲）。
2. **具名的右值引用参数，在函数体内部是 lvalue**。它有名字！所以 `relay` 里直接传 `s` 会退化为只读绑定；想继续传递「可搬走」属性必须 `std::move(s)`。这个细节是理解一切移动转发代码的钥匙。
3. 小知识：用 `const T&` 或 `T&&` 绑定一个临时对象，会把临时对象的生命延长到引用的作用域结束；日常传参不必依赖这条规则，知道即可。

顺带一提：`feed("orc")` 这种「字符串字面量」实参会先隐式转换出一个 `std::string` 临时对象（prvalue），因此选中的是 `T&&` 版本——转换产生的临时值是 prvalue，这条规则以后到处都会遇到。

### 1.4 拷贝 vs 移动：用 Image 类看清两种交接

现在把值类别接到对象行为上。一个游戏里最典型的重资源对象——一张纹理图像：名字、宽高，和一块堆上的像素数据。我们亲手给它写全「五个特殊成员函数」：拷贝构造、拷贝赋值、移动构造、移动赋值、析构（加上默认构造是六个「特殊成员函数」，本例用不到默认构造）。每个成员都打日志，让机器亲口告诉我们拷贝和移动的区别。

先看图景，再看代码：

```text
拷贝构造（深拷贝）——两个对象，两份资源：
  hero.pixels_       ──────▶ [1 MB 像素 A]
  backup.pixels_     ──────▶ [1 MB 像素 B]   （新分配内存，逐字节复制，慢但独立）

移动构造（窃取）——还是两个对象，一份资源：
  heroInScene.pixels_ ─────▶ [1 MB 像素]      （指针交接，零复制，快）
  hero.pixels_        ══════▶ nullptr         （原主变空壳，析构时无事可做）
```

完整可编译程序（本章的核心样例，后面多节反复引用它）：

```cpp
// ch01_image.cpp —— 自测资源类 Image：拷贝/移动/析构全日志（完整可编译）
// cl /std:c++20 /EHsc /W4 ch01_image.cpp
#include <cassert>
#include <cstdio>
#include <cstring>
#include <utility>   // std::move

class Image {
public:
    // 普通构造：分配一块堆内存当「像素数据」（RGBA，每像素 4 字节）
    Image(const char* name, std::size_t w, std::size_t h)
        : name_(name), width_(w), height_(h),
          pixels_(new unsigned char[w * h * 4])
    {
        std::printf("  [构造]     Image(%s, %zux%zu)\n", name_, width_, height_);
    }

    // 析构：释放堆内存
    ~Image() {
        if (pixels_)
            std::printf("  [析构]     Image(%s)，释放 %zu 字节\n", name_, byteCount());
        else
            std::printf("  [析构]     Image(被移动后的空壳)\n");
        delete[] pixels_;
    }

    // 拷贝构造：深拷贝——新分配内存，逐字节复制
    Image(const Image& other)
        : name_(other.name_), width_(other.width_), height_(other.height_),
          pixels_(new unsigned char[other.byteCount()])
    {
        std::memcpy(pixels_, other.pixels_, byteCount());
        std::printf("  [拷贝构造] Image(%s) —— 重新分配并复制了 %zu 字节\n", name_, byteCount());
    }

    // 拷贝赋值：放掉自己的旧资源，再深拷贝对方
    Image& operator=(const Image& other) {
        std::printf("  [拷贝赋值] %s = %s\n", name_, other.name_);
        if (this != &other) {                 // 自我赋值防护（1.7 节细讲）
            delete[] pixels_;                 // 先丢掉自己的旧像素
            width_ = other.width_;
            height_ = other.height_;
            pixels_ = new unsigned char[other.byteCount()];
            std::memcpy(pixels_, other.pixels_, byteCount());
        }
        return *this;
    }

    // 移动构造：窃取——把对方的指针与尺寸「整栋搬走」，对方变成空壳
    Image(Image&& other) noexcept             // noexcept 的深意见本章深入专题
        : name_(other.name_), width_(other.width_), height_(other.height_),
          pixels_(other.pixels_)              // 指针直接拿过来，零内存分配
    {
        other.pixels_ = nullptr;              // 关键：让对方析构时无资源可放
        other.width_ = other.height_ = 0;
        std::printf("  [移动构造] Image(%s) —— 只是接手指针，没有复制任何像素\n", name_);
    }

    // 移动赋值：放掉自己的旧资源，再窃取对方
    Image& operator=(Image&& other) noexcept {
        if (this != &other) {
            std::printf("  [移动赋值] %s ← 窃取 %s（放掉自己旧的 %zu 字节）\n",
                        name_, other.name_, byteCount());
            delete[] pixels_;
            name_ = other.name_;
            width_ = other.width_;
            height_ = other.height_;
            pixels_ = other.pixels_;
            other.pixels_ = nullptr;
            other.width_ = other.height_ = 0;
        }
        return *this;
    }

    std::size_t byteCount() const { return width_ * height_ * 4; }

private:
    const char* name_;
    std::size_t width_ = 0;
    std::size_t height_ = 0;
    unsigned char* pixels_ = nullptr;   // 默认成员初始化器：新对象从「空」开始（1.10 节）
};

int main() {
    std::printf("== 1. 构造 ==\n");
    Image hero("hero.png", 512, 512);

    std::printf("== 2. 拷贝（深拷贝：重新分配内存）==\n");
    Image heroBackup = hero;                    // 拷贝构造

    std::printf("== 3. 移动（指针交接：零复制）==\n");
    Image heroInScene = std::move(hero);        // 移动构造：hero 从此是空壳

    std::printf("== 4. 断言验证 ==\n");
    assert(heroInScene.byteCount() == 512u * 512u * 4u);
    assert(hero.byteCount() == 0);              // 空壳：按本类的约定，尺寸归零
    std::printf("  断言通过：heroInScene 资源完整，hero 已变成空壳\n");

    std::printf("== 5. 移动赋值 ==\n");
    heroInScene = Image("cache.png", 64, 64);   // 右值实参 → 选中移动赋值

    std::printf("== 程序结束，逆序析构 ==\n");
    return 0;
}
```

实测输出：

```text
== 1. 构造 ==
  [构造]     Image(hero.png, 512x512)
== 2. 拷贝（深拷贝：重新分配内存）==
  [拷贝构造] Image(hero.png) —— 重新分配并复制了 1048576 字节
== 3. 移动（指针交接：零复制）==
  [移动构造] Image(hero.png) —— 只是接手指针，没有复制任何像素
== 4. 断言验证 ==
  断言通过：heroInScene 资源完整，hero 已变成空壳
== 5. 移动赋值 ==
  [构造]     Image(cache.png, 64x64)
  [移动赋值] hero.png ← 窃取 cache.png（放掉自己旧的 1048576 字节）
  [析构]     Image(被移动后的空壳)
== 程序结束，逆序析构 ==
  [析构]     Image(cache.png)，释放 16384 字节
  [析构]     Image(hero.png)，释放 1048576 字节
  [析构]     Image(被移动后的空壳)
```

逐条拆解这份日志里的机制：

- **第 2 步**：`Image heroBackup = hero;` 这个 `=` 是**初始化**，不是赋值——新对象诞生，走拷贝构造。深拷贝分配了新的 1 MB 并逐字节复制：慢，但两个对象从此互不相干（这是「值语义」：每个对象都像独立的一份值）。
- **第 3 步**：`std::move(hero)` 把 `hero` 这个 lvalue 表达式转换成 xvalue（右值），于是选中移动构造。移动构造**只交接指针**，然后必须把 `other.pixels_` 置空——否则同一个指针会被析构两次（双重释放）。移动后的 `hero` 处于「空壳」状态。
- **第 5 步**：赋值号右边是右值（临时对象），所以选中**移动赋值**：先放掉自己的旧 1 MB，再窃取对方的 64 KB。随后的「析构（空壳）」就是那个临时对象——它的资源已经被 `heroInScene` 拿走了。
- **结尾**：三个对象按声明逆序析构，各打各的日志——1.2 节的规则在这里又一次生效。

两个工程要点：

- **被移动后的对象是什么状态，由你（类的作者）定义**。惯例是：保证它**可析构、可重新赋值**；标准库类型的惯例叫「有效但未指定状态（valid but unspecified）」——可以安全析构或赋新值，但不要假设它的具体内容。本章 Image 的契约是「空壳」。
- `assert` 是开发期自查工具，在定义了 `NDEBUG` 的 Release 构建里会被整体移除——所以断言里不要放有副作用的代码。

最后给你一个震撼对照：同样的 Image，如果把像素放进 `std::vector<unsigned char>`（内存由 vector 管理），就一行特殊成员函数都不用写：

```cpp
// 节选：rule of zero 版的 Image（对比上面手写五件的版本）
class ImageZero {
    std::string name_;
    std::vector<unsigned char> pixels_;   // 内存归 vector 管
public:
    ImageZero(std::string name, std::size_t w, std::size_t h)
        : name_(std::move(name)), pixels_(w * h * 4) {}
    // 拷贝/移动/析构：一行都不写，全部自动正确
};
```

这不是偷懒，这是更高级的正确性——它就是下一节的主角「rule of zero」。手写五件的 Image 是为了让你理解机器；工程实践中默认走 zero 版。

### 1.5 std::move 的真相：它只是转换，不移动任何东西

现在可以精确回答本章最重要的一句话了：

> **std::move 不移动任何东西。** 它是一个到右值引用的类型转换（cast），运行期一条指令都不产生；真正的移动，发生在随后被选中的**移动构造函数/移动赋值运算符**里。

概念级的「全貌」：

```cpp
// 节选：std::move 的本质。真实签名是模板（模板机制将在第 3 章展开），行为等价于：
//   std::move(x)  ≈  static_cast<Image&&>(x)
// 唯一效果：把表达式 x 从「lvalue」标记为「xvalue（可搬走）」，仅此而已。
```

由此推出四个常被忽视的事实：

1. **`std::move` 之后，如果没有右值重载来接，就什么也不会发生。** 最经典的场景——对 const 对象「移动」：

   ```cpp
   // 节选（复用 1.4 的 Image 类）
   const Image logo("logo.png", 64, 64);
   Image copy = std::move(logo);   // 实际调用的是【拷贝构造】！
   ```

   原因：`Image&&` 不能绑定 const 对象（它承诺要掏空人家，const 不答应），于是重载决议退回 `const Image&`——拷贝构造。静默地、毫无提示地。这就是为什么「const 成员 + move」组合经常是性能陷阱。

2. **`std::move` 不清空、不释放、不调用任何函数**。`std::move(x);` 单独一行写在那里，什么都不会发生，只是产生了一个临时的右值表达式然后被丢弃。

3. **移动之后对象没有「死」**。它处于你（或标准库）定义的「有效但未指定」状态：可以析构、可以赋新值，但别去读它的内容。工程纪律：`move` 出去之后，除非下一行就重新赋值，否则不要再用那个对象。

4. **返回局部变量时不要画蛇添足**：

   ```cpp
   // 节选
   Image loadImage() {
       Image img("level1.png", 1024, 1024);
       return img;                 // ✅ 编译器会走「拷贝消除/NRVO」快路，退路也是移动
       // return std::move(img);   // ⚠ 反模式：反而关上了 NRVO 的门，可能更慢
   }
   ```

   为什么是反模式，见本章深入专题第 4 小节（涉及 C++17 起的强制拷贝消除）。

把 1.3 节的结论和这里连起来，你就拿到了完整链条：**值类别决定重载决议，重载决议决定调用拷贝还是移动，真正的资源交接发生在移动构造/移动赋值函数体内**。`std::move` 只是链条第一环的「举牌员」。

### 1.6 三/五/零法则：特殊成员函数什么时候该自己写

上一节你手写了五个特殊成员函数，感觉掌控一切；但 C++ 工程的正解恰恰是「能不写就不写」。先把编译器的「默认生成规则」摆上台面——**你写（或不写）其中一个，会影响其他几个的生成**：

| 你写了…… | 默认构造 | 析构 | 拷贝构造/拷贝赋值 | 移动构造/移动赋值 |
| --- | --- | --- | --- | --- |
| 什么都不写 | 生成 | 生成 | 生成 | 生成（成员均可移动时） |
| 只写了析构函数 | 生成 | 你的 | 生成（此写法已被标准标记为废弃） | **不再生成** |
| 写了任一拷贝操作 | 生成 | 生成 | 你的 | **不再生成** |
| 写了任一移动操作 | 生成 | 生成 | **删除** | 你的 |

另外两个隐式删除的常见触发点：类里有**引用成员**或 **const 成员**时，赋值运算符被删除（赋值无法给它们「换内容」）。这份表是无数诡异 bug 的源头——最典型的是第一行与第二行之间的落差：**给类加了析构函数（哪怕 `= default`），移动操作就悄悄消失了**，之后所有「移动」静默退化成拷贝。

在这个背景下，三条法则依次登场：

- **Rule of Three（三法则，C++98 时代）**：如果你的类需要自定义**析构函数、拷贝构造、拷贝赋值**三者之一，那它几乎肯定三个都需要——因为「需要自定义析构」意味着类直接管理资源，而编译器默认生成的拷贝是浅拷贝，会造成双重释放。
- **Rule of Five（五法则，C++11 起）**：在上面的基础上补上**移动构造、移动赋值**。而且按生成规则，既然你写了析构，移动已经不会自动生成了，要么手写、要么显式 `= default`，否则移动语义名存实亡。
- **Rule of Zero（零法则，现代默认）**：**根本不要让类直接拥有裸资源**。把资源交给那些「自己管好自己一生一世」的成员类型——`std::string`、`std::vector`，以及第 2 章将展开的智能指针。于是五个特殊成员函数一个都不用写，拷贝/移动/析构全部自动正确。

**默认取舍的判断法**，一句话：

> 问自己：「这个类是否**直接拥有**一块需要手动释放的资源？」
> ——否（成员全是 string/vector/纯数据）：**rule of zero**，一行都不写。
> ——是（类里躺着裸指针/裸句柄）：**rule of five**，五件套齐全，移动操作记得标 `noexcept`。

游戏语境的三类典型：

```cpp
// 节选：三类设计的典型形状（示意片段，无 main；需要 <string> <vector> <algorithm> <utility>）

// ① Rule of zero：纯数据或「会自己活的」成员（绝大多数类应该长这样）
struct Particle {
    std::string tag;
    std::vector<float> pos;
};

// ② Rule of five：类直接拥有裸资源（多数场景将被第 2 章的智能指针/RAII 包装取代）
class Heightfield {          // 地形高度场：一片自己分配的浮点内存
public:
    explicit Heightfield(std::size_t n)
        : size_(n), data_(new float[n] {}) {}
    ~Heightfield() { delete[] data_; }

    Heightfield(const Heightfield& other)                    // 拷贝构造：深拷贝
        : size_(other.size_), data_(new float[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    Heightfield& operator=(const Heightfield& other) {       // 拷贝赋值（copy-and-swap，见 1.7）
        Heightfield tmp(other);
        swap(tmp);
        return *this;
    }

    Heightfield(Heightfield&& other) noexcept                // 移动构造：窃取
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;
    }
    Heightfield& operator=(Heightfield&& other) noexcept {   // 移动赋值：交换式
        Heightfield tmp(std::move(other));
        swap(tmp);
        return *this;
    }

    void swap(Heightfield& other) noexcept {
        std::swap(size_, other.size_);
        std::swap(data_, other.data_);
    }

private:
    std::size_t size_;
    float* data_;
};

// ③ 仅移动类型：复制无意义、移交所有权有意义的资源（拷贝显式删除 + 移动放行）
class Mesh {                 // 节选：独占一块不能复制的外部资源
public:
    Mesh(const Mesh&) = delete;             // 禁止拷贝
    Mesh& operator=(const Mesh&) = delete;
    Mesh(Mesh&&) = default;                 // 允许移动（显式写明，自文档化）
    Mesh& operator=(Mesh&&) = default;
    ~Mesh() = default;
    // ……构造等省略
};
```

注意 ② 的两个赋值我用了 1.7 节的「交换式」写法，既正确又省事。还想看「三个成员函数都不写但依然出 bug」的反面教材？本章自测题第 2 题的类 B 就是。

最后强调一次优先级：**rule of zero 是默认答案**，rule of five 是「类里必须躺裸资源」时的义务。当你发现自己工程里到处在写五件套，通常是设计往 zero 迁移的信号（资源包装的第 2 章会给你全套工具）。

### 1.7 swap 惯用法与自我赋值防护

先说一个冷知识级别的合法操作：**自我赋值 `x = x;` 完全合法**，而且现实中并不罕见——`a[i] = a[j];` 在 `i == j` 时、两个引用/指针别名指向同一对象时，都会发生。你的赋值运算符必须在这种场景下不出事。

看 1.4 节拷贝赋值的朴素写法为什么危险（假设去掉 `if (this != &other)`）：

```cpp
// 节选：有自我赋值漏洞的版本
Image& operator=(const Image& other) {
    delete[] pixels_;                                    // 若 this == &other：
    pixels_ = new unsigned char[other.byteCount()];      //   other.pixels_ 已经是野指针！
    std::memcpy(pixels_, other.pixels_, byteCount());    //   💥 读已释放内存
    // ……
}
```

`if (this != &other)` 的防护当然有效（1.4 的 Image 就这么写），但它只防自我赋值，不防另一个坑：**异常安全**——`delete[]` 之后如果 `new` 抛出内存不足异常，对象就处在「指针已删、新值未到」的残废状态。

**swap 惯用法（copy-and-swap）一步治两个病**：先在任何破坏发生之前把「可能失败的部分」全部做完（拷贝进临时对象），剩下的交换全是不抛异常的指针操作：

```cpp
// ch01_copy_and_swap.cpp —— copy-and-swap 惯用法（完整可编译）
#include <cstdio>
#include <utility>

class Buffer {
public:
    explicit Buffer(std::size_t n) : size_(n), data_(new int[n] {}) {
        std::printf("  构造 Buffer(%zu)\n", n);
    }
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& other) : size_(other.size_), data_(new int[other.size_]) {
        for (std::size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
        std::printf("  拷贝构造 Buffer(%zu)\n", size_);
    }

    // 拷贝赋值 = copy-and-swap：参数按值传入（拷贝在进函数前就完成了）
    Buffer& operator=(Buffer other) noexcept {   // 函数体只交换指针，不会抛
        std::printf("  赋值（copy-and-swap）\n");
        swap(*this, other);
        return *this;
    }                                            // other 带着我「旧的自己」在这里析构

    friend void swap(Buffer& a, Buffer& b) noexcept {
        std::swap(a.size_, b.size_);
        std::swap(a.data_, b.data_);
    }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    Buffer a(1024);
    Buffer b(16);
    std::printf("普通赋值 a = b：\n");
    a = b;      // 形参 other 是 b 的拷贝 → swap 后 a 拿到 b 的内容，旧的 a 随 other 析构
    std::printf("自我赋值 a = a：\n");
    a = a;      // 完全安全：拷贝出一份自己，再和自己交换——什么都不会坏
    std::printf("程序结束\n");
    return 0;
}
```

实测输出（GCC/Clang 可能对 `a = a;` 给出自赋值警告——它是对的，这里只是演示安全，不提倡这么写）：

```text
  构造 Buffer(1024)
  构造 Buffer(16)
普通赋值 a = b：
  拷贝构造 Buffer(16)
  赋值（copy-and-swap）
自我赋值 a = a：
  拷贝构造 Buffer(16)
  赋值（copy-and-swap）
程序结束
```

机制拆解：参数 `other` 按**值**传入——该做的深拷贝在进入函数体之前就完成了（这步可能抛异常，但此刻还没动过 `this`，对象完好）；`swap` 只交换两个指针和一个整数，绝不失败；函数返回时，形参 `other` 带着我们的旧资源析构。自我赋值？无非是「拷贝自己、和自己交换」，逻辑上自动安全。1.6 节 Heightfield 的赋值运算符就是同一招的「类内 swap」变体。

代价也要知道：copy-and-swap **总是**做一次拷贝，哪怕能证明不需要。极致优化路径（先判断 `this != &other`、再精心安排 delete/new 顺序保证异常安全）是库作者的活；对我们，正确性优先，copy-and-swap 是性价比最高的默认。顺带一提：给类提供一个 `noexcept` 的 `swap` 本身就是好习惯——1.6 的移动赋值和后面的容器扩容（深入专题）都受益于它。

### 1.8 const 正确性：从第一天养成的「编译器可验证承诺」

const 正确性不是洁癖，是**把「我不会改你」从口头承诺变成编译器强制契约**。游戏团队里它最实际的价值：读到 `const Image&` 参数，你立刻知道这个函数不会改纹理，敢放心在帧循环里传；而所有没标 const 的接口自动成为「重点怀疑对象」。

四个使用层级：

```cpp
// ch01_const.cpp —— const 正确性（完整可编译）
#include <cstdio>
#include <string>

class Skeleton {                    // 骨骼：角色动画的骨架
public:
    Skeleton(std::string name, std::size_t boneCount)
        : name_(std::move(name)), boneCount_(boneCount) {}

    // ① 层级一：const 成员函数——承诺不修改对象状态
    //    规则：const 对象只能调用 const 成员函数
    const std::string& name() const { return name_; }   // 返回 const&：不拷贝、不许改
    std::size_t boneCount() const { return boneCount_; }

    // ② 需要修改状态，就不标 const——接口自己会说话
    void rename(std::string newName) { name_ = std::move(newName); }

private:
    std::string name_;
    std::size_t boneCount_;
};

// ③ 层级二：只读参数一律 const&——不拷贝（对比按值传参的一次深拷贝），承诺不改
double boneRatio(const Skeleton& s, std::size_t totalBones) {
    return static_cast<double>(s.boneCount()) / static_cast<double>(totalBones);
}

int main() {
    Skeleton hero("hero", 64);
    const Skeleton locked("statue", 8);   // ④ 层级三：const 对象——编译期就锁死

    std::printf("%s：%zu 根骨骼\n", locked.name().c_str(), locked.boneCount());
    // locked.rename("x");  // 编译错误：const 对象不能调用非 const 成员函数

    std::printf("ratio = %.3f\n", boneRatio(hero, hero.boneCount() + locked.boneCount()));
    return 0;
}
```

（另有层级四：`const` 局部变量——值不再变化的变量尽量加 const，把「它不会变」编码进类型系统；以及指向 const 的指针/引用如 `const unsigned char*`，纹理数据对外只读暴露的标准姿势。）

几条实战纪律：

- **新写的成员函数，默认问一句「它该不该是 const」**。答案几乎总是「该」——除非它真的改状态。忘标 const 的代价是：将来 `const Image&` 参数的函数里想调它，编译不过，回头补 const 又可能引发连锁修改（const 传染是单向的：从内到外补齐即可）。
- **读参数用 `const T&`**，重对象（string、vector、Image）尤其如此；小对象（int、float、指针）按值传。这正是 1.3 节引用绑定表的第一行用途。
- getter 返回 `const std::string&` 而不是 `std::string`，省一次拷贝；返回对象内部地址的接口，返回类型带 const（把「别改我内部」写进签名）。
- 一个细节：按值传参时形参上的顶层 const（`void f(const int x)`）属于实现细节，头文件声明可以不写、.cpp 定义可以写，互不冲突——别在这种 const 上内耗。
- 极少数场景需要在 const 成员函数里改一个「逻辑上不算状态」的缓存成员，用 `mutable` 标注；今天知道有这个东西即可。
- 与 const 相关的编译期常量 `constexpr` 是另一套机制（编译期计算属于第 3 章的主题），本章不展开。

### 1.9 现代语法速览：auto、范围 for、结构化绑定、if constexpr、enum class、nullptr

这一节把现代 C++（C++11 起逐步引入、如今已是日常）的基本面一口气过完。每一件都先给「为什么要有它」，再给用法。完整演示程序在后，逐条先说清楚：

**auto——让编译器替你写类型。** 变量的类型从初始化器推导，长类型（迭代器、后续将遇到的 lambda）尤其舒服。两个纪律：一眼能看出类型时（`int hp = 100;`）就写明确类型，auto 用在「类型又长又显然」处；注意 auto 推导会**丢掉引用和顶层 const**——`auto img = images[0];` 拿到的是**拷贝**，想要引用请写 `const auto& img`。对 Image 这种重资源类，少一个 `&` 就多一次深拷贝。

**范围 for——遍历的默认姿势。** `for (const auto& item : items)` 替代下标循环；容器元素是重类型时务必用引用绑定避免逐元素拷贝。一个戒律：**不要在范围 for 里增删容器元素**（容器结构变化会使元素位置失效，失效规则的细节将在第 3 章展开）。

**结构化绑定（C++17）——一次拆开一个聚合。** `auto [x, y] = point;` 把结构体/pair/tuple 的成员一次性绑定到具名变量；遍历 map 时 `for (const auto& [name, score] : table)` 直接拿键值，可读性碾压 `it->first/it->second`。

**if constexpr（C++17）——编译期分支。** 条件在编译期求值，落选分支不参与运行。不写模板时它最有用的形态是「按编译期已知条件选代码」，比如按指针宽度区分 32/64 位路径；它的全部威力（配合模板做编译期多态）将在第 3 章展开。

**enum class（限定作用域枚举）——给状态一个安全的名字。** 与旧枚举相比：枚举值只在枚举名的作用域内（要写 `Rarity::Epic`）；**不隐式转换成 int**（把枚举当 int 传参直接编译错误）；可指定底层类型省内存。游戏里的状态、稀有度、槽位类型都该用它。

**nullptr——空指针的唯一写法。** 它有自己的类型 `std::nullptr_t`，能参与重载决议（老 `NULL` 本质是整数 0，遇到 `f(int)` 与 `f(char*)` 重载会选错）；比较、赋值语义清晰。新代码零理由再写 `NULL` 或 `0`。

完整可编译演示（游戏掉落结算）：

```cpp
// ch01_modern_syntax.cpp —— 现代语法速览（完整可编译）
#include <cstddef>
#include <cstdio>
#include <map>
#include <string>
#include <vector>

enum class LootRarity { Common, Rare, Epic, Legendary };   // 强类型枚举

struct LootDrop {
    std::string itemName;
    LootRarity rarity;
    int count;
};

const char* pointerWidthLabel() {          // if constexpr：编译期分支
    if constexpr (sizeof(void*) == 8)
        return "64 位进程";
    else
        return "32 位进程";
}

int main() {
    std::vector<LootDrop> drops = {
        { "红药水", LootRarity::Common,    5 },
        { "屠龙剑", LootRarity::Legendary, 1 },
        { "秘银锭", LootRarity::Rare,     12 },
    };

    // 范围 for + const 引用：不拷贝元素
    for (const LootDrop& d : drops) {
        std::printf("掉落：%s x%d\n", d.itemName.c_str(), d.count);
    }

    std::map<std::string, int> goldByPlayer{ { "alice", 120 }, { "bob", 77 } };

    // 结构化绑定（C++17）：直接拆出键与值
    for (const auto& [player, gold] : goldByPlayer) {
        std::printf("%s 拥有 %d 金币\n", player.c_str(), gold);
    }

    // auto：类型长而显然的场合交给编译器
    auto totalGold = 0;                    // int
    for (const auto& [name, coins] : goldByPlayer) totalGold += coins;
    std::printf("总金币 = %d\n", totalGold);

    // enum class：不隐式转 int
    LootRarity r = LootRarity::Epic;
    // int legacy = r;                    // 编译错误：必须显式
    int legacy = static_cast<int>(r);
    std::printf("Epic 的底层编号 = %d\n", legacy);

    // nullptr：空指针唯一正解
    LootDrop* target = nullptr;
    if (target == nullptr) std::printf("target 为空指针\n");

    std::printf("本程序运行于%s\n", pointerWidthLabel());
    return 0;
}
```

实测输出：

```text
掉落：红药水 x5
掉落：屠龙剑 x1
掉落：秘银锭 x12
alice 拥有 120 金币
bob 拥有 77 金币
总金币 = 197
Epic 的底层编号 = 2
target 为空指针
本程序运行于64 位进程
```

这些语法的共性：**都不改变底层机器模型，只是把意图表达得更准**。auto 没有运行期开销、enum class 只是作用域规则、结构化绑定是零成本的「起名字」。它们让你把省下的注意力花在真正难的事情上——比如下一节的初始化陷阱。

### 1.10 初始化陷阱：成员初始化顺序与统一初始化的收窄

初始化是 C++ 新手的重灾区，两个陷阱足以上「本章必考」名单。

**陷阱一：成员初始化顺序 = 声明顺序，与初始化列表的书写顺序无关。** 构造函数初始化列表**看起来**在按你写的顺序初始化，实际上编译器永远按**成员在类里声明的顺序**初始化。列表顺序写错只是骗了自己（编译器会警告），真正致命的是**声明顺序本身就错**：

```cpp
// ch01_init_order.cpp —— 成员初始化顺序陷阱（完整可编译，含故意保留的错误示范）
#include <cstdio>

struct ViewportBad {
    ViewportBad(int w, int h)
        : width_(w), height_(h),
          area_(width_ * height_)     // 看起来没问题……
    {}
    int area_;     // ⚠ 声明在最前 → 它其实【最先】被初始化！
    int width_;    //    此时 width_/height_ 还是未初始化的垃圾
    int height_;
};

struct ViewportGood {
    ViewportGood(int w, int h)
        : width_(w), height_(h), area_(width_ * height_)
    {}
    int width_;    // 声明顺序 = 初始化顺序
    int height_;
    int area_;
};

int main() {
    ViewportGood good(1920, 1080);
    std::printf("正确声明顺序：area = %d\n", good.area_);

    ViewportBad bad(1920, 1080);
    // 读取未初始化成员是未定义行为——输出不确定（教学演示，勿模仿）：
    std::printf("错误声明顺序：area = %d（未定义行为，多半是垃圾值）\n", bad.area_);
    return 0;
}
```

实测输出（垃圾值恰为 0，纯属本次运行运气）：

```text
正确声明顺序：area = 2073600
错误声明顺序：area = 0（未定义行为，多半是垃圾值）
```

工具提示：GCC/Clang 的 `-Wreorder`（构建时即开）会抓「列表顺序与声明顺序不一致」，还会提示「用了未初始化成员」；MSVC 对应警告 C5038 默认关闭，需要 `/w15038` 显式打开——建议你现在就把它加进工程。**纪律**：成员声明顺序与初始化依赖顺序保持一致；能用构造参数直接算的就别绕道成员（`area_(w * h)`）。

**陷阱二：统一初始化 `{}` 更严格，但规则要看清。** C++11 起花括号初始化（统一初始化）有两大卖点和一个大坑：

```text
卖点一：禁止收窄（narrowing）——放不进去就不让编译
卖点二：空 {} 是「零初始化」，int x{}; 保证是 0（未初始化问题的一键防御）
大  坑：{} 优先匹配 std::initializer_list 重载，遇到重载可能选到你没想用的那个
```

收窄规则逐行验证（以下判定均经编译器实测）：

| 语句 | 能否编译 | 说明 |
| --- | --- | --- |
| `int a = 3.14;` | ✅ | 隐式收窄成 3——历史包袱，`=` 不报错 |
| `int b{3.14};` | ❌ | `{}` 禁止浮点→整数 |
| `char c{65};` | ✅ | 常量 65 在 char 范围内 |
| `int x = 300; char d = x;` | ✅ | 运行期截断（300 按实现定义回绕成 44）——静默陷阱 |
| `int x = 300; char e{x};` | ❌ | x 不是常量，编译器不敢保证装得下 |
| `int f{3000000000};` | ❌ | 常量超出 int 范围 |
| `float g{0.1};` | ✅ | double 常量在 float 范围内（允许舍入） |
| `float h{1e40};` | ❌ | 超出 float 表示范围 |

工程建议：**局部变量和成员初始化默认用 `{}`**（白赚「禁收窄 + 防未初始化」两道保险）；但有两种例外要背下来——

例外 A：`{}` 会优先匹配 `initializer_list`，于是参数个数相同的两个构造函数含义完全不同：

```cpp
std::vector<int> countdown(3, 1);   // (count, value)：3 个 1 → {1, 1, 1}
std::vector<int> settings{3, 1};    // {列表}：       两个元素 → {3, 1}
```

写容器时想清楚要「列表」还是「计数」，选错初始化语法不会报错，只会默默给出错误的数据。

例外 B：`auto` 与 `{}` 的历史纠葛——`auto x{3};` 在 C++11 时代推出 `std::initializer_list<int>`，C++17 起修正为 `int`（如果你维护的是 C++11/14 代码，这里行为不同，要留意）；`auto y = {3};` 则任何标准都是 `initializer_list<int>`。

最后送一个同族的经典坑（「最令人费解的解析」）：`std::vector<Image> replayBuffer();` 不是「空 vector 对象」，而是「返回 vector 的函数声明」。想声明空容器，请写 `std::vector<Image> replayBuffer{};` 或 `= {}`。

### 1.11 调试器基本功：断点、单步、监视、调用栈与崩溃定位

从第一天起，调试器就是你观察本章所有概念（构造、析构、移动、悬垂）的显微镜。我们以 Visual Studio 为主讲（简述 gdb 等价物），把最常用的四件套过一遍，然后完整走一次崩溃定位。

**四件套（VS 快捷键）**：

| 能力 | 操作 | 用途 |
| --- | --- | --- |
| 断点 | 行号左侧单击，或 F9 | 程序在该行前停下 |
| 单步 | F10 步过 / F11 步入 / Shift+F11 步出 | F10 把函数调用当一行走完；F11 钻进函数内部 |
| 监视 | 调试 → 窗口 → 监视（Watch）/ 局部变量（Locals）/ 自动窗口（Autos） | 随时查看/监视表达式；悬停变量也有 DataTip |
| 调用栈 | 调试 → 窗口 → 调用栈（Ctrl+Alt+C） | 「我此刻是怎么走到这里的」——从栈顶（当前行）向下是调用者链 |

帧循环专属技巧——**条件断点**：右键断点 → 条件 → 填表达式（如 `g_frame == 600` 或 `hp <= 0`）。没有它，你在第 600 帧才出现的问题只能靠数一万次 F5。

**Debug 与 Release 构建的差异要心里有数**：MSVC Debug 构建关优化（`/Od`）、链接调试版运行时，调试堆会把**刚分配未初始化**的内存填成 `0xCDCDCDCD`、**未初始化的栈**字节是 `0xCC`、**已释放**的内存填成 `0xDDDDDDDD`。看到这些模式要条件反射——`0xCC`/`0xCD` ≈ 忘了初始化，`0xDD` ≈ 释放后使用。单步观察请用 Debug；但注意 Debug 与 Release 行为有差异（Release 优化会重排/消除代码、内存布局不同），「Debug 不崩 ≠ Release 没问题」。至于性能结论为什么必须用 Release 测，将在第 6 章展开。

**崩溃定位标准流程（以访问冲突为例）**：

1. 调试器下（F5）运行，崩溃时 VS 弹出异常提示「已引发异常：读取访问权限冲突」，点「中断」。
2. 打开**调用栈**窗口，从栈顶（崩溃点，常在库或 CRT 内部）逐帧向下双击，找到**你自己代码**的帧。
3. 读异常地址与数据：地址很小（`0x00000000` 附近）≈ 空指针解引用；`0xDDDDDDDD` ≈ 悬垂（释放后使用）；`0xCDCDCDCD` ≈ 未初始化。
4. 在**监视**窗口核对嫌疑指针的值，与相关容器的当前状态对比（地址是否还对得上）。
5. 修复后回归验证。

**实战：亲手抓一个悬垂指针。** 游戏里最经典的崩溃剧本——HUD 缓存了玩家指针，vector 一搬家，指针就指向已释放的旧内存（1.2 节埋的「堆对象存储期大于作用域」伏笔在此兑现）：

```cpp
// ch01_dangling.cpp —— 悬垂指针复现（完整可编译；含故意保留的 bug，即实践素材）
#include <cstdio>
#include <string>
#include <vector>

struct Player {
    std::string name;
    int hp = 100;
};

// HUD 组件：缓存了「当前目标玩家」的指针——bug 就埋在这
struct TargetHud {
    const Player* target = nullptr;      // ⚠ 缓存指针，而不是「目标的 ID/下标」
    void draw() const {
        std::printf("当前目标：%s（HP %d）\n", target->name.c_str(), target->hp);
    }
};

int main() {
    std::vector<Player> players;
    players.emplace_back(Player{ "alice", 100 });
    players.emplace_back(Player{ "bob", 80 });

    TargetHud hud;
    hud.target = &players[1];            // 指向 bob
    hud.draw();                          // 一切正常：当前目标：bob（HP 80）

    std::printf("—— 100 个新玩家加入，vector 扩容搬家 ——\n");
    for (int i = 0; i < 100; ++i) {
        players.emplace_back(Player{ "mob" + std::to_string(i), 30 });
    }

    std::printf("—— 下一帧 HUD 绘制 ——\n");
    hud.draw();                          // 💥 target 还指着已释放的旧内存
    std::printf("程序正常结束\n");
    return 0;
}
```

它的两种典型病象（都值得亲眼见一次）：

- **Debug 构建（MSVC 调试堆）**：旧内存已被 `0xDD` 毒化，`hud.target->name` 的内部指针读到 `0xDDDDDDDD`，第二次 `draw()` 几乎必然「读取访问权限冲突」。按上面五步走：中断 → 调用栈停在 `TargetHud::draw` → 监视窗口看 `hud.target` 的地址（如 `0x012a56e0`），再看 `players` 的数据指针——已经搬去新地址（如 `0x0131f4a0`），对不上号；再读 `*(Player*)0x012a56e0` 满眼 `0xDD`——实锤「释放后使用」。
- **Release 构建**：旧内存只是归还了堆，未必立刻崩溃。我在 Release 环境的实测输出是——

  ```text
  当前目标：bob（HP 80）
  —— 100 个新玩家加入，vector 扩容搬家 ——
  —— 下一帧 HUD 绘制 ——
  当前目标：（HP 80）
  ```

  名字变成乱码/空串、HP 却还是 80——数据烂了但程序活着。**这比崩溃更危险**：垃圾数据可能流进存档、联机包，而崩溃点与案发现场相距十万八千里。

修复方向（本阶段够用的两个）：缓存**下标或 ID** 而不是指针；或确知规模时 `reserve` 预留（缓解而非根治）。至于「指针/引用为什么会失效」的完整失效规则表，以及资源所有权的系统解法，将在第 3 章、第 2 章分别展开。

**事后也能查：崩溃转储（dump）**。玩家机器上崩了、你本机不复现怎么办？让程序生成转储文件（最简单：任务管理器右键进程 →「创建转储文件」，工程里通常用 `MiniDumpWriteDump` 集成崩溃上报），把 `.dmp` 拖进 Visual Studio，配好对应版本的符号（.pdb），就能看到崩溃瞬间的调用栈和局部变量——「案发录像」回放。概念级了解即可，工程落地会在你的渲染器项目里自然遇到。

**其他工具一句话**：Linux/命令行流派的 gdb（`g++ -g` 编译带调试信息；`break` 下断、`next/step` 单步、`bt` 看调用栈、`watch` 设观察点）与 VS Code 的 C/C++ 调试扩展提供等价能力，界面不同、概念完全一致。

---

## 深入专题：一次 vector 扩容的完整解剖——移动、noexcept 与拷贝消除

前面所有概念（值类别、拷贝/移动、析构次序、调试器）会在一个日常事件里同台演出：**vector 装满了**。这也是阶段 1 实践任务的核心观察点，值得单独解剖。

### 扩容时到底发生了什么

vector 承诺元素**连续存储**。当容量用尽，它的动作是：申请一块更大的内存（MSVC 约 1.5 倍、libstdc++ 约 2 倍，实现细节）→ 把旧元素**逐个搬到**新内存 → 析构旧元素 → 释放旧内存。关键问题：**「搬」是拷贝还是移动？**

### 实验代码与实测日志

```cpp
// ch01_vector_growth.cpp —— 观察一次 vector 扩容（完整可编译）
#include <cstdio>
#include <utility>
#include <vector>

class Texture {
public:
    explicit Texture(const char* tag) : tag_(tag), data_(new char[1024]) {
        std::printf("构造      %s\n", tag_);
    }
    ~Texture() {
        if (data_) std::printf("析构      %s\n", tag_);
        else       std::printf("析构      （移动后的空壳）\n");
        delete[] data_;
    }
    Texture(const Texture& other) : tag_(other.tag_), data_(new char[1024]) {
        std::printf("拷贝构造  %s   ← 重新分配并复制\n", tag_);
    }
    Texture(Texture&& other) noexcept              // ← 实验开关：试着删掉 noexcept
        : tag_(other.tag_), data_(other.data_)
    {
        other.data_ = nullptr;
        std::printf("移动构造  %s   ← 只交接指针\n", tag_);
    }
    Texture& operator=(const Texture&) = delete;   // 本例只观察构造路径
    Texture& operator=(Texture&&) = delete;

private:
    const char* tag_;
    char* data_;
};

Texture makeDefault() { return Texture("default.png"); }  // C++17：保证消除拷贝/移动

int main() {
    std::printf("— A. C++17 保证的拷贝消除 —\n");
    Texture t = makeDefault();                     // 日志只有一行「构造」

    std::printf("— B. reserve 之后 push_back —\n");
    std::vector<Texture> textures;
    textures.reserve(2);                           // 预留：接下来两次插入不搬家
    textures.push_back(Texture("brick.png"));      // 构造临时 → 移动进容器 → 空壳析构

    std::printf("— C. 再插入，触发扩容 —\n");
    textures.push_back(Texture("grass.png"));
    textures.push_back(Texture("stone.png"));      // 容量不够 → 扩容搬家

    std::printf("— 结束：逆序析构 —\n");
    return 0;
}
```

实测日志（Clang + MSVC STL；**各元素移动/析构的先后顺序属于实现细节，换个标准库可能不同，但「全程零拷贝构造」是稳定的**）：

```text
— A. C++17 保证的拷贝消除 —
构造      default.png
— B. reserve 之后 push_back —
构造      brick.png
移动构造  brick.png   ← 只交接指针
析构      （移动后的空壳）
— C. 再插入，触发扩容 —
构造      grass.png
移动构造  grass.png   ← 只交接指针
析构      （移动后的空壳）
构造      stone.png
移动构造  stone.png   ← 只交接指针
移动构造  brick.png   ← 只交接指针
移动构造  grass.png   ← 只交接指针
析构      （移动后的空壳）
析构      （移动后的空壳）
析构      （移动后的空壳）
— 结束：逆序析构 —
析构      brick.png
析构      grass.png
析构      stone.png
析构      default.png
```

逐段解读：

- **B 段**：`push_back(Texture(...))` 传入的是 prvalue 临时对象——先在调用点构造，再**移动**进容器槽位，最后临时对象以空壳身份析构。若先 `reserve` 好，插入就是「构造临时 + 移动」，永不触发大规模搬家。
- **C 段**：第三次 `push_back` 容量不足，扩容发生：新元素照常构造，**所有旧元素只被移动构造**，旧缓冲区的元素以空壳身份析构后整块释放——没有任何一次像素级复制。1 MB 的纹理搬家只需交接一个指针，这就是移动语义在容器里的价值。
- **收尾**：容器析构时元素逆序析构（`textures` 里是 stone → grass → brick），随后是 main 里先声明的 `t`——1.2 节的逆序规则在嵌套结构里同样成立。

### noexcept：一个关键字决定「移动还是拷贝」

把移动构造函数上的 `noexcept` 删掉再编译运行，C 段日志变成（同样实测）：

```text
构造      stone.png
移动构造  stone.png   ← 只交接指针
拷贝构造  brick.png   ← 重新分配并复制
拷贝构造  grass.png   ← 重新分配并复制
析构      brick.png
析构      grass.png
析构      （移动后的空壳）
```

**扩容悄悄退化成了拷贝**，性能一夜回到解放前，而且编译器一声不吭。原因：标准规定 vector 扩容搬迁按 `std::move_if_noexcept` 的语义行事——**只有移动构造是 `noexcept`（或类型不可拷贝）时才用移动**，否则退回拷贝。动机是异常安全（强保证）：如果搬到第 3 个元素时移动构造抛了异常，旧数据已被掏空、新数据不完整，两头落空；而拷贝即使中途失败，旧缓冲区毫发无损，整包丢弃即可。这条纪律请刻进肌肉记忆：

> **凡是「窃取资源」的移动构造/移动赋值，一律标 `noexcept`**——它们只做指针交接，本来就不该抛。这也是 1.4、1.6 示例里 noexcept 的来历（《Effective Modern C++》条目 14 专讲此事）。

### C++17 拷贝消除与 `return std::move` 反模式

回看日志 A 段：`Texture t = makeDefault();` 只有**一行「构造」**——没有临时对象、没有移动、没有拷贝。这是 C++17 的**强制拷贝消除（guaranteed copy elision）**：按值返回一个 prvalue 时，语言直接规定「在调用方的存储里构造这个对象」，连「可以省略」的优化都算不上，是标准保证的行为（C++17 之前这是可有可无的编译器优化 NRVO/RVO；引擎若停留在 C++14 及更早，同一段代码可能多出一次移动——移动很便宜，但不是零）。

这也解释了 1.5 节那条戒律：`return std::move(img);` 把返回表达式从「局部变量」变成了 xvalue，**恰好破坏了拷贝消除/NRVO 的适用条件**，反而强制走一次移动。什么都不写，编译器替你选最快的那条路。

最后把调试器和这场解剖接起来：在移动构造函数体内下一个断点（F9），F5 运行，命中后打开调用栈——你会看到 `push_back` 内部的扩容帧（标准库内部函数名因实现而异）正在调用你写的移动构造。日志给你「发生了什么」，调用栈给你「谁在什么时候干的」，两者合起来才是完整的证据链。

---

## 实践任务

三个任务对齐阶段 1 的实践要求。所有任务只需一个 .cpp 文件 + 命令行编译即可完成，不依赖任何后续章节的项目资产。

### 任务 1：写自测资源类（Image/Buffer），完整实现拷贝/移动/析构并全程打日志

从零（不要照抄 1.4，先自己写再对照）实现一个 `Buffer` 类：管理一块 `new int[n]` 分配的堆内存，成员 `std::size_t size_` 与 `int* data_`。要求：

1. 五个特殊成员函数**全部手写**：拷贝构造、拷贝赋值、移动构造（`noexcept`）、移动赋值（`noexcept`）、析构，每个成员函数第一行打印自己的名字与关键信息（如复制的字节数、窃取的指针）。
2. `main` 里依次演练：构造 → 拷贝构造 → 拷贝赋值 → 移动构造 → 移动赋值 → **自我赋值** `b = b;` → 程序结束自然析构。
3. 用 `assert` 验证：拷贝后双方独立（改一方数据，另一方不变）；移动后源对象处于你定义的「空壳」状态（`data_ == nullptr`）且能安全析构。
4. 数日志：构造/析构的次数应当配平，`new`/`delete` 的字节数应当配平。

**参考思路与验收要点**见章末参考答案；泄漏的仪器化检测工具（ASan、CRT 调试堆）将在第 2 章展开，本阶段用「日志配平」自查即可。

### 任务 2：三/五/零法则标注练习（不依赖后续资产的独立版本）

阶段 1 的原版任务要求给「第 2 节（计算机图形学）阶段 1 的手写数学库」逐类标注法则——若那个数学库尚未写出来，用下面这组给定的示例类完成同样的练习（数学库就绪后，把同一套标注套在 Vec2/Vec3/Vec4/Mat4 上即可，作为可选衔接）。对 A–E **逐类回答：该选 rule of zero / rule of five / 其他特殊设计？理由一句话**：

```cpp
// 标注练习素材（非完整程序——B 的特殊成员函数正是练习内容）
#include <string>

struct Particle { float x = 0.f, y = 0.f; };

struct Color {                                   // A
    float r = 0.f, g = 0.f, b = 0.f, a = 1.f;
};

class VertexBuffer {                             // B
public:
    explicit VertexBuffer(std::size_t vertexCount);   // 内部分配堆内存
    // ……其余特殊成员函数由你来判断与补全
private:
    float*      vertices_ = nullptr;   // 指向本类独占的一块堆内存
    std::size_t count_ = 0;
};

struct InputState {                              // C
    bool  keys[256] = {};
    float axisX = 0.f, axisY = 0.f;
};

struct NamedEntity {                             // D
    explicit NamedEntity(unsigned id, std::string name)
        : id_(id), name_(std::move(name)) {}
    const unsigned id_;                // 注意这个 const
    std::string name_;
};

struct HoverTarget {                             // E
    const Particle* target = nullptr;  // 非拥有：只「看着」别人，不管其生死
};
```

在 B 上补齐你选择的实现，并写最小单元测试（`assert` 即可）：拷贝后双方独立、移动后源对象可安全析构、自我赋值安全。

### 任务 3：在调试器中单步观察一次 vector 扩容

用本章深入专题的 `ch01_vector_growth.cpp`：

1. 编译运行，核对日志：扩容段应当**只出现移动构造，不出现拷贝构造**。
2. 删掉移动构造的 `noexcept` 再跑一遍，亲眼确认退化为拷贝构造；把结论写进代码注释后恢复 `noexcept`。
3. 在移动构造函数体内下断点，F5 命中后打开调用栈窗口，找到 `push_back` 内部的扩容帧；在监视窗口对比移动前后新旧两块缓冲区的地址。
4. （加分）在 `reserve(2)` 之前先连续 `push_back` 三次（不给预留），对比日志差异——理解 reserve 的价值。

---

## 自测题

先自己作答（口头或写在纸上），再对照章末参考答案。第 1–4 题对应阶段 1 的四条检验标准，务必都能独立完成。

**第 1 题（口述）**：不看书面材料，向同事讲清楚——`std::move(x)` 到底做了什么？没做什么？真正的「移动」发生在哪里？被移动后的 `x` 处于什么状态？

**第 2 题（法则判断）**：对下面三个类，分别判断该遵循 rule of zero、rule of five 还是特殊设计，并说出理由；指出其中隐藏的 bug：

```cpp
#include <string>

// A
class Sprite {
public:
    explicit Sprite(std::string path) : path_(std::move(path)) {}
private:
    std::string path_;
    int width_ = 0;
    int height_ = 0;
};

// B
class GpuTexture {
public:
    explicit GpuTexture(int handle) : handle_(new int(handle)) {}  // 模拟独占资源
    // 析构/拷贝/移动：一个都没写
private:
    int* handle_;
};

// C
class AudioStream {
public:
    explicit AudioStream(int device) : device_(device) {}
    AudioStream(const AudioStream&) = delete;
    AudioStream& operator=(const AudioStream&) = delete;
    AudioStream(AudioStream&&) = default;
    AudioStream& operator=(AudioStream&&) = default;
private:
    int device_;
};
```

**第 3 题（vector 扩容日志）**：某同学给 `Texture` 类写了移动构造但**忘标 `noexcept`**，运行深入专题的扩容实验后，C 段日志出现了 `拷贝构造 brick.png ← 重新分配并复制`。(a) 解释 vector 为什么宁可拷贝也不用他的移动构造；(b) 最少改动如何修复；(c) 这个行为背后是标准库的什么策略、为的是什么保证？

**第 4 题（调试器）**：程序在 MSVC Debug 构建下崩溃，VS 弹出「读取访问权限冲突」，你怀疑是悬垂指针。写出用 Visual Studio 定位成因的操作步骤（不少于 5 步），并说明 `0xDDDDDDDD`、`0xCDCDCDCD` 两种内存模式分别提示什么。

**第 5 题（初始化顺序）**：

```cpp
struct Arena {
    Arena(std::size_t n) : size_(n), slots_(size_ / 64) {}
    std::size_t slots_;
    std::size_t size_;
};
Arena a(640);
```

`a.slots_` 的值是多少？为什么？给出两种修复方式。

**第 6 题（narrowing）**：以下语句在 C++20 下各能否编译？（不查资料先判，再对照 1.10 节表格）
`int a{3.14};`　|　`char c{65};`　|　`int x = 300; char d = x;`　|　`int x = 300; char e{x};`　|　`float g{0.1};`

**第 7 题（析构次序）**：写出下面程序的完整输出（构造与析构每一行）：

```cpp
#include <cstdio>
struct Tracer {
    const char* name;
    explicit Tracer(const char* n) : name(n) { std::printf("构造 %s\n", name); }
    ~Tracer() { std::printf("析构 %s\n", name); }
};
Tracer g_load("全局资源");
static Tracer g_static("静态资源");
int main() {
    Tracer a("帧对象");
    {
        Tracer b("通道A");
        Tracer c("通道B");
    }
    static Tracer s("函数内静态");
    Tracer d("帧尾对象");
    std::printf("main 结束\n");
    return 0;
}
```

**第 8 题（引用重载）**：声明如下（`Image` 即 1.4 的类）：

```cpp
void take(const Image& img);   // ②
void take(Image&& img);        // ③
Image a("a.png", 32, 32);
take(a);                       // (1) 选哪个？
take(std::move(a));            // (2) 选哪个？
take(Image("b.png", 8, 8));    // (3) 选哪个？
const Image c("c.png", 8, 8);
take(std::move(c));            // (4) 选哪个？为什么？
```

**第 9 题（翻译单元与 ODR）**：`frame.cpp` 和 `main.cpp` 里都写了 `int g_frame = 0;`（且都未被任何机制豁免），构建会发生什么？错误发生在哪个阶段、错误名是什么？如果把 `int g_frame = 0;` 放进被两个 .cpp 都包含的 `common.h`，会发生同样的事吗？正确做法是什么？

---

## 常见误区

**误区一：把 C++ 当「带类的 C」学。** 症状：资源获取用裸 `new`、释放用配对的 `delete`，散落在函数各处，靠「记得调用」维持正确性。病根是没建立「析构是确定性的、与作用域绑定」的模型，错过了 C++ 相对 GC 语言的最大优势。纠偏：新代码从第一行起默认 rule of zero，资源包装的正解（RAII、智能指针）是第 2 章的主菜；本章你只需要先尝到 1.2 节「作用域即生命周期」的甜头。

**误区二：只背语法不建模型。** 症状：能背出移动构造的签名，却答不出「vector 扩容一次，N 个元素各自发生了什么」。语法是模型的投影，模型不在，语法背了也会用错。纠偏：本章实践任务 1 和 3 就是建模型的刻意练习——让日志和调试器替你验证每一次「我以为」，直到「我以为」和「实际发生」重合。

**误区三：以为 `std::move` 会移动。** 症状：写完 `std::move(x)` 就以为 `x` 已经被清空/搬走，甚至以为它有运行期开销。事实：`std::move` 是纯编译期转换（举牌员），可能发生的移动发生在**随后选中的移动构造/赋值**里；如果实参是 const、或目标类型没有移动重载，它可能静默退化为拷贝。自查口诀：**「move 之后用了吗？ move 之后谁接了？」**。

**误区四：以为被移动后的对象「已销毁/是空指针」。** 症状：移动后立刻解引用原对象，或反过来不敢对移动后的对象做任何事（连析构都怕）。事实：移动后的对象处于「有效但未指定」状态——可析构、可赋新值；具体是什么样子由类的作者定义（本章 Image 的契约是「空壳」），标准库类型只保证「能安全析构/赋值」，不保证内容。读代码时先找类文档里的移动后契约。

**误区五：以为 `{}` 万能、或以为它只是「另一种写法」。** 症状：`std::vector<int> v{3, 1};` 想要「3 个 1」；`auto x{3};` 在 C++11/14 代码库里拿到 `initializer_list`；反过来又有人因为怕 `{}` 而处处用 `=`，把收窄陷阱（`int a = 3.14;` 静默截断）全放进来。纠偏：记住 1.10 节的分工——默认 `{}`（防收窄、防未初始化），容器「计数构造」和 auto 的场合看清 initializer_list 例外。

**误区六：移动构造不标 `noexcept`。** 症状：类功能全对、单测全绿，性能却莫名差——vector 扩容在静默拷贝。这是本章唯一的「一个关键字差出一个数量级」的陷阱，机制与实验见深入专题；自查方法：对自写资源类跑一次扩容日志（实践任务 3）。

**误区七：混淆「值类别」与「类型」，或把值类别安在变量头上。** 症状：认为「rvalue 引用参数在函数里还是 rvalue」；以为字符串字面量是 rvalue。纠偏：值类别是**表达式**的属性——`x` 是 lvalue，`std::move(x)` 是 xvalue，`"s"` 也是 lvalue；具名的右值引用参数在函数体内是 lvalue（1.3 节 relay 的演示）。

**误区八：在 Debug 下观察一切、并把 Debug 行为当真理。** 症状：Debug 里单步走的代码和 Release 跑的不是同一条路径（优化器重排/消除）；Debug 有调试堆毒化模式而 Release 没有，悬垂问题在两个配置下病象完全不同（1.11 节实测）。纠偏：调试用 Debug，验证真机行为用 Release；性能结论一律 Release（量化方法将在第 6 章展开）。

**误区九：`return std::move(local);`。** 症状：听说移动很快，返回局部变量前也补一刀 move。事实：这会破坏拷贝消除/NRVO 条件，反而强制一次移动；直接 `return local;` 才是快路（深入专题第 4 小节）。

---

## 延伸资源

**learncpp.com**（免费、持续更新的系统教程，本阶段打底首选；章节编号以站内目录为准）：

- 第 3 章「Debugging C++ Programs」——调试器基本功的图文版，与本章 1.11 互为补充；
- 第 7 章「Scope, Duration, and Linkage」——作用域/存储期/链接的完整细则；
- 第 12 章「Compound Types: References and Pointers」、第 14–15 章「Introduction to Classes / More on Classes」——引用与类的细节手册；
- 第 19 章「Rvalue References, Move Semantics, and Forward References」——本章移动语义的加深阅读。

**《Effective Modern C++》（Scott Meyers）**，与本章直接相关的条目：

- 条目 23「理解 std::move 与 std::forward」——1.5 节的权威展开（forward 部分等学到模板再回来读第二遍）；
- 条目 17「理解特殊成员函数的生成机制」——1.6 节生成规则的完整版；
- 条目 14「不抛异常就声明 noexcept」——深入专题 noexcept 约定的出处；
- 条目 7「区别使用 () 和 {} 创建对象」——1.10 节初始化陷阱的全景；
- 条目 10「优先选用限定作用域的枚举」——enum class 的取舍论证；
- 条目 29「假设移动操作不存在、成本高昂、未被使用」——给移动语义泼的冷水，帮你校准预期。

**cppreference（全程工具书，语义疑问先查它）**：

- 值类别：<https://en.cppreference.com/w/cpp/language/value_category>
- 存储期：<https://en.cppreference.com/w/cpp/language/storage_duration>　｜　对象生存期：<https://en.cppreference.com/w/cpp/language/lifetime>
- 三/五/零法则：<https://en.cppreference.com/w/cpp/language/rule_of_three>
- std::move：<https://en.cppreference.com/w/cpp/utility/move>
- 列表初始化（含 narrowing 精确定义）：<https://en.cppreference.com/w/cpp/language/list_initialization>
- 拷贝消除：<https://en.cppreference.com/w/cpp/language/copy_elision>
- 翻译阶段与 ODR：<https://en.cppreference.com/w/cpp/language/translation_phases>　｜　<https://en.cppreference.com/w/cpp/language/definition>

**CppCon 讲座与专题书**：

- Nicolai Josuttis《C++ Move Semantics — The Complete Guide》——移动语义最系统的小册子，本章学完可当参考书翻；
- CppCon 2019, Nicolai Josuttis, *The Nightmare of Move Semantics for Simple Classes*——为什么「简单类」的移动构造也没那么简单，呼应 1.6 生成规则；
- CppCon 2018, Matt Godbolt, *What Has My Compiler Done for Me Lately?*——用 Compiler Explorer 亲眼看编译产物，给 1.1 的编译链路地图装上显微镜；
- CppCon 2019, Greg Law, *Debugging Linux C++: Tools and Techniques*——gdb 流派速成（Windows 玩家选听，与 1.11 末尾一句话呼应）。

---

## 参考答案

### 自测题

**第 1 题**。参考口述：`std::move(x)` 是一个到右值引用的**类型转换**，它把表达式 `x` 从左值标记为「可被搬走的右值（xvalue）」，除此之外**什么都没做**——不移动数据、不清空对象、不调用任何函数、运行期零开销。真正的移动发生在**随后被重载决议选中的移动构造函数/移动赋值运算符**里，由它们完成「窃取资源、把源对象置为空壳」的实际操作；如果不存在移动重载（或对象是 const 导致右值引用绑不上），同样的代码会静默走拷贝。被移动后的对象处于「有效但未指定」状态：能安全析构、能重新赋值，但内容不可假设——具体契约由类的作者定义。

**第 2 题**。

- **A（Sprite）**：rule of zero。成员是 `std::string` 和两个 int，没有裸资源，编译器生成的全部六个特殊成员函数都正确（string 的深拷贝/移动自带）。一行都不该手写。
- **B（GpuTexture）**：**藏着 bug**。它直接拥有裸资源（`new int`）却没写任何特殊成员函数：默认析构不会 `delete` → 泄漏；默认拷贝是浅拷贝 → 两个对象析构时对同一指针 `delete` 两次 → 双重释放崩溃；默认移动同样浅。正解：**rule of five**——析构 `delete[] handle_`、拷贝深拷（或按语义禁止）、移动窃取并置空（标 `noexcept`）。它就是阶段 1 检验标准里「任给一个类，能说出该遵循哪条法则」的反面典型。
- **C（AudioStream）**：特殊设计——**仅移动类型**（rule of five 的合法变体）：拷贝被显式 `= delete`（复制一个独占音频设备流没有意义），移动 `= default` 放行。设计自洽，前提是「独占、可移交」符合业务语义。要点：显式写出来的意图（delete/default）正是五法则「齐全」的体现。

**第 3 题**。(a) 扩容搬迁按 `std::move_if_noexcept` 语义执行：**只有移动构造是 `noexcept` 时**，vector 才敢在搬家中使用它；否则退回拷贝。他的移动构造没标 noexcept，vector 无法保证「搬到一半抛异常」时旧数据还能保持完整，只能选择可回滚的拷贝。(b) 最少改动：给移动构造加 `noexcept`（它只交接指针，本来就不抛）。(c) 策略名 `std::move_if_noexcept`，为的是**强异常保证**：扩容失败时原容器内容完好无损。

**第 4 题**。参考步骤：① 保持 Debug 配置 F5 运行，崩溃时在 VS 异常对话框点「中断」；② 打开调用栈窗口（Ctrl+Alt+C），从栈顶逐帧向下双击，定位到你自己代码里发起这次非法读取的帧；③ 读异常对话框里的非法地址模式——`0xDDDDDDDD` 提示**释放后使用**（悬垂），`0xCDCDCDCD` 提示**分配后未初始化**（MSVC 调试堆毒化模式）；④ 在监视窗口查嫌疑指针的值，并与它「应该指向」的对象（如容器的 data/size）当前地址对比，确认指针已指向旧地址/被毒化内存；⑤ 向上追溯指针的获取点（如缓存了 `&vec[i]` 后又触发了 `push_back` 扩容搬家），确认失效时机；⑥ 修复（改存下标/ID 等）后回归验证。`0xDD…`：内存已释放后被调试堆填充的模式；`0xCD…`：堆内存已分配但从未初始化的模式。

**第 5 题**。未定义行为——不能依赖任何值。成员按**声明顺序**初始化：`slots_` 在前，它用 `size_ / 64` 初始化时 `size_` 还没被初始化，读的是不确定值（实测可能碰巧是 0 或垃圾）。修复方式一：调整声明顺序，把 `size_` 声明在 `slots_` 之前（依赖谁就把谁放前面）；方式二：不经过成员，直接用参数计算——`Arena(std::size_t n) : size_(n), slots_(n / 64) {}`。预防：给成员写默认成员初始化器（`std::size_t slots_ = 0;`），并打开 `-Wreorder` / MSVC `/w15038` 让编译器盯着。

**第 6 题**。`int a{3.14};` ❌（浮点→整数恒为收窄）；`char c{65};` ✅（常量且在 char 范围内）；`int x = 300; char d = x;` ✅（`=` 拷贝初始化不禁收窄，运行期截断——陷阱）；`int x = 300; char e{x};` ❌（`x` 不是常量表达式，编译器拒绝冒险）；`float g{0.1};` ✅（double 常量在 float 范围内，允许舍入）。

**第 7 题**。完整输出：

```text
构造 全局资源        ← main 之前，按定义顺序
构造 静态资源
构造 帧对象
构造 通道A
构造 通道B
析构 通道B           ← 内层作用域结束，逆序
析构 通道A
构造 函数内静态      ← static 局部变量：首次执行到声明处才构造
构造 帧尾对象
main 结束
析构 帧尾对象        ← main 内自动对象逆序
析构 函数内静态      ← 静态对象在 main 之后析构，逆构造序
析构 静态资源
析构 全局资源
```

两个易错点：`static Tracer s(...)` 不是在 main 开始时构造，而是**首次执行到那行**才构造，因此「构造 函数内静态」出现在两次通道析构之后；静态存储期对象在 `main` 返回**之后**、按构造的逆序析构。

**第 8 题**。(1) `take(a)` → ② `const Image&`（a 是左值，右值引用接不住）；(2) `take(std::move(a))` → ③ `Image&&`（xvalue）；(3) `take(Image(...))` → ③（prvalue 临时对象）；(4) ② ——`std::move(c)` 产生的是 **const** 右值，`Image&&` 承诺要掏空对方、绑不上 const，于是退回 `const Image&`：**这次「移动」实际是拷贝**（正是 1.5 节的 const 退化场景）。

**第 9 题**。链接阶段报**重复定义**错误（MSVC：LNK2005「already defined」；GCC/Clang：multiple definition）——两个翻译单元各自定义了全局变量 `g_frame`，违反 ODR（一个名字在整个程序只能有一个定义）。放进 `common.h` 会发生**完全相同**的事：头文件被两个 .cpp 包含，预处理后每个翻译单元各有一份定义，链接时照样撞车。正确做法：头文件里只放**声明** `extern int g_frame;`，定义（`int g_frame = 0;`）放在恰好一个 .cpp 里。（类的定义可以出现在多个翻译单元，前提是每份逐 token 相同——这与函数/变量的规则不同，是 1.1 节 ODR 的另一面。）

### 实践任务

**任务 1（自测资源类 Buffer）**。参考思路：先写「普通构造 + 析构」跑通 new/delete 配对，再逐个加拷贝构造（`new` 新内存 + 逐元素复制）、拷贝赋值（建议直接用 copy-and-swap：`Buffer& operator=(Buffer other)` + `swap`）、移动构造（窃取指针 + 置空源 + `noexcept`）、移动赋值（同法）。日志格式建议 `[拷贝构造] size=1024, 复制 4096 字节`，与 1.4 的 Image 同构，方便对照。验收要点：

1. 五个特殊成员函数齐全，移动相关两个标了 `noexcept`；
2. main 演练序列覆盖：拷贝构造、拷贝赋值、移动构造、移动赋值、自我赋值，每个成员的日志都出现过；
3. `assert` 验证拷贝独立性（改一方另一方不变）与移动后空壳（源 `data_ == nullptr` 且可安全析构）；
4. 结束时构造/析构日志次数配平、new/delete 字节数配平；自查拷贝赋值是否能通过自我赋值与异常安全两问（copy-and-swap 天然通过）。

**任务 2（法则标注）**。参考答案：

- **A（Color）**：rule of zero。四个 float，无资源，编译器生成的拷贝/移动/析构全部正确。
- **B（VertexBuffer）**：rule of five。`vertices_` 是类独占的堆内存：析构要 `delete[]`；拷贝要深拷（分配 + 复制 `count_` 个 float）；移动窃取指针并置空源（标 `noexcept`）；两个赋值建议 copy-and-swap。
- **C（InputState）**：rule of zero。纯数据（定长数组 + 两个 float），无资源。
- **D（NamedEntity）**：rule of zero 的一个特例——由于 `const unsigned id_` 的存在，**编译器把拷贝/移动赋值隐式删除**（const 成员不能被重新赋值），拷贝构造/移动构造仍自动生成。设计上要自觉：如果这类对象需要「赋值换内容」，就别用 const 成员（改用普通成员 + 约定不改）；如果 ID 不可变正是需求，那「只可构造、不可赋值」就是正确行为，注释写明即可。
- **E（HoverTarget）**：rule of zero，但要补一句设计注释：它持有的是**非拥有**裸指针（只观察、不管生死），类的拷贝/移动全部正确；风险不在「法则」而在悬垂——被指的 Particle 死了它不知道（1.11 的 HUD 崩溃就是它），所有权的系统解法在第 2 章展开。

单元测试（针对 B）验收要点：深拷贝后改一方数据另一方不变；移动后源对象 `vertices_ == nullptr` 且能安全析构；`b = b;` 安全；移动构造标了 `noexcept`（可选加分：把 B 放进 `std::vector` 做扩容实验，日志确认只移动不拷贝）。可选衔接：第 2 节数学库（Vec2/Vec3/Vec4/Mat4）就绪后逐类套用本练习——预期结论：全是纯 float 数据，全部 rule of zero。

**任务 3（调试器观察扩容）**。参考思路与验收要点：

1. 基线日志：B 段「构造 + 移动 + 空壳析构」、C 段扩容时旧元素全部「移动构造」、全程零「拷贝构造」——达到阶段 1 检验标准「vector 扩容日志显示资源类只被移动未被拷贝」；
2. 删除 `noexcept` 后复跑：C 段出现「拷贝构造」，恢复后消失——能口头解释 move_if_noexcept 策略；
3. 断点证据：移动构造内断点命中 ≥ N 次（N=扩容搬迁元素数），调用栈可见 `push_back` → 扩容内部帧的调用链，监视窗口可见旧/新缓冲区地址变化；
4. 加分项：不做 `reserve` 直接插三个元素的日志，与 reserve 版对比，说得出 reserve 省掉了哪一类操作（提前扩容，避免反复搬家）。

---

*本章完。下一章（第 2 章）我们把「对象死了以后」的确定性变成资源管理的万能模式：RAII 与智能指针——你在这里写过的每一个 `delete[]`，都会在那里找到更安全的归宿。*
