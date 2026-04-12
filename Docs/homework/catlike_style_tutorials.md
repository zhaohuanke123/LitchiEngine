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

## 第四节：测试与验证

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

---

## 下一步

完成本教程后，继续学习：

**教程 2：向量数学应用 - MovingPlatform**

学习 Vector3 的更多操作，实现物体在两点之间移动。

---

# 教程 2：向量数学应用 - MovingPlatform

## 概述

通过创建一个在两点之间来回移动的平台组件，学习：

- Vector3 向量运算
- 线性插值 (Lerp)
- 时间控制与动画

**预期成果**：一个能在两点之间平滑移动的 MovingPlatform 组件。

## 前置要求

- 完成教程 1：组件系统入门
- 理解基本的向量概念
- 了解插值的概念

---

## 第一节：向量基础回顾

### 什么是向量？

在游戏开发中，Vector3 表示三维空间中的点或方向：

```cpp
Vector3 position(0, 1, 0);    // 位置：原点上方 1 单位
Vector3 direction(1, 0, 0);   // 方向：指向 X 轴正方向
```

### 向量运算

LitchiEngine 的 Vector3 支持常见运算：

```cpp
Vector3 a(1, 2, 3);
Vector3 b(4, 5, 6);

Vector3 sum = a + b;        // (5, 7, 9)  加法
Vector3 diff = b - a;       // (3, 3, 3)  减法
Vector3 scaled = a * 2.0f;  // (2, 4, 6)  标量乘法
float dist = a.Distance(b); // 距离计算
```

### 线性插值 (Lerp)

Lerp 是在两个值之间平滑过渡的核心方法：

```cpp
// Lerp 公式
result = start + (end - start) * t;

// Vector3::Lerp
Vector3 result = Vector3::Lerp(pointA, pointB, t);
```

当 `t = 0` 时，结果是 `pointA`；当 `t = 1` 时，结果是 `pointB`。

---

## 第二节：创建基础组件

### 第一步：创建头文件

在 `Gameplay/` 目录创建 `MovingPlatform.h`：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class MovingPlatform : public Component
    {
    public:
        MovingPlatform() = default;
        ~MovingPlatform() override = default;

        void OnUpdate() override;

        RTTR_ENABLE(Component)
    };
}
```

### 第二步：创建实现文件

创建 `MovingPlatform.cpp`：

```cpp
#include "MovingPlatform.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"
#include "Runtime/Core/Time/Time.h"

namespace LitchiRuntime
{
    void MovingPlatform::OnUpdate()
    {
        // TODO: 实现移动逻辑
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<MovingPlatform>("MovingPlatform")
            .constructor<>();
    }
}
```

**验证**: 编译项目，确保没有错误。

---

## 第三节：添加移动属性

### 问题：如何定义移动路径？

我们需要：
1. 起点 (pointA)
2. 终点 (pointB)
3. 移动速度
4. 记录当前位置

### 第三步：添加属性

更新头文件：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class MovingPlatform : public Component
    {
    public:
        MovingPlatform() = default;
        ~MovingPlatform() override = default;

        // 新增属性
        Vector3 pointA{0.0f, 0.0f, 0.0f};      // 起点（世界坐标）
        Vector3 pointB{0.0f, 2.0f, 0.0f};      // 终点（世界坐标）
        float speed = 1.0f;                      // 移动速度

        void OnUpdate() override;

    private:
        float m_progress = 0.0f;  // 当前进度 (0-1)
        bool m_movingToB = true;  // 移动方向

        RTTR_ENABLE(Component)
    };
}
```

### 第四步：注册属性

更新 RTTR 注册：

```cpp
RTTR_REGISTRATION
{
    rttr::registration::class_<MovingPlatform>("MovingPlatform")
        .constructor<>()
        .property("pointA", &MovingPlatform::pointA)
        .property("pointB", &MovingPlatform::pointB)
        .property("speed", &MovingPlatform::speed);
}
```

---

## 第四节：实现移动逻辑

### 理解移动模式

平台在 A 和 B 之间来回移动：

