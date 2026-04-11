# LitchiEngine 实习生引擎开发体验作业

## 概述

本作业通过给引擎添加小功能的方式，帮助实习生学习引擎各系统的运作方式。每个作业聚焦一个系统，体验完整的引擎开发流程。

---

## 作业 1：添加旋转组件 - 学习组件系统

### 学习目标

通过添加一个 `RotateComponent`，学习：
- 组件的基本结构
- 组件生命周期回调
- RTTR 反射注册
- Transform 操作

### 功能描述

创建一个旋转组件，挂载后让物体持续旋转。

### 开发流程

#### 第一步：参考现有组件

阅读引擎中已有的简单组件，理解组件结构：

```
Engine/Source/Runtime/Function/Framework/Component/
├── Base/Component.h          # 组件基类
├── Gameplay/                  # 简单功能组件放这里
└── Transform/Transform.h      # Transform 是必读的
```

重点看：
- `Component.h` - 理解基类有哪些生命周期方法
- `Transform.h` - 理解如何操作位置/旋转

#### 第二步：创建文件

在 `Gameplay/` 目录创建两个文件：

**RotateComponent.h**
```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class RotateComponent : public Component
    {
    public:
        RotateComponent();
        ~RotateComponent() override;

        // 旋转速度（度/秒）
        Vector3 rotationSpeed{0.0f, 45.0f, 0.0f};

        void OnUpdate() override;

        RTTR_ENABLE(Component)
    };
}
```

**RotateComponent.cpp**
```cpp
#include "RotateComponent.h"
#include "Runtime/Core/Time/Time.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"
#include "Runtime/Core/Math/Quaternion.h"
#include "Runtime/Core/Math/MathHelper.h"

namespace LitchiRuntime
{
    RotateComponent::RotateComponent() {}
    RotateComponent::~RotateComponent() {}

    void RotateComponent::OnUpdate()
    {
        Transform* transform = GetGameObject()->GetTransform();
        if (!transform) return;

        float dt = Time::GetDeltaTime();
        
        // 获取当前旋转，添加增量
        Quaternion rot = transform->GetRotationLocal();
        Quaternion delta = Quaternion::FromAngleAxis(
            rotationSpeed.y * Math::Helper::DEG_TO_RAD * dt, 
            Vector3::Up
        );
        transform->SetRotationLocal(rot * delta);
    }

    RTTR_REGISTRATION
    {
        rttr::registration::class_<RotateComponent>("RotateComponent")
            .constructor<>()
            .property("rotationSpeed", &RotateComponent::rotationSpeed);
    }
}
```

#### 第三步：添加到构建系统

修改 `Engine/Source/Runtime/CMakeLists.txt`，在源文件列表中添加：

```cmake
Function/Framework/Component/Gameplay/RotateComponent.cpp
```

#### 第四步：编译测试

1. 重新编译引擎
2. 打开编辑器
3. 创建一个 Cube
4. 添加 RotateComponent
5. 运行，观察物体是否旋转

### 验收标准

- [ ] 编译通过
- [ ] 编辑器 Inspector 中能看到 RotateComponent
- [ ] 能添加组件到 GameObject
- [ ] 物体能旋转

### 学到了什么

| 知识点 | 说明 |
|--------|------|
| Component 基类 | 所有组件继承自 Component |
| 生命周期 | OnUpdate 每帧调用 |
| RTTR 注册 | 让编辑器识别组件和属性 |
| Transform | 通过 GetTransform() 操作物体变换 |

---

## 作业 2：添加移动平台组件 - 学习向量数学

### 学习目标

通过添加一个 `MovingPlatform` 组件，学习：
- Vector3 向量运算
- Lerp 插值
- 时间控制

### 功能描述

创建一个在两点之间来回移动的平台组件。

### 开发流程

#### 第一步：理解向量运算

阅读 `Engine/Source/Runtime/Core/Math/Vector3.h`，理解：
- 向量加减
- 向量插值 (Lerp)
- 距离计算

