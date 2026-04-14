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
| 渲染入门 | 教程 2: 自定义着色器 | 进阶 | 教程 1 |

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

# 教程 2：自定义着色器 - 脉冲发光效果

## 概述

通过创建一个自定义着色器和材质，学习 LitchiEngine 的渲染管线。完成后，你将理解：

- 着色器文件结构
- 材质与着色器的关系
- Uniform 数据传递机制
- 常量缓冲区 (Constant Buffer)

**预期成果**：一个能让物体产生脉冲发光效果的着色器和材质。

## 前置要求

- 已完成教程 1：组件系统入门
- 了解 HLSL 基础语法（变量、函数、语义）
- 了解 GPU 渲染管线的基本概念

---

## 第一节：理解渲染管线

### 从组件到像素

当你添加 MeshRenderer 组件并设置材质后，渲染流程如下：

```
MeshFilter (网格数据)
     ↓
MeshRenderer (材质引用)
     ↓
Material (着色器 + 参数)
     ↓
Vertex Shader (顶点变换)
     ↓
Pixel Shader (像素着色)
     ↓
屏幕上的像素
```

### 关键文件路径

| 文件类型 | 路径 |
|----------|------|
| 着色器源码 | `Engine/Data/Engine/Shaders/` |
| 材质文件 | `Engine/Data/Engine/Materials/` |
| 着色器公共头文件 | `Engine/Data/Engine/Shaders/Common/` |
| Material 类 | `Engine/Source/Runtime/Function/Renderer/Rendering/Material.h` |

---

## 第二节：分析现有着色器

### 查看 forward.hlsl

**文件路径**: `Engine/Data/Engine/Shaders/forward.hlsl`

```hlsl
//= INCLUDES =========
#include "Common/common.hlsl"
//====================

Pixel_PosUvNorTan mainVS(Vertex_PosUvNorTan input)
{
    Pixel_PosUvNorTan output;

    output.position = mul(input.position, buffer_pass.transform);
    output.position = mul(output.position, buffer_rendererPath.view_projection);
    output.uv = input.uv;
    output.normal = normalize(mul(input.normal, (float3x3) buffer_pass.transform)).xyz;
    output.normal = normalize(mul(output.normal, (float3x3) buffer_rendererPath.view_projection)).xyz;
    output.tangent = normalize(mul(input.tangent, (float3x3) buffer_pass.transform)).xyz;
    output.tangent = normalize(mul(output.tangent, (float3x3) buffer_rendererPath.view_projection)).xyz;

    return output;
}

float4 mainPS(Pixel_PosUvNorTan input) : SV_Target
{
    return float4(0.644f, 0.003f, 0.005f, 1.0f);  // 固定红色
}
```

**代码解析**：

| 部分 | 说明 |
|------|------|
| `#include "Common/common.hlsl"` | 包含公共定义、常量缓冲区结构 |
| `Vertex_PosUvNorTan` | 输入顶点结构（位置、UV、法线、切线） |
| `Pixel_PosUvNorTan` | 输出到像素着色器的数据 |
| `buffer_pass.transform` | 当前物体的世界矩阵 |
| `buffer_rendererPath.view_projection` | 相机的视图投影矩阵 |
| `SV_Target` | 像素着色器输出颜色 |

### 常量缓冲区结构

**文件路径**: `Engine/Data/Engine/Shaders/Common/common_buffers.hlsl`

```hlsl
// 每帧更新 - 相机数据
cbuffer BufferRendererPath : register(b5)
{
    RendererPathBufferData buffer_rendererPath;
}

// 每物体更新 - 变换矩阵
[[vk::push_constant]]
PassBufferData buffer_pass;

// 每材质更新 - 材质参数
cbuffer BufferMaterial : register(b2)
{
    MaterialBufferData buffer_material;
}
```

**更新频率**：

| 缓冲区 | 更新频率 | 内容 |
|--------|----------|------|
| BufferRendererPath | 每帧 | 相机位置、视图投影矩阵 |
| buffer_pass (Push Constant) | 每物体 | 世界变换矩阵 |
| BufferMaterial | 每材质 | 颜色、粗糙度、金属度等 |

---

## 第三节：创建脉冲发光着色器

### 问题：如何让颜色随时间变化？