```
A -----> B
    t: 0 to 1

B -----> A
    t: 1 to 0
```

### 第五步：实现进度更新

更新 `OnUpdate`：

```cpp
void MovingPlatform::OnUpdate()
{
    Transform* transform = GetGameObject()->GetTransform();
    if (!transform) return;

    float dt = Time::GetDeltaTime();

    // 更新进度
    m_progress += dt * speed;

    // 检查是否到达终点
    if (m_progress >= 1.0f)
    {
        m_progress = 0.0f;
        m_movingToB = !m_movingToB;  // 反向
    }

    // 计算实际的插值参数
    float t = m_movingToB ? m_progress : (1.0f - m_progress);

    // 应用位置
    Vector3 newPos = Vector3::Lerp(pointA, pointB, t);
    transform->SetPosition(newPos);
}
```

**问题**：这种实现有个缺陷——速度会受帧率影响。让我们修正它。

### 第六步：改进实现

上面的实现有一个问题：速度应该表示"从 A 到 B 需要多少秒"，而不是模糊的"速度值"。

```cpp
void MovingPlatform::OnUpdate()
{
    Transform* transform = GetGameObject()->GetTransform();
    if (!transform) return;

    float dt = Time::GetDeltaTime();

    // 累计时间
    m_progress += dt * speed;

    // 使用往返插值
    float t = m_progress;
    if (t > 1.0f)
    {
        t = 2.0f - t;  // 反向
    }
    if (t > 2.0f)
    {
        t = 0.0f;
        m_progress = 0.0f;
    }

    Vector3 newPos = Vector3::Lerp(pointA, pointB, t);
    transform->SetPosition(newPos);
}
```

### 第七步：使用 PingPong 数学

更优雅的实现使用数学函数：

```cpp
void MovingPlatform::OnUpdate()
{
    Transform* transform = GetGameObject()->GetTransform();
    if (!transform) return;

    float dt = Time::GetDeltaTime();

    // 累计时间
    m_progress += dt * speed;

    // PingPong 效果：t 在 0-1 之间来回
    float t = m_progress - (int)m_progress;  // 取小数部分
    int cycle = (int)m_progress;

    if (cycle % 2 == 1)
    {
        t = 1.0f - t;  // 奇数周期反向
    }

    Vector3 newPos = Vector3::Lerp(pointA, pointB, t);
    transform->SetPosition(newPos);
}
```

---

## 第五节：完整代码

### 头文件

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class MovingPlatform : public Component
    {
    public:
        MovingPlatform() = default;
        ~MovingPlatform() override = default;

        Vector3 pointA{0.0f, 0.0f, 0.0f};
        Vector3 pointB{0.0f, 2.0f, 0.0f};
        float speed = 1.0f;

        void OnUpdate() override;

    private:
        float m_progress = 0.0f;

        RTTR_ENABLE(Component)
    };
}
```

### 实现文件

```cpp
#include "MovingPlatform.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"
#include "Runtime/Core/Time/Time.h"

