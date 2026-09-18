# 第 7 章设计简报｜大型工程阅读与重构：以 UE 源码与 Core Guidelines 为标尺

目标文件：`tutorial/ch07-large-codebase-reading-and-refactoring.md`。对齐 Stages.md 第 1 节·阶段 7。**本章是第 1 节 C++ 教程的收官章。**

## 章元信息
- 章标题（H1 原文）：`# 第 7 章｜大型工程阅读与重构：以 UE 源码与 Core Guidelines 为标尺`
- 阶段学习目标（原文）：获得在百万行级代码库中定位、理解、修改代码的方法论；以 C++ Core Guidelines 与 Unreal Engine 源码为标尺形成工程审美，能以测试护航小步重构。
- 检验标准（原文，学习目标与自测题 1–3 对位）：
  1. 能向他人讲清所选 UE 调用链每层职责与跨模块/跨线程边界（含 SceneProxy 存在的理由）；
  2. clang-tidy 零告警且至少拦截过一次真实缺陷（留案例记录）；
  3. 重构前后单元测试全绿、渲染器输出逐帧一致；
  4. 能举出 UE 三处「标准库之外自建设施」并解释动机。

## 连续性契约
- 开篇必须兑现 ch06 结尾钩子：三门仪表都盯着自己的代码——本章面对百万行别人的代码。
- **全教程弧线回收**（开篇点名合流）：ch01 调试器=解剖刀、ch02 RAII/所有权=读懂资源归属、ch03 模板与词汇类型=读懂泛型接口、ch04 CMake=读懂工程结构、ch05 并发=读懂线程边界、ch06 性能=读懂热点与数据布局。
- 回收 ch03 3.8 挂账：「错误处理策略见阶段 7」→ 7.8 系统兑现（异常 vs 错误码 vs assert；含 std::expected 前瞻的正式展开）。
- 回收 ch05 5.11：MSVC /analyze 一句带过处「第 7 章的 clang-tidy 接棒」→ 7.9 兑现。
- 回收 ch04：CTest 测试资产与多配置工程 → 7.9 行为保持测试与 clang-tidy 集成的地基。
- 跨节观念指针（各一句）：7.6 渲染管线细节归第 3 节、RHI 归第 6 节；7.7/7.4 与第 4 节引擎架构对照一句。
- 收尾段：**全教程收官**——回望七章弧线（一句话每章），指向下一步（第 3 节 OpenGL 与 P1 实操、UE 源码阅读为长期复利资产，联动根纲领推荐资源第 14 条）。

## 分节大纲（9 节 + 自由小节）

### 7.1 从「读懂自己的代码」到「读懂百万行」
- 规模现实：UE 源码数百万行、模块数百个——通读不可能也无意义。
- 读大代码的三个目的（定位缺陷/学设计/做变更）决定读法；「定点阅读」工作流总览：症状或入口 → 调用链追踪 → 断点验证 → 导览笔记。
- 工具地图：IDE 查找引用/调用层级、grep/ripgrep、编译数据库（compile_commands.json，回收 ch04 检索词：CMAKE_EXPORT_COMPILE_COMMANDS + clangd）、调试器（ch01 1.11 的能力正式上岗）。
- 心态：允许暂时当黑盒，先契约后内脏。

### 7.2 阅读方法论：调用点向上追，断点为地图
- 向上追：从「谁调用了它」出发逐层上溯（Find References / Call Hierarchy）；向下追：深入被调链。
- 静态追不到的三类：虚函数（动态分派）、函数指针/委托/lambda、宏与生成代码——各自用断点+调用栈补（断点停下的地方，调用栈=免费的实时调用图）。
- 调用栈读法回收 ch01 1.11；日志验证（日志断点/临时打印）；反汇编验证的适用时机（怀疑优化器/内联改写了直觉——概念级）。
- 导览笔记模板（对位实践任务 1 的千字笔记）：调用链文字图 + 每层一句话职责 + 数据怎么流 + 我验证过的断点清单 + 未解之谜清单。

### 7.3 UE 源码地图：模块、目录与获取方式
- 获取：GitHub EpicGames/UnrealEngine（注册 UE 账号并关联 GitHub 后可见/可克隆——Stages 实践任务原文；也可下 ZIP 不带历史）；克隆体积提醒（仓库很大，浅克隆/ZIP 更友好）。
- 目录总览：Engine/Source/Runtime（Core / CoreUObject / Engine / Renderer 等）vs Editor vs Developer vs Plugins；一级口诀：**Core=容器/线程/内存等基础设施，CoreUObject=反射与对象系统，Engine=游戏框架（Actor/Component/World），Renderer=渲染线程的世界**。
- 模块=构建+物理隔离单元：`.Build.cs` 声明依赖（Public/Private 依赖的模块层对应物——ch04 target 传递的观念平移一句）；模块依赖单向性。
- 新人阅读的三个好入口与「按模块过滤」的 IDE 用法。

