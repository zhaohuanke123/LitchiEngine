# LitchiEngine Catlike Coding 风格教程系列

## 系列概述

本教程系列采用 Catlike Coding 的教学风格，通过渐进式、问题驱动的方式引导实习生学习 LitchiEngine 的核心系统。

### 教学原则

| 原则 | 说明 |
|------|------|
| 渐进式构建 | 从最小可行实现开始，每步只添加一个功能 |
| 问题驱动 | 先提出问题/需求，再解释解决方案 |
| 增量展示 | 不展示完整文件，只展示变化部分 |
| 即时验证 | 每个改动后立即展示预期结果 |
| 概念分层 | What/Why/How 三层解释 |

### 系列规划

| 系列 | 教程 | 难度 | 前置要求 |
|------|------|------|----------|
| 引擎基础 | 教程 1: 组件系统入门 | 入门 | 无 |
| 渲染原理 | 教程 2: 追踪一次 Draw Call | 进阶 | 教程 1 |

### 开发环境要求

- Visual Studio 2019 或更高版本
- CMake 3.16+
- 熟悉 C++17 基础语法
- 了解面向对象编程概念

---

# 教程 1：组件系统入门 - RotateComponent

## 概述

通过创建一个简单的旋转组件，学习 LitchiEngine 的组件系统架构。完成后，你将理解：

- 组件的基本结构
- 组件生命周期回调
- RTTR 反射注册
- Transform 操作

**预期成果**：一个能让物体持续旋转的 RotateComponent。

## 前置要求

- 已成功编译 LitchiEngine
- 熟悉 C++ 类继承概念
- 了解游戏引擎的基本概念（GameObject、Component）

---

## 第一节：理解组件架构

### 什么是组件？

在 LitchiEngine 中，GameObject 是场景中的实体，而 Component 是挂载在 GameObject 上的功能模块。一个 GameObject 可以有多个 Component，每个 Component 负责特定的功能。

```
GameObject (Cube)
    ├── Transform        // 位置、旋转、缩放
    ├── MeshFilter       // 网格数据
    ├── MeshRenderer     // 渲染外观
    └── RotateComponent  // 我们将创建的组件
```

### 组件基类

所有组件都继承自 `Component` 基类。让我们先了解它提供了什么。

**文件路径**: `Engine/Source/Runtime/Function/Framework/Component/Base/component.h`

```cpp
class Component : public ScriptObject {
public:
    // 获取所属的 GameObject
    GameObject* GetGameObject() const { return m_gameObject; }

    // 生命周期回调
    virtual void OnAwake();      // 场景开始前调用
    virtual void OnEnable();     // 组件启用时调用
    virtual void OnStart();      // 场景开始时调用
    virtual void OnUpdate();     // 每帧调用
    virtual void OnFixedUpdate(); // 物理帧调用
    virtual void OnDisable();    // 组件禁用时调用
    virtual void OnDestroy();    // 组件销毁时调用

    // 物理回调
    virtual void OnTriggerEnter(GameObject* game_object);
    virtual void OnTriggerExit(GameObject* game_object);

    RTTR_ENABLE(ScriptObject)  // 启用 RTTR 反射
};
```

### 为什么使用 RTTR？

RTTR (Run Time Type Reflection) 是一个 C++ 反射库，它让编辑器能够：

- 发现组件类型
- 在 Inspector 中显示属性
- 序列化/反序列化组件

---

## 第二节：创建最小组件

### 第一步：创建头文件

在 `Engine/Source/Runtime/Function/Framework/Component/Gameplay/` 目录下创建 `RotateComponent.h`：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"

namespace LitchiRuntime
{
    class RotateComponent : public Component
    {
    public:
        RotateComponent() = default;
        ~RotateComponent() override = default;

        RTTR_ENABLE(Component)
    };
}
```

**验证**: 此时项目应该能编译通过，但这个组件还没有任何功能。

### 第二步：添加生命周期方法

现在添加 `OnUpdate` 方法，它会在每一帧被调用。

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"

namespace LitchiRuntime
{
    class RotateComponent : public Component
    {
    public:
        RotateComponent() = default;
        ~RotateComponent() override = default;

        // 新增：每帧调用
        void OnUpdate() override;

        RTTR_ENABLE(Component)
    };
}
```

### 第三步：创建实现文件

创建 `RotateComponent.cpp`：

```cpp
#include "RotateComponent.h"

namespace LitchiRuntime
{
    void RotateComponent::OnUpdate()
    {
        // 暂时什么都不做
    }
}
```

**注意**：RTTR 注册不放在这里，而是统一放在 `TypeRegister.h` 中。

