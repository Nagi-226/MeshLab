# MeshLab 二次开发优先级规划

> 基于二次开发潜力评估报告，针对「个人学习/技术探索」场景的专项规划

## 一、为什么算法功能扩展（Filter 插件）是最佳切入点

评估报告给出了五个维度的评分：界面功能 4.0、算法功能 4.5、文件格式 4.0、渲染引擎 3.8、整体架构 4.0。从学习角度出发，**算法功能扩展（Filter 插件）**是当之无愧的第一优先级，原因如下：

**第一，它是评分最高的方向（4.5/5.0）**，意味着开发体验最成熟、踩坑最少。MeshLab 提供了完整的插件体系，仅 `FilterPlugin` 一个类型就涵盖了 30 多个官方插件，从简单到复杂应有尽有，学习资料极为丰富。

**第二，一次开发可以覆盖最多技能点**。写一个 Filter 插件的过程中，你自然会接触到 C++ 模板与继承体系（`MeshLabPlugin` -> `MeshLabPluginLogger` -> `FilterPlugin`）、Qt 信号/槽与元对象系统（`Q_OBJECT`、`Q_INTERFACES`、`Q_PLUGIN_METADATA`）、CMake 构建配置（`add_meshlab_plugin` 函数）、VCGlib 几何算法库（`tri::Update*`、`tri::Clean*`、`tri::Append` 等），以及三维网格数据结构的基本概念（顶点、面、法线、包围盒等）。这比单独学界面或 IO 插件覆盖面广得多。

**第三，反馈周期极短**。写完一个 Filter 插件后，编译出一个 `.dll` 放进 `plugins/` 目录，重启 MeshLab 就能在菜单里看到你的滤镜，加载一个模型点一下就能看到效果。这种「写代码 -> 看到效果」的闭环在图形学项目中非常难得。

**第四，完全不影响原有功能**。插件是动态加载的 MODULE 库，与主程序零耦合。你的插件即使有 bug，最差情况也只是 MeshLab 跳过它不加载，不会导致崩溃或数据丢失。

---

## 二、推荐的学习路线（三阶段）

### 阶段一：理解框架（第 1-2 周）

**目标**：搭建开发环境，编译项目，跑通一个示例插件。

**具体步骤**：

1. **搭建开发环境**。安装 CMake（3.18+）、Qt 5.15、Visual Studio（或 MinGW），确保能完整编译 MeshLab。Windows 上建议使用 VS 生成器，调试体验最好。

2. **阅读 filter_sample 源码**。这是官方提供的新版示例插件，位于 `MeshLab-src/src/meshlabplugins/filter_sample/`，包含三个文件：
   - `filter_sample.h`（83 行）：类声明，展示了完整的接口实现
   - `filter_sample.cpp`（229 行）：核心逻辑，包含参数定义和顶点随机位移算法
   - `CMakeLists.txt`（10 行）：极简的构建配置

3. **理解类继承关系**。关键继承链为 `MeshLabPlugin`（插件基类，定义 `pluginName()`）-> `MeshLabPluginLogger`（日志功能，提供 `log()` 模板函数）-> `FilterPlugin`（滤镜接口，定义 `applyFilter()` 等纯虚函数）。类声明中需要三个宏：`Q_OBJECT`（Qt 元对象）、`MESHLAB_PLUGIN_IID_EXPORTER(FILTER_PLUGIN_IID)`（插件元数据 + `getMLVersion()`）、`Q_INTERFACES(FilterPlugin)`（Qt 接口声明）。

4. **理解构建注册流程**。`src/CMakeLists.txt` 的 `MESHLAB_PLUGINS` 列表包含所有插件路径 -> `add_meshlab_plugin()` 函数（定义在 `src/cmake/meshlab_tools.cmake`）将插件编译为 MODULE 库 -> 输出到 `distrib/plugins/` 目录。新增插件只需两步：写一个 `add_meshlab_plugin(...)` 的 CMakeLists.txt，然后在 `MESHLAB_PLUGINS` 列表中加上路径。

**这个阶段要掌握的技能**：CMake 基础、Qt 元对象系统（MOC 编译）、C++ 多继承与纯虚函数。

### 阶段二：实现第一个自定义插件（第 3-6 周）

**目标**：从零写一个属于自己的 Filter 插件。

**推荐实现：顶点法线可视化增强插件**。这个功能的学习价值很高——你需要遍历所有顶点的法线数据，使用 VCGlib 的更新算法，并通过 `RichParameterList` 定义用户参数。具体来说，插件可以提供以下参数：
- 法线显示长度比例（`RichFloat`）
- 是否只显示选中面的法线（`RichBool`）
- 法线颜色（`RichColor`）

**开发流程**：

1. 在 `src/meshlabplugins/` 下创建 `filter_normal_enhance/` 目录
2. 编写 `filter_normal_enhance.h`，参考 `filter_sample.h` 的结构，继承 `QObject` 和 `FilterPlugin`
3. 编写 `filter_normal_enhance.cpp`，实现 `pluginName()`、`filterName()`、`filterInfo()`、`filterArity()`、`initParameterList()`、`applyFilter()` 六个核心方法
4. 编写 `CMakeLists.txt`，调用 `add_meshlab_plugin(filter_normal_enhance filter_normal_enhance.cpp filter_normal_enhance.h)`
5. 在 `src/CMakeLists.txt` 的 `MESHLAB_PLUGINS` 列表中添加 `meshlabplugins/filter_normal_enhance`
6. 编译、运行、调试

