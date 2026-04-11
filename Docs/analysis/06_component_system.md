# 组件系统架构分析

## 1. 模块概述

组件系统位于 `Engine/Source/Runtime/Function/Framework/`，采用 GameObject-Component 架构设计。这是游戏引擎中最核心的设计模式之一，允许灵活的对象组合。

## 2. 类继承关系

### 2.1 继承层次

```
Object (RTTR 反射基类)
  └── ScriptObject (可被脚本引用)
        ├── GameObject (游戏对象容器)
        └── Component (组件基类)
              ├── Transform (变换组件)
              ├── MeshFilter (网格过滤器)
              ├── MeshRenderer (网格渲染器)
              ├── SkinnedMeshRenderer (蒙皮网格渲染器)
              ├── Camera (相机)
              ├── Light (光源)
              ├── ScriptComponent (脚本组件)
              ├── Collider (碰撞器基类)
              │     ├── BoxCollider
              │     ├── SphereCollider
              │     └── CapsuleCollider
              ├── RigidActor (刚体基类)
              │     ├── RigidDynamic
              │     └── RigidStatic
              ├── Animator (动画器)
              └── UI组件
                    ├── UICanvas
                    ├── UIImage
                    └── UIText
```

### 2.2 RTTR 反射集成

```cpp
class Component : public ScriptObject {
public:
    // ...

    RTTR_ENABLE(ScriptObject)  // 启用 RTTR 反射
};

class Transform : public Component {
public:
    // ...

    RTTR_ENABLE(Component)  // 继承链
};
```

## 3. GameObject - 游戏对象

### 3.1 核心职责

- 作为组件的容器
- 管理组件的生命周期
- 处理层级关系 (父子)
- 触发组件事件

### 3.2 类定义

```cpp
class GameObject : public ScriptObject {
public:
    // 构造/析构
    GameObject() {}
    GameObject(const std::string& name, int64_t& id, bool& isPlaying, Scene* scene);
    ~GameObject() override;

    // 名称
    void SetName(std::string name) { SetObjectName(name); }
    std::string GetName() { return GetObjectName(); }

    // 场景关联
    Scene* GetScene();
    void SetScene(Scene* scene);

    // 层级
    unsigned char GetLayer() { return m_layer; }
    void SetLayer(unsigned char layer) { m_layer = layer; }

    // 激活状态
    void SetActive(bool active);
    bool GetActive() { return m_active; }

    // 父子关系
    bool SetParent(GameObject* parent);
    bool HasParent();
    GameObject* GetParent();
    std::list<GameObject*> GetChildren();

    // 快捷访问
    Transform* GetTransform();
    MeshRenderer* GetMeshRenderer();

    // 事件
    Event<Component*> ComponentAddedEvent;
    Event<Component*> ComponentRemovedEvent;
    Event<ScriptComponent*> BehaviourAddedEvent;
    Event<ScriptComponent*> BehaviourRemovedEvent;

    // 静态事件
    static Event<GameObject*> DestroyedEvent;
    static Event<GameObject*> CreatedEvent;
    static Event<GameObject*, GameObject*> AttachEvent;
    static Event<GameObject*> DettachEvent;

    // 组件操作
    template <class T = Component> T* AddComponent();
    template <class T = Component> void AttachComponent(T* component);
    template <class T = Component> T* GetComponent() const;
    template <class T = Component> T* GetComponent(const uint64_t unmanagedId);
    std::vector<Component*>& GetComponents() { return m_componentList; }
    void ForeachComponent(std::function<void(Component*)> func);
    bool RemoveComponent(Component* component);

    // 生命周期
    void OnAwake();
    void OnEnable();
    void OnStart();
    void OnDisable();
    void OnDestroy();
    void OnUpdate();
    void OnFixedUpdate();
    void OnLateUpdate();
    void OnEditorUpdate();

    // 物理回调
    void OnCollisionEnter(Collider* other);
    void OnCollisionStay(Collider* other);
    void OnCollisionExit(Collider* other);
    void OnTriggerEnter(Collider* other);
    void OnTriggerStay(Collider* other);
    void OnTriggerExit(Collider* other);

    // 序列化
    virtual void PostResourceLoaded() override;

    // 公开成员
    int64_t m_id{0};           // 场景内唯一ID
    int64_t m_parentId{0};     // 父对象ID
    std::vector<Component*> m_componentList;

private:
    Scene* m_scene = nullptr;
    bool m_active = true;
    bool m_isPlaying{ false };
    unsigned char m_layer{0};
    bool m_destroyed = false;
    bool m_sleeping = true;
    bool m_awaked = false;
    bool m_started = false;
    bool m_wasActive = false;
};
```

