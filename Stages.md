# 学习阶段路线

> 本文件是 [Topic.md](./Topic.md) 第 1–6 节的**分阶段详细路线**：每阶段含学习目标、核心知识点、实践任务、检验标准与建议时长，末尾附推荐资源与常见误区。
> Topic.md 保留精炼正文与边界声明，本文件承载执行细节；各节可独立查阅，按需跳转。
> **时长说明**：各阶段标注的"约 X–Y 周"按每周 10–20 小时投入估算，仅为相对量级参考，可按个人节奏整体缩放。
> **项目视图**：跨章节项目的里程碑对齐与「项目×章节」双向对照见 [Projects.md](./Projects.md)。

---


## 第 1 节 c++ 基础+深度理解

**阶段 1｜现代 C++ 基石：对象模型、值语义与移动语义（约 2–3 周）**
- 学习目标：建立“对象有类型、有存储、有生命周期”的底座世界观，吃透 C++ 最独特也最核心的值语义与拷贝/移动机制；搭好开发环境，从第一天起就会用调试器单步观察程序。
- 核心知识点：编译到运行粗链路——预处理/编译/链接、翻译单元与 ODR 概念级；栈与堆、自动存储期、作用域与析构次序；值类别 lvalue/rvalue 与引用重载（const T& vs T&&）；拷贝构造/赋值 vs 移动构造/赋值、std::move 只是转换不移动；三/五/零法则与 rule of zero 的默认取舍；swap 惯用法与自我赋值防护；const 正确性；现代语法速览——auto、范围 for、结构化绑定、if constexpr、enum class、nullptr；初始化陷阱（成员初始化顺序、统一初始化的 narrowing）；调试器基本功——断点/单步/监视、调用栈与崩溃转储定位。
- 实践任务：写自测资源类（Image/Buffer，管理一块堆内存）完整实现拷贝/移动/析构并在各特殊成员函数打日志；为第 2 节手写数学库逐类标注“零/三/五法则该选哪个”并补齐单元测试；在调试器中单步观察一次 vector 扩容（日志验证移动构造被调用、拷贝被消除）。
- 检验标准：能口述“std::move 不移动任何东西，真正的移动发生在移动构造函数里”；对任给类能说出该遵循 rule of zero 还是 rule of five 及理由；vector 扩容日志显示资源类只被移动未被拷贝；能用调试器独立定位一个悬垂引用类崩溃的成因。

**阶段 2｜RAII 与所有权：从裸句柄到智能指针（约 2 周）**
- 学习目标：把“资源获取即初始化”内化为唯一资源管理模式，具备为任意 C API 裸资源（GL 句柄、Vulkan 对象、文件、锁）现场设计 RAII 包装的能力——第 6 节“及时用 RAII 重构”所需能力的直接出处。
- 核心知识点：RAII 的普适性——不止内存：文件、互斥锁、GL/Vulkan 句柄皆可析构释放；unique_ptr 语义与零开销抽象；自定义 deleter 包装 C API 句柄；shared_ptr 控制块、make_shared 差异、循环引用与 weak_ptr 破环；“引用计数原子性 ≠ 对象本身线程安全”；enable_shared_from_this 与 lambda 捕获 this 的悬垂坑；所有权词汇表——拥有/借用（引用、span、string_view 借而不拥有）与生存期责任；静态/全局对象析构顺序陷阱概念级；泄漏与悬垂检测工具——AddressSanitizer、MSVC CRT 调试堆。
- 实践任务：把第 3 节渲染器的 GL 资源（Texture/Shader/Mesh/FBO）重构为 RAII 包装类（自写 GLHandle 型句柄类或 unique_ptr+自定义 deleter），消灭散落的裸 glDelete*；构造“渲染中途提前 return/抛异常”的路径，用 ASan/CRT 调试堆验证零泄漏（此工具即第 4 节资源管理阶段“用第 1 节工具验证无泄漏”的落点）。
- 检验标准：渲染器中每个 GL 对象恰有一个 RAII 属主，全代码检索不到裸 glDelete 调用；强制 early-return 路径下 ASan/CRT 调试堆零泄漏报告；能现场为 Vulkan 风格 create/destroy C API 设计 RAII 包装；能举出一个“shared_ptr 是偷懒”的反例（所有权本可唯一却共享）。

**阶段 3｜标准库与泛型编程：从用模板到写模板（约 2–3 周）**
- 学习目标：把标准库用到“选型有依据、迭代器失效有戒心”，并跨过“只会用模板”到“能写带约束的模板”的门槛，理解模板是编译期代码生成机制而非运行期魔法。
- 核心知识点：容器选型与复杂度——vector/deque/unordered_map/map 的取舍、迭代器失效规则；算法库优先于手写循环——sort/find/transform/remove-erase 惯用法；string/string_view 与 SSO；lambda 与捕获——捕获 this 悬垂、init capture；std::function 的类型擦除开销边界 vs 模板参数；模板实例化模型——为何模板定义要放头文件；类模板/函数模板、全特化与偏特化；C++20 concepts 约束写法与旧 SFINAE 对照；constexpr/consteval 编译期计算；常用 type traits；变参模板与折叠表达式初步；CRTP 静态多态概念级（引擎代码常见形状）。
- 实践任务：为渲染器写泛型 ResourceManager<Key, Value, Loader>（加载/缓存/去重/句柄化），用 concepts 约束 Loader 接口；把第 2 节数学库模板化，float/double 双精度同一套测试；用标准算法替换渲染器中至少三处手写循环。
- 检验标准：能口述模板从实例化到生成代码的过程并解释头文件依赖成因；给出同一段参数约束的 SFINAE 与 concepts 两版实现并说明可读性收益；ResourceManager 同时管理纹理与着色器两类资源而核心逻辑零复制；能说出 std::function 相比模板参数的性能代价与各自适用边界。

**阶段 4｜工具链工程化：CMake 构建体系（约 1–2 周）**
- 学习目标：摆脱 IDE 一键工程的黑盒，能用现代 CMake 组织中型项目、接入第三方库与测试，产出可交接的构建工程——第 6 节阶段 1“用 CMake 搭建项目（复用第 1 节工具链）”在此直接兑现。
- 核心知识点：CMake 核心模型——target 及属性传递（target_link_libraries/target_include_directories 的 PUBLIC/PRIVATE/INTERFACE 语义）；Debug/Release 多配置与编译选项挂接；FetchContent/find_package 接第三方库（glfw、glad、assimp）；库目标与可执行目标拆分——渲染器拆为静态库+demo 应用；CTest 接单元测试；编译警告基线（/W4 或 -Wall -Wextra 的取舍）；out-of-source 构建、生成器与 IDE/Ninja 工程；显式列举源文件 vs glob 的取舍。
- 实践任务：把第 3 节渲染器 + 第 2 节数学库 + 阶段 3 的 ResourceManager 改造为规范 CMake 工程（库目标+应用目标+测试目标，FetchContent 拉取 glfw/assimp），Debug/Release 双配置一键构建，CTest 跑通全部单元测试。
- 检验标准：新环境克隆仓库后 configure→build→test 三步内从零构建出可运行的渲染器与全绿测试；能解释任意一条链接错误的“缺符号/重复符号”来自哪个 target 传递环节；新增 Vulkan 目标只需添加一个 CMake 目标而不动既有结构——第 6 节复用此工程的衔接点。

