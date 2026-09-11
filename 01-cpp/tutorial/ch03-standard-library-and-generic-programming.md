# 第 3 章｜标准库与泛型编程：从用模板到写模板

欢迎来到第 3 章。第 2 章结尾我们说过：那里借出去的 `std::span` 与 `std::string_view`——两个「借而不拥有」的视图类型——会在本章正式转正（3.3 与 3.8 节）。前两章我们回答了「对象怎么活、资源谁管」；这一章回答另一对问题：**轮子怎么选、怎么造**。前半章（3.1–3.8）把标准库这只现成的轮子用对：容器选型有依据、迭代器失效有戒心、算法库优先于手写循环、可调用对象与词汇类型各归其位；后半章（3.9–3.14）跨过那道门槛——从「只会用模板」到「能写带约束的模板」。跨门槛之前先把一句话钉在墙上，它是本章的世界观，也是你检验标准里的第一道口述题：

> **模板是编译期的代码生成机制，不是运行期魔法。** 你写下的模板只是「生成代码的配方」；编译器用你给的具体类型代入配方，生成出一份份真实代码。运行期没有任何玄学，只有编译器替你写好的、本来就要写的那几份函数。

前两章欠的账，本章全部结清。下面这张表在对应小节兑现，读到这里先混个脸熟，往后每还一笔我们会点名：

| # | 前章承诺 | 兑付位置 |
| --- | --- | --- |
| ① | 第 1 章 1.5：「`std::move` 的真实签名是模板，机制将在第 3 章展开」 | 3.9 |
| ② | 第 1 章 1.8 末：「constexpr 编译期计算属于第 3 章的主题」 | 3.12 |
| ③ | 第 1 章 1.9 戒律 + 1.11 HUD 崩溃：「指针为什么会悬、完整失效规则将在第 3 章展开」 | 3.2 |
| ④ | 第 1 章 1.9：「`if constexpr` 配合模板的全部威力将在第 3 章展开」 | 3.13 |
| ⑤ | 第 2 章 2.8：「lambda 捕获的完整机制（按值/按引用/init capture）将在第 3 章展开」 | 3.5 |
| ⑥ | 第 2 章 2.9：「`span`/`string_view` 的选型细节将在第 3 章展开」 | 3.3 + 3.8 |
| ⑦ | 第 2 章延伸资源：「EMC++ 条 31/32 init capture 是第 3 章剧透，可先读一半」 | 3.5 |

本章的终点是一条工程铁律，和前两章的「零裸 `glDelete`」同级：**面对任何一个「加载-缓存-去重」的需求，你能现场写出带 concepts 约束的 `ResourceManager<Key, Value, Loader>`——同一套核心逻辑管理纹理和着色器两类资源而核心逻辑零复制，并且说得出它每一次拷贝发生在哪、为什么可以不发生**。这是本章实践任务 1 的交付物，也是第 3 节渲染器项目 P1 的直接存货。

---

## 学习目标

对齐本领域阶段 3（标准库与泛型编程）的学习目标。读完本章，你应该能够：

1. **容器选型有依据、失效有戒心**：给出 vector/deque/unordered_map/map 各自的复杂度画像与最适岗位；迭代器/引用/指针「什么操作让什么东西失效」能对答如流（第 1 章 HUD 崩溃的完整司法解释）。
2. **算法库优先于手写循环**：sort/find_if/transform/remove-erase 成为本能；能解释 `std::remove` 为什么不删东西、C++20 的 `std::erase_if` 为什么一步到位。
3. **吃透可调用对象全景**：函数指针/仿函数/lambda/`std::function`/`std::bind` 五者关系一张图；会写含占位符与 `std::ref` 的 bind 并给等价 lambda；说清 `std::function` 类型擦除的代价边界 vs 模板参数。
4. **词汇类型按场景点名**：「可能没有值」→ `optional`、「有限集合选一」→ `variant`（+`visit` 穷举状态机）、「任意类型」→ `any`（窄用途）、连续数据传参 → `span`。
5. **能写带约束的模板**：理解实例化模型与「定义为何放头文件」；会写函数/类模板、全特化与偏特化；用 concepts 给模板参数立契约，并能对照旧 SFINAE 说清可读性收益；会用 `constexpr`/`consteval`/type traits/变参折叠/`if constexpr` 组装编译期逻辑；知道 CRTP 长什么样。
6. **建立纪律**：先具体后泛化、concepts 约束先行；不为想象中的泛化写模板、不硬塞 C++20 全家桶（3.8 的注记框会划清 ranges/coroutines 的归属）。

> **环境约定（全章适用）**：Windows + Visual Studio 2022（「使用 C++ 的桌面开发」工作负载），语言标准 `/std:c++20`，警告级别 `/W4`。本章源码含中文注释，命令行编译请加 `/utf-8`（否则 MSVC 默认按本地代码页读源文件，中文注释会被误读成一串莫名其妙的语法错误）：
>
> ```bat
> cl /utf-8 /std:c++20 /EHsc /W4 源文件.cpp
> ```
>
> GCC / Clang 等价命令：`g++ -std=c++20 -Wall -Wextra 源文件.cpp` 或 `clang++ -std=c++20 -Wall -Wextra 源文件.cpp`。本章全部「完整示例」均在 MSVC 19.44 工具集下编译零警告、运行并逐行核对过输出；**报错节选均为真实编译输出**（MSVC 19.44、中文语言包环境——你若用英文版工具链，报错关键词相同，看 `error Cxxxx` 编号即可）。涉及 C++17/20 差异或引擎停留在 C++17 会受影响的场景，正文显式标注（本章 3.11 concepts、3.13 部分用法**依赖 C++20**，C++17 平替在各节给出）。`sizeof`、SSO 容量、扩容倍率等数字均为 MSVC x64 实测口径，属实现细节，随实现而异——记规律，别背数字。

---

## 分节正文

### 3.1 容器选型与复杂度：vector、deque、unordered_map、map

标准库容器是一批「已经造好的轮子」，但选错轮子的代价是真实的：高频中间插入选 vector，每帧都在付搬家的钱；要有序遍历选了 unordered_map，只好每次遍历前再 sort 一遍。本章的选型纪律一句话：**选型 = 访问模式 × 复杂度 × 失效画像**。前两个因素本节讲，失效画像是选型的第三维度，下一节单独展开。

先把四个主力容器的画像摆到一张表里（复杂度均摊口径；n 为元素数）：

| 容器 | 内存布局 | 尾插/尾删 | 首插/首删 | 中间插/删 | 随机访问 | 查找 | 遍历顺序 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `vector` | 连续内存 | 均摊 O(1) | O(n) | O(n) | O(1) | O(n) | 插入序 |
| `deque` | 分段连续 | O(1) | O(1) | O(n) | O(1)* | O(n) | 插入序 |
| `map` | 红黑树 | O(log n) | O(log n) | O(log n) | 无 | O(log n) | **按键有序** |
| `unordered_map` | 哈希桶 | 均摊 O(1) | — | 均摊 O(1) | 无 | **均摊 O(1)**，最差 O(n) | **不稳定（哈希序）** |

\* deque 的随机访问复杂度也是 O(1)，但比 vector 多一跳：分段存储意味着「先查段表、再进段内」（两跳 vs 一跳），实测口径下常数更大——这也是「别把 deque 当更高级的 vector」的原因之一。它的本职是**两头都要 O(1) 进出的队列**。

看四个容器各自上岗的完整示例（背包、任务队列、资源缓存、排行榜——每个数据结构都落在它最自然的岗位上）：

```cpp
// ch03_containers.cpp —— 四个容器各自的「本职岗位」（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_containers.cpp
#include <cstddef>
#include <cstdio>
#include <deque>
#include <functional>
#include <map>
#include <string>
#include <unordered_map>
#include <vector>

struct Item {                 // 背包物品：数量会增减、按位访问、顺序无所谓
    std::string name;
    int         count = 0;
};

struct Task {                 // 每帧任务队列：队尾进（新任务）、队首出（执行）
    std::string name;
};

struct Texture {              // 资源缓存：按「资源名」随机查找
    std::size_t bytes = 0;
};

struct ScoreEntry {           // 排行榜：需要随时按分数有序遍历
    std::string player;
    int         score = 0;
};

int main() {
    std::printf("== 1. vector：背包物品（尾部增删 + 按位访问）==\n");
    std::vector<Item> bag;
    bag.push_back({"药水", 5});
    bag.push_back({"卷轴", 2});
    bag[0].count += 3;        // 随机访问 O(1)
    for (const Item& it : bag) std::printf("  %s x%d\n", it.name.c_str(), it.count);

    std::printf("== 2. deque：每帧任务队列（两头都进）==\n");
    std::deque<Task> tasks;
    tasks.push_back({"渲染天空盒"});     // 本帧末尾排入
    tasks.push_front({"读输入"});        // 紧急任务插到最前，下一轮先跑
    tasks.push_back({"更新粒子"});
    for (const Task& t : tasks) std::printf("  %s\n", t.name.c_str());

    std::printf("== 3. unordered_map：按名字查贴图缓存 ==\n");
    std::unordered_map<std::string, Texture> cache;
    cache.reserve(1024);                    // 预留桶：预判 1024 条内不 rehash
    cache.emplace("hero.png",  Texture{512 * 1024});
    cache.emplace("slime.png", Texture{128 * 1024});
    cache.emplace("tiles.png", Texture{256 * 1024});
    std::printf("  bucket_count=%zu load_factor=%.2f\n",
                cache.bucket_count(), cache.load_factor());
    auto it = cache.find("slime.png");      // 平均 O(1)
    if (it != cache.end()) std::printf("  slime.png 占 %zu KB\n", it->second.bytes / 1024);

    std::printf("== 4. unordered_map 遍历顺序：不稳定（哈希序）==\n");
    for (const auto& [name, tex] : cache)  // C++17 结构化绑定（第 1 章 1.9）
        std::printf("  %s (%zu KB)\n", name.c_str(), tex.bytes / 1024);

    std::printf("== 5. map：排行榜（按分数有序）==\n");
    std::map<int, ScoreEntry, std::greater<int>> board;  // greater：分数从高到低
    board.emplace(980, ScoreEntry{"玩家A", 980});
    board.emplace(1200, ScoreEntry{"玩家B", 1200});
    board.emplace(845, ScoreEntry{"玩家C", 845});
    for (const auto& [score, entry] : board)           // 遍历天然有序
        std::printf("  %d 分  %s\n", score, entry.player.c_str());
    return 0;
}
```

实测输出：

```text
== 1. vector：背包物品（尾部增删 + 按位访问）==
  药水 x8
  卷轴 x2
== 2. deque：每帧任务队列（两头都进）==
  读输入
  渲染天空盒
  更新粒子
== 3. unordered_map：按名字查贴图缓存 ==
  bucket_count=1024 load_factor=0.00
  slime.png 占 128 KB
== 4. unordered_map 遍历顺序：不稳定（哈希序）==
  hero.png (512 KB)
  slime.png (128 KB)
  tiles.png (256 KB)
== 5. map：排行榜（按分数有序）==
  1200 分  玩家B
  980 分  玩家A
  845 分  玩家C
```

三个值得停下来讲的隐性成本：

**vector 的搬家税（预埋下一节）。** `push_back` 超容量就整体搬家（第 1 章深入专题解剖过的那场搬家）。搬家本身均摊后不贵，贵的是它引发的**失效连带**——所有指向元素的指针、引用、迭代器集体悬空。3.2 节给完整规则表。

**deque 的分段存储。** 典型实现是一串定长「段」+ 一张段表：

```text
段表:  [ptr0] [ptr1] [ptr2] ...
         ↓      ↓      ↓
       [a0..aN][b0..bN][c0..cN]     ← 元素住在各段里，段间不保证相邻
```

首尾插删只动段表两端，元素地址稳如泰山——代价是随机访问要「先查段表再进段」，两跳。**遍历模式决定容器命运**的伏笔这里先埋一句：连续性对缓存的意义在第 6 章展开。

**unordered_map 的 rehash 与装载因子。** 哈希桶数量随元素增长重建（rehash），触发条件是装载因子（元素数/桶数）超过上限（默认 `max_load_factor = 1.0`）。示例里 `cache.reserve(1024)` 是预判式预留：1024 条以内不 rehash——对「启动时批量灌缓存」的场景，一次 `reserve` 省掉 log 次全表重摆。rehash 的失效规则（迭代器全灭、元素引用/指针却不动）是 3.2 的重点反直觉条目。

还有一条纪律级的提醒：`unordered_map` 遍历顺序不稳定，**任何依赖遍历序的逻辑（回放、网络同步、逐帧 diff）都是 bug 工厂**；需要顺序就选 `map`（按键有序，O(log n)）或者自己维护一个「键 → 序号」的 vector 副本。另外本节的对比只谈复杂度量级，「实测耗时怎么测才可信」是测量方法论问题，第 6 章展开。

### 3.2 迭代器失效规则：把第 1 章的 HUD 崩溃升级成完整规则表

第 1 章 1.11 的 HUD 崩溃还记得吗：游戏对象从 `vector` 搬了家，HUD 缓存的指向血条的裸指针一夜白头。当时我们只给了两条应急修复（缓存下标、提前 `reserve`），并把「完整失效规则表」的账挂到了本章。现在兑现。

先把机理说透，规则表就不用死记了。**失效 = 你记录的「位置凭证」指向了旧地皮。** 三种凭证的风险天差地别：

- **指针/引用**指向元素本体——只有元素**搬家或销毁**时失效；
- **迭代器**是「容器认证的位置凭证」（对 vector 是裸指针，对 map 是带树信息的节点句柄）——除了搬家/销毁，**容器结构性调整**（rehash、树旋转后的语义约束）也会让它失灵，哪怕元素安然无恙；
- **下标**是唯一不碰地址的凭证——只有「位置前的元素被删」时才错位。

完整规则表（以 cppreference「Iterator invalidation」页为准；✗=失效，✓=仍有效；插入均指单个元素）：

| 容器 | 操作 | 迭代器 | 引用/指针 | 备注 |
| --- | --- | --- | --- | --- |
| `vector` | 插入（未触发扩容） | 插入点**之前** ✓，之后 ✗ | 同左 | 尾插未扩容时全 ✓（仅 `end()` 失效） |
| `vector` | 插入（触发扩容） | 全部 ✗ | 全部 ✗ | 第 1 章的搬家现场 |
| `vector` | 删除 | 删除点及之后 ✗ | 同左 | 被删元素起全员前挪 |
| `deque` | 首尾插入 | 全部 ✗ | **全部 ✓** | 迭代器带「全局位置」语义，首插会作废它 |
| `deque` | 中间插入 | 全部 ✗ | 全部 ✗ | 两段都可能搬 |
| `deque` | 首尾删除 | 仅被删处 ✗ | 同左 | |
| `list` | 插入 | 全部 ✓ | 全部 ✓ | 节点式容器的天生优势 |
| `list` | 删除 | 仅被删处 ✗ | 同左 | |
| `map`/`set` | 插入 | 全部 ✓ | 全部 ✓ | 红黑树节点插入不挪老节点 |
| `map`/`set` | 删除 | 仅被删处 ✗ | 同左 | |
| `unordered_map` | 插入（未 rehash） | 全部 ✓ | 全部 ✓ | |
| `unordered_map` | 插入（触发 rehash） | **全部 ✗** | **全部 ✓** | **反直觉重点**：桶重摆，元素本体没动 |
| `unordered_map` | 删除 | 仅被删处 ✗ | 同左 | |

两张防呆卡：**「迭代器失效 ≠ 元素失效」**（unordered_map rehash 行）——元素还活着，但你手里的迭代器已经作废；**「失效不等于必崩」**——读失效迭代器是未定义行为，可能读到貌似正常的旧值、可能崩、也可能什么都不发生，UB 不归你预测（第 1 章的原话：把 UB 当抽奖，总有一天头奖是自己）。上表用完整示例过一遍：