我们需要：
1. 获取时间值（来自引擎）
2. 使用正弦函数产生周期性变化
3. 将时间传递给着色器

### 第一步：创建着色器文件

在 `Engine/Data/Engine/Shaders/` 目录下创建 `Pulse.hlsl`：

```hlsl
//= INCLUDES =========
#include "Common/common.hlsl"
//====================

// 像素着色器输出
struct PixelOutput
{
    float4 color : SV_Target0;
};

// 顶点着色器
Pixel_PosUvNorTan mainVS(Vertex_PosUvNorTan input)
{
    Pixel_PosUvNorTan output;

    // 变换到裁剪空间
    output.position = mul(input.position, buffer_pass.transform);
    output.position = mul(output.position, buffer_rendererPath.view_projection);

    // 传递纹理坐标和法线
    output.uv = input.uv;
    output.normal = normalize(mul(input.normal, (float3x3) buffer_pass.transform));
    output.tangent = normalize(mul(input.tangent, (float3x3) buffer_pass.transform));

    return output;
}

// 像素着色器
PixelOutput mainPS(Pixel_PosUvNorTan input)
{
    PixelOutput output;

    // 使用帧数据的 delta_time 和 frame 计算脉冲
    float time = buffer_frame.frame * buffer_frame.delta_time;
    float pulse = 0.5 + 0.5 * sin(time * 3.0);  // 3.0 控制脉冲速度

    // 基础颜色（青色）
    float3 baseColor = float3(0.0, 0.8, 0.8);

    // 混合脉冲效果
    float3 finalColor = baseColor * (0.5 + 0.5 * pulse);

    output.color = float4(finalColor, 1.0);

    return output;
}
```

**验证**: 此时着色器文件已创建，但还不能被引擎识别。

### 第二步：理解 FrameBufferData

查看 `common_buffers.hlsl` 中的帧数据结构：

```hlsl
struct FrameBufferData
{
    float2 resolution_render;
    float2 resolution_output;

    float2 taa_jitter_current;
    float2 taa_jitter_previous;

    float delta_time;  // 帧间隔时间
    uint frame;        // 帧计数器
    float gamma;
    uint options;
};

cbuffer BufferFrame : register(b0)
{
    FrameBufferData buffer_frame;
}
```

**为什么用 frame * delta_time？**

| 方案 | 问题 |
|------|------|
| 直接用 `buffer_frame.frame` | 帧数增长太快，闪烁过快 |
| 直接用 `buffer_frame.delta_time` | 只有一帧的时间，无法累积 |
| `frame * delta_time` | 累积时间，平滑过渡 |

---

## 第四节：创建材质文件

### 第三步：创建材质 JSON

在 `Engine/Data/Engine/Materials/` 目录下创建 `Pulse.mat`：

```json
{
  "vertexType": "PosUvNorTan",
  "shaderPath": ":Shaders/Pulse.hlsl",
  "uniformInfoList": []
}
```

**字段说明**：

| 字段 | 说明 |
|------|------|
| `vertexType` | 顶点类型，决定顶点着色器输入结构 |
| `shaderPath` | 着色器路径，`:` 前缀表示引擎资源 |
| `uniformInfoList` | 自定义 uniform 参数列表（暂时为空） |

### 第四步：在编辑器中使用

1. 重新编译引擎（着色器会被编译为 SPIR-V）
2. 打开 LitchiEditor
3. 创建一个 Cube
4. 在 MeshRenderer 组件中设置 Material Path 为 `:Materials/Pulse.mat`
5. 运行场景

**预期效果**: 物体呈现青色脉冲发光效果。

---

## 第五节：添加可配置参数

### 问题：如何让设计师调整颜色和速度？

我们需要添加自定义 uniform 参数，让材质可配置。

### 第五步：更新着色器

修改 `Pulse.hlsl`，添加自定义参数：