**阶段 5｜并发与内存模型：数据竞争的边界（约 2–3 周）**
- 学习目标：理解 C++ 内存模型划定的合法边界——什么构成数据竞争、标准原语如何安全通信；目标定为“能识别与治理竞态、默认顺序一致、必要时才降级”，不追求精通 lock-free。
- 核心知识点：std::thread/jthread、参数传递与悬垂引用坑；数据竞争的定义与未定义行为本质（不是“偶尔读到旧值”）；mutex + lock_guard/scoped_lock（锁的 RAII，呼应阶段 2）；condition_variable 与虚假唤醒；一次性行动——call_once、static 局部变量初始化；atomic 基本操作；六种内存序概览——为何 seq_cst 是安全默认、acquire/release 配对是主要降级路径、relaxed 的窄用途；接口级竞争——线程安全容器并不存在；false sharing 概念（与阶段 6 缓存行呼应）；ThreadSanitizer/静态分析抓竞态。
- 实践任务：给渲染器实现“主线程渲染 + 工作线程异步加载模型/纹理”（共享任务队列+锁+条件变量），加载完成回主线程上屏；先故意写一版有数据竞争的实现，用 TSan/静态分析抓出并修复；写一份本引擎并发约定短文档（哪些数据可跨线程、谁负责同步）。
- 检验标准：能给出最小数据竞争示例并解释为何是未定义行为；能口述 acquire/release 如何替代一把重锁完成配对同步；修复前 TSan 报竞争、修复后零报告；异步加载期间帧率不塌、完成后纹理正确上屏。

**阶段 6｜实时性能意识：帧预算、缓存友好与热路径零分配（约 2–3 周）**
- 学习目标：建立“每帧都是一次 16.6ms 预算考试”的世界观：会用剖析器定位帧耗时、理解缓存层级对数据布局的支配作用、把热路径零分配炼成肌肉记忆——第 4 节 ECS“数据布局与缓存命中”的性能出处。
- 核心知识点：帧预算算术（60fps=16.6ms、144fps=6.9ms）与各环节分摊；测量方法论——采样剖析器/帧计数器/时间线三分工、微基准陷阱（Release 才有意义、防被优化器吃掉）；缓存层级——L1/L2/L3 容量与延迟量级、cache line 与预取；AoS vs SoA 及“遍历模式决定布局”（第 4 节 ECS“Component 是纯数据”的性能理由）；分支预测友好模式；堆分配的隐藏成本——分配器竞争/系统调用/碎片，逐帧 new/delete 为何大忌；零分配手法——reserve 预留、对象池/arena、swap-and-pop；string_view 传参与临时字符串规避；除法/开方等贵运算意识。
- 实践任务：用 Visual Studio 性能剖析器或 Tracy 为渲染器做帧耗时画像，产出“问题-假设-改动-数据”四栏优化记录；做 AoS vs SoA 粒子更新对照实验（10 万粒子、同算法两版布局）；把渲染器主循环改造为零分配（预分配逐帧容器，用分配器 hook/断言验证每帧堆分配次数为 0）。
- 检验标准：能解读一份火焰图并指出渲染器前三大耗时点（第 4 节阶段 2 用同一 profiler 对比三代对象模型，直接复用此能力）；SoA 版粒子更新有可复现的量化差距且能用缓存行解释；主循环零分配有运行时证据；能口述“60fps 每帧预算多少、一次堆分配贵在哪”。

**阶段 7｜大型工程阅读与重构：以 UE 源码与 Core Guidelines 为标尺（约 2–3 周）**
- 学习目标：获得在百万行级代码库中定位、理解、修改代码的方法论；以 C++ Core Guidelines 与 Unreal Engine 源码为标尺形成工程审美，能以测试护航小步重构。
- 核心知识点：UE 源码地图——模块结构（Runtime/Core/CoreUObject/Renderer 等）、TArray/TSharedPtr/FGuid 等自建基础设施与标准库的对照及自造轮子的动机；UE 命名与代码约定为一套独立纪律的原因；Core Guidelines 选读——R 资源管理、ES 错误处理、Per 性能、T 模板核心条目；错误处理策略边界——异常 vs 错误码 vs assert，游戏引擎普遍禁异常的原因；阅读方法论——从调用点向上追、以断点为地图、以日志/反汇编验证猜想；重构护栏——clang-tidy/clang-format 自动化、行为保持测试先行、小步提交。
- 实践任务：在 UE 源码（GitHub: EpicGames/UnrealEngine，注册关联后可克隆）完成一次定点阅读：沿渲染调用链 UPrimitiveComponent → FPrimitiveSceneProxy → Renderer 模块绘制路径读通，写千字导览笔记；为渲染器工程建立 clang-tidy 静态检查基线并清零告警；自选一处执行完整小步重构（抽取函数/消除全局状态），测试保证行为不变。
- 检验标准：能向他人讲清所选 UE 调用链每层职责与跨模块/跨线程边界（含 SceneProxy 存在的理由）；clang-tidy 零告警且至少拦截过一次真实缺陷（留案例记录）；重构前后单元测试全绿、渲染器输出逐帧一致；能举出 UE 三处“标准库之外自建设施”并解释动机。

**推荐资源**：清单第 5 条 cppreference 为全程工具书，任何语义疑问先查它再搜论坛；第 4 条 CppCon 按阶段选听经典——阶段 2《There Are No Zero-Cost Abstractions》（Chandler Carruth）、阶段 6《Data-Oriented Design and C++》（Mike Acton）、阶段 5 并发专题（Fedor Pikus 等）；第 14 条 Unreal Engine 官方文档与源码支撑阶段 7；第 9 条 GDC Vault 检索 CPU 性能与引擎工程优化讲座。补充：learncpp.com（免费、持续更新的系统教程，阶段 1 打底）、《Effective Modern C++》（Scott Meyers，阶段 1–3 精读）、《C++ Concurrency in Action》第 2 版（Anthony Williams，阶段 5 主教材）、Modern CMake（https://cliutils.gitlab.io/modern-cmake/，阶段 4）、C++ Core Guidelines（https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines，阶段 7 标尺）、Tracy Profiler（https://github.com/wolfpld/tracy，阶段 6）。

**常见误区**：
- 把 C++ 当“带类的 C”学：裸 new/delete 散落各处、资源手动配对释放，错过 RAII 世界观；新代码从第一行起就该 rule of zero。
- 只背语法不建模型：说不出 vector 扩容时拷贝/移动/析构各发生几次，值语义停留在名词层面。
- 万物 shared_ptr：所有权含糊化、循环引用泄漏，并把“引用计数线程安全”误当“对象线程安全”。
- 在 Debug 配置下测性能下结论：不开优化（MSVC /Od 还叠迭代器调试开销）数据全部作废；基准一律 Release，且防编译器把被测代码优化掉。
- 热路径逐帧 new 与临时字符串拼接：堆分配与碎片无声侵蚀帧预算，帧循环零分配应成纪律而非优化项。
- 用 sleep 或“多跑几遍没崩”对待竞态：数据竞争是未定义行为，正解是同步原语或重构数据所有权，并用 TSan 验证。
- 模板滥用与追新：为想象中的泛化写模板、硬塞 C++20 全家桶（ranges/coroutines），换来报错灾难与编译时间膨胀；先具体后泛化、concepts 约束先行。
- 阅读大厂源码从 main 顺读：百万行级代码库应定点追踪调用链+调试器断点验证，通读必然迷路。

## 第 2 节 计算机图形学（数学与渲染理论）

