# 第 4 章｜工具链工程化：CMake 构建体系

欢迎来到第 4 章。第 3 章结尾我们说过：要把「这一路凑合用的编译命令（`cl /utf-8 /std:c++20 ...`、手敲头文件路径）升级成正经的 CMake 工程」——现在兑付。回想一下前三章你亲手忍受过的一切：每个示例都要重敲一遍完整的 `cl` 命令行；两个 `.cpp` 就得手拼链接顺序；头文件在别的目录就要手写 `/I 路径`；编译产物 `.obj` 散落在源码堆里；换一台机器，这一切从头再来一遍。**这些不是你的问题，是「没有构建系统」的问题。** 本章把它连根拔掉，同时偿还三笔旧账——第 1 章 1.1 埋下的「构建系统工程化」，第 3 章 3.9 承诺的「头文件怎么组织进构建工程」，以及第 3 章章末的这句预告本身。

先钉墙上一句话，它是本章的世界观：

> **CMake 不是编译器，是构建系统的生成器；工程里的一切依赖与选项都长在 target 上，不散落在命令行或 IDE 设置里。** 你写的 CMakeLists.txt 不是编译指令，是「生成工程文件」的说明书；真正干活的是你机器上的工具链，而 target 是让它们各就各位的组织单位。

前几章欠的账，本章全部结清：

| # | 前章承诺 | 兑付位置 |
| --- | --- | --- |
| ① | 第 1 章 1.1：「构建系统工程化将在第 4 章展开，这里只需要地图」 | 4.1（把「预处理/编译/链接」地图升级为「构建系统如何驱动这张地图」），4.2 深化 |
| ② | 第 3 章 3.9：「『头文件怎么组织进构建工程』第 4 章（CMake）展开」 | 4.4（`target_include_directories` 与属性传递），4.3 预备（目录布局） |
| ③ | 第 3 章章末：「把凑合用的编译命令升级成正经的 CMake 工程」 | 开篇接棒 + 4.1 的「cl 命令 → CMake 翻译表」+ 全章主线工程 |
| ④ | 第 3 章 3.9 的模板 LNK2019 现场 + 全章手敲命令 | 4.1（痛点清点）/ 4.4 + 深入专题（链接错误归因） |

本章的终点是一条工程铁律，和前几章的「零裸 `glDelete`」同级：**新环境克隆仓库后，`configure → build → test` 三条命令从零得到可运行的 demo 与全绿测试；往这个工程里加一个新目标，只是添加一个 target，而不是改动任何既有文件。** 这也是第 3 节渲染器项目 P1「CMake 工程化」里程碑的直接存货。

---

## 学习目标

对齐本领域阶段 4（工具链工程化：CMake 构建体系）的学习目标。读完本章，你应该能够：

1. **摆脱 IDE 一键工程黑盒**：说清 configure 与 build 两个阶段各自发生什么、生成的 `.sln`/`.vcxproj` 为什么是「生成物」而非源头；脱离 IDE，用三条命令完成从零构建。
2. **掌握 target 模型与属性传递**：`PUBLIC` / `PRIVATE` / `INTERFACE` 三键的语义与传播边界成为本能；能解释「为什么 demo 只链 resman 也能用 math」，并给任意一条链接错误归因到具体传递环节。
3. **驾驭多配置与编译选项挂接**：Debug/Release 在 Visual Studio 生成器下如何贯穿 build 与 ctest；C++20 基线、`/utf-8`、`/W4` 警告基线在 CMake 里的标准挂法（ch03 手敲参数的全部归宿）。
4. **接入第三方库**：FetchContent 与 find_package 两条路线的边界与选型；版本锁定纪律；imported target（`glfw::glfw`）为什么和你自己写的 target 是同一个模型。
5. **让「全绿」成为一条命令**：CTest 的测试注册与运行；多配置生成器下 `-C` 参数为什么不可省；会看「Not Run」与「Passed」背后的机制。
6. **产出可交接工程**：目录约定、显式源文件列举、`.gitignore`、README 三步说明；新同事（或第 6 节的你自己）克隆后三步内从零构建。第 6 节阶段 1「用 CMake 搭建项目」将直接复用本章产出。

> **环境约定（全章适用）**：Windows + Visual Studio 2022（MSVC 19.44 工具集，同 ch01–ch03）+ CMake ≥ 3.20（本章示例在 **CMake 4.2.1** 下逐条实测，`cmake --version` 可自查）。示例统一显式指定生成器：`-G "Visual Studio 17 2022" -A x64`；不写 `-G` 时 CMake 会自动选机器上最新的 VS（本机自动选中的是更新的 VS 18 2026——生成器随环境而异，教程一律显式指定保证可复现）。Ninja 未安装：Ninja 等价命令以侧栏给出，标注「依据官方文档口径、未在本机实测」。C++ 标准仍是 C++20（R5）；源文件统一存 UTF-8 编码，MSVC 下必须挂 `/utf-8`（4.5 有三组真实翻车现场）。所有报错节选均为本机真实输出（MSVC 19.44、中文语言包；行号与绝对路径有截断）。CMake 语法为跨平台通用写法（R6），游戏语境只体现在工程素材的命名与组织上。

---

## 分节正文

### 4.1 从手敲 cl 到 CMakeLists：构建系统到底替你驱动了什么

先把 ch03 攒下的痛点摆上台面——每一条你都应该亲痛过：

1. 每个 `.cpp` 示例都要重敲一遍完整命令行：`cl /utf-8 /std:c++20 /EHsc /W4 ch03_xxx.cpp`——漏一个参数，行为就悄悄变味；
2. 两个以上 `.cpp` 文件就要手写链接清单，顺序、输出名（`/Fe`）全靠人脑记；
3. 头文件不在当前目录就得 `/I some/path` 手拼，路径一多就是灾难；
4. `.obj` 中间产物散落在源码目录，和 `.cpp` 混在一起，仓库没法看；
5. 换台机器（或交给同事），以上全部靠口口相传，**不可复现**。

现在兑付还账 ①。回到第 1 章 1.1 的那张地图：一个源文件变成可执行程序，要经过**预处理 → 编译 → 链接**三站。当时说「构建系统工程化在第 4 章展开」——现在补全这一层：**构建系统就是驱动这张地图的自动化层**。它对每个源文件跑预处理和编译、把 `.obj` 收进一个统一的构建目录、在正确的时机调用链接器、并记住「哪个文件改过了、哪些可以不重编」。你手敲的 `cl` 命令行，本质上是在人工扮演这个角色。

那 CMake 又是什么？**构建系统的生成器**。构建系统（Make、Ninja、MSBuild……）需要一份「工程文件」来知道怎么干活：MSBuild 读 `.vcxproj`，Ninja 读 `build.ninja`。手写这些文件又繁琐又绑平台，于是 CMake 出场：你写一份平台无关的 `CMakeLists.txt`，CMake 读它，**生成**你机器上构建系统认识的工程文件。所以准确的分工是两阶段：

- **configure（配置）**：跑你的 CMakeLists.txt，检测编译器，生成工程文件（`.sln`/`.vcxproj` 或 `build.ninja`）；
- **build（构建）**：驱动底层构建系统（VS 生成器下就是 MSBuild）真编译、真链接。

看第一份工程。惯例：从 `hello, CMake!` 开始：

```text
hello_cmake/
├── CMakeLists.txt     # 工程的源头说明书
└── main.cpp
```

```cmake
# CMakeLists.txt —— 最小但完整的工程说明书
cmake_minimum_required(VERSION 3.20)   # 最低版本要求（本节末尾详谈这行为什么要命）
project(hello_cmake)                   # 工程名；同时检测并选定编译器
add_executable(hello main.cpp)         # 声明一个名叫 hello 的可执行目标，源文件是 main.cpp
```

```cpp
// main.cpp
#include <cstdio>

int main() {
    std::printf("hello, CMake!\n");
    return 0;
}
```

两条命令，从说明书到可执行文件（全部命令在工程根目录执行）：

```bat
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
```

`-S .` 源码目录是当前目录（CMakeLists.txt 所在处）；`-B build` 构建目录叫 `build`——**它现在还不存在，CMake 会创建它**；`-G` 选生成器，`-A x64` 选 64 位平台。这就是 configure 阶段，本机实测输出（节选）：

```text
-- The C compiler identification is MSVC 19.44.35222.0
-- The CXX compiler identification is MSVC 19.44.35222.0
-- Check for working CXX compiler: .../cl.exe - skipped
-- Detecting CXX compile features - done
-- Configuring done (3.1s)
-- Generating done (0.0s)
-- Build files have been written to: .../hello_cmake/build
```

（顺带：`-S/-B` 是 CMake 3.13+ 的现代写法；老教程里的 `mkdir build && cd build && cmake ..` 是同一件事的两步版——知道即可，新代码统一 `-S/-B`，少一次 `cd`。）

CMake 检测到 MSVC 19.44、验证编译器可用、生成了工程文件。接着第二条命令：

```bat
cmake --build build --config Release
```

「构建 `build` 目录里的工程，配置选 Release」。VS 生成器下这会调 MSBuild 真正编译链接：

```text
  main.cpp
  hello.vcxproj -> D:\...\hello_cmake\build\Release\hello.exe
```

```text
D:\...\hello_cmake> build\Release\hello.exe
hello, CMake!
```

从此，`cl` 命令行的每一个参数都有了固定的家。这张翻译表建议截图贴墙——它是 ch03 到 ch04 的完整换乘图（还账 ③ 的核心兑付）：

| ch03 的手敲参数/痛点 | CMake 里的家 | 落点小节 |
| --- | --- | --- |
| `cl /utf-8` | `add_compile_options(...)` | 4.5 |
| `/std:c++20` | `set(CMAKE_CXX_STANDARD 20)` + `..._REQUIRED ON` | 4.5 |
| `/W4` | 警告基线（`/W4` 或 `-Wall -Wextra`） | 4.5 |
| `/I ../include` 手敲路径 | `target_include_directories` | 4.4 |
| 命令行列多个 `.cpp`、`/Fe` 定名 | `add_executable(app a.cpp b.cpp)` | 本节、4.3 |
| `.obj` 散落源码树 | `build/` 目录统一收纳 | 4.2 |
| 每次全量手敲 | configure 一次，build 无数次，增量重编 | 4.2 |

**关于第一行，一个必须讲的血泪教学点。** `cmake_minimum_required(VERSION 3.20)` 里的版本号不是「我们用这么新的 CMake」，而是「**本工程最低需要这个版本的 CMake**」——老版本 CMake 读到这行会直接拒绝 configure。它还顺带锁定一组「政策」（policy，行为开关的集合，概念知道即可）。写多少合适？写你团队**实际最低可用**的版本。抄老教程写 `2.8` 或 `3.1` 会怎样？本机（CMake 4.2.1）实测：

```text
CMake Error at CMakeLists.txt:1 (cmake_minimum_required):
  Compatibility with CMake < 3.5 has been removed from CMake.

  Update the VERSION argument <min> value.  Or, use the <min>...<max> syntax
  to tell CMake that the project requires at least <min> but has been updated
  to work with policies introduced by <max> or earlier.

  Or, add -DCMAKE_POLICY_VERSION_MINIMUM=3.5 to try configuring anyway.


-- Configuring incomplete, errors occurred!
```

