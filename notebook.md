Open GL

    OpenGL 状态机核心机制:
    OpenGL 底层采用状态机架构设计。
    其核心特征为：大多数图形接口并不直接接受对象句柄作为参数，而是通过切换全局缓冲绑定目标来改变当前图形管线上下文的激活对象。后续的配置指令将隐式地作用于该激活目标。






VAO,VBO,EBO

1.VBO：顶点缓冲对象 (Vertex Buffer Object)
    VBO 的本质是 GPU 显存中的一块原始、无结构的二进制数据块。
    它不具备任何语义信息，只负责高速存储由 CPU 运送过来的裸数据（raw data）。

    具体记录内容： 连续存放的顶点属性数据。这些数据以单精度浮点数 float 或定点数形式排列，可以包含：
    1.顶点三维坐标数据：X, Y, Z
    2.顶点颜色数据：R, G, B, A
    3.纹理坐标数据：U, V法线向量数据：N_x, N_y, N_z
    
    状态特征： 没有任何关于“如何阅读”的配置，只有一串连续的字节流

    数据流向： 通过 glBindBuffer(GL_ARRAY_BUFFER, VBO) 将特定 VBO 挂载至顶点属性绑定目标；再通过 glBufferData 函数将 CPU 端的连续内存数据流复制至该显存空间。

2.EBO:元素缓冲对象 (Element Buffer Object)
    EBO 同样是一块显存缓冲区，但它具有明确的数据类型限制（unsigned int），专门用来存储顶点索引。

    具体记录内容：
    存储图元装配所需的拓扑索引 (Topological Indices)。它将顶点数据源（坐标、颜色等）与图元连接顺序解耦，避免了共享顶点在显存中的冗余存储。这些整数代表 VBO 中顶点的绝对序号（从 0 开始编号）。

    状态特征： 它是一张“点名册”或“连线说明书”，其内容严格指示了显卡应该以何种顺序、复用哪些顶点来组装几何图元。

3.VAO：顶点数组对象 (Vertex Array Object)
    VAO 本身不存储任何实质性的顶点坐标或索引数据。它是一个状态容器，现代 OpenGL 强制要求在核心模式下必须使用 VAO 来存储和管理顶点特定阶段的所有状态配置。


    具体记录内容（封装的状态列表）：
    | 状态项 (State Item) | 数据类型/封装内容 (Data Type / Encapsulated Content) | 触发接口 (Trigger API) |
| :--- | :--- | :--- |
| **属性通道启用状态**<br>(Attribute Enable States) | 布尔数组 (`GL_TRUE` / `GL_FALSE`) | `glEnableVertexAttribArray` |
| **顶点属性配置元数据**<br>(Attribute Specification) | 组件数量 (Size)、数据类型 (Type)、归一化标识 (Normalized) | `glVertexAttribPointer` |
| **内存布局映射规则**<br>(Memory Layout Mapping) | 步长 (Stride)、起始偏移量 (Offset) | `glVertexAttribPointer` |
| **VBO 句柄映射关联**<br>(VBO Explicit Association) | 记录当前属性通道对应的数据源 VBO ID | `glVertexAttribPointer`<br>(隐式读取 `GL_ARRAY_BUFFER`) |
| **专属索引缓冲绑定**<br>(Element Array Buffer Binding) | 记录当前激活的 EBO ID | `glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ...)` |



   | 函数名称 | 它的本质属性 | 初始化阶段需要它吗？ | 每一帧渲染循环（while里）需要它吗？ |
| :--- | :--- | :--- | :--- |
| `glGen*`系列 | **资源申请** | **是的**（只申请一次） | ❌ **绝对不要**（否则显存每秒泄露60次） |
| `glBufferData` | **数据灌注** | **是的**（一次性把模型运进显卡） | ❌ **不要**（除非你的模型顶点在实时跳舞） |
| `glVertexAttribPointer` | **规则录制** | **是的**（配合 VAO 录制一次） | ❌ **不要**（VAO 已经帮你记住了） |
| `glBindVertexArray` | **一键换档** | **是的**（开启录制） | **是的**（每一帧都要一键恢复特定物体的配置） |
| `glUseProgram` | **衣服切换** | **可选** | **是的**（每一帧用来指定用哪个 Shader 画图） |
| `glDrawArrays` / `glDrawElements` | **触发绘制 (Draw Call)** | ❌ **不需要** | **是的**（每一帧的临门一脚，真正命令显卡干活） |


    隐式绑定与解绑逻辑：
    VBO 的多对一隐式映射： 当执行 glVertexAttribPointer(index, ...) 时，状态机会读取当前挂载在 GL_ARRAY_BUFFER 上的 VBO 句柄，并将其硬编码记录在当前激活 VAO 的第 index 号属性卡槽中。因此，配置完成后解绑 GL_ARRAY_BUFFER（即执行 glBindBuffer(GL_ARRAY_BUFFER, 0)）是安全的，VAO 已完整保留了对该 VBO 的引用。

    EBO 的强力所有权封装： 当 glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO) 执行时，若当前存在激活的 VAO，该 EBO 句柄会直接写入该 VAO 的专属索引绑定卡槽。因此，在当前 VAO 未解绑前，绝对禁止解绑 GL_ELEMENT_ARRAY_BUFFER（即不能执行 glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0)），否则将直接清空当前 VAO 内部记录的索引状态。