**阶段 1｜前置数学：系统学线性代数与微积分核心（约 2–3 周）**
- 学习目标：以系统学习建立线性代数与微积分的可靠根基，把向量/矩阵当几何语言而非抽象符号；明确图形学只消费其中一个明确子集，学完即进入正文。
- 核心知识点：线性代数——点乘（投影、夹角、光照贡献）与叉乘（定向、法线、三角形朝向）；矩阵=线性变换、乘法=变换复合、正交矩阵性质；齐次坐标与仿射变换（平移为何要升维）；行/列向量与左右手系约定差异。微积分——导数/梯度（高度场求法线）、定积分=连续求和（为渲染方程铺路）。概率直觉——期望、方差、大数定律（为蒙特卡洛铺路）。可先用 3Blue1Brown《线性代数的本质》建立画面感，再以系统课程/教材为主线过一遍并动笔推题。
- 实践任务：手写 C++ 数学库（Vec2/3/4、Mat4 及常用运算），每函数一句注释说明几何意义；配单元测试验证关键性质（旋转矩阵正交性 RᵀR=I、叉乘垂直性）。
- 检验标准：不查资料手推旋转/缩放矩阵，能做基础的求导/积分题；能口述“点乘为何出现在 Lambert 光照、叉乘为何出现在背面剔除”；数学库通过全部性质测试。

**阶段 2｜变换与投影：一个顶点的完整旅程（约 1–2 周）**
- 学习目标：逐环节讲清顶点从模型空间到屏幕像素的变换链，做到“画面不对能定位是哪一环出错”。
- 核心知识点：MVP 各变换职责；lookAt 视图矩阵构造；正交/透视投影矩阵完整推导；透视除法（w 分量来源）与视口变换；法线变换用逆转置 (M⁻¹)ᵀ 的原因；各 API 的 NDC/深度范围差异——第 3/6 节常见坑点的理论出处。
- 实践任务：在阶段 1 数学库上实现 lookAt/ortho/perspective/viewport；手工计算一个顶点的完整 MVP 链路并与标准结果对照。
- 检验标准：白板推导透视投影矩阵；解释“w 如何带来近大远小”；给定“看不见/拉伸/镜像”等症状能点名出错环节。

**阶段 3｜光栅化：从三角形到像素（约 2–3 周）**
- 学习目标：用 CPU 复刻 GPU 光栅化全过程，建立“屏幕=离散采样”的世界观。
- 核心知识点：三角形覆盖判断（边缘函数/叉乘同号）；重心坐标插值与透视校正插值（屏幕空间线性≠世界空间线性）；深度测试、z-fighting 与深度的非线性；采样理论与走样，SSAA/MSAA 原理（FXAA/TAA 知思想即可）；纹理采样：UV、双线性过滤、mipmap 抗走样。
- 实践任务：软件光栅器（纯 C++，输出图片文件）逐级迭代：线框 → 实心 → 深度测试 → 纹理与透视校正插值；渲染与第 3 节同款模型并对照差异。
- 检验标准：解释“远处纹理闪烁为何 mipmap 能治”；手算三角形内一点的透视校正插值；光栅器能正确渲染带纹理与遮挡的旋转模型。

**阶段 4｜着色与材质：局部光照原理（约 2 周）**
- 学习目标：从“会调参数”到“能推导”，并为 PBR/IBL 预埋概念。
- 核心知识点：Blinn-Phong 三分量逐项推导；着色频率 flat/Gouraud/Phong 差异；纹理不止颜色：法线/凹凸/位移贴图与切线空间；阴影贴图两 Pass 原理与痼疾（自遮挡、彼得潘宁）；环境贴图与球谐（SH）概念预埋。
- 实践任务：给软件光栅器加 Blinn-Phong + 法线贴图 + 阴影贴图；与第 3 节 OpenGL 光照结果对照，用理论解释每处差异。
- 检验标准：推导漫反射 cosθ 项与半程向量高光；解释“法线贴图看得见凹凸摸不到边”；手画 shadow map 流程并指出典型伪影成因。

**阶段 5｜几何表示：曲线、曲面与网格（约 1–2 周）**
- 学习目标：建立几何表示方法地图，理解游戏资产（模型/地形/曲线）背后的数学。
- 核心知识点：隐式（SDF 等）与显式（参数曲面/网格）表示的优劣与适用场景；贝塞尔曲线：Bernstein 基、de Casteljau 递推、凸包性质；B 样条/NURBS 概念级了解；贝塞尔曲面与 Catmull-Clark 细分；网格三件事：细分/简化（QEM 边坍缩思想）/正规化；LOD 与曲面细分的理论出处。
- 实践任务：实现 de Casteljau 并可视化贝塞尔曲线与控制点；（选做）对照阅读 meshoptimizer 的网格简化思路。
- 检验标准：手算一次 de Casteljau 细分；口述隐式/显式各自“容易做什么、难做什么”；说清 LOD 在游戏中解决什么问题。

**阶段 6｜光线追踪与辐射度量学（约 2–3 周）**
- 学习目标：换一条渲染路线，从“对每个像素投光线”的第一性原理出发，并装备物理度量语言。
- 核心知识点：光线-球、光线-三角形（Möller–Trumbore）求交；AABB/BVH 加速结构概念；Whitted 风格递归追踪（反射/折射/阴影光线）；辐射度量学四件套：flux/intensity/irradiance/radiance 及单位；渲染方程直观解读；radiance 沿直线守恒为何使它成为渲染核心量。
- 实践任务：CPU 光线追踪器：球+三角形场景、Whitted 递归、阴影光线；渲染经典三球场景并与光栅化结果对比。
- 检验标准：推导光线-球求交判别式；逐项解释渲染方程；说出光栅化与光线追踪的本质差异（前向 vs 反向、并行单位不同）。

**阶段 7｜路径追踪与物理渲染入门（约 2–3 周）**
- 学习目标：用蒙特卡洛解渲染方程，补齐 PBR 理论根基。
- 核心知识点：蒙特卡洛积分与无偏估计、方差与噪点来源、重要性采样入门；路径追踪算法（弹射、俄罗斯轮盘）；BRDF 定义与性质；Lambert BRDF=ρ/π 推导；微表面模型直觉（Cook-Torrance：GGX/几何项/菲涅尔）；IBL 与 split-sum 为何成立；色彩空间：线性 vs sRGB、gamma 校正、色调映射。
- 实践任务：追踪器升级为路径追踪（漫反射+镜面+发光体 → Cornell Box）；加 GGX 金属粗糙材质，渲染金属度/粗糙度扫描网格，与第 3 节 PBR 结果对照；制作一组 linear/sRGB 混用的错误对比图。
- 检验标准：解释样本数与噪点关系及重要性采样如何降方差；推导 ρ/π；说出 gamma 混用的视觉症状（发灰/过曝）；Cornell Box 整体能量随弹射数增加不漂移。

**阶段 8｜动画与模拟概览（约 1–2 周）**
- 学习目标：建立实时动画与物理模拟图景，看懂引擎中骨骼动画与物理系统的原理层。
- 核心知识点：关键帧与插值（线性/样条）；骨骼层级变换、蒙皮矩阵、线性混合蒙皮（LBS）及其缺陷；质点弹簧与布料；数值积分：显式欧拉发散 → 半隐式欧拉 → Verlet；刚体与碰撞检测概念；粒子/流体（PBD、SPH）概念级了解。
- 实践任务：布料/弹簧质点 demo，对比显式与半隐式欧拉的稳定性；（选做）加载播放一个 glTF 骨骼动画并打印骨骼矩阵链。
- 检验标准：解释显式欧拉为何发散、半隐式为何稳定；描述蒙皮顶点变换流程（绑定姿态逆 × 当前骨骼变换 × 权重混合）；说明角色动画为何用骨骼蒙皮而非顶点级物理。

**推荐资源**：清单第 2 条 GAMES101（课程主页 https://sites.cs.ucsb.edu/~lingqi/teaching/games101.html ，B 站有完整视频与作业）为主线，八个阶段与其大纲一一对应；第 3 条 Scratchapixel 按专题随查；第 11 条 3Blue1Brown、第 12 条 Ray Tracing in One Weekend、第 13 条 PBRT 分见各阶段；GAMES202（https://sites.cs.ucsb.edu/~lingqi/teaching/games202.html）是本节学完后的实时渲染进阶下一站。