### 7.4 UE 的自建基础设施：TArray、TSharedPtr、FString、FGuid（对位检验标准④）
- 为什么引擎自建轮子（动机清单，正文核心）：①跨平台/跨编译器一致性（引擎支持的编译器与平台远比 std 保证的广）；②内存控制（自定义分配器、OOM 策略）；③反射/序列化/GC 集成（UPROPERTY 才能被 GC 追踪——std 类型天生不可见）；④模块边界的二进制兼容承诺（热加载/DLC）；⑤引擎工具链与调试可视化。
- 对照表（表格）：TArray↔std::vector（接口神似、分配器可换、Slack 语义差异一句）；TSharedPtr↔std::shared_ptr（**且不允许管理 UObject——GC 才是 UObject 的属主，双属主打架**；这是 ch02 2.9 所有权词汇表的引擎级案例）；FString↔std::string（TCHAR 宽字符生态、FName 的全局字典去重一句）；FGuid↔无标准对应（跨进程/跨机器全局唯一 ID 的工程需求）。
- 结论（粗体）：每一处自建都对应一条 std 满足不了的引擎约束——自建不是炫技，是约束使然。

### 7.5 UE 命名约定：一套独立纪律
- 前缀体系：F=普通类、U=UObject 派生、A=Actor、S=Slate 控件、E=枚举、I=接口类、T=模板、G=全局（GEngine）、b=布尔成员（bIsAlive）；PascalCase、成员不带 m_ 前缀（以引擎源码 Source/README 口径为准）。
- 为什么是纪律而非风格洁癖：①U/UCLASS 前缀是反射系统与 UHT 的语法前提；②前缀=类型的「出身证明」，全局可检索；③百万行规模下的一致性=降低认知税。
- 对标准工程的启示（粗体）：**约定本身可以不同，「约定被机械强制」才是重点**——clang-format/.clang-tidy + CI（接 7.9）。

### 7.6 一条绘制调用链的定点解剖：UPrimitiveComponent → FPrimitiveSceneProxy → Renderer（对位检验标准①）
- 游戏线程侧：UPrimitiveComponent（Transform/可见性/材质等游戏态）。
- 跨线程桥：**FPrimitiveSceneProxy 存在的理由**——渲染器整体跑在独立渲染线程、比游戏线程落后一至两帧（Epic 官方文档口径）；渲染线程不能直接读 UObject 游戏态（否则就是 ch05 讲的数据竞争）；于是把渲染所需的**只读快照**拷进 proxy，随命令队列交接；组件销毁时 proxy 的销毁同样排队到渲染线程。
- ENQUEUE_RENDER_COMMAND 宏：把 lambda 排进渲染线程命令队列——ch05 BlockingQueue 的引擎版（回收一句）。
- 渲染线程侧（概念级走一遍）：FScene 收集 proxies → FSceneRenderer 统筹一帧 → 各 Mesh Pass Processor 收集绘制命令 → RHI 抽象层（D3D/Vulkan/Metal 后端；细节归第 3/6 节，观念指针）。
- 跨模块/跨线程边界文字图：Engine 模块游戏态 --（命令队列+快照）--> Renderer 模块渲染态。
- 定点验证建议：在自定义/引擎自带 mesh 组件的 SceneProxy 相关函数下断点跑 PIE 看调用栈；不同 UE 版本细节有演化，「以你手上的版本为准，方法论不变」。
- 本节同时是 ch05（线程边界词汇）的实践演习与第 3 节（渲染管线）的观念预埋。

### 7.7 Core Guidelines：工程审美的成文法
- 是什么：Bjarne Stroustrup 与 Herb Sutter 主持维护的规则库（isocpp.github.io/CppCoreGuidelines），编号按主题族（R 资源/ES 表达式与语句/Per 性能/T 模板/CP 并发…），每条带理由与强制方式，配 GSL 支持库。
- 怎么用（粗体立场）：**不当法条背诵，当代码评审的检查表与「为什么」的引用源**。
- 四族选读（内容为准，条号以线上版为准的声明放段首）：
  - R 族（资源）：R.11「避免显式调用 new/delete」（已核实条号；MSVC 对应 C26409）——ch02 全章世界观的成文版；再选 1–2 条资源参数传递类规则（内容描述）。
  - ES 族（表达式与语句）：ES.60「new/delete 只出现在资源管理层」（已核实）；边界与下标安全族（span/gsl::index 与 ch03 3.8 的呼应）。
  - Per 族（性能）：先测量后优化、数据布局优先于微操（与 ch06 的呼应，内容描述不给条号）。
  - T 族（模板）：概念约束优先于裸 typename（ch03 3.11 的成文版，内容描述）。
  - CP 族（并发）一句带过（ch05 已覆盖主干，第 7 章不展开）。