namespace LitchiRuntime
{
    void MovingPlatform::OnUpdate()
    {
        Transform* transform = GetGameObject()->GetTransform();
        if (!transform) return;

        float dt = Time::GetDeltaTime();
        m_progress += dt * speed;

        // PingPong 插值
        float t = m_progress - (int)m_progress;
        if (((int)m_progress) % 2 == 1)
        {
            t = 1.0f - t;
        }

        Vector3 newPos = Vector3::Lerp(pointA, pointB, t);
        transform->SetPosition(newPos);
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<MovingPlatform>("MovingPlatform")
            .constructor<>()
            .property("pointA", &MovingPlatform::pointA)
            .property("pointB", &MovingPlatform::pointB)
            .property("speed", &MovingPlatform::speed);
    }
}
```

---

## 练习

### 1. 基础练习：暂停功能

添加 `bool paused` 属性，控制平台是否移动。

### 2. 进阶练习：缓动效果

使用 SmoothStep 代替线性插值，让平台在起点和终点处有缓入缓出效果。

**提示**：
```cpp
float smoothT = t * t * (3 - 2 * t);  // SmoothStep
```

### 3. 挑战：多路径点

扩展组件支持多个路径点，而不仅是 A、B 两点。

---

## 验证标准

- [ ] 平台在 pointA 和 pointB 之间移动
- [ ] 移动速度与帧率无关
- [ ] 编辑器中可配置 pointA、pointB、speed
- [ ] 速度改变后移动频率相应变化

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| Vector3::Lerp | 向量线性插值 |
| 时间累加 | 用 deltaTime 控制动画进度 |
| PingPong 模式 | 实现来回往复运动 |
| 帧无关动画 | 使用 deltaTime 确保一致性 |

---

## 下一步

继续学习：

**教程 3：物理系统入门 - TriggerZone**

学习如何使用物理系统检测碰撞和触发事件。

---

# 教程 3：物理系统入门 - TriggerZone

## 概述

通过创建一个触发区域组件，学习：

- 物理碰撞器 (Collider)
- 触发器事件 (Trigger)
- 组件间协作

**预期成果**：一个能检测物体进入并触发事件的 TriggerZone 组件。

## 前置要求

- 完成教程 1、2
- 了解基本的物理概念（碰撞、触发器）
- 理解事件驱动编程

---

## 第一节：理解物理系统

### 碰撞器 vs 触发器

| 类型 | 特点 | 用途 |
|------|------|------|
| 碰撞器 | 产生物理碰撞，阻止物体穿过 | 墙壁、地面、障碍物 |
| 触发器 | 只检测重叠，不产生碰撞 | 检测区域、传送门、收集品 |

### LitchiEngine 的物理组件

```
Engine/Source/Runtime/Function/Framework/Component/Physcis/
├── collider.h          # 碰撞器基类
├── BoxCollider.h       # 盒形碰撞器
├── SphereCollider.h    # 球形碰撞器
├── RigidStatic.h       # 静态刚体（不动）
└── RigidDynamic.h      # 动态刚体（可移动、受力）
```

### 触发器的工作原理

1. 触发器需要有 Collider 组件（设置为 Trigger）
2. 触发器需要 RigidActor（Static 或 Dynamic）
3. 进入触发器的物体也需要 Collider

---

## 第二节：创建基础结构

### 第一步：创建头文件

创建 `TriggerZone.h`：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class GameObject;

    class TriggerZone : public Component
    {
    public:
        TriggerZone() = default;
        ~TriggerZone() override = default;

        // 事件：当物体进入时触发
        Event<GameObject*> OnPlayerEnter;

        void OnAwake() override;
        void OnTriggerEnter(GameObject* other) override;

        RTTR_ENABLE(Component)
    };
}
```

### 第二步：创建实现文件

创建 `TriggerZone.cpp`：

```cpp
#include "TriggerZone.h"
#include "Runtime/Function/Framework/Component/Physcis/BoxCollider.h"
#include "Runtime/Function/Framework/Component/Physcis/RigidStatic.h"

namespace LitchiRuntime
{
    void TriggerZone::OnAwake()
    {
        // TODO: 设置碰撞器
    }

    void TriggerZone::OnTriggerEnter(GameObject* other)
    {
        // TODO: 触发事件
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<TriggerZone>("TriggerZone")
            .constructor<>();
    }
}
```

---

## 第三节：自动配置碰撞器

### 问题：触发器需要碰撞器

TriggerZone 本身只是一个逻辑组件，它需要 Collider 来检测物理重叠。我们可以：

1. 要求用户手动添加 Collider —— 麻烦
2. 自动添加所需组件 —— 更好的体验

### 第三步：在 OnAwake 中配置

```cpp
void TriggerZone::OnAwake()
{
    // 检查或添加 BoxCollider
    BoxCollider* collider = GetGameObject()->GetComponent<BoxCollider>();
    if (!collider)
    {
        collider = GetGameObject()->AddComponent<BoxCollider>();
    }

    // 设为触发器
    collider->SetIsTrigger(true);

    // 检查或添加 RigidStatic（触发器不需要物理模拟）
    RigidStatic* rigid = GetGameObject()->GetComponent<RigidStatic>();
    if (!rigid)
    {
        GetGameObject()->AddComponent<RigidStatic>();
    }
}
```