```hlsl
//= INCLUDES =========
#include "Common/common.hlsl"
//====================

// Material Buffer Name Must Be "Material"
// register(b10) is required by the engine for material constants
struct MaterialData
{
    float3 u_baseColor;      // offset: 0, size: 12
    float u_pulseSpeed;      // offset: 12, size: 4 (packed after float3)
    float u_pulseIntensity;  // offset: 16, size: 4 (new 16-byte row)
    float3 padding;          // offset: 32, size: 12 (padding for 16-byte alignment)
};

cbuffer Material : register(b10)
{
    MaterialData materialData;
};

struct PixelOutput
{
    float4 color : SV_Target0;
};

Pixel_PosUvNorTan mainVS(Vertex_PosUvNorTan input)
{
    Pixel_PosUvNorTan output;

    output.position = mul(input.position, buffer_pass.transform);
    output.position = mul(output.position, buffer_rendererPath.view_projection);

    output.uv = input.uv;
    output.normal = normalize(mul(input.normal, (float3x3) buffer_pass.transform));
    output.tangent = normalize(mul(input.tangent, (float3x3) buffer_pass.transform));

    return output;
}

PixelOutput mainPS(Pixel_PosUvNorTan input)
{
    PixelOutput output;

    float time = buffer_frame.frame * buffer_frame.delta_time;
    float pulse = 0.5 + 0.5 * sin(time * materialData.u_pulseSpeed);
    float intensity = 1.0 - materialData.u_pulseIntensity * (1.0 - pulse);

    float3 finalColor = materialData.u_baseColor * intensity;
    output.color = float4(finalColor, 1.0);

    return output;
}
```

**重要说明**：

| 要点 | 说明 |
|------|------|
| `cbuffer Material` | 缓冲区名称必须为 `Material`，引擎会按名称查找 |
| `register(b10)` | 引擎为材质参数预留的槽位，必须使用此槽位 |
| `MaterialData` 结构体 | 包含所有自定义参数，注意16字节对齐 |
| 成员名称 | 必须与材质 JSON 中的 `name` 一致（如 `u_baseColor`）|

**为什么用 register(b10)？**

LitchiEngine 的常量缓冲区槽位分配：

| 槽位 | 用途 | 说明 |
|------|------|------|
| b0 | BufferFrame | 帧数据（时间、分辨率等）|
| b2 | BufferMaterial | 引擎内置材质参数 |
| b5 | BufferRendererPath | 渲染路径数据（相机矩阵等）|
| **b10** | **自定义材质参数** | 用户自定义材质 uniform |

### 第六步：更新材质文件

修改 `Pulse.mat`：

```json
{
  "vertexType": "PosUvNorTan",
  "shaderPath": ":Shaders/Pulse.hlsl",
  "uniformInfoList": [
    {
      "Type": "UniformInfoVector3",
      "name": "u_baseColor",
      "vector": {
        "x": 0.0,
        "y": 0.8,
        "z": 0.8
      }
    },
    {
      "Type": "UniformInfoFloat",
      "name": "u_pulseSpeed",
      "value": 3.0
    },
    {
      "Type": "UniformInfoFloat",
      "name": "u_pulseIntensity",
      "value": 0.5
    }
  ]
}
```

**Uniform 类型映射**：

| 着色器类型 | JSON 类型 | C++ 类型 |
|------------|-----------|----------|
| `float` | UniformInfoFloat | float |
| `float2` | UniformInfoVector2 | Vector2 |
| `float3` | UniformInfoVector3 | Vector3 |
| `float4` | UniformInfoVector4 | Vector4 |
| `Texture2D` | UniformInfoTexture | RHI_Texture* |

---

## 第六节：理解材质系统

### Material 类的工作流程

**文件路径**: `Engine/Source/Runtime/Function/Renderer/Rendering/Material.h`

```cpp
class Material : public IResource
{
public:
    // 设置 uniform 值
    template<typename T>
    void SetValue(const std::string& name, const T& value);

    // 获取 uniform 值
    template<typename T>
    const T& GetValue(const std::string& key);

    // 设置纹理
    void SetTexture(const std::string& name, RHI_Texture* texture);

    // 获取着色器
    MaterialShader* GetShader() { return m_shader; }

private:
    MaterialRes* m_materialRes;                    // 序列化数据
    MaterialShader* m_shader;                      // 着色器
    std::map<std::string, std::any> m_uniformDataList;  // uniform 数据
    std::shared_ptr<RHI_ConstantBuffer> m_valueConstantBuffer;  // GPU 缓冲区
};
```

### 数据流向