```cpp
// ch03_invalidation.cpp —— 迭代器/指针/引用什么时候失效（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_invalidation.cpp
#include <cstdio>
#include <map>
#include <string>
#include <unordered_map>
#include <vector>

int main() {
    std::printf("== 1. vector 扩容 = 整体搬家：指向元素的指针全部悬空 ==\n");
    std::vector<int> v;
    v.reserve(2);                       // 先把容量压到 2，逼出下一次扩容
    v.push_back(10);
    v.push_back(20);
    const int* pFirst = v.data();       // 缓存了首元素地址（第 1 章 1.11 的 HUD 干的事）
    std::printf("  扩容前: data=%p 元素=%d\n", (const void*)pFirst, *pFirst);
    v.push_back(30);                    // 第 3 个元素：容量 2 不够 → 搬家
    std::printf("  扩容后: data=%p（新地址）\n", (const void*)v.data());
    std::printf("  结论：旧指针指向老地皮，解引用是未定义行为——别学别用\n");
    pFirst = v.data();                  // 修复方式：搬家后重新取

    std::printf("== 2. map 插入：别人的迭代器/指针毫发无损 ==\n");
    std::map<std::string, int> hp;
    hp["hero"] = 100;
    const int* stable = &hp.begin()->second;           // 缓存 hero 血量的地址
    hp["slime"] = 10;                                  // 再插一条
    std::printf("  插入 slime 后 hero.hp 仍是 %d（地址 %p 依然有效）\n",
                *stable, (const void*)stable);

    std::printf("== 3. unordered_map rehash：迭代器全灭，元素指针却不动 ==\n");
    std::unordered_map<int, std::string> saves;
    saves[1] = "存档一";
    const std::string* elem = &saves[1];               // 指向「元素」的指针
    std::printf("  rehash 前: bucket_count=%zu 元素=%s@%p\n",
                saves.bucket_count(), elem->c_str(), (const void*)elem);
    for (int i = 2; i <= 64; ++i) saves[i] = "存档";    // 连续插入，必然触发 rehash
    std::printf("  rehash 后: bucket_count=%zu 元素=%s@%p（地址没变！）\n",
                saves.bucket_count(), elem->c_str(), (const void*)elem);
    std::printf("  但指向这里的「迭代器」已全部失效——规则见正文表格\n");

    std::printf("== 4. erase 的半失效：删除点之后全部失效 ==\n");
    std::vector<int> scores = {50, 60, 70, 80, 90};
    auto it = scores.begin() + 1;       // 指向 60
    std::printf("  删除前: *it=%d\n", *it);
    scores.erase(scores.begin());       // 删掉 50：其后元素整体前挪
    // std::printf("%d", *it);          // ⚠ 反面教材（不执行）：it 已失效，解引用是未定义行为
    it = scores.begin() + 1;            // 正解：修改容器后重新取迭代器
    std::printf("  重新取后: *it=%d\n", *it);
    return 0;
}
```

实测输出（地址类数值每次运行不同，看「变没变」而非数值本身；MSVC x64）：

```text
== 1. vector 扩容 = 整体搬家：指向元素的指针全部悬空 ==
  扩容前: data=0000017602749CA0 元素=10
  扩容后: data=0000017602747400（新地址）
  结论：旧指针指向老地皮，解引用是未定义行为——别学别用
== 2. map 插入：别人的迭代器/指针毫发无损 ==
  插入 slime 后 hero.hp 仍是 100（地址 000001760274BAB0 依然有效）
== 3. unordered_map rehash：迭代器全灭，元素指针却不动 ==
  rehash 前: bucket_count=8 元素=存档一@000001760273BA58
  rehash 后: bucket_count=64 元素=存档一@000001760273BA58（地址没变！）
  但指向这里的「迭代器」已全部失效——规则见正文表格
== 4. erase 的半失效：删除点之后全部失效 ==
  删除前: *it=60
  重新取后: *it=70
```

注意第 1 段我们**没有**真的去解引用搬家前的旧指针——那行代码以反面教材注释的形式留在纸上，因为「演示 UB」和「演示失效」是两回事：地址变了就是失效的铁证，不需要再踩一脚。

收尾三条工程戒律（第 1 章 1.9 的「范围 for 里不增删元素」在此正式归队，它是第三条的特例）：

1. **跨「容器修改」存活的缓存，缓存下标/ID，不缓存指针/引用/迭代器**——下标最钝感，ID 连下标错位都免疫（第 1 章 1.11 的修复方案一，现在你知道为什么它是首选）。
2. **修改容器后，重新取一切位置凭证**——`it = c.erase(it)` 这种「用返回值续命」是最常见的正确姿势。
3. **遍历与结构修改不共舞**——范围 for 循环体里 `push_back/erase` 该容器，等于在飞行中给飞机换轮子；需要边遍历边删，用 3.4 的 remove-erase，或换 `std::map` 这类「插入不伤人」的容器。

### 3.3 string、string_view 与 SSO

第 2 章 2.9 的借用词汇表把 `std::string_view` 列为「借一段字符串」，并承诺选型细节在本章展开。兑现它，外加一个常被忽略的事实：你以为你了解 `std::string`，但它的第一性格其实是 **SSO（Small String Optimization，小字符串优化）**。

**事实一：短字符串根本不碰堆。** MSVC x64 下 `sizeof(std::string)` 是 32 字节，其中一段空间直接当缓冲用——不超过 15 字符的字符串就住在对象肚子里，构造、拷贝、析构**零堆分配**；超长才下堆（libstdc++ 同为 32/15，libc++ 是 24/22，实现细节，以实测为准）。实证：

```cpp
// ch03_string_sso.cpp —— string 的 SSO 与 string_view 的借用边界（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_string_sso.cpp
#include <cstdio>
#include <cstdlib>
#include <new>
#include <string>
#include <string_view>

// 全局分配计数器：复用第 2 章深入专题的手法，数一数「短串到底动没动堆」
static std::size_t g_allocs = 0;
void* operator new(std::size_t n) {
    ++g_allocs;
    return std::malloc(n);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

// string_view 传参：不拷贝、不管你从哪来（string / 字面量 / char 数组全收）
std::size_t nameBytes(std::string_view sv) { return sv.size(); }

int main() {
    std::printf("== 1. SSO：短字符串住在对象肚子里，零堆分配 ==\n");
    std::printf("  sizeof(std::string)=%zu（MSVC x64 口径，实现细节）\n", sizeof(std::string));
    g_allocs = 0;
    std::string shortName = "hero.png";               // 8 字符 < SSO 容量 15
    std::printf("  构造短串: 堆分配次数=%zu\n", g_allocs);
    std::printf("  data=%p 对象地址=%p（紧贴对象内部 → 短串就住在栈上的 string 里）\n",
                (const void*)shortName.data(), (const void*)&shortName);

    std::printf("== 2. 长字符串才真正下堆 ==\n");
    g_allocs = 0;
    std::string longName(64, 'x');                    // 64 字符 > 15，触发堆分配
    std::printf("  构造长串: 堆分配次数=%zu capacity=%zu\n", g_allocs, longName.capacity());

    std::printf("== 3. string_view：一视同仁的零拷贝传参 ==\n");
    std::printf("  nameBytes(string) = %zu\n", nameBytes(shortName));
    std::printf("  nameBytes(字面量) = %zu\n", nameBytes("slime.png"));
    std::string_view sv = shortName;                  // view 指向 shortName 的缓冲
    sv.remove_prefix(5);                              // 零拷贝切掉前缀 "hero."
    std::printf("  remove_prefix 后 view=%s（原 string 没动）\n", std::string(sv).c_str());

    std::printf("== 4. string_view 不能过夜（反面教材，已注释）==\n");
    // std::string_view dangling = std::string("temp");  // ⚠ 不执行：
    // 临时 string 在这一句结束时析构，dangling 从此是悬垂视图
    std::string_view param = shortName;               // 正解：只借「活得比自己久」的串
    (void)param;
    return 0;
}
```

实测输出（地址每次运行不同；capacity=79 是 MSVC 的分配取整，又是实现细节）：

```text
== 1. SSO：短字符串住在对象肚子里，零堆分配 ==
  sizeof(std::string)=32（MSVC x64 口径，实现细节）
  构造短串: 堆分配次数=0
  data=00000001000FFCE0 对象地址=00000001000FFCE0（紧贴对象内部 → 短串就住在栈上的 string 里）
== 2. 长字符串才真正下堆 ==
  构造长串: 堆分配次数=1 capacity=79
== 3. string_view：一视同仁的零拷贝传参 ==
  nameBytes(string) = 8
  nameBytes(字面量) = 9
  remove_prefix 后 view=png（原 string 没动）
== 4. string_view 不能过夜（反面教材，已注释）==
```

SSO 解释了两个日常现象：为什么「到处按值传 `std::string`」在短串场景下没想象中慢（拷贝 32 字节，无分配）——但**别拿它当许可**：长路径、贴图名、日志串随时越线，且逐帧热路径上「没分配」和「零成本」仍是两回事（分配的隐藏成本第 6 章细算）；也解释了为什么 `data()` 的地址有时在对象里、有时在堆上——同一个类型，两种存储，随内容长度切换。

**事实二：`string_view` 是「指针 + 长度」，它的全部义务是「借」。** 作为参数它一视同仁：`std::string`、字面量、`char` 数组、另一个 view 都能隐式变成它，零拷贝；`remove_prefix/remove_suffix/substr` 都是 O(1) 视图操作（对比 `std::string::substr` 要拷贝一份新串）。但它有三条铁律：

1. **不保证 NUL 结尾**——喂给 `fopen(path.c_str())` 这类 C API 前先转 `std::string` 或确认来源；给图形 API 喳 shader 源码（带长度的指针对）反而是 view 的主场。
2. **借用不延长生存期**（第 2 章 2.9 的结论原样适用）——绑了临时 string 就是悬垂；**view 几乎只应出现在函数形参和局部变量里**，存成成员或跨帧持有 = 埋雷。
3. **它不知道字符串语义**——`view` 可能指向栈、堆、静态区，比较与哈希按内容进行，但生命周期归原主。

选型口诀（兑付第 2 章的承诺⑥前半，`span` 侧在 3.8 收口）：**函数参数默认 `string_view`；要保存、要修改、要过 NUL 结尾的 C API，用 `std::string`。** 引擎停留在 C++17？`string_view` 本身就是 C++17 的，放心用（C++14 及以下的老工程才需要 `const std::string&` 兼容）。

### 3.4 算法库优先于手写循环：sort、find_if、transform、remove-erase

标准算法库上百个算法，本节用一条粒子系统流水线串起四个代表，但先立住论点：**手写循环是 bug 的温床，标准算法是把意图写进代码的命名工具。** `std::sort` 三个字告诉读者「这是排序、复杂度 O(n log n)、不会越界」；三行 `for` 循环告诉读者「这里有三个变量、两个边界和一个可能写反的不等式，祝你好运」。此外算法还能在编译期根据迭代器类别选更优实现——这是手写永远拿不到的免费午餐。

上流水线（过滤死粒子 → 按距离排序绘制 → 找目标 → 批量结算伤害 → 汇总）：

```cpp
// ch03_algorithms.cpp —— 标准算法替换手写循环：粒子系统一条龙（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_algorithms.cpp
#include <algorithm>
#include <cstdio>
#include <numeric>
#include <vector>

struct Particle {
    int   id;
    float dist;      // 距相机距离（绘制排序用）
    int   hp;        // 生命值，<= 0 视为死亡
};

// 比较器：先写成语义化的仿函数（下一节的 lambda 会给它减负）
struct FarToNear {
    bool operator()(const Particle& a, const Particle& b) const { return a.dist > b.dist; }
};

int main() {
    std::vector<Particle> ps = {
        {1, 12.5f, 3}, {2, 3.0f, 0}, {3, 8.8f, 5}, {4, 20.1f, 0}, {5, 5.5f, 1},
    };

    std::printf("== 1. remove-erase：剔除死亡粒子 ==\n");
    std::printf("  删除前 size=%zu\n", ps.size());
    auto newEnd = std::remove_if(ps.begin(), ps.end(),
                                 [](const Particle& p) { return p.hp <= 0; });
    std::printf("  remove_if 后 size=%zu（没变！只是把活的全搬到了前面）\n", ps.size());
    for (auto it = newEnd; it != ps.end(); ++it)
        std::printf("    逻辑尾残余: id=%d\n", it->id);
    ps.erase(newEnd, ps.end());                          // 第二步：物理删除
    std::printf("  erase 后 size=%zu\n", ps.size());

    std::printf("== 2. sort：按距离从远到近排绘制顺序 ==\n");
    std::sort(ps.begin(), ps.end(), FarToNear{});        // 远的先画（画家算法）
    for (const Particle& p : ps)
        std::printf("    id=%d dist=%.1f\n", p.id, p.dist);

    std::printf("== 3. find_if：按 id 找粒子 ==\n");
    auto hit = std::find_if(ps.begin(), ps.end(),
                            [](const Particle& p) { return p.id == 3; });
    if (hit != ps.end()) std::printf("    找到 id=3 dist=%.1f\n", hit->dist);

    std::printf("== 4. transform：批量结算灼烧伤害 ==\n");
    std::vector<float> damages(ps.size(), 0.0f);
    std::transform(ps.begin(), ps.end(), damages.begin(),
                   [](const Particle& p) { return p.dist * 2.0f; });
    std::printf("    首个伤害值=%.1f\n", damages.front());

    std::printf("== 5. accumulate：汇总存活粒子的总血量 ==\n");
    int total = std::accumulate(ps.begin(), ps.end(), 0,
                                [](int acc, const Particle& p) { return acc + p.hp; });
    std::printf("    总血量=%d\n", total);

    std::printf("== 6. C++20 一行版：std::erase_if ==\n");
    std::erase_if(ps, [](const Particle& p) { return p.id == 5; });
    std::printf("    erase_if 后 size=%zu\n", ps.size());
    return 0;
}
```

实测输出：

```text
== 1. remove-erase：剔除死亡粒子 ==
  删除前 size=5
  remove_if 后 size=5（没变！只是把活的全搬到了前面）
    逻辑尾残余: id=4
    逻辑尾残余: id=5
  erase 后 size=3
== 2. sort：按距离从远到近排绘制顺序 ==
    id=1 dist=12.5
    id=3 dist=8.8
    id=5 dist=5.5
== 3. find_if：按 id 找粒子 ==
    找到 id=3 dist=8.8
== 4. transform：批量结算灼烧伤害 ==
    首个伤害值=25.0
== 5. accumulate：汇总存活粒子的总血量 ==
    总血量=9
== 6. C++20 一行版：std::erase_if ==
    erase_if 后 size=2
```

**本节重头戏：remove-erase 惯用法。** 为什么 `std::remove_if` 之后 `size()` 纹丝不动？因为 remove 系算法根本**无权删除元素**——它只有迭代器，而删除要动容器的 `size`（那是容器成员函数的职权，算法拿到的只是「范围的两端」）。于是它的语义是：把「留下」的元素逐个往前搬，返回**新逻辑尾**迭代器；逻辑尾之后的残余元素处于「值仍有效但已不重要」的状态（示例里被打印出来的 id=4/id=5）。物理删除必须补第二刀 `erase`。C++20 给了这个两步舞一个正式的一步版：`std::erase_if(c, pred)`（自由函数、按容器重载、返回删除个数）——**新代码优先**；经典两步仍要会读，老代码里遍地都是。C++17 的写法就是上面那个两步惯用法本身。

第二课：**迭代器类别是算法的入场券。** `std::sort` 需要随机访问迭代器（要能 `it + n`、`last - first` 跳着访问）；`std::list` 只提供双向迭代器，只能一步步走。把示例第 2 段的容器换成 `std::list` 再 `std::sort`，真实报错长这样：

```text
编译输出节选（MSVC 19.44，中文语言包；行号与中间行已截断，以你本机输出为准）：
...\include\algorithm(8404): error C2676: 二进制“-”: “const std::_List_unchecked_iterator<
    std::_List_val<std::_List_simple_types<Particle>>>”不定义该运算符或到预定义
    运算符可接收的类型的转换
...\include\xutility(4528): note: 可能是“unknown-type std::operator -(...”
```

读法：报错中心是「list 迭代器不定义 `-`」——`sort` 内部要算 `last - first`（区间长度），链表迭代器给不出这个减法；类型名 `std::_List_unchecked_iterator` 就是在场证明。这就是「迭代器类别」的具体含义：**算法在编译期检查入场券，缺能力就报错**——比运行期静默出错好得多。正解：`std::list` 自带成员函数 `list::sort`（链表归并实现，不受迭代器类别限制）。其余两个必修知识点：`std::sort` **不稳定**（相等元素的相对顺序不保证，保序用 `std::stable_sort`，代价是额外内存）；`find_if` 返回 `end()` 表示没找到——**永远先比对 `end()` 再解引用**。

到这里你还欠一个疑问：那些 `[](...) { ... }` 是什么？下一节正式回答，并顺手给本节的 `FarToNear` 减负。

### 3.5 lambda 与捕获的完整机制：按值、按引用、init capture

上一节结尾的疑问现在回答：`[](const Particle& p) { return p.hp <= 0; }` 是 lambda 表达式——**编译器替你现场生成的一个匿名仿函数类 + 一个对象**，`[]` 里写的是它要记住哪些变量。第 2 章 2.8 讲过它的一个切片（`[this]` 捕获的保命组合拳），本节把完整机制一次讲透：捕获什么、怎么捕获、活多久。三问不清，lambda 就是悬垂发生器；三问清楚，它就是 C++ 里最锋利的胶水。

先看完整示例，再拆机制：

