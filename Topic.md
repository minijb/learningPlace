# 学习计划

用户角色:
角色：基础游戏开发工程师
需求：深入的学习游戏开发相关事宜

agent 角色:
担任教师，对用户进行指导, 默认用户是不了解的，可以对用户进行询问。

## 学习主线与项目联动

- 单一资产闭环：第 3 节的自写小渲染器是贯穿全程的核心实践资产，各节实践围绕它逐层演进——第 2 节用纯 CPU 的软件光栅器与路径追踪器复刻其渲染效果，与 OpenGL 结果逐项对照印证理论；第 4 节在其上搭建 2D 小引擎，做架构层面的演进；第 5 节把 compute 能力并入其中，形成 GPU 驱动渲染闭环；第 6 节把它复刻迁移为 Vulkan 后端，实现同一渲染器的双 API 对照。
- 联动只体现在实践项目的演进上：每节知识均自包含、完整覆盖本节主题，不因联动删减任何知识覆盖；跳过某一节时，其余各节的正文与路线仍然成立。
- 各节可独立学习、按需调整顺序；涉及前置关系之处（如第 6 节建议在第 3 节模型加载主体完成后启动）已在对应节的边界声明中注明。
- 双线先行起步：第 1 节阶段 1–2 与第 2 节阶段 1 并行起步，之后再进入第 3 节 OpenGL 主体；两线只约定相对先后，不绑定绝对时长。
- 各节的分阶段详细路线（阶段目标 / 实践任务 / 检验标准 / 资源 / 误区）统一收录于 [Stages.md](./Stages.md)，跨章节实践项目的里程碑清单与「项目×章节」双向对照收录于 [Projects.md](./Projects.md)，正文仅保留精炼要点。

## 跨章节实践项目

- 四个跨章节项目把散落在各节阶段里的实践任务收敛到几份持续演进的代码资产上，是「学习主线与项目联动」的落地载体；项目不新增学习内容，其里程碑与各节阶段的实践任务一一对应。
- **P1 自写渲染器**（贯穿主线资产，参与第 1/3/4/5/6 节）：自第 3 节窗口与三角形起步，经「基础可用 → RAII 化与 CMake 工程化 → 光照与模型加载 → 抽出 IRenderer 后端接口 → 延迟渲染与 PBR → compute 并入 → Vulkan 复刻」逐层演进；最终形态为同一套渲染器挂双 API 后端、GPU 驱动特性齐备，P2/P4 均运行其上。
- **P2 2D 小引擎**（第 4 节项目，建于 P1 之上）：在渲染器的 2D 路径上按「游戏循环 → 对象模型 → 行为/输入 → 事件 → 资源 → 分层与序列化」搭出 2D 小引擎；最终形态为一个 JSON 驱动场景、渲染收敛于 IRenderer、以架构可演进为目标的可玩小游戏。
- **P3 软件光栅器 + CPU 路径追踪器**（第 2 节 CPU 对照项目）：纯 C++、不绑定图形 API，复刻 GPU 所做的光栅化与全局光照，渲染与 P1 同款学习资产并逐项对照 OpenGL 结果；最终形态为两份能把「会用」印证为「懂原理」的 CPU 参照实现。
- **P4 GPU 驱动渲染特性集**（第 5 节产出，并入 P1）：以 compute 基本功与并行算法原语打底，产出 GPU 粒子、compute 剔除与间接绘制、compute 后处理并入渲染器；最终形态即 P1 的 GPU 驱动渲染闭环。
- 各项目阶段里程碑均为相对先后、不绑定绝对时长；里程碑清单、完成判据与「项目×章节」双向对照见 [Projects.md](./Projects.md)。

