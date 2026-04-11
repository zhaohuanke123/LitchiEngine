# LitchiEngine 引擎原理学习教程系列

## 系列概述

本教程系列采用 **读源码优先** 的教学理念，通过引导实习生阅读引擎源码、理解原理，再通过实践验证理解。

### 教学原则

| 原则 | 说明 |
|------|------|
| 读源码优先 | 每个知识点先引导阅读引擎关键代码，指出文件路径和关键行 |
| 理解原理 | 问题围绕"为什么这样设计"、"是怎么实现的" |
| 实践验证 | 添加功能是为了验证理解，不是复制粘贴 |
| 渐进式 | 从简单到复杂，每步只解决一个问题 |

### 教程结构模板

每个教程遵循以下结构：

```markdown
## 读源码
引导阅读引擎关键代码，指出文件路径和关键行。

## 理解原理
回答问题：
- 这段代码为什么这样写？
- 引擎是怎么实现这个功能的？
- 设计决策是什么？

## 实践验证
添加一个小功能，验证你理解了原理。

## 预期理解 & 验证方法
明确应该理解什么，以及如何验证理解正确。
```

### 系列规划

| 教程 | 引擎系统 | 核心问题 | 难度 |
|------|----------|----------|------|
| 1 | 组件系统 | 组件是怎么注册和管理的？生命周期怎么调度？ | 入门 |
| 2 | Transform 系统 | 父子层级怎么实现？世界/本地坐标怎么转换？ | 入门 |
| 3 | 物理系统 | 引擎怎么封装 PhysX？碰撞事件怎么传递？ | 进阶 |
| 4 | 渲染系统 | 渲染管线怎么工作？DrawCall 怎么提交？ | 进阶 |

### 前置要求

- 已成功编译 LitchiEngine
- 熟悉 C++17 基础语法
- 了解面向对象编程概念
- 准备好 IDE（推荐 Visual Studio 或 VS Code）

---

# 教程 1：组件系统原理 - RotateComponent

## 概述

通过创建一个旋转组件，深入理解 LitchiEngine 的组件系统架构。完成本教程后，你将能回答：

- 组件是如何被引擎识别和创建的？
- 生命周期回调是如何被调度的？
- RTTR 反射机制是如何工作的？

**预期成果**：一个能让物体持续旋转的 RotateComponent，并理解其背后的原理。

---

## 第一节：读源码 - Component 基类

### 打开 Component 基类

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Base/component.h`

**任务 1：找到 RTTR_ENABLE 宏**

在第 135 行，你会看到：

```cpp
RTTR_ENABLE(ScriptObject)
```

**问题思考**：
1. `RTTR_ENABLE` 是什么？为什么需要这个宏？
2. 为什么参数是 `ScriptObject` 而不是其他类？

**探索提示**：
- RTTR 是 Run Time Type Reflection 的缩写
- 搜索项目中其他使用 `RTTR_ENABLE` 的类，观察模式
- 查看 `Engine/ThirdParty/` 下的 RTTR 库文档

---

**任务 2：理解生命周期回调**

阅读第 53-98 行的生命周期方法声明：

```cpp
virtual void OnAwake();
virtual void OnEnable();
virtual void OnStart();
virtual void OnUpdate();
virtual void OnFixedUpdate();
virtual void OnLateUpdate();
virtual void OnDisable();
virtual void OnDestroy();
```

**问题思考**：
1. 为什么需要这么多生命周期方法？它们分别用于什么场景？
2. 为什么都是 `virtual` 函数？
3. 谁负责调用这些方法？

**探索提示**：
- 打开 `Engine/Source/Runtime/Function/Framework/GameObject/GameObject.cpp`
- 搜索 `OnAwake`、`OnUpdate` 等调用位置
- 理解调用链

---

### 打开 GameObject 实现

文件路径：`Engine/Source/Runtime/Function/Framework/GameObject/GameObject.cpp`

**任务 3：理解生命周期调度**

找到第 160-176 行的 `OnAwake`、`OnEnable`、`OnStart` 实现：

```cpp
void GameObject::OnAwake()
{
    m_awaked = true;
    std::for_each(m_componentList.begin(), m_componentList.end(), [](auto* element) { element->OnAwake(); });
}