### 3.3 组件添加

```cpp
template <class T>
inline T* GameObject::AddComponent() {
    // 1. 创建组件
    T* component = new T();

    // 2. 附加到 GameObject
    AttachComponent(component);

    // 3. 资源加载后回调
    component->PostResourceLoaded();

    // 4. 触发事件
    ComponentAddedEvent.Invoke(component);

    // 5. 如果在播放模式且激活，立即调用生命周期
    if (m_isPlaying && GetActive()) {
        component->OnAwake();
        component->OnEnable();
        component->OnStart();
    }

    return dynamic_cast<T*>(component);
}

template <class T>
inline void GameObject::AttachComponent(T* component) {
    // 1. 设置所属 GameObject
    component->SetGameObject(this);

    // 2. 通过反射获取类型名
    type t = type::get<T>();
    std::string component_type_name = t.get_name().to_string();
    component->SetObjectName(component_type_name);

    // 3. 添加到组件列表
    m_componentList.push_back(component);
}
```

### 3.4 组件获取

```cpp
template <class T>
inline T* GameObject::GetComponent() const {
    // 1. 获取反射类型
    type t = type::get<T>();
    std::string component_type_name = t.get_name().to_string();

    // 2. 遍历组件列表查找
    for (auto iter = m_componentList.begin(); iter != m_componentList.end(); iter++) {
        if ((*iter)->get_type().get_name() == component_type_name) {
            return dynamic_cast<T*>(*iter);
        }
    }

    // 3. 查找派生类
    auto derived_classes = t.get_derived_classes();
    for (auto derived_class : derived_classes) {
        std::string derived_class_type_name = derived_class.get_name().to_string();

        for (auto iter = m_componentList.begin(); iter != m_componentList.end(); iter++) {
            if ((*iter)->get_type().get_name() == derived_class_type_name) {
                return dynamic_cast<T*>(*iter);
            }
        }
    }

    return nullptr;
}
```

### 3.5 生命周期管理

```cpp
void GameObject::OnAwake() {
    if (m_awaked) return;
    m_awaked = true;

    for (auto& component : m_componentList) {
        component->OnAwake();
    }
}

void GameObject::OnEnable() {
    for (auto& component : m_componentList) {
        component->OnEnable();
    }
}

void GameObject::OnStart() {
    if (m_started) return;
    m_started = true;

    for (auto& component : m_componentList) {
        component->OnStart();
    }
}

void GameObject::OnUpdate() {
    for (auto& component : m_componentList) {
        component->OnUpdate();
    }
}

void GameObject::OnDisable() {
    for (auto& component : m_componentList) {
        component->OnDisable();
    }
}

void GameObject::OnDestroy() {
    for (auto& component : m_componentList) {
        component->OnDestroy();
    }
}
```

## 4. Component - 组件基类

### 4.1 类定义

```cpp
class Component : public ScriptObject {
public:
    // 构造/析构
    Component();
    ~Component() override;

    // 所属 GameObject
    void SetGameObject(GameObject* game_object) { m_gameObject = game_object; }
    GameObject* GetGameObject() const { return m_gameObject; }

    // 资源回调
    void PostResourceModify() override;
    void PostResourceLoaded() override;

    // 生命周期
    virtual void OnAwake();
    virtual void OnEnable();
    virtual void OnStart();
    virtual void OnEditorUpdate() {}
    virtual void OnUpdate();
    virtual void OnFixedUpdate();
    virtual void OnLateUpdate() {}
    virtual void OnDisable();
    virtual void OnDestroy() {}

    // 渲染回调
    virtual void OnPreRender();
    virtual void OnPostRender();

    // 触发器回调
    virtual void OnTriggerEnter(GameObject* game_object);
    virtual void OnTriggerExit(GameObject* game_object);
    virtual void OnTriggerStay(GameObject* game_object);

private:
    GameObject* m_gameObject;

    RTTR_ENABLE(ScriptObject)
};
```

### 4.2 生命周期流程