### 第四步：在 TypeRegister.h 中注册

**文件路径**: `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h`

在 `Framework Object Types` 区域添加：

```cpp
rttr::registration::class_<RotateComponent>("RotateComponent")
    .constructor<>()(rttr::policy::ctor::as_raw_ptr);
```

**验证**: 重新编译项目，应该成功通过。此时组件已可在编辑器中使用，但还没有功能。

---

## 第三节：实现旋转功能

### 问题：如何让物体旋转？

要让物体旋转，我们需要：
1. 获取物体的 Transform 组件
2. 每帧修改旋转值
3. 需要知道帧间隔时间（deltaTime）以保证旋转速度稳定

### 理解 Transform

Transform 是每个 GameObject 必有的组件，管理位置、旋转、缩放。

**文件路径**: `Engine/Source/Runtime/Function/Framework/Component/Transform/transform.h`

关键方法：

```cpp
class Transform : public Component {
public:
    // 旋转相关
    Quaternion GetRotation() const;           // 世界旋转
    Quaternion GetRotationLocal() const;      // 本地旋转
    void SetRotation(const Quaternion& rot);
    void SetRotationLocal(const Quaternion& rot);
    void Rotate(const Quaternion& delta);     // 增量旋转
};
```

### 第五步：获取 Transform

更新 `RotateComponent.cpp`：

```cpp
#include "RotateComponent.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"

namespace LitchiRuntime
{
    void RotateComponent::OnUpdate()
    {
        // 获取 Transform 组件
        Transform* transform = GetGameObject()->GetTransform();
        if (!transform) return;

        // TODO: 实现旋转
    }
}
```

**为什么需要空指针检查？**

虽然每个 GameObject 都有 Transform，但养成防御性编程的习惯很重要。如果组件被错误地使用，程序不会崩溃。

### 第六步：添加旋转速度属性

回到头文件，添加可配置的旋转速度：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class RotateComponent : public Component
    {
    public:
        RotateComponent() = default;
        ~RotateComponent() override = default;

        // 新增：旋转速度（度/秒）
        Vector3 rotationSpeed{0.0f, 45.0f, 0.0f};

        void OnUpdate() override;

        RTTR_ENABLE(Component)
    };
}
```

**Vector3 说明**：`rotationSpeed{0.0f, 45.0f, 0.0f}` 表示每秒绕 Y 轴旋转 45 度。

### 第七步：实现旋转逻辑

更新实现文件：

```cpp
#include "RotateComponent.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"
#include "Runtime/Core/Time/Time.h"
#include "Runtime/Core/Math/Quaternion.h"
#include "Runtime/Core/Math/MathHelper.h"

