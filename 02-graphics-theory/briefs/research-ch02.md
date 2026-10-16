# 第 2 章调研素材（变换与投影：一个顶点的完整旅程）

> 主代理网络调研所得事实清单，供简报设计引用。来源域名均为可靠出处；简报与正文引用时以「内容描述」为主，链接只用于延伸资源节。全程多轮、多关键词、多源交叉验证；检索不到共识的争议点如实标注「存在分歧」。

## 一、MVP 变换链与各空间的定义、次序（调研确认）

- 标准术语源 LearnOpenGL「Coordinate Systems」：顶点在成为屏幕像素前依次经过 local space（局部/模型空间）→ world space（世界空间）→ view space（观察/相机空间）→ clip space（裁剪空间）→ NDC（normalized device coordinates，标准化设备坐标）→ screen space（屏幕空间）。过渡坐标系的意义：某些操作在特定空间里做更方便（learnopengl.com；中文版 learnopengl-cn.github.io「坐标系统」页同步核实）。
- LearnOpenGL 口径的逐段职责：model 把局部坐标摆到世界；view 把世界坐标变换为「以相机为原点」的观察坐标；projection 把观察坐标压进裁剪视体；顶点着色器必须输出裁剪坐标，OpenGL 期待透视除法后 NDC 的 x/y/z ∈ [−1,1]、超界不可见；NDC 再经 glViewport 变到屏幕坐标（learnopengl.com）。
- GAMES101 L4「拍照三步」讲法（MVP 的经典记忆结构）：「找好地方摆人（Model）→ 找好角度放相机（View）→ 按快门（Projection）」——三步合称 MVP 变换（sites.cs.ucsb.edu GAMES101_Lecture_04.pdf；社区笔记 xiao-h.com/assets/GAMES101.pdf 交叉确认）。
- songho.ca「OpenGL Transformation」给出同一链路的 API 视角：object coords →(Model)→ eye coords →(Projection)→ clip coords →(÷w)→ NDC →(viewport)→ window coords；固定管线把 Model 与 View 合并在 GL_MODELVIEW 一个矩阵里（songho.ca/opengl/gl_transform.html）。Khronos OpenGL Wiki「Coordinate Transformations」与「Vertex Post-Processing」页给出同一管线（模型坐标→世界→眼→裁剪→透视除法→视口+深度范围），并明确裁剪发生在裁剪空间、以 −w ≤ x,y,z ≤ w 为判据（wikis.khronos.org/opengl）。
- Scratchapixel 从光栅化视角复述同一次序：模型在 object space 定义、经 object-to-world 矩阵进 world space；光栅化第一步是把顶点投到画布、第二步光栅化成像素（scratchapixel.com）。
- 教学要点（多源交叉确认：learnopengl.com、songho.ca、wikis.khronos.org、cs.cornell.edu）：MVP 连乘次序（列向量右乘）P·V·M·v——离 v 最近的是 model；「先裁剪后除法」（除 w 之前用 ±w 做裁剪边界）是硬件实际行为，也是「看不见」症状的判读地图。

## 二、lookAt 视图矩阵的构造推导（调研确认）

- 统一构造路线（多源交叉确认：christianermann.dev、how-to-3d.twodee.org、geertarien.com、songho.ca、learnopengl.com、swarthmore cs40 讲义）：给定相机位置 eye、目标点 target、世界向上向量 up 三参数 → 构造相机正交基（forward/right/up 三个互相垂直的单位向量）→ 视图矩阵 = 旋转的转置 + 平移的负：基向量按行摆放（列向量右乘约定下），平移列 = −(各基·eye)。
- 几何直觉：视图变换 = 把世界「反向搬进相机坐标系」——相机固定在原点、朝 −Z（OpenGL 眼空间约定）；移相机 = 用相机变换的逆去移整个场景（songho.ca/opengl/gl_camera.html；wikis.khronos.org/opengl「Viewing and Transformations」：gluLookAt 计算的就是逆相机变换；opengl.org 旧版 FAQ 第 8 节同口径）。
- 正交基构造顺序（LearnOpenGL 自定义 lookAt 源码，learnopengl.com/code_viewer camera-exercise2）：zaxis = normalize(position − target)（相机 z 基指向相机后方——右手系相机看 −Z）；xaxis = normalize(cross(worldUp, zaxis))；yaxis = cross(zaxis, xaxis)。世界 up 只用来「定向」，非正交时先出 x 再回推 y，天然正交化。
- 分歧/口径注记（如实标注）：forward 轴的符号约定有两派——OpenGL/GLM 右手系把相机 −Z 定为前方（z 基 = eye − target，LearnOpenGL 口径）；D3D 左手系把 +Z 定为前方（z 基 = target − eye，geertarien.com 文中源码即此口径）。songho.ca 的 gluLookAt 推导取 f = normalize(center − eye) 再在矩阵第三行取负，殊途同归。**两派矩阵互为镜像差异只在符号，教学选定一套（右手系、看 −Z）并声明即可**。
- GAMES101 口径补充（sites.cs.ucsb.edu GAMES101_Lecture_04.pdf）：视图矩阵记为 R_view·T_view，其中旋转部分由正交矩阵「逆=转置」性质直接写出——这是本章复用 ch01「RᵀR=I」性质测试的最佳落点。

