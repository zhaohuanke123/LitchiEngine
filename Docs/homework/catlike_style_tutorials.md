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
| 引擎基础 | 教程 2: 向量数学应用 | 入门 | 教程 1 |
| 引擎基础 | 教程 3: 物理系统入门 | 进阶 | 教程 2 |
| 引擎基础 | 教程 4: 事件系统应用 | 进阶 | 教程 3 |

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

## 下一步



---

## 系列总结

恭喜完成引擎基础系列教程！你已学习：

| 教程 | 核心知识点 |
|------|-----------|
| RotateComponent | 组件架构、生命周期、Transform、四元数 |


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