```cpp
// ch03_lambda_capture.cpp —— lambda 捕获的完整机制（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_lambda_capture.cpp
#include <cstdio>
#include <string>
#include <utility>
#include <vector>

int main() {
    std::printf("== 1. lambda 是编译器生成的仿函数对象，先看它多大 ==\n");
    auto noop = [] {};                               // 空捕获：无状态
    std::printf("  sizeof(空捕获 lambda)=%zu（空对象也要占 1 字节撑门面）\n", sizeof(noop));

    std::printf("== 2. 按值捕获：拷贝进闭包，各玩各的 ==\n");
    int frame = 60;
    auto byValue = [frame] { std::printf("  闭包里的 frame=%d\n", frame); };
    frame = 120;                                     // 改原变量
    byValue();                                       // 闭包里的还是 60

    std::printf("== 3. mutable：想改按值捕获的副本，得先声明 ==\n");
    auto counter = [count = 0]() mutable {           // C++14 init capture：顺便演示
        return ++count;
    };
    counter(); counter();
    std::printf("  闭包内计数=%d（副本活着，跨调用持久）\n", counter());

    std::printf("== 4. 按引用捕获：借用，不延长生存期 ==\n");
    int fps = 60;
    std::printf("  sizeof(按引用捕获 int)=%zu（闭包里只存了一个指针的宽度）\n",
                sizeof([&fps] { return fps; }));
    // auto dangling() { int local = 1; return [&] { return local; }; }
    // ⚠ 反面教材（不执行）：返回捕获局部变量的 lambda —— local 一死，引用就悬垂

    std::printf("== 5. init capture：把大缓冲「搬」进闭包 ==\n");
    std::vector<char> upload(1024 * 1024, 'x');      // 1MB 上传缓冲
    const void* before = upload.data();
    auto task = [buf = std::move(upload)] {          // 移动语义（第 1 章）的捕获版
        std::printf("  闭包拿到 %zu 字节缓冲\n", buf.size());
    };
    std::printf("  搬家前 data=%p（闭包与原变量是同一块堆内存）\n", before);
    task();
    std::printf("  搬家后原 vector data=%p（所有权已移交，原变量空了）\n",
                (const void*)upload.data());

    std::printf("== 6. 捕获开销速览（MSVC x64 实测口径）==\n");
    std::printf("  空 lambda=%zu  值捕 int=%zu  值捕两个 int=%zu  init capture 挪 vector=%zu\n",
                sizeof(noop), sizeof([f = 1] { (void)f; }),
                sizeof([a = 1, b = 2] { (void)a; (void)b; }),
                sizeof(task));
    return 0;
}
```

实测输出（地址每次运行不同）：

```text
== 1. lambda 是编译器生成的仿函数对象，先看它多大 ==
  sizeof(空捕获 lambda)=1（空对象也要占 1 字节撑门面）
== 2. 按值捕获：拷贝进闭包，各玩各的 ==
  闭包里的 frame=60
== 3. mutable：想改按值捕获的副本，得先声明 ==
  闭包内计数=3（副本活着，跨调用持久）
== 4. 按引用捕获：借用，不延长生存期 ==
  sizeof(按引用捕获 int)=8（闭包里只存了一个指针的宽度）
== 5. init capture：把大缓冲「搬」进闭包 ==
  搬家前 data=000001FB3AB87060（闭包与原变量是同一块堆内存）
  闭包拿到 1048576 字节缓冲
  搬家后原 vector data=0000000000000000（所有权已移交，原变量空了）
== 6. 捕获开销速览（MSVC x64 实测口径）==
  空 lambda=1  值捕 int=4  值捕两个 int=8  init capture 挪 vector=24
```

编译器眼里的 lambda 没有任何魔法。`[frame] { ... }` 大约展开成这样（对照第 3 段的 `counter`，一比一对应）：

```cpp
// 节选：编译器生成的等价物（示意）
struct __AnonymousCounter {
    int count;                                 // 捕获的变量成为成员
    int operator()() { return ++count; }       // 函数体进 operator()，mutable 去掉 const
};
// auto counter = [count = 0]() mutable {...} 等价于：__AnonymousCounter counter{0};
```

所以「闭包」就是一个对象，`sizeof` 实测才是理解捕获开销的钥匙：空捕获 1 字节，值捕 int 加 4，按引用捕获只存 8 字节指针，init capture 挪进去的 `vector` 也只占 24 字节——**闭包存的是 vector 对象本身，不是那 1MB 堆内存**，缓冲区所有权随移动语义移交（搬家前 data 指针与闭包共用同一块堆，原 `vector` 移交后变空——第 1 章的移动语义在捕获语法里的原样重演）。

三问的完整答案：

1. **捕什么**：`[frame]` 显式点名永远优于 `[=]`/`[&]` 一把拟。EMC++ 条 31 的警告原样适用：`[=]` 捕局部变量是拷贝，但碰上成员变量实际捕的是 **`this` 裸指针**（第 2 章 2.8 的悬垂回调原样复发）；`[&]` 则把作用域里所有变量全变成借用——回调一旦逃出当前作用域（注册进队列、跨帧执行），全员悬垂。
2. **怎么捕**：按值 `[x]`（拷贝，`mutable` 才能改副本）、按引用 `[&x]`（借用，不延长生存期）、init capture `[buf = std::move(buf)]`（C++14，表达式初始化成员——「把对象**移**进闭包」的标准姿势，也是把不可拷贝对象（如 `unique_ptr`）带进闭包的唯一途径）。
3. **活多久**：闭包的生命周期由接住它的那个变量/容器决定；被借的变量不会因为被捕获就多活一纳秒。跨作用域登记回调，只允许两种组合：**按值捕（或 init capture 移入）**，或第 2 章的 `weak_from_this()` 组合拳。

（真异步、跨线程的回调队列归第 5 章；本章示例全部单线程。）

### 3.6 可调用对象全景：函数指针、仿函数、std::bind 与 std::invoke

「可调用物」（callable）是能跟上一对括号的东西。C++ 里至少五种：函数、函数指针、仿函数、lambda、成员函数指针。它们**类型各异却都能调**——本节先让它们同框，再补两个把混乱变成秩序的工具：`std::invoke`（统一怎么调）与 `std::bind`（提前绑参数）。

```cpp
// ch03_callables.cpp —— 可调用对象全景：五种可调用物同框 + bind/invoke（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_callables.cpp
#include <cstdio>
#include <functional>
#include <string>
#include <utility>

struct Target {                       // 被打的目标
    explicit Target(std::string n) : name(std::move(n)) {}
    std::string name;
    int hp = 1000;
};

// 形态一：普通函数（函数指针）
void slash(Target& t) { std::printf("  [函数指针] 挥砍 %s\n", t.name.c_str()); }

// 形态二：仿函数（重载 operator() 的类对象）
struct Fireball {
    int damage;
    void operator()(Target& t) const
        { std::printf("  [仿函数] 火球 %s -%d\n", t.name.c_str(), damage); }
};

class Wizard {                        // 成员函数也能当可调用物（配 std::invoke/bind 用）
public:
    void cast(Target& t, int damage) { std::printf("  [成员函数] %s 施法 %s -%d\n",
                                                   title_.c_str(), t.name.c_str(), damage); }
    void cast([[maybe_unused]] Target& t, const char* spell) {   // [[maybe_unused]]：重载消歧演示里 t 不参与打印，压掉 /W4 的 C4100
        std::printf("  [成员函数·重载] %s 念唱 %s\n", title_.c_str(), spell); }
private:
    std::string title_ = "大法师";
};

int main() {
    Target dragon("巨龙");

    std::printf("== 1. 五种可调用物，同一个调用姿势 obj(args) ==\n");
    void (*fp)(Target&) = slash;              // 形态一：函数指针
    fp(dragon);
    Fireball fire{50};                        // 形态二：仿函数
    fire(dragon);
    auto bite = [&dragon](int dmg) {          // 形态三：lambda（闭包）
        dragon.hp -= dmg;
        std::printf("  [lambda] 撕咬 剩余 %d\n", dragon.hp);
    };
    bite(30);
    Wizard mage;                              // 形态四：成员函数指针（要配对象用）
    void (Wizard::*memFn)(Target&, int) = &Wizard::cast;

    std::printf("== 2. std::invoke：统一调用语法（成员指针这种最啰嗦的形态也能一行调）==\n");
    std::invoke(fp, dragon);
    std::invoke(fire, dragon);
    std::invoke(memFn, mage, dragon, 70);     // (对象.*memFn)(实参) 的等价写法

    std::printf("== 3. std::bind：占位符重排参数 ==\n");
    using namespace std::placeholders;        // _1, _2 ...（namespace std::placeholders）
    auto hitDragon = std::bind(memFn, &mage, _1, 40);   // 预绑对象与伤害，目标留白
    hitDragon(dragon);                        // 调用时才提供 _1 的位置

    std::printf("== 4. bind 默认按值拷贝实参：std::ref 才是按引用 ==\n");
    int bonus = 5;
    auto byValue = std::bind([](int b) { std::printf("  byValue 伤害加成=%d\n", b); }, bonus);
    auto byRef   = std::bind([](int& b) { std::printf("  byRef   伤害加成=%d\n", b); },
                             std::ref(bonus));
    bonus = 99;                               // 之后再改外部变量
    byValue();                                // 拷贝进去的是旧值 5
    byRef();                                  // 引用绑定看到的是 99

    std::printf("== 5. 重载成员函数交给 bind/invoke：先消歧 ==\n");
    void (Wizard::*castInt)(Target&, int) = &Wizard::cast;              // 挑出 int 版
    void (Wizard::*castSpell)(Target&, const char*) =
        static_cast<void (Wizard::*)(Target&, const char*)>(&Wizard::cast);
    std::invoke(castInt, mage, dragon, 15);
    std::invoke(castSpell, mage, dragon, "烈烽风暴");    // static_cast 挑出字符串版

    std::printf("== 6. 等价 lambda 对照（工程默认写法）==\n");
    auto hitDragonLambda = [&mage, &dragon] { mage.cast(dragon, 40); };
    hitDragonLambda();
    return 0;
}
```

实测输出：

```text
== 1. 五种可调用物，同一个调用姿势 obj(args) ==
  [函数指针] 挥砍 巨龙
  [仿函数] 火球 巨龙 -50
  [lambda] 撕咬 剩余 970
== 2. std::invoke：统一调用语法（成员指针这种最啰嗦的形态也能一行调）==
  [函数指针] 挥砍 巨龙
  [仿函数] 火球 巨龙 -50
  [成员函数] 大法师 施法 巨龙 -70
== 3. std::bind：占位符重排参数 ==
  [成员函数] 大法师 施法 巨龙 -40
== 4. bind 默认按值拷贝实参：std::ref 才是按引用 ==
  byValue 伤害加成=5
  byRef   伤害加成=99
== 5. 重载成员函数交给 bind/invoke：先消歧 ==
  [成员函数] 大法师 施法 巨龙 -15
  [成员函数·重载] 大法师 念唱 烈烽风暴
== 6. 等价 lambda 对照（工程默认写法）==
  [成员函数] 大法师 施法 巨龙 -40
```

**`std::invoke` 的价值一句话：把「怎么调」从语法噪音里解放出来。** 普通函数、仿函数、lambda 是 `f(args)`，成员函数指针却是 `(obj.*memFn)(args)`——模板代码里想「拿到什么都能调」就得两种语法都写。`std::invoke(f, obj, args...)` 一套语法吃下所有形态（3.13 的 concepts 小节还会用它写「可被这样调用」的约束）。

**`std::bind`：先完整学会，再决定不用。** 它做的事叫「部分应用」：把多参函数的一部分参数提前绑定、留几个占位符（`_1`/`_2`，在 `std::placeholders` 里）以后填。三个必知细节，示例里各就各位：① 绑定成员函数时第一个参数是成员指针、第二个是对象（或对象指针）；② **实参默认按值拷贝**——想要引用必须 `std::ref`/`std::cref` 包一层，示例第 4 段的 `5 → 99` 就是这条规则的现场证据；③ 函数有重载时，`&Wizard::cast` 本身有歧义，要用 `static_cast` 指定成员函数指针类型消歧。嵌套 bind（bind 的结果再作另一个 bind 的实参）语法上合法，可读性急速崩塌，这里点到为止——你只需要在读到时能认出来。bind 还能把参数**重排**（`std::bind(f, _2, _1)` 交换两个实参的位置），这是它比 `bind_front` 多出的能力。

**工程立场三行**（与 EMC++ 条 34、abseil Tip of the Week #108《Avoid std::bind》一致）：

1. **新代码默认 lambda**：显式列出捕获，读代码的人一眼看清「活了多久、拷了什么、借了什么」；泛型 lambda（`[](auto x)`，C++14 起）能力更强；还能内联。
2. **最常见的「绑前几个参数」用 `std::bind_front`**（C++20）：语义清晰、不吞 `std::ref` 那套坑；但它不支持占位符重排——那也是 bind 唯一的存留场景（而重排参数的代码几乎总该重写成 lambda）。
3. **bind 只需「读得懂遗留代码」**——检验标准里要求你会写含占位符与 `std::ref` 的绑定并给等价 lambda，本节和自测第 5 题都在练这个。

还剩一位主角没登场：五种可调用物**类型各异**，想把它们装进同一个容器当回调存起来怎么办？这是 `std::function` 的主场——下一节。

### 3.7 std::function 的类型擦除：开销边界 vs 模板参数

先回答「为什么需要它」。3.6 说过五种可调用物类型各异：每个 lambda 都有自己独一无二的类型（编译器生成的匿名类），`int(*)(int)` 和仿函数 `Acc` 更是八竿子打不着。想把「签名相同的各种可调用物」**装进同一个容器**、放进**同一个成员变量**、或穿过**已编译好的模块边界**——模板参数做不到（模板要求编译期就知道具体类型），函数指针装不下带状态的 lambda。唯一解是**类型擦除**：把「可调用物」连同「怎么调它」一起打包，对外只留一个统一的壳。`std::function<void(Particle&)>` 就是标准库的壳。

擦除不是免费的。机制层面（黑盒级；完整解剖在深入专题手写一遍）：

- **构造时**：闭包被拷/移进不透明存储。小闭包塞进对象内的小缓冲；**超出缓冲就堆分配**（MSVC x64 实测 `sizeof(std::function<void(Particle&)>)=64`，48 字节以内零分配，56 字节即下堆——阈值实测见深入专题）；
- **调用时**：经一次函数指针间接跳转——编译器通常**无法内联**目标函数；
- **拷贝时**：经由另一个间接的管理函数深拷贝。

同一个回调调用点，两版写法摆在一起看（完整示例含全局 `operator new` 分配计数——第 2 章深入专题的手法，数出堆分配的现场）：

```cpp
// ch03_function_vs_template.cpp —— std::function 的类型擦除代价 vs 模板参数（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_function_vs_template.cpp
#include <cstdio>
#include <cstdlib>
#include <functional>
#include <new>
#include <vector>

static std::size_t g_allocs = 0;                  // 复用第 2 章深入专题的分配计数手法
void* operator new(std::size_t n) { ++g_allocs; return std::malloc(n); }
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

struct Particle { float x = 0.0f, y = 0.0f; };
struct BigState { char bytes[128] = {}; };   // 128 字节状态：故意把闭包撑大到小缓冲装不下

// 调用点版本 A：std::function —— 闭包被类型擦除，调用走间接
void stepAll(std::function<void(Particle&)> behavior, std::vector<Particle>& ps) {
    for (Particle& p : ps) behavior(p);
}

// 调用点版本 B：模板参数 —— F 的具体类型参与编译，零擦除零间接（可内联）
template <class F>
void stepAllT(F&& behavior, std::vector<Particle>& ps) {
    for (Particle& p : ps) behavior(p);
}

int main() {
    std::vector<Particle> ps(3);

    std::printf("== 1. 小闭包：std::function 不触发堆分配（塞进了对象内的小缓冲）==\n");
    g_allocs = 0;
    stepAll([](Particle& p) { p.x += 1.0f; }, ps);
    std::printf("  sizeof(std::function<void(Particle&)>)=%zu  分配次数=%zu\n",
                sizeof(std::function<void(Particle&)>), g_allocs);

    std::printf("== 2. 大闭包：超出小缓冲 → 构造 function 时堆分配 ==\n");
    g_allocs = 0;
    stepAll([state = BigState{}](Particle& p) {     // 按值捕 128 字节 → 闭包 128+ 字节
        p.x += 1.0f + (state.bytes[0] ? 0.0f : 0.0f);
    }, ps);
    std::printf("  大闭包版分配次数=%zu（类型擦除的堆分配现场）\n", g_allocs);

    std::printf("== 3. 模板参数版：同样的大闭包，零分配 ==\n");
    g_allocs = 0;
    stepAllT([state = BigState{}](Particle& p) { p.x += 1.0f; }, ps);
    std::printf("  模板参数版分配次数=%zu（F 的类型一路参与编译）\n", g_allocs);

    std::printf("== 4. 存储异构回调：这才是 std::function 的本职 ==\n");
    std::vector<std::function<void(Particle&)>> behaviors;   // 类型各异 → 只能擦除
    behaviors.push_back([](Particle& p) { p.x += 1.0f; });
    behaviors.push_back([](Particle& p) { p.y += 2.0f; });
    for (auto& b : behaviors) b(ps[0]);
    std::printf("  两个不同类型的闭包住进了同一个 vector\n");
    return 0;
}
```

实测输出：

```text
== 1. 小闭包：std::function 不触发堆分配（塞进了对象内的小缓冲）==
  sizeof(std::function<void(Particle&)>)=64  分配次数=0
== 2. 大闭包：超出小缓冲 → 构造 function 时堆分配 ==
  大闭包版分配次数=1（类型擦除的堆分配现场）
== 3. 模板参数版：同样的大闭包，零分配 ==
  模板参数版分配次数=0（F 的类型一路参与编译）
== 4. 存储异构回调：这才是 std::function 的本职 ==
  两个不同类型的闭包住进了同一个 vector
```