CMake 4.x 起彻底移除了对 3.5 以下版本的兼容——网上三年前的教程抄来的 `VERSION 2.8.12` 会当场爆炸。报错里提到的 `<min>...<max>` 区间语法（如 `3.20...3.28`，意为「最低 3.20、按最高 3.28 的行为适配」）是工程里常见的进阶写法，本章统一用单版本 `3.20` 保持简单。

### 4.2 生成器与 out-of-source 构建：configure、build，以及「CMake 不是编译器」

configure 完之后 `build/` 里多了什么？看一眼（节选）：

```text
build/
├── hello_cmake.sln              ← VS 解决方案（生成物！）
├── hello.vcxproj                ← 每个目标一个工程文件（生成物！）
├── ALL_BUILD.vcxproj            ← CMake 自带的「构建所有」辅助目标
├── ZERO_CHECK.vcxproj           ← CMake 自带的「CMakeLists 变了没？」自检目标
└── Release/
    └── hello.exe                ← 产物
```

（树取自 hello_cmake；主线工程的 build 树形状相同，只是每个子目录多出自己的 `.vcxproj`。）破解 IDE 黑盒的第一层：**你在 VS 里双击打开的 `.sln` 是生成物**——每次 CMakeLists.txt 变化，configure 都会重新生成它们。手改 `.sln`（加个源文件、改个选项）在下一次 configure 时被无声覆盖。源头永远只有一份：CMakeLists.txt。这也是「IDE 一键工程」的黑盒本质：绿色按钮背后跑的就是这套生成物，而 CMake 把说明书从 IDE 手里拿回到你手里的文本文件中。

两阶段职责再压实一遍：**configure 读 CMakeLists.txt 生成工程文件，不碰任何 `.cpp`；build 驱动 MSBuild 真编译真链接**。由此立刻推出一条工程纪律——**out-of-source 构建**：构建产物全部住在 `build/`，源码树永远干净；`build/` 可以随时整个删掉重建而不心疼（它 100% 可再生，检验标准 ①「新环境三步重建」的机制根基就在这）。反例是 in-source 构建（`cmake -S . -B .`）：产物与源码混居，删也不是留也不是，仓库直接报废。别试，记结论。

**生成器决定了「配置」怎么用——这是本章第一个高翻车点，必须表格化：**

| | Visual Studio 生成器（多配置） | Ninja / Make（单配置） |
| --- | --- | --- |
| configure 时 | 不选配置 | 必须选：`-DCMAKE_BUILD_TYPE=Release` |
| build 时 | `--config Debug/Release` 任选 | 无配置概念，直接 `cmake --build build` |
| 产物位置 | `build/Debug/`、`build/Release/` 分目录 | 单一目录（configure 时定死） |
| `CMAKE_BUILD_TYPE` | **无效**（被 `--config` 取代） | **有效**（就是靠它） |
| ctest | `ctest -C Release` | `ctest`（配置已定） |

本章主线用 VS 生成器（本机在位），所以「配置」贯穿 build 与 ctest——4.6 会亲手踩一次忘带 `-C` 的坑。若你的机器装了 Ninja，等价写法是：

> **侧栏（Ninja 等价写法，依据官方文档口径、未在本机实测）**：`cmake -S . -B build-ninja -G Ninja -DCMAKE_BUILD_TYPE=Release` 然后 `cmake --build build-ninja`。Ninja 更轻快、产物单一目录，是 CI 与大工程的常见选择；IDE 体验则 VS 生成器更顺。两者读同一份 CMakeLists.txt——这正是「构建系统的生成器」的收益：说明书一份，工程文件随机器生成。

最后看构建系统的记忆能力。改一个 `.cpp`，再 build：

```text
（touch libs/math/src/vec3.cpp 后）
  vec3.cpp                                    ← 只有它重编了
  math.vcxproj -> ...\libs\math\Release\math.lib      ← 重链库
  demo.vcxproj -> ...\apps\demo\Release\demo.exe      ← 重链可执行文件
```

`main.cpp` 没动就**没有重编**——构建系统跟踪着文件依赖，改哪个编哪个。什么都没改再 build 一次：所有目标「已是最新」，除了检查几乎什么都不做。这就是 4.1 翻译表最后一行「configure 一次、build 无数次」的机制。而源码树与产物树的对应关系也在此定型：`libs/math/src/vec3.cpp` 的产物住在 `build/libs/math/`，`apps/demo/main.cpp` 的产物住在 `build/apps/demo/`——**build 目录是源码树的镜像 + 配置名**，找产物不再靠猜。

顺带分清两种「改动」的待遇：改 `.cpp`/`.hpp`，build 自己就处理（增量重编）；改 **CMakeLists.txt**，下一次 build 开头的 `Checking Build System` 会发现说明书变了，自动重跑 configure 再继续编——这就是 `ZERO_CHECK` 目标存在的意义。唯一需要你手动重新 configure 的场景：新增/删除**源文件**（清单问题，4.8 详谈）或改了 cache 变量。

「随时可删」也别只听我说的，实测一遍——把 `build/` 整个删掉，同样的三条命令再来一轮，产物与测试一模一样（本章 4.8 有全流程实录）。这正是「`build/` 100% 可再生」的现场版：它不在版本库里、不携带任何秘密，丢了毫不可惜。顺带一句构建并行：`cmake --build build --parallel` 会让多个编译进程同时跑（不同 `.cpp` 之间天然互不依赖），日常构建建议加上；构建并行是「多个编译进程」的事，与本书后面要讲的程序内并发是两个世界，别混淆。

### 4.3 库目标与可执行目标拆分：渲染器工程的正确形状

hello_cmake 全程单文件，真实工程长不了那样。从本节起，全章的主线工程 `mini_engine/` 登场——一个教学缩小版「渲染器前哨」：数学库（ch03 任务 2 的 Vec3 骨架）+ 资源管理（ch03 任务 1 的 ResourceManager）+ demo 应用。它刻意贴合 P1 渲染器「数学库 + 资源管理 + 应用」的三角，但**不碰任何图形 API**——第 3 节真渲染器启动后照此结构迁移即可（回指一句，不展开）。

为什么第一件事是「拆库」而不是「堆文件」？三个理由，每个都在后面有自己的戏：

1. **编译边界**：改 demo 的 `main.cpp` 不必重编 math 库（4.2 的增量重编在工程层的体现）；
2. **测试复用**：同一个库目标可以被测试可执行文件链接（4.6 的直接伏笔）；
3. **依赖声明**：谁用谁、谁的服务不外泄，写在 target 声明里而不是口口相传（4.4 的主角）。

v1 形状——一个静态库 + 一个可执行：

```text
mini_engine/
├── CMakeLists.txt                 # 顶层说明书
├── libs/
│   └── math/                      # 静态库
│       ├── CMakeLists.txt
│       ├── include/math/vec3.hpp  # 对外头文件（只有它给别人看）
│       └── src/vec3.cpp           # 实现（库的私产）
└── apps/
    └── demo/
        ├── CMakeLists.txt
        └── main.cpp
```

先看目录约定，这是还账 ② 的第一半（「头文件怎么组织」的物理层）：**`include/<库名>/` 只放对外头文件，`src/` 放实现**。`include/math/vec3.hpp` 里多包一层 `math/` 目录不是强迫症——它让 `#include "math/vec3.hpp"` 自带库名，十个库的头文件也不会互相撞名。头文件里只有声明，实现（链接实体）住在 `vec3.cpp`：

```cpp
// libs/math/include/math/vec3.hpp
#pragma once
// math/vec3.hpp —— 教学缩小版数学库（ch03 任务 2 的 Vec3 骨架微调版）
// dot/length 放进 .cpp：让库里有真实的「链接实体」，链接错误的实验才有实体可炸
namespace math {

struct Vec3 {
    float x{}, y{}, z{};
};

float dot(const Vec3& a, const Vec3& b);      // 点积：实现在 vec3.cpp
float length(const Vec3& v);                  // 模长：实现在 vec3.cpp
Vec3 operator+(const Vec3& a, const Vec3& b); // 向量加法

} // namespace math
```

```cpp
// libs/math/src/vec3.cpp
#include "math/vec3.hpp"
#include <cmath>

namespace math {

float dot(const Vec3& a, const Vec3& b) {
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

float length(const Vec3& v) {
    return std::sqrt(dot(v, v));
}

Vec3 operator+(const Vec3& a, const Vec3& b) {
    return Vec3{a.x + b.x, a.y + b.y, a.z + b.z};
}

} // namespace math
```

三份 CMakeLists——顶层、库、应用，注意每份只管自己的事：

```cmake
# CMakeLists.txt（顶层）
cmake_minimum_required(VERSION 3.20)
project(mini_engine)

# C++20 基线（R5 的 CMake 化；三件套缺一不可，4.5 详讲）
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ch03 每次手敲的 /utf-8 在 CMake 里的家（含义与报错现场 4.5 展开）
add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

add_subdirectory(libs/math)
add_subdirectory(apps/demo)
```

```cmake
# libs/math/CMakeLists.txt
add_library(math STATIC            # 静态库目标，名字叫 math
    src/vec3.cpp
)
# 对外头文件的搜索路径。PUBLIC 的含义 4.4 展开：先记住「库的使用者也需要它」
target_include_directories(math PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

```cmake
# apps/demo/CMakeLists.txt（v1 版）
add_executable(demo main.cpp)
# 链接 math。PRIVATE 的含义同样 4.4 展开
target_link_libraries(demo PRIVATE math)
```

```cpp
// apps/demo/main.cpp（v1 版）
#include <cstdio>
#include "math/vec3.hpp"

int main() {
    using math::Vec3;
    // 三个粒子，摆成一个直角三角形看看点积与模长
    Vec3 a{3.0f, 0.0f, 0.0f};     // 在 x 轴上
    Vec3 b{0.0f, 4.0f, 0.0f};     // 在 y 轴上
    Vec3 c = a + b;               // (3, 4, 0)

    std::printf("dot(a, b)  = %.1f   （互相垂直，应为 0）\n", math::dot(a, b));
    std::printf("length(c)  = %.1f   （勾股定理，应为 5）\n", math::length(c));
    std::printf("dot(c, c)  = %.1f   （模的平方，应为 25）\n", math::dot(c, c));
    return 0;
}
```

配置、构建、运行，本机实录：

```text
> cmake -S . -B build -G "Visual Studio 17 2022" -A x64
-- Configuring done (3.2s)
-- Generating done (0.0s)
-- Build files have been written to: .../mini_engine/build

> cmake --build build --config Release
  vec3.cpp
  main.cpp
  demo.vcxproj -> ...\build\apps\demo\Release\demo.exe