- GSL 一句话：gsl::span/index/not_null——std 词汇类型的姊妹篇（ch03 3.8 呼应）。

### 7.8 错误处理策略：异常、错误码与 assert 的边界（兑现 ch03 3.8 挂账）
- 三件工具各管一段（先立分诊表）：**assert/check=程序员 bug 专用**（不该发生的事，发布版可编译掉）；**错误码/expected=可预期的失败**（文件不存在、资产损坏——调用方必须处理的「正常业务」）；**异常=不可恢复但可上抛的资源/构造失败**（且栈展开靠 RAII 清理——ch02 的世界观在异常路径的兑现）。
- std::expected 正式展开（ch03 只前瞻）：语义（值或错误，无分配无异常）、与 optional/pair<bool,T> 的差别、返回值链式 and_then 一小段示例（完整可编译，标注 C++23、MSVC 支持情况以文档为准）。
- 游戏引擎普遍禁异常的原因（对位 Stages 原文，理由清单）：①栈展开的运行时代价与二进制膨胀（unwind 表、landing pad 干扰优化与内联）；②实时帧预算下「一帧内 unwind 的不可预算性」（回收 ch06 世界观）；③跨 DLL/模块边界的异常语义泥潭；④图形/系统 API 全是 C 接口边界；⑤UE 明确走 -fno-exceptions 路线，容器越界用 check 家族替代异常。
- UE 的分级 assert（工业级设计，值得学）：check（致命崩溃=报告 bug）、ensure（报一次但继续跑）、verify（表达式在发布版也要求值）。
- 禁异常引擎里的构造失败替代：两段初始化/工厂函数返回错误码或 expected（工业惯用法一句）。

### 7.9 重构护栏：clang-tidy、clang-format 与小步提交（对位检验标准②③）
- 前提：**重构=行为保持的结构改动**（Fowler 定义），与优化、修 bug 分开提交——回收 ch06（优化要四栏数据记录）。
- 护栏一：行为保持测试先行——对要动的模块先补测试（ch04 CTest 资产复用）；渲染输出「逐帧一致」的判定：固定种子+固定输入跑 N 帧对输出做哈希（概念级，为 P1 埋点；对位检验标准③）。
- 护栏二：clang-format——风格全自动，零摩擦零讨论；.clang-format 进仓库根。
- 护栏三：clang-tidy——静态检查基线：检查组（readability/modernize/bugprone/performance/cppcoreguidelines）概念介绍；.clang-tidy 文件+精选白名单起步、渐进收紧；CMake 集成 `CMAKE_CXX_CLANG_TIDY`（ch04 工程再复用）；与 MSVC /analyze 的互补（跨平台统一、可进 CI；回收 ch05 5.11 的接棒承诺）；「至少拦截一次真实缺陷」的工作习惯（案例记录模板：检查名/代码片段/为什么是真缺陷）。
- 护栏四：小步提交——每步可编译、可测试、可回滚；commit=一个意图；重构与功能混提是大忌。
- 工作流闭环：红→绿→重构；重构与优化两类改动不混。

## 自由小节（深入专题）：UE 前缀背后的反射系统一瞥
- UCLASS/UPROPERTY/UFUNCTION 宏展开后是什么：UHT（UnrealHeaderTool）在正常编译前扫描头文件，生成 .generated.h/.cpp——类型注册表、属性表、GC 引用网络、蓝图暴露、网络复制。
- 为什么 UE 代码「长得不像标准 C++」：宏不是装饰，是代码生成指令；读引擎代码遇到看不懂的宏→查对应 .generated 文件。
- GC 如何靠反射工作：标记-清扫遍历 UPROPERTY 引用网（呼应 ch02：shared_ptr 在 UObject 域失灵、GC 才是属主的机制原因）。
- 同型物对照一句：Qt 的 moc/Q_OBJECT。
- 篇幅 120–180 行，纯文字+少量示意代码。