模板参数版为什么零代价：`F` 的具体类型**参与编译**——每个传入的闭包类型都会让 `stepAllT` 生成一份专属代码，调用点直接内联成普通指令（这就是 3.9 讲的实例化）。代价是**代码膨胀与编译期可见性**：调用者必须能看到模板定义（3.9 的头文件问题），且每个新闭包类型都多一份实例。

适用边界表（存下来，这是检验标准 ④ 的答案骨架）：

| 场景 | 用谁 | 原因 |
| --- | --- | --- |
| 存储异构回调（容器/成员变量） | `std::function` | 类型各异必须擦除，模板做不到 |
| 跨模块/动态库接口边界 | `std::function` | 边界另一侧没有模板定义可实例化 |
| 运行期注册/替换回调（事件、UI） | `std::function` | 回调类型编译期未知 |
| 泛型算法/框架的形参（可内联热路径） | 模板参数 | 零擦除、可内联；本节实测零分配 |
| 同一调用点只有一两种调用者 | 直接收具体类型/模板 | 别为泛化而泛化（先具体后泛化） |

性能只说到机制为止（间接、可能分配、不可内联）；「慢多少倍」依赖基准测试方法论——第 6 章展开。深入专题会把手伸进黑盒，亲手写一个 48 字节的 MiniFunction，看清那两跳间接长什么样。

### 3.8 词汇类型：optional、variant、any、span——以及一句 C++23 前瞻

「词汇类型」（vocabulary types）是标准库提供的「表达通用语义的通用语」：函数接口里一写出来，双方就懂意。四种语义，四个类型，按「什么样的值」对号入座：

| 语义 | 类型 | 一句话 |
| --- | --- | --- |
| 可能没有值 | `std::optional<T>` | 「可能没有」写进返回类型，哨兵值退休 |
| 有限集合选一 | `std::variant<A, B, C>` | 知道自己当前是谁的安全 union |
| 任意类型 | `std::any` | 类型安全的 `void*`，窄用途 |
| 连续数据的一段（不拥有） | `std::span<T>` | 数组传参的默认姿势（`string_view` 的泛化版） |

```cpp
// ch03_vocab_types.cpp —— optional / variant / any / span（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_vocab_types.cpp
#include <any>
#include <array>
#include <cstdint>
#include <cstdio>
#include <optional>
#include <span>
#include <string>
#include <unordered_map>
#include <variant>
#include <vector>

using TextureId = int;                           // 句柄（第 2 章的「拥有」词汇表）

// ① optional：「可能没有」写进返回类型，不再用 -1/nullptr 当哨兵
std::optional<TextureId> findTexture(const std::unordered_map<std::string, TextureId>& table,
                                     std::string_view name) {
    auto it = table.find(std::string(name));
    if (it == table.end()) return std::nullopt;  // 明确的「没有」
    return it->second;
}

// ② variant：三态状态机（每态自带数据）
struct Idle      { int  restFrames = 0; };
struct Moving    { float targetX, targetY; float speed; };
struct Attacking { int  targetId; int cooldownFrames; };

// overloaded 惯用法：让 visit 按「谁匹配谁处理」展开
template <class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts> overloaded(Ts...) -> overloaded<Ts...>;   // 推导指引

using EnemyState = std::variant<Idle, Moving, Attacking>;

// ④ span：连续数据的「指针 + 长度」二合一借用，vector/数组/array 通吃
float computeBBoxMinX(std::span<const float> xs) {
    float m = xs[0];
    for (float x : xs) if (x < m) m = x;
    return m;
}

int main() {
    std::printf("== 1. optional：查找可能失败，就用类型表达 ==\n");
    std::unordered_map<std::string, TextureId> textures{{"hero.png", 7}};
    if (auto id = findTexture(textures, "hero.png"))
        std::printf("  hero.png → id=%d\n", *id);
    auto missing = findTexture(textures, "ghost.png");
    std::printf("  ghost.png → %s（value_or 兜底：%d）\n",
                missing.has_value() ? "有" : "没有", missing.value_or(-1));

    std::printf("== 2. variant + visit：编译期穷举的状态机 ==\n");
    EnemyState state = Idle{30};
    auto describe = overloaded{
        [](const Idle& s)      { std::printf("  待机（已歇 %d 帧）\n", s.restFrames); },
        [](const Moving& s)    { std::printf("  移动向 (%.1f, %.1f) 速度 %.1f\n",
                                             s.targetX, s.targetY, s.speed); },
        [](const Attacking& s) { std::printf("  攻击目标 %d（冷却 %d 帧）\n",
                                             s.targetId, s.cooldownFrames); },
    };
    std::visit(describe, state);
    state = Moving{100.0f, 240.0f, 3.5f};        // 换状态：整个状态连同数据一起换
    std::visit(describe, state);
    std::printf("  当前是 Moving 吗？%s\n",
                std::holds_alternative<Moving>(state) ? "是" : "否");

    std::printf("== 3. any：真动态类型，窄用途 ==\n");
    std::any scriptVar = std::string("魔剑·星陨");
    std::printf("  脚本变量=%s\n", std::any_cast<const std::string&>(scriptVar).c_str());
    scriptVar = 42;                              // 换类型随便换（代价：类型擦除）
    std::printf("  现在是 int=%d\n", std::any_cast<int>(scriptVar));

    std::printf("== 4. span：连续数据传参的默认姿势 ==\n");
    std::vector<float> va{3.0f, 1.0f, 2.0f};
    float ca[4] = {5.0f, 4.0f, 7.0f, 6.0f};
    std::array<float, 3> aa = {9.0f, 8.0f, 9.5f};
    std::printf("  从 vector:  minX=%.1f\n", computeBBoxMinX(va));   // 隐式构造 span
    std::printf("  从原生数组: minX=%.1f\n", computeBBoxMinX(ca));
    std::printf("  从 std::array: minX=%.1f\n", computeBBoxMinX(aa));
    return 0;
}
```

实测输出：

```text
== 1. optional：查找可能失败，就用类型表达 ==
  hero.png → id=7
  ghost.png → 没有（value_or 兜底：-1）
== 2. variant + visit：编译期穷举的状态机 ==
  待机（已歇 30 帧）
  移动向 (100.0, 240.0) 速度 3.5
  当前是 Moving 吗？是
== 3. any：真动态类型，窄用途 ==
  脚本变量=魔剑·星陨
  现在是 int=42
== 4. span：连续数据传参的默认姿势 ==
  从 vector:  minX=1.0
  从原生数组: minX=4.0
  从 std::array: minX=8.0
```

**optional：让「没有」成为类型系统的一等公民。** 查找贴图可能失败——用 `-1` 当哨兵，调用方忘判就中招；用 `nullptr` 表达「没找到」，和「合法句柄 0」的边界含混不清。`optional<TextureId>` 把两种可能（有/没有）写进类型，`has_value()`/`operator bool` 判空、`value_or()` 提供兜底、`*opt`/`opt->` 取值（没值时解引用是 UB，和裸指针同一纪律）。判空 + 取值的惯用法是 `if (auto id = find(...))`——分支内保证有值。

**variant：知道自己是谁的 union。** 原生 union 不记录「当前是哪个成员」，读错成员是 UB；`variant<Idle, Moving, Attacking>` 随时知道自己装的是谁，读错（`std::get<Attacking>(state)` 在当前是 Moving 时）抛 `std::bad_variant_access`（异常的语义细节第 7 章展开，这里记住「它是运行期受控报错而非 UB」即可）。真正的招牌是 `std::visit` + `overloaded` 惯用法：一组 lambda 一次匹配所有备选——**漏写一个分支就是编译错误**（把示例里的 Attacking lambda 删掉，真实报错核心是 `variant(...): invoke: 未找到匹配的重载函数`——visit 无法为 Attacking 分派 handler）：

```text
编译输出节选（MSVC 19.44，已大幅截断；关键行，以你本机输出为准）：
...\include\variant(1564): error C2672: “invoke”: 未找到匹配的重载函数
...\include\variant(1564): note: 用下列模板参数:
...\include\variant(1564): note: “_Ty1=Attacking &”
```

「新增状态 → 改 variant 列表 → 编译器逐个揪出漏改的调用点」，这就是「编译期穷举」的工程价值：第 1 章的 `enum class` + 散落各处的 switch 做不到这一点（漏一个 case 编译器最多给个警告）。variant 状态机每态自带数据（`Moving` 的目标点与速度住在状态里，不用另开成员），完整的三态机是实践任务 4；「状态 + 行为 + 转移」的架构化版本叫状态模式，第 4 节展开（一句划界，不展开）。

**any：90% 想用它的时候其实要 variant。** `any` 能装任意类型的值（类型安全的 `void*`），换类型不用改声明——听起来万能，代价是类型擦除（可能堆分配、`any_cast` 类型写错抛异常、没有任何编译期检查）。正经场景只有「真·动态类型」：脚本引擎的变量槽、编辑器的属性面板。备选集合在编译期已知时，variant 永远是更好的答案。口诀（业界通行）：**每次想用 union，用 variant；每次想用 void*，用 any；每次想返回 nullptr 表错误，用 optional。**

**span：连续数据传参的默认姿势（兑付第 2 章承诺⑥的另一半）。** 之前函数收一段顶点数据只有两种姿势：`const std::vector<float>&`（逼调用方必须是 vector——数组党被迫先拷一份构造 vector）或 `const float* + size_t`（指针和长度分家，忘了同步就出事）。`std::span<const float>` 一个类型吃下 vector、原生数组、`std::array` 三种来源（示例实测），自带长度、不拥有、零拷贝——它就是 `string_view` 的泛化版，借用词汇表的第 5 位成员（借用不延长生存期的纪律原样适用：span 指向临时 vector 过夜 = 悬垂）。要「可写视图」用 `std::span<float>`。引擎停留在 C++17？`span` 是 C++20 的，平替：`gsl::span`（Guidelines Support Library）或老实的指针+长度对。

> **C++23 前瞻注记（基线外，各留一句）**：C++23 有 `std::expected`（optional 的「带错误原因」版，错误处理策略的裁决在第 7 章）、ranges（标准算法的管道化演进，如 `views::filter | views::transform`）、coroutines（协程，异步/并发的入门券，第 5 章之后再回头）。它们存在，本章不展开——硬塞新特性换报错灾难，见本章误区 1。

### 3.9 模板实例化模型：为何模板定义要放头文件（从 std::move 的真实签名说起）

到这里，上半场「用标准库」收官，下半场换挡「写模板」。但造轮子的第一条纪律恰恰是克制——先立世界观，再学语法。第 1 章 1.5 说 `std::move` 只是个「举牌员」，它的真实签名是模板——当时欠下的解释现在兑付。它的真身（节选自标准库声明，去掉了注释与命名细节）：

```cpp
template <class T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept;
```

手写一个行为等价的版本，配合第 1 章的 Image 类当场验证：

```cpp
// ch03_instantiation.cpp —— std::move 的真身与模板实例化（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_instantiation.cpp
#include <cstdio>
#include <type_traits>
#include <utility>

// 第 1 章 1.5 的「举牌员」，完整签名版（std::move 长得几乎一模一样）：
template <class T>
constexpr std::remove_reference_t<T>&& my_move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}

struct Image {                     // 第 1 章的 Image 简化版：拷贝/移动打日志
    explicit Image(int w) : width(w) { std::printf("  [构造] Image(%d)\n", width); }
    Image(const Image& o) : width(o.width) { std::printf("  [拷贝] Image(%d)\n", width); }
    Image(Image&& o) noexcept : width(o.width) {
        o.width = 0;
        std::printf("  [移动] Image(%d)\n", width);
    }
    int width;
};

template <class T>                 // 函数模板：一套逻辑，多个类型各得一份代码
T lerp(const T& a, const T& b, float t) { return a + (b - a) * t; }

struct Vec3 {
    float x = 0.0f, y = 0.0f, z = 0.0f;
    Vec3 operator*(float s) const { return {x * s, y * s, z * s}; }
    Vec3 operator+(const Vec3& o) const { return {x + o.x, y + o.y, z + o.z}; }
    Vec3 operator-(const Vec3& o) const { return {x - o.x, y - o.y, z - o.z}; }
};

int main() {
    std::printf("== 1. my_move 与 std::move 行为等价 ==\n");
    Image title(512);
    Image movedTo(std::move(title));     // 标准版：触发移动
    Image movedToo(my_move(movedTo));    // 手写版：同样触发移动
    std::printf("  原对象 width=%d（被掏空）\n", movedTo.width);

    std::printf("== 2. 一次模板，两份代码：lerp<float> 与 lerp<Vec3> ==\n");
    float f = lerp(0.0f, 10.0f, 0.5f);
    Vec3  a{0, 0, 0}, b{10, 20, 30};
    Vec3  v = lerp(a, b, 0.5f);
    std::printf("  lerp<float> =%.1f  地址=%p\n", f, (void*)&lerp<float>);
    std::printf("  lerp<Vec3>  =(%.1f, %.1f, %.1f) 地址=%p\n",
                v.x, v.y, v.z, (void*)&lerp<Vec3>);
    std::printf("  （两个地址不同 → 编译器生成了两份独立函数）\n");
    return 0;
}
```

实测输出（地址每次运行不同）：

```text
== 1. my_move 与 std::move 行为等价 ==
  [构造] Image(512)
  [移动] Image(512)
  [移动] Image(512)
  原对象 width=0（被掏空）
== 2. 一次模板，两份代码：lerp<float> 与 lerp<Vec3> ==
  lerp<float> =5.0  地址=00007FF7B57B11A0
  lerp<Vec3>  =(5.0, 10.0, 15.0) 地址=00007FF7B57B11E0
  （两个地址不同 → 编译器生成了两份独立函数）
```

拆解 `my_move` 的两个零件：`T&&` 在**推导语境**下是个特殊形状（万能引用），既能接左值也能接右值（完整机制是引用折叠，超出本章基线，见延伸资源 EMC++ 条 26–28；这里只需要知道「`std::move(左值)` 时 T 被推导成 `左值类型&`」）；`remove_reference_t` 把 `Image&` 上的引用剥掉，保证返回的永远是「右值引用」——举牌员的「举牌」动作就是这一次类型变换，函数体里没有任何运行期动作（`noexcept`、零开销）。

现在立下半场的世界观：**模板 = 编译期代码生成器。** 编译器看到模板定义时只做语法检查（「配方写得通吗」）；看到**使用点**（如 `lerp<float>`）才把具体类型代入、生成真实代码（「照配方炒一盘」）——这一步叫**实例化**。`lerp<float>` 和 `lerp<Vec3>` 是两份不同的机器码（示例打印了两个函数地址作为铁证）；同理，`std::move<Image>` 是编译器为你生成的一个真实函数，仅此而已。

这个模型直接解释了那个经典规则：**为什么模板定义要放头文件。** 链接器要的「`lerp<int>` 那份代码」必须在**使用点所在的翻译单元**里被实例化出来；而编译器处理某个 `.cpp` 时只能看见它 `#include` 的东西。把模板定义藏进另一个 `.cpp`，别的文件就只剩一份「配方名」——编译能过，链接时找不到 `lerp<int>` 那份实体。完整案发现场：

```cpp
// tpl_lib.cpp —— 有人把模板定义写进了 .cpp
template <class T>
T lerp(T a, T b, T t) { return a + (b - a) * t; }
static double g_force = lerp(1.0, 2.0, 0.5);   // 本文件实例化了一份 double 版（没人要它）

// tpl_main.cpp —— 调用方只看到声明
template <class T> T lerp(T a, T b, T t);
int main() { return lerp(1, 2, 3) - 2; }
```

```text
编译与链接输出节选（MSVC 19.44，中文语言包；以你本机输出为准）：
tpl_main.obj : error LNK2019: 无法解析的外部符号 "int __cdecl lerp<int>(int,int,int)"
    (??$lerp@H@@YAHHHH@Z)，函数 main 中引用了该符号
tpl_main.exe : fatal error LNK1120: 1 个无法解析的外部命令
```

看那个符号名：`??$lerp@H@@YAHHHH@Z`——`@H` 就是 `<int>`。链接器要的是「int 版成品」，而 `tpl_lib.cpp` 里只炒出了 double 版（int 版的配方在处理 main 时看不见，根本没人炒）。这与第 1 章 1.1 的链接错误地图接上了：普通函数的 LNK2019 是「声明了没定义」，模板的 LNK2019 是「**使用点看不到完整定义，实例化不出来**」。所以：模板（及大多数泛型代码）放头文件，这不是风格偏好，是实例化模型的硬约束。缓解手段也有（把实例化集中到一个 `.cpp` 里显式声明 + `extern template` 阻止别处重复实例化），属于编译时间优化，概念级知道即可。「头文件怎么组织进构建工程」第 4 章（CMake）展开。

用模板的三个入门形状速览，为后两节铺路：**函数模板**（`lerp<T>`，实参推导自动完成）；**类模板**（`std::vector<int>`，下一节系统化）；**别名模板**（`template <class T> using Vec = std::array<T, 3>;`——给模板起短名，`Vec<float>` 即 `std::array<float, 3>`）。

