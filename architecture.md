# Project Architecture

## Overview

- **Product goal**: LitchiEngine (荔枝引擎) 是一个个人学习开发的游戏引擎，使用 C++17 编写，主要面向 Windows 平台，目标是提供一个完整的学习和实验平台。
- **Primary users**: 学习游戏引擎开发的开发者，特别是学过 Games101/Games104 的实习生。
- **Success criteria**: 能够成功编译运行，支持场景编辑、脚本编写、渲染预览等核心功能。

## Constraints

- **Technical constraints**:
  - C++17 标准
  - Windows 平台优先
  - Vulkan 1.3 图形 API
  - Mono 运行时 (C# 脚本)

- **Business constraints**:
  - 个人学习项目，无商业化压力
  - 代码质量和可读性优先

- **External dependencies**:
  - Vulkan SDK
  - PhysX (物理引擎)
  - Mono (C# 运行时)
  - ImGui (UI 框架)
  - Assimp (模型加载)
  - FreeImage (图像加载)
  - spdlog (日志)
  - GLFW (窗口管理)
  - RTTR (运行时反射)

## Stack

- **Frontend**: ImGui (编辑器 UI)
- **Backend**: C++17 核心运行时
- **Data/storage**: JSON 序列化，文件系统资源管理
- **Auth**: 无
- **Deployment**: 本地构建，CMake 生成项目文件

## Core Flows

1. **Main user flow**:
   - 启动编辑器 → 创建/加载场景 → 添加 GameObject → 添加组件 → 编辑属性 → 运行游戏

2. **Failure and retry flow**:
   - 编译错误 → 检查 CMake 配置 → 重新生成项目
   - 运行时错误 → 检查日志 → 修复问题

3. **Admin or support flow**:
   - 开发者模式：直接修改引擎源码 → 重新编译 → 测试

## Data and State

- **Core entities**:
  - `GameObject`: 游戏对象容器
  - `Component`: 组件基类 (Transform, MeshRenderer, Camera, Light, ScriptComponent 等)
  - `Scene`: 场景管理
  - `Material`: 材质资源
  - `Mesh`: 网格资源
  - `Prefab`: 预制体

- **State transitions**:
  - GameObject: Created → Active → Inactive → Destroyed
  - Component: Added → Enabled → Disabled → Removed
  - Scene: Unloaded → Loading → Loaded → Playing → Stopped

- **Long-running jobs**: 无

## Interfaces

- **API routes or service boundaries**:
  - RHI (Render Hardware Interface): 渲染硬件抽象层
  - ScriptEngine: 脚本引擎接口
  - SceneManager: 场景管理接口
  - AssetManager: 资产管理接口

- **External APIs**: 无

- **Webhooks, queues, or schedulers**: 无

## Module Architecture

### 1. Core 模块 (`Engine/Source/Runtime/Core/`)
基础设施层，提供引擎运行的基础设施。

| 子目录 | 职责 | 关键类 |
|--------|------|--------|
| `App/` | 应用程序入口 | `ApplicationBase` |
| `Global/` | 全局服务定位 | `ServiceLocator` |
| `Meta/Reflection/` | 运行时反射 | `Object`, RTTR 集成 |
| `Meta/Serializer/` | 序列化系统 | `Serializer` |
| `Math/` | 数学库 | `Vector3`, `Matrix`, `Quaternion`, `BoundingBox`, `Frustum` |
| `Tools/Eventing/` | 事件系统 | `Event` |
| `Window/` | 窗口管理 | `Window`, `InputManager` |

### 2. Function 模块 (`Engine/Source/Runtime/Function/`)
功能实现层，实现引擎的核心功能。

| 子目录 | 职责 | 关键类 |
|--------|------|--------|
| `Framework/` | GameObject-Component 框架 | `GameObject`, `Component`, `Transform` |
| `Renderer/` | 渲染系统 | `Renderer`, `RendererPath`, `RHI_Device` |
| `Scripting/` | 脚本系统 | `ScriptEngine`, `ScriptClass`, `ScriptInstance` |
| `Scene/` | 场景管理 | `Scene`, `SceneManager` |
| `UI/` | UI 框架 | ImGui 集成 |
| `Prefab/` | 预制体系统 | `Prefab` |

### 3. Resource 模块 (`Engine/Source/Runtime/Resource/`)
资源管理层，管理各类资源的加载、缓存和生命周期。

| 类 | 职责 |
|----|------|
| `AResourceManager<T>` | 资源管理器模板基类 |
| `AssetManager` | 资产序列化/反序列化 |
| `ModelManager` | 模型资源管理 |
| `TextureManager` | 纹理资源管理 |
| `MaterialManager` | 材质资源管理 |
| `ShaderManager` | 着色器资源管理 |

### 4. Platform 模块 (`Engine/Source/Runtime/Platform/`)
平台抽象层，处理平台相关的差异。

## Rendering System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Renderer (高层渲染器)                    │
│  - 管理渲染资源 (Shaders, Textures, Buffers)                 │
│  - 渲染管线调度 (RendererPath)                                │
│  - 渲染Pass (Forward, Shadow, SkyBox, UI, Debug)            │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    RendererPath (渲染路径)                    │
│  - SceneView, GameView, AssetView                            │
│  - 视锥剔除 (Frustum Culling)                                │
│  - 渲染目标管理 (RenderTarget)                               │
│  - 光源数据管理 (LightGroup)                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    RHI_Device (设备抽象层)                    │
│  - 队列管理 (Graphics, Compute, Copy)                        │
│  - 描述符池管理 (Bindless Descriptors)                       │
│  - 管线缓存 (Pipeline Cache)                                 │
│  - 内存管理 (VMA集成)                                        │
│  - 命令池管理 (CommandPool)                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Vulkan Backend                            │
│  - Vulkan_Device.cpp (设备初始化)                             │
│  - Vulkan_Pipeline.cpp, Vulkan_Shader.cpp, etc.             │
│  - VMA (Vulkan Memory Allocator)                            │
└─────────────────────────────────────────────────────────────┘
```

### RHI 核心接口

```cpp
class RHI_Device {
public:
    // 生命周期
    static void Initialize();
    static void Tick(const uint64_t frame_count);
    static void Destroy();

    // 队列操作
    static void QueueSubmit(RHI_Queue_Type type, ...);
    static void QueuePresent(void* swapchain, ...);

    // 描述符管理
    static void CreateDescriptorPool();
    static void AllocateDescriptorSet(void*& resource, ...);

    // 管线管理 (带缓存)
    static void GetOrCreatePipeline(RHI_PipelineState& pso,
                                     RHI_Pipeline*& pipeline,
                                     RHI_DescriptorSetLayout*& layout);
};
```

### 渲染流程

1. **Shadow Map Pass**: 从光源视角渲染深度图
2. **SkyBox Pass**: 渲染天空盒
3. **Forward Pass**: 前向渲染 (PBR材质)
4. **UI Pass**: 渲染UI元素
5. **Debug Pass**: 编辑器调试绘制

## Script System Architecture

### Mono/C# 脚本绑定

```
ScriptEngine (引擎入口)
    ├── ScriptClass (脚本类定义)
    │       └── MonoClass* (Mono运行时句柄)
    └── ScriptInstance (脚本实例)
            ├── MonoObject* (托管对象)
            └── ScriptClass* (类定义)
```

### C++ 与 C# 对象映射

- C++ 端: `ScriptObject` 包含 `m_unmanagedId`
- C# 端: `ScriptObject` 包含对应的 `m_umanagedId`
- 通过 `InternalCalls` 进行双向通信

### 脚本生命周期

```
OnAwake() → OnEnable() → OnStart() → OnUpdate() → OnDisable() → OnDestroy()
```

## Component System Architecture

### 类继承关系

```
Object
  └── ScriptObject (可被脚本引用)
        ├── GameObject (游戏对象容器)
        └── Component (组件基类)
              ├── Transform
              ├── MeshFilter
              ├── MeshRenderer
              ├── SkinnedMeshRenderer
              ├── Camera
              ├── Light
              ├── ScriptComponent
              ├── Collider
              ├── RigidActor
              ├── Animator
              └── UI组件
```

### 关键组件

| 组件 | 职责 | 文件位置 |
|------|------|----------|
| `Transform` | 位置/旋转/缩放，层级管理 | `Framework/Component/Transform/` |
| `MeshRenderer` | 网格渲染 | `Framework/Component/Renderer/` |
| `Camera` | 相机 | `Framework/Component/Camera/` |
| `Light` | 光源 | `Framework/Component/Light/` |
| `ScriptComponent` | 脚本组件 | `Framework/Component/Script/` |

## Design Patterns

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | ApplicationBase, Renderer, RHI_Device | 全局访问点 |
| 服务定位器 | ServiceLocator | 解耦服务依赖 |
| 组合模式 | GameObject-Component | 灵活的对象组合 |
| 观察者模式 | Event系统 | 解耦事件通知 |
| 模板方法模式 | Component生命周期 | 统一流程，子类扩展 |
| 工厂模式 | AResourceManager | 资源创建抽象 |
| 策略模式 | RendererPath | 不同渲染路径切换 |
| 代理模式 | ScriptObject | C++/C#对象桥接 |
| 缓存模式 | Pipeline缓存, DescriptorSet缓存 | 减少GPU资源创建开销 |

## Validation

- **Required automated checks**: CMake 构建成功
- **Required browser or manual checks**: 编辑器启动，场景加载，渲染正常
- **Blockers that require human setup**: Vulkan SDK 安装，Mono 安装

## Key File Paths

| 模块 | 文件路径 |
|------|----------|
| 应用入口 | `Engine/Source/Runtime/Core/App/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |
| 反射系统 | `Engine/Source/Runtime/Core/Meta/Reflection/` |
| RHI 接口 | `Engine/Source/Runtime/Function/Renderer/RHI/` |
| Vulkan 后端 | `Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/` |
| 渲染器 | `Engine/Source/Runtime/Function/Renderer/Rendering/` |
| 脚本引擎 | `Engine/Source/Runtime/Function/Scripting/` |
| GameObject | `Engine/Source/Runtime/Function/Framework/GameObject/` |
| Component | `Engine/Source/Runtime/Function/Framework/Component/` |
| 场景管理 | `Engine/Source/Runtime/Function/Scene/` |
| 资源管理 | `Engine/Source/Runtime/Resource/` |
| 着色器 | `Engine/Data/Engine/Shaders/` |
| C# 脚本核心 | `Engine/Source/ScriptCore/` |
| 编辑器 | `Engine/Source/Editor/` |