```
场景加载
    │
    ├── GameObject 创建
    │       │
    │       ├── AddComponent<Transform>()
    │       └── AddComponent<...>()
    │
    ├── PostResourceLoaded()
    │
    └── 播放模式开始
            │
            ├── OnAwake()      // 仅一次
            ├── OnEnable()
            ├── OnStart()      // 仅一次
            │
            └── 循环
                    ├── OnUpdate()
                    ├── OnFixedUpdate()
                    └── OnLateUpdate()

场景卸载
    │
    ├── OnDisable()
    └── OnDestroy()
```

## 5. Transform - 变换组件

### 5.1 核心职责

- 管理位置、旋转、缩放
- 处理层级变换
- 提供世界/局部空间转换

### 5.2 类定义

```cpp
class Transform : public Component {
public:
    Transform();
    ~Transform() override;

    //= 位置 ======================================================================
    Vector3 GetPosition()             const { return m_matrix.GetTranslation(); }
    const Vector3& GetPositionLocal() const { return m_localPosition; }
    void SetPosition(const Vector3& position);
    void SetPositionLocal(const Vector3& position);

    //= 旋转 ======================================================================
    Quaternion GetRotation()             const { return m_matrix.GetRotation(); }
    const Quaternion& GetRotationLocal() const { return m_localRotation; }
    void SetRotation(const Quaternion& rotation);
    void SetRotationLocal(const Quaternion& rotation);

    //= 缩放 ======================================================================
    Vector3 GetScale()             const { return m_matrix.GetScale(); }
    const Vector3& GetScaleLocal() const { return m_localScale; }
    void SetScale(const Vector3& scale);
    void SetScaleLocal(const Vector3& scale);

    //= 变换操作 ==================================================================
    void Translate(const Vector3& delta);
    void Rotate(const Quaternion& delta);

    //= 方向向量 ==================================================================
    Vector3 GetUp()       const;
    Vector3 GetDown()     const;
    Vector3 GetForward()  const;
    Vector3 GetBackward() const;
    Vector3 GetRight()    const;
    Vector3 GetLeft()     const;

    //= 层级管理 ==================================================================
    void SetParent(Transform* new_parent);
    Transform* GetChildByIndex(uint32_t index);
    Transform* GetChildByName(const std::string& name);
    void RemoveChild(Transform* child);
    void AddChild(Transform* child);
    bool IsDescendantOf(Transform* transform) const;
    void GetDescendants(std::vector<Transform*>* descendants);

    bool IsRoot()                          const { return m_parent == nullptr; }
    bool HasParent()                       const { return m_parent != nullptr; }
    bool HasChildren()                     const { return GetChildrenCount() > 0; }
    uint32_t GetChildrenCount()            const { return static_cast<uint32_t>(m_children.size()); }
    Transform* GetRoot() { return HasParent() ? GetParent()->GetRoot() : this; }
    Transform* GetParent()                 const { return m_parent; }
    std::vector<Transform*>& GetChildren() { return m_children; }

    //= 矩阵 ======================================================================
    const Matrix& GetMatrix()                    const { return m_matrix; }
    const Matrix& GetLocalMatrix()               const { return m_localMatrix; }
    const Matrix& GetMatrixPrevious()            const { return m_previousMatrix; }

private:
    void UpdateTransform();
    Matrix GetParentTransformMatrix() const;

    // 局部空间
    Vector3 m_localPosition;
    Quaternion m_localRotation;
    Vector3 m_localScale;

    // 世界空间
    Matrix m_matrix;        // 世界矩阵
    Matrix m_localMatrix;   // 局部矩阵

    // 层级
    Transform* m_parent = nullptr;
    std::vector<Transform*> m_children;

    // 脏标记
    bool m_is_dirty = false;
    Matrix m_previousMatrix;

    // 变化检测
    bool m_isPositionChangedThisFrame = false;
    bool m_isRotationChangedThisFrame = false;
    bool m_isScaleChangedThisFrame = false;

    RTTR_ENABLE(Component)
};
```

### 5.3 变换更新

```cpp
void Transform::UpdateTransform() {
    // 1. 计算局部矩阵
    m_localMatrix = Matrix::CreateScale(m_localScale) *
                    Matrix::CreateFromQuaternion(m_localRotation) *
                    Matrix::CreateTranslation(m_localPosition);

    // 2. 计算世界矩阵
    if (m_parent) {
        m_matrix = m_localMatrix * m_parent->GetMatrix();
    } else {
        m_matrix = m_localMatrix;
    }

    // 3. 更新子节点
    for (auto* child : m_children) {
        child->UpdateTransform();
    }
}
```