namespace LitchiRuntime
{
    void RotateComponent::OnUpdate()
    {
        Transform* transform = GetGameObject()->GetTransform();
        if (!transform) return;

        float dt = Time::GetDeltaTime();

        // 当前旋转
        Quaternion currentRot = transform->GetRotationLocal();

        // 计算这一帧的旋转增量（绕 Y 轴）
        float angleRadians = rotationSpeed.y * Math::Helper::DEG_TO_RAD * dt;
        Quaternion deltaRot = Quaternion::FromAngleAxis(angleRadians, Vector3::Up);

        // 应用旋转
        transform->SetRotationLocal(currentRot * deltaRot);
    }
}
```

**代码详解**：

| 行 | 说明 |
|----|------|
| `Time::GetDeltaTime()` | 获取上一帧到这一帧的时间（秒） |
| `Math::Helper::DEG_TO_RAD` | 度到弧度的转换系数 |
| `Quaternion::FromAngleAxis()` | 从轴角创建四元数 |
| `currentRot * deltaRot` | 四元数乘法，组合旋转 |

**为什么使用四元数？**

四元数避免了欧拉角的万向锁问题，且插值更平滑。旋转组合只需简单乘法。

### 第八步：注册属性到 RTTR

**重要**：LitchiEngine 的 RTTR 注册统一放在 `TypeRegister.h` 中，而不是每个组件的 cpp 文件。

**文件路径**: `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h`

找到 `Framework Object Types` 区域，添加注册：

```cpp
rttr::registration::class_<RotateComponent>("RotateComponent")
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("rotationSpeed", &RotateComponent::rotationSpeed);
```

**为什么放在 TypeRegister.h？**

| 原因 | 说明 |
|------|------|
| 集中管理 | 所有组件注册在一处，便于维护 |
| 编译优化 | 减少编译单元，加快编译速度 |
| 依赖清晰 | 避免循环依赖问题 |

**as_raw_ptr 策略说明**：

| 策略 | 创建方式 | 适用场景 |
|------|----------|----------|
| 默认（无策略） | 栈对象 | 不适合引擎组件 |
| `as_raw_ptr` | 原始指针（堆） | **引擎组件必须** |

**如果不使用 as_raw_ptr 会怎样？**

场景序列化/反序列化时会崩溃：
1. Play → Stop 切换时场景恢复失败
2. Prefab 实例化时组件创建失败

`.property()` 让编辑器能在 Inspector 面板中显示和编辑这个属性。

---

## 第五节：在 Inspector 中注册组件

### 问题：组件如何出现在 "Add Component" 列表中？

RTTR 注册让组件可被序列化，但还需要在 Inspector 面板中添加入口。

### 第九步：添加头文件引用

**文件路径**: `Engine/Source/Editor/source/Panels/Inspector.cpp`

在文件顶部的 include 区域添加：

```cpp
#include "Runtime/Function/Framework/Component/Gameplay/RotateComponent.h"
```

### 第十步：添加组件选择器选项

找到 `componentSelectorWidget.choices` 的定义位置（约第 69 行），添加：

```cpp
componentSelectorWidget.choices.emplace(18, "RotateComponent");
```

**注意**：数字 18 是选项的索引，确保不与已有索引重复。

### 第十一步：添加组件创建逻辑

在 `addComponentButton.ClickedEvent` 的 switch 语句中添加：

```cpp
case 18: GetTargetActor()->AddComponent<RotateComponent>(); break;
```

### 第十二步：添加按钮状态检查

在 `componentSelectorWidget.ValueChangedEvent` 的 switch 语句中添加：

```cpp
case 18: defineButtonsStates(GetTargetActor()->GetComponent<RotateComponent>()); return;
```

**完整修改位置**：

| 位置 | 作用 |
|------|------|
| 头文件 include | 让编译器知道 RotateComponent 类型 |
| choices.emplace | 在下拉列表中显示选项 |
| switch case (ClickedEvent) | 点击按钮时创建组件 |
| switch case (ValueChangedEvent) | 检查组件是否已存在，禁用按钮 |

---

## 第六节：测试与验证

### 测试步骤

1. 重新编译引擎
2. 打开 LitchiEditor
3. 在场景中创建一个 Cube
4. 在 Inspector 中点击 "Add Component"
5. 选择 "RotateComponent"
6. 运行场景

### 预期效果

- Cube 应该持续绕 Y 轴旋转
- 修改 `rotationSpeed.y` 可以改变旋转速度
- 修改 `rotationSpeed.x` 或 `z` 可以改变旋转轴

### 常见问题诊断

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 组件不显示在列表中 | RTTR 注册失败 | 检查 TypeRegister.h 中的注册 |
| 物体不旋转 | OnUpdate 未被调用 | 检查 GameObject 是否激活 |
| 旋转速度异常 | deltaTime 问题 | 确认 Time::GetDeltaTime() 正常 |
| Play/Stop 崩溃 | 缺少 as_raw_ptr | 添加 `(rttr::policy::ctor::as_raw_ptr)` |
| 组件属性不显示 | 未注册 property | 添加 `.property("name", &Class::member)` |

---


## 验证标准

- [ ] 编译通过，无警告
- [ ] RotateComponent 出现在编辑器的组件列表中
- [ ] 能成功添加组件到 GameObject
- [ ] Inspector 中显示 rotationSpeed 属性
- [ ] 运行时物体正确旋转
- [ ] 修改属性值能影响旋转行为
- [ ] Play/Stop 切换不崩溃（as_raw_ptr 正确配置）

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| Component 基类 | 所有组件的父类，提供生命周期回调 |
| OnUpdate() | 每帧调用的方法，用于实现游戏逻辑 |
| Transform | 控制物体的位置、旋转、缩放 |
| Quaternion | 表示旋转，避免万向锁 |
| RTTR 注册 | 在 TypeRegister.h 中统一注册，让编辑器识别组件 |
| as_raw_ptr | RTTR 构造策略，组件必须使用 |
| Time::GetDeltaTime() | 帧无关动画的关键 |
| Inspector 注册 | 在 Inspector.cpp 中添加组件到下拉列表 |

### 创建新组件的完整流程

1. **创建头文件** - 继承 Component，添加 RTTR_ENABLE
2. **创建实现文件** - 实现生命周期方法
3. **RTTR 注册** - 在 TypeRegister.h 中注册类和属性
4. **Inspector 注册** - 在 Inspector.cpp 中添加下拉选项和创建逻辑
5. **编译测试** - 验证组件出现在列表中且功能正常

---

## 延伸：C# 脚本 vs C++ 组件

### 问题：为什么 C# 脚本不需要手动注册？

你可能注意到，在 Unity 或其他使用 C# 的引擎中，创建脚本只需要：

```csharp
// C# 脚本 - 不需要注册！
public class RotateScript : MonoBehaviour
{
    public Vector3 rotationSpeed;  // 直接定义，自动显示在 Inspector
    