**这个阶段要掌握的技能**：VCGlib 的 `tri::UpdateNormal<CMeshO>` 和 `tri::UpdateBounding<CMeshO>` 用法、`RichParameterList` 各类参数的定义方式、`MeshDocument` 和 `CMeshO` 的数据访问方式、`vcg::CallBackPos` 进度回调。

### 阶段三：逐步扩展技能覆盖面（第 7-16 周）

**目标**：通过不同类型的插件开发，系统性地掌握 C++/Qt/OpenGL。

**3a. IO 插件（建议第 7-8 周）**

实现一个简单的自定义格式导入/导出插件。例如支持 OBJ 文件的自定义扩展（如附加法线质量信息）。IO 插件的接口比 Filter 稍复杂，需要实现 `importFormats()`、`exportFormats()`、`open()`、`save()` 四个纯虚函数，以及 `exportMaskCapability()`。`io_base` 插件是 PLY 格式的参考实现，代码量大但结构清晰。

**收获**：文件 I/O 流操作、二进制/ASCII 格式解析、`FileFormat` 结构体用法。

**3b. Decorate 插件（建议第 9-10 周）**

Decorate 插件在三维视图上绘制叠加信息（如坐标轴、标注、测量线等）。实现一个「网格统计信息叠加」装饰器：在视图角落实时显示当前网格的顶点数、面数、边界框尺寸等信息。这会自然地引导你接触 OpenGL 绘制 API。

**收获**：OpenGL 基础绘制（`glBegin/glEnd` 或现代方式）、视图坐标变换、`DecoratePlugin` 接口。

**3c. Edit 插件（建议第 11-12 周）**

Edit 插件提供交互式编辑工具（如点选择、变换操纵器等）。实现一个「交互式顶点拾取」编辑工具，点击网格上的点可以高亮显示并输出坐标。

**收获**：Qt 事件处理（鼠标/键盘）、OpenGL 拾取算法、`EditPlugin` 和 `EditTool` 接口。

**3d. Render 插件（建议第 13-16 周）**

Render 插件涉及自定义渲染管线，是最复杂的类型。可以先从修改现有着色器入手，阅读 `shaders/` 目录下的 `.vert` 和 `.frag` 文件，理解 GLSL 着色器的基本结构。然后尝试实现一个简单的线框+法线可视化渲染模式。

**收获**：GLSL 着色器编程、OpenGL 渲染状态管理、帧缓冲对象（FBO）、`RenderPlugin` 接口。

---

## 三、各方向学习价值对比

| 维度 | 学习技能覆盖 | 开发难度 | 成就感速度 | 推荐学习顺序 |
|------|------------|---------|----------|------------|
| Filter 插件 | C++继承、Qt元对象、CMake、VCGlib几何算法 | 低 | 快（1-2周出成果） | **第1个** |
| IO 插件 | 文件I/O、格式解析、数据序列化 | 低 | 快（1-2周） | 第2个 |
| Decorate 插件 | OpenGL绘制、视图叠加、2D渲染 | 中 | 中（2-3周） | 第3个 |
| Edit 插件 | Qt事件系统、OpenGL拾取、交互逻辑 | 中高 | 中（3-4周） | 第4个 |
| Render 插件 | GLSL着色器、渲染管线、GPU编程 | 高 | 慢（4-6周） | 第5个 |
| 界面扩展 | Qt Widget、停靠面板、工具栏 | 低 | 快 | 任何阶段穿插 |

---

## 四、关键注意事项

**不要参考 samplefilterdoc**。虽然 `unsupported/plugins_unsupported/samplefilterdoc/` 目录下也有一个示例插件，但它使用的是旧版接口（`MeshFilterInterface`、`Q_EXPORT_PLUGIN`、`RichParameterSet`），与当前 MeshLab 2020+ 的接口不兼容。正确的参考蓝本是 `src/meshlabplugins/filter_sample/`。

**善用 VCGlib 的 sample 程序**。`MeshLab-src/src/vcglib/apps/sample/` 目录下有大量独立的小程序（如 `trimesh_curvature`、`trimesh_smooth`、`trimesh_sampling` 等），每个只有几百行代码，演示了 VCGlib 各种算法的用法。这些是学习几何算法的最佳材料，可以在不依赖 MeshLab 的情况下独立编译运行。

**插件输出目录**。编译成功后，插件 `.dll` 会输出到 `distrib/plugins/` 目录。如果只想编译单个插件而不编译整个项目，可以使用 CMake 的目标指定构建：`cmake --build . --target filter_sample`。

**日志而非对话框**。`applyFilter()` 中严禁使用 QMessageBox 或任何 GUI 交互，所有信息通过 `log()` 函数输出到 MeshLab 底部的日志面板。

---

## 五、总结

对于以技术学习为目标的二次开发，**Filter 插件开发是最佳起点**。它能以最低的入门门槛覆盖最广的技术栈，提供最快的正向反馈，并且完全不影响 MeshLab 的原有功能。建议从研读 `filter_sample` 的 229 行代码开始，两周内完成第一个自定义插件的编译和运行，然后按照 IO -> Decorate -> Edit -> Render 的顺序逐步扩展，在 3-4 个月内系统性掌握 C++/Qt/OpenGL 在三维图形应用中的完整开发范式。

---

*规划完成时间：2026年4月3日*
*依据：MeshLab二次开发潜力评估报告 + 源码深度分析*