> build\apps\demo\Release\demo.exe
dot(a, b)  = 0.0   （互相垂直，应为 0）
length(c)  = 5.0   （勾股定理，应为 5）
dot(c, c)  = 25.0   （模的平方，应为 25）
```

三个细节值得停一停：① 产物在 `build\apps\demo\Release\`——VS 生成器把产物放在「目标在源码树中的镜像位置 + 配置名」下，所以**从源码树结构就能预测产物位置**；② `add_library(math STATIC)` 的 `STATIC` 指静态库（`.lib` 在链接时拷贝进 exe，简单直接，教学与中小工程默认选它；`SHARED` 动态库与 `MODULE` 插件属进阶话题，需要时查官方文档）。另注意一个小坑：`add_library(math ...)` 里的 `math` 是 **target 名**，只在 CMake 世界里使用（`target_link_libraries` 写它）；落盘的文件叫 `math.lib`，是构建系统按平台惯例生成的——在 CMake 里永远写 target 名，不要写文件名；③ 顶层那个带 `$<...>` 花括号怪东西的 `add_compile_options` 行你先当咒语照抄，4.5 逐字拆解它。

### 4.4 target 属性传递：PUBLIC / PRIVATE / INTERFACE——现代 CMake 的心脏

v1 里你照抄了两行带「谜语」的代码：`target_include_directories(math PUBLIC ...)` 和 `target_link_libraries(demo PRIVATE math)`。本节把谜底揭开——这是全章最重要的一节。

先立模型。每个 target 除了「怎么构建自己」，还随身携带一份**使用要求（usage requirements）**：「**想正确使用我，你还需要什么**」——头文件搜索路径、宏定义、要链接的库……。编译 math 自己需要的和它的使用者需要的，**是两份不同的清单**：`vec3.cpp` 编译时需要 `<cmath>`（自己的事）；demo 编译时需要找到 `math/vec3.hpp`（使用者的事）。三键就是控制「每条要求装进哪份清单」的开关：

| 键 | 装进自己？ | 传给使用者？ | 一句话口诀 |
| --- | --- | --- | --- |
| `PRIVATE` | ✓ | ✗ | 自己用，不外传（实现细节） |
| `INTERFACE` | ✗ | ✓ | 只外传，自己不用（纯头文件库的一切） |
| `PUBLIC` | ✓ | ✓ | 自己用，也传给使用者（对外接口的需要） |

库作者视角的决策法：这条要求出现在**我的对外头文件**里吗？出现（使用者编译时也要它）→ `PUBLIC`；只在我的 `.cpp` 里用 → `PRIVATE`；我是纯头文件库，一切都在使用者的编译里 → `INTERFACE`。

现在给主线工程加装第二个库。`resman` 是 ch03 任务 1 的 `ResourceManager` 定稿——**模板库，全部实现在头文件里**（3.9 的「定义放头文件」在工程层的直接推论：没有 `.cpp` 就没有东西可编，所以它是**INTERFACE 库**）：

```text
mini_engine/（v2 新增部分）
└── libs/
    └── resman/
        ├── CMakeLists.txt
        └── include/resman/
            ├── resource_manager.hpp
            └── memory_loader.hpp
```

```cmake
# libs/resman/CMakeLists.txt
# INTERFACE 库：自己不编译任何 .cpp（模板库——定义必须住头文件，ch03 3.9）
# 但它对「使用者」的要求一应俱全，且全部自动传递
add_library(resman INTERFACE)
target_include_directories(resman INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
# 用 resman 就必然用到 math：一句话让传递性替你接线
target_link_libraries(resman INTERFACE math)
```

头文件全文（`resource_manager.hpp` + `memory_loader.hpp`）就是 ch03 任务 1 定稿的原样搬用，此处不再重复占据篇幅；实践任务一节有完整素材清单。顶层 CMakeLists 只需加一行 `add_subdirectory(libs/resman)`，demo 改成——

```cmake
# apps/demo/CMakeLists.txt（v2 版）
add_executable(demo main.cpp)
# 只链 resman：math 经由 resman 的 INTERFACE 传递进来（本节的高潮）
target_link_libraries(demo PRIVATE resman)
```

本机实测（demo 的 main.cpp 同时用了 `math::length` 与 resman 的去重加载）：

```text
> cmake --build build --config Release
  main.cpp
  demo.vcxproj -> ...\build\apps\demo\Release\demo.exe

> build\apps\demo\Release\demo.exe
length(a+b) = 5.0
brick pixels: 12 34 56 | load_count = 2（两次 brick 只算了 1 次）
```

demo 从头到尾只链接了 `resman` 一个名字，却同时用上了 math 的 `length` 和 resman 的缓存。**传递性**是本节的高潮：resman 的 INTERFACE 要求「连着 math」一起传给 demo——依赖图自动接线。不信这张图只存在于文字里？CMake 能把它画出来（`cmake --graphviz=deps.dot -S . -B build`，生成 Graphviz 格式的依赖图，本机实录节点与边）：

```text
demo（Executable，蛋形）
  └─ resman（Interface Library，五边形）
       └─ math（Static Library，八边形）      // resman -> math，虚线 = Interface 边
```

`resman → math` 那条**虚线**正是「INTERFACE 要求」的图形化——虚线语义＝只传要求、不产生链接实体，和三键表一一对得上。这也是实践任务 2 的自查手段之一。

还有一层「眼见为实」：usage requirements 落到地上就是编译器命令行里的一两个参数。用 `cmake --build build --config Debug --verbose` 看 demo 的编译命令（本机节选）：

```text
cl ... /I"...\mini_engine\libs\math\include" /Zi /nologo /W4 ... /std:c++20 ...
```

那个 `/I...libs\math\include` 就是 `target_include_directories(math PUBLIC ...)` 经传递链送达的实体——4.1 翻译表里「`/I` 手敲路径」的自动化替身。这就是检验标准 ③「新增目标不动既有结构」的能量来源。这个模型你其实很快会再见：4.7 的 `glfw::glfw` 这样的第三方目标，用的就是同一套 usage requirements（同一颗心脏，第三次跳动）。

**光看正确版学不会敬畏，上两个破坏实验**——都是一行改动，都值得你亲手做一遍。

**实验 (a)：把传递改成私藏。** `libs/math/CMakeLists.txt` 里一个词，`PUBLIC` → `PRIVATE`：

```cmake
target_include_directories(math PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)   # ⚠ 破坏点
```

```text
> cmake --build build --config Release
  main.cpp
...\apps\demo\main.cpp(4,10): error C1083: 无法打开包括文件: “math/vec3.hpp”:
  No such file or directory [...\build\apps\demo\demo.vcxproj]
```

demo 找不到头文件了——math 的 include 路径变成了「自己编 `.cpp` 时才用」的私产，不再传给使用者。注意这发生在**编译期**（C1083 是编译错误）：头文件搜索路径属于编译阶段。

**实验 (b)：把「能编译」当「没问题」。** 更隐蔽也更真实的错误：有人觉得「链接线断了怕什么，把 include 路径手动补上不就完了」——在 resman 里手拼 math 的 include 路径，同时把 `target_link_libraries(resman INTERFACE math)` 这行删掉：

```cmake
# libs/resman/CMakeLists.txt（⚠ 破坏版）
add_library(resman INTERFACE)
target_include_directories(resman INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../math/include)   # ⚠ 手拼 include 路径
# target_link_libraries(resman INTERFACE math)      # ⚠ 链接线被「清理」掉了
```

编译顺利通过（头文件找得到了！），然后在**链接期**爆炸：

```text
> cmake --build build --config Release
  main.cpp
main.obj : error LNK2019: 无法解析的外部符号 "float __cdecl math::length(struct math::Vec3 const &)"
  (?length@math@@YAMAEBUVec3@1@@Z)，函数 main 中引用了该符号 [...\demo.vcxproj]
main.obj : error LNK2019: 无法解析的外部符号 "struct math::Vec3 __cdecl math::operator+(struct math::Vec3 const &,struct math::Vec3 const &)"
  (??Hmath@@YA?AUVec3@0@AEBU10@0@Z)，函数 main 中引用了该符号 [...\demo.vcxproj]
...\demo.exe : fatal error LNK1120: 2 个无法解析的外部命令 [...\demo.vcxproj]
```

两个实验合起来，就是「链接错误归因」的雏形：**先分层——编译错（C 开头）还是链接错（LNK 开头）？编译错找头文件与宏，链接错找 target 链接线。** 简版三步法：① 看报错里缺的符号名（如 `?length@math@@...`，`math::length` 的修饰名）；② 判断这个符号该由哪个 target 提供；③ 检查那个 target 与使用者之间的传递声明（谁该 `PUBLIC`/`INTERFACE` 却没写）。完整解剖——包括用工具亲眼看「库里到底有没有那个符号」——放本章深入专题，那里会同时结清 ch01 1.1 的 LNK2019/LNK2005 名词与 ch03 3.9 的模板版 LNK2019 两笔旧账。

现在正式还账 ②。第 3 章 3.9 的原话：「头文件怎么组织进构建工程，第 4 章（CMake）展开」——答案分两半：**物理层**，`include/<库名>/` + `src/` 的目录约定（4.3）；**逻辑层**，头文件搜索路径不靠 `/I` 手拼、不靠环境变量，而是作为 target 的 usage requirement 按三键声明、沿依赖图自动传播（本节）。模板库「没有 `.cpp`」的这一特性，还决定了它在工程里的形态——INTERFACE 库，一根 `.cpp` 都不用编，要求却一分不少。

### 4.5 Debug/Release 多配置与编译选项挂接：警告基线、/utf-8 与 C++20

v1/v2 的顶层 CMakeLists 里你照抄了两组选项，本节逐行拆解，并把 ch03 翻译表剩下的三行（`/std:c++20`、`/W4`、`/utf-8`）全部落地。

先说「配置」。4.2 的表说过：VS 生成器是多配置——同一个 build 目录里，`--config Debug` 与 `--config Release` 各建各的、产物分目录：

```text
build\apps\demo\Debug\demo.exe      ← /Od 零优化 + /RTC1 运行时检查 + 调试信息，给调试器用
build\apps\demo\Release\demo.exe    ← /O2 全优化 + NDEBUG，给玩家用
```

双配置构建实录（本机）：

```text
> cmake --build build --config Debug
  vec3.cpp   main.cpp   test_vec3.cpp   test_resman.cpp   ...
  demo.vcxproj -> ...\build\apps\demo\Debug\demo.exe

> cmake --build build --config Release
  demo.vcxproj -> ...\build\apps\demo\Release\demo.exe
```

（注意 Debug 那次把四个源文件全部重编了——这是首次构建 Debug 配置；两个配置的中间产物互不相干，各自缓存各自的增量。）

且「配置」不止贯穿 build——4.6 会看到 ctest 也要 `-C`。两个配置到底差在哪？别背清单，看编译器实际收到的命令行（`cmake --build build --config ... --verbose` 的实测节选，只保留差异项）：

```text
Debug   : cl ... /Zi /Od /Ob0 /RTC1 /MDd /std:c++20 /W4 ...
Release : cl ... /O2 /Ob2 /DNDEBUG /MD  /std:c++20 /W4 ...
```

- `/Od`（禁止优化）vs `/O2 /Ob2`（全优化加内联）——这就是「Debug 下测性能数据全废」的直接原因；
- `/RTC1`（运行时检查）、`/MDd`（调试运行库）只存在于 Debug——换来的慢换的是调试器里的精确报错；
- `/DNDEBUG` 只存在于 Release——它会让 `assert()` 整体失效（第 1 章 1.8 讲过 assert 是调试工具）；所以「只在 assert 里做副作用」的 bug，Release 下才现形；
- 你挂的基线两份里都在：`/std:c++20`、`/W4`——选项挂对了地方，配置怎么切都带着。

回扣一句 Stages 的老话「在 Debug 配置下测性能，数据全部作废」：**机制出处**就是上面这行 `/Od` 外加迭代器调试层，同一份代码 Debug 与 Release 能差出一个数量级——性能测量方法论归第 6 章，本章只负责让你永远不再混淆这两个目录里的两个 exe。

**C++20 基线三件套**（顶层，全工程生效）：

```cmake
set(CMAKE_CXX_STANDARD 20)           # 等价 /std:c++20：要 C++20
set(CMAKE_CXX_STANDARD_REQUIRED ON)  # 不许静默回退：编译器不支持就直接报错
set(CMAKE_CXX_EXTENSIONS OFF)        # 要标准 C++，不要编译器扩展（-std=c++20 而非 gnu++20）
```

第二行是灵魂：没有它，不支持 C++20 的老编译器会**静默回退**到旧标准，然后你收到一堆莫名其妙的语法错误。`REQUIRED ON` 把「支持不了」提前到 configure 期明说。

**`/utf-8` 与三组真实翻车现场。** ch03 环境约定框说过：源文件含中文，MSVC 必须 `/utf-8`。当时是纪律，现在讲机制：MSVC 对没有 BOM 的源文件默认按**系统本地代码页**读（简体中文 Windows 是 GBK/936），而现代编辑器默认存 **UTF-8**。两边不商量，就靠运气。运气好是什么样？一个含中文注释的 UTF-8 文件按 GBK 读，注释区被读成乱码但不碰语法——**编译通过，什么都没发生**。运气坏呢？本机真实实验：把 mini_engine 顶层的 `/utf-8` 那行删掉重新构建：

```text
> cmake --build build --config Release
  vec3.cpp
...\include\math\vec3.hpp(1,1): warning C4819: 该文件包含不能在当前代码页(936)中表示的字符。
  请将该文件保存为 Unicode 格式以防止数据丢失 [...\math.vcxproj]
...\include\math\vec3.hpp(13,1): error C2059: 语法错误:“}” [...\math.vcxproj]
...\include\math\vec3.hpp(13,1): error C2143: 语法错误: 缺少“;”(在“}”的前面) [...\math.vcxproj]
```

没改任何源码，只删了一行 CMake——编译就炸了。机制：UTF-8 的中文字符占 3 字节，按 GBK 两字节一组去切，某些字节组合会把**后面的标点符号吞进乱码字符里**（GBK 的第二字节可以落到 ASCII 区），头文件里某个 `;` 或换行被吃掉，语法结构塌方——报错位置（第 13 行的 `}`）与真凶（前面注释的乱码）隔了十万八千里，这是它臭名昭著的原因。还有第三种排列：从老编辑器/老同事手里接到 **GBK 编码**的源文件，再挂上 `/utf-8`（按 UTF-8 读 GBK 字节），本机实测收获 30 条警告：

```text
main_gbk.cpp(1): warning C4828: 文件包含在偏移 0x16 处开始的字符，
  该字符在当前源字符集中无效(代码页 65001)。