#### 第二步：创建组件

**MovingPlatform.h**
```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Math/Vector3.h"

namespace LitchiRuntime
{
    class MovingPlatform : public Component
    {
    public:
        MovingPlatform();

        // 起点、终点
        Vector3 pointA{0, 0, 0};
        Vector3 pointB{0, 2, 0};
        
        // 移动速度
        float speed = 1.0f;

        void OnUpdate() override;

    private:
        float m_time = 0.0f;
        bool m_toB = true;

        RTTR_ENABLE(Component)
    };
}
```

**MovingPlatform.cpp**
```cpp
#include "MovingPlatform.h"
#include "Runtime/Core/Time/Time.h"
#include "Runtime/Function/Framework/Component/Transform/Transform.h"

namespace LitchiRuntime
{
    MovingPlatform::MovingPlatform() {}

    void MovingPlatform::OnUpdate()
    {
        Transform* transform = GetGameObject()->GetTransform();
        if (!transform) return;

        // 累加时间
        m_time += Time::GetDeltaTime() * speed;

        // 计算插值 (0-1 之间来回)
        float t = m_time;
        if (t > 1.0f)
        {
            t = 2.0f - t;  // 反向
            if (t < 0.0f)
            {
                t = 0.0f;
                m_time = 0.0f;
            }
        }

        // 在两点间插值
        Vector3 pos = Vector3::Lerp(pointA, pointB, t);
        transform->SetPosition(pos);
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

#### 第三步：测试

1. 创建一个 Cube 作为平台
2. 添加 MovingPlatform
3. 设置 pointA 和 pointB
4. 运行观察平台来回移动

### 验收标准

- [ ] 平台能在两点间移动
- [ ] 修改 pointA/pointB 能改变移动路径
- [ ] 修改 speed 能改变移动速度

### 学到了什么

| 知识点 | 说明 |
|--------|------|
| Vector3::Lerp | 向量线性插值 |
| 时间累加 | 用 deltaTime 控制动画 |
| 属性暴露 | 公有成员变量可被 RTTR 注册 |

---

## 作业 3：添加触发区域组件 - 学习物理系统

### 学习目标

通过添加一个 `TriggerZone` 组件，学习：
- 物理碰撞器
- 触发器事件
- OnTriggerEnter 回调

### 功能描述

创建一个触发区域，当玩家进入时触发事件。

### 开发流程

#### 第一步：理解物理系统

阅读以下文件：
- `Engine/Source/Runtime/Function/Physics/physics.h` - 物理核心 API
- `Engine/Source/Runtime/Function/Framework/Component/Physcis/Collider.h` - 碰撞器基类

重点理解：
- Collider 组件如何工作
- SetIsTrigger(true) 的作用
- OnTriggerEnter 何时被调用

#### 第二步：创建组件

**TriggerZone.h**
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
        TriggerZone();

        // 触发事件
        Event<GameObject*> OnPlayerEnter;

        void OnAwake() override;
        void OnTriggerEnter(GameObject* other) override;

        RTTR_ENABLE(Component)
    };
}
```