**为什么要添加 RigidStatic？**

在 PhysX 中，触发器需要 RigidActor 作为物理实体。RigidStatic 表示这个物体不会移动（不受物理影响），适合作为触发区域。

---

## 第四节：处理触发事件

### 第四步：实现 OnTriggerEnter

```cpp
void TriggerZone::OnTriggerEnter(GameObject* other)
{
    // 触发事件
    OnPlayerEnter.Invoke(other);

    // 输出日志
    DEBUG_LOG_INFO("物体进入触发区域: {}", other->GetName());
}
```

### Event 类的使用

LitchiEngine 的 Event 类提供了发布-订阅模式：

```cpp
// 定义事件
Event<GameObject*> OnPlayerEnter;

// 触发事件
OnPlayerEnter.Invoke(someGameObject);

// 订阅事件（在其他组件中）
triggerZone->OnPlayerEnter += [](GameObject* obj) {
    // 处理逻辑
};
```

---

## 第五节：完整代码

### 头文件

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class GameObject;

    class TriggerZone : public Component
    {
    public:
        TriggerZone() = default;
        ~TriggerZone() override = default;

        Event<GameObject*> OnPlayerEnter;

        void OnAwake() override;
        void OnTriggerEnter(GameObject* other) override;

        RTTR_ENABLE(Component)
    };
}
```

### 实现文件

```cpp
#include "TriggerZone.h"
#include "Runtime/Function/Framework/Component/Physcis/BoxCollider.h"
#include "Runtime/Function/Framework/Component/Physcis/RigidStatic.h"
#include "Runtime/Core/Log/Log.h"

namespace LitchiRuntime
{
    void TriggerZone::OnAwake()
    {
        BoxCollider* collider = GetGameObject()->GetComponent<BoxCollider>();
        if (!collider)
        {
            collider = GetGameObject()->AddComponent<BoxCollider>();
        }
        collider->SetIsTrigger(true);

        RigidStatic* rigid = GetGameObject()->GetComponent<RigidStatic>();
        if (!rigid)
        {
            GetGameObject()->AddComponent<RigidStatic>();
        }
    }

    void TriggerZone::OnTriggerEnter(GameObject* other)
    {
        OnPlayerEnter.Invoke(other);
        DEBUG_LOG_INFO("物体进入触发区域: {}", other->GetName());
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<TriggerZone>("TriggerZone")
            .constructor<>();
    }
}
```

---

## 第六节：测试

### 测试场景设置

1. 创建一个 Cube 作为触发区域
2. 添加 TriggerZone 组件（会自动添加 BoxCollider 和 RigidStatic）
3. 创建另一个 Cube 作为玩家
4. 给玩家添加 RigidDynamic（让物理引擎处理它）
5. 运行场景，让玩家落入触发区域

### 预期效果

- 控制台输出 "物体进入触发区域" 日志
- 玩家穿过触发区域（不产生物理碰撞）

---

## 练习

### 1. 基础练习：添加离开事件

添加 `OnTriggerExit` 的处理和 `OnPlayerExit` 事件。

### 2. 进阶练习：标签过滤

添加 `std::string targetTag` 属性，只响应特定标签的物体。

```cpp
void TriggerZone::OnTriggerEnter(GameObject* other)
{
    if (targetTag.empty() || other->GetTag() == targetTag)
    {
        OnPlayerEnter.Invoke(other);
    }
}
```

### 3. 挑战：计数触发器

添加计数功能，记录当前在触发区域内的物体数量。

```cpp
private:
    std::set<GameObject*> m_objectsInside;

public:
    int GetCount() const { return m_objectsInside.size(); }
