# 第一个Filter插件实施方案

## 目标

完成第一个自定义Filter插件的开发、编译、加载和验证全流程。插件功能为「顶点法线可视化增强」：计算网格顶点法线，用颜色编码法线方向，用户可调节箭头长度和颜色模式。

## 总体进度拆分

分为7个里程碑，每个里程碑独立可验证。

---

## 里程碑1：搭建开发环境（预计1-2天）

### 1.1 确认构建工具链

需要确认以下工具已安装：

| 工具 | 最低版本 | 验证命令 |
|------|----------|----------|
| CMake | 3.18+ | `cmake --version` |
| Visual Studio | 2019/2022（含C++桌面开发） | 启动VS确认 |
| Qt5 | 5.15+ | `qmake --version` 或检查安装路径 |
| Git | 最新 | `git --version` |

### 1.2 配置CMake项目

```powershell
# 创建构建目录（避免在源码目录内构建）
cd "e:\Github Project\MeshLab"
mkdir build
cd build

# 执行CMake配置（替换Qt路径为实际路径）
cmake ../MeshLab-src -DCMAKE_PREFIX_PATH="C:/Qt/5.15.2/msvc2019_64" -G "Visual Studio 17 2022"
```

### 1.3 编译验证

```powershell
# 编译filter_sample插件（验证环境是否正常）
cmake --build . --target filter_sample --config Release
```

**验证标准**：`build/distrib/plugins/` 目录下生成 `filter_sample.dll`，无编译错误。

### 1.4 加载验证

将 `filter_sample.dll` 复制到MeshLab安装目录的 `plugins/` 文件夹，启动MeshLab，确认菜单 Filters → Smoothing → Random Vertex Displacement 存在。

### 1.5 学习阅读清单

在等待编译期间，精读以下三个文件（这是开发新插件需要的全部知识）：

1. `MeshLab-src/src/meshlabplugins/filter_sample/filter_sample.h`（83行）
2. `MeshLab-src/src/meshlabplugins/filter_sample/filter_sample.cpp`（229行）
3. `MeshLab-src/src/common/plugins/interfaces/filter_plugin.h`（261行）

---

## 里程碑2：创建插件骨架（预计1天）

### 2.1 创建插件目录

```
MeshLab-src/src/meshlabplugins/filter_normal_enhance/
├── CMakeLists.txt
├── filter_normal_enhance.h
└── filter_normal_enhance.cpp
```

### 2.2 编写CMakeLists.txt

```cmake
set(SOURCES filter_normal_enhance.cpp)
set(HEADERS filter_normal_enhance.h)
add_meshlab_plugin(filter_normal_enhance ${SOURCES} ${HEADERS})
```

这和filter_sample完全一致。`add_meshlab_plugin` 函数（定义在 `src/cmake/meshlab_tools.cmake`）会自动：
- 创建 MODULE 类型的共享库（.dll）
- 链接 `meshlab-common` 库
- 设置输出到 `plugins/` 目录

### 2.3 编写头文件骨架

```cpp
#ifndef FILTER_NORMAL_ENHANCE_H
#define FILTER_NORMAL_ENHANCE_H

#include <common/plugins/interfaces/filter_plugin.h>

class FilterNormalEnhance : public QObject, public FilterPlugin
{
    Q_OBJECT
    MESHLAB_PLUGIN_IID_EXPORTER(FILTER_PLUGIN_IID)
    Q_INTERFACES(FilterPlugin)

public:
    // 动作ID枚举 - 每个枚举值对应菜单中的一个功能
    enum { FP_NORMAL_VISUALIZE, FP_NORMAL_FLIP };

    FilterNormalEnhance();
    virtual ~FilterNormalEnhance();

    // 必须实现的接口
    QString pluginName() const;
    QString filterName(ActionIDType filter) const;
    QString pythonFilterName(ActionIDType f) const;
    QString filterInfo(ActionIDType filter) const;
    FilterClass getClass(const QAction* a) const;
    FilterArity filterArity(const QAction*) const;
    int getPreConditions(const QAction *) const;
    int postCondition(const QAction* ) const;

    // 可选实现 - 定义参数对话框
    RichParameterList initParameterList(const QAction*, const MeshModel &/*m*/);

    // 核心执行函数
    std::map<std::string, QVariant> applyFilter(
            const QAction* action,
            const RichParameterList & parameters,
            MeshDocument &md,
            unsigned int& postConditionMask,
            vcg::CallBackPos * cb);

private:
    bool visualizeNormals(
            MeshDocument &md,
            vcg::CallBackPos *cb,
            bool updateNormals,
            Scalarm normalLength,
            int colorMode);

    bool flipNormals(
            MeshDocument &md,
            vcg::CallBackPos *cb);
};

#endif
```

