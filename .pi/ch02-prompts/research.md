你是「02-graphics-theory」领域教程的第 2 章调研代理。工作目录：D:/workplace/learningPlace（所有相对路径以此为根）。

# 任务

为教程第 2 章《变换与投影：一个顶点的完整旅程》做网络调研，把核实过的事实清单写入 `briefs/research-ch02.md`（用 write 工具落盘，UTF-8，中文）。调研本身必须用 web_search 工具真实检索，多轮、多关键词、多源交叉验证；禁止凭记忆编造事实或链接。

# 先读这些文件（理解体例、边界与覆盖基线）

1. `Stages.md` 第 2 节「阶段 2｜变换与投影：一个顶点的完整旅程」——本章知识覆盖基线（原文要点：MVP 各变换职责；lookAt 视图矩阵构造；正交/透视投影矩阵完整推导；透视除法（w 分量来源）与视口变换；法线变换用逆转置 (M⁻¹)ᵀ 的原因；各 API 的 NDC/深度范围差异——第 3/6 节常见坑点的理论出处。实践任务：在阶段 1 数学库上实现 lookAt/ortho/perspective/viewport；手工计算一个顶点的完整 MVP 链路并与标准结果对照。检验标准：白板推导透视投影矩阵；解释「w 如何带来近大远小」；给定「看不见/拉伸/镜像」等症状能点名出错环节）。
2. `briefs/research-ch01.md` —— 第 1 章调研体例模板，你产出的文件必须沿用同样的结构与口吻。
3. `briefs/ch01-brief.md` —— 第 1 章简报，了解调研素材如何被消费，以及 ch01 留给第 2 章的接口清单（透视除法、lookAt/ortho/perspective/viewport、法线逆转置、相机正交基）。
4. 根 `Topic.md` 第 2 节——领域边界（本节全程纯 CPU、不绑定任何图形 API；API 实操归第 3 节，Vulkan 归第 6 节）。

# 调研主题清单（每个主题至少一轮检索，关键主题多轮换词）

1. MVP 变换链职责划分：local/model space、world space、view space、clip space、NDC、screen space 各空间的定义与先后次序（LearnOpenGL「Coordinate Systems」是标准术语源；Scratchapixel、GAMES101 L4 交叉验证）。
2. lookAt 视图矩阵构造推导：相机位置 eye、目标 target、上向量 up → forward/right/up 正交基 → 旋转+平移组合成视图矩阵（多源：LearnOpenGL、songho.ca、Scratchapixel、3D Math Primer）。
3. 正交投影矩阵完整推导：左右/上下/近远 → 映射到 NDC 的逐分量推导（GAMES101 L4 给了标准推导；songho.ca、Scratchapixel、glMatrix 源码口径交叉验证）。
4. 透视投影矩阵完整推导：视锥体、相似三角形得 x/y、z 的特殊处理、−2zn/(f−n)（或等价形式）的来历；「投影矩阵不是可逆变换、w 存深度信息」的经典论证（GAMES101 L4、Scratchapixel、www.opengl.org/wiki、乐长炯?——以实际检索到的可靠来源为准）。
5. 透视除法与 w 分量：clip space (x,y,z,w) ÷ w → NDC；w 的几何意义（原深度坐标）；w 如何带来近大远小（1/z 插值的正确性伏笔——透视校正插值归第 3 章，只埋一句）。
6. 视口变换：NDC → 屏幕像素坐标的线性映射（glViewport 语义、视口矩阵形式）。
7. 法线变换为何用逆转置 (M⁻¹)ᵀ：非均匀缩放下原矩阵变换法线不再垂直于表面；数学论证多源核实（LearnOpenGL、Scratchapixel、Fabian Giesen 的文章、 forums 数学推导）。
8. 各 API 的 NDC/深度范围差异：OpenGL x/y/z ∈ [−1,1] 且 z ∈ [−1,1]；Direct3D/Metal z ∈ [0,1]；Vulkan y 翻转 + z ∈ [0,1]；行/列主序与矩阵约定差异——这些是第 3/6 节坑点的理论出处（检索 Khronos wiki、Vulkan spec、Microsoft docs、Matthias Moulin? / 知名博客的对照表，来源要可靠）。
9. 深度缓冲与非线性深度的第一印象（z 非线性、近处精度高——LearnOpenGL depth 一章；深入归第 3 章光栅化，本章点到即止，调研仍收集以便简报裁量）。
10. 教学资源核实（延伸资源候选）：GAMES101 Lecture 04（Transformation Cont.）与 Lecture 05；LearnOpenGL Coordinate Systems / Depth 章节；Scratchapixel 对应章节；3D Math Primer for Graphics and Game Development（gamemath.com 免费在线）变换章节；songho.ca「OpenGL Transformation」；PBRT 或 Lengyel 对应章节；每个链接要核实可达/存在（可用 web_search 摘要或 fetch 验证），不可达的剔除。

# 输出文件结构（沿 research-ch01.md 体例）

- 开头引语块（`> 主代理网络调研所得事实清单，供简报设计引用……`，说明来源域名可靠、正文引用以内容描述为主）。
- 按上述调研主题分节（一、二、三……），每条事实注明来源域名，多源交叉确认的显式标注「多源交叉确认：A、B、C」。
- 单独一节「教学资源定位（延伸资源候选，均已核实存在）」。
- 单独一节「与 ch01 的衔接点」（后向引用可用素材：ch01 数学库 Vec2/3/4、Mat4 的现有 API——读 `tutorial/ch01-prerequisite-math.md` 的 1.10 节确认 API 清单后写进衔接点；第 2 章要在其上补 lookAt/ortho/perspective/viewport）。
- 末节「Stages.md 阶段 2 覆盖基线（原文要点，简报必须全覆盖）」。

# 硬性要求

- 全程中文；术语首次出现给中英对照（如「透视除法（perspective divide）」）。
- 事实必须来自真实检索结果；链接只写核实过的；检索不到共识的争议点如实标注「存在分歧」并给出两方说法。
- 文件写完后，在最终回复里给出：① 已写入路径；② 每个调研主题 1–2 句的关键结论摘要；③ 核实过/剔除的链接清单。