### 3.10 类模板与函数模板：全特化与偏特化

3.9 的 `lerp` 把「函数模板」的入门形状亮过了：实参自动推导（`lerp(0.0f, 10.0f, 0.5f)` 不用写 `<float>`）。本节把它补系统，再讲泛化的第二重武器——特化：**同一个模板，给某个类型开小灶**。

先把类模板的语法补齐。类模板与函数模板最大的差别：**实参推导通常不发生**（C++17 CTAD 少数场景可推导，如 `std::pair p(1, 2.0)`，概念级知道即可），尖括号一般要自己写（`std::vector<int>`）。成员函数在类内定义与普通函数无异；在类外定义要带模板头：

```cpp
// 节选：类模板成员的类外定义语法
template <class T>
class Pool {
public:
    void reset();
private:
    std::size_t live_ = 0;
};
template <class T>          // 每个成员前都要再写一遍模板头
void Pool<T>::reset() { live_ = 0; }
```

然后是特化。用「资产类型注册表」当示例：不同资产类型需要不同的元信息——纹理要上 GPU、着色器要编译、指针只是个「看看而已」的非拥有视图。主模板给默认策略，全特化给指定类型定制，偏特化给一**类**类型（如所有指针）定制：

```cpp
// ch03_template_specialization.cpp —— 全特化与偏特化（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_template_specialization.cpp
#include <cstdio>

struct Texture { int width = 0, height = 0; };
struct Shader { int stages = 0; };

// 主模板：默认策略（任何没被特化的类型都落到这里）
template <class T>
struct AssetKind {
    static constexpr const char* name = "generic";
    static constexpr bool        needsGpuUpload = false;
};

// 全特化：为 Texture 定制——「成品」，不再是模板
template <>
struct AssetKind<Texture> {
    static constexpr const char* name = "texture";
    static constexpr bool        needsGpuUpload = true;
};

// 全特化：为 Shader 定制
template <>
struct AssetKind<Shader> {
    static constexpr const char* name = "shader";
    static constexpr bool        needsGpuUpload = true;
};

// 偏特化：为「所有指针类型」定制——仍是模板（T 还没定），「半成品」
template <class T>
struct AssetKind<T*> {
    static constexpr const char* name = "pointer (non-owning view)";
    static constexpr bool        needsGpuUpload = false;
};

int main() {
    std::printf("== 1. 同一个模板，不同类型不同待遇 ==\n");
    std::printf("  AssetKind<int>     -> %s\n", AssetKind<int>::name);      // 主模板
    std::printf("  AssetKind<Texture> -> %s\n", AssetKind<Texture>::name);  // 全特化
    std::printf("  AssetKind<Shader>  -> %s\n", AssetKind<Shader>::name);   // 全特化
    std::printf("  AssetKind<int*>     -> %s\n", AssetKind<int*>::name);     // 偏特化

    std::printf("== 2. 经典易错点：Texture* 命中的是偏特化，不是 Texture 全特化 ==\n");
    std::printf("  AssetKind<Texture*> -> %s\n", AssetKind<Texture*>::name);

    std::printf("== 3. 编译期断言替你盯住规则 ==\n");
    static_assert(AssetKind<Texture>::needsGpuUpload);
    static_assert(!AssetKind<int>::needsGpuUpload);
    static_assert(!AssetKind<Texture*>::needsGpuUpload);   // 指针版不上传 GPU
    std::printf("  三条 static_assert 全部通过（编译期就验证完）\n");
    return 0;
}
```

实测输出：

```text
== 1. 同一个模板，不同类型不同待遇 ==
  AssetKind<int>     -> generic
  AssetKind<Texture> -> texture
  AssetKind<Shader>  -> shader
  AssetKind<int*>     -> pointer (non-owning view)
== 2. 经典易错点：Texture* 命中的是偏特化，不是 Texture 全特化 ==
  AssetKind<Texture*> -> pointer (non-owning view)
== 3. 编译期断言替你盯住规则 ==
  三条 static_assert 全部通过（编译期就验证完）
```

三条规则收束：① **函数模板只有全特化**（`template <> ... f<int>(...)`），没有偏特化——需要「按类型族分派」时，做法是用重载 + 约束（下一节）；顺带一句警告：函数模板的全特化**不参与重载决议**，和重载混用时容易出「你以为选了特化、其实选了主模板再隐式转换」的坑，点到为止。② **类模板两者都有**，匹配规则：全特化 > 偏特化 > 主模板；`AssetKind<Texture*>` 命中偏特化而非 Texture 全特化（尖括号里写的是 `Texture*`，与全特化的 `Texture` 不是同一个类型）——示例第 2 段当堂演示了这个经典易错点。③ 对比口诀：**全特化是成品（T 已定，不是模板），偏特化是半成品（T 未定，仍是模板，仍需实例化）**。标准库自己的 `std::less`、`std::hash` 就是这么给你留「给自己的类型开小灶」的口的。

### 3.11 concepts：带约束的模板，与它取代的 SFINAE

无约束模板有个工程代价：**报错发生在离案发现场很远的地方**。给只接受数字的 `almostEqual` 喳一个 `std::string`，模板体内的 `a - b` 才爆，报错指向模板内部某一行，而非你的调用行——项目稍大、模板嵌套几层，就是新手闻风丧胆的百行报错。**概念先行**是解药：把「对 T 的要求」写成具名的、可组合的、可复用的约束，让它成为模板接口的一部分——违约时报错直接指向你的调用行和那个具名概念。

```cpp
// ch03_concepts.cpp —— concepts 约束与 SFINAE 对照（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_concepts.cpp
#include <concepts>
#include <cstdio>
#include <memory>
#include <string>
#include <type_traits>

// ① 定义 concept：具名的编译期谓词（可组合）
template <class T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// requires 表达式：探测「这个类型有没有我要的接口」
template <class L, class K, class V>
concept ResourceLoader = requires(const L& loader, const K& key) {
    { loader.load(key) } -> std::same_as<std::unique_ptr<V>>;   // load(key) 且返回值为此类型
};

// ② 两种挂法：concepts 简写形参 vs requires 子句
template <Numeric T>
bool almostEqual(T a, T b, T eps) {
    T diff = a > b ? a - b : b - a;
    return diff <= eps;
}

template <class T>
    requires Numeric<T>                       // requires 子句版：约束更复杂的算式用这种
bool almostEqualRef(T a, T b, T eps) { return almostEqual(a, b, eps); }

// ③ 约束参与重载决议：更特化的约束自动胜出（不用 tag dispatch）
template <Numeric T>
const char* pick(T) { return "通用 Numeric 版"; }
template <std::integral T>
const char* pick(T) { return "更特化的 integral 版"; }

// SFINAE 老写法：enable_if 做同样的事（C++17 遗产，读得懂即可）
template <class T, std::enable_if_t<std::is_arithmetic_v<T>, int> = 0>
bool almostEqualSfinae(T a, T b, T eps) { return almostEqual(a, b, eps); }

struct MeshLoader {                           // 满足 ResourceLoader 的 Loader
    std::unique_ptr<int> load(const std::string&) const { return nullptr; }
};
struct BrokenLoader { };                      // 没有 load：违约现场

int main() {
    std::printf("== 1. concepts 版：约束写在签名里 ==\n");
    std::printf("  almostEqual(1.0, 1.05, 0.1) = %s\n",
                almostEqual(1.0, 1.05, 0.1) ? "true" : "false");
    std::printf("  almostEqualRef(3, 4, 2)     = %s\n",
                almostEqualRef(3, 4, 2) ? "true" : "false");

    std::printf("== 2. 违约即编译错误，报错直接指向约束 ==\n");
    // almostEqual(std::string{"a"}, std::string{"b"}, std::string{"c"});
    // ⚠ 不执行：std::string 不满足 Numeric —— 编译输出见正文报错节选

    std::printf("== 3. subsumption：更特化的约束在重载决议中胜出 ==\n");
    std::printf("  pick(3.14) -> %s\n", pick(3.14));
    std::printf("  pick(42)   -> %s\n", pick(42));

    std::printf("== 4. requires 表达式探测接口：Loader 合规检查 ==\n");
    constexpr bool ok     = ResourceLoader<MeshLoader, std::string, int>;
    constexpr bool broken = ResourceLoader<BrokenLoader, std::string, int>;
    std::printf("  MeshLoader 满足契约=%s  BrokenLoader 满足契约=%s\n",
                ok ? "true" : "false", broken ? "true" : "false");

    std::printf("== 5. SFINAE 版也能用（老代码里你会遇到它）==\n");
    std::printf("  almostEqualSfinae(2.0, 2.1, 0.2) = %s\n",
                almostEqualSfinae(2.0, 2.1, 0.2) ? "true" : "false");
    return 0;
}
```

实测输出：

```text
== 1. concepts 版：约束写在签名里 ==
  almostEqual(1.0, 1.05, 0.1) = true
  almostEqualRef(3, 4, 2)     = true
== 2. 违约即编译错误，报错直接指向约束 ==
== 3. subsumption：更特化的约束在重载决议中胜出 ==
  pick(3.14) -> 通用 Numeric 版
  pick(42)   -> 更特化的 integral 版
== 4. requires 表达式探测接口：Loader 合规检查 ==
  MeshLoader 满足契约=true  BrokenLoader 满足契约=false
== 5. SFINAE 版也能用（老代码里你会遇到它）==
  almostEqualSfinae(2.0, 2.1, 0.2) = true
```

四个零件逐个说：**① 定义**——concept 是编译期谓词，`||`/`&&` 组合，标准库已备好一大批常用概念（`std::integral`、`std::floating_point`、`std::same_as<A,B>`、`std::convertible_to`、`std::equality_comparable`、`std::invocable`、`std::movable`……cppreference「Named requirements / C++20 concepts」页是速查卡）。**② 挂法两种**——`template <Numeric T>` 简写（约束一目了然）与 `requires` 子句（约束长、或要写在已有模板声明上时用）；`requires` **表达式**则是接口探测器：`{ loader.load(key) } -> std::same_as<std::unique_ptr<V>>` 一行写出「有 load、能吃 key、返回值恰好是这个类型」——实践任务 1 的 Loader 契约就是它。**③ 约束参与重载决议**——`pick(42)` 自动选更特化的 `std::integral` 版（integral 包含于 numeric 之内，这叫 subsumption 包含关系）；C++20 之前要靠「标签分发」这种黑活才能做到的事，现在是声明式的。**④ 编译期能当布尔用**——`ResourceLoader<...>` 直接当 `constexpr bool`，把接口契约变成可 `static_assert` 的事实。

现在把同一件事的 SFINAE 版拉出来对照。SFINAE（Substitution Failure Is Not An Error，「代入失败不是错，只是这个候选出局」）是 C++98/11 时代的老手艺：利用「代入失败会被静默丢弃」这个规则，用 `std::enable_if` 造一个「数字类型才存在」的重载。看它俩的违约报错对比（本节高光素材，均为真实编译输出节选）：

```text
—— concepts 版违约：almostEqual(std::string{"a"}, ...) ——
编译输出节选（MSVC 19.44，中文语言包；已截断，以你本机输出为准）：
broken/t_concepts_err.cpp(5): error C2672: “almostEqual”: 未找到匹配的重载函数
broken/t_concepts_err.cpp(4): note: 可能是“bool almostEqual(T,T,T)”
broken/t_concepts_err.cpp(5): note: 未满足关联约束
broken/t_concepts_err.cpp(4): note: 计算结果为 false 的概念“Numeric<std::string>”
broken/t_concepts_err.cpp(3): note: 计算结果为 false 的概念“std::integral<std::string>”

—— SFINAE 版违约：almostEqualSfinae(std::string{"a"}, ...) ——
编译输出节选（MSVC 19.44，中文语言包；已截断，以你本机输出为准）：
broken/t_sfinae_err.cpp(5): error C2672: “almostEqual”: 未找到匹配的重载函数
broken/t_sfinae_err.cpp(4): note: 可能是“bool almostEqual(T,T,T)”
broken/t_sfinae_err.cpp(5): note: “bool almostEqual(T,T,T)”: 无法推导“__formal”的 模板 参数
broken/t_sfinae_err.cpp(3): note: “std::enable_if_t<false,int>”: 未能使别名模板专用化
```

两份报错都短（GCC/Clang 的 SFINAE 报错更冗长），但信息密度天壤之别：concepts 版明确说「概念 `Numeric<std::string>` 结果为 false」——**约束名 + 违约类型都在场**；SFINAE 版说的是「无法推导 `__formal` 的模板参数」「enable_if_t<false> 未能专用化」——你得自己脑补出「哦，是因为类型不满足那个藏在第 2 个模板参数里的把戏」。concepts 的两条可读性收益就此齐活：**声明处一眼可见约束**（不用读函数体、不用解谜 enable_if），**违约时报错短且指向模板本身**。C++17 平替结论（R5 标注）：引擎停在 C++17 就写 `enable_if`，这是当年的正解——但写法按「读得懂」的标杆要求自己即可。

纪律收束（与误区 1 口径一致）：**写不写模板，先具体后泛化；决定写模板，约束先行**——第一件事是把对 T 的要求写成 concept，而不是等实例化时报错。

### 3.12 constexpr 与 consteval：把计算搬到编译期

第 1 章 1.8 埋的账在此结清（承诺②）。三个关键字按层级分工：`constexpr` 变量（编译期常量）、`constexpr` 函数（**可以**在编译期执行的函数）、`consteval`（C++20，**必须**在编译期执行的函数）。先看本章的主场景——游戏与实时渲染的经典手法：**编译期生成正弦查找表**。8-bit 时代没有浮点单元，全靠查表；今天启动时零成本建表仍是热路径优化与嵌入式部署的常用招。

```cpp
// ch03_constexpr.cpp —— constexpr/consteval：把计算搬到编译期（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_constexpr.cpp
#include <array>
#include <cstdio>

constexpr double kPi = 3.14159265358979323846;

// 编译期可用的 sin 近似：归一化到 [-π, π] 后用 9 阶泰勒级数
constexpr double sinApprox(double x) {
    while (x >  kPi) x -= 2.0 * kPi;
    while (x < -kPi) x += 2.0 * kPi;
    double term = x, sum = x;                    // x - x^3/3! + x^5/5! - ...
    for (int n = 1; n <= 9; ++n) {
        term *= -(x * x) / ((2 * n) * (2 * n + 1));
        sum += term;
    }
    return sum;
}

// 256 项正弦查找表：程序启动前就已经在只读段里躺好了
inline constexpr std::array<int, 256> kSinTable = [] {
    std::array<int, 256> t{};
    for (int i = 0; i < 256; ++i)
        t[i] = static_cast<int>(sinApprox(2.0 * kPi * i / 256.0) * 1000.0);
    return t;
}();

constexpr int square(int x) { return x * x; }    // 同一函数，两个世界都能用

consteval int onlyCompileTime(int x) {           // consteval：强制编译期（立即函数）
    return x * 10;
}

int main() {
    std::printf("== 1. 编译期正弦查找表：main 还没开始，表已生成 ==\n");
    std::printf("  kSinTable[0]=%d  kSinTable[64]=%d  kSinTable[128]=%d\n",
                kSinTable[0], kSinTable[64], kSinTable[128]);
    static_assert(kSinTable[0] == 0);                       // 编译期校验
    static_assert(kSinTable[64] == 1000);                   // sin(π/2) = 1 → 1000
    static_assert(kSinTable[192] == -1000);
    std::printf("  三条 static_assert 编译期通过（零运行期成本）\n");

    std::printf("== 2. constexpr 函数是「许可」不是「指令」==\n");
    constexpr int s1 = square(7);                // 常量语境 → 编译期算
    int runtime_x = 9;
    int s2 = square(runtime_x);                  // 运行期语境 → 运行期算
    std::printf("  constexpr 语境 square(7)=%d（已烙进指令）\n", s1);
    std::printf("  运行期语境 square(9)=%d（普通函数调用）\n", s2);

    std::printf("== 3. consteval：运行期调用直接编译错误 ==\n");
    constexpr int a = onlyCompileTime(5);        // 编译期：OK
    std::printf("  onlyCompileTime(5)=%d\n", a);
    // int rt = 5;
    // int b = onlyCompileTime(rt);
    // ⚠ 不执行：运行期实参味 consteval —— 编译输出见正文报错节选
    return 0;
}
```

实测输出：

```text
== 1. 编译期正弦查找表：main 还没开始，表已生成 ==
  kSinTable[0]=0  kSinTable[64]=1000  kSinTable[128]=0
  三条 static_assert 编译期通过（零运行期成本）
== 2. constexpr 函数是「许可」不是「指令」==
  constexpr 语境 square(7)=49（已烙进指令）
  运行期语境 square(9)=81（普通函数调用）
== 3. consteval：运行期调用直接编译错误 ==
  onlyCompileTime(5)=50
```

三件事必须分清：

