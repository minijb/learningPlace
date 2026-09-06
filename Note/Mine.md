## 1. 左值,右值,将亡值

1. 字符串字面量 为 lvalue
2. `std::string`   临时值 为 rvalue
3. std::move 的真相：它只是转换，不移动任何东西

**`std::move` 之后，如果没有右值重载来接，就什么也不会发生**

**`std::move` 不清空、不释放、不调用任何函数**

**移动之后对象没有「死」**

**返回局部变量时不要画蛇添足**： 程序会自动走快路

******************

### 三/五/零法则：特殊成员函数什么时候该自己写

上一节你手写了五个特殊成员函数，感觉掌控一切；但 C++ 工程的正解恰恰是「能不写就不写」。先把编译器的「默认生成规则」摆上台面——**你写（或不写）其中一个，会影响其他几个的生成**：

| 你写了……         | 默认构造 | 析构 | 拷贝构造/拷贝赋值                | 移动构造/移动赋值      |
| ---------------- | -------- | ---- | -------------------------------- | ---------------------- |
| 什么都不写       | 生成     | 生成 | 生成                             | 生成（成员均可移动时） |
| 只写了析构函数   | 生成     | 你的 | 生成（此写法已被标准标记为废弃） | **不再生成**           |
| 写了任一拷贝操作 | 生成     | 生成 | 你的                             | **不再生成**           |
| 写了任一移动操作 | 生成     | 生成 | **删除**                         | 你的                   |

另外两个隐式删除的常见触发点：类里有**引用成员**或 **const 成员**时，赋值运算符被删除（赋值无法给它们「换内容」）。**给类加了析构函数（哪怕 `= default`），移动操作就悄悄消失了**，之后所有「移动」静默退化成拷贝。

- **Rule of Three（三法则，C++98 时代）**：如果你的类需要自定义**析构函数、拷贝构造、拷贝赋值**三者之一，那它几乎肯定三个都需要——因为「需要自定义析构」意味着类直接管理资源，而编译器默认生成的拷贝是浅拷贝，会造成双重释放。
- **Rule of Five（五法则，C++11 起）**：在上面的基础上补上**移动构造、移动赋值**。而且按生成规则，既然你写了析构，移动已经不会自动生成了，要么手写、要么显式 `= default`，否则移动语义名存实亡。
- **Rule of Zero（零法则，现代默认）**：**根本不要让类直接拥有裸资源**。把资源交给那些「自己管好自己一生一世」的成员类型——`std::string`、`std::vector`，以及第 2 章将展开的智能指针。于是五个特殊成员函数一个都不用写，拷贝/移动/析构全部自动正确。

**默认取舍的判断法**，一句话：

> 问自己：「这个类是否**直接拥有**一块需要手动释放的资源？」
> ——否（成员全是 string/vector/纯数据）：**rule of zero**，一行都不写。
> ——是（类里躺着裸指针/裸句柄）：**rule of five**，五件套齐全，移动操作记得标 `noexcept`。

## swap 惯用法与自我赋值防护

swap 存在的两个问题(尤其是移动)：
1. 自我赋值
2. new 内存不住 
swap 惯用法（copy-and-swap）一步治两个病：先在任何破坏发生之前把「可能失败的部分」全部做完（拷贝进临时对象），剩下的交换全是不抛异常的指针操作：


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
```

机制拆解：参数 other 按值传入——该做的深拷贝在进入函数体之前就完成了（这步可能抛异常，但此刻还没动过 this，对象完好）；swap 只交换两个指针和一个整数，绝不失败；函数返回时，形参 other 带着我们的旧资源析构。自我赋值？无非是「拷贝自己、和自己交换」，逻辑上自动安全。1.6 节 Heightfield 的赋值运算符就是同一招的「类内 swap」变体。

简单来说就是推荐使用 swap 数值的转移 --- 存在一个**问题**，总是有以此拷贝


## const 正确性：从第一天养成的「编译器可验证承诺」

const 只是一个承诺。

**四个使用层级**
1. 成员函数 ： 承诺不修改对象状态
2. 函数参数只读：一律 const &承诺不改不拷贝。
3. 对象 ： 编辑器锁死
4. 局部变量 ： 值不在变化，尽量使用 const

几条实战纪律：

- 新写的成员函数，默认问一句「它该不该是 const」。答案几乎总是「该」——除非它真的改状态。忘标 const 的代价是：将来 const Image& 参数的函数里想调它，编译不过，回头补 const 又可能引发连锁修改（const 传染是单向的：从内到外补齐即可）。
- 读参数用 const T&，重对象（string、vector、Image）尤其如此；小对象（int、float、指针）按值传。这正是 1.3 节引用绑定表的第一行用途。
- getter 返回 const std::string& 而不是 std::string，省一次拷贝；返回对象内部地址的接口，返回类型带 const（把「别改我内部」写进签名）。
- 一个细节：按值传参时形参上的顶层 const（void f(const int x)）属于实现细节，头文件声明可以不写、.cpp 定义可以写，互不冲突——别在这种 const 上内耗。
- 极少数场景需要在 const 成员函数里改一个「逻辑上不算状态」的缓存成员，用 mutable 标注；今天知道有这个东西即可。
- 与 const 相关的编译期常量 constexpr 是另一套机制（编译期计算属于第 3 章的主题），本章不展开。

## 新特性

- auto——让编译器替你写类型。
- 范围 for——遍历的默认姿势。for (const auto& item : items) 替代下标循环
- 结构化绑定（C++17）——一次拆开一个聚合。 auto [x, y] = point; 把结构体/pair/tuple 的成员一次性绑定到具名变量；遍历 map 时 for (const auto& [name, score] : table) 直接拿键值，可读性碾压 it->first/it->second。
- if constexpr（C++17）——编译期分支。 条件在编译期求值，落选分支不参与运行。
- enum class（限定作用域枚举）——给状态一个安全的名字。 与旧枚举相比：枚举值只在枚举名的作用域内（要写 Rarity::Epic）；不隐式转换成 int（把枚举当 int 传参直接编译错误）；可指定底层类型省内存。游戏里的状态、稀有度、槽位类型都该用它。
- nullptr——空指针的唯一写法。 它有自己的类型 std::nullptr_t，能参与重载决议（老 NULL 本质是整数 0，遇到 f(int) 与 f(char*) 重载会选错）；比较、赋值语义清晰。新代码零理由再写 NULL 或 0。

## 陷阱

陷阱一：成员初始化顺序 = 声明顺序，与初始化列表的书写顺序无关。 构造函数初始化列表看起来在按你写的顺序初始化，实际上编译器永远按成员在类里声明的顺序初始化。列表顺序写错只是骗了自己（编译器会警告），真正致命的是声明顺序本身就错：

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
```

陷阱二：统一初始化 {} 更严格，但规则要看清。 C++11 起花括号初始化（统一初始化）有两大卖点和一个大坑：

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


!!! : 移动构造函数推荐使用 noexcept ， 尤其是 vector 存放数据的时候。 没有的话会默认退回到复制拷贝
构建时机：函数 static ，在用到的时候才进行初始化

```cpp
void take(const Image& img);   // ②
void take(Image&& img);        // ③
Image a("a.png", 32, 32);
take(a);                       // (1) 选哪个？ 2
take(std::move(a));            // (2) 选哪个？ 3
take(Image("b.png", 8, 8));    // (3) 选哪个？ 3
const Image c("c.png", 8, 8);                 
take(std::move(c));            // (4) 选哪个？为什么？ 2 承诺修改，但是有 const ，所以回退
```