void GameObject::OnEnable()
{
    std::for_each(m_componentList.begin(), m_componentList.end(), [](auto* element) { element->OnEnable(); });
}

void GameObject::OnStart()
{
    m_started = true;
    std::for_each(m_componentList.begin(), m_componentList.end(), [](auto* element) { element->OnStart(); });
}
```

**问题思考**：
1. `std::for_each` 在做什么？
2. 为什么 `OnAwake` 和 `OnStart` 要设置 `m_awaked` 和 `m_started` 标志？
3. 如果一个组件在 `OnAwake` 中禁用了自己，后续的 `OnEnable` 和 `OnStart` 会怎样？

---

**任务 4：理解组件列表管理**

找到第 346 行的成员变量：

```cpp
std::vector<Component*> m_componentList;
```

然后打开 `GameObject.inl`（在同一目录下），找到 `AddComponent` 的模板实现：

```cpp
template <class T>
T* GameObject::AddComponent()
{
    // ... 创建组件并添加到 m_componentList
}
```

**问题思考**：
1. 组件是如何被创建的？（提示：RTTR 的 `create` 方法）
2. 组件添加后，生命周期方法何时被调用？

---

## 第二节：理解原理 - RTTR 注册机制

### 打开 Transform.cpp 看注册示例

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Transform/transform.cpp`

在文件末尾，你会看到 RTTR 注册块：

```cpp
RTTR_REGISTRATION
{
    rttr::registration::class_<Transform>("Transform")
        .constructor<>()
        .property("m_localPosition", &Transform::m_localPosition)
        .property("m_localRotation", &Transform::m_localRotation)
        .property("m_localScale", &Transform::m_localScale);
}
```

**理解要点**：

| 代码 | 作用 |
|------|------|
| `RTTR_REGISTRATION` | 注册块开始，全局只执行一次 |
| `class_<Transform>("Transform")` | 注册类，"Transform" 是运行时名称 |
| `.constructor<>()` | 注册默认构造函数 |
| `.property("name", &Class::member)` | 注册属性，使其可序列化、可编辑 |

**为什么需要注册？**

1. **编辑器识别**：编辑器通过 RTTR 获取所有组件类型，显示在 "Add Component" 列表中
2. **序列化**：场景保存时，通过 RTTR 遍历属性并写入文件
3. **反序列化**：加载场景时，通过 RTTR 创建组件实例并恢复属性

---

## 第三节：实践验证 - 创建 RotateComponent

### 任务：创建一个最小组件

现在你已经理解了组件注册的原理，尝试创建一个 `RotateComponent`。

**要求**：
1. 不要直接复制代码，参考 `Transform.cpp` 自己写
2. 只包含 RTTR 注册，先不实现任何功能
3. 编译后在编辑器中验证组件是否出现

**步骤提示**：
1. 在 `Engine/Source/Runtime/Function/Framework/Component/Gameplay/` 创建 `RotateComponent.h`
2. 创建对应的 `RotateComponent.cpp`
3. 参考 Transform 的 RTTR 注册，写出自己的注册块
4. 修改 CMakeLists.txt 添加源文件

**验证方法**：
- 编译成功后，打开 LitchiEditor
- 创建一个 GameObject，点击 "Add Component"
- 在组件列表中找到 "RotateComponent"

---

### 任务：实现旋转功能

当最小组件验证成功后，添加旋转逻辑。

**阅读参考**：
- Transform 的旋转方法：`Engine/Source/Runtime/Function/Framework/Component/Transform/transform.h`
- Time 类：`Engine/Source/Runtime/Core/Time/Time.h`