（……共 30 条 C4828）
```

（老工具链上这个场景常直接升级为 `error C2001: 常量中有换行符`——字符串字面量被误读吞掉引号；本机 19.44 表现为 C4828 警告风暴。方向相反的两组现场，同一根病根。）工程解法两条，缺一不可：**源文件统一存 UTF-8**（编辑器设置一次）；**CMake 里显式挂 `/utf-8`**，把「读什么编码」从机器本地说了算变成工程说了算：

```cmake
# 顶层 CMakeLists.txt —— 只对 MSVC 生效；$<...> 是「生成器表达式」，
# configure 期按条件展开成真值。这里读作：若编译器 ID 是 MSVC，追加 /utf-8 选项
add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
```

**警告基线 `/W4`。** ch03 每次手敲的 `/W4` 同样值得挂进 CMake——它在你写下代码的当下就抓问题（而不是等运行时崩溃）。跨编译器写法：

```cmake
# 警告基线：MSVC 用 /W4，GCC/Clang 用 -Wall -Wextra
if(MSVC)
    add_compile_options(/W4)
else()
    add_compile_options(-Wall -Wextra)
endif()
```

（进阶写法是全用生成器表达式：`add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/W4>" "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>>:-Wall;-Wextra>")`——效果相同，选可读性。老项目里你还会见到 `set(CMAKE_CXX_FLAGS ...)` 直接拼字符串 flags 的写法：能跑，但绕开了 target 模型、难以按目标差异化，新工程别学。）基线工作的证据，本机实测：故意在 demo 里塞一个未引用变量——

```text
...\apps\demo\main.cpp(9,9): warning C4189: “unused”: 局部变量已初始化但不引用
  [...\build\apps\demo\demo.vcxproj]
```

删掉变量，警告归零。**Debug 专属选项**顺理成章：给配置条件再加一层，只有 Debug 配置才追加：

```cmake
# 例：仅 MSVC 的 Debug 配置开启 /JMC（Just My Code 调试支持）
add_compile_options("$<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Debug>>:/JMC>")
```

`$<CONFIG:Debug>` 在 Release 配置下展开为假、整条选项消失——这就是「按配置挂选项」的标准姿势。生成器表达式值得单独记一张迷你速查（本章用到的全部）：

| 表达式 | 何时为真/展开为什么 | 本章用途 |
| --- | --- | --- |
| `$<CXX_COMPILER_ID:MSVC>` | 编译器是 MSVC | 挂 `/utf-8`、`/W4` |
| `$<CONFIG:Debug>` | 当前配置是 Debug | Debug 专属选项 |
| `$<AND:...>` / `$<NOT:...>` | 布尔组合 | 「MSVC 且 Debug」这种复合条件 |

至此 4.1 翻译表全部兑付：`/std:c++20`、`/W4`、`/utf-8` 都在 CMake 里安了家，configure 一次、处处生效、换机可复现。

### 4.6 CTest：让「全绿」成为一条命令

测试基础设施的哲学朴素到让人失望：**一个测试就是一个可执行文件，退出码 0 = 通过，非 0 = 失败**。CTest 负责的只是「登记有哪些测试、跑它们、汇总报告」。而 4.3 拆库的收益在这里兑现：`test_vec3` 链 `math`、`test_resman` 链 `resman`——**同一个库目标，被应用和测试两个消费者复用**，谁也不用拷谁的源码。

v3 新增 `tests/` 子目录：

```cmake
# tests/CMakeLists.txt
# 测试就是「又一个可执行目标」——复用 4.3 拆出来的库（拆库收益在此兑现）
add_executable(test_vec3 test_vec3.cpp)
target_link_libraries(test_vec3 PRIVATE math)
add_test(NAME vec3_properties COMMAND test_vec3)      # 登记：名字 + 怎么跑

add_executable(test_resman test_resman.cpp)
target_link_libraries(test_resman PRIVATE resman)
add_test(NAME resman_dedup COMMAND test_resman)
```

顶层再加两行（`enable_testing()` 必须在**顶层** CMakeLists 调用——ctest 到顶层 build 目录找测试清单）：

```cmake
# CMakeLists.txt（顶层，追加在 add_subdirectory 之后）
enable_testing()
add_subdirectory(tests)
```

测试正文不引任何框架（gtest 属实践任务的扩展层），返回码约定 + 手写断言足矣。`test_vec3` 沿用 ch03 任务 2 的性质测试（正交点积为零、交换律、`length(zero)==0`、勾股 3-4-5）；`test_resman` 是 ch03 任务 1 的验收点（同 Key 二次加载 `load_count == 1`、两个 Handle 指向同一对象）：

```cpp
// tests/test_vec3.cpp
#include <cmath>
#include <cstdio>
#include "math/vec3.hpp"

namespace {
int failures = 0;
void expect_near(const char* what, float got, float want, float eps = 1e-5f) {
    if (std::fabs(got - want) > eps) {
        std::printf("  FAIL %s: got %.6f want %.6f\n", what, got, want);
        ++failures;
    }
}
} // namespace

int main() {
    using math::Vec3;
    expect_near("dot 正交为零",    math::dot(Vec3{1, 0, 0}, Vec3{0, 1, 0}), 0.0f);
    expect_near("dot 交换律",      math::dot(Vec3{1, 2, 3}, Vec3{4, 5, 6}),
                                     math::dot(Vec3{4, 5, 6}, Vec3{1, 2, 3}));
    expect_near("length(zero)==0", math::length(Vec3{}), 0.0f);
    expect_near("勾股 3-4-5",      math::length(Vec3{3, 4, 0}), 5.0f);
    if (failures == 0) { std::printf("test_vec3: all green\n"); return 0; }
    std::printf("test_vec3: %d failure(s)\n", failures);
    return 1;   // CTest 的约定：退出码非 0 = 测试失败
}
```

```cpp
// tests/test_resman.cpp
#include <cstdio>
#include "resman/resource_manager.hpp"
#include "resman/memory_loader.hpp"

int main() {
    resman::ResourceManager<resman::TextureData> rm;
    resman::MemoryLoader loader({{"brick.png", "12 34 56"}});
    auto h1 = rm.load(loader, "brick.png");
    auto h2 = rm.load(loader, "brick.png");   // 同一个 Key 二次加载

    int failures = 0;
    if (rm.load_count() != 1) {   // 去重验收点：只允许真正 load 一次
        std::printf("  FAIL 去重失效: load_count=%u\n", rm.load_count()); ++failures;
    }
    if (&rm.get(h1) != &rm.get(h2)) {   // 两个 Handle 必须指向同一个对象
        std::printf("  FAIL Handle 未命中同一缓存对象\n"); ++failures;
    }
    if (rm.get(h1).pixels != "12 34 56") {
        std::printf("  FAIL 像素内容不符\n"); ++failures;
    }
    if (failures == 0) { std::printf("test_resman: all green\n"); return 0; }
    return 1;
}
```

跑起来。**多配置生成器下，ctest 必须带 `-C` 选配置**：

```text
> ctest --test-dir build -C Release
    Start 1: vec3_properties
1/2 Test #1: vec3_properties ..................   Passed    0.02 sec
    Start 2: resman_dedup
2/2 Test #2: resman_dedup .....................   Passed    0.26 sec

100% tests passed, 0 tests failed out of 2

Total Test time (real) =   0.45 sec
```

现在亲手踩一次坑（本机实录）：不带 `-C` 会怎样？

```text
> ctest --test-dir build
Test project .../mini_engine/build
    Start 1: vec3_properties
Test not available without configuration.  (Missing "-C <config>"?)
1/2 Test #1: vec3_properties ..................***Not Run   0.00 sec
    Start 2: resman_dedup
Test not available without configuration.  (Missing "-C <config>"?)
2/2 Test #2: resman_dedup .....................***Not Run   0.00 sec

0% tests passed, 2 tests failed out of 2

Total Test time (real) =   0.01 sec
```

