# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

LitchiEngine (荔枝引擎) 是一个个人学习开发的游戏引擎，使用 C++17 编写，主要面向 Windows 平台。

## 构建命令

```bash
# 生成项目文件并构建 (Debug)
cmake -B Build -DCMAKE_BUILD_TYPE=Debug
cmake --build Build --config Debug

# 生成项目文件并构建 (Release)
cmake -B Build -DCMAKE_BUILD_TYPE=Release
cmake --build Build --config Release

# 或使用批处理脚本 (Windows)
GenerateProjects_CMake.bat
```

构建产物位于 `Build/bin/Debug` 或 `Build/bin/Release`。

## 主要目标

- **LitchiRuntime** - 核心运行时静态库
- **LitchiEditor** - 编辑器可执行程序 (默认启动项目)
- **LitchiStandalone** - 独立运行程序
- **LitchiScriptCore** - C# 脚本核心库

## 架构概览

### 源码结构

```
Engine/Source/
├── Runtime/           # 核心运行时
│   ├── Core/          # 核心系统 (App, Math, Log, Time, Window, Screen)
│   ├── Function/      # 功能模块
│   │   ├── Renderer/  # 渲染系统
│   │   ├── Scene/     # 场景管理
│   │   ├── Physics/   # 物理系统 (PhysX)
│   │   ├── Scripting/ # 脚本系统 (Mono/C#)
│   │   ├── UI/        # UI系统 (ImGui)
│   │   ├── Framework/ # 组件框架
│   │   └── Prefab/    # 预制体系统
│   ├── Resource/      # 资源管理器
│   └── Platform/      # 平台相关代码
├── Editor/            # 编辑器
├── Standalone/        # 独立运行程序
└── ScriptCore/        # C# 脚本核心
```

### 渲染系统

渲染系统采用 RHI (Render Hardware Interface) 抽象层设计：

- **RHI 层** (`Engine/Source/Runtime/Function/Renderer/RHI/`) - 硬件抽象接口
- **Vulkan 实现** (`Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/`) - Vulkan 后端实现
- **渲染器** (`Engine/Source/Runtime/Function/Renderer/Rendering/`) - 高层渲染逻辑

主要渲染特性：
- Forward 渲染路径
- 阴影映射
- PBR 材质
- 天空盒
- 骨骼动画

着色器位于 `Engine/Data/Engine/Shaders/`，使用 HLSL 编写，通过 DXCompiler 编译为 SPIR-V。

### 脚本系统

使用 Mono 运行时实现 C# 脚本支持：
- `ScriptEngine` - 脚本引擎核心
- `ScriptClass` - 脚本类句柄
- `ScriptInstance` - 脚本实例
- `ScriptCore/Source/` - C# 端绑定代码

### 组件系统

基于 GameObject-Component 架构：
- `GameObject` - 游戏对象
- `Scene` / `SceneManager` - 场景管理
- 组件定义位于 `Framework/Component/`

### 命名空间

所有引擎代码位于 `LitchiRuntime` 命名空间。

## 第三方依赖

主要依赖位于 `Engine/ThirdParty/`：
- **Vulkan SDK** - 图形 API
- **PhysX** - 物理引擎
- **Mono** - C# 运行时
- **ImGui** - UI 框架
- **Assimp** - 模型加载
- **FreeImage** - 图像加载
- **FreeType** - 字体渲染
- **spdlog** - 日志系统
- **GLFW** - 窗口管理
- **RTTR** - 运行时类型反射

## 资源路径

- 着色器: `Engine/Data/Engine/Shaders/`
- 配置: `Engine/Config/`
- 构建输出资源: `Build/bin/{Debug|Release}/Data/`