    void Update()
    {
        transform.Rotate(rotationSpeed * Time.deltaTime);
    }
}
```

而在 LitchiEngine 的 C++ 中，我们需要：
1. RTTR 注册
2. Inspector 注册
3. as_raw_ptr 策略

### 原因：反射机制不同

| 特性 | C# (Mono) | C++ (RTTR) |
|------|-----------|------------|
| 反射 | 语言内置 | 需要手动注册 |
| 类型信息 | 运行时自动获取 | 编译时生成 |
| 属性访问 | 自动支持 | 需要显式注册 |
| 序列化 | 自动处理 | 需要手动配置 |

### C# 反射原理

C# 的 Mono 运行时可以在运行时获取类型的所有信息：

```csharp
// C# 可以在运行时遍历所有字段
Type type = typeof(RotateScript);
FieldInfo[] fields = type.GetFields();  // 自动获取所有 public 字段
```

引擎加载 C# 程序集时，Mono 会自动：
1. 扫描所有类
2. 提取字段信息
3. 生成元数据

**相关代码**：`Engine/Source/Runtime/Function/Scripting/ScriptEngine.cpp`

```cpp
// 加载程序集时自动扫描类型
void ScriptEngine::LoadAssemblyClasses()
{
    // Mono 提供的 API 自动获取类型信息
    MonoClass* monoClass = mono_class_from_name(image, nameSpace, className);
    
    // 遍历所有字段
    while (MonoClassField* field = mono_class_get_fields(monoClass, &iterator))
    {
        MonoType* type = mono_field_get_type(field);
        // 自动记录字段类型...
    }
}
```

### C++ 需要手动注册的原因

C++ 是静态编译语言，编译后：
- 类型信息被"擦除"
- 字段名变成内存偏移
- 没有运行时类型信息（除非手动添加）

RTTR 库通过宏在编译时生成元数据：

```cpp
// RTTR 在编译时生成类型信息
RTTR_REGISTRATION
{
    rttr::registration::class_<RotateComponent>("RotateComponent")
        .property("rotationSpeed", &RotateComponent::rotationSpeed);
}
```

### 对比图

```
┌─────────────────────────────────────────────────────────┐
│                    C# 脚本流程                           │
├─────────────────────────────────────────────────────────┤
│  写代码 → 编译 → 引擎加载程序集 → Mono 自动提取元数据    │
│                          ↓                              │
│                    Inspector 自动显示                    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   C++ 组件流程                           │
├─────────────────────────────────────────────────────────┤
│  写代码 → RTTR 宏注册 → 编译 → 引擎读取 RTTR 元数据      │
│                          ↓                              │
│                    Inspector 显示（需手动添加）          │
└─────────────────────────────────────────────────────────┘
```

### 想深入了解？

如果你想了解 C# 脚本系统的实现细节，可以阅读：

| 文件 | 内容 |
|------|------|
| `ScriptEngine.h/cpp` | Mono 运行时初始化、脚本实例管理 |
| `ScriptCore/Source/InternalCalls.cs` | C# 到 C++ 的内部调用绑定 |
| `ScriptRegister.cpp` | C++ 端的脚本注册 |

**探索任务**：
1. 打开 `ScriptEngine.cpp`，找到 `LoadAssemblyClasses` 函数
2. 观察它如何使用 Mono API 遍历类型和字段
3. 对比 C++ 的 RTTR 注册方式

---

# 教程 2：追踪一次 Draw Call - 理解渲染管线

## 概述

本教程带你深入引擎内部，追踪一次完整的 Draw Call 从发起到执行的全过程。完成后，你将理解：

- 渲染系统的分层架构
- MeshRenderer 如何被发现和调用
- 渲染路径如何组织渲染流程
- RHI 层如何封装图形 API
- 一个 Draw Call 的完整生命周期

**预期成果**：能够在代码中定位渲染管线的每个关键节点，理解"物体为什么会出现在屏幕上"。

## 前置要求

- 已完成教程 1：组件系统入门
- 了解游戏循环的基本概念
- 有调试 C++ 代码的经验（断点、调用栈）

---

## 第一节：问题引入

### 当你在编辑器中创建一个 Cube...

1. 在 Hierarchy 中右键 → Create → Cube
2. Cube 出现在 Scene View 中
3. 你能看到它、选中它、移动它

**问题**：这个 Cube 是如何被渲染出来的？

让我们从两个关键组件开始追踪：

```
GameObject (Cube)
    ├── Transform        // 位置、旋转、缩放
    ├── MeshFilter       // 网格数据（顶点、索引）
    └── MeshRenderer     // 材质引用 ← 这是渲染的起点！