**常见误区**：
- 只看视频不写作业——GAMES101 的价值七成在作业，两个贯穿项目必须亲手迭代。
- 第一遍就用 GLM 代替手写数学库——错过建立几何直觉的关键机会；工程化阶段再换 GLM。
- 前置数学贪多求全、迟迟不进图形学正文——只学前置子集，学完即走。
- 全程跳过推导只记结论——理论根基空，违背本节定位。
- 把软件光栅器当性能工程做——本节目的是理解原理，性能意识归第 1 节。
- 跳过辐射度量学直接抄 PBR 公式——结果是“能跑但讲不清”。
- 一步到位写工业级路径追踪器——应迭代：Whitted → 漫反射蒙特卡洛 → 完整 BRDF/重要性采样。

## 第 3 节 OpenGL（渲染 API 实操）

**阶段 1｜起步：窗口、首个三角形与可调试环境（约 1–2 周）**
- 学习目标：建立对 OpenGL 的正确心智模型——它是图形 API 状态机规范而非软件库；搭好工程环境画出第一个三角形，并从第一天起点亮“仪表盘”：调试输出与抓帧调试器先于一切复杂功能就位。
- 核心知识点：OpenGL 3.3 core profile 与上下文创建、GLFW 窗口与输入、GLAD 扩展加载；图形管线概览：顶点着色器 → 图元装配/光栅化 → 片元着色器（两段着色器即本节 GLSL 的全部活动范围，深入归第 5 节）；着色器编译/链接/错误查询与 uniform 设置；VAO/VBO/EBO 职责与关系（VAO 记住“顶点怎么读”，VBO 记住“数据是什么”）；CPU↔GPU 数据流第一课：glBufferData 是拷贝上传而非引用共享、buffer orphaning 动机；glDebugMessageCallback 调试输出与级别过滤；NVIDIA 独显环境确认（驱动版本与 core profile 支持）。
- 实践任务：以 CMake 搭建渲染器骨架工程（工具链习惯与第 1 节合流），确立“教程 demo 验证 → 沉淀进渲染器”的双线工作流；实现 Shader 封装（加载/编译/链接，报错可读、支持重载）；渲染纯色三角形与 uniform 变色四边形；接入调试输出，故意写错一个着色器与一次非法枚举观察报错形态；用 RenderDoc 完成首次抓帧，逐项检视 draw call、管线状态与顶点数据。
- 检验标准：能口述 VAO/VBO/EBO 各自“记住什么”并回答“删掉 VAO 只留 VBO 会怎样”；调试输出零高级别报错稳定运行；能在 RenderDoc 中定位该帧 draw call、查看顶点缓冲内容与着色器源码；能解释“修改 CPU 端数组后画面为何不变”，说出数据何时真正走向 GPU。

**阶段 2｜变换、相机与纹理：渲染器基础可用（约 2 周）**
- 学习目标：让物体经完整的模型-视图-投影链路正确落屏，掌握纹理资源与采样状态实操，拿到自由观察的相机——本阶段末自写渲染器达成里程碑“基础可用”，成为第 4 节 2D 小引擎的最低渲染地基。
- 核心知识点：纹理对象全流程：stb_image 加载、wrap/filter 参数、mipmap 生成与三线性采样、像素行对齐与 y 翻转；sRGB 纹理概念级认知（色彩空间深入归第 2 节）；GLM 消费结论式使用（推导归第 2 节）：translate/rotate/scale、perspective/lookAt；NDC 与视口变换；深度测试开启、z-fighting 初见与应对；varying 插值与 flat 限定符；FPS 相机：欧拉角 pitch/yaw 约定、鼠标轮询、WASD 移动向量推导。
- 实践任务：渲染多个不同变换的纹理立方体并验证深度遮挡正确；实现鼠标+WASD 自由相机与滚轮 FOV；采样参数对比实验（最近/线性/mip 开关各截一图归档）；补一条 2D 路径：单纹理 quad + 2D 变换矩阵渲染精灵，为第 4 节小引擎验证地基。
- 检验标准：能解释“每个物体各自的 model、共享的 view/projection”这一组织方式及原因；RenderDoc 中可核对 mip 层级与纹理绑定正确；相机俯仰限制在 ±90° 内无翻转；里程碑“渲染器基础可用”：一份场景描述即可驱动纹理物体集合 + 自由相机 + 深度正确的完整画面，第 4 节可直接在此之上起步。

**阶段 3｜光照与材质（约 2–3 周）**
- 学习目标：从“贴图盒子”升级为“被光照亮的场景”：接线并调参局部光照管线，建立材质从 uniform 硬编码到贴图驱动的演进意识；光照数学推导归第 2 节，本节聚焦“在 API 上正确搭起来”。
- 核心知识点：Phong 与 Blinn-Phong 的 GLSL 落地与分量调参；法线矩阵为何用逆转置；光源类型接线：方向光、点光与距离衰减、聚光内外锥软化；多光源 uniform 数组组织；材质贴图组合：diffuse/specular/自发光贴图与缺省兜底链；阴影贴图两 Pass 实操：深度贴图渲染、采样比较、bias 对治自遮挡与 peter-panning；光照的 Gamma 正确姿势（线性空间计算、输出校正）。
- 实践任务：渲染器顶点格式扩展为 position/normal/uv；实现“方向光 + 多盏点光 + 一盏聚光”组合场景；实现材质贴图驱动（同一模型换材质零改码）；实现 shadow mapping，并用 RenderDoc 检查 shadow pass 的深度贴图内容——此场景同时冻结为第 2 节软件光栅器的光照对照基准。
- 检验标准：能解释法线为何不能直接用 model 矩阵变换；阴影场景无明显 acne/peter-panning 并能口述 bias 权衡；能在 RenderDoc 中找到 shadow depth pass、读出深度编码并解释；改一处光照参数能预测画面变化方向（调参具备因果模型）。

**阶段 4｜模型加载与渲染器分层：模型加载主体完成（约 2–3 周）**
- 学习目标：从手写顶点走向真实美术资产：打通 Assimp/glTF 加载管线并固定“学习资产”；同时把已具规模的代码分层收敛为可复用渲染器模块——本阶段末同时达成里程碑“模型加载主体完成”（第 6 节 Vulkan 复刻的启动前提）与“渲染器分层收敛”（第 4 节 IRenderer 接口化与第 6 节双后端对照的对接面）。
- 核心知识点：Assimp 的 aiScene/aiMesh/aiMaterial 场景图与后处理标志；顶点/索引/材质提取与索引化组织；材质-贴图兜底链；Mesh/Model/Texture 类职责与文件划分；glTF 格式要点：节点层级与 PBR 材质字段预埋（供阶段 6 与第 2 节对照消费）；资源目录约定、加载失败容错与日志；工程分层：应用 demo / 场景装配 / 渲染器核心单向依赖，GL 调用全部收进渲染器模块（接口抽象由第 4 节完成，本节先收拢实现）。
- 实践任务：实现 Model/Mesh 加载器，渲染一个多 mesh 多贴图模型并固定为“学习资产”（建议 glTF 源，供第 2/5/6 节同款复用对照）；执行分层重构：应用层不再直接触达 GL 调用，渲染器以“初始化 / 加载资源 / 提交绘制”三段式对外；做一个通用观察 demo：任意丢入模型即可绕相机旋转展示（自带光照与阴影）。
- 检验标准：任意 glTF/OBJ 模型（含多 mesh、多贴图、嵌套层级）加载渲染无报错；能画出 Assimp 场景图到自研 Model 树的映射关系图；全工程检查不到应用层出现裸 gl* 调用；里程碑“模型加载主体完成”：学习资产 + 光照 + 相机 + 阴影的完整场景冻结为对照基准，第 6 节可据此启动 Vulkan 复刻；里程碑“渲染器分层收敛”：第 4 节可直接在其上定义 IRenderer 搭 2D 小引擎、第 6 节以此为复刻迁移源。