三个细节都值得记：① 现象是 **Not Run**——不是「测试失败」，是「多配置生成器不知道你想跑哪个配置，索性一个都不跑」；② CTest 的提示其实很贴心，括号里直接告诉你缺什么；③ 退出码非 0（本机实测为 8）——脚本/CI 里判断「是否全绿」要看退出码，别肉眼扫输出。两个日常好用的参数：`ctest -N` 只列出测试名不运行（本机实录：`Test #1: vec3_properties`、`Test #2: resman_dedup` 两行，方便确认注册是否生效）；`ctest -C Release --output-on-failure` 在失败时打印测试的完整输出（成功时安静）——调试失败测试时的首选组合。顺带认识惯例开关：官方 CTest 模块约定用 `BUILD_TESTING` 选项控制「要不要注册测试」（`option(BUILD_TESTING ...)` + `if(BUILD_TESTING)` 包住测试注册），CI 上可以一关全关；本章从简直接 `enable_testing()`，知道惯例即可。修 bug 的标准节奏：先造一个失败的测试（红），修到全绿，留下记录——实践任务会要求你贴这条「红→绿」记录。

### 4.7 第三方库接入：FetchContent 与 find_package

自己写的库管好了，别人写的库怎么进来？CMake 给两条正路，先划边界：

- **FetchContent**：configure 期把依赖的**源码**拉下来，作为子工程并入你的构建——它和你自己的 target 同处一个依赖图，一起编译。
- **find_package**：依赖已经**预装/预编译**在你机器上（vcpkg 装的、系统包管理器装的、手动装好的），CMake 去找到它，给你一个现成的 **imported target**（如 `glfw::glfw`）。

选型直觉：小而编译快的库、需要锁死版本、或不想让「装环境」挡住新同事——FetchContent；体量大编译慢（如 LLVM）、或团队有统一的预装环境——find_package。Windows 现实工程里 vcpkg（配合 CMake toolchain 文件）是 find_package 路线的主流包管理器，延伸资源有链接。两者不是二选一，大工程常混用。

**FetchContent 的机制，先在本地真实验证**（不碰网络，用 `SOURCE_DIR` 指向一个现成目录——它和 `GIT_REPOSITORY` 拉取走的是同一条 `MakeAvailable` 之路）：

```text
fetch_demo/
├── CMakeLists.txt
├── main.cpp
└── libs/fake_glfw/                 # 一个「假装是 glfw」的本地第三方库
    ├── CMakeLists.txt
    └── include/fake_glfw/glfw.h
```

```cmake
# libs/fake_glfw/CMakeLists.txt —— 第三方库作者的那一侧
add_library(fake_glfw INTERFACE)
add_library(fake_glfw::glfw ALIAS fake_glfw)   # 命名空间::名字 的目标别名（惯例）
target_include_directories(fake_glfw INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

```cmake
# CMakeLists.txt —— 使用者这一侧
add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

include(FetchContent)
FetchContent_Declare(fake_glfw
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/fake_glfw   # 本地演示；真实场景换 GIT_REPOSITORY/GIT_TAG
)
FetchContent_MakeAvailable(fake_glfw)    # = 「拉下来 + add_subdirectory」一步到位

add_executable(main main.cpp)
target_link_libraries(main PRIVATE fake_glfw::glfw)
```

（`main.cpp` 就一行 `fake_glfw::init()` 打印，从略。注意这个小工程也挂了 `/utf-8`——4.5 的纪律面前，三行的小工程也不豁免，这是本章写作时真实撞出来的第二次编码翻车。）本机实录：

```text
> cmake -S . -B build -G "Visual Studio 17 2022" -A x64
-- Configuring done (5.2s)
-- Build files have been written to: .../fetch_demo/build