```

---

## 验证标准

- [ ] 添加 TriggerZone 后自动添加所需组件
- [ ] 触发区域不产生物理碰撞
- [ ] 物体进入时触发事件
- [ ] 控制台输出正确日志

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| Collider | 定义物理形状 |
| IsTrigger | 设为触发器模式 |
| RigidStatic | 静态物理体 |
| OnTriggerEnter | 触发器进入回调 |
| Event<> | 事件系统，组件间通信 |

---

## 下一步

继续学习：

**教程 4：事件系统应用 - TimerTrigger**

深入学习事件系统，创建计时触发器组件。

---

# 教程 4：事件系统应用 - TimerTrigger

## 概述

通过创建一个计时触发器组件，深入学习：

- Event 事件系统
- 订阅/触发事件
- 组件间通信

**预期成果**：一个可配置的计时器，时间到后触发事件。

## 前置要求

- 完成教程 1、2、3
- 理解回调函数概念
- 了解观察者模式

---

## 第一节：理解事件系统

### 为什么需要事件系统？

传统的问题：组件 A 想通知组件 B

```cpp
// 不好的方式：直接引用
class ComponentA {
    ComponentB* b;
    void DoSomething() {
        b->OnSomethingHappened();  // 紧耦合
    }
};
```

使用事件的优势：

```cpp
// 好的方式：使用事件
class ComponentA {
    Event<> OnSomething;
    void DoSomething() {
        OnSomething.Invoke();  // 松耦合
    }
};

// 组件 B 订阅事件
a->OnSomething += []() { /* 处理 */ };
```

### LitchiEngine 的 Event 类

```cpp
template<class... ArgTypes>
class Event {
public:
    // 添加监听器
    ListenerID AddListener(Callback callback);
    ListenerID operator+=(Callback callback);

    // 移除监听器
    bool RemoveListener(ListenerID id);
    bool operator-=(ListenerID id);

    // 触发事件
    void Invoke(ArgTypes... args);
};
```

---

## 第二节：创建计时器组件

### 第一步：创建头文件

创建 `TimerTrigger.h`：

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class TimerTrigger : public Component
    {
    public:
        TimerTrigger() = default;
        ~TimerTrigger() override = default;

        void OnUpdate() override;

    private:
        RTTR_ENABLE(Component)
    };
}
```

### 第二步：添加计时属性

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class TimerTrigger : public Component
    {
    public:
        TimerTrigger() = default;
        ~TimerTrigger() override = default;

        // 计时时长（秒）
        float duration = 3.0f;

        // 是否循环
        bool loop = false;

        // 事件：时间到
        Event<> OnTimerEnd;

        void OnUpdate() override;

        // 控制方法
        void Start();
        void Stop();
        void Reset();

    private:
        float m_elapsed = 0.0f;
        bool m_running = false;

        RTTR_ENABLE(Component)
    };
}
```

---

## 第三节：实现计时逻辑

### 第三步：实现控制方法

创建 `TimerTrigger.cpp`：

```cpp
#include "TimerTrigger.h"
#include "Runtime/Core/Time/Time.h"

namespace LitchiRuntime
{
    void TimerTrigger::Start()
    {
        m_running = true;
        m_elapsed = 0.0f;
    }

    void TimerTrigger::Stop()
    {
        m_running = false;
    }

    void TimerTrigger::Reset()
    {
        m_elapsed = 0.0f;
    }
}
```

### 第四步：实现 OnUpdate

```cpp
void TimerTrigger::OnUpdate()
{
    if (!m_running) return;

    m_elapsed += Time::GetDeltaTime();

    if (m_elapsed >= duration)
    {
        // 触发事件
        OnTimerEnd.Invoke();

        if (loop)
        {
            m_elapsed = 0.0f;
        }
        else
        {
            m_running = false;
        }
    }
}
```

### 第五步：RTTR 注册

```cpp
RTTR_REGISTRATION
{
    rttr::registration::class_<TimerTrigger>("TimerTrigger")
        .constructor<>()
        .property("duration", &TimerTrigger::duration)
        .property("loop", &TimerTrigger::loop);
}
```

---

## 第四节：使用示例

### 场景：延时启动

结合 TriggerZone，创建一个延时触发的门：

```cpp
// 在某个组件中
void DoorController::OnAwake()
{
    // 找到 TimerTrigger
    TimerTrigger* timer = GetGameObject()->GetComponent<TimerTrigger>();

    // 订阅事件
    m_listenerId = timer->OnTimerEnd += [this]() {
        OpenDoor();
    };
}