```

### 探索任务

打开 Visual Studio，在以下文件中设置断点：

| 文件 | 行号位置 | 断点目的 |
|------|----------|----------|
| `MeshRenderer.h` | `GetMaterial()` | 理解材质如何被获取 |
| `Renderer.cpp` | `Tick()` | 理解渲染主循环入口 |
| `Vulkan_CommandList.cpp` | `DrawIndexed()` | 理解最终 GPU 调用 |

运行调试，观察调用栈，你将看到完整的数据流。

---

## 第二节：渲染系统的分层架构

### 三层架构

LitchiEngine 的渲染系统采用经典的三层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层                        │
│                                                             │
│  GameObject ── MeshRenderer ── MeshFilter ── Material       │
│                                                             │
│  职责：组织场景数据，定义"要渲染什么"                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  渲染路径层                        │
│                                                             │
│  RendererPath ── 收集对象 ── 视锥剔除 ── 排序               │
│                                                             │
│  职责：决定"渲染哪些对象"，优化渲染效率                        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   RHI 抽象层 (RHI)                           │
│                                                             │
│  RHI_CommandList ── PSO ── 缓冲区 ── Draw Call              │
│                                                             │
│  职责：封装图形 API，提供跨平台能力                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
                         GPU 执行
```

### 为什么需要分层？

| 分层方式 | 优点 | 缺点 |
|----------|------|------|
| 不分层（直接调用 Vulkan） | 代码简单直接 | 无法移植，耦合严重 |
| **三层架构** | 职责清晰，易于扩展和移植 | 代码量大，调用链长 |

**分层带来的好处**：
1. **跨平台**：RHI 层可以替换为 DirectX、Metal 等后端
2. **解耦**：组件层不需要知道 Vulkan 的存在
3. **优化**：渲染路径层可以做剔除、排序等优化

---

## 第三节：组件层 - MeshRenderer 如何被发现

### 关键问题

MeshRenderer 只是一个挂在 GameObject 上的组件，它如何被渲染系统"发现"？

### 追踪代码

**文件**: `Engine/Source/Runtime/Function/Renderer/Rendering/RendererPath.cpp`

```cpp
void RendererPath::UpdateSceneObject()
{
    // 获取场景中所有 GameObject
    auto& gameObjects = m_scene->GetAllGameObjectList();

    // 清空上一帧的可渲染对象列表
    m_renderables.clear();

    for (auto& gameObject : gameObjects)
    {
        // ★ 关键：查找 MeshRenderer 组件 ★
        auto meshRenderer = gameObject->GetComponent<MeshRenderer>();
        if (meshRenderer && meshRenderer->GetMaterial())
        {
            m_renderables.push_back(gameObject);
        }

        // 同样处理 SkinnedMeshRenderer（骨骼动画）
        auto skinnedRenderer = gameObject->GetComponent<SkinnedMeshRenderer>();
        if (skinnedRenderer && skinnedRenderer->GetMaterial())
        {
            m_renderables.push_back(gameObject);
        }
    }
}
```

**发现机制**：
1. `RendererPath` 持有场景引用 (`m_scene`)
2. 每帧调用 `UpdateSceneObject()` 遍历所有 GameObject
3. 使用 `GetComponent<MeshRenderer>()` 查找组件
4. 将有材质的对象加入渲染列表

### 思考题

**Q**: 如果一个 GameObject 有 MeshFilter 但没有 MeshRenderer，会发生什么？

**A**: 它会被忽略，不会被渲染。MeshFilter 只提供网格数据，MeshRenderer 才负责"告诉渲染系统要渲染"。

---

## 第四节：渲染路径层 - 从收集到剔除

### 渲染路径的职责

```
RendererPath
    ├── UpdateSceneObject()   // 收集可渲染对象
    ├── FrustumCullAndSort()  // 视锥剔除 + 排序
    ├── UpdateLight()         // 更新光源数据
    └── 管理渲染目标、相机等
```

### 视锥剔除

**文件**: `Engine/Source/Runtime/Function/Renderer/Rendering/RendererPath.cpp`