### 2.4 编写实现文件骨架

构造函数中注册动作、switch语句分发，其余函数返回基本值。骨架能编译通过即可，功能逻辑在里程碑3填充。

### 2.5 注册到构建系统

在 `MeshLab-src/src/CMakeLists.txt` 的 `MESHLAB_PLUGINS` 列表中添加一行（第142行filter_sample之后）：

```cmake
meshlabplugins/filter_normal_enhance
```

### 2.6 验证

```powershell
cmake --build . --target filter_normal_enhance --config Release
```

**验证标准**：编译通过，生成 `filter_normal_enhance.dll`。

---

## 里程碑3：实现核心算法（预计2-3天）

### 3.1 实现参数定义（initParameterList）

为 FP_NORMAL_VISUALIZE 定义三个参数：

```cpp
case FP_NORMAL_VISUALIZE:
    // 布尔开关：是否重新计算法线
    parlst.addParam(RichBool("UpdateNormals", true,
        "Recompute Normals",
        "Toggle the recomputation of the normals after processing."));

    // 百分比参数：法线箭头长度（基于模型包围盒对角线）
    parlst.addParam(RichPercentage("NormalLength",
        m.cm.bbox.Diag()/50.0f,  // 默认值：包围盒对角线的1/50
        0.0f,                     // 最小值
        m.cm.bbox.Diag(),         // 最大值
        "Normal Arrow Length",
        "Length of the normal visualization arrows."));

    // 枚举参数：颜色编码模式
    parlst.addParam(RichEnum("ColorMode", 0,
        QStringList() << "Direction (RGB)" << "Curvature" << "Uniform",
        "Color Mode",
        "How to color the normals: RGB direction mapping, curvature estimation, or uniform color."));
    break;
```

FP_NORMAL_FLIP 无需参数。

### 3.2 实现法线可视化算法（visualizeNormals）

核心逻辑流程：

```
1. 获取网格对象：CMeshO &m = md.mm()->cm;
2. 确保法线存在：如果需要，调用 vcg::tri::UpdateNormal<CMeshO>::PerVertex(m)
3. 确保顶点颜色存在：如果需要，调用 m.vert.EnableColor()
4. 遍历所有顶点：
   a. 根据ColorMode计算颜色
   b. 设置顶点颜色：m.vert[i].C() = vcg::Color4b(r, g, b, 255)
   c. 报告进度：cb(100*i/m.vert.size(), "Computing normals...")
5. 更新包围盒：vcg::tri::UpdateBounding<CMeshO>::Box(m)
6. 输出日志：log("Successfully computed normals for %i vertices", m.vn)
```

颜色模式实现：
- **Direction (RGB)**：将法线 (nx, ny, nz) 映射到 (R, G, B)，值域 [-1,1] 映射到 [0,255]
- **Curvature**：使用VCGlib的曲率估算，映射到颜色梯度
- **Uniform**：统一使用蓝色

### 3.3 实现法线翻转算法（flipNormals）

```
1. 获取网格对象
2. 遍历所有顶点：m.vert[i].N() = -m.vert[i].N()
3. 遍历所有面：m.face[i].N() = -m.face[i].N()
4. 更新日志
```

### 3.4 实现条件声明

```cpp
// 前置条件：需要顶点数据
int FilterNormalEnhance::getPreConditions(const QAction*) const {
    return MeshModel::MM_VERTCOORD | MeshModel::MM_FACENUMBER;
}

// 后置条件：会修改法线和顶点颜色
int FilterNormalEnhance::postCondition(const QAction*) const {
    return MeshModel::MM_VERTNORMAL | MeshModel::MM_VERTCOLOR;
}
```

### 3.5 验证

编译通过，加载到MeshLab中，菜单正确显示在 Filters → Normal 分类下。

---

## 里程碑4：UI交互完善（预计1天）

### 4.1 完善filterInfo描述

为每个动作编写详细的HTML格式描述（会显示在参数对话框顶部）：

```cpp
case FP_NORMAL_VISUALIZE:
    return "Visualize vertex normals using color coding.<br><br>"
           "This filter computes per-vertex normals and encodes their "
           "direction as RGB colors:<br>"
           "<b>X component</b> maps to Red<br>"
           "<b>Y component</b> maps to Green<br>"
           "<b>Z component</b> maps to Blue<br><br>"
           "Useful for inspecting mesh normal consistency and detecting "
           "flipped or incorrect normals before export.";
```