**TriggerZone.cpp**
```cpp
#include "TriggerZone.h"
#include "Runtime/Function/Framework/Component/Physcis/BoxCollider.h"
#include "Runtime/Function/Framework/Component/Physcis/RigidStatic.h"

namespace LitchiRuntime
{
    TriggerZone::TriggerZone() {}

    void TriggerZone::OnAwake()
    {
        // 自动添加碰撞器（如果没有）
        BoxCollider* collider = GetGameObject()->GetComponent<BoxCollider>();
        if (!collider)
        {
            collider = GetGameObject()->AddComponent<BoxCollider>();
        }
        
        // 设为触发器
        collider->SetIsTrigger(true);

        // 静态刚体（触发器不需要物理模拟）
        RigidStatic* rigid = GetGameObject()->GetComponent<RigidStatic>();
        if (!rigid)
        {
            GetGameObject()->AddComponent<RigidStatic>();
        }
    }

    void TriggerZone::OnTriggerEnter(GameObject* other)
    {
        // 触发事件
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

#### 第三步：测试

1. 创建一个 Cube 作为触发区域
2. 添加 TriggerZone（会自动添加 BoxCollider 和 RigidStatic）
3. 创建另一个带 RigidDynamic 的物体
4. 运行，让物体进入触发区域

### 验收标准

- [ ] 组件自动添加所需的 Collider
- [ ] 物体进入时触发事件
- [ ] 控制台输出日志

### 学到了什么

| 知识点 | 说明 |
|--------|------|
| Collider | 碰撞器定义物理形状 |
| IsTrigger | 触发器只检测不产生碰撞 |
| OnTriggerEnter | 触发器进入回调 |
| 组件组合 | 一个物体可以有多个组件协同工作 |

---

## 作业 4：添加计时触发器 - 学习事件系统

### 学习目标

通过添加一个 `TimerTrigger` 组件，学习：
- Event 事件系统
- 订阅/触发事件
- 组件间通信

### 功能描述

创建一个计时器，时间到后触发事件，可让其他组件响应。

### 开发流程

#### 第一步：理解事件系统

阅读 `Engine/Source/Runtime/Core/Tools/Eventing/Event.h`，理解：
- 如何定义事件
- 如何订阅事件 (+=)
- 如何触发事件 (Invoke)

#### 第二步：创建组件

**TimerTrigger.h**
```cpp
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"
#include "Runtime/Core/Tools/Eventing/Event.h"

namespace LitchiRuntime
{
    class TimerTrigger : public Component
    {
    public:
        TimerTrigger();

        // 计时时长
        float duration = 3.0f;
        
        // 是否循环
        bool loop = false;

        // 事件：时间到
        Event<> OnTimerEnd;

        void OnUpdate() override;

        void Start();
        void Stop();

    private:
        float m_elapsed = 0.0f;
        bool m_running = false;

        RTTR_ENABLE(Component)
    };
}
```

**TimerTrigger.cpp**
```cpp
#include "TimerTrigger.h"
#include "Runtime/Core/Time/Time.h"

namespace LitchiRuntime
{
    TimerTrigger::TimerTrigger() {}

    void TimerTrigger::Start()
    {
        m_running = true;
        m_elapsed = 0.0f;
    }

    void TimerTrigger::Stop()
    {
        m_running = false;
    }

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

    RTTR_REGISTRATION
    {
        rttr::registration::class_<TimerTrigger>("TimerTrigger")
            .constructor<>()
            .property("duration", &TimerTrigger::duration)
            .property("loop", &TimerTrigger::loop);
    }
}
```

#### 第三步：使用示例

在另一个组件中使用：

```cpp
void SomeComponent::OnAwake()
{
    TimerTrigger* timer = GetGameObject()->GetComponent<TimerTrigger>();
    if (timer)
    {
        // 订阅事件
        timer->OnTimerEnd += []() {
            DEBUG_LOG_INFO("时间到！");
        };
        timer->Start();
    }
}
```

### 验收标准

- [ ] 计时器能正常计时
- [ ] 时间到触发事件
- [ ] 循环模式正常工作

### 学到了什么

| 知识点 | 说明 |
|--------|------|
| Event<> | 事件类，用于组件间通信 |
| += 订阅 | 添加事件监听器 |
| Invoke | 触发事件 |

---

## 附录：常用文件路径

| 系统 | 路径 |
|------|------|
| 组件基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |
| 物理系统 | `Engine/Source/Runtime/Function/Physics/` |
| 时间管理 | `Engine/Source/Runtime/Core/Time/` |
| 事件系统 | `Engine/Source/Runtime/Core/Tools/Eventing/` |

---

**文档时间**: 2026-04-10