```
材质 JSON 文件 (.mat)
       ↓
Material::LoadFromFile()
       ↓
MaterialRes (反序列化数据)
       ↓
Material::PostResourceLoaded()
       ↓
m_uniformDataList (运行时数据)
       ↓
Material::UpdateRenderData()
       ↓
RHI_ConstantBuffer (GPU 缓冲区)
       ↓
着色器读取
```

### 16字节对齐规则

HLSL 常量缓冲区要求 16 字节对齐。这是初学者最常遇到的问题！

**正确示例（Pulse.hlsl 中的 MaterialData）**：

```hlsl
struct MaterialData
{
    float3 u_baseColor;      // offset: 0,  size: 12 bytes
    float u_pulseSpeed;      // offset: 12, size: 4 bytes  ← 填充到 16 字节边界
    float u_pulseIntensity;  // offset: 16, size: 4 bytes  ← 新的 16 字节行开始
    float3 padding;          // offset: 32, size: 12 bytes ← 填充到完整 16 字节
};
// Total: 48 bytes (3 × 16)
```

**内存布局图示**：

```
┌────────────────────────────────────────┐
│ Row 1 (16 bytes)                        │
│ ├─ u_baseColor.x (4)                    │
│ ├─ u_baseColor.y (4)                    │
│ ├─ u_baseColor.z (4)                    │
│ └─ u_pulseSpeed (4)                     │
├────────────────────────────────────────┤
│ Row 2 (16 bytes)                        │
│ ├─ u_pulseIntensity (4)                 │
│ ├─ padding.x (4)                        │
│ ├─ padding.y (4)                        │
│ └─ padding.z (4)                        │
└────────────────────────────────────────┘
```

**常见错误**：

```hlsl
// 错误 ✗ - float3 后面紧跟 float2，跨越 16 字节边界
struct BadData
{
    float3 color;      // 12 bytes
    float2 speed;      // 8 bytes - 错误！会从偏移 16 开始
                       // 但引擎可能从偏移 12 开始写入，导致数据错位
};

// 正确 ✓ - 添加填充
struct GoodData
{
    float3 color;      // 12 bytes
    float _pad1;       // 4 bytes - 填充到 16
    float2 speed;      // 8 bytes (从 16 开始)
    float2 _pad2;      // 8 bytes - 填充到 32
};
```

**对齐规则总结**：

| 类型 | 大小 | 对齐要求 | 打包规则 |
|------|------|----------|----------|
| `float` | 4 bytes | 4 bytes | 可以放在任何 4 的倍数位置 |
| `float2` | 8 bytes | 8 bytes | 必须放在 8 的倍数位置 |
| `float3` | 12 bytes | **16 bytes** | 必须放在 16 的倍数位置！ |
| `float4` | 16 bytes | 16 bytes | 必须放在 16 的倍数位置 |
| `matrix` | 64 bytes | 16 bytes | 每行是一个 float4 |

**实用技巧**：字段排列顺序

1. 先放 `float4` 和 `matrix`（16 字节对齐）
2. 再放 `float3`（后面留 4 字节给 float）
3. 再放 `float2`（8 字节对齐）
4. 最后放 `float`（填充缝隙）

---

## 第七节：运行时修改材质参数

### 问题：如何在代码中动态改变材质属性？

创建一个组件来控制脉冲效果。

### 第七步：创建 PulseController 组件

**头文件**: `Engine/Source/Runtime/Function/Framework/Component/Gameplay/PulseController.h`

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class PulseController : public Component
    {
    public:
        PulseController();
        ~PulseController() override;

        // Getter/Setter 用于 RTTR 属性绑定
        Vector3 GetBaseColor() const { return m_baseColor; }
        void SetBaseColor(Vector3 color) { m_baseColor = color; }

        float GetPulseSpeed() const { return m_pulseSpeed; }
        void SetPulseSpeed(float speed) { m_pulseSpeed = speed; }

        float GetPulseIntensity() const { return m_pulseIntensity; }
        void SetPulseIntensity(float intensity) { m_pulseIntensity = intensity; }

        void OnStart() override;
        void OnUpdate() override;
        void OnEditorUpdate() override;

    private:
        Vector3 m_baseColor{0.0f, 0.8f, 0.8f};
        float m_pulseSpeed = 3.0f;
        float m_pulseIntensity = 0.5f;

        RTTR_ENABLE(Component)
    };
}
```

**为什么使用 Getter/Setter？**

| 方式 | 优点 | 缺点 |
|------|------|------|
| 直接暴露成员变量 (`public`) | 简单直接 | 无法添加验证逻辑，无法触发回调 |
| **Getter/Setter** | 可添加验证、支持计算属性、符合封装原则 | 代码略多 |

在 LitchiEngine 中，使用 Getter/Setter 是推荐的做法，因为：
1. 可以在设置值时添加范围检查
2. 便于调试（可在 setter 中打断点）
3. RTTR 完美支持 getter/setter 绑定

### 第八步：实现组件逻辑

**实现文件**: `Engine/Source/Runtime/Function/Framework/Component/Gameplay/PulseController.cpp`

```cpp
#include "Runtime/Core/pch.h"
#include "PulseController.h"