## 6. MeshRenderer - 网格渲染器

### 6.1 类定义

```cpp
enum class MeshRendererType {
    MeshRenderer,
    SkinMeshRenderer
};

class MeshRenderer : public Component {
public:
    MeshRenderer();
    ~MeshRenderer() override;

    // 材质
    void SetMaterialPath(std::string materialPath) { m_materialPath = materialPath; }
    std::string GetMaterialPath() { return m_materialPath; }
    Material* SetMaterial(Material* material);
    Material* SetMaterial(const std::string& filePath);
    virtual void SetDefaultMaterial();
    Material* GetMaterial() const { return m_material; }
    std::string GetMaterialName() const;
    auto HasMaterial() const { return m_material != nullptr; }

    // 类型
    virtual MeshRendererType GetMeshRendererType() const {
        return MeshRendererType::MeshRenderer;
    }

    // 阴影
    void SetCastShadows(const bool castShadows) { m_castShadows = castShadows; }
    auto GetCastShadows() const { return m_castShadows; }

    // 回调
    void PostResourceModify() override;
    void PostResourceLoaded() override;
    void OnUpdate() override;
    void OnEditorUpdate() override;

protected:
    std::string m_materialPath;
    bool m_useDefaultMaterial = false;
    Material* m_material = nullptr;
    bool m_castShadows = true;

    RTTR_ENABLE(Component)
};
```

## 7. Camera - 相机组件

### 7.1 类定义

```cpp
enum class ClearFlags {
    SolidColor,
    SkyBox,
    DontClear,
    DepthOnly
};

struct CameraLightDesc {
    float m_aperture = 2.8f;
    float m_shutter_speed = 1.0f / 60.0f;
    float m_iso = 500.0f;
};

struct CameraViewport {
    Vector2 m_pos;
    Vector2 m_size;
};

class Camera : public Component {
public:
    Camera();
    ~Camera();

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;

    // 矩阵
    const Matrix& GetViewMatrix()           const;
    const Matrix& GetProjectionMatrix()     const;
    const Matrix& GetViewProjectionMatrix() const;

    // 射线拾取
    const Ray ComputePickingRay();
    void Pick();

    // 坐标转换
    Vector2 WorldToScreenCoordinates(const Vector3& position_world) const;
    Rectangle WorldToScreenCoordinates(const BoundingBox& bounding_box) const;
    Vector3 ScreenToWorldCoordinates(const Vector2& position_screen, const float z) const;

    // 清除
    ClearFlags GetClearFlags();
    void SetClearFlags(ClearFlags clearFlags);
    const Color& GetClearColor() const;
    void SetClearColor(const Color& color);

    // 曝光
    CameraLightDesc GetCameraLightDesc();
    float GetEv100() const;
    float GetExposure() const;

    // 投影
    void SetProjectionType(ProjectionType projection);
    void SetNearPlane(float near_plane);
    void SetFarPlane(float far_plane);
    void SetFovHorizontal(float fov);

    // 视锥剔除
    bool IsInViewFrustum(MeshFilter* renderable) const;
    bool IsInViewFrustum(const Vector3& center, const Vector3& extents) const;

    // FPS 控制
    bool GetFirstPersonControlEnabled() const;
    void SetFirstPersonControlEnabled(const bool enabled);

    RenderCamera* GetRenderCamera();

private:
    RenderCamera* m_renderCamera;

    ClearFlags m_clearFlags = ClearFlags::SolidColor;
    Color m_clearColor;
    ProjectionType m_projection_type = Projection_Perspective;
    float m_near_plane = 0.1f;
    float m_far_plane = 4000.0f;
    float m_fov_horizontal = 90.0f;
    CameraLightDesc m_cameraLightDesc;
    CameraViewport m_cameraViewport;
    unsigned char m_depth;
    unsigned char m_culling_mask;

    RTTR_ENABLE(Component)
};
```

## 8. 组件类型总览