## 三、正交投影矩阵的完整推导（调研确认）

- GAMES101 标准推导（sites.cs.ucsb.edu GAMES101_Lecture_04.pdf）：直观版——相机在原点看 −Z、up 朝 Y，丢掉 z 再把矩形平移缩放到 [−1,1]²；正式版——把长方体 [l,r]×[b,t]×[f,n]（**注意相机看 −Z，n、f 取负值且 n > f**）映射到规范立方体（canonical cube）[−1,1]³：先平移（中心到原点）后缩放。两步合并：
  M_ortho = S(2/(r−l), 2/(t−b), 2/(n−f)) · T(−(r+l)/2, −(t+b)/2, −(n+f)/2)。
- Cornell CS4620 09viewing 讲义交叉确认同一矩阵，并明确「view volume 沿 −z，f 是比 n 更负的数，故 n > f」；UCSD CSE167 viewing 讲义补充「OpenGL glOrtho 惯例：传正的距离、内部取负」——**「n/f 正负号约定」是推导正交矩阵时第一颗雷，多源一致确认**（cs.cornell.edu、cseweb.ucsd.edu、xiao-h.com 笔记）。
- songho.ca「OpenGL Projection Matrix」给出 glOrtho 口径的最终矩阵（正距离参数版，z 行带负号），与上述 GAMES101 版本代数等价（songho.ca/opengl/gl_projectionmatrix.html）。
- 分量逐项推导口径（jcsites.juniata.edu CS330 讲义）：x_ndc = 2·x/(r−l) − (r+l)/(r−l)，y、z 同构——「先缩放到长度 2、再平移到居中」的逐轴映射是白板推导的可复用模板（多源交叉确认：cs.cornell.edu、cseweb.ucsd.edu、jcsites.juniata.edu）。
- 工程口径交叉验证：GLM 提供 orthoRH_NO / orthoRH_ZO（NO = NDC z ∈ [−1,1] OpenGL 口径，ZO = [0,1] D3D/Vulkan 口径）；glMatrix v4 同样拆 orthoNO / orthoZO（glm.g-truc.net、glmatrix.net 文档核实存在）——正交投影的「z 范围双口径」在主流数学库里是显式 API 分叉，佐证第八节 API 差异主题。

## 四、透视投影矩阵的完整推导（调研确认）

- GAMES101「挤压」推导（多源交叉确认：sites.cs.ucsb.edu GAMES101_Lecture_04.pdf 原始出处；xiao-h.com、caterpillarstudygroup.github.io、blog.mickeymiao.cn、腾讯云开发者社区等多份社区笔记复述一致）：
  1. 相似三角形：视锥内点 (x,y,z) 在近平面上的像为 x′ = n/z·x、y′ = n/z·y；
  2. 齐次化技巧：(x,y,z,1) → (nx, ny, 未知, z)——前三行解出、第四行设 (0,0,1,0) 让 w 承载 z；
  3. 用三个不变量解第三行：近平面上点不变、远平面上点的 z 不变、远平面中心点不变 → 得挤压矩阵
     M_persp→ortho = [n 0 0 0; 0 n 0 0; 0 0 n+f −fn; 0 0 1 0]（GAMES101 z 取负值约定）；
  4. M_persp = M_ortho · M_persp→ortho（先压成长方体、再做正交投影）。