#include "Runtime/Function/Framework/GameObject/GameObject.h"
#include "Runtime/Function/Framework/Component/Renderer/MeshRenderer.h"
#include "Runtime/Function/Renderer/Rendering/Material.h"

namespace LitchiRuntime
{
    PulseController::PulseController()
        : m_baseColor(0.0f, 0.8f, 0.8f)
        , m_pulseSpeed(3.0f)
        , m_pulseIntensity(0.5f)
    {
    }

    PulseController::~PulseController()
    {
    }

    void PulseController::OnStart()
    {
        // OnStart 在场景开始时调用一次
        // 可以在这里做初始化检查
        MeshRenderer* renderer = GetGameObject()->GetComponent<MeshRenderer>();
        if (!renderer)
        {
            // 记录警告：没有 MeshRenderer 组件
            return;
        }

        Material* material = renderer->GetMaterial();
        if (!material)
        {
            // 记录警告：没有材质
            return;
        }

        // 可以在这里检查材质是否有需要的 uniform 参数
    }

    void PulseController::OnUpdate()
    {
        // OnUpdate 在运行模式下每帧调用
        MeshRenderer* renderer = GetGameObject()->GetComponent<MeshRenderer>();
        if (!renderer) return;

        Material* material = renderer->GetMaterial();
        if (!material) return;

        // 更新材质参数
        material->SetValue("u_baseColor", m_baseColor);
        material->SetValue("u_pulseSpeed", m_pulseSpeed);
        material->SetValue("u_pulseIntensity", m_pulseIntensity);
    }

    void PulseController::OnEditorUpdate()
    {
        // OnEditorUpdate 在编辑器模式下每帧调用
        // 这样即使不点击 Play，也能在编辑器中预览效果
        MeshRenderer* renderer = GetGameObject()->GetComponent<MeshRenderer>();
        if (!renderer) return;

        Material* material = renderer->GetMaterial();
        if (!material) return;

        material->SetValue("u_baseColor", m_baseColor);
        material->SetValue("u_pulseSpeed", m_pulseSpeed);
        material->SetValue("u_pulseIntensity", m_pulseIntensity);
    }
}
```

**生命周期方法说明**：

| 方法 | 调用时机 | 用途 |
|------|----------|------|
| `OnStart()` | 场景开始时调用一次 | 初始化、检查依赖 |
| `OnUpdate()` | **运行模式**下每帧调用 | 游戏逻辑 |
| `OnEditorUpdate()` | **编辑器模式**下每帧调用 | 编辑器预览效果 |

### 第九步：注册组件到 RTTR

**文件路径**: `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h`

1. 在文件顶部添加 include：
```cpp
#include "Runtime/Function/Framework/Component/Gameplay/PulseController.h"
```

2. 在 `Framework Object Types` 区域添加注册：
```cpp
rttr::registration::class_<PulseController>("PulseController")
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("baseColor", &PulseController::GetBaseColor, &PulseController::SetBaseColor)
    .property("pulseSpeed", &PulseController::GetPulseSpeed, &PulseController::SetPulseSpeed)
    .property("pulseIntensity", &PulseController::GetPulseIntensity, &PulseController::SetPulseIntensity);