> cmake --build build --config Release && build\Release\main.exe
fake_glfw::init() = 1
```

真实网络拉取时，源码会被安置在 `build/_deps/<名字>-src/`（旁边还有 `-build`、`-subbuild` 两个辅助目录；本地 `SOURCE_DIR` 模式没有 `-src`，源码原地不动）。看懂了吗：**依赖变成了依赖图里的普通 target**——`fake_glfw::glfw` 的 INTERFACE 要求自动传给 `main`，和 4.4 的 `resman`→`math` 是同一套机制。4.4 的那颗心脏，第三次跳动。

**glfw 的真实写法**（机制同上，把 `SOURCE_DIR` 换成 git 仓库 + 版本锁定）：

```cmake
include(FetchContent)
FetchContent_Declare(glfw
    GIT_REPOSITORY https://github.com/glfw/glfw.git
    GIT_TAG        3.4              # 版本锁定纪律：钉死 tag，不用 master（误区 8）
    GIT_SHALLOW    TRUE             # 浅克隆：只取这一个提交，省时间省空间
)
# 关掉依赖自带的无用目标（示例/测试/文档），构建时间与告警都省下来。
# CACHE BOOL "" FORCE 三件套的含义：塞进全局缓存、类型为布尔、已有值也强制覆盖——
# 因为 glfw 自己的 CMakeLists 里会用 option() 声明这些开关，你必须抢在它之前定死
set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_TESTS    OFF CACHE BOOL "" FORCE)
set(GLFW_BUILD_DOCS     OFF CACHE BOOL "" FORCE)
set(GLFW_INSTALL        OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(glfw)

# 之后你的目标就能：target_link_libraries(demo PRIVATE glfw::glfw)
```

assimp 同理（`GIT_TAG` 锁定其发布 tag，`ASSIMP_BUILD_TESTS OFF` 等选项见其仓库文档；它体量大，首次拉取和编译要有心理预期）。**glad 特殊一点**：它是 OpenGL 加载器的**代码生成器**——glad2 仓库 FetchContent 之后用 `glad_add_library(glad_gl CORE)` 之类的一行生成加载器库，细节属第 3 节 OpenGL 实操（一句回指，不展开），此处概念级知道「它也是 FetchContent 接入」即可。

> **关于网络，如实声明**：FetchContent 的机制已在上面的本地演示中全流程验证；glfw 的真实拉取在本章写作时**实测过但未完成**——本机网络到 GitHub 的大包传输不稳定（git 报 `fetch-pack: invalid index-pack output`，`_deps/glfw-src` 只落了一半）。写法依据 glfw 官方构建文档（延伸资源），你执行时若下载失败，见下方的处置段。
>
> **网络问题处置**：① 已经下载成功过一次的依赖，离线复用靠 `set(FETCHCONTENT_SOURCE_DIR_GLFW <已下载的 glfw 目录>)` 或全量离线开关 `set(FETCHCONTENT_FULLY_DISCONNECTED ON)`（此时 CMake 只用 `_deps` 里已有的源码，不再联网）；② 公司内网配代理或把仓库镜像到内网 git 后改 `GIT_REPOSITORY` 指向镜像；③ 实在拉不下来就降级——核心学习路径不依赖 glfw/assimp（本章实践任务的分层验收就是为此设计的），等待网络好转或换 find_package/vcpkg 路线补上。

find_package 的完整展开（Config 模式、搜索路径、`REQUIRED` 的语义）值得单独一章，消费姿势先立起来：拿到 imported target 后，把它当普通 target 链接——它的 include 路径、链接库、编译定义会作为 usage requirements 自动传过来。惯用形态一眼看懂：

```cmake
find_package(glfw3 3.4 CONFIG REQUIRED)   # 找不到就 configure 期报错（REQUIRED）
# ...
target_link_libraries(demo PRIVATE glfw::glfw)   # 之后和自己的 target 同一姿势
```

又是 4.4，第四次跳动。两条路线的最终判断标准一致：**谁在什么时刻提供 target 无关紧要，重要的是你的工程只依赖「target + usage requirements」这一层抽象。**

### 4.8 收束成可交接工程：源文件列举 vs glob、目录约定与三步重建

工程最后一个决定：源文件清单怎么维护。新手最爱 `file(GLOB SRC *.cpp)` + `add_executable(demo ${SRC})`——「加文件不用改 CMakeLists，多爽」。两个陷阱，都来自同一个根因：**CMake 不实时感知文件系统，清单只在 configure 期生成一次**。① 你新增 `demo2.cpp` 后直接 build——它不在清单里，**根本不会被编译**（忘重新 configure 的话）；② 反过来删掉 `boss.cpp`，清单里还有它，构建时对着不存在的文件报错，或（配合缓存）留下幽灵符号。官方提供的缓解 `CONFIGURE_DEPENDS`（让每次 build 前重扫目录、发现变化就重新 generate）官方文档自己的措辞是「不推荐」：跨生成器的可靠性无保证，且每次构建都要付一遍扫描成本——它把「清单准确性」这个确定性问题变成「大概率没问题」的祈祷。结论：**默认显式列举源文件**。每加一个 `.cpp` 顺手在 CMakeLists 里添一行——这个「麻烦」正是编译边界的显式化；目录大了就多拆子目录、各管各的 `add_subdirectory`。最终的顶层 CMakeLists 呈现稳定的「五段式」：

```cmake
# CMakeLists.txt（顶层·终稿全貌）
cmake_minimum_required(VERSION 3.20)          # ① 最低版本
project(mini_engine)                          # ② 工程名

# ③ 全局基线：标准 + 选项
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
if(MSVC)
    add_compile_options(/W4)
else()
    add_compile_options(-Wall -Wextra)
endif()

# ④ 子目录各管各的目标
add_subdirectory(libs/math)
add_subdirectory(libs/resman)
add_subdirectory(apps/demo)
add_subdirectory(tools/report)                # ← 4.8 新增（见下）

# ⑤ 测试基础设施
enable_testing()
add_subdirectory(tests)
```

配合两条仓库卫生：`.gitignore` 至少写这几行——`build/`（构建目录全员）、`CMakeCache.txt`（万一有人做过 in-source 构建的遗物也一并防住）、`CMakeUserPresets.json`（个人本地预设，不该强加给全组）；README 里贴三步说明（4.8 末尾给定稿）。新目标 `tools/report` 的全部改动，验证一下「加目标不动既有文件」：

```text
新增 tools/report/CMakeLists.txt：
    add_executable(report main.cpp)
    target_link_libraries(report PRIVATE math)   # 库被第三个消费者复用
新增 tools/report/main.cpp：一个十来行的小工具（算包围盒对角线，打印输出）
顶层 CMakeLists.txt 添 1 行：add_subdirectory(tools/report)
既有文件改动：0
```

```text
> cmake --build build --config Release
  report.vcxproj -> ...\build\tools\report\Release\report.exe

> build\tools\report\Release\report.exe
对角线长度 = 4.583
```

未来的 `vulkan_demo`、第 6 节要复用这个工程的渲染目标——都是同样的三行（新增 CMakeLists + main + 顶层一行）。检验标准 ③ 的扩展性，你已经亲眼验证过了。

README 里给新同事的三步说明也定稿了（贴在仓库根 README.md，比口头交接可靠）：

````markdown
## 构建与测试（Windows + VS2022）

```bat
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
ctest --test-dir build -C Release
```

产物：`build\\apps\\demo\\Release\\demo.exe`；测试全绿标志：`100% tests passed`。
依赖：CMake ≥ 3.20、Visual Studio 2022（含 C++ 桌面开发工作负载）。无其他第三方依赖。
````

**三步重建全流程**（模拟新同事：换个全新的 build 目录，本机实录）：

```text
> cmake -S . -B build-fresh -G "Visual Studio 17 2022" -A x64
-- Configuring done (3.2s) / -- Generating done (0.0s)
-- Build files have been written to: .../mini_engine/build-fresh

> cmake --build build-fresh --config Release
（编译全部目标，0 错误 0 警告）

> ctest --test-dir build-fresh -C Release
100% tests passed, 0 tests failed out of 2
```

三条命令，从零到全绿——本章终点铁律兑付，检验标准 ① 的正片。（CMake ≥ 3.24 还提供 `--fresh` 让已有 build 目录忘掉旧缓存重新 configure；效果等同删目录重建。）

> **可选加分框：CMakePresets.json。** `-G/-A/配置名` 每次手敲既累又易错，可以固化进工程根目录的 `CMakePresets.json`。本章主线工程用到的最小可用版（已实测）：
>
> ```json
> {
>   "version": 3,
>   "configurePresets": [
>     {
>       "name": "windows-default",
>       "generator": "Visual Studio 17 2022",
>       "architecture": "x64",
>       "binaryDir": "${sourceDir}/build"
>     }
>   ],
>   "buildPresets": [
>     { "name": "release", "configurePreset": "windows-default", "configuration": "Release" }
>   ],
>   "testPresets": [
>     { "name": "release", "configurePreset": "windows-default", "configuration": "Release" }
>   ]
> }
> ```
>
> 之后三步变成 `cmake --preset windows-default`、`cmake --build --preset release`、`ctest --preset release`——VS/VSCode/CI 都认这份文件（`cmake --list-presets` 可查）。超出阶段基线的加分项，先用好裸命令，再回来享这个福。

到这里，主线工程收敛为最终形态，这也是你交给后续章节、交给下一个同事的形状：

```text
mini_engine/
├── CMakeLists.txt                 # 顶层：版本→project→全局基线→add_subdirectory→测试
├── CMakePresets.json              # 可选：预设三连
├── .gitignore                     # build/ 等
├── libs/
│   ├── math/                      # 静态库：usage requirements 的主教材
│   │   ├── CMakeLists.txt
│   │   ├── include/math/vec3.hpp
│   │   └── src/vec3.cpp
│   └── resman/                    # INTERFACE 库（纯头文件）：传递性的主教材
│       ├── CMakeLists.txt
│       └── include/resman/
│           ├── resource_manager.hpp
│           └── memory_loader.hpp
├── apps/demo/                     # 可执行：链 resman（传递拿到 math）
│   ├── CMakeLists.txt
│   └── main.cpp
├── tools/report/                  # 第二个可执行：扩展性演示
│   ├── CMakeLists.txt
│   └── main.cpp
└── tests/
    ├── CMakeLists.txt             # enable_testing + add_test
    ├── test_vec3.cpp
    └── test_resman.cpp
```

---

## 深入专题：一条链接错误的解剖——缺符号/重复符号来自哪个 target 传递环节

4.4 给了归因三步法的简版；本专题把它升级为可执行的调试流程。三条故障链全部在主线工程上人为制造（每个都真实跑过并还原），不需要新工程。这也是系列「解剖盒」传统：ch03 解剖过 `std::function` 的黑盒，本章解剖链接器的黑盒。先回收两笔旧账做铺垫：ch01 1.1 给过两个名词——LNK2019（缺符号）与 LNK2005（重复符号）；ch03 3.9 解剖过模板版 LNK2019（「使用点看不到完整定义，实例化不出来」）。今天在 target 图上把它们焊死。

**故障链 1（对照组）：编译期错误——连链接的机会都没有。** math 的 include 改 `PRIVATE`（4.4 实验 a）：`error C1083: 无法打开包括文件`。层次判断的第一课：**C 开头的错误发生在编译期**，头文件搜索路径、宏、语法都在这一层；它们的修复现场是 `target_include_directories` 的传递声明，跟链接器无关。

**故障链 2（缺符号）：LNK2019 + LNK1120。** demo 用了 `math::length` 但链接线断开（4.4 实验 b）。这次的解剖对象是报错里的符号名：

```text
main.obj : error LNK2019: 无法解析的外部符号 "float __cdecl math::length(struct math::Vec3 const &)"
  (?length@math@@YAMAEBUVec3@1@@Z)，函数 main 中引用了该符号
...\demo.exe : fatal error LNK1120: 2 个无法解析的外部命令
```

- 第一行给的是**人类可读形态**：`float math::length(const math::Vec3&)`——谁、什么签名；
- 括号里的 `?length@math@@YAMAEBUVec3@1@@Z` 是**修饰名（mangled name）**：编译器把命名空间、参数表编码进符号名以支持重载。逐字拆开看（MSVC 方言）：`?` 函数符号开头；`length` 名字；`@math@` 命名空间；`@@` 限定区结束；`YA` 外部全局函数、cdecl 调用约定；`M` 返回类型 float；`AEBUVec3@1` 参数——`A` 引用、`EB` const 修饰、`UVec3@` 结构体 Vec3、结尾的 `1` 「回指第一个出现过的作用域」即 `math::`。它就是 ch01 1.1 的 `??$lerp@H@@YAHHHH@Z` 同族。链接器的世界里只有修饰名，读懂它就能直接读出「谁、什么签名」；
- `main.obj : ...` 告诉你**谁要它**（main.obj），LNK2019 的意思是「我要的这个符号，所有喂给你的 `.lib`/`.obj` 里都没找到」。

归因三步法完整版：① 读符号名 → 锁定提供者（`math::length` 显然归 math 库）；② **查证库里到底有没有这个符号**——别猜，用 VS 自带的 `dumpbin`（在 x64 Native Tools 命令行里，或 cl.exe 同目录）看静态库的符号表：

```text
> dumpbin /LINKERMEMBER build\libs\math\Release\math.lib
（输出节选）
  1D2 ??Hmath@@YA?AUVec3@0@AEBU10@0@Z
  1D2 ?dot@math@@YAMAEBUVec3@1@0@Z
  1D2 ?length@math@@YAMAEBUVec3@1@@Z
```

`?length@math@@YAMAEBUVec3@1@@Z` 就在列表里——**库里有货，链接器没收到**。结论从「可能 math 写错了」收窄为「math 的传递声明断了」：回去检查依赖链上谁该 `PUBLIC`/`INTERFACE` 却没写。③ 修复后重链。这套「报错 → 工具查证 → 传递声明」的流程把链接错误从玄学变成检索。对照 ch03 3.9 的模板版：普通函数缺定义是「库里没这道菜」，模板缺实例化是「菜谱在使用点看不见、根本没人下厨」——报错形似（都是 LNK2019），病灶在源头上完全不同，而 dumpbin 的符号表就是分辨它们的显影液。

**故障链 3（重复符号）：LNK2005 + LNK1169。** 往 `resource_manager.hpp` 里塞一个非 `inline` 的函数定义（实验后已还原）：

```cpp
// ⚠ 破坏实验：非 inline 的定义住在头文件里
int helper() { return 42; }
```

再给 demo 加第二个源文件 `extra.cpp`（`add_executable(demo main.cpp extra.cpp)`），两个 `.cpp` 都包含这个头：

```text
extra.obj : error LNK2005: "int __cdecl helper(void)" (?helper@@YAHXZ) 已经在 main.obj 中定义
...\demo.exe : fatal error LNK1169: 找到一个或多个多重定义的符号
```

这正是 ch01 1.1 埋过的 **ODR（单一定义规则）**：头文件被多个翻译单元包含，每个 `.obj` 里都躺了一份 `helper` 的实体，链接器合并时撞车。修法两选：函数标 `inline`（允许跨翻译单元重复定义、链接时合并）；或定义搬去 `.cpp`、头文件只留声明。**推论**：头文件里的函数定义必须 `inline`（或模板，或类内定义——后两者天然豁免），这也是为什么 ch03 的模板库头文件从没炸过 ODR。还有一个更阴险的变体只提一句：两个静态库各编了一份同符号（比如都内置了同一份工具函数），链接器静默选一份，**不报错**——两个「看似相同」的实现从此可能分道扬镛，这是工程里最难查的一类 ODR 违规，本章不展开。

收束成速查卡（检验标准 ②的口袋版）：

| 症状 | 层次 | 归因动作 |
| --- | --- | --- |
| `C1083` 找不到头文件 / `C2xxx` 语法语义错 | 编译期 | 查头文件搜索路径的传递声明（`target_include_directories` 三键）；语法错先看乱码/编码（4.5） |
| `LNK2019` 缺符号 + `LNK1120` | 链接期·少 | 读符号名 → `dumpbin /LINKERMEMBER` 查证提供者 → 修 `target_link_libraries` 传递链 |
| `LNK2005` 重复符号 + `LNK1169` | 链接期·多 | 查头文件里的非 inline 定义（ODR）；库与库的静默重复单独警惕 |

## 实践任务

> **关于「渲染器」**：任务中提到的「渲染器 + 数学库 + ResourceManager 三素材工程化」指 P1 自写渲染器项目的 CMake 化（阶段 4 实践任务原文）。P1 未启动时按下面的**独立版本**完成，验收标准完全一致；P1 启动后迁移即「可选 P1 衔接点」。
>
> **分层验收声明**：任务 1 是核心层，**全程本地可验证、不依赖网络**；任务 2、3 是扩展层，依赖网络拉取，失败**不影响任务 1 的验收**（4.7 的处置段给出了降级路径）。先交付一个核心层全绿的工程，再按余力上扩展。

### 任务 1（核心层）：三素材组成规范 CMake 工程

**交付物**：把主线工程这套结构独立复刻一遍（自己写全部 CMake，不许回头看一眼抄一行——写完再对照）：`math` 静态库 + `resman` INTERFACE 库 + `demo` 应用 + `test_vec3`/`test_resman` 两个测试目标 + CTest 全绿 + Debug/Release 双配置构建。

**素材包**（E3 独立版，全部是本章正文已给出的文件，按小节取用即可）：

| 文件 | 出处 | 说明 |
| --- | --- | --- |
| `libs/math/include/math/vec3.hpp`、`src/vec3.cpp` | 4.3 全文 | 点积/模长在 `.cpp`，库有链接实体 |
| `libs/resman/include/resman/resource_manager.hpp` | 4.4 + 本节下方补全 | ch03 任务 1 定稿；`memory_loader.hpp` 同目录 |
| `apps/demo/main.cpp` | 本节下方补全 | 同时用 math 与 resman |
| `tests/test_vec3.cpp`、`tests/test_resman.cpp` | 4.6 全文 | 返回码约定，不引框架 |
| 各 `CMakeLists.txt` | 4.3–4.8 | 顶层五段式收束形态为准 |

`resource_manager.hpp` 与 `memory_loader.hpp` 的完整文本（正文 4.4 只给了 CMake 侧，源码在此补全，可直接滕抄）：

```cpp
// libs/resman/include/resman/resource_manager.hpp —— ch03 任务 1 定稿
#pragma once
#include <cstdint>
#include <concepts>
#include <memory>
#include <string>
#include <unordered_map>
#include <vector>
#include <utility>

namespace resman {

struct Handle { std::uint32_t slot; };   // 轻量句柄：拷贝零成本

template <class L, class V>
concept ResourceLoaderFor =
    requires(const L& l, const std::string& key) {
        { l.load(key) } -> std::same_as<std::unique_ptr<V>>;
    };

template <class V>
class ResourceManager {
public:
    template <ResourceLoaderFor<V> L>
    Handle load(const L& loader, const std::string& key) {
        if (auto it = slots_.find(key); it != slots_.end()) {
            return Handle{it->second};              // 命中缓存：去重
        }
        std::unique_ptr<V> res = loader.load(key);
        const auto slot = static_cast<std::uint32_t>(storage_.size());
        storage_.push_back(std::move(res));         // unique_ptr 搬入：Value 零拷贝
        slots_.emplace(key, slot);
        ++load_count_;
        return Handle{slot};
    }
    const V& get(Handle h) const { return *storage_[h.slot]; }
    std::uint32_t load_count() const { return load_count_; }

private:
    std::unordered_map<std::string, std::uint32_t> slots_;
    std::vector<std::unique_ptr<V>> storage_;
    std::uint32_t load_count_ = 0;
};

} // namespace resman
```

```cpp
// libs/resman/include/resman/memory_loader.hpp —— 内存注册表 Loader，不碰图形 API
#pragma once
#include <memory>
#include <stdexcept>
#include <string>
#include <unordered_map>
#include <utility>

namespace resman {

struct TextureData {                     // 假装是纹理：一行像素数据够教学用了
    std::string pixels;
};

class MemoryLoader {
public:
    explicit MemoryLoader(std::unordered_map<std::string, std::string> files)
        : files_(std::move(files)) {}
    std::unique_ptr<TextureData> load(const std::string& key) const {
        auto it = files_.find(key);
        if (it == files_.end()) throw std::out_of_range("no such asset: " + key);
        return std::make_unique<TextureData>(TextureData{it->second});
    }
private:
    std::unordered_map<std::string, std::string> files_;
};

} // namespace resman
```

```cpp
// apps/demo/main.cpp（v2 版）
#include <cstdio>
#include <string>
#include <unordered_map>
#include "math/vec3.hpp"
#include "resman/resource_manager.hpp"
#include "resman/memory_loader.hpp"

int main() {
    // ① math：摆一个直角三角形（v1 的老朋友）
    math::Vec3 a{3.0f, 0.0f, 0.0f};
    math::Vec3 b{0.0f, 4.0f, 0.0f};
    std::printf("length(a+b) = %.1f\n", math::length(a + b));

    // ② resman：同一个 Key 加载两次，第二次应命中缓存
    resman::ResourceManager<resman::TextureData> textures;
    resman::MemoryLoader loader({{"brick.png", "12 34 56"},
                                 {"steel.png", "AA BB CC"}});
    auto h1 = textures.load(loader, "brick.png");
    auto h1b = textures.load(loader, "brick.png");   // 去重：不应触发第二次 load
    auto h2 = textures.load(loader, "steel.png");
    std::printf("brick pixels: %s | load_count = %u（两次 brick 只算了 1 次）\n",
                textures.get(h1).pixels.c_str(), textures.load_count());
    return 0;
}
```

**验收要点**（对应检验标准 ①③）：

1. 换全新目录（或 `--fresh`）三步重建，`ctest -C Release` 全绿，终端输出贴进笔记；
2. `--config Debug` 与 `--config Release` 各构建一次，指出两组产物目录，并说明为什么它们互不干扰；
3. demo 的 `CMakeLists.txt` 里只出现 `target_link_libraries(demo PRIVATE resman)` 一条链接声明，demo 依然能用 math——口述这条链怎么传过来的；
4. ctest 至少 2 条测试全绿；顺手做一次「红→绿」：故意改坏一个断言，贴 `ctest --output-on-failure` 的失败输出，修回全绿；
5. 口述「再新增一个 `tools/stats` 目标要写几行、动几个既有文件」（答案：新增 2 文件 + 顶层 1 行，既有文件零改动）。

**可选 P1 衔接**：P1 启动后，把真渲染器、真数学库按同构迁移（库 + 应用 + 测试 + FetchContent 三方库），即 P1「CMake 工程化」里程碑的落点。

### 任务 2（扩展层 A，依赖网络）：FetchContent 接 glfw + assimp

**交付物**：在任务 1 工程上新增 `cmake/third_party.cmake`（或顶层声明段）：FetchContent 拉 glfw 与 assimp（各自 `GIT_TAG` 锁定发布 tag、`GIT_SHALLOW TRUE`、关闭 examples/tests/docs 选项），demo 链接 `glfw::glfw`（编译链接通过即验收，**不要求开窗口**——图形归第 3 节，本章不越界）。

**验收要点**：① `GIT_TAG` 钉死具体 tag（用 master/主干 = 不合格，理由见误区 8）；② 能用 `cmake --build build --target help`（Ninja/单配置）或 VS 解决方案资源管理器里看到 glfw/assimp 目标已进依赖图（任选一种自查方式）；③ 贴一次 configure 输出（含 FetchContent 下载/安装日志），如实标注本步骤依赖网络；④ 网络失败时按 4.7 处置段走降级路径，并记录卡在哪一步。

### 任务 3（扩展层 B，依赖网络，与任务 2 二选一即可）：googletest 替换裸测试

FetchContent 拉 googletest（锁 tag），`test_vec3`/`test_resman` 迁移为 gtest 断言版（`EXPECT_FLOAT_EQ`、`EXPECT_EQ` 替换手写 `expect_near`），链接 `gtest_main` 省掉手写 main；`add_test` 登记不变，`ctest -C Release` 应与迁移前同为全绿。验收口径同任务 2：锁 tag、贴实录、失败不影响核心层。

## 自测题

答题时先不回看正文；第 1–3 题直接对位本阶段三条检验标准。

**第 1 题（口述，检验标准 ①）**：新机器克隆仓库后，从零到全绿是哪三条命令？每条各自发生什么（configure 生成什么 / build 驱动什么 / ctest 找什么）？为什么 `build/` 目录可以随时整个删掉而不心疼？

**第 2 题（改错+归因，检验标准 ②）**：某同学的工程装成了这样（mini_engine 变体）：

```cmake
# libs/math/CMakeLists.txt（装错版）
add_library(math STATIC src/vec3.cpp)
target_include_directories(math PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)   # ⚠

# libs/resman/CMakeLists.txt（装错版）
add_library(resman INTERFACE)
target_include_directories(resman INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
# ⚠ 忘了 target_link_libraries(resman INTERFACE math)

# apps/demo/CMakeLists.txt
add_executable(demo main.cpp)          # main.cpp 同时用了 math::length 和 resman
# ⚠ 忘了 target_link_libraries(demo PRIVATE resman)
```

他先后收到两条报错：`error C1083: 无法打开包括文件: "math/vec3.hpp"` 与 `error LNK2019 ... fatal error LNK1120`。请：① 分层——哪条是编译期、哪条是链接期？② 各自归因到具体传递环节（注意两条报错各对应哪个装错点）；③ 给出最小修复。

**第 3 题（写码，检验标准 ③）**：在最终形态的 mini_engine 上新增一个 `vulkan_demo` 可执行目标（链 math/resman、吃全局 /utf-8 与 /W4 基线）。写出全部改动（几行、分布在哪些文件）；为什么既有文件零改动？（末句衔接：第 6 节将复用此工程。）

**第 4 题（输出预测）**：`cmake_minimum_required(VERSION 2.8.12)` 在 CMake 4.2.1 下 configure 会发生什么？报错大意是什么？为什么 CMake 4.x 要这么做？正确的写法是什么？

**第 5 题（改错）**：某工程这样收集源文件：

```cmake
file(GLOB SRC *.cpp)
add_executable(demo ${SRC})
```

症状 A：新增 `demo2.cpp` 后直接 build，它没被编进去；症状 B：删除 `boss.cpp` 后 build，链接器仍报它定义的符号有问题。逐个解释成因并给出修法；追问：若给 GLOB 加上 `CONFIGURE_DEPENDS`，哪个症状消失、哪个仍存在、代价是什么？

**第 6 题（场景判断）**：VS 生成器下：① `cmake --build build --config Debug` 之后直接跑 `ctest`（不带 `-C`），会发生什么？② 只构建过 Debug，跑 `ctest -C Release` 会怎样？③ 在 Debug 配置下对比两版实现的耗时并下结论，可靠吗（机制层面的原因）？

**第 7 题（选键）**：三个场景各选 `PUBLIC`/`PRIVATE`/`INTERFACE` 并说明理由：① `math` 静态库对外的 `include/`；② `resman` 纯头文件库的 include 路径声明与它对 math 的链接；③ demo 内部用的某第三方库头文件（不外泄给任何使用者）。追问：库实现 `.cpp` 里 include 的私有头该用哪个键？

**第 8 题（概念对比）**：FetchContent 与 find_package 各自的适用场景与优劣？为什么 FetchContent 拉下来的 glfw 也以 `glfw::glfw` 这种 imported target 的面目出现——它和 4.4 的 usage requirements 是什么关系？

## 常见误区

**误区一：`file(GLOB)` 当默认。** 症状：新增文件不进构建、删除文件残留幽灵报错，「CMake 是不是坏了」。病根：源清单只在 configure 期生成一次，CMake 不实时感知文件系统。纠偏：显式列举为默认；`CONFIGURE_DEPENDS` 官方明言不推荐（跨生成器可靠性无保证 + 每次 build 都扫描的开销）。

**误区二：IDE 黑盒回归。** 症状：只会点 VS 绿色按钮；把 `.sln` 当源头手改；或反过来以为「CMake 在编译我的代码」。病根：没分清源头（CMakeLists.txt）与生成物（`.sln`/`.vcxproj`），没分清生成器（CMake）与构建系统（MSBuild/Ninja）。纠偏：手改 `.sln` 必被覆盖；三步命令必须脱离 IDE 能跑通，IDE 只是生成物的查看器。

**误区三：`cmake_minimum_required` 随意写。** 症状：抄老教程写 `2.8` → CMake 4.x 直接拒绝 configure（4.1 实测报错原文）；或随手写最新版本号 → 老机器/CI 上配不了。病根：把「最低版本」当「我装的这个版本」填。纠偏：写团队实际最低可用版本（本章口径 3.20），需要时用 `3.20...3.28` 区间语法。

**误区四：目录级全局命令滥用。** 症状：`include_directories`/`link_directories`/`add_definitions` 满天飞，出现「这个文件能编、那个不能编」的玄学。病根：全局命令把要求撒进整个目录，绕开了 target 模型，依赖关系不可见。纠偏：一切挂 target（`target_include_directories` 等三键家族）；目录级命令只在维护祖传代码时被迫认识。

**误区五：Debug/Release 混淆。** 症状：build 忘 `--config`、ctest 忘 `-C`（Not Run 还以为测试坏了）；在 Debug 下测性能得出「方案 A 慢一倍」的结论。病根：多配置语义没进肌肉；忘了 MSVC Debug 是 `/Od` + 迭代器调试层。纠偏：「配置」贯穿 build 与 ctest；基准一律 Release——测量方法论归第 6 章，此处只记纪律。

**误区六：把生成物提交进仓库 / in-source 构建。** 症状：仓库里躺着 `build/`、`CMakeCache.txt`，换机必炸；源码树里散落 `.obj`。病根：没建立「产物可再生、源码即真相」的 out-of-source 世界观。纠偏：`.gitignore` 写 `build/`；构建目录随时可删可重建（检验标准 ① 的机制根基）。

**误区七：C++ 标准与编码选项漏设。** 症状：老编译器上「明明写了 C++20 语法却报一堆莫名语法错」（忘 `CMAKE_CXX_STANDARD_REQUIRED`，静默回退）；中文注释文件在同事机器上炸出 C4819/乱码（没挂 `/utf-8`，4.5 三组现场）。病根：把「我这台机器能过」当跨环境承诺。纠偏：顶层一次设全三件套与 `/utf-8`；源文件统一存 UTF-8；常查 4.1 翻译表。

**误区八：依赖不锁版本。** 症状：`GIT_TAG master` 今天能拉、明天拉下来的代码行为变了；「上周还好好的」式依赖故障。病根：浮动引用让构建不可复现——你依赖的不是「glfw 3.4」而是「glfw 主干此刻的样子」。纠偏：`GIT_TAG` 钉死发布 tag（要升级时手动换 tag、跑全量测试）；预装路线则锁定包版本。

## 延伸资源

- **Modern CMake**（Henry Schreiner，持续更新）：https://cliutils.gitlab.io/modern-cmake/ —— 本章「target 一切」世界观的正统续读，Stages 推荐资源已收录；
- **CMake 官方文档（latest）**：https://cmake.org/cmake/help/latest/ —— 工具书地位对位 cppreference，任何语义疑问先查它；重点页：`cmake-build(7)` 手册与 FetchContent 模块页（https://cmake.org/cmake/help/latest/module/FetchContent.html）；
- **CMake 官方 Tutorial**：https://cmake.org/cmake/help/latest/guide/tutorial/ —— Step by Step 动手型补练，与本章互补；
- **CMakePresets 规范**：https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html —— 4.8 可选框的完整字段；
- **Professional CMake: A Practical Guide / Practices and Patterns**（Craig Scott，付费书）：https://crascit.com/professional-cmake/ —— 工程化进阶主读，作者即 FetchContent 的维护者；
- **glfw 官方构建文档**：https://www.glfw.org/docs/latest/build.html —— 4.7 FetchContent 写法的权威出处；
- **vcpkg**：https://vcpkg.io —— Windows 预装路线的包管理器（find_package + toolchain 文件），与 FetchContent 互补；
- 检索关键词（不硬给链接）：`CMAKE_EXPORT_COMPILE_COMMANDS clangd`（把编译命令导出给编辑器语义补全——VS 生成器不产出该文件，Ninja 路线的加分区）；`CGold CMake`（另一本免费在线书，内容稍老）。

## 参考答案

### 自测题

**第 1 题**。三条命令：① `cmake -S . -B build -G "Visual Studio 17 2022" -A x64`——configure：跑 CMakeLists.txt，检测编译器，**生成**底层构建系统的工程文件（VS 生成器下是 `.sln`/`.vcxproj`），不碰任何 `.cpp`；② `cmake --build build --config Release`——build：驱动 MSBuild 真正预处理、编译、链接，产物落在 `build/<目标镜像路径>/Release/`；③ `ctest --test-dir build -C Release`——ctest：到 build 目录的测试清单里找 `add_test` 登记过的测试，逐个运行并按**退出码**判定通过与否（0 = 通过）。`build/` 可以随时删：它的每一个字节都是 configure/build 从源码树再生出来的，不含任何「只有它有」的信息——这是 out-of-source 构建的机制保障；也是它能进 `.gitignore` 的理由：版本库只存「真相」（源码 + CMakeLists），不存「再生物」。

**第 2 题**。① 分层：`C1083`（找不到头文件）是**编译期**错误——头文件搜索路径在编译阶段起作用；`LNK2019/LNK1120` 是**链接期**错误——符号解析在链接阶段。② 归因：`C1083 "math/vec3.hpp"` → math 的 include 路径用了 `PRIVATE`，不再作为 usage requirement 传给使用者 → demo 编译时找不到头；修复：改回 `target_include_directories(math PUBLIC ...)`。`LNK2019`（缺 `math::` 的符号）→ demo 与 math 之间的链接线没接（demo 忘链 resman，或 resman 忘了 `INTERFACE math`）→ math.lib 没进 demo 的链接输入；修复：补上 `target_link_libraries(resman INTERFACE math)`，demo 只链 resman 即可。③ 最小修复共两行，均在提供者一侧——**在使用者侧手拼路径/手动指定 `.lib` 全是反模式**。

**第 3 题**。全部改动三处：新增 `apps/vulkan_demo/CMakeLists.txt`（`add_executable(vulkan_demo main.cpp)` + `target_link_libraries(vulkan_demo PRIVATE resman)`）；新增 `apps/vulkan_demo/main.cpp`；顶层 `CMakeLists.txt` 添一行 `add_subdirectory(apps/vulkan_demo)`。既有文件零改动的原因：`/utf-8` 与 `/W4` 是挂在全局的基线（`add_compile_options`），C++20 三件套在顶层——新目标生下来就继承；而它需要的库经 resman 的 INTERFACE 一并传递。这正是第 6 节将复用此工程时「新增 Vulkan 目标只需添加一个 CMake 目标」的现场演示。

**第 4 题**。configure 直接报错（4.1 实测原文）：`Compatibility with CMake < 3.5 has been removed from CMake.`——CMake 4.x 移除了对 3.5 以下版本兼容（含 `2.8.12` 这种上古写法），因为维护这些行为开关的成本远超价值。正确写法：填团队实际最低可用版本（本章口径 `VERSION 3.20`），需要兼容区间用 `3.20...3.28` 语法。

**第 5 题**。根因同一个：`GLOB` 的清单只在 configure 期生成，build 不感知文件系统变化。症状 A：`demo2.cpp` 不在旧清单里，直接 build 不会被编译；修法：重新 configure（或干脆显式列举——根治）。症状 B：清单里还有 `boss.cpp`，configure 期它还在、build 期文件没了，产生幽灵目标/残留符号；修法：重新 configure 让清单跟上现实，或显式列举后从清单删掉它。`CONFIGURE_DEPENDS` 追问：它让 build 前重扫目录并触发重新 generate，两个症状通常都能自愈；但官方文档明言**不推荐**——跨生成器可靠性无保证，且每次 build 都扫描有开销。它把确定性换成了便利，显式列举仍是默认答案。

**第 6 题**。① 每条测试显示 **Not Run**（`Test not available without configuration. (Missing "-C <config>"?)`），汇总 0% passed，退出码非 0（本机实测 8）——多配置生成器不知道你要跑哪个配置目录下的可执行文件，索性一个都不跑。② 只构建过 Debug 却跑 `-C Release`：ctest 去找 `Release` 目录下的测试可执行文件——没构建过，测试同样以 Not Run/找不到告终。注意 ctest **不会替你 build**；要么先 `cmake --build build --config Release`，要么用 `ctest --build-and-test` 一体化模式。③ 不可靠：Debug 配置是 `/Od` 禁优化 + `/RTC1` 运行时检查 + `/MDd` 调试运行库（4.5 的命令行实录），测的是「未优化代码的相对行为」，与 Release 下的真实热点分布可以完全不同——性能对比一律 Release；机制出处见 4.5，测量方法论归第 6 章。

**第 7 题**。① `PUBLIC`：对外头文件是接口的一部分——math 自己编译要用，使用者也必须要，两头都装。② 都是 `INTERFACE`：resman 是纯头文件库，「自己不编译任何东西」，include 路径与「需要链 math」这两条要求都只对使用者有意义——自己那一份清单是空的。③ `PRIVATE`：自己编译用、不外泄——写进 usage requirements 反而会污染使用者的编译命令（谁都不该看到的头文件路径传了出去，还会掩盖真实的依赖关系）。追问：私有头只在库自己的 `.cpp` 里被 include，用 `PRIVATE`——如果它和对外头文件同住 `include/` 不合适，惯用做法是另建 `src/` 下的私有头目录并加一条 `target_include_directories(math PRIVATE .../src)`，把「接口」与「实现细节」在目录层就分开。

**第 8 题**。FetchContent：configure 期拉**源码**并入自己的构建——版本锁死在仓库里、新同事零环境配置，代价是每次全新 configure 要下载、依赖自身也要占编译时间。find_package：找**预装/预编译**好的依赖，给你 imported target——编译快、版本由包管理器（vcpkg 等）统一维护，代价是「环境要先装好」，换机器要重装。关系：FetchContent 拉下来的 glfw 源码会作为子工程被 configure，它自己的 CMakeLists 里定义了带 usage requirements 的目标并起了 `glfw::glfw` 别名——你在 4.7 的 fake_glfw 里亲手写过同一套三行（`add_library` + `ALIAS` + `INTERFACE` 要求）。所以 imported target 和你自己写的 target 消费方式完全一致：链它、收它的传递要求——4.4 的模型是全 CMake 世界的通用货币，不区分「自己的」和「别人的」target。

### 实践任务

**任务 1（核心层）参考思路**：工程骨架照 4.8 终稿目录树；验收要点 1–5 的达成路径——①全新目录三步重建即 4.8 实录的重演，贴出 `100% tests passed` 行即达标；②两组产物分别在 `build/apps/demo/Debug/` 与 `build/Release/`（每个目标的镜像路径下），互不干扰是因为多配置生成器把各配置的编译参数与产物完全分开；③传递链口述模板：「resman 是 INTERFACE 库，`target_link_libraries(resman INTERFACE math)` 把 math 挂进它的接口要求；demo 链 resman 时这分要求原样传入，math.lib 进了 demo 的链接输入、math 的 include 路径进了 demo 的编译命令」；④红→绿记录模板：改坏断言（如把 `!= 1` 改成 `!= 2`）→ `ctest -C Release --output-on-failure` 贴 `resman_dedup` 的 FAIL 输出与 `Failed` 汇总 → 修回 → `100% tests passed`；⑤第 5 题验收答案已在题目括号内。常见坑预警：demo 忘删对 math 的直接链接（就验证不了传递性）；`enable_testing()` 写进了 tests 子目录（ctest 找不到清单）。

**任务 2/3（扩展层）参考思路**：glfw 的声明块照 4.7 全文；assimp 同构（`GIT_TAG` 锁其 GitHub 发布 tag，`ASSIMP_BUILD_TESTS OFF`、`ASSIMP_INSTALL OFF` 常关）。自查目标是否就位：VS 打开 `.sln` 看解决方案里多出的 `glfw`/`assimp` 工程，或 build 目录里 `_deps/glfw-src` 存在且 configure 日志有 `FetchContent: ... glfw` 字样。gtest 迁移要点：`FetchContent_Declare(googletest GIT_REPOSITORY https://github.com/google/googletest.git GIT_TAG <锁定的 tag>)`；`FetchContent_MakeAvailable(googletest)`；测试目标改链 `GTest::gtest_main`，删掉手写 main，断言换 `EXPECT_*`；`add_test` 行不变。网络失败先重读 4.7 处置段再动手——这正是它存在的意义。

---

*本章完。从这一章起，你交付的不再是单文件示例，而是一个「新机器三步重建、全绿可验证、加目标不动老文件」的工程。下一章我们把并发请进来：数据竞争为什么是未定义行为、mutex 与条件变量怎么守住共享的任务队列——而你的第一个实战场景，就是给本章这个工程接上「主线程渲染 + 工作线程异步加载」的异步队列，让 ResourceManager 的加载真正离开主线程。*