- 社区笔记共识补充：挤压后「近平面与远平面之间的点 z 会更靠远/近平面会变」（不是线性不动）——这是 z 非线性的代数根源之一，与第九节深度非线性呼应（caterpillarstudygroup.github.io、腾讯云开发者社区、iewug.github.io）。
- OpenGL 口径最终矩阵（多源交叉确认：songho.ca、scratchapixel.com、cs.utexas.edu、dbarbs.net、jeesunkim.com、thorsten.suckow-homberg.de）：以正距离 n、f 与近平面矩形 l/r/b/t 表达，第三行 (0, 0, −(f+n)/(f−n), −2fn/(f−n))、第四行 (0, 0, −1, 0)——**−2fn/(f−n) 一项即「z 的特殊处理」的产物**，其来源是「近平面映到 z_ndc=−1、远平面映到 +1」两个约束的联立解。songho.ca 页面有完整逐步图解。
- fovY/aspect 参数化（GAMES101 L05 开头，sites.cs.ucsb.edu GAMES101_Lecture_05.pdf）：t = |n|·tan(fovY/2)、r = aspect·t，再代入 l/r/b/t 版矩阵——「vertical fov + aspect ratio」是现代 API（glm::perspective、glMatrix perspective）的通用参数。
- 工程口径直接核实（github.com/toji/gl-matrix src/mat4.ts）：f = 1/tan(fovy/2)、out[0] = f/aspect；NO 版 out[10] = (far+near)·nf、out[11] = −1；ZO 版按 [0,1] 改写。GLM perspectiveRH_NO/RH_ZO 注释明说 ZO 版「near/far 对应 NDC z 的 0 和 +1（Direct3D 裁剪体定义）」（github.com/g-truc/glm、glm.g-truc.net）。与教学推导逐项对得上。
- 「投影矩阵不可逆/w 存深度」论证的精确口径（调研后需谨慎表述，两说并存、需区分）：教科书意义的「投影」（把 z 压平丢深度的真投影矩阵）行列式为 0、不可逆（cs.um.edu.mt 讲义：flatten z 的 Po 丢失全部深度信息、det=0；Stack Overflow「Decomposition of glFrustum」确认透视变换是 projective 而非 affine、不保平行）；而**图形 API 的 4×4「投影矩阵」本身是可逆的**（这正是把深度编码进 z/w 保住信息的动机，thorsten.suckow-homberg.de 推导文与 GAMES101 挤压推导都隐含此性质；GAMES101 亦提到 M_persp→ortho 反向可做 far→near）——**真正不可逆的分水岭是透视除法：除完丢掉 w 后无法从 NDC 恢复深度比例**。简报表述建议：「压扁投影不可逆，API 投影矩阵可逆但除法后不可逆」。

## 五、透视除法（perspective divide）与 w 分量（调研确认）

- 定义与位置（多源交叉确认：cs.utexas.edu、mauriciopoppe.com、webgl.brown37.net、carmencincotti.com、songho.ca）：顶点经投影矩阵得裁剪坐标 (x,y,z,w)，GPU 随后除以 w 得 NDC：(x/w, y/w, z/w)。裁剪在除法之前、以 −w ≤ x,y,z ≤ w 为界（wikis.khronos.org/opengl「Vertex Post-Processing」；mauriciopoppe.com：裁剪用 ±w 而非 ±1 正是为了在除法前完成）。
- w 的几何意义（多源交叉确认：mauriciopoppe.com、deepraster.com、scratchapixel.com、courses.grainger.illinois.edu）：透视投影下 w_clip = −z_view（相机空间深度的负值；右手系相机前方 z_view < 0 故 w > 0）——「w 是视空间深度的一份拷贝」；正交投影 w 恒为 1、除法是空操作（mauriciopoppe.com 原句：the division by −z is what makes distant objects appear smaller, and the w-coordinate carries that division）。
- 「w 如何带来近大远小」的标准答案（多源交叉确认：mauriciopoppe.com、cs.utexas.edu、deepraster.com、learnopengl.com）：除以 w（即除以深度）后，同一物体的屏幕坐标随距离 1/z 缩放——远处 w 大、商小、显得小；平行线向灭点会聚也是同一除法的结果。
- ch01 伏笔的正式收束点：NVIDIA Cg Tutorial「w 是除数」（ch01 1.6 引用过）在本章变成可计算事实——齐次坐标从「区分点/向量」的第二语义升级为「透视的载体」。
- 前向指针（只埋一句）：除以 w 后屏幕空间线性插值 ≠ 视空间线性——透视校正插值（perspective-correct interpolation）问题留给第 3 章（CMU 15-462 L5「Perspective-Incorrect Interpolation」章节确认该主题归属光栅化；本章只埋「1/z 插值才正确」一句）。