**问题思考**：
1. 如何获取 deltaTime？
2. 如何获取当前 GameObject 的 Transform？
3. 如何修改旋转？

**验证方法**：
- 运行场景，观察物体是否旋转
- 在 Inspector 中修改属性，观察行为变化

---

## 第四节：深入理解 - 生命周期调度原理

### 读源码：Scene 如何驱动 GameObject

文件路径：`Engine/Source/Runtime/Function/Scene/SceneManager.cpp` 或 `Scene.cpp`

找到场景的主循环，观察它如何调用 GameObject 的生命周期方法。

**问题思考**：
1. 场景是如何遍历所有 GameObject 的？
2. 为什么 `OnFixedUpdate` 和 `OnUpdate` 是分开的？
3. 如果一个 GameObject 在 `OnUpdate` 中被删除，会发生什么？

---

## 预期理解 & 验证方法

完成本教程后，你应该能够回答：

| 问题 | 预期理解 |
|------|----------|
| RTTR_ENABLE 做了什么？ | 启用运行时类型反射，让类可被 RTTR 系统识别 |
| 组件是如何被创建的？ | 通过 RTTR 的 `create` 方法，或模板函数 `AddComponent<T>()` |
| 生命周期方法的调用顺序是什么？ | OnAwake -> OnEnable -> OnStart -> OnUpdate(循环) -> OnDisable -> OnDestroy |
| 谁调用组件的生命周期方法？ | GameObject 遍历 m_componentList 并调用每个组件的方法 |

**验证方法**：
1. 在你的组件中添加日志输出，观察调用顺序
2. 在 `GameObject::OnUpdate` 打断点，观察调用栈
3. 尝试在 `OnAwake` 中 `SetActive(false)`，观察行为

---

# 教程 2：Transform 系统原理 - 层级管理

## 概述

Transform 是每个 GameObject 必有的组件，管理位置、旋转、缩放，以及父子层级关系。完成本教程后，你将理解：

- 父子层级是如何存储和管理的？
- 世界坐标和本地坐标是如何转换的？
- 矩阵更新的时机和策略是什么？

---

## 第一节：读源码 - Transform 结构

### 打开 Transform 头文件

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Transform/transform.h`

**任务 1：理解成员变量**

阅读第 96-127 行的私有成员变量：

```cpp
// local
Vector3 m_localPosition;
Quaternion m_localRotation;
Vector3 m_localScale;

Matrix m_matrix;        // 世界矩阵
Matrix m_localMatrix;   // 本地矩阵

Transform* m_parent = nullptr;
std::vector<Transform*> m_children;
```

**问题思考**：
1. 为什么需要同时存储本地坐标和世界坐标？
2. `m_matrix` 和 `m_localMatrix` 分别代表什么？
3. 为什么用 `std::vector<Transform*>` 存储子节点，而不是 `std::vector<GameObject*>`？

---

**任务 2：理解 Get/Set 方法**

阅读第 29-48 行的位置/旋转/缩放的 Get/Set 方法：

```cpp
Vector3 GetPosition()             const { return m_matrix.GetTranslation(); }
const Vector3& GetPositionLocal() const { return m_localPosition; }
void SetPosition(const Vector3& position);
void SetPositionLocal(const Vector3& position);
```

**问题思考**：
1. `GetPosition()` 和 `GetPositionLocal()` 有什么区别？
2. 为什么 `GetPosition()` 需要从矩阵中提取，而不是直接存储？
3. `SetPosition()` 为什么比 `SetPositionLocal()` 复杂？

---

## 第二节：读源码 - 矩阵更新原理

### 打开 Transform 实现文件

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Transform/transform.cpp`

**任务 3：理解 UpdateTransform**

找到第 27-47 行的 `UpdateTransform` 方法：