```

**注意**：使用 getter/setter 绑定时，`.property()` 接受三个参数：
- 属性名（显示在 Inspector 中）
- Getter 方法指针
- Setter 方法指针

### 第十步：在 Inspector 中注册组件

**文件路径**: `Engine/Source/Editor/source/Panels/Inspector.cpp`

1. 在文件顶部添加 include：
```cpp
#include "Runtime/Function/Framework/Component/Gameplay/PulseController.h"
```

2. 找到 `componentSelectorWidget.choices` 的定义位置（约第 80 行），添加：
```cpp
componentSelectorWidget.choices.emplace(18, "PulseController");
```

3. 在 `addComponentButton.ClickedEvent` 的 switch 语句中添加：
```cpp
case 18: GetTargetActor()->AddComponent<PulseController>(); break;
```

4. 在 `componentSelectorWidget.ValueChangedEvent` 的 switch 语句中添加：
```cpp
case 18: defineButtonsStates(GetTargetActor()->GetComponent<PulseController>()); return;
```

**完整修改位置**：

| 位置 | 作用 |
|------|------|
| 头文件 include | 让编译器知道 PulseController 类型 |
| choices.emplace | 在下拉列表中显示选项 |
| switch case (ClickedEvent) | 点击按钮时创建组件 |
| switch case (ValueChangedEvent) | 检查组件是否已存在，禁用按钮 |

---

## 验证标准

- [ ] 着色器编译无错误
- [ ] 材质文件正确加载
- [ ] 物体显示脉冲发光效果
- [ ] Inspector 中可调整 PulseController 参数
- [ ] PulseController 组件能动态控制效果
- [ ] 编辑器模式下也能预览脉冲效果（OnEditorUpdate）

---

## 常见问题排查

### 着色器编译问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 材质参数不生效 | cbuffer 名称错误 | 必须命名为 `Material` |
| 数据错位 | register 槽位错误 | 必须使用 `register(b10)` |
| 变量未定义 | 成员名与 JSON 不匹配 | 确保 HLSL 变量名与 JSON 中 `name` 一致 |

### 组件问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 组件不显示在列表中 | RTTR 注册失败 | 检查 TypeRegister.h 中的注册 |
| 材质参数不更新 | OnUpdate/OnEditorUpdate 未调用 | 检查组件是否激活 |
| 属性不显示在 Inspector | 未正确绑定 getter/setter | 确保 `.property()` 参数正确 |

### 调试技巧

在 `OnStart()` 中添加调试日志，检查材质和 uniform 信息：

```cpp
void PulseController::OnStart()
{
    MeshRenderer* renderer = GetGameObject()->GetComponent<MeshRenderer>();
    if (!renderer) return;

    Material* material = renderer->GetMaterial();
    if (!material) return;

    // 打印所有 uniform 参数
    auto& uniforms = material->GetUniformsData();
    for (auto& pair : uniforms)
    {
        DEBUG_LOG_INFO("Uniform: {} type: {}", pair.first, pair.second.type().name());
    }

    // 打印着色器的 uniform 信息
    auto shader = material->GetShader();
    if (shader)
    {
        auto& globalUniforms = shader->GetGlobalShaderUniformDict();
        for (auto& pair : globalUniforms)
        {
            DEBUG_LOG_INFO("Shader uniform: {} offset: {} size: {}",
                pair.first, pair.second.location, pair.second.size);
        }
    }
}
```

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| 着色器结构 | 顶点着色器 + 像素着色器 |
| 常量缓冲区 | GPU 数据传递机制，必须使用 `register(b10)` 和 `Material` 名称 |
| 材质文件 | JSON 格式的材质配置 |
| Uniform 参数 | 着色器可配置参数，名称必须一致 |
| Material 类 | 运行时材质管理，`SetValue()` 更新参数 |
| 16字节对齐 | HLSL 缓冲区对齐规则 |
| Getter/Setter | RTTR 属性绑定的推荐方式 |
| OnEditorUpdate | 编辑器模式预览效果 |

### 创建自定义着色器的完整流程

1. **编写着色器** - 在 `Engine/Data/Engine/Shaders/` 创建 .hlsl 文件
   - 使用 `cbuffer Material : register(b10)` 定义材质参数
   - 参数名与材质 JSON 中一致
2. **创建材质** - 在 `Engine/Data/Engine/Materials/` 创建 .mat 文件
   - 定义 uniformInfoList 参数
3. **创建控制器组件** - 继承 Component，实现 OnUpdate 和 OnEditorUpdate
4. **注册组件** - 在 TypeRegister.h 中注册 RTTR
5. **注册 Inspector** - 在 Inspector.cpp 中添加组件选择器入口
6. **测试验证** - 编译运行，检查效果

---

## 延伸：着色器编译流程

### 问题：HLSL 如何变成 GPU 可执行代码？

LitchiEngine 使用 DXCompiler 将 HLSL 编译为 SPIR-V：

```
Pulse.hlsl (HLSL 源码)
       ↓