顶点着色器&片段着色器

1.顶点着色器
    1.1核心职能
    顶点着色器是管线的第一站。它的核心任务是处理彼此孤立的几何顶点数据，计算顶点在标准化设备坐标中的最终投影位置。


    1.2 数据输入与输出
    输入数据源：直接来自于 CPU 泵入显存的顶点缓冲对象(VBO)。

    布局限定符：
    在 GLSL 中使用 layout (location = n) in 语法声明输入变量。该语法在 GPU 硬件层面指定了属性流水线的输入索引。CPU 端通过 glVertexAttribPointer(n, ...) 和 glEnableVertexAttribArray(n) 与此处的位置值 n 进行严格对齐。该机制消除了运行时调用 glGetAttribLocation 检索属性名称的字符串匹配开销。
    
    最终输出目标 —— 内置变量 gl_Position：
    顶点着色器计算出的 3D 坐标必须赋值给 GLSL 的内置变量 gl_Position。该变量在幕后是一个四维的齐次坐标向量vec4(X, Y, Z, W)。
    其中： 引入W=1.0（代表点）或 W=0.0（代表方向），可以将 3D 空间中的旋转、缩放、平移统一合并为一个4*4矩阵与4*1向量的乘法。



2.片段着色器
    2.1核心职能    
    处理经过光栅化和插值后生成的离散片段，计算并决定屏幕像素的最终 RGBA 颜色值。


    2.2强制输出
    必须显式声明一个类型为 out vec4 的输出变量，用于向颜色缓冲区写入最终颜色。



3.着色器阶段间通信与变量匹配机制
    顶点着色器与片段着色器之间的数据交互遵循严格类型与名称匹配铁律。

    3.1发送端
    顶点着色器中使用 out 关键字定义变量：out vec4 vertexColor;。


    3.2接收端
    片段着色器中使用 in 关键字定义同名、同类型的变量：in vec4 vertexColor;。


    3.3链接期绑定
    当执行 glLinkProgram 时，OpenGL 链接器会审阅各阶段的输入输出声明，建立内部数据传输通道。




GLSL 向量操作与语法特性

    1. 分量命名系统
    GLSL 为多维向量 (vec2, vec3, vec4) 的分量提供了三套语义化的访问符号。它们在底层完全等价且可以互换，但建议依据上下文语义选用：
    空间几何坐标： x, y, z, w
    颜色通道： r, g, b, a
    纹理贴图坐标： s, t, p, q


    2.分量重组
    允许通过任意顺序组合上述分量访问符号，实现向量分量的快速抽取、复制、乱序与升降维操作。



核心 API 状态切换与生命周期对账
| 函数名 (API Name) | 核心状态机行为 (State Machine Action) | 执行频次策略 (Execution Frequency) |
| :--- | :--- | :--- |
| `glGenVertexArrays` /<br>`glGenBuffers` | 申请并分配资源句柄 ID | **初始化阶段：** 仅执行一次。禁止写入渲染循环，否则引发显存泄漏。 |
| `glBufferData` | 向当前绑定的缓冲卡槽分配并复制显存数据 | **初始化阶段：** 一次性写入静态数据。仅在顶点动态更新时进入渲染循环。 |
| `glVertexAttribPointer` | 配置读取规则并与当前 `GL_ARRAY_BUFFER` 隐式绑定 | **初始化阶段：** 配合 VAO 录制一次。禁止写入渲染循环。 |
| `glBindVertexArray` | 激活或切换指定的 VAO，一键恢复其内部封装的所有特定状态 | **渲染循环阶段：** 每一帧执行，用于高速切换渲染目标对象的上下文状态。 |
| `glUseProgram` | 全局切换图形管线当前的着色器执行程序 | **渲染循环阶段：** 每一帧执行，用于切换当前物体的着色与光照算法。 |
| `glDrawElements` | 依据当前 VAO 内封装的 EBO 索引执行硬件加速绘制指令 | **渲染循环阶段：** 每一帧渲染流程的最终触发点 (**Draw Call**)。 |
   