```cpp
void Transform::UpdateTransform()
{
    // Compute local transform
    m_localMatrix = Matrix(m_localPosition, m_localRotation, m_localScale);

    // Compute world transform
    if (m_parent)
    {
        m_matrix = m_localMatrix * m_parent->GetMatrix();
    }
    else
    {
        m_matrix = m_localMatrix;
    }

    // Update children
    for (Transform* child : m_children)
    {
        child->UpdateTransform();
    }
}
```

**问题思考**：
1. 为什么世界矩阵 = 本地矩阵 * 父矩阵？
2. 为什么要递归更新所有子节点？
3. 如果层级很深，会不会有性能问题？

---

**任务 4：理解世界坐标设置**

找到第 49-55 行的 `SetPosition`：

```cpp
void Transform::SetPosition(const Vector3& position)
{
    if (GetPosition() == position)
        return;

    SetPositionLocal(!HasParent() ? position : position * GetParent()->GetMatrix().Inverted());
}
```

**问题思考**：
1. 为什么要判断 `GetPosition() == position`？
2. `position * GetParent()->GetMatrix().Inverted()` 在做什么？
3. 为什么设置世界坐标需要修改本地坐标？

---

## 第三节：读源码 - 父子关系管理

**任务 5：理解 SetParent**

找到第 195-246 行的 `SetParent` 方法：

```cpp
void Transform::SetParent(Transform* new_parent)
{
    // Early exit if the parent is this transform (which is invalid).
    if (new_parent)
    {
        if (GetObjectId() == new_parent->GetObjectId())
            return;
    }

    // ... 处理旧父节点
    // ... 添加到新父节点
    // ... 更新变换
}
```

**问题思考**：
1. 为什么需要检查 "parent is this"？
2. 如果 new_parent 是当前节点的子节点，会发生什么？
3. `IsDescendantOf` 是如何工作的？

---

**任务 6：理解 _Internal 方法**

找到第 274-322 行的 `SetParent_Internal`、`AddChild_Internal`、`RemoveChild_Internal`：

```cpp
void Transform::AddChild_Internal(Transform* child)
{
    LC_ASSERT(child != nullptr);
    if (child->GetObjectId() == GetObjectId())
        return;

    lock_guard lock(m_childAddRemoveMutex);
    if (!(find(m_children.begin(), m_children.end(), child) != m_children.end()))
    {
        m_children.emplace_back(child);
    }
}
```

**问题思考**：
1. 为什么需要 `_Internal` 方法？直接用公开方法有什么问题？
2. 为什么 `AddChild_Internal` 需要 mutex？
3. 为什么要检查子节点是否已存在？

---

## 第四节：实践验证 - 打印矩阵验证理解

### 任务：创建 DebugTransform 组件

创建一个组件，在运行时打印 Transform 的矩阵信息，验证你对层级和矩阵更新的理解。

**要求**：
1. 在 `OnUpdate` 中打印本地矩阵和世界矩阵
2. 观察父子关系变化时的矩阵变化
3. 验证 `m_matrix = m_localMatrix * m_parent->GetMatrix()`

**预期输出示例**：

```
[DebugTransform] Cube (no parent)
  Local Matrix: [1,0,0,0], [0,1,0,0], [0,0,1,0], [0,0,0,1]
  World Matrix: [1,0,0,0], [0,1,0,0], [0,0,1,0], [0,0,0,1]

[DebugTransform] ChildCube (parent: Cube)
  Local Matrix: [1,0,0,0], [0,1,0,0], [0,0,1,0], [1,0,0,1]
  World Matrix: [1,0,0,0], [0,1,0,0], [0,0,1,0], [1,0,0,1]
```

**验证方法**：
1. 创建一个父物体和子物体
2. 移动父物体，观察子物体的世界矩阵变化
3. 移动子物体，观察本地矩阵和世界矩阵的差异

---

## 预期理解 & 验证方法

完成本教程后，你应该能够回答：