DXCompiler (编译器)
       ↓
Pulse.vert.spv (顶点着色器 SPIR-V)
Pulse.frag.spv (像素着色器 SPIR-V)
       ↓
Vulkan 加载执行
```

**相关代码**: `Engine/Source/Runtime/Function/Renderer/RHI/RHI_DirectXShaderCompiler.cpp`

### 为什么选择 HLSL？

| 着色器语言 | 优点 | 缺点 |
|------------|------|------|
| HLSL | Windows 生态友好、DXCompiler 支持好 | 需要编译转换 |
| GLSL | OpenGL/Vulkan 原生支持 | 工具链较弱 |
| Slang | 现代化设计、跨平台 | 生态较小 |

LitchiEngine 选择 HLSL + DXCompiler 方案，可以：
- 使用 Visual Studio 的着色器调试工具
- 生成优化的 SPIR-V 代码
- 支持 #include 指令

---

## 下一步

恭喜完成渲染入门教程！你已学习：

| 教程 | 核心知识点 |
|------|-----------|
| RotateComponent | 组件架构、生命周期、Transform |
| PulseShader | 着色器管线、材质系统、Uniform 参数 |

### 进阶方向

1. **PBR 材质** - 学习物理渲染
2. **后处理效果** - 学习全屏着色器
3. **计算着色器** - 学习 GPU 通用计算

---

## 附录：常用着色器语义

| 语义 | 说明 |
|------|------|
| `POSITION0` | 顶点位置 |
| `TEXCOORD0` | 纹理坐标 |
| `NORMAL0` | 法线 |
| `TANGENT0` | 切线 |
| `SV_POSITION` | 裁剪空间位置（系统值） |
| `SV_Target` | 像素着色器输出颜色 |

## 附录：调试技巧

### 着色器编译错误

查看编译输出日志，常见错误：

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 未定义的变量 | 拼写错误或缺少 include | 检查变量名和头文件 |
| 语义不匹配 | 输入输出结构不一致 | 确保 VS 输出 = PS 输入 |
| 常量缓冲区对齐 | 16字节对齐问题 | 添加 padding 字段 |

### 渲染问题排查

1. **物体不显示** - 检查变换矩阵、相机视锥体
2. **颜色错误** - 检查着色器计算逻辑
3. **材质不加载** - 检查 JSON 格式和路径

---

**文档时间**: 2026-04-14
**风格参考**: Catlike Coding (https://catlikecoding.com/)



---

## 系列总结

恭喜完成引擎基础系列教程！你已学习：

| 教程 | 核心知识点 |
|------|-----------|
| RotateComponent | 组件架构、生命周期、Transform、四元数 |
| PulseShader | 着色器管线、材质系统、Uniform 参数、常量缓冲区 |


### 下一步学习方向

1. **渲染入门系列** - 学习 RHI、着色器
2. **脚本系统系列** - 学习 C# 脚本绑定
3. **动画系统系列** - 学习骨骼动画

---

## 附录：常用路径

| 系统 | 路径 |
|------|------|
| 组件基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |
| 物理系统 | `Engine/Source/Runtime/Function/Physics/` |
| 时间管理 | `Engine/Source/Runtime/Core/Time/` |
| 事件系统 | `Engine/Source/Runtime/Core/Tools/Eventing/` |
| 脚本系统 | `Engine/Source/Runtime/Function/Scripting/` |
| C# 脚本核心 | `Engine/Source/ScriptCore/Source/` |
| RTTR 类型注册 | `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h` |

---

**文档时间**: 2026-04-11
**风格参考**: Catlike Coding (https://catlikecoding.com/)
