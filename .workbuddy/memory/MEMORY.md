# MeshLab项目长期记忆

## 项目基本信息
- **项目名称**: MeshLab
- **项目类型**: 开源三维网格处理工具
- **许可证**: GNU GPL v3
- **主要用途**: 3D扫描数据处理、网格编辑、三维可视化
- **技术栈**: C++、Qt、OpenGL、CMake
- **开发团队**: Visual Computing Lab, ISTI - CNR (意大利国家研究委员会)
- **核心库**: VCGlib (Visual Computing Library)

## 核心功能特点
1. **网格处理**: 编辑、清理、修复、检查三维三角网格
2. **文件格式支持**: PLY、OBJ、STL、3DS、DAE、GLTF等
3. **插件系统**: 可扩展的插件架构，支持Filter、Edit、Decorate、Render、IO五种插件类型
4. **多视图**: 支持多个三维视图同时显示
5. **专业工具**: 针对3D扫描数据的专业处理工具

## 项目结构
- 主程序: `meshlab.exe`
- 源代码: `MeshLab-src/` 目录
- 插件: `plugins/` 目录
- 着色器: `shaders/` 目录
- 示例文件: `sample/` 目录
- CMake构建系统: 完善的跨平台构建配置

## 二次开发潜力评估（2026年4月2日）
### 综合评分：4.06/5.0（优秀水平）

#### 各维度评分：
1. **界面功能扩展**: 4.0/5.0 - Qt框架支持良好，可扩展工具栏、面板
2. **算法功能扩展**: 4.5/5.0 - VCGlib集成优秀，插件架构成熟
3. **文件格式支持扩展**: 4.0/5.0 - IO插件系统完善，易于扩展
4. **渲染引擎优化**: 3.8/5.0 - OpenGL架构稳定，有现代化空间
5. **整体架构改进**: 4.0/5.0 - 代码质量良好，构建系统完善

### 关键发现
- **插件架构成熟**: 基于Qt的插件系统设计优秀，支持动态加载
- **开发资源丰富**: 包含完整示例（samplefilterdoc）、开发模板
- **算法基础强大**: 深度集成VCGlib，提供数百种网格处理算法
- **社区生态活跃**: GitHub社区持续维护，有学术背景支持

### 二次开发建议
- **优先方向**: 算法扩展和插件开发（价值最高）
- **入门路径**: 从研究samplefilterdoc示例开始
- **技术栈要求**: C++、Qt、OpenGL、CMake
- **实施阶段**: 分三阶段（入门准备→功能扩展→深度优化）

### 开发文档和资源
- **插件接口**: 位于`MeshLab-src/src/common/plugins/interfaces/`
- **示例插件**: `MeshLab-src/unsupported/plugins_unsupported/samplefilterdoc/`
- **核心头文件**: `meshlab_plugin.h`、`filter_plugin.h`、`plugin_manager.h`
- **参数系统**: `RichParameterList`动态参数管理系统

## 开发环境（2026年4月3日验证通过）
- **Qt**: E:\Qt\5.15.2\msvc2019_64（MeshLab要求最低Qt 5.15.0）
- **Visual Studio**: E:\VS2026（v18.4，MSVC 19.50，v144/v145工具集）
- **CMake**: 3.29.0（位于 C:\Program Files\CMake\bin）
- **Ninja**: 1.13.2
- **MeshLab版本**: 2025.07（单精度）
- **构建方式**: 必须用Ninja生成器（CMake 3.29不支持VS2026生成器）
- **构建命令**: 需先激活VS DevShell，再cmake -B build -G Ninja -DCMAKE_PREFIX_PATH="E:\Qt\5.15.2\msvc2019_64"
- **filter_sample编译验证通过**: 生成40,960字节的DLL

## 技术特点
- 基于Qt的跨平台GUI
- OpenGL渲染引擎，支持GLSL着色器
- 插件化架构设计，支持五种插件类型
- 支持大型网格数据处理，有性能优化机制
- 多语言界面支持
- CMake构建系统，支持模块化编译

## 使用场景
1. 3D扫描数据处理和优化
2. 三维模型编辑和修复
3. 网格质量分析和改进
4. 三维可视化展示
5. 文件格式转换
6. 算法开发和测试平台
7. 科研和教育工具