## 六、视口变换（viewport transformation）（调研确认）

- 语义（docs.gl glViewport 规格 + songho.ca/opengl/gl_viewport.html）：glViewport(x, y, w, h) 定义 NDC → 窗口坐标的仿射映射，(x_ndc, y_ndc) = (−1,−1) 映到视口左下角、(+1,+1) 映到右上角；窗口坐标保留 z 供深度测试，z 由 glDepthRange 另行映射（默认 [0,1]）。
- 矩阵形式（多源交叉确认：cs.cornell.edu 09viewing、GAMES101 L05（sites.cs.ucsb.edu GAMES101_Lecture_05.pdf）、mauriciopoppe.com、sibaku.github.io）：把 [−1,1]² 线性映射到 [x, x+width]×[y, y+height]——缩放 (width/2, height/2)、平移 (x + width/2, y + height/2)，z 轴顺带映射深度范围；LearnOpenGL 书内例子「NDC (−0.5, 0.5) 经 800×600 视口变到 (200, 450)」可作数值例题（learnopengl.com/book 预览 PDF）。
- 口径差异注记（如实标注）：多数讲义（Cornell、GAMES101、mauriciopoppe）用 width/2 无偏移版；「像素中心 vs 像素边界」的半像素偏移（如 (n−1)/2 项）在 Foley《Computer Graphics: Principles and Practice》一系教材中出现——**教程建议采用 width/2 主流口径并一句注记半像素约定的存在**，不展开。
- Vulkan 对照（docs.vulkan.org VkViewport 参考页）：VkViewport 额外带 minDepth/maxDepth 字段且允许 minDepth > maxDepth（reversed-Z 的官方后门）——视口变换在 Vulkan 里显式接管了 OpenGL 中 glDepthRange 的职责（第八节详述）。

## 七、法线变换为何用逆转置 (M⁻¹)ᵀ（调研确认）

- 经典论证（Eric Lengyel，teraton.com/blog/transforming-normals.html）：设切向量 t′ = Mt、法线 n 垂直于表面 ⇒ 需 n′·t′ = 0；取 n′ = (M⁻¹)ᵀn 时 n′ᵀt′ = nᵀM⁻¹Mt = nᵀt = 0 恒成立——逆转置是「保持垂直关系」的唯一选择。Nathan Reed 两部曲（reedbeta.com）从 Grassmann 代数视角给出同一结论的深化：法线本质是协变向量（covector），与切向量（逆变向量）天然对偶。
- 病灶图示（PBRT 3ed 2.8.3「Normals」，pbr-book.org）：圆被非均匀压扁后，直接用 M 变换法线得到的向量明显不再垂直于表面；切向量用 M、法线用 (M⁻¹)ᵀ 是 PBRT 的 API 设计。图示（LearnOpenGL Basic Lighting 同款圆/球示意）是「为什么不能直接乘」的直观证据。
- LearnOpenGL 工程口径（learnopengl.com/Lighting/Basic-Lighting + Review 页）：法线矩阵（normal matrix）= model（或 model-view）矩阵去掉平移的 3×3 部分再做逆转置；等比例缩放下逆=转置可退化为直接用原 3×3；变换后需 normalize 重新归一（paroj.github.io gltut「Normal Transformation」补 normalize 论证）。
- 化简边界（多源交叉确认：learnopengl.com、stackoverflow.com、teraton.com）：**只含旋转与均匀缩放时 (M⁻¹)ᵀ ∝ M（差一个标量），normalize 后等价——「均匀缩放随便乘、非均匀缩放必须逆转置」是教程可直接立的一句话结论**；这也是 Stages 第 3 节「法线矩阵为何用逆转置」坑点的理论出处。
- 来源注记：提示词候选名单中的「Fabian Giesen 的文章」经检索核实——其 ryg blog 线性代数系列与 row-major/column-major 文（fgiesen.wordpress.com）对矩阵约定主题有权威论述，但法线逆转置的专门推导以 Lengyel 与 Nathan Reed 为最可靠出处，本章按实际来源引用。