| 问题 | 预期理解 |
|------|----------|
| 为什么用矩阵表示变换？ | 矩阵可以统一表示位置、旋转、缩放，且变换组合只需矩阵乘法 |
| 世界坐标和本地坐标的关系？ | 世界坐标 = 本地坐标经过父节点链的累积变换 |
| 为什么设置世界坐标要转成本地坐标？ | 引擎内部存储的是本地坐标，渲染时才计算世界坐标 |
| 子节点更新如何触发？ | 父节点变换变化时，递归调用所有子节点的 UpdateTransform |

**验证方法**：
1. 手动计算一个简单层级的矩阵，与引擎输出对比
2. 在 `UpdateTransform` 打断点，观察调用链
3. 修改子节点的本地坐标，确认世界坐标计算正确

---

# 教程 3：物理系统原理 - TriggerZone

## 概述

LitchiEngine 使用 PhysX 作为物理引擎。完成本教程后，你将理解：

- 引擎是如何封装 PhysX 的？
- 碰撞事件是如何从 PhysX 传递到游戏代码的？
- Trigger 和 Collider 的区别是什么？

---

## 第一节：读源码 - 物理核心封装

### 打开 Physics 类

文件路径：`Engine/Source/Runtime/Function/Physics/physics.h`

**任务 1：理解 PhysX 封装层次**

阅读类的静态成员和方法：

```cpp
class Physics {
public:
    static void Init();
    static void FixedUpdate(float fixedDeltaTime);

    static PxRigidDynamic* CreateRigidDynamic(...);
    static PxRigidStatic* CreateRigidStatic(...);
    static PxShape* CreateBoxShape(...);

private:
    static PxPhysics* px_physics_;
    static PxScene* px_scene_;
};
```

**问题思考**：
1. 为什么 `Physics` 类全是静态方法？
2. `PxPhysics` 和 `PxScene` 分别代表什么？
3. 为什么需要 `Init()` 和 `FixedUpdate()`？

---

**任务 2：理解 FixedUpdate**

搜索 `Physics::FixedUpdate` 的实现，观察它如何调用 `px_scene_->simulate()`。

**问题思考**：
1. 物理模拟是如何触发的？
2. 为什么物理更新和渲染更新分开？
3. `fetchResults` 是什么意思？

---

## 第二节：读源码 - Collider 组件

### 打开 Collider 基类

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Physcis/collider.h`

**任务 3：理解 Collider 与 PhysX 的连接**

阅读第 186-254 行的受保护成员：

```cpp
PxShape* m_pxShape = nullptr;
PxMaterial* m_pxMaterial = nullptr;
bool m_isTrigger = false;
RigidActor* m_rigidActor = nullptr;
```

**问题思考**：
1. `PxShape` 是什么？它和 `RigidActor` 的关系是什么？
2. 为什么 Collider 需要持有 RigidActor 的引用？
3. `m_isTrigger` 如何影响 PhysX 的行为？

---

### 打开 Collider 实现

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Physcis/collider.cpp`

**任务 4：理解 OnAwake 流程**

找到第 25-31 行：

```cpp
void Collider::OnAwake()
{
    CreatePhysicMaterial();
    CreateShape();
    UpdateTriggerState();
    RegisterToRigidActor();
}
```

**问题思考**：
1. 为什么按这个顺序创建？
2. `RegisterToRigidActor` 做了什么？
3. 如果 GameObject 没有 RigidActor 会怎样？

---

**任务 5：理解 Trigger 状态设置**

找到第 41-49 行的 `UpdateTriggerState`：

```cpp
void Collider::UpdateTriggerState() {
    if (m_pxShape == nullptr) {
        return;
    }
    m_pxShape->setSimulationFilterData(PxFilterData(m_isTrigger ? 1 : 0, 0, 0, 0));
    m_pxShape->userData = GetGameObject();
}
```

**问题思考**：
1. `setSimulationFilterData` 是什么？
2. 为什么把 `userData` 设置为 GameObject？
3. 如何区分 Trigger 和普通 Collider？

---