**① constexpr 函数是「许可」不是「指令」。** `square(7)` 出现在常量语境（初始化 `constexpr` 变量、数组大小、`static_assert`、模板实参）→ 编译期执行；`square(runtime_x)` 出现在运行期语境 → 普通函数调用。「没有法子命令编译器『能编译期就编译期』」，只能靠调用语境逐个钉死（这是 constexpr 最常见的误解，也是本章误区 8 的病根）。

**② 头文件里的 constexpr 表为什么带 `inline`。** `constexpr` 变量默认内部链接——多个 `.cpp` 各自有一份副本，地址各不相同（违反第 1 章 ODR 的「仅一次定义」精神）；`inline` 让全工程共享一份。函数同理（constexpr 函数默认按 inline 处理）。头文件里放编译期数据（LUT、常量表）的标准姿势就是 `inline constexpr`——3.9 的「模板定义放头文件」与 ODR 在此会师：**头文件里的实体，要么模板（按需实例化）、要么 inline（共享一份）、要么 constexpr（编译期消解）**。

**③ consteval（C++20）：把「必须编译期」写进签名。** 运行期实参直接编译错误（真实报错：

```text
编译输出节选（MSVC 19.44，中文语言包；以你本机输出为准）：
broken/t_consteval_err.cpp(2): error C7595: “onlyCompileTime”: 对即时函数的调用不是常量表达式
broken/t_consteval_err.cpp(2): note: 因读取超过生命周期的变量而失败
```

）。什么时候用：格式化、表生成、配置校验这类「算错就是写错」的逻辑——立即函数让错误在编译期爆炸。顺带一句 `constinit`（C++20）：保证静态/全局变量**被常量表达式初始化**（治第 2 章 2.10 静态初始化顺序病的另一味药），概念级知道即可。

最后防混淆：`constexpr` 管「**什么时候算**」，下一节的 `if constexpr` 管「**编译期选哪段代码**」——名字像，职责不同。另外 `std::array` 全线支持 constexpr（比 C 风格数组好放进头文件工具箱），本节示例已身兼示范。

### 3.13 type traits、变参模板与折叠表达式——以及 if constexpr 编译期多态

本节把三块积木拼成一个闭环：**traits 查类型**、**变参模板吃任意参数**、**`if constexpr` 按类型分流**——拼完后你会得到「编译期多态」的完整形态，兑付第 1 章 1.9 的账（承诺④：那里只展示了「按指针宽度选代码」的前菜）。

```cpp
// ch03_traits_variadic.cpp —— type traits、变参模板与 if constexpr 编译期多态（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_traits_variadic.cpp
#include <cstdio>
#include <cstring>
#include <string>
#include <type_traits>
#include <vector>

// ① traits：编译期的「类型查询」——::value 老写法 与 _v 新写法
static_assert(std::is_integral_v<int> && !std::is_integral_v<float>);
static_assert(std::is_same_v<std::remove_reference_t<int&>, int>);
static_assert(std::is_pointer_v<int*> && !std::is_pointer_v<int>);
static_assert(std::is_same_v<std::common_type_t<int, double>, double>);

template <class> inline constexpr bool dependent_false_v = false;   // 永假的「依赖型」假

// ② 变参模板 + 折叠表达式：游戏日志的雏形
void printOne(int v)                    { std::printf("%d", v); }
void printOne(const char* v)            { std::printf("%s", v); }
void printOne(double v)                 { std::printf("%.1f", v); }
template <class... Args>
void logInfo(Args&&... args) {
    ((printOne(args)), ...);            // 逗号折叠：依次展开每个包元素
    std::printf("\n");
}

// ③ if constexpr 编译期多态：按类型「能力」分流，落选分支根本不实例化
struct Player {                          // 有 serialize 成员：走自己的序列化
    std::string name = "hero";
    void serialize() const { std::printf("  Player::serialize（成员函数版）\n"); }
};
struct Particle {                        // 平凡可拷贝：走内存快照
    float x = 0.0f, y = 0.0f;
};
struct LayeredNoise {                    // 两者都不满足：编译期直接拒绝
    std::vector<float> octaves{1.0f};
};

template <class T>
void save(const T& v) {
    if constexpr (requires { v.serialize(); }) {
        v.serialize();                                        // 有成员序列化就用它
    } else if constexpr (std::is_trivially_copyable_v<T>) {
        std::printf("  [内存快照] %zu 字节原样写入\n", sizeof(T));
        unsigned char raw[sizeof(T)];
        std::memcpy(raw, &v, sizeof(T));                      // 快照可用：T 平凡可拷贝
        (void)raw;
    } else {
        static_assert(dependent_false_v<T>, "save() 需要类型提供 serialize() 或平凡可拷贝");
    }
}

int main() {
    std::printf("== 1. 变参模板：一个 logInfo 吃任意参数组合 ==\n");
    logInfo("帧 ", 240, " 耗时 ", 3.5, " ms");
    logInfo("粒子存活 ", 512);

    std::printf("== 2. if constexpr：同一调用点，三种类型三份生成代码 ==\n");
    Player p;
    Particle pt{1.0f, 2.0f};
    save(p);                                 // → Player::serialize
    save(pt);                                // → 内存快照
    // save(LayeredNoise{});                 // ⚠ 不执行：编译错误，见正文报错节选
    std::printf("  （第三个分支在 T=LayeredNoise 实例化时才会触发编译错误）\n");

    std::printf("== 3. 落选分支「根本不实例化」的实证 ==\n");
    std::printf("  T=Particle 时 save 里没有 serialize 调用也编译通过——\n");
    std::printf("  那个分支被编译期裁掉了（虚函数版本可做不到这一点）\n");
    return 0;
}
```

实测输出：

```text
== 1. 变参模板：一个 logInfo 吃任意参数组合 ==
帧 240 耗时 3.5 ms
粒子存活 512
== 2. if constexpr：同一调用点，三种类型三份生成代码 ==
  Player::serialize（成员函数版）
  [内存快照] 8 字节原样写入
  （第三个分支在 T=LayeredNoise 实例化时才会触发编译错误）
== 3. 落选分支「根本不实例化」的实证 ==
  T=Particle 时 save 里没有 serialize 调用也编译通过——
  那个分支被编译期裁掉了（虚函数版本可做不到这一点）
```

**traits = 编译期类型查询。** `std::is_integral_v<T>` 问「T 是整数吗」，`std::remove_reference_t<T>` 把 `T&` 变回 `T`（3.9 的 `my_move` 里那位就是它，在此认祖归宗），`std::common_type_t` 问「俩类型运算时听谁的」。命名规律：`is_xxx` 是类模板（取值 `::value`，C++11 老写法）；`_v` 后缀是变量模板（直接得值）；`_t` 后缀是别名模板（直接得类型）。常用速查：`is_same/is_pointer/is_integral/is_arithmetic/is_trivially_copyable/is_move_constructible`、`remove_reference/remove_pointer/decay`、`common_type`——cppreference「type traits」页按需查。

**变参模板 = 编译期参数包。** `template <class... Args>` 把任意个类型收进包里；`args...` 是包展开（一变多）；`(模式, ...)` 是折叠表达式（C++17）——`((printOne(args)), ...)` 把包逐项展开成 `printOne(a1), printOne(a2), ...`。求和是 `(args + ...)`，这些「一行的递归展开」是变参日志、`std::format`、`emplace_back` 的底层机制。

**闭环：`if constexpr` + 3.11 的 requires = 编译期多态。** `save<Player>` 实例化出「调成员函数」的版本，`save<Particle>` 实例化出「memcpy 快照」的版本，`save<LayeredNoise>` 在 `static_assert` 处编译失败（真实报错：

```text
编译输出节选（MSVC 19.44，中文语言包；已截断，以你本机输出为准）：
broken/t_save_err.cpp(11): error C2338: static_assert failed: 'save() 需要类型提供 serialize() 或平凡可拷贝'
broken/t_save_err.cpp(13): note: 查看对正在编译的函数 模板 实例化“void save<LayeredNoise>(const T &)”的引用
        with
        [
            T=LayeredNoise
        ]
```

）。同一次调用点，三种类型，三份不同的生成代码。与虚函数多态对照一句：虚函数是**运行期**查虚表选函数（一个函数体服务所有类型）；`if constexpr` 是**编译期**裁剪出多份特化代码。关键差异是「落选分支**根本不实例化**」——所以 T=Particle 的实例里那段 `v.serialize()` 根本不存在，写「对该类型不合法的表达式」也不报错（第 1 章 1.9 只讲了非模板形态，现在补全：这正是它能做编译期分派的原因）。附注：最后那个 `else` 分支里写 `static_assert(false)` 在 C++20 是错的（模板未实例化时就会被判定），所以要借助依赖模板参数的 `dependent_false_v`——C++23 已放宽此限制，用 C++20 写法过手即可。CRTP 是同一思想的「类形态」，下一节。

### 3.14 CRTP：静态多态概念级

CRTP（Curiously Recurring Template Pattern，奇异递归模板模式）——名字吓人，形状只有一行：**基类模板以派生类为实参**。它把 3.13 的编译期多态搬进类体系：框架代码（update 时序、日志、注册）在基类写一遍，具体行为由派生类提供，而派发在编译期完成、零虚函数开销。引擎代码里很常见（数学库的运算符基类、ECS 的系统基类都是这个形状），概念级掌握即可：

```cpp
// ch03_crtp.cpp —— CRTP：编译期绑定的静态多态（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_crtp.cpp
#include <cstdio>

// 基类把「派生类」当模板参数：框架代码写一遍，钩子由派生类提供
template <class Derived>
struct Updateable {
    void update(float dt) {
        static_cast<Derived*>(this)->onUpdate(dt);   // 编译期就知道调谁——没有虚表
    }
};

class Player : public Updateable<Player> {
public:
    void onUpdate(float dt) { std::printf("  Player 步进 %.2f 秒（奔跑碰撞）\n", dt); }
};

class OrbitCamera : public Updateable<OrbitCamera> {
public:
    void onUpdate(float dt) { std::printf("  OrbitCamera 步进 %.2f 秒（绕轨旋转）\n", dt); }
};

// 对照组：虚函数版——同样的形状，运行期查表选函数
class IUpdatable {
public:
    virtual ~IUpdatable() = default;
    virtual void update(float dt) = 0;
};
class VPlayer : public IUpdatable {
public:
    void update(float dt) override { std::printf("  VPlayer 步进 %.2f 秒（虚表版）\n", dt); }
};

int main() {
    std::printf("== 1. CRTP：同一套框架代码，各自接线 ==\n");
    Player p;
    OrbitCamera cam;
    p.update(0.016f);              // 调用目标在编译期确定：Player::onUpdate
    cam.update(0.016f);

    std::printf("== 2. 没有虚表：对象更瘦，调用可内联 ==\n");
    std::printf("  sizeof(Player)   =%zu（CRTP 版，无 vptr）\n", sizeof(Player));
    std::printf("  sizeof(VPlayer)  =%zu（虚函数版，多一个 vptr）\n", sizeof(VPlayer));

    std::printf("== 3. 忘写钩子 = 编译错误（静态多态的「编译期检查」卖点）==\n");
    // class Broken : public Updateable<Broken> { };
    // Broken b; b.update(0.016f);
    // ⚠ 不执行：Broken 没写 onUpdate() —— 编译输出见正文报错节选
    return 0;
}
```

实测输出：

```text
== 1. CRTP：同一套框架代码，各自接线 ==
  Player 步进 0.02 秒（奔跑碰撞）
  OrbitCamera 步进 0.02 秒（绕轨旋转）
== 2. 没有虚表：对象更瘦，调用可内联 ==
  sizeof(Player)   =1（CRTP 版，无 vptr）
  sizeof(VPlayer)  =8（虚函数版，多一个 vptr）
== 3. 忘写钩子 = 编译错误（静态多态的「编译期检查」卖点）==
```

三个要点：① `static_cast<Derived*>(this)` 是唯一机关——基类知道「自己是哪个派生类」（模板实参），派发就是一次普通的函数调用，可内联、无 vptr（`sizeof` 对比：1 vs 8）。② 忘写钩子 `onUpdate` 编译失败——「实现不完整」从运行期崩溃提前到编译期（真实报错：

```text
编译输出节选（MSVC 19.44，中文语言包；已截断，以你本机输出为准）：
broken/t_crtp_err.cpp(2): error C2039: "onUpdate": 不是 "Broken" 的成员
broken/t_crtp_err.cpp(2): note: 在编译 类 模板 成员函数“void Updateable<Broken>::update(float)”时
```

）。③ 但 CRTP 没有「运行期多态」：无法把 `Player` 和 `OrbitCamera` 混装一个容器统一 update——那是虚函数/第 4 节接口的领地。两个世界的取舍：**编译期已知全部类型 → CRTP；需要运行期混装异构对象 → 虚函数**。纪律照旧：CRTP 有样板成本，别滥用——先具体后泛化。顺带一句 C++23 注记：deducing this（显式对象形参）将让部分 CRTP 场景退休（C++23，基线外，知道有这回事即可）。ECS 里系统基类的完整讨论归第 4 节；性能账第 6 章。

## 深入专题：std::function 的类型擦除——亲手解剖一次「存任意可调用物」的代价

3.7 把 `std::function` 当黑盒讲了机制与边界。本专题把盒子拆开：手写一个同型同律的 `MiniFunction`，亲眼看「类型在哪一行被擦掉」「两跳间接长什么样」「堆分配回退何时发生」。这已经是本书第二次解剖类型擦除——第 2 章参考答案第 6 题里，`shared_ptr` 的 deleter 就是被这样擦进控制块的；同一机制再次现形，下次（第 4 节的渲染接口抽象）你应该能自己认出来。

```cpp
// ch03_mini_function.cpp —— 深入专题：手写 MiniFunction，解剖类型擦除（完整可编译）
// cl /utf-8 /std:c++20 /EHsc /W4 ch03_mini_function.cpp
#include <cstdio>
#include <cstdlib>
#include <new>
#include <utility>

static std::size_t g_allocs = 0;
void* operator new(std::size_t n) { ++g_allocs; return std::malloc(n); }
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

struct Big { char pad[64] = {}; };                    // 64 字节闭包：故意超缓冲

template <class> class MiniFunction;                  // 只声明形状：R(Args...) 叫「函数类型」

template <class R, class... Args>
class MiniFunction<R(Args...)> {
    static constexpr std::size_t kBuffer = 32;        // 小缓冲（SBO）：真 std::function 的更大
    union Storage {
        char local[kBuffer];
        void* heap;                                   // 大闭包下堆
    };
    // 内存布局：local 的最后一个字节挪用为「在堆上吗」标记（演示版从简）
    static bool&  onHeap(Storage& s) noexcept { return *reinterpret_cast<bool*>(&s.local[kBuffer - 1]); }
    static bool   onHeap(const Storage& s) noexcept { return *reinterpret_cast<const bool*>(&s.local[kBuffer - 1]); }

    Storage store_{};                                 // 不透明存储：闭包的「类型」在这消失
    R     (*invoke_)(Storage&, Args...) = nullptr;    // 调用入口（函数指针）
    void  (*manage_)(Storage&, Storage*) = nullptr;   // 生命周期入口（拷贝/析构）

    template <class F>
    static F* target(Storage& s) noexcept {
        return onHeap(s) ? static_cast<F*>(s.heap)
                         : reinterpret_cast<F*>(s.local);  // ★ 擦除点：具体类型只在这一行露脸
    }
    template <class F>
    static R invokeImpl(Storage& s, Args... args) {       // ★ 每种闭包类型各生成一份
        return (*target<F>(s))(std::forward<Args>(args)...);   // ← 间接调用发生在这
    }
    template <class F>
    static void manageImpl(Storage& src, Storage* dst) {  // dst==nullptr → 析构；否则 → 深拷贝
        if (dst == nullptr) {
            if (onHeap(src)) delete static_cast<F*>(src.heap);
            else             target<F>(src)->~F();
        } else if (onHeap(src)) {
            dst->heap = new F(*static_cast<F*>(src.heap));
            onHeap(*dst) = true;
        } else {
            new (dst->local) F(*target<F>(src));          // 拷贝进新对象的本地缓冲
            onHeap(*dst) = false;
        }
    }

public:
    MiniFunction() = default;
    template <class F>                                    // 构造：闭包连同「怎么调它」一起打包
    MiniFunction(F f) {
        if constexpr (sizeof(F) <= kBuffer - 1 && alignof(F) <= alignof(Storage)) {
            new (store_.local) F(std::move(f));           // placement new：小闭包住本地
            onHeap(store_) = false;
        } else {
            store_.heap = new F(std::move(f));            // 大闭包下堆（可能的那次分配）
            onHeap(store_) = true;
        }
        invoke_ = &invokeImpl<F>;                         // ★ 到此为止，F 的类型被「忘掉」
        manage_ = &manageImpl<F>;
    }
    ~MiniFunction() { if (manage_) manage_(store_, nullptr); }

    MiniFunction(const MiniFunction& o) {
        invoke_ = o.invoke_;
        manage_ = o.manage_;
        if (o.manage_) o.manage_(const_cast<Storage&>(o.store_), &store_);
        //                     ↑ 拷贝源逻辑上只读，但管理函数签名统一用 Storage&，这里显式脱衣
    }
    MiniFunction& operator=(MiniFunction o) noexcept {    // copy-and-swap
        Storage ts = store_;  store_ = o.store_;  o.store_ = ts;
        auto* ti = invoke_;   invoke_ = o.invoke_; o.invoke_ = ti;
        auto* tm = manage_;   manage_ = o.manage_; o.manage_ = tm;
        return *this;
    }

    R operator()(Args... args) { return invoke_(store_, std::forward<Args>(args)...); }
    explicit operator bool() const noexcept { return invoke_ != nullptr; }
};

int main() {
    std::printf("== 1. 与 std::function 同型的用法 ==\n");
    int bullets = 0;
    MiniFunction<void()> fire = [&bullets] { ++bullets; };
    fire(); fire(); fire();
    std::printf("  调用三次后 bullets=%d（闭包状态经由擦除后的存储回访）\n", bullets);

    std::printf("== 2. 小闭包住本地缓冲，零分配 ==\n");
    g_allocs = 0;
    MiniFunction<void()> small = [a = 1] { (void)a; };
    std::printf("  sizeof(MiniFunction<void()>)=%zu  构造分配=%zu\n", sizeof(small), g_allocs);

    std::printf("== 3. 大闭包超缓冲 → 堆分配（与真 function 同一逻辑）==\n");
    g_allocs = 0;
    MiniFunction<void()> big = [pad = Big{}] { (void)pad; };
    std::printf("  大闭包构造分配=%zu\n", g_allocs);

    std::printf("== 4. 拷贝：管理函数指针接管深拷贝 ==\n");
    g_allocs = 0;
    MiniFunction<void()> copy = small;
    copy();
    std::printf("  小闭包拷贝分配=%zu（拷进新缓冲）后又调用一次成功\n", g_allocs);
    return 0;
}
```