### 4.2 设置正确的菜单分类

```cpp
FilterNormalEnhance::FilterClass FilterNormalEnhance::getClass(const QAction *a) const
{
    switch(ID(a)) {
    case FP_NORMAL_VISUALIZE:
    case FP_NORMAL_FLIP:
        return FilterPlugin::Normal;  // 放在 Normal 子菜单下
    default:
        assert(0);
        return FilterPlugin::Generic;
    }
}
```

### 4.3 设置applyFilter返回值

让命令行调用也能获取有用信息：

```cpp
std::map<std::string, QVariant> FilterNormalEnhance::applyFilter(...)
{
    std::map<std::string, QVariant> output;
    switch(ID(action)) {
    case FP_NORMAL_VISUALIZE:
        visualizeNormals(md, cb, ...);
        output["processed_vertices"] = QVariant(md.mm()->cm.vn);
        break;
    case FP_NORMAL_FLIP:
        flipNormals(md, cb);
        output["flipped_normals"] = QVariant(md.mm()->cm.vn);
        break;
    }
    return output;
}
```

### 4.4 验证

在MeshLab中测试完整交互流程：
1. 加载一个网格模型
2. 打开 Filters → Normal → Visualize Normals
3. 参数对话框正确显示三个参数
4. 描述文本正确渲染
5. 执行后顶点颜色正确更新
6. 日志输出正确

---

## 里程碑5：边界情况处理与测试（预计1-2天）

### 5.1 边界情况

需要处理的特殊情况：

| 情况 | 处理方式 |
|------|----------|
| 网格无法线数据 | 自动计算：`vcg::tri::UpdateNormal<CMeshO>::PerVertex(m)` |
| 网格为点云（无面） | 使用 `UpdateNormal::PerVertex` 基于邻域估算 |
| 网格顶点数为0 | 抛出 `MLException("Mesh has no vertices")` |
| 法线为零向量 | 跳过该顶点颜色设置 |
| 用户取消操作 | 检查 `cb` 回调，支持中断 |

### 5.2 错误处理

```cpp
#include <common/mlexception.h>

bool FilterNormalEnhance::visualizeNormals(...)
{
    CMeshO &m = md.mm()->cm;

    // 空网格检查
    if (m.vn == 0) {
        throw MLException("Mesh has no vertices. Cannot compute normals.");
    }

    // 确保法线已启用
    m.vert.EnableNormal();
    if (m.fn > 0) {
        // 三角网格 - 使用面法线计算顶点法线
        m.face.EnableNormal();
        vcg::tri::UpdateNormal<CMeshO>::PerVertexNormalizedPerFace(m);
    } else {
        // 点云 - 使用PCA估算法线
        vcg::tri::UpdateNormal<CMeshO>::PerVertex(m);
    }

    // 确保颜色已启用
    m.vert.EnableColor();

    // ... 遍历处理 ...
}
```

### 5.3 测试矩阵

用以下模型文件进行测试：

| 模型 | 来源 | 测试目的 |
|------|------|----------|
| Bunny.ply | MeshLab sample目录 | 标准三角网格 |
| icosahedron.obj | 自行生成（filter_create） | 低面数简单网格 |
| 扫描点云 | 自行生成 | 纯点云（无面） |
| 空网格 | filter_create极端参数 | 边界情况 |

---

## 里程碑6：完善与优化（预计1-2天）

### 6.1 添加pymeshlab支持

```cpp
QString FilterNormalEnhance::pythonFilterName(ActionIDType f) const
{
    switch(f) {
    case FP_NORMAL_VISUALIZE:
        return "compute_normal_visualization";
    case FP_NORMAL_FLIP:
        return "flip_vertex_and_face_normals";
    default:
        assert(0);
        return QString();
    }
}
```

### 6.2 代码注释完善

按照MeshLab项目风格添加Doxygen注释，参考filter_sample.cpp中的注释风格。

### 6.3 版权声明

在头文件和实现文件顶部添加版权声明（参考filter_sample的格式）。

---

## 里程碑7：提交与文档（预计0.5天）

### 7.1 Git提交

