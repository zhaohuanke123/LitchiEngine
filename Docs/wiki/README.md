# LitchiEngine Wiki

> 荔枝引擎 - 个人学习开发的游戏引擎

---

## 目录

1. [快速入门](#1-快速入门)
2. [架构概览](#2-架构概览)
3. [核心系统](#3-核心系统)
   - [3.1 Core 模块](#31-core-模块)
   - [3.2 应用程序框架](#32-应用程序框架)
   - [3.3 输入系统](#33-输入系统)
   - [3.4 序列化系统](#34-序列化系统)
4. [渲染系统](#4-渲染系统)
   - [4.1 RHI 抽象层](#41-rhi-抽象层)
   - [4.2 Vulkan 后端](#42-vulkan-后端)
   - [4.3 渲染管线](#43-渲染管线)
   - [4.4 着色器系统](#44-着色器系统)
   - [4.5 材质系统](#45-材质系统)
5. [功能系统](#5-功能系统)
   - [5.1 组件系统](#51-组件系统)
   - [5.2 场景管理](#52-场景管理)
   - [5.3 脚本系统](#53-脚本系统)
   - [5.4 物理系统](#54-物理系统)
   - [5.5 动画系统](#55-动画系统)
   - [5.6 UI 系统](#56-ui-系统)
6. [资源管理](#6-资源管理)
7. [编辑器](#7-编辑器)
   - [7.1 编辑器架构](#71-编辑器架构)
   - [7.2 运行模式](#72-运行模式)
8. [开发指南](#8-开发指南)
   - [8.1 代码规范](#81-代码规范)
   - [8.2 扩展组件](#82-扩展组件)
   - [8.3 添加资源类型](#83-添加资源类型)
9. [常见问题](#9-常见问题)
10. [学习资源](#10-学习资源)

---

## 1. 快速入门

### 环境要求

- **操作系统**: Windows 10/11
- **编译器**: Visual Studio 2019+ 或兼容编译器
- **CMake**: 3.20+
- **Vulkan SDK**: 1.3+
- **Mono 运行时**: 用于 C# 脚本支持

### 构建步骤

```bash
# 生成项目文件并构建 (Debug)
cmake -B Build -DCMAKE_BUILD_TYPE=Debug
cmake --build Build --config Debug

# 或使用批处理脚本 (Windows)
GenerateProjects_CMake.bat
```

构建产物位于 `Build/bin/Debug` 或 `Build/bin/Release`。

### 项目结构

```
Engine/
├── Source/
│   ├── Runtime/        # 核心运行时
│   │   ├── Core/       # 核心系统
│   │   ├── Function/   # 功能模块
│   │   ├── Resource/   # 资源管理
│   │   └── Platform/   # 平台相关
│   ├── Editor/         # 编辑器
│   ├── Standalone/     # 独立运行程序
│   └── ScriptCore/     # C# 脚本核心
├── ThirdParty/         # 第三方依赖
├── Data/               # 引擎资源
└── Config/             # 配置文件
```

---

## 2. 架构概览

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        LitchiEditor                              │
│  (编辑器应用，ImGui UI，场景编辑，资源浏览)                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                      LitchiRuntime                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│  │    Core     │ │  Function   │ │  Resource   │                │
│  │  (基础设施)  │ │  (功能模块)  │ │  (资源管理)  │                │
│  └─────────────┘ └─────────────┘ └─────────────┘                │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                       RHI Layer                                  │
│  (渲染硬件抽象层，统一图形 API 接口)                              │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    Vulkan Backend                                │
│  (Vulkan 1.3 实现，VMA 内存管理，DXCompiler 着色器编译)           │
└─────────────────────────────────────────────────────────────────┘
```

### 核心设计原则

| 原则 | 说明 |
|------|------|
| **模块化** | 各系统独立，通过接口通信 |
| **数据驱动** | 配置和资源驱动行为 |
| **组合优于继承** | GameObject-Component 架构 |
| **抽象层设计** | RHI 屏蔽图形 API 差异 |

### 设计模式应用

| 模式 | 应用位置 |
|------|----------|
| 单例 | Renderer, RHI_Device, SceneManager |
| 服务定位器 | ServiceLocator |
| 组合 | GameObject-Component |
| 观察者 | Event 系统 |
| 策略 | RendererPath |
| 桥接 | RHI ↔ Vulkan 实现 |
| 模板方法 | Component 生命周期 |
| 工厂 | AResourceManager |

---

## 3. 核心系统

### 3.1 Core 模块

Core 模块是引擎的基础设施层，提供核心功能支持。

#### 数学库

**文件位置**: `Engine/Source/Runtime/Core/Math/`

| 类 | 功能 |
|-----|------|
| `Vector2/3/4` | 向量运算 |
| `Matrix` | 4x4 矩阵，列主序存储 |
| `Quaternion` | 四元数旋转 |
| `BoundingBox` | 轴对齐包围盒 |
| `Frustum` | 视锥体，用于剔除 |

**注意**: 矩阵使用 Column-major 内存布局，与 GPU 兼容。

#### 反射系统

基于 RTTR (Run-Time Type Reflection) 实现：

```cpp
// 类型注册示例
RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &MyComponent::m_speed);
}
```

#### 事件系统

```cpp
// 定义事件
Event<int, float> myEvent;

// 添加监听器
ListenerID id = myEvent.AddListener([](int a, float b) { /* ... */ });

// 触发事件
myEvent.Invoke(42, 3.14f);

// 移除监听器
myEvent.RemoveListener(id);
```

**详细文档**: [Core 模块分析](analysis/01_core_module.md)

### 3.2 应用程序框架

#### 生命周期

```
Initialize() → Run() (主循环) → Shutdown()
     │              │              │
     ▼              ▼              ▼
  初始化子系统    Update/Render   清理资源
```

#### 初始化顺序

```
路径初始化 → ConfigManager → Profiler → Debug → FileSystem
→ 资源管理器 → ServiceLocator注册 → Time → 导入器
→ Window → InputManager → Renderer → Physics → 事件注册
```

#### 主循环

```cpp
while (!window->ShouldClose()) {
    window->PollEvents();     // 处理窗口事件
    InputManager::Tick();     // 更新输入状态
    Update();                 // 游戏逻辑更新
    Render();                 // 渲染
    InputManager::ClearEvents(); // 清除帧内事件
    window->SwapBuffers();    // 交换缓冲区
}
```

#### 编辑器 vs 独立应用

| 特性 | Editor | Standalone |
|------|--------|------------|
| 项目选择 | ProjectHub 界面 | 直接加载 |
| 渲染视图 | 多视图 | 单视图 |
| UI 系统 | ImGui 编辑器 | 无 |
| 运行模式 | 编辑/播放/帧步进 | 仅播放 |

**详细文档**: [应用程序框架分析](analysis/20_application_framework.md)

### 3.3 输入系统

#### 两种查询方式

| 方式 | 方法 | 用途 |
|------|------|------|
| 事件驱动 | `IsKeyPressed()` | 一次性动作（跳跃、射击） |
| 状态轮询 | `GetKeyState()` | 持续动作（移动） |

#### 使用示例

```cpp
// 一次性动作：跳跃
if (InputManager::IsKeyPressed(EKey::KEY_SPACE)) {
    Jump();
}

// 持续动作：移动
if (InputManager::GetKeyState(EKey::KEY_W) == EKeyState::KEY_DOWN) {
    MoveForward();
}

// 鼠标位置
Vector2 mousePos = InputManager::GetMousePosition();
Vector2 mouseDelta = InputManager::GetMouseDelta();
```

**详细文档**: [输入系统分析](analysis/18_input_system.md)

### 3.4 序列化系统

#### 架构

```
应用层 (GameObject, Material, Scene)
         │
         ▼
序列化层 (Serializer: JSON 读写)
         │
         ▼
反射层 (RTTR: 运行时类型信息)
```

#### 使用方式

```cpp
// 序列化到 JSON
std::string json = Serializer::SerializeToJson(object);

// 从 JSON 反序列化
MyClass obj;
Serializer::DeserializeFromJson(json, obj);
```

#### 可序列化类型要求

1. 使用 `RTTR_ENABLE()` 宏启用反射
2. 注册类型和属性
3. 多态类型添加 `Polymorphic` 元数据

```cpp
class MyComponent : public Component {
public:
    float m_speed = 1.0f;
    RTTR_ENABLE(Component)
};

RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &MyComponent::m_speed);
}
```

**详细文档**: [序列化系统分析](analysis/19_serialization_system.md)

---

## 4. 渲染系统

### 4.1 RHI 抽象层

RHI (Render Hardware Interface) 是渲染硬件抽象层，屏蔽不同图形 API 的差异。

#### 核心接口

```cpp
class RHI_Device {
public:
    static void Initialize();
    static void Tick(const uint64_t frame_count);
    static void Destroy();

    // 队列操作
    static void QueueSubmit(RHI_Queue_Type type, ...);

    // 描述符管理
    static void AllocateDescriptorSet(void*& resource, ...);

    // 管线管理 (带缓存)
    static void GetOrCreatePipeline(RHI_PipelineState& pso, ...);
};
```

**详细文档**: [RHI 抽象层分析](analysis/02_rhi_abstraction.md)

### 4.2 Vulkan 后端

#### 特性

- Vulkan 1.3 API
- 动态渲染 (VK_KHR_dynamic_rendering)
- VMA 内存管理
- DXCompiler 着色器编译 (HLSL → SPIR-V)
- Bindless 资源绑定

#### 管线缓存

使用哈希键缓存 Pipeline 对象，避免重复创建：

```cpp
size_t hash = ComputePipelineHash(pso);
if (m_pipelineCache.find(hash) != m_pipelineCache.end()) {
    return m_pipelineCache[hash];
}
// 创建新管线...
```

**详细文档**: [Vulkan 后端分析](analysis/03_vulkan_backend.md)

### 4.3 渲染管线

#### 渲染流程

```
ShadowMap Pass → SkyBox Pass → Forward Pass → UI Pass → Debug Pass
```

#### Pass 说明

| Pass | 功能 |
|------|------|
| ShadowMap | 从光源视角渲染深度图 |
| SkyBox | 渲染天空盒 |
| Forward | 前向渲染，PBR 材质 |
| UI | ImGui UI 渲染 |
| Debug | 编辑器调试绘制 |

**详细文档**: [渲染管线分析](analysis/04_rendering_pipeline.md)

### 4.4 着色器系统

#### 常量缓冲区布局

| Buffer | 绑定点 | 用途 |
|--------|--------|------|
| Frame | b0 | 全局帧数据 |
| Camera | b1 | 相机参数 |
| Light | b2 | 光源数据 |
| Material | b10 | 材质属性 |

#### PBR 材质实现

- GGX/Trowbridge-Reitz 法线分布函数
- Schlick Fresnel 近似
- Smith 几何遮蔽函数

**详细文档**: [着色器系统分析](analysis/09_shader_system.md)

### 4.5 材质系统

#### Material vs MaterialShader

| 类 | 职责 |
|-----|------|
| `MaterialShader` | 着色器组合、描述符反射、类型信息 |
| `Material` | 实例数据、Uniform 值、纹理绑定 |

#### Uniform 数据流

```
编辑器 SetValue → m_uniformDataList → SyncToDataBuffer → GPU
```

**详细文档**: [着色器与材质系统分析](analysis/17_shader_material_system.md)

---

## 5. 功能系统

### 5.1 组件系统

#### 类继承关系

```
Object
  └── ScriptObject
        ├── GameObject (游戏对象容器)
        └── Component (组件基类)
              ├── Transform
              ├── MeshRenderer
              ├── Camera
              ├── Light
              ├── ScriptComponent
              └── ...
```

#### 组件生命周期

```
OnAwake() → OnEnable() → OnStart() → OnUpdate() → OnDisable() → OnDestroy()
```

#### 添加新组件

1. 继承 `Component` 或其子类
2. 重写生命周期方法
3. 添加 RTTR 注册
4. 在 `Engine/Source/Runtime/CMakeLists.txt` 添加源文件

**详细文档**: [组件系统分析](analysis/06_component_system.md)

### 5.2 场景管理

#### Scene 类职责

- 管理 GameObject 列表
- 分配唯一 ID
- 驱动组件生命周期

#### Prefab 系统

使用原型模式实现深拷贝实例化：

```cpp
GameObject* instance = Prefab::Instantiate();
```

**详细文档**: [场景管理分析](analysis/07_scene_management.md)

### 5.3 脚本系统

#### 架构

```
ScriptEngine (引擎入口)
    ├── ScriptClass (脚本类定义)
    │       └── MonoClass* (Mono运行时句柄)
    └── ScriptInstance (脚本实例)
            ├── MonoObject* (托管对象)
            └── ScriptClass* (类定义)
```

#### 脚本生命周期

```
OnAwake() → OnEnable() → OnStart() → OnUpdate() → OnDisable() → OnDestroy()
```

**详细文档**: [脚本系统分析](analysis/05_script_system.md)

### 5.4 物理系统

基于 PhysX 实现：

- `RigidActor`: 刚体组件
- `Collider`: 碰撞体组件
- `Physics`: 物理场景管理

**详细文档**: [物理系统分析](analysis/11_physics_system.md)

### 5.5 动画系统

- `Animator`: 动画控制器组件
- `SkinnedMeshRenderer`: 蒙皮网格渲染
- 支持骨骼动画和动画混合

**详细文档**: [动画系统分析](analysis/12_animation_system.md)

### 5.6 UI 系统

基于 ImGui 实现：

- `UICanvas`: UI 画布
- `UIImage`: 图像组件
- `UIText`: 文本组件
- `UIButton`: 按钮组件

**详细文档**: [UI 系统分析](analysis/13_ui_system.md)

---

## 6. 资源管理

### 资源管理器模板

```cpp
template <typename T>
class AResourceManager {
public:
    std::shared_ptr<T> Load(const std::string& path);
    void Unload(const std::string& path);
    void Reload(const std::string& path);
};
```

### 资源路径约定

| 路径格式 | 说明 | 示例 |
|----------|------|------|
| `:Path` | 引擎内置资源 | `:Textures/default.png` |
| `Path` | 项目资源 | `Textures/my_texture.png` |

**详细文档**: [资源管理分析](analysis/08_resource_management.md)

---

## 7. 编辑器

### 7.1 编辑器架构

#### 面板系统

| 面板 | 功能 |
|------|------|
| Inspector | 属性检视 |
| Hierarchy | 层级视图 |
| SceneView | 场景编辑 |
| GameView | 游戏预览 |
| AssetBrowser | 资源浏览 |

#### Inspector 反射集成

使用 RTTR 反射自动绘制属性编辑器：

```cpp
// 自动遍历属性并绘制
for (auto& prop : type.get_properties()) {
    DrawProperty(prop, instance);
}
```

**详细文档**: [编辑器架构分析](analysis/10_editor_architecture.md)

### 7.2 运行模式

#### 编辑器状态机

```
EDIT ──StartPlaying──▶ PLAY ──PauseGame──▶ PAUSE
  ▲                       │                   │
  │                       │StopPlaying        │StopPlaying
  └───────────────────────┴───────────────────┘
```

#### 场景备份机制

播放前序列化场景为 JSON，停止时反序列化恢复。

**详细文档**: [编辑器运行模式分析](analysis/16_editor_play_mode.md)

---

## 8. 开发指南

### 8.1 代码规范

#### 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `GameObject`, `MeshRenderer` |
| 成员变量 | `m_` 前缀 + camelCase | `m_gameObject`, `m_isPlaying` |
| 局部变量 | camelCase | `position`, `deltaTime` |
| 函数名 | PascalCase，动词开头 | `GetTransform`, `Update` |
| 常量 | UPPER_CASE | `rhi_max_render_target_count` |
| 枚举 | PascalCase | `RHI_Format::R8_Unorm` |

#### 文件组织

- 使用 `#pragma once`
- 包含顺序：标准库 → 第三方库 → 引擎头文件

**详细文档**: [代码规范总结](analysis/14_code_standards.md)

### 8.2 扩展组件

```cpp
// MyComponent.h
class MyComponent : public Component {
public:
    void OnUpdate() override;

private:
    float m_speed = 1.0f;
    RTTR_ENABLE(Component)
};

// MyComponent.cpp
RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &MyComponent::m_speed);
}

void MyComponent::OnUpdate() {
    // 每帧更新逻辑
}
```

### 8.3 添加资源类型

1. 继承 `IResource` 基类
2. 创建对应的资源管理器
3. 添加 RTTR 注册
4. 实现序列化支持

---

## 9. 常见问题

### Q: 编译失败，找不到 Vulkan SDK

确保已安装 Vulkan SDK 并设置环境变量 `VULKAN_SDK`。

### Q: 运行时崩溃，无法加载场景

检查资源路径是否正确，引擎资源以 `:` 开头。

### Q: 脚本组件无法加载

确保 Mono 运行时已安装，且 C# 脚本已正确编译为 DLL。

### Q: 着色器编译失败

检查 HLSL 语法，确保使用正确的着色器模型版本。

### Q: 如何添加新的输入类型？

1. 在对应枚举文件中添加新类型
2. 在 InputManager 中添加事件回调
3. 提供查询接口

---

## 10. 学习资源

### 教程文档

- [引擎原理学习教程](homework/engine_principle_tutorials.md) - 通过实践学习引擎原理
- [Catlike Coding 风格教程](homework/catlike_style_tutorials.md) - 渐进式教程设计
- [实习生扩展开发作业](homework/intern_assignments.md) - 扩展开发练习

### 分析文档索引

| 文档 | 内容 |
|------|------|
| [Core 模块](analysis/01_core_module.md) | 数学库、反射、事件系统 |
| [RHI 抽象层](analysis/02_rhi_abstraction.md) | 渲染硬件接口设计 |
| [Vulkan 后端](analysis/03_vulkan_backend.md) | Vulkan 实现细节 |
| [渲染管线](analysis/04_rendering_pipeline.md) | Forward 渲染、阴影、PBR |
| [脚本系统](analysis/05_script_system.md) | Mono/C# 脚本集成 |
| [组件系统](analysis/06_component_system.md) | GameObject-Component 架构 |
| [场景管理](analysis/07_scene_management.md) | Scene、Prefab 系统 |
| [资源管理](analysis/08_resource_management.md) | 资源加载、缓存、序列化 |
| [着色器系统](analysis/09_shader_system.md) | HLSL、常量缓冲区 |
| [编辑器架构](analysis/10_editor_architecture.md) | 面板系统、ImGui 集成 |
| [物理系统](analysis/11_physics_system.md) | PhysX 集成 |
| [动画系统](analysis/12_animation_system.md) | 骨骼动画 |
| [UI 系统](analysis/13_ui_system.md) | ImGui UI 组件 |
| [代码规范](analysis/14_code_standards.md) | 命名、格式、最佳实践 |
| [Catlike Coding 风格](analysis/15_catlike_coding_style.md) | 教程设计方法论 |
| [编辑器运行模式](analysis/16_editor_play_mode.md) | Play/Pause/Stop 状态机 |
| [着色器与材质系统](analysis/17_shader_material_system.md) | Material、MaterialShader |
| [输入系统](analysis/18_input_system.md) | 键盘、鼠标输入处理 |
| [序列化系统](analysis/19_serialization_system.md) | JSON 序列化、RTTR 集成 |
| [应用程序框架](analysis/20_application_framework.md) | 生命周期、ServiceLocator |

---

## 更新日志

- **2026-04-14**: 完成 Task #21-24，生成完整 Wiki 文档
- **2026-04-12**: 完成 Task #19-20，分析编辑器运行模式和材质系统
- **2026-04-11**: 完成 Task #16-18，调研 Catlike Coding 风格，设计教程
- **2026-04-10**: 完成 Task #1-15，完成所有核心系统分析

---

*LitchiEngine - 一个用于学习的游戏引擎项目*