## 第三节：读源码 - 碰撞事件回调

### 打开 SimulationEventCallback

文件路径：`Engine/Source/Runtime/Function/Physics/SimulationEventCallback.h`

**任务 6：理解 onContact**

阅读第 44-82 行的 `onContact` 方法：

```cpp
void onContact(const PxContactPairHeader& pairHeader, const PxContactPair* pairs, PxU32 count) override {
    while (count--) {
        const PxContactPair& current = *pairs++;

        for (int i = 0; i < 2; ++i) {
            PxShape* shape = current.shapes[i];
            bool is_trigger = shape->getSimulationFilterData().word0 & 0x1;

            if (!is_trigger) continue;

            GameObject* gameObject = static_cast<GameObject*>(shape->userData);
            GameObject* another_game_object = static_cast<GameObject*>(another_shape->userData);

            if (current.events & PxPairFlag::eNOTIFY_TOUCH_FOUND) {
                another_game_object->ForeachComponent([gameObject](Component* component) {
                    component->OnTriggerEnter(gameObject);
                });
            }
        }
    }
}
```

**问题思考**：
1. PhysX 是如何触发 `onContact` 的？
2. 为什么需要检查 `is_trigger`？
3. 事件是如何从 PhysX 传递到 Component 的 `OnTriggerEnter` 的？
4. 为什么用 `ForeachComponent` 而不是直接调用？

---

**任务 7：理解事件流程**

画出完整的事件流程图：

```
PhysX 检测到碰撞
    |
    v
SimulationEventCallback::onContact
    |
    v
检查 shape 是否为 Trigger
    |
    v
获取 GameObject (通过 userData)
    |
    v
GameObject::ForeachComponent
    |
    v
Component::OnTriggerEnter
```

---

## 第四节：实践验证 - 创建 TriggerZone

### 任务：创建一个触发区域组件

基于你对物理系统的理解，创建一个 `TriggerZone` 组件。

**要求**：
1. 自动添加所需的物理组件（BoxCollider、RigidStatic）
2. 实现 `OnTriggerEnter` 和 `OnTriggerExit`
3. 提供 Event 供其他组件订阅

**设计问题思考**：
1. 为什么选择 RigidStatic 而不是 RigidDynamic？
2. 如果用户手动修改了 BoxCollider 的 isTrigger，你的组件如何处理？
3. 如何过滤特定的 GameObject（例如只检测玩家）？

---

## 预期理解 & 验证方法

完成本教程后，你应该能够回答：

| 问题 | 预期理解 |
|------|----------|
| Physics 类的作用是什么？ | 封装 PhysX API，提供统一的物理接口 |
| Collider 如何与 PhysX 交互？ | 创建 PxShape 并附加到 PxRigidActor |
| Trigger 的工作原理？ | 设置 filterData 标记，在 onContact 中检测并发送事件 |
| userData 的作用？ | 在 PhysX 对象和引擎对象之间建立映射 |

**验证方法**：
1. 在 `onContact` 中打印日志，观察触发时机
2. 在 `Collider::OnAwake` 打断点，观察创建流程
3. 测试两个 Trigger 是否会互相触发（答案：不会，Trigger 只与非 Trigger 碰撞）

---

# 教程 4：渲染系统原理 - 自定义着色器

## 概述

渲染系统是游戏引擎最复杂的部分之一。LitchiEngine 采用 RHI（Render Hardware Interface）抽象层设计。完成本教程后，你将理解：

- RHI 层是如何抽象图形 API 的？
- 着色器是如何编译和加载的？
- DrawCall 是如何提交的？

---

## 第一节：读源码 - RHI 抽象层

### 打开 RHI_Shader 类

文件路径：`Engine/Source/Runtime/Function/Renderer/RHI/RHI_Shader.h`

**任务 1：理解着色器抽象**

阅读第 24-83 行：