## 八、各 API 的 NDC/深度范围差异（调研确认，第 3/6 节坑点的理论出处）

- OpenGL 传统口径：视体 −w ≤ x,y,z ≤ w，除法后 NDC x/y/z ∈ [−1,1]；NDC y 轴朝上（wikis.khronos.org/opengl「Vertex Post-Processing」；opengl.org 旧文档 4.3 节确认 z 经 glDepthRange 映射）。
- Vulkan 口径（docs.vulkan.org 官方文档）：裁剪后 NDC z ∈ [0,1]（Depth guide 原文：anything outside of [0,1] is clipped）；NDC y 轴朝下（点向下）——OpenGL 投影矩阵直接搬进 Vulkan 会 y 翻转 + 深度半带裁掉（多源交叉确认：matthewwellings.com「The new Vulkan Coordinate System」、saschawillems.de「Flipping the Vulkan viewport」、johannesugb.github.io「Why projection matrices typically used with OpenGL fail with Vulkan」、TU Wien Fiedler 2022 primer）。
- TU Wien Fiedler 2022 primer（cg.tuwien.ac.at，Vulkan 系列论文配套讲义）给出对照总表：NDC x 恒朝右；OpenGL y-up + z ∈ [−1,1]，Vulkan y-down + z ∈ [0,1]；并指出两者 NDC 手性相反（GL 左手、Vulkan 右手）——这是「同一理论、两套约定」的学术级对照来源。
- Direct3D/Metal：NDC z ∈ [0,1]、y 朝下（贴图/屏幕坐标系一致）；Unity 官方文档跨 API 对照表与 realtech-vr.com 对照表确认（docs.unity3d.com、realtech-vr.com）。Metal 同 D3D 口径。
- 官方弥合机制（权威出处）：OpenGL 4.5 ARB_clip_control 扩展——glClipControl(GL_UPPER_LEFT, GL_ZERO_TO_ONE) 可切到 D3D/Vulkan 口径（registry.khronos.org 扩展规范原文）；Vulkan 侧 VK_EXT_depth_clip_control 的 negativeOneToOne 可切回 [−1,1]（docs.vulkan.org VkViewport 参考页与 Depth guide）；数学库侧 GLM_FORCE_DEPTH_ZERO_TO_ONE / perspectiveZO、glMatrix ZO 版（glm.g-truc.net、glmatrix.net）。
- reversed-Z 补充（供简报裁量）：VkViewport 的 minDepth/maxDepth 允许倒挂；D3D 系引擎（含 Unity D3D 平台）常用 reversed-Z + 浮点深度提升远处精度（docs.unity3d.com、zephyr3d.org 文档、第九节 NVIDIA/Reed 出处呼应）。
- 行/列主序与矩阵约定差异（与 ch01 1.7 接续）：fgiesen.wordpress.com「Row-major vs. column-major and GL ES」——存储主序（内存摆放）与乘法方向（数学约定）是正交的两件事；GLM（列主序+列向量）与 DirectXMath（行主序+行向量）是两端代表。本章只用结论级一句，深入已在 ch01、实操归第 3/6 节。
- 注记：提示词候选「Matthias Moulin」与「乐长炯」未能在本轮检索中找到可靠对应来源（提示词本身标问号），已剔除；以 TU Wien primer、Khronos/Vulkan 官方文档、Unity 官方文档与知名社区文章（saschawillems、matthewwellings、johannesugb）为对照表来源。

## 九、深度缓冲与非线性深度的第一印象（调研确认；深入归第 3 章，本章点到即止）

