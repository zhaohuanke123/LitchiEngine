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

**注意**: MSVC 环境下 `LitchiEditor` 为默认启动项目。

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

**组件生命周期**:
```
OnAwake() → OnEnable() → OnStart() → OnUpdate() → OnDisable() → OnDestroy()
```

**添加新组件**:
1. 继承 `Component` 或其子类
2. 重写生命周期方法
3. 添加 RTTR 注册
4. 在 `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h` 中添加 include 和 RTTR 注册
5. 在 `Engine/Source/Runtime/CMakeLists.txt` 添加源文件

### 序列化系统

基于 RTTR 反射的 JSON 序列化：
- `Serializer::SerializeToJson()` / `DeserializeFromJson()` - 核心序列化方法
- 多态类型使用 `Polymorphic` 元数据标记
- 对象引用通过 ID 机制实现，避免指针序列化

### 命名空间

所有引擎代码位于 `LitchiRuntime` 命名空间。

## 命名规范

- **类名**: PascalCase (如 `GameObject`, `MeshRenderer`)
- **成员变量**: `m_` 前缀 + camelCase (如 `m_gameObject`, `m_isPlaying`)
- **局部变量**: camelCase
- **函数名**: PascalCase，动词开头
- **常量**: UPPER_CASE (如 `rhi_max_render_target_count`)
- **枚举**: PascalCase (如 `RHI_Format::R8_Unorm`)
- **命名空间**: PascalCase (`LitchiRuntime`, `LitchiEditor`)

## 代码风格

- **大括号**: 类和函数定义使用换行大括号
- **头文件组织**: 使用 `#pragma once`，包含顺序：标准库 → 第三方库 → 引擎头文件
- **RTTR 反射**: 所有组件类需要 RTTR 注册以支持编辑器序列化

```cpp
// 组件类示例
class MyComponent : public Component {
public:
    void OnUpdate() override;

private:
    float m_speed = 1.0f;
    RTTR_ENABLE(Component)
};

RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &MyComponent::m_speed);
}
```

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

**资源路径约定**:
- 引擎资源: 以 `:` 开头 (如 `:Textures/default.png`)
- 项目资源: 相对路径 (如 `Textures/my_texture.png`)

## 设计模式

| 模式 | 应用位置 |
|------|----------|
| 单例 | Renderer, RHI_Device, SceneManager |
| 服务定位器 | ServiceLocator |
| 组合 | GameObject-Component |
| 观察者 | Event 系统 |
| 策略 | RendererPath |
| 桥接 | RHI ↔ Vulkan 实现 |

## 关键文件路径

| 组件基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |
| 事件系统 | `Engine/Source/Runtime/Core/Tools/Eventing/` |
| RTTR 类型注册 | `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h` |

## 项目文档

### 引擎分析文档

详细分析文档位于 `docs/analysis/` 目录：

| 文档 | 内容概述 |
|------|----------|
| [01_core_module.md](docs/analysis/01_core_module.md) | Core 模块架构：数学库、反射系统、事件系统、窗口管理 |
| [02_rhi_abstraction.md](docs/analysis/02_rhi_abstraction.md) | RHI 抽象层设计：设备接口、命令列表、管线状态 |
| [03_vulkan_backend.md](docs/analysis/03_vulkan_backend.md) | Vulkan 后端实现：设备初始化、管线创建、着色器编译 |
| [04_rendering_pipeline.md](docs/analysis/04_rendering_pipeline.md) | 渲染管线：Shadow Pass、Forward Pass、PBR 材质 |
| [05_script_system.md](docs/analysis/05_script_system.md) | 脚本系统：Mono 运行时、C++/C# 互操作、生命周期 |
| [06_component_system.md](docs/analysis/06_component_system.md) | 组件系统：GameObject、Component、Transform |
| [07_scene_management.md](docs/analysis/07_scene_management.md) | 场景管理：Scene、SceneManager、Prefab |
| [08_resource_management.md](docs/analysis/08_resource_management.md) | 资源管理：ResourceManager、AssetManager、序列化 |
| [09_shader_system.md](docs/analysis/09_shader_system.md) | 着色器系统：HLSL、常量缓冲区、DXCompiler |
| [10_editor_architecture.md](docs/analysis/10_editor_architecture.md) | 编辑器架构：面板系统、Inspector、Hierarchy |
| [11_physics_system.md](docs/analysis/11_physics_system.md) | 物理系统：PhysX 集成、Collider、RigidActor |
| [12_animation_system.md](docs/analysis/12_animation_system.md) | 动画系统：骨骼动画、Animator、蒙皮渲染 |
| [13_ui_system.md](docs/analysis/13_ui_system.md) | UI 系统：ImGui 集成、UI 组件 |
| [14_code_standards.md](docs/analysis/14_code_standards.md) | 代码规范：命名约定、设计模式、扩展流程 |
| [15_catlike_coding_style.md](docs/analysis/15_catlike_coding_style.md) | Catlike Coding 教学风格调研 |
| [16_editor_play_mode.md](docs/analysis/16_editor_play_mode.md) | 编辑器运行模式：Play/Pause/Stop 状态机 |
| [17_shader_material_system.md](docs/analysis/17_shader_material_system.md) | 着色器与材质管理：Material、MaterialShader |
| [18_input_system.md](docs/analysis/18_input_system.md) | 输入系统：InputManager、键盘鼠标处理 |
| [19_serialization_system.md](docs/analysis/19_serialization_system.md) | 序列化系统：RTTR 反射、JSON 序列化、多态支持 |
| [20_application_framework.md](docs/analysis/20_application_framework.md) | 应用程序框架：生命周期、ServiceLocator |

### 引擎 Wiki

完整的引擎参考手册：[docs/wiki/README.md](docs/wiki/README.md)