实测输出：

```text
== 1. 与 std::function 同型的用法 ==
  调用三次后 bullets=3（闭包状态经由擦除后的存储回访）
== 2. 小闭包住本地缓冲，零分配 ==
  sizeof(MiniFunction<void()>)=48  构造分配=0
== 3. 大闭包超缓冲 → 堆分配（与真 function 同一逻辑）==
  大闭包构造分配=1
== 4. 拷贝：管理函数指针接管深拷贝 ==
  小闭包拷贝分配=0（拷进新缓冲）后又调用一次成功
```

**解剖报告（三颗星的位置）**：

1. **擦除点在构造函数**：`MiniFunction<F>` 的构造模板知道 `F` 的全部类型信息，它把「怎么调」（`&invokeImpl<F>`）和「怎么管理」（`&manageImpl<F>`）打包成两个普通函数指针存起来——**从这一行起，类的其余成员函数再也看不到 `F`**，只有不透明的 `Storage`。类型信息没有被消灭，它被「冻结」进了那两个函数指针指向的静态函数里。
2. **调用是两跳间接**：`fire()` → `invoke_`（函数指针跳转）→ `*target<F>(s)`（取出闭包）→ 闭包的 `operator()`。对比模板参数版：零跳、可内联（3.7 的结论在源码级得到确认）。
3. **堆分配回退在构造时**：小闭包 placement new 进 32 字节缓冲（实测 0 分配）；64 字节闭包超限，`new F(...)` 一次堆分配（实测 1 次）——真 `std::function` 的 SBO 缓冲更大（所以 3.7 里 64 字节的 `std::function` 装下小闭包无需分配），逻辑一模一样。

三家实现对照（MSVC x64 实测口径，实现细节随实现而异）：`sizeof(std::function<void()>)=64`，小缓冲可装下不超过 48 字节的闭包（实测：48 字节闭包零分配、56 字节即下堆）；libstdc++ 总 32（缓冲 16）；libc++ 总 48——数字背不住也不用背，规律只有一条：**对象内都有一个 SBO 小缓冲，超出才下堆**。

工程边界收束（3.7 边界表的决策清单版）：存储异构回调 / 跨模块接口 / 运行期注册 → 类型擦除是唯一解，用 `std::function`；热路径、编译期已知调用者、想要内联 → 模板参数。耗时基准的方法论第 6 章展开——本专题的分配计数与 `sizeof` 已经是**不依赖计时就能拿到的机制性证据**，这个取证思路比数字本身更值钱。

## 实践任务

> **关于「渲染器」**：任务中提到的「渲染器」指 P1 自写渲染器项目（第 3 节）。P1 未启动时按各任务的**独立版本**完成，验收标准完全一致；P1 启动后迁移即「可选 P1 衔接点」。

### 任务 1：泛型 ResourceManager<Key, Value, Loader>（concepts 约束版）

**交付物**：`ResourceManager` 类模板（头文件形式交付，呼应 3.9 的定义放头文件）+ 演示 `main`：同一套核心逻辑同时管理 `TextureData`（含 `std::vector<std::byte>` 像素）与 `ShaderSource`（含 `std::string` 源码）两类资源。

**设计规格**：
- 核心数据：`std::unordered_map<Key, std::uint32_t>`（键 → 槽位号）+ `std::vector<std::unique_ptr<Value>>`（槽位表）。**槽位表元素用 `unique_ptr` 是刻意的**——槽位表增长搬家时，资源对象本体地址不动（3.2 失效规则的实战应用）；
- 对外只发轻量 `struct Handle { std::uint32_t slot; };`，`get(Handle)` 返回 `Value&`；
- Loader 用 concept 约束：`requires(const L& l, const K& k) { { l.load(k) } -> std::same_as<std::unique_ptr<V>>; }`（3.11 的 `ResourceLoader` 原样搬用）；
- 同步版即可，**不做异步**（异步加载队列是第 5 章主题，别越界）。

**验收要点**：① 写一个没有 `load` 的假 Loader → 编译失败且报错指向约束处（贴报错，用 3.11 的读法解读）；② **零复制证明**：`Value` 不可拷贝（`unique_ptr` 持有天然如此）而核心逻辑编译通过，再用第 1 章的日志计数器旁证拷贝构造调用次数为 0（检验标准 ③）；③ 去重：同一 Key 二次 `get` 命中缓存（load 计数 = 1）；④ **加分项（概念点）**：说说「Handle 长期持有后槽位被复用」会出什么问题、代数计数（generation counter）为什么能发现——完整句柄方案属第 4 节资源管理，不展开。

**独立版本**：Loader 从「内存资源注册表」加载（`std::unordered_map<std::string, std::string>` 模拟磁盘文件内容），不碰任何图形 API。**可选 P1 衔接**：渲染器就绪后换成真文件/stb_image Loader，句柄接渲染器纹理表。

### 任务 2：数学库模板化——float/double 同一套测试

**交付物**：模板化 `Vec2<T>/Vec3<T>`（运算符 +、-、*（标量）、点积、length）+ 同一套模板化单元测试（性质测试：`dot(a,b) == dot(b,a)`、`length(zero) == 0`、正交向量点积为 0）。

**设计规格**：concepts 约束 `std::floating_point<T>`（C++17 平替：`enable_if`）；浮点相等比较用 `almostEqual<T>(a, b)`，容差取 `std::numeric_limits<T>::epsilon() * scale`——**epsilon 随 T 实例化**正是模板化的核心收益（float 和 double 各得各自的机器精度，出题必答点）；至少 2 条性质用 `static_assert` 编译期验证。

**验收要点**：① `float`/`double` 双实例化跑同一套测试全绿；② 能口述「模板化后测试代码减半，加第三个类型（如 `long double`）零新增逻辑」；③ 试图写 `almostEqual` 的 int 调用点 → concept 拦截编译失败（贴报错）。

**独立版本**：若第 2 节数学库尚未手写，先用下面 ~30 行的非模板 Vec3 骨架改造（模板化它即可）；已有数学库的直接整库模板化。

```cpp
// 任务 2 独立版本起点：非模板 Vec3 骨架（约 30 行）
struct Vec3 {
    float x = 0, y = 0, z = 0;
    Vec3 operator+(const Vec3& o) const { return {x + o.x, y + o.y, z + o.z}; }
    Vec3 operator-(const Vec3& o) const { return {x - o.x, y - o.y, z - o.z}; }
    Vec3 operator*(float s)       const { return {x * s, y * s, z * s}; }
    float dot(const Vec3& o)      const { return x * o.x + y * o.y + z * o.z; }
    float length()                const;
};
```

### 任务 3：用标准算法替换手写循环（≥ 3 处）

**交付物**：先看教程给的「手写循环重灾区」（下方 60 行），替换其中**至少 3 处**为标准算法；提交替换前后输出逐字节一致的对照记录。

```cpp
// ch03_rewrite_me.cpp —— 手写循环重灾区（你的改造对象）
#include <cstdio>
#include <string>
#include <vector>

struct Particle { int id; float dist; int hp; };
struct DrawItem { int particleId; float dist; };

int main() {
    std::vector<Particle> ps = {{1, 12.5f, 3}, {2, 3.0f, 0}, {3, 8.8f, 5}, {4, 20.1f, 0}};

    // 重灾区 1：手工过滤死亡粒子（下标 + erase 的错位坑，O(n²)）
    for (std::size_t i = 0; i < ps.size(); ) {
        if (ps[i].hp <= 0) ps.erase(ps.begin() + i);
        else ++i;
    }

    // 重灾区 2：手工冒泡排序（O(n²)，比较方向全靠瞪眼维护）
    std::vector<DrawItem> draws;
    for (const Particle& p : ps) draws.push_back({p.id, p.dist});
    for (std::size_t i = 0; i < draws.size(); ++i)
        for (std::size_t j = i + 1; j < draws.size(); ++j)
            if (draws[i].dist < draws[j].dist) {
                DrawItem t = draws[i]; draws[i] = draws[j]; draws[j] = t;
            }

    // 重灾区 3：手工找 id==3 的粒子
    const Particle* found = nullptr;
    for (const Particle& p : ps) if (p.id == 3) { found = &p; break; }

    // 重灾区 4：手工算总伤害
    float total = 0;
    for (const Particle& p : ps) total += p.dist * 2.0f;

    std::printf("存活=%zu 首个=%s id3距离=%.1f 总伤害=%.1f\n",
                ps.size(), draws.empty() ? "-" : "有", found ? found->dist : -1.0f, total);
    std::printf("绘制序: %d -> %d\n", draws[0].particleId, draws[1].particleId);
    return 0;
}
```

**验收要点**：① 行为等价（替换前后输出 diff 为空；重灾区 2 的比较符方向照原代码**修复后**的语义对齐——先跑原程序记录正确输出）；② 每处替换能口述：算法名、复杂度、迭代器类别要求；③ 存活过滤必须走 remove-erase 或 `std::erase_if`，并能解释「remove 只逻辑搬移」的两步语义；④ 至少一处用 lambda 作谓词/比较器（与 3.4/3.5 呼应）。

**独立版本**：本任务天然独立（教学代码即素材）。**可选 P1 衔接**：渲染器启动后把同样的替换流程跑一遍渲染器主循环，逐处记录「问题-算法-收益」三栏表。

### 任务 4：variant+visit 状态机 + std::function/模板参数双版回调

**交付物**：① 待机/移动/攻击三态状态机：`std::variant<Idle, Moving, Attacking>`（每态自带数据，如 `Moving{ Vec2 target; float speed; }`）+ `std::visit` + `overloaded` 处理转移逻辑；② 同一回调「帧更新通知」的两种调用点：`std::function<void(State&)>` 版与 `template <class F> void onUpdate(F&& f)` 版，**注释逐行标出类型擦除发生在哪**（function 版：构造时闭包被擦进不透明存储、调用经函数指针间接；模板版：F 的具体类型参与编译、零擦除零间接）。

**验收要点**：① visit 穷举：删掉一个 handler → 编译错误（贴报错，证明「编译期强制穷举」）；② 转移图覆盖三态，且新增「眩晕」态时改动局部化（只加一个 state 结构 + 一个 handler + 相应转移——为第 4 节状态模式埋的点，任务描述里一句话点明联动即可）；③ 两版回调的类型擦除注释能对上 3.7/深入专题的机制描述（检验标准 ④ 的实操版）；④ 能口述两版适用边界。

**独立版本**：天然独立（无 P1 依赖）。**可选 P1 衔接**：作为敌人 AI 雏形接入渲染器 demo 场景。

---

## 自测题

先自己作答（口头或写在纸上），再对照章末参考答案。第 1–6 题对应阶段 3 的六条检验标准，务必都能独立完成。

**第 1 题（口述，检验标准 ①）**：模板从「你写下的代码」到「机器码」经历了什么？为什么模板定义通常要放头文件？把模板函数定义放进 `.cpp` 会遇到什么错误、为什么？

**第 2 题（现场写码，检验标准 ②）**：写一个「只接受浮点类型」的 `almostEqual<T>(T a, T b, T eps)`：concepts 版与 SFINAE（`enable_if`）版各一份，说明 concepts 的可读性收益，并描述两者违约时报错的差异。

**第 3 题（改错，检验标准 ③）**：下面这段 ResourceManager 每次都出错。找出全部拷贝点并修复（提示：`unique_ptr` 槽位 + 引用返回），并回答：为什么把 `Value` 设计成不可拷贝能让编译器替你验证零拷贝？

```cpp
// 节选：有病的版本
template <class V>
class Cache {
    std::unordered_map<std::string, V> table_;
public:
    V get(const std::string& key) {           // 病灶 A
        if (table_.count(key)) return table_[key];   // 病灶 B
        V loaded = loadFromDisk(key);
        table_[key] = loaded;                 // 病灶 C
        return loaded;                        // 病灶 D
    }
    static V loadFromDisk(const std::string& key);
};
```

**第 4 题（口述+边界，检验标准 ④）**：`std::function` 相比模板参数有哪些性能代价（机制层面至少三条）？各自的最佳适用场景是什么？

**第 5 题（现场写码，检验标准 ⑤）**：用 `std::bind` 写出「对 `player` 调用 `takeDamage(int)`：伤害值留白（占位符 `_1`），并把外部击杀计数器 `killCount` 按引用绑入（`std::ref`）」；给出等价 lambda；追问：`takeDamage` 有重载时怎么办？

**第 6 题（选型，检验标准 ⑥）**：四个场景各选一个词汇类型并说明理由：① 配置项「可能没有值」；② 技能效果「无/灼烧/冰冻三选一」且各自数据不同；③ 脚本引擎的变量槽「任意类型」；④ 顶点数组传参。末问：`span` 为什么是连续数据传参的默认姿势（对比 `const std::vector<T>&` 与 `const T* + size_t`）？

**第 7 题（判断+修法）**：三段代码逐个判断失效情况并给修法：① 范围 for 中 `v.push_back(...)` 后继续用循环变量；② `unordered_map` 插入触发 rehash 后，用旧**迭代器** vs 用旧**元素指针**；③ `map::erase(it)` 后继续用 `it` vs 用 `++it` 先存的后缀副本。

**第 8 题（概念）**：`v.erase(std::remove_if(v.begin(), v.end(), pred), v.end())` 两步各做了什么？为什么 `remove_if` 不真正删除元素？删除后 `size()` 从哪一步开始变？C++20 有没有一步写法？

**第 9 题（输出预测）**：不运行程序，预测输出与 `sizeof(closure)`（MSVC x64）：

```cpp
int base = 10;
std::vector<int> buf(3, 7);
auto show = [b = std::move(buf)]() mutable { b.push_back(base); return b.size(); };
std::printf("%zu\n", show());
base = 99;
std::printf("%zu\n", show());
std::printf("%zu %d\n", buf.size(), base);
auto borrow = [&base] { return base; };
std::printf("%d %zu\n", borrow(), sizeof(borrow));
```

追问：一个函数返回 `[&]` 捕获局部变量的 lambda，会发生什么？

## 常见误区

**误区一：模板滥用与追新。** 症状：为想象中的泛化写模板（只有一个调用者就上 `template <class T>`）、硬塞 C++20 全家桶（ranges/coroutines 装门面），换来报错灾难与编译时间膨胀。纠偏：**先具体后泛化**——出现第二个真实用例再模板化；**concepts 约束先行**——决定写模板，第一件事是把对 T 的要求写成约束（3.11）；ranges/coroutines 是 C++20 存在物但超出本章基线，不追（3.8 注记框）。两条纪律口径统一：「写不写模板，先具体后泛化；决定写模板，约束先行」。

**误区二：迭代器失效无戒心。** 症状：范围 for 里增删元素、把迭代器/指向元素的指针缓存过夜（第 1 章 HUD 崩溃的高级形态）、以为「没崩就是没事」。病根：UB 不等于崩溃，失效是一张彩票，头奖总有一天轮到你。纠偏：3.2 规则表 + 三条戒律（缓存下标/ID、修改后重取、遍历与结构修改不共舞）。

**误区三：选容器只看顺不顺手。** 症状：高频中间插入用 vector；需要有序遍历却用 unordered_map 再每次 sort；把 deque 当「更高级的 vector」随手替换 vector，不知分段存储的两跳随机访问。纠偏：选型 = 访问模式 × 复杂度 × 失效画像（3.1 对比表）；默认 vector，有明确理由才换。