**阶段 5｜帧缓冲、后处理与实例化绘制（约 2–3 周）**
- 学习目标：掌握 FBO 离屏渲染与“先渲到纹理、再屏幕加工”的现代引擎标配模式；用实例化把重复绘制合并为单次调用，建立 draw call 预算与状态排序意识。
- 核心知识点：FBO 构成（color/depth attachment）、完整性检查与常见失败原因；离屏渲染与屏幕 pass 分离；后处理链级联：反转/灰度/核卷积（模糊、边缘检测），逐效果带调试开关；离屏 MSAA 与 resolve；Gamma 正确性：卷积在线性空间、末端正 gamma；实例化：per-instance divisor（glVertexAttribDivisor）、变换矩阵打包进实例缓冲、glDrawElementsInstanced；draw call 成本模型：CPU 提交与状态切换开销 vs GPU 绘制本身，状态排序与纹理数组/图集思路。
- 实践任务：渲染器加入离屏渲染路径与可级联后处理链（至少两级效果串联）；实例化渲染 1000+ 实例的学习资产阵列或小行星带，每实例独立变换；用 RenderDoc 统计优化前后 draw call 数与帧时间（NVIDIA 卡可辅以 Nsight Graphics 预热，三件套深入归第 5 节）。
- 检验标准：能口述一次 draw call 的 CPU/GPU 成本构成，实例化后 draw call 数量下降且有帧时间数据佐证；能解释后处理为何必须在线性空间卷积、离屏 MSAA 为何需要 resolve；后处理链逐级开关不影响前级结果（正确性与可调试性并存）；分层结构在本阶段扩展后依然成立（GL 调用未泄漏回应用层）。

**阶段 6｜延迟渲染与 PBR/IBL：PBR 完成（约 3–4 周）**
- 学习目标：搭起两大现代实时渲染主线：延迟管线与 PBR/IBL。Cook-Torrance 与 split-sum 的理论推导归第 2 节，本节负责“在 API 上正确落地”，并产出供第 2 节 CPU 复刻逐项对照的 PBR 基准结果。
- 核心知识点：G-buffer 多渲染目标（MRT）布局：position/normal/albedo+材质参数的打包与精度取舍；全屏 quad 光照 pass 与 G-buffer 采样；前向 vs 延迟取舍：光源数量 vs 带宽、透明物体前向回退；PBR 直接光照落地：albedo/metallic/roughness/AO 参数流与 Cook-Torrance 三件套接线；IBL 三项预计算：辐照度卷积、预滤波环境 mip、BRDF LUT（split-sum 的“会用”层）；HDR 管线与色调映射。
- 实践任务：把学习资产场景改为延迟管线（G-buffer 可视化调试视图 + 光照 pass + 透明物体回退）；实现 PBR 材质：渲染金属度/粗糙度 0–1 步进扫描矩阵，冻结为第 2 节对照基准资产；实现 IBL：天空盒加载与三项预计算 pass 接入场景；HDR 输出与色调映射收尾。
- 检验标准：G-buffer 各 attachment 有可视化调试视图且内容逐项核对正确（法线图/反照率图）；能说清“100 盏点光在前向与延迟下的成本结构差异”；同一材质在前向/延迟两条管线下视觉一致；里程碑“PBR 完成”：扫描矩阵 + IBL 场景冻结为第 2 节软件光栅器/路径追踪器的对照基准。

**阶段 7｜现代延伸：DSA 与 AZDO 低开销路线（约 2–3 周）**
- 学习目标：从传统绑定模式升级到 OpenGL 4.5+ 现代用法，理解 AZDO“把驱动侧隐藏开销削掉”的思想与手段；同步观念从“glBufferData 拷贝”进化到“持久映射 + 显式 fence”，为第 5 节 GPU 驱动渲染与第 6 节显式 API 迁移铺观念地基。
- 核心知识点：DSA 直接状态访问：glCreate*/glNamedBuffer*/glTexture* 系列与“绑定→修改→解绑”模式的逐操作对照及迁移收益；持久映射：glNamedBufferStorage + GL_MAP_PERSISTENT_BIT、环形缓冲与帧内偏移、glFenceSync/glClientWaitSync 的 CPU↔GPU 同步边界（同步语义深化归第 5/6 节）；间接绘制：glMultiDrawElementsIndirect 的 command buffer 结构（本节只做 CPU 端填写，GPU 端生成命令归第 5 节 compute）；纹理数组/图集与“一次绑定、多次绘制”；AZDO 全景：合并提交、减少状态校验与冗余切换。
- 实践任务：渲染器资源层迁移 DSA 写法（buffer/texture/framebuffer 的创建与更新，保留开关与绑定版对照）；动态 uniform 更新改为持久映射 + 环形缓冲，消除每帧 glBufferData 重分配；用 glMultiDrawElementsIndirect 把阶段 5 的实例化场景重写为单次调用；记录迁移前后 RenderDoc 时间线与帧时间对照，形成一份“传统 vs 现代”小报告。
- 检验标准：能对任一资源写出“绑定版 vs DSA 版”等价调用序列并说明差异点；持久映射版无同步报错，能解释 fence 为何放在该处（何时 CPU 会踩到 GPU 正在读的区段）；间接绘制版与逐实例 glDrawElementsInstanced 版渲染结果一致；里程碑“现代 API 观念就绪”：CPU↔GPU 显式同步与命令式提交习惯建立，第 5 节 compute 并入（以 LearnOpenGL SSBO 章节作回顾起点）与第 6 节 Vulkan 复刻的观念、资产两线就绪。

**推荐资源**：清单第 1 条 LearnOpenGL（中文版 https://learnopengl-cn.github.io）为主线骨架，阶段 1–6 与其“入门 → 光照 → 模型加载 → 高级 OpenGL → 高级光照/PBR”章节一一对应，阶段 7 教程未覆盖、由以下补充承接；第 21 条 RenderDoc 自阶段 1 起作为调试主线工具；第 9 条 GDC Vault 于阶段 7 检索 Approaching Zero Driver Overhead in OpenGL（AZDO 思想源头演讲）。少量补充：docs.gl（https://docs.gl）为函数级权威速查；Khronos OpenGL Wiki（https://www.khronos.org/opengl/wiki/）是 DSA、持久映射与同步语义的权威参考；glTF 2.0 规格（https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html）对应阶段 4。

**常见误区**：
- 只跟教程抄 demo、不沉淀自写渲染器——本节价值一半在双线中的渲染器资产，抄完即弃会让后续各节失去联动地基。
- GL 调用散落在各 demo 的 main 里越堆越多，不封装、不分层——全局状态机 + 面条代码，到模型加载阶段就不敢动代码。
- 出问题不抓帧、不看调试输出，靠改魔法参数碰运气——黑盒式调试是本节最大反模式，调试环境阶段 1 就该点亮。
- 在变换与光照处陷入数学推导出不来，或反过来跳过推导只记结论——本节以“会用结论、知道理论归第 2 节”为界，两个极端都违背定位。
- 着色器越写越深、滑向“GLSL 艺术创作”——本节着色器止于“能读懂并使用”，compute 与深入 GLSL 归第 5 节。
- 依赖上一帧残留状态（绑定、深度测试、剔除开关），渲染“时对时错”——每个 pass 应显式声明所需状态，状态机残留是隐形 bug 温床。
- 每帧 glBufferData 重分配、逐 uniform 全量上传——养成即用即抛习惯，到 AZDO 阶段被迫大改，应尽早切换持久映射。
- RenderDoc 只在崩溃时才打开——应作为常规观察窗口，每个阶段主动抓帧核对管线状态与资源内容。