## 1. c++ 基础+深度理解
- 以现代 C++（17/20）为语言基准，先夯实 RAII、智能指针、值语义与移动语义等资源管理根基，再进入模板/泛型、可调用对象（std::function 等）与词汇类型（optional/variant/span）等高级主题。
- 理解 C++ 内存模型与多线程并发的基本边界：数据竞争、同步原语与内存序。
- 面向游戏实时性建立性能意识：帧预算、缓存友好数据布局与热路径零分配习惯。
- 具备工具链基本功：CMake 构建、调试器与性能剖析器定位问题。
- 以 Unreal Engine 源码与 C++ Core Guidelines 为标尺，培养大型工程阅读与重构素养。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 1 节 c++ 基础+深度理解](./Stages.md#第-1-节-c-基础深度理解)**。
> 📁 本节领域目录：[01-cpp/](./01-cpp/)（领域 Topic.md、共识 Context.md 与主教材教程 tutorial/）。

## 2. 计算机图形学（数学与渲染理论）
- 先以约 2–3 周系统学习线性代数与微积分核心：向量/矩阵的几何意义、矩阵变换与齐次坐标、导数/梯度与积分的直观；可配 3Blue1Brown 作直观辅助，但以系统学习为主线，学完图形学所需的这一个子集即进入正文，不无限前置。
- 以 GAMES101 课程为主线、Scratchapixel 为随查参考，两个纯 C++ 项目贯穿全程：软件光栅器与 CPU 路径追踪器，课程与项目双线推进。
- 吃透“一个顶点的完整旅程”：模型-视图-投影变换、透视除法与视口变换，能从零推导透视投影矩阵、按症状定位变换链路出错环节。
- 用软件光栅化复刻 GPU 所做之事：三角形覆盖、深度测试、透视校正插值、纹理采样与 mipmap、走样与抗锯齿原理，并掌握 Blinn-Phong、法线/阴影贴图等着色与材质理论。
- 换一条渲染路线理解全局：光线求交与 Whitted 光线追踪 → 辐射度量学与渲染方程 → 蒙特卡洛路径追踪与微表面 BRDF，把第 3 节“会用”的 PBR/IBL 升级为“懂原理”。
- 概览几何表示与动画模拟：贝塞尔曲线/曲面、隐式与显式表示、网格细分与简化；骨骼蒙皮、质点弹簧与数值积分的稳定性直觉。
- 边界声明：本节全程纯 CPU、不绑定任何图形 API——API 实操归第 3 节，GLSL 深入与 Compute 归第 5 节（本节 CPU 路径追踪器留作第 5 节 GPU 化的实践素材），Vulkan 归第 6 节，性能工程归第 1 节。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 2 节 计算机图形学（数学与渲染理论）](./Stages.md#第-2-节-计算机图形学数学与渲染理论)**。

## 3. OpenGL（渲染 API 实操）
- 以 LearnOpenGL 路线（窗口与首个三角形 → 变换与相机 → 光照 → 模型加载）为教程骨架，同步以一个自写小渲染器贯穿实操，教程与项目双线推进。
- 掌握核心资源与状态实操：VAO/VBO、纹理与采样、FBO 帧缓冲，理解 CPU↔GPU 数据流与同步。
- 按主流路线推进进阶专题：实例化绘制、帧缓冲与后处理、延迟渲染、PBR/IBL。
- 向现代用法延伸：4.5+ 的 DSA 直接状态访问与 AZDO 低开销方向（持久映射、间接绘制），理解其与传统绑定模式的差异。
- 建立图形调试习惯：以抓帧调试器与调试输出定位渲染问题。
- 边界声明：本节聚焦 API 实操，着色器到“能读懂并使用”为止；图形数学与渲染理论归第 2 节，GLSL 深入与 Compute 归第 5 节。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 3 节 OpenGL（渲染 API 实操）](./Stages.md#第-3-节-opengl渲染-api-实操)**。

## 4. 游戏开发软件架构设计
- 以 Game Programming Patterns 为教程骨架（游戏循环 → 状态/命令模式 → 组件化与 ECS → 事件解耦 → 资源管理 → 空间划分与序列化），同步在第 3 节自写渲染器之上搭一个 2D 小引擎贯穿实操，模式与工程双线推进。
- 从实时软件的时间模型入手：固定/可变时间步的游戏循环、update 与 render 分离、帧预算下的调度取舍，理解游戏与“请求-响应”式软件的本质差异。
- 掌握对象模型三代演进：深继承树的爆炸问题 → GameObject+组件组合 → ECS 数据导向设计，理解各自动机与适用规模，并与 Unreal Actor/Component 体系对照。
- 结构化两类核心问题：“谁做决定”（状态机/分层状态机/行为树/命令模式）与“谁通知谁”（观察者/事件队列/延迟派发），警惕全局单例与 God Object 蔓延。
- 管好重资源与生命周期：句柄替代裸指针、对象池、脏标记、异步加载；场景组织（变换层级/空间划分）与序列化三分（存档、关卡数据、编辑器数据）。
- 以分层架构收敛模块边界：平台层/核心层/引擎模块/游戏逻辑单向依赖，渲染收敛为可替换后端接口（为第 6 节 Vulkan 复刻埋点），并以 Unreal 模块体系为参照。
- 建立模式批判意识：逐个评估设计模式在游戏中的代价（性能、复杂度、调试性），以“该模式解决的问题在此真实存在吗”为准绳取舍，反对模式堆砌与提前抽象。
- 边界声明：本节聚焦组织架构，C++ 语言功底归第 1 节，渲染 API 细节归第 3 节，空间与变换数学归第 2 节，GPU 并行化归第 5 节，Vulkan 后端实现归第 6 节；小引擎不以画面为目标，以架构可演进为目标。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 4 节 游戏开发软件架构设计](./Stages.md#第-4-节-游戏开发软件架构设计)**。

## 5. GPU 编程
- 从第 3 节“能读懂并使用”的起点出发系统深入 GLSL：类型与限定符、内置变量与布局限定、精度与编译器行为，达到能独立编写与调试图形/计算着色器的水平。
- 以 OpenGL 4.3+ Compute Shader 为第一实践入口：调度模型（local_size、工作组、dispatch）、SSBO 与图像读写、barrier 与内存可见性，理解 GPU 端数据流与同步语义。
- 硬件按 NVIDIA 独显规划，以 CUDA + PMPP（第 5 版）深化体系认知：SIMT 执行与线程层级、warp 与控制流发散、内存层级（寄存器/shared/constant/texture/global）的带宽与延迟量级，打通 thread/block/grid ↔ invocation/workgroup/dispatch 术语映射，能回答“为什么这样写慢”。
- 掌握并行算法基础模式：reduction 与 scan（延伸 histogram、stream compaction），理解步复杂度/工作复杂度权衡，并能在图形场景中落地。
- 以 Nsight 三件套（Systems/Graphics/Compute）建立测量驱动的性能工程习惯：区分计算/带宽/同步瓶颈，理解 occupancy 与访存合并的真实含义，以带宽利用率为优化标尺。
- 将并行能力并入第 3 节自写渲染器：GPU 粒子（SOA 布局 + ping-pong 更新）、GPU 剔除与间接绘制、compute 后处理，形成 GPU 驱动渲染闭环。
- 以 GPU Gems 作按需检索的扩展案例库（而非通读），辅以 GDC/GTC 现代引擎 GPU 驱动管线讲座跟进工业界实践。
- 边界声明：本节聚焦 GPU 侧并行编程与 CUDA 主线，CUDA 仅作学习载体、不深入其生态特有库与部署；渲染数学与理论归第 2 节，图形 API 实操归第 3 节与第 6 节（Vulkan compute 的 API 细节归第 6 节），引擎级架构设计归第 4 节。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 5 节 GPU 编程](./Stages.md#第-5-节-gpu-编程)**。

## 6. Vulkan
- 以官方中文教程（docs.vulkan.net.cn，即 vulkan-tutorial 官方中文版）为骨架（环境与校验层 → 交换链与三角形 → 图形管线与描述符 → 帧循环与同步），同步以一个自写 Vulkan 渲染后端贯穿实操，教程与项目双线推进。
- 掌握显式 API 的对象模型与生存期管理：instance/device/queue、swapchain、command buffer 与队列提交、descriptor sets 与 push constants，理解相对 OpenGL“控制权从驱动转移到应用”意味着什么。
- 以同步为主线攻克最陡的学习曲线：fence/semaphore/barrier/timeline semaphore 各自的管辖范围，对 CPU↔GPU 与 GPU 内同步分层建模，摆脱 DeviceWaitIdle 兜底，养成“每一处等待都有明确理由”的习惯。
- 图形管线双路线推进：先以传统 render pass/framebuffer 建立完整心智模型，再过渡到 Vulkan 1.3 dynamic rendering，进而理解 render graph 思想所要解决的屏障与 pass 编排自动化问题。
- 内存与资源管理以 VMA 为工业标准路线：staging 上传、image layout 转移、持久映射与缓冲复用，配合 GLSL→SPIR-V 工具链（glslc/shaderc + 热重载）形成完整的资产与着色器流水线。
- 沿用第 3 节调试习惯并升级：validation layers 全程开启、RenderDoc 抓帧与 OpenGL 版结果逐帧对照，把生存期与同步错误在开发期暴露。
- 进阶按主流方向选学：多线程命令录制、bindless 资源绑定、mesh shader 与硬件光追，理解各特性“省了什么、贵在哪、什么规模值得用”。
- 边界声明：本节聚焦显式 API 实操与渲染后端工程化，着色器算法与 compute 深入归第 5 节，渲染理论归第 2 节，引擎整体架构归第 4 节；建议在第 3 节完成模型加载主体后启动本节，以“第 3 节小渲染器的 Vulkan 复刻”作为迁移检验里程碑。

> 📋 本节分阶段详细路线（学习目标 / 核心知识点 / 实践任务 / 检验标准 / 推荐资源 / 常见误区）见 **[第 6 节 Vulkan](./Stages.md#第-6-节-vulkan)**。

## 推荐参考网站(只是推荐一个搜索方向)

1. LearnOpenGL（https://learnopengl.com，中文版 https://learnopengl-cn.github.io）— 现代 OpenGL 系统教程，对应第 3 节
2. GAMES101（现代计算机图形学入门）— 图形学数学与渲染理论公开课，对应第 2 节
3. Scratchapixel — 计算机图形学理论与实现参考，对应第 2 节
4. CppCon — C++ 工程与深度讲座视频库，对应第 1 节
5. cppreference（https://en.cppreference.com/）— C++ 语言与标准库权威文档，对应第 1 节
6. Game Programming Patterns — 游戏架构模式在线免费书，对应第 4 节
7. GPU Gems（NVIDIA，全套免费在线）— GPU 编程与渲染技术实例库，对应第 5 节；成书较早（2004–2007），系统性的现代 GPU 通用编程可另见 PMPP（Programming Massively Parallel Processors，第 5 版）
8. Valve 开发者 Wiki（https://developer.valvesoftware.com/wiki/Zh/Main_Page）— 商业级引擎工程实践参考
9. GDC / GDC Vault — 行业会谈第一手来源：渲染、架构、优化实践
10.https://docs.vulkan.net.cn/tutorial/latest/03_Drawing_a_triangle/00_Setup/00_Base_code.html
11. 3Blue1Brown《线性代数的本质》— 前置数学阶段的直观辅助，对应第 2 节
12. Ray Tracing in One Weekend（https://raytracing.github.io）— 光线追踪/路径追踪的最佳上手实践读物，对应第 2 节
13. PBRT Book 第 4 版（https://www.pbr-book.org）— 物理渲染权威教材（选读参考），对应第 2 节
14. Unreal Engine 官方文档（https://dev.epicgames.com/documentation/en-us/unreal-engine）— 引擎架构对照与源码阅读的依据，对应第 1/4 节
15. EnTT（https://github.com/skypjack/EnTT）— 业界主流 C++ ECS 库，对象模型演进的对照阅读对象，对应第 4 节
16. CUDA C++ 与 Nsight 官方文档（https://docs.nvidia.com/cuda/）— CUDA 主线与剖析三件套的权威参考，对应第 5 节
17. vulkan-tutorial.com 英文原版（https://vulkan-tutorial.com）— 第 10 条官方中文教程的英文原版，对应第 6 节
18. vkguide.dev — 以自建引擎为目标的 Vulkan 工程化教程，对应第 6 节
19. Vulkan-Samples（https://github.com/KhronosGroup/Vulkan-Samples）与 Sascha Willems examples（https://github.com/SaschaWillems/Vulkan）— 官方与社区两大特性示例库，对应第 6 节
20. VMA（https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator）— 工业标准 GPU 内存管理库，对应第 6 节
21. RenderDoc（https://renderdoc.org）— 跨 API 图形抓帧调试器，对应第 3/5/6 节