**误区四：以为 `remove` 真的删了。** 症状：`std::remove_if` 之后直接读 `v.size()` 惊呼「没删掉」；或忘记 erase 直接遍历逻辑尾之后的残余。纠偏：remove 系只做逻辑搬移，删除是 `erase` 的事（3.4 实测：size 不变 + 逻辑尾残余）；C++20 记住 `std::erase_if` 一步到位。

**误区五：string_view/span 悬垂。** 症状：view 绑了临时 string 还存成成员；span 指向函数局部的 vector 被返回出去；「测试时没崩」就上线。病根：「借用不延长生存期」（第 2 章 2.9）没内化——视图的全部义务就是「只活在当前调用窗口」。纠偏：3.3/3.8 的铁律：view 几乎只作形参和局部变量；要保存就拷成 string/vector。

**误区六：`std::function` 当回调默认。** 症状：热路径逐帧回调也套 `std::function`（类型擦除 + 可能堆分配 + 阻碍内联全踩）；反面：明明要存储异构回调却硬上模板参数（装不进同一容器）。纠偏：3.7 边界表，按「存储 / 边界 / 热路径」三问选型：存储异构与接口边界 → `std::function`；热路径与编译期已知 → 模板参数。

**误区七：lambda 默认捕获一把拟。** 症状：`[&]` 图省事，借的引用随作用域死亡而悬垂；`[=]` 以为捕获了成员对象，实际捕的是 `this` 裸指针（第 2 章 2.8 的坑原样复发）。纠偏：EMC++ 条 31——显式列出每个捕获；用 3.5 的捕获三问（捕什么 / 怎么捕 / 活多久）逐个过堂；跨作用域回调只允许按值捕、init capture 移入、或 `weak_from_this()` 组合拳。

**误区八：以为 constexpr 函数一定在编译期执行。** 症状：两个方向——以为写了 `constexpr` 就万事编译期（实际上运行期语境照跑运行期），或反向因为「怕编译变慢」不敢用而无视启动期收益；把 const 和 constexpr 混为一谈（const 是运行期只读，constexpr 是编译期常量）。纠偏：3.12 的「许可 vs 指令」实测 + 三层分工：constexpr 变量（编译期常量）/ constexpr 函数（两栖，看语境）/ consteval（强制编译期）。

---

## 延伸资源

本章内容多而杂，延伸资源按「想继续深挖哪块」自取（外部经典定位：章末延伸参考，非必读前置——R1）。

**系统教程（站内目录，章节编号以 learncpp.com 当前目录为准）**

- learncpp.com：容器与迭代器章节、标准算法概览、`std::string`/`std::string_view` 小节、lambda 表达式章节（含捕获与悬垂）、函数指针与 `std::function`、模板章节（函数模板/类模板/模板特化）、constexpr 系列——与本章小节一一对应，适合当对照阅读。

**经典书目（只列本章相关条目）**

- 《Effective Modern C++》（Scott Meyers）：条 31「避免使用默认捕获模式」、条 32「使用初始化捕获将对象移入闭包」——第 2 章欠的下半页现在可以读全了（兑付⑦）；条 34「优先选用 lambda 而非 std::bind」——3.6 工程立场的出处；条 26–28（万能引用 / 引用折叠 / `std::forward`）——3.9 只开了门的那个房间，选读。

**工具书（全程）**

- cppreference：容器库首页（含复杂度表）、**迭代器失效页（本章最高频）**、算法库总览、`std::function`、`std::optional`/`std::variant`/`std::any`/`std::span` 各条目、concepts 命名要求页（3.11 速查卡）、折叠表达式、`constexpr`/`consteval` 说明符页。语义疑问先查它再搜论坛（本领域纪律）。

**文章与讲座**

- Abseil Tip of the Week #108《Avoid std::bind》（abseil.io/tips/108）——3.6 立场的业界第二出处，篇幅很短，值得全文读。
- CppCon 演讲（检索关键词而非具体链接，视频平台与频道同名可寻）：Walter E. Brown《Modern Template Metaprogramming: A Compendium》（traits 与元编程的经典双讲）；Ben Deane / Jason Turner《constexpr ALL the things!》（编译期计算能走多远的天花板演示）；Klaus Iglberger 关于类型擦除（外部多态）的讲座——本专题 MiniFunction 的进阶续篇。

## 参考答案

### 自测题

**第 1 题**。三段旅程：① 你写下模板——编译器只做**语法检查**（「配方写得通吗」），不生成任何机器码；② 使用点（如 `lerp<float>`）——编译器把具体类型代入配方，**实例化**出一份真实函数（此时才做完整类型检查、生成代码）；③ 链接器把多份实例去重合并（同名同参的实例只留一份）。每个不同的 T 都得到一份独立机器码——这是「模板是编译期代码生成机制」的字面含义。头文件规则的成因：链接器要的「`lerp<int>` 那份成品」必须在**使用点所在的翻译单元**里被实例化，而编译器只能看见该 `.cpp` include 过的东西——所以使用点必须能看到完整定义，定义放头文件。把定义藏进 `.cpp`：编译能过（声明满足了语法检查），链接时报 **LNK2019 无法解析的外部符号**（符号名里带着 `lerp<int>` 的算 `??$lerp@H@@YAHHHH@Z`）——因为那个 `.cpp` 里只实例化出了它自己用到的类型，你要的类型根本没人炒过这盘菜。

**第 2 题**。concepts 版（3.11 原样可用）：

```cpp
template <class T>
concept Floating = std::floating_point<T>;

template <Floating T>
bool almostEqual(T a, T b, T eps) {
    T diff = a > b ? a - b : b - a;
    return diff <= eps;
}
```

SFINAE 版：

```cpp
template <class T, std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
bool almostEqual(T a, T b, T eps) {
    T diff = a > b ? a - b : b - a;
    return diff <= eps;
}
```

可读性收益两条：① **声明处一眼可见约束**——concepts 版签名直接说「T 是浮点」，SFINAE 版的约束藏在第二个模板参数的默认实参里（还得理解「非类型参数默认值 + enable_if_t」这个把戏）；② **违约报错短且指向模板本身**——concepts 版报「概念 `Floating<std::string>` 结果为 false」（约束名与违约类型都在场），SFINAE 版报「无法推导模板参数 / enable_if_t<false> 未能专用化」（你得自己反推原因；GCC 的版本能拉出几十行）。C++17 引擎无 concepts，`enable_if` 就是正解——读懂它即可，别再新写复杂 SFINAE。

**第 3 题**。四个病灶全是拷贝：A `get` 按值返回（每调用必拷一份 V）；B `table_[key]` 取出是拷贝；C `table_[key] = loaded` 把局部对象拷进容器；D `return loaded` 再拷一次。修复：容器存 `unique_ptr`（拷贝被物理消灭），接口返 `V&`：

```cpp
template <class V>
class Cache {
    std::unordered_map<std::string, std::unique_ptr<V>> table_;
public:
    V& get(const std::string& key) {
        auto it = table_.find(key);
        if (it == table_.end())
            it = table_.emplace(key, std::make_unique<V>(loadFromDisk(key))).first;
        return *it->second;                     // 引用返回：全程零拷贝
    }
    static V loadFromDisk(const std::string&) { return V(64); }
};

struct TextureData {                            // 刻意做成不可拷贝
    explicit TextureData(std::size_t n) : pixels(n) {}
    TextureData(const TextureData&) = delete;
    TextureData(TextureData&&) = default;
    std::vector<std::byte> pixels;
};
```

为什么「不可拷贝」能验证零拷贝：把 `TextureData` 的拷贝构造 `= delete` 后，**任何一次意外拷贝都是编译错误**——修复后的代码能编译通过，就是「核心逻辑零复制」的编译器级证明（配合同地址断言/拷贝计数日志双保险，实测：两次 `get("hero.png")` 返回同地址，检验标准 ③ 达成）。此外 `unique_ptr` 槽位还有 3.2 的复利：槽位表扩容搬家时资源对象地址不动。

**第 4 题**。机制性代价至少三条：① **构造/拷贝时可能堆分配**——闭包超出 SBO 小缓冲（MSVC 实测阈值 48 字节）就下堆；② **调用时一次间接跳转**——经内部函数指针调度，目标通常无法内联，还多一次缓存不友好的跳转；③ **类型擦除的固有税**——闭包本身可能比 lambda 大（`sizeof(std::function)=64`），拷贝经由管理函数间接进行。适用边界：**存储异构回调、跨模块/动态库边界、运行期注册替换 → `std::function`**（类型擦除是唯一解）；**泛型算法/框架形参、热路径、编译期已知调用者 → 模板参数**（单态化、可内联、零分配）。一句话：`std::function` 为「统一类型」付费，模板参数为「每一份实例化」付编译期代码体积——运行期各归其位。

**第 5 题**。参考实现（实测输出：两次受击后击杀计数 = 1）：

```cpp
using namespace std::placeholders;
auto onHit = std::bind(
    [](Player& p, int dmg, int& kills) {
        p.takeDamage(dmg);
        if (p.dead()) ++kills;
    },
    std::ref(player), _1, std::ref(killCount));
onHit(30);   // 调用者只提供伤害值 → 填进 _1
```

要点：`_1` 是「运行期第一个实参」的占位符（绑定对象/计数器是编译期定死的）；`player` 与 `killCount` 都包 `std::ref`——bind 默认按值拷贝实参，不包的话回调改的是副本（3.6 的 `5 → 99` 现场）。等价 lambda：

```cpp
auto onHit = [&player, &killCount](int dmg) {
    player.takeDamage(dmg);
    if (player.dead()) ++killCount;
};
```

重载追问：`&Player::takeDamage` 若有多个重载，名字本身有歧义——用 `static_cast<void (Player::*)(int)>(&Player::takeDamage)` 先挑出目标签名再交给 bind/invoke（3.6 第 5 段的原样手法）。

**第 6 题**。① 配置项可能没有 → `std::optional<T>`（无值是合法状态，`value_or` 兜底）；② 技能效果三选一且数据各异 → `std::variant<None, Burn, Freeze>`（备选集合编译期已知，visit 穷举强制处理每个分支；union 不可用，any 是杀鸡用牛刀）；③ 脚本变量任意类型 → `std::any`（真动态类型，备选集合编译期未知，接受类型擦除代价）；④ 顶点数组传参 → `std::span<const Vertex>`。span 是默认姿势的原因：对比 `const vector<T>&`——它逼调用方必须是 vector（原生数组/std::array 党被迫先拷一份构造 vector），而 span 三种来源通吃；对比 `const T* + size_t`——指针与长度分家，忘了同步就是悬垂/越界，span 一个对象自带长度且边界可查（`size()`/`operator[]`）。本质：span 是「指针 + 长度」的类型安全打包，借用语义与 `string_view` 同源（第 2 章 2.9 词汇表的第 5 位成员）。

**第 7 题**。① **失效且是 UB**：范围 for 等价于「迭代器从头走到尾」，循环体内 `push_back` 可能触发扩容搬家——迭代器全灭，继续用是未定义行为；修法：先收集要加的元素循环外加，或改用下标循环并接受 size 变化，或换 3.4 的「先算后改」结构。② **不同答案**：rehash 后旧**迭代器**全部失效（继续用是 UB）；旧**元素指针/引用**依然有效（rehash 重摆桶与节点链，不搬元素本体）——3.2 表格的反直觉重点行。③ `map::erase(it)` 使 `it` 失效（被删节点析构），继续用是 UB；`it = map.erase(it)`（用返回值续命）或先 `auto next = std::next(it); map.erase(it); it = next;` 都对——首推返回值写法。而 `++it` 先存后缀副本的写法（`map.erase(it++)`）对 map 可行、对 vector **错误**（erase 令其后全部失效）——容器个性要看清楚。

**第 8 题**。第一步 `std::remove_if`：遍历区间，把「不满足 pred」的元素逐个**前移覆盖**，返回新逻辑尾——size 不变、容器长度不变，逻辑尾之后是若干「残余值」。它不删元素的原因：算法只拿得到迭代器（范围两端），而删除元素必须改容器的 size——那是成员函数 `erase` 的职权，算法无权越界。`size()` 在第二步 `erase(newEnd, v.end())` 才真正变小（物理删除残余段）。C++20 一步写法：`std::erase_if(v, pred)`（自由函数，返回删除个数）。

**第 9 题**。逐行推：① init capture 把 `buf`（3 个 7）移入闭包；`show()` 追加 `base`（当时 10）→ size 4，打 `4`。② mutable 闭包是同一个对象，状态跨调用存活；`base` 改 99 后再调，追加 99 → size 5，打 `5`。③ 原 `buf` 被移空 → `0`；`base` 是 99 → 整行 `0 99`。④ `borrow` 按引用捕 `base`，读 99；闭包只存一个指针 → `sizeof` 是 8 → 整行 `99 8`。完整输出：

```text
4
5
0 99
99 8
```

（注意一个语法点：init capture `[base, b = std::move(buf)]` 不附带默认捕获——`base` 必须显式列出，编译器会拒绝隐式捕获。）追问：返回 `[&]` 捕获局部变量的 lambda，局部变量在函数返回时析构，闭包里的引用从此悬垂——调用是 UB（第 2 章 2.8 悬垂回调的同步版）。修法：按值捕获或 init capture 移入。

### 实践任务

**任务 1（泛型 ResourceManager）**。参考思路：头文件里定义 `ResourceManager`，成员就三样：`std::unordered_map<Key, std::uint32_t> lookup_`、`std::vector<std::unique_ptr<V>> slots_`、`std::vector<std::uint32_t> generations_`（加分项用）。`get_or_load(const K& key)`：查表命中返 `Handle{slot}`；未命中 `loader.load(key)` 得 `unique_ptr<V>`、压入槽位表、记 generation、建键映射。`get(Handle)` 返 `*slots_[h.slot]`。用 3.11 的 `ResourceLoader` concept 约束 Loader 模板参数。验收要点按任务描述四条核对；特别演示「槽位表 `push_back` 触发搬家后，先前拿到的 `Handle` 依旧能 `get` 出同一对象」（`unique_ptr` 槽位的价值现场）与「同一 Key 二次加载 load 计数 = 1」（去重生效）。可选 P1 衔接：Loader 换文件读取，Handle 接渲染器纹理表。

**任务 2（数学库模板化）**。参考思路：`template <std::floating_point T> struct Vec3 { T x, y, z; ... }`，运算符全部按值返回 T 版；`almostEqual` 用 3.11 版本，容差 `std::numeric_limits<T>::epsilon() * T(16)`（scale 起点可调）。性质测试写成模板函数对 `Vec3<float>`、`Vec3<double>` 各调一次；「正交点积为 0」这类纯编译期可算的性质用 `static_assert(almostEqual(dot(a,b), T(0), eps))`——注意 `almostEqual` 需要 constexpr 化（3.12 的「许可」：常量语境自动编译期）。验收：float/double 全绿；`almostEqual(int, int, int)` 编译失败贴报错；口述「第三类型零新增逻辑」。

**任务 3（替换手写循环）**。参考替换清单（原始输出实测为 `存活=2 首个=有 id3距离=8.8 总伤害=42.6` 与 `绘制序: 1 -> 3`）：① 过滤 → `std::erase_if(ps, [](const Particle& p) { return p.hp <= 0; })`（或经典 remove-erase 两步）；② 冒泡 → `std::sort(draws.begin(), draws.end(), [](const DrawItem& a, const DrawItem& b) { return a.dist > b.dist; })`（远到近 = 降序，比较符 `>`）；③ 手工查找 → `std::find_if` + id 谓词，先判 `!= end()`；④ 手工累加 → `std::accumulate` + lambda 初值 0 起步。验收：替换前后输出逐字节一致（`diff` 为空）；每处口述「算法名 / 复杂度 / 迭代器类别」——sort 要随机访问（vector 满足）；erase_if 是容器自由函数不走迭代器类别那套。

**任务 4（状态机 + 双版回调）**。参考思路：`using EnemyState = std::variant<Idle, Moving, Attacking>;`（每态带各自数据成员）+ `overloaded` 三 lambda 处理「进入/持续」；转移逻辑用 `std::visit` 返回新状态（或以 `std::get_if` 做带上下文的转移函数）。双版回调照 3.7 示例的 A/B 两函数形状，注释对上两跳间接/单态化两套措辞。验收重点第 ① 条的报错就是 3.8 那段 `variant(1564): invoke: 未找到匹配的重载函数`；第 ② 条「新增眩晕态」体验点：加 `struct Stunned { int frames; };` 进 variant 列表 → 编译器立刻报出所有没处理它的 visit 点——「编译期穷举」的价值亲测。可选 P1 衔接：接入渲染器 demo 当敌人 AI 雏形。

---

*本章完。你现在既会用标准库的轮子——选型有依据、失效有戒心、算法优先；也能造自己的轮子——从 `my_move` 到带 concepts 约束的模板，编译期的代码生成对你不再是魔法。下一章我们把这一路凑合用的编译命令（`cl /utf-8 /std:c++20 ...`、手敲头文件路径）升级成正经的 CMake 工程：target、属性传递、三方库与 CTest——第 1 节的工程化阶段，正是从这一章的头文件与这一节的 ResourceManager 开始的。*