```cpp
void RendererPath::FrustumCullAndSort()
{
    // 获取相机视锥体
    const auto& frustum = m_camera->GetFrustum();

    m_visible_meshes.clear();

    for (auto& obj : m_renderables)
    {
        // 获取物体包围盒
        auto bounds = obj->GetComponent<MeshFilter>()->GetMesh()->GetBounds();

        // ★ 视锥剔除检测 ★
        if (frustum.Intersects(bounds))
        {
            m_visible_meshes.push_back(obj);
        }
    }

    // 按材质排序，减少状态切换
    std::sort(m_visible_meshes.begin(), m_visible_meshes.end(),
        [](GameObject* a, GameObject* b) {
            // 排序逻辑...
        });
}
```

**为什么需要视锥剔除？**

| 方案 | 绘制的物体 | 性能 |
|------|-----------|------|
| 不剔除 | 场景中所有物体 | 浪费 GPU 资源 |
| **视锥剔除** | 仅相机可见的物体 | 大幅提升性能 |

### 排序的意义

```
不排序：物体 A(材质1) → 物体 B(材质2) → 物体 C(材质1)
       切换材质 → 切换材质 → 切换材质 = 2 次切换

排序后：物体 A(材质1) → 物体 C(材质1) → 物体 B(材质2)
       保持材质 → 切换材质 = 1 次切换
```

减少材质切换 = 减少状态变更 = 更好的性能。

---

## 第五节：渲染器层 - Forward 渲染流程

### 渲染主循环

**文件**: `Engine/Source/Runtime/Function/Renderer/Rendering/Renderer.cpp`

```cpp
void Renderer::Tick()
{
    // 1. 更新渲染路径（收集、剔除、排序）
    for (auto& rendererPath : m_rendererPaths)
    {
        rendererPath->Update();
    }

    // 2. 获取命令列表
    auto cmd_list = RHI_Device::GetPrimaryCommandList();

    // 3. 执行渲染
    for (auto& rendererPath : m_rendererPaths)
    {
        Render4BuildInSceneView(cmd_list, rendererPath);
    }

    // 4. 提交命令
    cmd_list->End();
    cmd_list->Submit();

    // 5. 呈现
    m_swapChain->Present();
}
```

### Forward 渲染 Pass

**文件**: `Engine/Source/Runtime/Function/Renderer/Rendering/Renderer_Passes.cpp`

```cpp
void Renderer::Pass_ForwardPass(RHI_CommandList* cmd_list, RendererPath* rendererPath)
{
    // 获取可见的网格对象
    auto& meshes = rendererPath->GetVisibleMeshes();

    for (auto& gameObject : meshes)
    {
        // ========== 获取渲染数据 ==========

        // 1. 网格数据
        auto meshFilter = gameObject->GetComponent<MeshFilter>();
        auto mesh = meshFilter->GetMesh();
        auto vertex_buffer = mesh->GetVertexBuffer();
        auto index_buffer = mesh->GetIndexBuffer();

        // 2. 材质数据
        auto meshRenderer = gameObject->GetComponent<MeshRenderer>();
        auto material = meshRenderer->GetMaterial();

        // 3. 变换矩阵
        auto transform = gameObject->GetTransform();
        Matrix4x4 worldMatrix = transform->GetWorldMatrix();

        // ========== 设置渲染状态 ==========

        // 4. 创建/获取 Pipeline State Object
        RHI_PipelineState pso;
        pso.shader_vertex = material->GetVertexShader();
        pso.shader_pixel = material->GetPixelShader();
        pso.render_target_color_textures[0] = rendererPath->GetRenderTarget();
        pso.depth_stencil_texture = rendererPath->GetDepthTexture();
        // ... 其他状态

        cmd_list->SetPipelineState(pso);

        // 5. 绑定顶点/索引缓冲区
        cmd_list->SetBufferVertex(vertex_buffer);
        cmd_list->SetBufferIndex(index_buffer);

        // 6. 设置常量缓冲区（相机、材质参数）
        SetConstantBuffers(cmd_list, material, rendererPath);

        // 7. Push Constants（变换矩阵等高频数据）
        cmd_list->PushConstants(&worldMatrix, sizeof(Matrix4x4));

        // ========== 发出 Draw Call ==========

        // 8. ★ 最终的 Draw Call ★
        cmd_list->DrawIndexed(
            mesh->GetIndexCount(),    // 索引数量
            0,                         // 起始索引
            0                          // 顶点偏移
        );
    }
}
```

**关键数据流向**：

```
GameObject
    ├── MeshFilter → Mesh → VertexBuffer / IndexBuffer
    ├── MeshRenderer → Material → Shader / Textures / Constants
    └── Transform → WorldMatrix → PushConstants
                                        ↓
                              DrawIndexed()
```

---