## 第 4 节 游戏开发软件架构设计

**阶段 1｜游戏循环与实时架构（约 1–2 周）**
- 学习目标：理解游戏是“持续运行、主动轮询、有帧预算”的实时软件，写出结构正确的游戏循环。
- 核心知识点：循环三段式（输入 → update(dt) → render）；可变步长 vs 固定步长+渲染插值，物理/逻辑为何需要确定性步长；帧预算（16.6ms/8.3ms）下的追赶与降级策略；UE 对照：World 的 Tick 调度与 TickFunction 分组依赖。
- 实践任务：把“方块按帧移动”的错误 demo 改造为固定步长+插值版本，作为第 3 节渲染器之上的 2D 小引擎骨架。
- 检验标准：锁 30/60/144 帧率下运动速度与物理行为一致；能解释“为什么不能在 while(running) 里裸算 dt”。

**阶段 2｜对象模型演进：继承 → 组件 → ECS（约 2–3 周）**
- 学习目标：理解“如何组织场景里成百上千个行为各异的对象”，掌握三代模型的动机与适用规模。
- 核心知识点：深继承树的组合爆炸问题；GameObject+Component 组合模型及组件间通信难点；ECS：Entity 只是 ID、Component 是纯数据、System 是逻辑，数据布局与缓存命中（衔接第 1 节）；UE 对照：AActor+UActorComponent 体系、UE5 Mass Entity 作为大规模 ECS 的认知锚点。
- 实践任务：同一批“会移动、会碰撞、会闪烁”的方块，分别用深继承、组合、自写 toy ECS 各实现一版。
- 检验标准：能画出三种模型结构图并陈述取舍；用第 1 节的 profiler 对比三版在 1000+ 实体下的帧耗时差异。

**阶段 3｜行为决策与输入：谁做决定（约 2 周）**
- 学习目标：把敌人 AI 与玩家输入从散落的 if-else 提炼为显式结构。
- 核心知识点：有限状态机：枚举 switch → 状态对象 → 分层状态机（HFSM）的演进理由；行为树：选择/序列/并行组合节点、装饰器、黑板，与 FSM 的适用边界（响应式 vs 长流程规划）；命令模式：输入抽象、撤销/重放（为回放系统埋点）；UE 对照：行为树资产+黑板的可视化调试、Enhanced Input 的命令化思路。
- 实践任务：为敌人实现“巡逻-追击-攻击-逃跑”的 FSM 与行为树各一版；玩家输入以 Command 抽象接入小引擎。
- 检验标准：新增“眩晕”状态/行为时改动范围局部化；行为树有可视化调试输出（当前执行到哪个节点）。

**阶段 4｜事件与消息系统：谁通知谁（约 2 周）**
- 学习目标：解耦模块间通信，同时不牺牲可调试性。
- 核心知识点：观察者的经典坑：订阅者销毁后的悬垂回调、注册泄漏与 RAII 反注册（衔接第 1 节）；立即回调 vs 排队派发：“帧内修改正在遍历的容器”必须延迟到帧末；服务定位器与单例的边界：便利 vs 隐藏依赖；UE 对照：多播委托与弱类型消息路由并存的设计。
- 实践任务：小引擎加入事件总线，用事件版重构“受伤→死亡→掉落→加分”调用链，与直接调用版对比代码结构。
- 检验标准：订阅者中途销毁不崩溃；能举出一个“必须帧末处理”的例子并解释原因；能回答“这个事件是谁发出的”（调试器/日志可追）。

**阶段 5｜资源与生命周期管理（约 2 周）**
- 学习目标：管住纹理/网格/音频等重资源的加载、共享、卸载与失效。
- 核心知识点：句柄（ID+代数）替代裸指针：悬垂资源指针是引擎级事故；资源注册表与缓存去重；引用计数 vs GC vs 显式分代的取舍；异步加载与加载界面；粒子等高频对象池化；脏标记避免无谓重算；UE 对照：UPROPERTY 与 GC、Asset Manager 异步流送。
- 实践任务：实现句柄式资源管理器（纹理/网格）+ 粒子对象池。
- 检验标准：句柄失效时安全降级不崩溃；用第 1 节工具验证无泄漏；池化后热路径零分配。

**阶段 6｜场景组织、序列化与引擎分层（约 2–3 周）**
- 学习目标：让数据和代码各归其位，小引擎从“一堆代码”变成有边界的引擎。
- 核心知识点：场景树与变换层级（父子矩阵复合，数学推导归第 2 节，本节用结论）；四叉树/BVH 概念级认知；序列化三分：存档（运行时状态）≠ 关卡数据（静态配置）≠ 编辑器数据；分层：平台层/核心层/引擎模块/游戏逻辑单向依赖，渲染收敛为抽象后端接口（为第 6 节 Vulkan 复刻埋点）；UE 对照：模块与插件边界、World/Level/Actor 层级。
- 实践任务：实现场景树+变换层级；JSON 关卡格式（保存/加载整个实体-组件场景）；把所有渲染调用收敛到 IRenderer 接口后面。
- 检验标准：一个 JSON 文件即可实例化完整场景；渲染后端换成 stub 实现时游戏逻辑零改动。

**阶段 7｜综合：小引擎 1.0 与模式复盘（约 2–3 周）**
- 学习目标：整合全部专题完成可玩小游戏；建立“模式的代价清单”这一批判意识。
- 核心知识点：每个模式的成本维度（性能开销/间接层复杂度/调试可见性）；KISS 与 YAGNI 在游戏语境的落地；“先有痛点，再上模式”。
- 实践任务：用小引擎完成一个小游戏（平台跳跃或太空射击）；为每个用了/没用/中途移除的模式写一条架构决策记录（ADR），说明理由。
- 检验标准：小游戏从头到尾可玩；ADR 覆盖全部已学模式；能针对任一模式回答“去掉它会坏什么、留着它多花什么”。

**推荐资源**：清单第 6 条 Game Programming Patterns（免费在线）为主线教材，按阶段对应章节推进；第 8 条 Valve 开发者 Wiki 是阶段 4/6 的商业级参照；第 9 条 GDC Vault 重点检索 Overwatch Gameplay Architecture and Netcode（ECS+确定性模拟的经典案例）与 engine architecture 主题演讲；第 14 条 Unreal 官方文档支撑各阶段 UE 对照（源码阅读与第 1 节合并）；第 15 条 EnTT 是阶段 2 的对照阅读对象；加深读物：《Game Engine Architecture》（Jason Gregory）。

**常见误区**：
- 模式堆砌与提前抽象：为“架构完美”而非具体痛点引入间接层；自写引擎最常见的死法是“引擎还没写完，游戏一个没做”。
- 把 ECS 当银弹：几十个实体的小项目强行 ECS，复杂度反噬；ECS 收益在规模与数据布局，不在时髦。
- 全局单例与 God Object：所有模块都能访问一切，等于没有边界。
- 生命周期失控：悬垂回调、遍历容器时增删实体、模块间销毁顺序依赖；事件系统会放大此问题。
- 事件滥用：一切皆事件导致调用链不可见，“谁发出了这个事件”无从追查，调试成本超过解耦收益。
- 序列化后补：先写死代码与内存布局，后期才发现存档/关卡格式无法演进。

## 第 5 节 GPU 编程

