# C++ Traits 教程（入门篇）

## 1. 什么是 Traits？

Traits（特性萃取）是一种**类型信息提取技术**：用一个模板类，根据传入类型 T 提供关于 T 的附加信息（类型、常量等）。

```cpp
template <typename T>
struct Traits {
    static constexpr const char* name = "unknown";
};
```

核心思想：**把类型的信息从类型本身中分离出来，让算法在编译期获取这些信息。**

## 2. 为什么需要 Traits？

考虑一个需求：打印不同类型的名字。

```cpp
// 不传参时无法推断，直接特化解决
template <typename T> struct TypeName;
template <> struct TypeName<int>    { static constexpr const char* value = "int"; };
template <> struct TypeName<double> { static constexpr const char* value = "double"; };
template <> struct TypeName<float>  { static constexpr const char* value = "float"; };

template <typename T>
void print() {
    std::cout << TypeName<T>::value << "\n";
}
```

**特化（specialization）是 Traits 的基础手段。**

## 3. 经典应用：类型萃取

### 3.1 萃取关联类型

```cpp
template <typename Container>
void print_front(const Container& c) {
    // 从容器类型萃取元素类型
    using T = typename Container::value_type;
    if (!c.empty())
        std::cout << static_cast<T>(c.front());
}
```

### 3.2 判断是否为指针

```cpp
template <typename T> struct is_pointer {
    static constexpr bool value = false;
};
template <typename T> struct is_pointer<T*> {
    static constexpr bool value = true;
};

// C++17 起可直接用编译期常量
if constexpr (is_pointer<T>::value) { /* ... */ }
```

### 3.3 去除 const 和引用

```cpp
template <typename T> struct remove_const          { using type = T; };
template <typename T> struct remove_const<const T> { using type = T; };

template <typename T> struct remove_ref            { using type = T; };
template <typename T> struct remove_ref<T&>        { using type = T; };
template <typename T> struct remove_ref<T&&>       { using type = T; };

// 使用
remove_const<const int>::type x = 5;   // x 是 int
```

## 4. 类型推导：type_traits 头文件

标准库 `<type_traits>` 已提供大量工具：

| 工具 | 作用 |
|---|---|
| `std::is_same<A, B>::value` | 判断两个类型是否相同 |
| `std::is_integral<T>` | 是否为整型 |
| `std::is_arithmetic<T>` | 是否为算术类型 |
| `std::decay_t<T>` | 退化类型（去 const、引用、数组转指针） |
| `std::enable_if_t<cond, T>` | 条件编译（SFINAE） |
| `std::conditional_t<cond, A, B>` | 条件选择类型 |

```cpp
template <typename T>
auto add(T a, T b) {
    if constexpr (std::is_integral_v<T>) {
        return a + b;              // 整数：直接加
    } else {
        return static_cast<T>(a + b); // 浮点：可扩展逻辑
    }
}
```

## 5. Traits 的实际价值

1. **编译期分派**：为不同类型选择最优实现（如迭代器 category）
2. **接口统一**：`std::numeric_limits<T>::max()` 就是一个典型的 traits 类
3. **零开销**：所有判断在编译期完成，无运行时成本

## 6. 小结

- Traits = **模板 + 特化**，本质是一种编译期查询
- 萃取类型信息 → `using type = ...`
- 萃取常量信息 → `static constexpr auto value = ...`
- 现代 C++ 优先用标准库 `<type_traits>` + `if constexpr` + `_v` / `_t` 后缀

---

需要我展开某个部分（比如 `enable_if` / SFINAE、iterator_traits 实例）可以继续问。