- 事实核心（多源交叉确认：learnopengl.com「Depth Testing」、wikis.khronos.org/opengl「Depth Buffer Precision」、developer.nvidia.com「Visualizing Depth Precision」、reedbeta.com「Depth Precision Visualized」）：透视投影后的深度值与 1/z 成正比而非线性距离——NDC 竖曲线把精度集中堆在近平面附近，远处精度指数衰减；zNear 越接近 0 精度坍塌越剧烈（OpenGL Wiki：zNear 逼近 0 是深度精度第一杀手；zFar 拉远伤害小得多）。
- 与第四节的代数勾连（简报可用作收束）：z_ndc = A·z_view + B 形式的分子分母（−2fn/(f−n) 一项）除以 w 后即成 a + b/z 型——非线性是推导的直接副产品，不是硬件玄学（scratchapixel.com、reedbeta.com 推导一致）。
- z-fighting 与 reversed-Z：共面/近共面物体因精度不足闪烁（z-fighting）；浮点深度 + reversed-Z（近=1 远=0）让浮点的高精度小数值区间对齐远处（developer.nvidia.com、zephyr3d.org；Unity D3D 平台实践佐证 docs.unity3d.com）。本章只作「第一印象」，深度测试机制与插值归第 3 章光栅化。
- GAMES101 L5 已含 z-buffer 与插值内容（sites.cs.ucsb.edu GAMES101_Lecture_05.pdf）——本章资源指针到 L4 变换为止，L5 起归第 3 章使用（分界口径与 Stages 一致）。

## 十、教学资源定位（延伸资源候选，均已核实存在）

- GAMES101 课件 L04「Transformation Cont.」（http://sites.cs.ucsb.edu/~lingqi/teaching/resources/GAMES101_Lecture_04.pdf ）——本章主线：MVP、视图变换、正交/透视推导全覆盖；L05「Rasterization 1」（…/GAMES101_Lecture_05.pdf ）开头含 fovY/aspect 与视口矩阵，其余归第 3 章。课程主页 https://sites.cs.ucsb.edu/~lingqi/teaching/games101.html （ch01 已核实，Stages 第 2 节主线资源）。
- LearnOpenGL「Coordinate Systems」（https://learnopengl.com/Getting-Started/Coordinate-Systems ）与「Camera」（https://learnopengl.com/Getting-Started/Camera ，自定义 lookAt 源码见 code_viewer camera-exercise2）、中文版「坐标系统/摄像机」页（https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/ 、…/09%20Camera/ ）——术语源 + lookAt 工程口径。
- songho.ca 三件套：「OpenGL Transformation」（https://www.songho.ca/opengl/gl_transform.html ，管线总图与 GL_MODELVIEW）、「OpenGL Projection Matrix」（https://www.songho.ca/opengl/gl_projectionmatrix.html ，透视/正交逐步图解）、「OpenGL Camera」（https://www.songho.ca/opengl/gl_camera.html ，gluLookAt 推导）；另有「OpenGL Viewport Transform」（https://www.songho.ca/opengl/gl_viewport.html ）。
- Scratchapixel「The Perspective and Orthographic Projection Matrix」专课（https://www.scratchapixel.com/lessons/3d-basic-rendering/perspective-and-orthographic-projection-matrix/ ，building-basic-perspective-projection-matrix 等章节核实存在）——最详尽的逐步推导参考。
- 3D Math Primer for Graphics and Game Development, 2nd ed（https://gamemath.com/book/ 全书免费在线）——第 5 章「Matrices and Linear Transformations」（https://gamemath.com/book/matrixtransforms.html 核实存在）含正交投影/变换分类（5.3、5.7 节），相机与投影矩阵章节随查。
- Eric Lengyel「Transforming Normals」（https://teraton.com/blog/transforming-normals.html ）——法线逆转置最干净的代数推导；Nathan Reed「Normals and the Inverse Transpose」两部曲（https://www.reedbeta.com/blog/normals-inverse-transpose-part-1/ ）——进阶对偶视角；同站「Depth Precision Visualized」（https://www.reedbeta.com/blog/depth-precision-visualized/ ）。
- PBRT 3ed「Applying Transformations」2.8.3 Normals 小节（https://www.pbr-book.org/3ed-2018/Geometry_and_Transformations/Applying_Transformations ）——法线变换的教科书表述。
- TU Wien「Primer on Coordinate Systems」（Fiedler 2022，https://www.cg.tuwien.ac.at/research/publications/2022/fiedler-2022-vlk/ 站内 PDF）——OpenGL/Vulkan NDC 对照的学术来源；配套社区文章 Sascha Willems「Flipping the Vulkan viewport」（https://www.saschawillems.de/blog/2019/03/29/flipping-the-vulkan-viewport/ ）、Matthew Wellings「The new Vulkan Coordinate System」（https://matthewwellings.com/blog/the-new-vulkan-coordinate-system/ ）、Johannes Unterguggenberger「Why projection matrices typically used with OpenGL fail with Vulkan」（https://johannesugb.github.io/gpu-programming/why-do-opengl-proj-matrices-fail-in-vulkan/ ）——第 6 节前向指针的候选延伸。
- NVIDIA「Visualizing Depth Precision」（https://developer.nvidia.com/blog/visualizing-depth-precision/ ）——1/z 精度可视化，第九节主题。
- Khronos OpenGL Wiki（https://wikis.khronos.org/opengl/ ，Vertex Post-Processing / Depth Buffer Precision 等页）与 Vulkan 官方文档（https://docs.vulkan.org ，VkViewport / Depth guide）——规范级出处。
- glMatrix v4 源码（https://glmatrix.net/docs/v4/classes/Mat4.html ，perspectiveNO/ZO 文档）与 GLM matrix_clip_space（https://glm.g-truc.net/0.9.9/api/a00243.html ）——工程口径交叉验证对象（第 3 节将换 GLM 的理论伏笔）。
- 检索中未采用/剔除：候选名单中的「乐长炯」「Matthias Moulin」未找到可靠来源；docs.gl 仅为函数速查（glViewport 规格已核实，列入第 3 节资源更合适）。