**阶段 1｜GLSL 深入与 Compute Shader 实操入口（OpenGL 4.3+）（约 2–3 周）**
- 学习目标：从“能读懂”升级为“能编写”，独立实现并调试图形与计算着色器；掌握 compute 的调度与同步模型，能安全地在 GPU 端读写数据。
- 核心知识点：GLSL 系统：向量/矩阵类型与 swizzle、in/out/uniform/layout 限定符、精度与编译器行为；compute 调度模型：gl_GlobalInvocationID/gl_WorkGroupID/gl_LocalInvocationID 三层坐标、local_size 与 dispatch 的乘积关系；数据通路：SSBO、image load/store、原子操作与原子计数器；同步语义：glMemoryBarrier 各位含义（何时必须加、加在哪端）与 barrier() 组内同步；工程模式：ping-pong 双缓冲、CPU 读回仅用于验证。
- 实践任务：Hello Compute（写入 → dispatch → barrier → 读回校验）；灰度/反色/高斯模糊 compute 化并与 fragment 版逐像素比对；引入 shared memory 做 tiled 模糊或组内归约，粗测与无 shared 版的耗时差距。
- 检验标准：不查资料画出三层调用坐标映射图，解释一次 dispatch 产生多少线程；能举例回答“何时需要 GL_SHADER_STORAGE_BARRIER_BIT、何时需要 GL_TEXTURE_FETCH_BARRIER_BIT”；模糊结果与 fragment 版一致无撕裂，能口述 ping-pong 与读回同步的必要性。

**阶段 2｜GPU 体系结构与 SIMT：CUDA + PMPP 前半（约 3–4 周）**
- 学习目标：建立硬件视角，理解 GPU 吞吐导向设计与“用并行换延迟”的本质；打通 CUDA 与 GLSL 的术语映射。
- 核心知识点：CPU vs GPU 设计哲学：延迟隐藏 vs 延迟规避；SIMT 执行：warp 调度、控制流发散的代价与重汇聚；线程层级与硬件映射：workgroup/block 落在 SM 上，block 内可共享、block 间不可直接通信；内存层级全景：寄存器/shared/local/global/constant/texture 的容量、带宽、延迟量级，coalesced 访存与 cache line；CPU↔GPU 数据搬运：PCIe 带宽、异步传输与流水（呼应第 3 节 AZDO 思想）。
- 实践任务：用 CUDA 复刻阶段 1 的模糊/归约，跑通 nvcc 工具链并用 Nsight Systems 看时间线；warp 内分支交错 vs 按分支重排数据的发散对比实验；矩阵转置 naive vs shared tiling 两版，记录带宽利用率变化。
- 检验标准：能默写内存层级表（每级典型容量、延迟、带宽数量级）；解释 shared tiling 为何提升转置带宽（coalesced 读写路径）；能给出 CUDA/GLSL 术语对照表并正确互译一段简单 kernel。

**阶段 3｜并行算法模式：reduction / scan 及图形落地（约 3–4 周）**
- 学习目标：掌握可复用的并行算法原语，理解复杂度权衡，能识别渲染管线中“这其实是个 reduction/scan”的场景。
- 核心知识点：reduction：树形归约、两级方案（块内 shared 树形 + 块间原子或第二趟）、原子串行化陷阱；scan：inclusive/exclusive 前缀和、Hillis-Steele（步优）vs Blelloch（工作优）；histogram：原子竞争与 privatization；stream compaction：scan 驱动的 scatter 紧凑化——粒子/剔除系统的核心模式；步复杂度 vs 工作复杂度的权衡框架。
- 实践任务：实现两级 reduction 与 Blelloch exclusive scan（配标准测试集：全 0、全 1、递增序列）；图形落地 A：图像直方图 + 均衡化后处理；图形落地 B：scan + scatter 做“存活粒子压缩”。
- 检验标准：全部测试通过并能口述两级方案的划分理由；举出渲染管线中至少 3 个可归约为 reduction/scan 的场景（遮挡剔除的可见列表生成、粒子压缩、实例包围盒归约等）。

**阶段 4｜性能工程：occupancy、带宽与 Nsight 三件套（约 2–3 周）**
- 学习目标：建立“基线 → 剖析 → 单变量修改 → 复测”的测量驱动工作流，摆脱感觉式优化。
- 核心知识点：瓶颈分类：计算受限/带宽受限/延迟受限/同步受限，roofline 模型入门；occupancy 的真实含义与“高 occupancy ≠ 高性能”的边界条件；访存合并、事务粒度、bank 冲突及其在 GLSL shared 中的迁移；工具分工：Nsight Systems（整机时间线、CPU/GPU 重叠）、Nsight Graphics（帧级渲染剖析）、Nsight Compute（kernel 级指标：带宽利用率、发散率）；CPU↔GPU 同步点识别：读回、不必要 fence、隐式同步对流水线的破坏。
- 实践任务：为阶段 2–3 的 kernel 建立性能基线表（耗时、有效带宽、带宽利用率、occupancy）；完成一轮规范优化迭代（如归约原子版 → 两级版）并记录前后指标；用 Nsight Systems 检查提交与执行重叠、消除至少一个同步点。
- 检验标准：产出“问题-假设-改动-数据”四栏优化记录，每个结论有数据支撑；能据 Nsight Compute 报告判断某 kernel 属于带宽还是计算瓶颈，并给出对应优化方向。

**阶段 5｜图形应用专题：并入第 3 节渲染器（约 4–6 周）**
- 学习目标：在第 3 节自写渲染器上实现 GPU 驱动渲染的典型特性，形成“图形 API + 并行计算”闭环。
- 核心知识点：GPU 粒子系统：发射器模型、半隐式欧拉积分、SOA 布局、ping-pong 模拟、存活压缩（阶段 3 产出）、billboard 输出（实例化或间接绘制）；GPU 剔除：compute 视锥剔除生成实例可见列表，进阶 HiZ 遮挡剔除与 depth pyramid；间接绘制：glMultiDrawElementsIndirect 与 GPU 端写 command buffer（呼应第 3 节 AZDO）；后处理链 compute 化：分离模糊、bloom、tiled 后处理与 shared 复用；数据路径选型：image load/store、SSBO、采样纹理的一致性代价。
- 实践任务：GPU 粒子系统（10 万+ 粒子，模拟与压缩全在 GPU，CPU 每帧零分配）；compute 视锥剔除 + 间接绘制，海量实例压到极少量 draw call；选做其一：HiZ 遮挡剔除或 compute 版 bloom 后处理链。
- 检验标准：功能正确（无 ghost 粒子、剔除无漏判），Nsight 下帧时间有量化提升或不劣化；确认热路径无 CPU↔GPU 强同步点、无每帧读回；能向他人讲清“CPU 上传 → compute 更新 → 归约/压缩 → 间接绘制”的完整数据流。

**推荐资源**：清单第 7 条 PMPP（第 5 版）为阶段 2–4 主教材，GPU Gems 1–3 作按需检索的案例库；第 16 条 CUDA 与 Nsight 官方文档为阶段 2–4 的权威参考；第 1 条 LearnOpenGL 的 SSBO 章节作阶段 1 回顾起点；第 9 条 GDC Vault 检索 GPU 驱动渲染、剔除与 compute 管线讲座。

**常见误区**：
- 用 CPU 思维写 GPU：依赖相邻线程的执行顺序、逐步赋值传递数据。
- 把原子操作当免费午餐：大量原子竞争导致隐性串行化。
- 忽略 barrier 与内存模型的“能跑就行”：竞态是概率性 bug，换驱动/换卡即翻车。
- 两个极端：不测量就优化，或盲目拉满 occupancy（寄存器压力反噬性能）。
- 调试用读回常态化：读回即同步，破坏流水线，仅验证时使用。
- shared memory 万能论：低复用小数据上徒增复杂度与 bank conflict 风险。
- CUDA 与 GLSL 术语混着记：务必建立 block↔workgroup、warp↔subgroup、global↔SSBO 对照表。
- 把 PMPP 当理论书通读不动手：每章示例须在自己环境跑过再前进。