## 第六节：RHI 抽象层 - 跨平台的关键

### RHI 是什么？

RHI (Render Hardware Interface) 是一个抽象层，它定义了一套与平台无关的渲染接口：

```
应用代码
    ↓
RHI_CommandList::DrawIndexed()  // 平台无关的接口
    ↓
┌─────────────┬─────────────┬─────────────┐
│   Vulkan    │  DirectX 12 │    Metal    │
└─────────────┴─────────────┴─────────────┘
```

### RHI_CommandList 的关键方法

**文件**: `Engine/Source/Runtime/Function/Renderer/RHI/RHI_CommandList.h`

```cpp
class RHI_CommandList
{
public:
    // 管线状态
    virtual void SetPipelineState(const RHI_PipelineState& pso) = 0;

    // 资源绑定
    virtual void SetBufferVertex(RHI_Buffer* buffer) = 0;
    virtual void SetBufferIndex(RHI_Buffer* buffer) = 0;
    virtual void SetTexture(uint32_t slot, RHI_Texture* texture) = 0;
    virtual void SetConstantBuffer(uint32_t slot, RHI_ConstantBuffer* buffer) = 0;

    // 高频数据
    virtual void PushConstants(void* data, uint32_t size) = 0;

    // ★ 绘制命令 ★
    virtual void DrawIndexed(uint32_t index_count, uint32_t first_index, int32_t vertex_offset) = 0;

    // 命令提交
    virtual void End() = 0;
    virtual void Submit() = 0;
};
```

### Vulkan 后端实现

**文件**: `Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/Vulkan_CommandList.cpp`

```cpp
void Vulkan_CommandList::DrawIndexed(uint32_t index_count, uint32_t first_index, int32_t vertex_offset)
{
    // 1. 确保渲染通道活跃
    RenderPassBegin();

    // 2. 绑定动态描述符集（常量缓冲区、纹理等）
    descriptor_sets::set_dynamic(m_device, m_descriptor_set_dynamic, ...);
    vkCmdBindDescriptorSets(m_command_buffer, ...);

    // 3. ★ Vulkan API 调用 ★
    vkCmdDrawIndexed(
        m_command_buffer,
        index_count,      // 索引数量
        1,                // 实例数量（非实例化渲染为 1）
        first_index,      // 起始索引
        vertex_offset,    // 顶点偏移
        0                 // 起始实例
    );

    m_draw_calls++;  // 统计 Draw Call 数量
}
```

**这就是 Draw Call 的终点**：从 `MeshRenderer` 组件开始，经过多层抽象，最终调用 `vkCmdDrawIndexed()` 将绘制命令发送到 GPU。

---

## 第七节：完整调用链总结

### 一次 Draw Call 的旅程

```
┌─────────────────────────────────────────────────────────────────┐
│ 应用入口                                                         │
│ ApplicationBase::Tick()                                         │
│ └─▶ Renderer::Tick()                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 渲染路径更新                                                     │
│ RendererPath::Update()                                          │
│ ├─▶ UpdateSceneObject()     // 收集 MeshRenderer                │
│ ├─▶ FrustumCullAndSort()    // 视锥剔除 + 排序                   │
│ └─▶ UpdateLight()           // 更新光源                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ Forward 渲染 Pass                                                │
│ Renderer::Pass_ForwardPass()                                    │
│ ├─▶ meshFilter->GetMesh()->GetVertexBuffer()                    │
│ ├─▶ meshRenderer->GetMaterial()                                 │
│ ├─▶ cmd_list->SetPipelineState(pso)                             │
│ ├─▶ cmd_list->SetBufferVertex/Index()                           │
│ ├─▶ cmd_list->PushConstants(worldMatrix)                        │
│ └─▶ cmd_list->DrawIndexed()                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ RHI 层                                                          │
│ Vulkan_CommandList::DrawIndexed()                               │
│ ├─▶ RenderPassBegin()        // 确保渲染通道开启                 │
│ ├─▶ vkCmdBindDescriptorSets()// 绑定资源                         │
│ └─▶ vkCmdDrawIndexed()       // ★ GPU 命令 ★                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 命令提交                                                         │
│ cmd_list->End()              // vkEndCommandBuffer()            │
│ cmd_list->Submit()           // vkQueueSubmit()                 │
│ swapChain->Present()         // vkQueuePresentKHR()             │
└─────────────────────────────────────────────────────────────────┘
```

### 各层次职责总结