void DoorController::OnDestroy()
{
    // 清理监听器
    TimerTrigger* timer = GetGameObject()->GetComponent<TimerTrigger>();
    if (timer)
    {
        timer->OnTimerEnd -= m_listenerId;
    }
}
```

### 场景：循环事件

创建闪烁的灯光：

```cpp
void LightBlinker::OnAwake()
{
    TimerTrigger* timer = GetGameObject()->GetComponent<TimerTrigger>();
    timer->loop = true;
    timer->duration = 0.5f;

    timer->OnTimerEnd += [this]() {
        ToggleLight();
    };

    timer->Start();
}
```

---

## 第五节：完整代码

### 头文件

```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class TimerTrigger : public Component
    {
    public:
        TimerTrigger() = default;
        ~TimerTrigger() override = default;

        float duration = 3.0f;
        bool loop = false;

        Event<> OnTimerEnd;

        void OnUpdate() override;

        void Start();
        void Stop();
        void Reset();

        bool IsRunning() const { return m_running; }
        float GetProgress() const { return m_elapsed / duration; }

    private:
        float m_elapsed = 0.0f;
        bool m_running = false;

        RTTR_ENABLE(Component)
    };
}
```

### 实现文件

```cpp
#include "TimerTrigger.h"
#include "Runtime/Core/Time/Time.h"

namespace LitchiRuntime
{
    void TimerTrigger::Start()
    {
        m_running = true;
        m_elapsed = 0.0f;
    }

    void TimerTrigger::Stop()
    {
        m_running = false;
    }

    void TimerTrigger::Reset()
    {
        m_elapsed = 0.0f;
    }

    void TimerTrigger::OnUpdate()
    {
        if (!m_running) return;

        m_elapsed += Time::GetDeltaTime();

        if (m_elapsed >= duration)
        {
            OnTimerEnd.Invoke();

            if (loop)
            {
                m_elapsed = 0.0f;
            }
            else
            {
                m_running = false;
            }
        }
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<TimerTrigger>("TimerTrigger")
            .constructor<>()
            .property("duration", &TimerTrigger::duration)
            .property("loop", &TimerTrigger::loop);
    }
}
```

---

## 练习

### 1. 基础练习：进度访问

添加 `GetProgress()` 方法，返回当前进度 (0-1)。

### 2. 进阶练习：带参数的事件

修改事件为 `Event<float>`，触发时传递剩余时间或当前进度。

### 3. 挑战：计时器组

创建 `TimerGroup` 组件，管理多个计时器，支持序列触发。

```cpp
class TimerGroup : public Component {
public:
    std::vector<float> durations;
    int currentIndex = 0;
    Event<int> OnTimerComplete;  // 传递完成的计时器索引
};
```

---

## 验证标准

- [ ] 计时器能正常计时
- [ ] 时间到触发事件
- [ ] 循环模式正常工作
- [ ] Start/Stop/Reset 方法正常
- [ ] 事件订阅者能正确响应

---

## 总结

本教程学习了：

| 知识点 | 说明 |
|--------|------|
| Event<> | 事件类，实现观察者模式 |
| += 订阅 | 添加事件监听器 |
| -= 取消订阅 | 移除事件监听器 |
| Invoke | 触发事件 |
| 松耦合 | 通过事件解耦组件 |

---

## 系列总结

恭喜完成引擎基础系列教程！你已学习：

| 教程 | 核心知识点 |
|------|-----------|
| RotateComponent | 组件架构、生命周期、Transform、四元数 |
| MovingPlatform | 向量数学、插值、时间控制 |
| TriggerZone | 物理系统、碰撞器、触发器 |
| TimerTrigger | 事件系统、组件通信 |

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

---

**文档时间**: 2026-04-11
**风格参考**: Catlike Coding (https://catlikecoding.com/)