## 第 6 节 Vulkan

**阶段 1｜启程：显式 API 心智模型与可调试环境（约 1 周）**
- 学习目标：理解 Vulkan 与 OpenGL 的哲学差异（隐式状态机 → 显式对象+显式同步），搭好可调试的开发环境，完成实例与设备初始化。
- 核心知识点：Vulkan 对象模型总览（驱动不再隐式帮忙）；instance 与 layer/extension 机制；validation layers；物理设备选择与 queue family；logical device 与窗口 surface；对象销毁顺序与生存期初步。
- 实践任务：用 CMake 搭建项目（复用第 1 节工具链）；骨架程序完成 instance → surface → device 创建并接入校验层；故意写一个错误（如颠倒销毁顺序）观察 validation 输出。
- 检验标准：骨架程序 validation 零警告运行；能说清“Vulkan 比 OpenGL 多出的一千行初始化都在做什么”及 extension 与 validation layer 的区别。

**阶段 2｜上屏：交换链、命令提交与同步入门（约 2 周）**
- 学习目标：打通“渲染一帧并呈现”的最小闭环，建立 CPU↔GPU 同步的初步模型。
- 核心知识点：swapchain 创建参数、图像获取与呈现；image view；command pool/buffer 录制；queue submit 与 present；fence（CPU 等帧完成）与 semaphore（图像获取/呈现交接）的分工；in-flight frames 双帧并行；窗口 resize 与 swapchain 重建。
- 实践任务：渲染清屏色与渐变三角形（对应 vulkan-tutorial “Drawing a triangle”完成点）；处理 resize/minimize 时的 swapchain 重建；实现 2 个 in-flight frame。
- 检验标准：无 validation 错误稳定运行；能手画“一帧的时间线”解释 fence 与 semaphore 各自挡住什么；删掉某个 semaphore 后能解释观察到的撕裂或卡死现象。

**阶段 3｜管线与着色器：图形管线双路线（约 2–3 周）**
- 学习目标：建立 graphics pipeline 的完整对象模型，打通 GLSL→SPIR-V 工具链，同一场景以两种方式建管线并对比。
- 核心知识点：管线全状态（vertex input/IA/viewport/光栅化/MSAA/blending）；shader module 与 SPIR-V；pipeline layout；descriptor set layout/pool/allocate/update 与 push constants；传统路线：render pass + framebuffer（attachments、subpass dependency）；新路线：Vulkan 1.3 dynamic rendering 及两者取舍；pipeline cache。
- 实践任务：渲染带 MVP uniform（descriptor set）的立方体；用 push constants 传时间做动画；同一立方体用 render pass 与 dynamic rendering 各实现一遍并对比代码量；接入 glslc/shaderc 编译脚本与着色器热重载。
- 检验标准：能解释 shader 中变量与 descriptor binding 的对应关系；能说出 dynamic rendering 省掉了什么、subpass dependency 在解决什么；两条路线代码可互换且结果一致。

**阶段 4｜内存与资源：上传、屏障与 VMA（约 2–3 周）**
- 学习目标：掌握 Vulkan 内存与资源管理的工业实践，攻克 GPU 内同步（barrier/image layout）。
- 核心知识点：内存类型（HOST_VISIBLE/HOST_COHERENT/DEVICE_LOCAL）；staging buffer 与顶点/索引/纹理上传；image layout 转移与 pipeline barrier（synchronization2）；深度缓冲接入；VMA 的创建/映射/池化与内存统计；每帧动态 uniform（持久映射+偏移或 UBO 数组）。
- 实践任务：加载第 3 节用过的同一模型与纹理（复用学习资产，便于对照）；用 VMA 重构阶段 3 的裸内存分配；实现帧内动态 uniform 更新；接入深度测试的多物体场景。
- 检验标准：渲染结果与第 3 节 OpenGL 版本对得上（RenderDoc 抓帧逐 pass 对比）；能完整叙述一次纹理上传的 layout 转移与 barrier 链条；能看懂 VMA 统计并说清内存去向。

**阶段 5｜后端工程化与复刻迁移：帧结构、render graph（约 3–4 周）**
- 学习目标：从“能跑的 demo”升级到“能当引擎后端”，完成第 3 节小渲染器的 Vulkan 复刻迁移。
- 核心知识点：概念对照迁移：VAO/VBO ↔ vertex input 状态、FBO ↔ render pass attachments、glClear ↔ load op、全局状态机 ↔ 显式对象；render graph/帧图思想（声明 pass 与资源读写 → 自动排序、自动屏障、资源别名）；primary/secondary command buffer 与多线程录制；timeline semaphore 简化多队列同步；per-frame 资源池与生命周期管理；pipeline cache 落盘加速启动。
- 实践任务：把第 3 节自写小渲染器（至少到光照）迁移为 Vulkan 后端；用简化版 render graph 描述该渲染器的 pass，由图自动生成屏障与渲染参数（如已学第 4 节，可挂到 IRenderer 接口后）；（选学）用一个 worker 线程录制 secondary command buffer。
- 检验标准：迁移版与 OpenGL 版帧结果一致；删掉任一处手工屏障后 render graph 自动补齐仍正确；1080p 中端 GPU 上稳定在 16ms 帧预算内；能讲清 render graph 相比手写 render pass 的收益与代价。

**阶段 6｜进阶选学：现代 Vulkan 特性（约 2–3 周）**
- 学习目标：按主流方向各取其一，理解特性的适用规模，而非追新。
- 核心知识点：bindless（descriptor indexing + update-after-bind）；GPU-driven 渲染（indirect draw + compute 剔除，概念衔接第 5 节）；mesh shader；硬件光追（RT pipeline + acceleration structure）；多队列与多线程深化。
- 实践任务：任选 1–2 个特性做独立 demo（可参照 Vulkan-Samples 与 Sascha Willems 示例改造）。
- 检验标准：能向他人讲清所选特性“省了什么、贵在哪、什么项目规模值得用”，并说明它对渲染后端架构的影响。

**推荐资源**：清单第 10 条官方中文教程为骨架主线，第 17 条 vulkan-tutorial.com 英文原版可对照阅读；第 18 条 vkguide.dev 对应阶段 5 的工程化方向；第 19 条 Vulkan-Samples 与 Sascha Willems examples 为官方与社区两大示例库；第 20 条 VMA 是阶段 4 的工业标准；第 21 条 RenderDoc 承接第 3 节抓帧调试习惯；第 9 条 GDC Vault 检索 Vulkan 与 render graph 相关演讲。

**常见误区**：
- 照抄教程代码而忽略 validation 输出——校验层就是 Vulkan 的“编译器报错”，关掉等于蒙眼开发。
- 把教程单文件结构一抄到底，不及时用 RAII 重构（直接调用第 1 节能力），代码越长越不敢动。
- 同步概念混成一锅粥：fence 管 CPU↔GPU、semaphore 管 queue 交接与呈现、barrier 管 GPU 内部依赖，用错对象或一律 DeviceWaitIdle 兜底。
- 把 Vulkan 当 OpenGL 用：每帧重建 pipeline、逐 draw 上传，既感受不到收益又加深“Vulkan 繁琐”的偏见。
- 新旧教程混搭困惑：2021 年前的资料几乎全是 render pass 写法，需主动识别资料所处路线。
- image layout 转移或 barrier 遗漏导致“偶现黑屏/花屏、只在部分驱动复现”，应用 RenderDoc 的 layout 时间线定位。
- 基础同步与生存期未过关就直奔 bindless/mesh shader，进阶学习退化为抄示例。
- 把 render graph 当必需品：小项目手写 render pass 完全可行，render graph 是规模化后的工程选择。