### 8.1 内置组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| `Transform` | `Component/Transform/` | 位置/旋转/缩放，层级管理 |
| `MeshFilter` | `Component/Renderer/` | 网格数据引用 |
| `MeshRenderer` | `Component/Renderer/` | 网格渲染，材质管理 |
| `SkinnedMeshRenderer` | `Component/Renderer/` | 蒙皮网格渲染 |
| `Camera` | `Component/Camera/` | 相机视图，投影矩阵 |
| `Light` | `Component/Light/` | 光源 (方向光/点光源/聚光灯) |
| `ScriptComponent` | `Component/Script/` | C# 脚本绑定 |
| `Animator` | `Component/Animation/` | 骨骼动画 |
| `BoxCollider` | `Component/Physics/` | 盒形碰撞器 |
| `SphereCollider` | `Component/Physics/` | 球形碰撞器 |
| `CapsuleCollider` | `Component/Physics/` | 胶囊碰撞器 |
| `RigidDynamic` | `Component/Physics/` | 动态刚体 |
| `RigidStatic` | `Component/Physics/` | 静态刚体 |
| `UICanvas` | `Component/UI/` | UI 画布 |
| `UIImage` | `Component/UI/` | UI 图片 |
| `UIText` | `Component/UI/` | UI 文本 |

### 8.2 组件分类

```
核心组件
├── Transform (必需)

渲染组件
├── MeshFilter
├── MeshRenderer
├── SkinnedMeshRenderer
├── Camera
└── Light

物理组件
├── Collider (基类)
│     ├── BoxCollider
│     ├── SphereCollider
│     └── CapsuleCollider
└── RigidActor (基类)
      ├── RigidDynamic
      └── RigidStatic

脚本组件
└── ScriptComponent

动画组件
└── Animator

UI 组件
├── UICanvas
├── UIImage
└── UIText
```

## 9. 事件系统

### 9.1 GameObject 事件

```cpp
// 实例事件
Event<Component*> ComponentAddedEvent;
Event<Component*> ComponentRemovedEvent;
Event<ScriptComponent*> BehaviourAddedEvent;
Event<ScriptComponent*> BehaviourRemovedEvent;

// 静态事件
static Event<GameObject*> DestroyedEvent;
static Event<GameObject*> CreatedEvent;
static Event<GameObject*, GameObject*> AttachEvent;
static Event<GameObject*> DettachEvent;
```

### 9.2 事件使用

```cpp
// 订阅组件添加事件
gameObject->ComponentAddedEvent += [](Component* component) {
    DEBUG_LOG_INFO("Component added: {}", component->GetObjectName());
};

// 订阅全局创建事件
GameObject::CreatedEvent += [](GameObject* go) {
    DEBUG_LOG_INFO("GameObject created: {}", go->GetName());
};
```

## 10. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 组合模式 | GameObject-Component | 灵活组合功能 |
| 模板方法模式 | Component 生命周期 | 统一流程，子类扩展 |
| 观察者模式 | Event 系统 | 解耦事件通知 |
| 工厂模式 | AddComponent | 创建组件实例 |
| 访问者模式 | ForeachComponent | 遍历组件执行操作 |

## 11. 扩展新组件

### 11.1 步骤

1. 继承 `Component` 或其子类
2. 实现 RTTR 反射注册
3. 重写生命周期方法
4. 实现组件功能

### 11.2 示例

```cpp
// MyComponent.h
#pragma once

#include "Runtime/Function/Framework/Component/Base/component.h"

namespace LitchiRuntime {
    class MyComponent : public Component {
    public:
        MyComponent();
        ~MyComponent() override;

        void OnAwake() override;
        void OnStart() override;
        void OnUpdate() override;

        // 自定义属性
        float GetSpeed() const { return m_speed; }
        void SetSpeed(float speed) { m_speed = speed; }

    private:
        float m_speed = 1.0f;

        RTTR_ENABLE(Component)
    };
}

// MyComponent.cpp
#include "MyComponent.h"

RTTR_REGISTRATION {
    rttr::registration::class_<LitchiRuntime::MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &LitchiRuntime::MyComponent::GetSpeed, &LitchiRuntime::MyComponent::SetSpeed);
}

namespace LitchiRuntime {
    MyComponent::MyComponent() {}
    MyComponent::~MyComponent() {}

    void MyComponent::OnAwake() {
        DEBUG_LOG_INFO("MyComponent OnAwake");
    }

    void MyComponent::OnStart() {
        DEBUG_LOG_INFO("MyComponent OnStart");
    }

    void MyComponent::OnUpdate() {
        // 每帧逻辑
    }
}
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