## 实践任务（3 个核心任务；E3 声明放开头，UE 任务给独立替代版）
- 任务 1：UE 源码定点阅读 + 千字导览笔记（对位检验标准①）：沿 7.6 路线读通 UPrimitiveComponent → FPrimitiveSceneProxy → Renderer 绘制路径，断点验证至少 3 处；笔记按 7.2 模板。**独立替代版（无 UE 环境时，验收口径一致）**：对 ch04 mini_engine + ch05 异步加载做同规格「跨线程调用链导览」（主线程提交→工作线程加载→结果回主线程），标出每层职责与线程边界。可选 P1 衔接：P1 到 M4 后对真渲染器做同款导览（P1 增量「clang-tidy 与重构护栏」的前置阅读）。
- 任务 2：clang-tidy 基线（对位检验标准②）：给 mini_engine（或 P1 工程）建 .clang-tidy 起步集合（readability/modernize/bugprone/performance 精选），CMAKE_CXX_CLANG_TIDY 接入，清零存量告警，记录「至少拦截一次真实缺陷」的案例。验收：基线文件+零告警证据+案例记录（检查名/代码/为什么是真缺陷）。工具可用性注：clang-tidy 随 LLVM/VS 工具链分发，以官方文档为准。
- 任务 3：小步重构实战（对位检验标准③）：自选一处（抽取函数/消除全局状态/消灭魔法数，建议在 mini_engine）；先补行为保持测试（复用 ch04 test 目标）→ 红→绿→重构小循环 ≥3 步提交 → 测试全绿；有渲染输出则加逐帧一致性证明（固定种子+哈希）。验收：提交序列（日志/截图）+前后测试全绿+一段「为什么这不是优化」（区分重构与优化，回收 ch06 四栏）。

## 自测题（8 题，第 1–3 题对位检验标准）
①口述：SceneProxy 存在的理由——渲染线程为什么不能直接读 UObject（对位检验标准①）。②列举：UE 三处自建设施+动机（对位检验标准④）。③方法论：静态追调用链的三类「追不到」与各自的补法。④场景分诊：资产损坏/程序员越界 bug/构造函数失败/跨模块初始化——四场景选 错误码或 expected/check/异常（禁异常引擎语境下给替代）并说理由。⑤概念：Core Guidelines 的正确使用姿势 + GSL 是什么。⑥护栏匹配：四护栏（测试/clang-format/clang-tidy/小步提交）各挡什么风险。⑦对比：clang-tidy 与编译器警告的关系与差异。⑧模块地图：给 5 个 UE 符号（如 FVector、AActor、FPrimitiveSceneProxy、UPrimitiveComponent、FSceneRenderer），判断所属模块与线程侧。

## 常见误区（8 条，症状/病根/纠偏）
①从 main/入口顺读百万行（对位 Stages 原文「通读必然迷路」）。②把 UE 自建轮子当「落后」（先问动机再评判——7.4 的动机清单）。③抄引擎风格不知所以然（前缀是反射+UHT 强制的；该学的是「约定被机械强制」）。④Core Guidelines 当法条背诵或圣旨盲从（它是评审检查表与论证引用源）。⑤异常/错误码教条战（按错误类别分诊，不问语境站队）。⑥重构顺手优化/修 bug（混合提交毁回滚粒度；重构行为不变、优化要四栏数据）。⑦clang-tidy 全开检查（告警疲劳→全员忽略；精选渐进）。⑧读源码不写导览笔记（三天后重读全忘；断点清单+调用链图是复利资产）。

## 延伸资源（7–9 条，带确定链接）
- Unreal Engine 官方文档（dev.epicgames.com/documentation——Stages 清单第 14 条）。
- EpicGames/UnrealEngine GitHub 仓库（注册关联后可克隆——任务 1 素材）。
- C++ Core Guidelines（isocpp.github.io/CppCoreGuidelines——本章主标尺）。
- clang-tidy 文档（clang.llvm.org/extra/clang-tidy/）。
- clang-format 文档（clang.llvm.org/docs/ClangFormat.html）。
- 《重构：改善既有代码的设计》第 2 版（Martin Fowler）。
- 《修改代码的艺术》（Working Effectively with Legacy Code，Michael Feathers）。
- Epic 官方「Threading the Renderer」相关文档/直播（检索词：Unreal rendering thread documentation）。

## 覆盖清单（写作自查，逐条对应 Stages 阶段 7 知识点）
1. UE 源码地图（模块结构）→ 7.3
2. TArray/TSharedPtr/FGuid 自建设施对照与动机 → 7.4
3. UE 命名约定为独立纪律 → 7.5
4. Core Guidelines 选读（R/ES/Per/T）→ 7.7
5. 错误处理策略边界（异常/错误码/assert、引擎禁异常原因）→ 7.8
6. 阅读方法论（调用点向上追、断点为地图、日志/反汇编验证）→ 7.1/7.2
7. 重构护栏（clang-tidy/clang-format、测试先行、小步提交）→ 7.9
8. UE 调用链定点阅读（UPrimitiveComponent→SceneProxy→Renderer）→ 7.6