## 十一、与 ch01 的衔接点（后向引用可用素材）

- ch01 数学库现有 API（tutorial/ch01-prerequisite-math.md 1.10 节确认）：Vec2/3/4 加减与标量乘、dot、cross(Vec3)、length/normalize；Mat4 identity/mul、rotateX/Y/Z、scale、translate（列向量右乘、平移在最后一列）、transformPoint/transformVector。**lookAt/ortho/perspective/viewport 在 1.10 有「显式缺席声明」，正是本章 P3·M0 后半的兑现清单**。
- ch01 1.4 叉乘「构建局部坐标系」种子 → lookAt 的正交基构造直接回收（cross 两次出标准正交基）。
- ch01 1.5 正交矩阵 RᵀR=I、逆=转置 → lookAt 旋转部分直接写出、GAMES101 R_view 推导口径的现成地基。
- ch01 1.6 齐次坐标 w 语义（w=1 点、w=0 向量、w 是除数的一句伏笔）→ 本章透视除法正式收束；ch01-brief「留给第 2 章的接口清单」四项（透视除法、lookAt/ortho/perspective/viewport、法线逆转置、相机正交基）即本章必兑清单。
- ch01 1.7 约定之战（行/列向量、主序、左右手）→ 本章全程沿用「列向量右乘、右手系、相机看 −Z」选定声明；第 3/6 节跨 API 坑点全部溯源至此。
- ch01 性质测试基建（assert 式自测、容差比较 1e-5 量级、相对容差地板）→ 本章 MVP 链路手工计算对答案的裁判；新增性质测试候选：view·view⁻¹=I 类、投影矩阵作用后近平面角点 z_ndc=−1、NDC 边界点经视口变换落到预期像素。
- ch01 深入专题「高度场法线」→ 本章法线逆转置 (M⁻¹)ᵀ 的直接后继：同一批法线经非均匀缩放变换后「直接乘 M vs 逆转置」的对照实验可复用该专题场景（简报裁量）。
- 01-cpp 侧挂账：无新挂账；ch01-brief 前向指针四条（透视除法/MVP/lookAt 系列/法线逆转置）均为本章正文义务。

## 十二、Stages.md 阶段 2 覆盖基线（原文要点，简报必须全覆盖）

- 核心知识点：MVP 各变换职责；lookAt 视图矩阵构造；正交/透视投影矩阵完整推导；透视除法（w 分量来源）与视口变换；法线变换用逆转置 (M⁻¹)ᵀ 的原因；各 API 的 NDC/深度范围差异——第 3/6 节常见坑点的理论出处。
- 实践任务：在阶段 1 数学库上实现 lookAt/ortho/perspective/viewport；手工计算一个顶点的完整 MVP 链路并与标准结果对照。
- 检验标准：白板推导透视投影矩阵；解释「w 如何带来近大远小」；给定「看不见/拉伸/镜像」等症状能点名出错环节。
- 边界（根 Topic.md 第 2 节）：全程纯 CPU、不绑定任何图形 API——API 实操归第 3 节，GLSL/Compute 归第 5 节，Vulkan 归第 6 节；本章「各 API 差异」仅作理论对照地图，不涉实操。