```cpp
class RHI_Shader : public Object
{
public:
    void Compile(const RHI_Shader_Stage type, const std::string& file_path, bool async, ...);
    RHI_ShaderCompilationState GetCompilationState() const;

    const std::vector<RHI_Descriptor>& GetDescriptors() const;
    const std::shared_ptr<RHI_InputLayout>& GetInputLayout() const;
    void* GetRhiResource() const { return m_rhi_resource; }

private:
    void* m_rhi_resource = nullptr;  // 实际的 Vulkan/其他 API 资源
};
```

**问题思考**：
1. `m_rhi_resource` 是什么？为什么要用 `void*`？
2. `RHI_Shader_Stage` 有哪些类型？
3. 为什么 `Compile` 有 `async` 参数？

---

**任务 2：理解 Descriptor**

阅读 `RHI_Descriptor` 的定义：

```cpp
// 在 RHI_Descriptor.h 中
struct RHI_Descriptor {
    // ... 描述着色器资源（Uniform Buffer、Texture 等）
};
```

**问题思考**：
1. Descriptor 是什么？为什么需要它？
2. 它与 Vulkan 的 Descriptor Set 有什么关系？

---

## 第二节：读源码 - 渲染器

### 打开 Renderer 类

文件路径：`Engine/Source/Runtime/Function/Renderer/Rendering/Renderer.h`

**任务 3：理解渲染 Pass**

阅读第 141-154 行的 Pass 方法：

```cpp
static void Pass_ShadowMaps(RHI_CommandList* cmd_list, RendererPath* rendererPath, const bool is_transparent_pass);
static void Pass_SkyBox(RHI_CommandList* cmd_list, RendererPath* rendererPath);
static void Pass_ForwardPass(RHI_CommandList* cmd_list, RendererPath* rendererPath, const bool is_transparent_pass);
static void Pass_UIPass(RHI_CommandList* cmd_list, RendererPath* rendererPath);
```

**问题思考**：
1. 为什么渲染分成多个 Pass？
2. `RHI_CommandList` 是什么？
3. `RendererPath` 是什么？为什么需要它？

---

**任务 4：理解 ForwardPass**

搜索 `Pass_ForwardPass` 的实现，观察它如何：
1. 遍历场景中的渲染对象
2. 设置着色器和材质
3. 提交 DrawCall

**问题思考**：
1. 如何决定渲染顺序？
2. 透明物体和不透明物体为什么要分开？
3. `cmd_list->DrawIndexed()` 做了什么？

---

## 第三节：读源码 - MeshRenderer 组件

### 打开 MeshRenderer

文件路径：`Engine/Source/Runtime/Function/Framework/Component/Renderer/MeshRenderer.h`

**任务 5：理解 MeshRenderer 的职责**

阅读关键成员和方法：

```cpp
class MeshRenderer : public Component {
public:
    Material* GetMaterial() const { return m_material; }
    void SetMaterial(Material* material);

    void OnUpdate() override;

private:
    Material* m_material = nullptr;
    bool m_castShadows = true;
};
```

**问题思考**：
1. MeshRenderer 持有 Material，那 Mesh 数据在哪里？
2. `OnUpdate` 做了什么？
3. `m_castShadows` 如何影响渲染？

---

### 打开 MeshRenderer.cpp

找到 `PostResourceLoaded` 方法：

```cpp
void MeshRenderer::PostResourceLoaded()
{
    // 加载材质
    // ...
}
```

**问题思考**：
1. 材质是如何加载的？
2. 材质文件包含哪些信息？

---

## 第四节：读源码 - 着色器编译

### 找到着色器目录

文件路径：`Engine/Data/Engine/Shaders/`

观察着色器文件结构：
- `.hlsl` 文件（HLSL 源码）
- 编译后的 SPIR-V 文件

**任务 6：理解着色器编译流程**

搜索 `RHI_DirectXShaderCompiler` 或 `DXCompiler` 相关代码：

