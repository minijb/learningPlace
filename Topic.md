# 学习计划

用户角色:
角色：基础游戏开发工程师
需求：深入的学习游戏开发相关事宜

agent 角色:
担任教师，对用户进行指导, 默认用户是不了解的，可以对用户进行询问。

## 1. OpenGL（渲染 API 实操）
- 以 LearnOpenGL 路线（窗口与首个三角形 → 变换与相机 → 光照 → 模型加载）为教程骨架，同步以一个自写小渲染器贯穿实操，教程与项目双线推进。
- 掌握核心资源与状态实操：VAO/VBO、纹理与采样、FBO 帧缓冲，理解 CPU↔GPU 数据流与同步。
- 按主流路线推进进阶专题：实例化绘制、帧缓冲与后处理、延迟渲染、PBR/IBL。
- 向现代用法延伸：4.5+ 的 DSA 直接状态访问与 AZDO 低开销方向（持久映射、间接绘制），理解其与传统绑定模式的差异。
- 建立图形调试习惯：以抓帧调试器与调试输出定位渲染问题。
- 边界声明：本节聚焦 API 实操，着色器到“能读懂并使用”为止；图形数学与渲染理论归第 2 节，GLSL 深入与 Compute 归第 5 节。

## 2. 计算机图形学（数学与渲染理论）

## 3. c++ 基础+深度理解
- 以现代 C++（17/20）为语言基准，先夯实 RAII、智能指针、值语义与移动语义等资源管理根基，再进入模板/泛型等高级主题。
- 理解 C++ 内存模型与多线程并发的基本边界：数据竞争、同步原语与内存序。
- 面向游戏实时性建立性能意识：帧预算、缓存友好数据布局与热路径零分配习惯。
- 具备工具链基本功：CMake 构建、调试器与性能剖析器定位问题。
- 以 Unreal Engine 源码与 C++ Core Guidelines 为标尺，培养大型工程阅读与重构素养。
## 4. 游戏开发软件架构设计

## 5. GPU 编程

## 6. vulkan 

## 推荐参考网站(只是推荐一个搜索方向)

1. LearnOpenGL（https://learnopengl.com，中文版 https://learnopengl-cn.github.io）— 现代 OpenGL 系统教程，对应第 1 节
2. GAMES101（现代计算机图形学入门）— 图形学数学与渲染理论公开课，对应第 2 节
3. Scratchapixel — 计算机图形学理论与实现参考，对应第 2 节
4. CppCon — C++ 工程与深度讲座视频库，对应第 3 节
5. cppreference（https://en.cppreference.com/）— C++ 语言与标准库权威文档，对应第 3 节
6. Game Programming Patterns — 游戏架构模式在线免费书，对应第 4 节
7. GPU Gems（NVIDIA，全套免费在线）— GPU 编程与渲染技术实例库，对应第 5 节；成书较早（2004–2007），系统性的现代 GPU 通用编程可另见 PMPP（Programming Massively Parallel Processors，第 5 版）
8. Valve 开发者 Wiki（https://developer.valvesoftware.com/wiki/Zh/Main_Page）— 商业级引擎工程实践参考
9. GDC / GDC Vault — 行业会谈第一手来源：渲染、架构、优化实践
10.https://docs.vulkan.net.cn/tutorial/latest/03_Drawing_a_triangle/00_Setup/00_Base_code.html