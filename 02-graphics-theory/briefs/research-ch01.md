# 第 1 章调研素材（前置数学：线性代数与微积分核心）

> 主代理网络调研所得事实清单，供简报设计（glm-5.3）引用。来源域名均为可靠出处；简报与正文引用时以「内容描述」为主，链接只用于延伸资源节。

## 一、点乘与叉乘的图形学用途（调研确认）

- 点乘：a·b = |a||b|cosθ。图形学四大用途（多源交叉确认：3D Math Primer、Scratchapixel、GAMES101 L2、mysimulator.uk）：
  1. 求两向量夹角（cosθ = a·b/(|a||b|)）；判断「同向/垂直/反向」（点乘正/零/负）。
  2. 投影：a 在 b 上的投影 = (a·b̂)b̂；可把向量分解为平行+垂直两分量。
  3. Lambert 漫反射光照：N·L = cosθ，光照贡献直接取点乘——「点乘为何出现在 Lambert」的标准答案。
  4. 背面剔除：N·V < 0 判定背面、跳过三角形（dot 决定 forward/backward）。
- 叉乘：a×b 垂直于两者、方向右手定则、|a×b| = |a||b|sinθ。用途：
  1. 三角形法线：N = normalize((B−A)×(C−A))。
  2. 判断左右关系（GAMES101：cross 决定 left/right、inside/outside——点在三角形内的同号判定）。
  3. 构建局部坐标系（三个互相垂直的轴）。
- GAMES101 L2 的讲法（可借鉴的教学顺序）：向量 = 方向+长度、无绝对起点 → 归一化 → 加法（平行四边形/三角形法则）→ 点乘（角度/分解/前后/投影）→ 叉乘（左右/内外）→ 矩阵。**向量默认列向量**。
- 3D Math Primer（gamemath.com，Fletcher Dunn，全书免费在线）第 2 章 Vectors 即按 dot/cross 组织，第 7-10 章 lighting/matrices；Lengyel《Mathematics for 3D Game Programming and Computer Graphics》第 2-4 章为向量/矩阵/变换标准参考。

## 二、矩阵 = 线性变换；齐次坐标为何存在（调研确认）

- 线性变换两条件：F(a+b)=F(a)+F(b)、F(ca)=cF(a)；矩阵与线性变换一一对应，矩阵乘法 = 变换复合（CS184 L4、CSE167、Scratchapixel）。
- **平移不是线性变换**：违反原点保持（f(0)=a≠0）（UCSD CSE167 讲义）。仿射变换 = 线性变换 + 平移。
- 齐次坐标解法（多源一致）：2D 点 =(x,y,1)、2D 向量 =(x,y,0)（UC Berkeley CS184 讲义）；点 w=1、方向 w=0，平移只作用于点；4×4 矩阵统一表达全部变换，复合 = 连乘。NVIDIA Cg Tutorial 第 4 章：w 是除数（透视除法伏笔，本章不展开、留第 2 章变换链）。
- 正交矩阵：RᵀR = I（列向量组标准正交），旋转矩阵是正交的——性质测试的直接出处（Stages 检验标准原文「旋转矩阵正交性 RᵀR=I」）。
- CS184 讲义强调：变换顺序不可交换（先旋转后平移 ≠ 先平移后旋转）——矩阵乘法次序的教学点。

## 三、行/列向量与行/列主序约定（调研确认，易混淆重灾区）

- 两套约定（mindcontrol.org hplus / toqoz.fyi / Scratchapixel / tomhultonharrop.com 多源）：
  - **数学教材与 OpenGL 惯用**：列向量、右乘（Mv）→ 变换从右往左读。
  - **DirectX 历史惯用**：行向量、左乘（vM）→ 从左往右读；两版矩阵互为转置。
- **存储主序（row-major vs column-major）与乘法方向是两个独立概念**：OpenGL 列主序存储 + 列向量约定；D3D 传统上行主序存储 + 行向量约定——但完全存在「列向量 + 行主序存储」等组合（Pavel Šmejkal 文章专门澄清）。GLM（列主序、列向量）vs DirectXMath（行主序、行向量）。
- 教学结论（本章可直接用）：**选定一套约定并全程一致，比争论哪套对更重要**；读别人代码先看它是 vM 还是 Mv。GAMES101 用列向量右乘。
- 左右手系：右手系（OpenGL/GLM/GAMES101 默认）vs 左手系（DirectX 传统）；叉乘方向依赖手性；跨 API 移植时 Z 轴方向/深度范围差异是坑（详细展开归后续章与第 3/6 节，本章只立概念）。