```cpp
// 在 RHI_DirectXShaderCompiler.h 或类似文件中
class RHI_DirectXShaderCompiler {
    // 将 HLSL 编译为 SPIR-V
};
```

**问题思考**：
1. 为什么要用 HLSL 而不是 GLSL？
2. 编译流程是什么？什么时候编译？
3. 如何支持多平台？

---

## 第五节：实践验证 - 修改着色器

### 任务：修改默认着色器

找到引擎的默认 PBR 着色器，尝试修改：

**要求**：
1. 添加一个简单的颜色调整（例如让所有物体偏红）
2. 观察渲染结果

**步骤提示**：
1. 找到 `Engine/Data/Engine/Shaders/` 下的 PBR 着色器
2. 修改片段着色器的输出颜色
3. 重新运行引擎，观察变化

**验证方法**：
- 所有使用该着色器的物体颜色发生变化
- 理解着色器修改如何影响渲染结果

---

### 进阶任务：创建自定义着色器

如果时间允许，尝试创建一个简单的自定义着色器：

**要求**：
1. 创建新的 `.hlsl` 文件
2. 在代码中加载并使用
3. 实现简单的视觉效果（例如纯色、渐变）

---

## 预期理解 & 验证方法

完成本教程后，你应该能够回答：

| 问题 | 预期理解 |
|------|----------|
| RHI 的作用是什么？ | 抽象不同图形 API，提供统一接口 |
| 着色器是如何加载的？ | HLSL 源码 -> DXC 编译 -> SPIR-V -> 加载到 GPU |
| DrawCall 是如何提交的？ | 通过 CommandList 记录命令，提交给 GPU 执行 |
| 渲染 Pass 是什么？ | 渲染管线的一个阶段，如阴影 Pass、前向 Pass |

**验证方法**：
1. 在 `Pass_ForwardPass` 打断点，观察渲染对象列表
2. 修改着色器，观察渲染变化
3. 使用 RenderDoc 等工具分析 DrawCall

---

# 附录：源码路径速查表

| 系统 | 文件路径 |
|------|----------|
| 组件基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/component.h` |
| GameObject | `Engine/Source/Runtime/Function/Framework/GameObject/GameObject.h` |
| Transform | `Engine/Source/Runtime/Function/Framework/Component/Transform/transform.h` |
| 物理系统 | `Engine/Source/Runtime/Function/Physics/physics.h` |
| Collider | `Engine/Source/Runtime/Function/Framework/Component/Physcis/collider.h` |
| 事件回调 | `Engine/Source/Runtime/Function/Physics/SimulationEventCallback.h` |
| RHI Shader | `Engine/Source/Runtime/Function/Renderer/RHI/RHI_Shader.h` |
| Renderer | `Engine/Source/Runtime/Function/Renderer/Rendering/Renderer.h` |
| MeshRenderer | `Engine/Source/Runtime/Function/Framework/Component/Renderer/MeshRenderer.h` |
| 着色器文件 | `Engine/Data/Engine/Shaders/` |
| 数学库 | `Engine/Source/Runtime/Core/Math/` |

---

# 学习建议

## 如何有效阅读源码

1. **从入口点开始**：找到功能的主入口，如 `Physics::Init()`、`Renderer::Tick()`
2. **追踪调用链**：使用 IDE 的 "Go to Definition" 功能
3. **画架构图**：边读边画类图、流程图
4. **写测试代码**：创建小 Demo 验证理解

## 如何提问

好的问题示例：
- "为什么 `UpdateTransform` 要递归更新子节点？"
- "`SetPosition` 为什么要先检查坐标是否相同？"
- "Trigger 和普通 Collider 在 PhysX 中有什么区别？"

不好的问题示例：
- "这段代码是什么意思？"（没有具体指出哪里不理解）
- "怎么实现旋转？"（应该先尝试阅读相关代码）

---

**文档时间**: 2026-04-11
**教学理念**: 读源码优先，理解原理，实践验证