| 层次 | 关键类 | 核心职责 |
|------|--------|----------|
| **组件层** | MeshRenderer, MeshFilter | 持有渲染数据，被场景管理 |
| **渲染路径层** | RendererPath | 收集、剔除、排序可渲染对象 |
| **渲染器层** | Renderer | 组织 Pass，设置状态，发出 Draw Call |
| **RHI 层** | RHI_CommandList | 封装图形 API，跨平台抽象 |
| **后端层** | Vulkan_* | 调用 Vulkan API，GPU 命令提交 |

---

## 第八节：实践任务

### 任务 1：设置断点追踪

在以下位置设置断点，运行调试，观察调用栈：

1. `MeshRenderer::GetMaterial()`
2. `RendererPath::UpdateSceneObject()`
3. `Renderer::Pass_ForwardPass()`
4. `Vulkan_CommandList::DrawIndexed()`

**记录**：每个断点触发时的完整调用栈。

### 任务 2：分析渲染帧

1. 在 `Renderer::Tick()` 开始和结束处添加日志
2. 运行场景，观察每帧的渲染时间
3. 在 `Pass_ForwardPass()` 中统计 Draw Call 数量

**思考**：一个有 100 个物体的场景，一帧会产生多少次 `DrawIndexed()` 调用？

### 任务 3：理解剔除效果

1. 创建一个场景，放置 10 个 Cube
2. 将相机移动到只能看到 3 个 Cube 的位置
3. 在 `FrustumCullAndSort()` 中添加日志，对比 `m_renderables.size()` 和 `m_visible_meshes.size()`

---

## 验证标准

- [ ] 能在代码中定位渲染管线的 4 个层次
- [ ] 能解释 MeshRenderer 如何被渲染系统发现
- [ ] 能描述视锥剔除的作用和位置
- [ ] 能追踪一次完整的 Draw Call 调用链
- [ ] 能解释 RHI 层的跨平台价值

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| 三层架构 | 组件层 → 渲染路径层 → RHI 层 |
| 发现机制 | 遍历场景，GetComponent<MeshRenderer>() |
| 视锥剔除 | 只渲染相机可见的物体 |
| Forward 渲染 | 逐物体设置状态，发出 Draw Call |
| RHI 抽象 | 平台无关接口，支持多后端 |
| Draw Call 旅程 | 从组件到 vkCmdDrawIndexed() |

### 核心理解

**"物体为什么会出现在屏幕上？"**

1. **数据准备**：MeshRenderer + MeshFilter + Transform 提供了渲染所需的所有数据
2. **发现收集**：RendererPath 遍历场景，发现有这些组件的 GameObject
3. **优化剔除**：视锥剔除排除不可见物体，排序减少状态切换
4. **状态设置**：Renderer 组织 Pipeline State，绑定资源
5. **命令发出**：RHI 层封装为平台无关的 Draw Call
6. **GPU 执行**：Vulkan 后端调用 API，GPU 绘制

---

## 延伸阅读

### 想深入了解？

| 主题 | 推荐文件 |
|------|----------|
| RHI 抽象设计 | `Engine/Source/Runtime/Function/Renderer/RHI/RHI_Device.h` |
| Vulkan 后端 | `Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/` |
| 渲染 Pass 组织 | `Engine/Source/Runtime/Function/Renderer/Rendering/Renderer_Passes.cpp` |
| 材质系统 | `Engine/Source/Runtime/Function/Renderer/Rendering/Material.h` |

### 进阶方向

1. **添加新的渲染 Pass** - 学习如何扩展渲染管线
2. **实现延迟渲染** - 理解 G-Buffer 和多 Pass 渲染
3. **GPU 性能分析** - 使用 RenderDoc 分析 Draw Call

---

**文档时间**: 2026-04-14
**风格参考**: Catlike Coding (https://catlikecoding.com/)

---

## 系列总结

恭喜完成引擎基础系列教程！你已学习：

| 教程 | 核心知识点 |
|------|-----------|
| RotateComponent | 组件架构、生命周期、Transform、四元数 |
| 追踪 Draw Call | 渲染管线分层架构、视锥剔除、RHI 抽象、Draw Call 生命周期 |

### 下一步学习方向

1. **着色器开发** - 学习 HLSL、材质系统
2. **脚本系统系列** - 学习 C# 脚本绑定
3. **动画系统系列** - 学习骨骼动画

---

## 附录：常用路径

| 系统 | 路径 |
|------|------|
| 组件基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/` |
| 渲染器 | `Engine/Source/Runtime/Function/Renderer/Rendering/` |
| RHI 抽象层 | `Engine/Source/Runtime/Function/Renderer/RHI/` |
| Vulkan 后端 | `Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |
| RTTR 类型注册 | `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h` |

---

**文档时间**: 2026-04-14
**风格参考**: Catlike Coding (https://catlikecoding.com/)