## 四、微积分与概率的图形学消费点（调研确认）

- 导数/梯度 → 高度场求法线：h(x,y) 的法线 ∝ normalize(−∂h/∂x, −∂h/∂y, 1)（DEV.to normal map 文章、CMU 15-462 L2 vector calculus preview：directional derivative/gradient/Laplacian/Hessian 清单）。法线贴图正是逐像素存梯度信息（概念点到，切线空间归第 4 章）。
- 定积分 = 连续求和 → 渲染方程就是积分（TU Wien「Mathematical Basics of Monte Carlo Rendering Algorithms」：蒙特卡洛为处理渲染中的复杂积分/求和而生；期望/方差/大数定律是其数学基础）。Springer《Calculus for Computer Graphics》（John Vince）强调微积分与几何的联系面向动画/游戏读者。
- 概率直觉（为第 7 阶段路径追踪铺路，本章概念级）：期望 = 长期平均、方差 = 离散程度、大数定律 = 样本均值收敛期望（噪点随样本数收敛的直觉伏笔，一句前向指针即可）。

## 五、教学资源定位（延伸资源候选，均已核实存在）

- GAMES101（sites.cs.ucsb.edu/~lingqi/，B 站完整视频）——本节主线推荐，L2-L3 与本章对位；社区笔记多（xiao-h.com/assets/GAMES101.pdf）。
- 3Blue1Brown《线性代数的本质》（Essence of Linear Algebra）——Stages 原文指定「先建立画面感」。
- Scratchapixel（scratchapixel.com）geometry 课：vectors、matrices、transforming points and vectors、row-major vs column-major 专文。
- 3D Math Primer for Graphics and Game Development, 2nd ed（gamemath.com 全书免费）。
- Eric Lengyel《Mathematics for 3D Game Programming and Computer Graphics》4th ed。
- CMU 15-462 lecture 列表（15462.courses.cs.cmu.edu）vector calculus review 可作进阶对照。

## 六、与 01-cpp 的衔接点（后向引用可用）

- 学习者刚完成第 1 节：C++20 熟练、有 rule of zero/模板/单元测试/CMake/性能意识底子——数学库代码可按 C++20 写（constexpr、concepts 约束标量类型可顺带用）。
- 01-cpp 阶段 3 实践曾要求「为第 2 节数学库逐类标注零/三/五法则」——本章数学库落地时后向引用兑现（Vec/Mat 应 rule of zero）。
- 单元测试：01-cpp ch04 讲过 CTest；数学库性质测试（RᵀR=I、叉乘垂直性、|a×b|=|a||b|sinθ）可用最小 assert 式自测或接 CTest（简报裁量，勿强制引入工程复杂度）。
- 性能意识（ch06）：数学库设计一句对齐（如 Vec 运算按值返回、无堆分配）即可，不展开。

## 七、Stages.md 阶段 1 覆盖基线（原文要点，简报必须全覆盖）

- 核心知识点：线代——点乘（投影/夹角/光照贡献）与叉乘（定向/法线/三角形朝向）；矩阵=线性变换、乘法=复合、正交矩阵性质；齐次坐标与仿射变换（平移为何升维）；行/列向量与左右手系约定差异。微积分——导数/梯度（高度场求法线）、定积分=连续求和（渲染方程铺路）。概率直觉——期望、方差、大数定律（蒙特卡洛铺路）。建议先 3B1B 建立画面感再系统过。
- 实践任务：手写 C++ 数学库（Vec2/3/4、Mat4 及常用运算），每函数一句几何意义注释；单元测试验证 RᵀR=I、叉乘垂直性。
- 检验标准：①不查资料手推旋转/缩放矩阵、能做基础求导积分题；②口述点乘在 Lambert、叉乘在背面剔除；③数学库通过全部性质测试。
- 误区对位：只看视频不写作业 / 第一遍就用 GLM 代替手写 / 前置数学贪多求全迟迟不进正文 / 全程跳过推导只记结论。