```powershell
git add MeshLab-src/src/meshlabplugins/filter_normal_enhance/
git commit -m "添加法线可视化增强Filter插件 (filter_normal_enhance)

新插件包含两个功能：
1. 顶点法线可视化 - 用颜色编码法线方向(RGB)
2. 法线翻转 - 翻转所有顶点和面法线

技术栈覆盖：C++17, Qt元对象系统, VCGlib几何算法, CMake构建"
git push origin master
```

### 7.2 编写插件说明文档

创建 `filter_normal_enhance/README.md`，包含功能说明、使用方法和截图。

---

## 关键代码参考

### 类继承链

```
QObject                    ← Qt元对象系统（信号/槽/反射）
    ↓
MeshLabPlugin              ← 插件基类（pluginName、version）
    ↓
MeshLabPluginLogger        ← 日志功能（log函数）
    ↓
FilterPlugin               ← Filter接口（applyFilter等纯虚函数）
    ↓
FilterNormalEnhance        ← 你的实现
```

### 必须的三个Qt宏

```cpp
Q_OBJECT                                    // 启用元对象系统
MESHLAB_PLUGIN_IID_EXPORTER(FILTER_PLUGIN_IID)  // Qt插件元数据 + getMLVersion()
Q_INTERFACES(FilterPlugin)                  // 声明实现的接口
```

### 文件末尾必须有

```cpp
MESHLAB_PLUGIN_NAME_EXPORTER(FilterNormalEnhance)  // 展开为空，但是约定标记
```

### RichParameterList参数类型速查

| 类型 | 构造函数 | 读取方法 |
|------|----------|----------|
| 布尔 | `RichBool(name, default, label, tooltip)` | `parameters.getBool(name)` |
| 整数 | `RichInt(name, default, label, tooltip)` | `parameters.getInt(name)` |
| 浮点 | `RichFloat(name, default, label, tooltip)` | `parameters.getFloat(name)` |
| 百分比 | `RichPercentage(name, def, min, max, label, tooltip)` | `parameters.getAbsPerc(name)` |
| 枚举 | `RichEnum(name, index, QStringList, label, tooltip)` | `parameters.getEnum(name)` |
| 颜色 | `RichColor(name, QColor, label, tooltip)` | `parameters.getColor(name)` |
| 字符串 | `RichString(name, QString, label, tooltip)` | `parameters.getString(name)` |

### VCGlib常用操作速查

```cpp
CMeshO &m = md.mm()->cm;                                     // 获取当前网格
m.vert.size()                                                // 顶点数量
m.vert[i].P()                                                // 顶点位置 (Point3m)
m.vert[i].N()                                                // 顶点法线 (Point3m)
m.vert[i].C()                                                // 顶点颜色 (Color4b)
m.face[i].N()                                                // 面法线 (Point3m)
m.cm.bbox.Diag()                                             // 包围盒对角线长度
vcg::tri::UpdateNormal<CMeshO>::PerVertexNormalizedPerFace(m); // 计算顶点法线
vcg::tri::UpdateBounding<CMeshO>::Box(m);                     // 更新包围盒
m.vert.EnableNormal();                                       // 启用法线属性
m.vert.EnableColor();                                        // 启用颜色属性
```

---

## 预期时间线

| 里程碑 | 预计耗时 | 累计耗时 | 可交付物 |
|--------|----------|----------|----------|
| M1: 搭建环境 | 1-2天 | 2天 | 可编译filter_sample的环境 |
| M2: 创建骨架 | 1天 | 3天 | 可编译的空插件DLL |
| M3: 核心算法 | 2-3天 | 6天 | 功能完整的插件DLL |
| M4: UI完善 | 1天 | 7天 | 菜单分类正确的插件 |
| M5: 边界处理 | 1-2天 | 9天 | 健壮的插件 |
| M6: 优化完善 | 1-2天 | 11天 | 可发布的插件 |
| M7: 提交文档 | 0.5天 | 11.5天 | Git提交+文档 |

**总计约2周**，考虑到学习曲线，预留缓冲期建议按3周规划。

---

## 遇到问题时的排查路径

| 问题 | 排查步骤 |
|------|----------|
| CMake配置失败 | 检查Qt路径、CMake版本、VS版本是否匹配 |
| 编译链接错误 | 确认filter_plugin.h路径正确、meshlab-common已编译 |
| MeshLab找不到插件 | 确认DLL在plugins目录、DLL依赖的meshlab-common.dll存在 |
| 菜单中看不到 | 确认MESHLAB_PLUGINS列表已添加、DLL文件名正确 |
| 运行时崩溃 | 检查网格数据是否启用了所需属性（EnableNormal等） |
