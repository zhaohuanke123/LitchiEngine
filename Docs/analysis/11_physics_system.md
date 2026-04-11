# 物理系统分析

## 1. 模块概述

物理系统位于 `Engine/Source/Runtime/Function/Physics/`，使用 NVIDIA PhysX 物理引擎实现物理模拟。物理组件位于 `Engine/Source/Runtime/Function/Framework/Component/Physcis/`。

## 2. 架构设计

### 2.1 类关系

```
Physics (物理核心 API)
    │
    ├── PxPhysics (PhysX 实例)
    ├── PxScene (物理场景)
    ├── PxControllerManager (角色控制器管理器)
    └── PxMaterial (物理材质)

Collider (碰撞器基类)
    ├── BoxCollider (盒形碰撞器)
    ├── SphereCollider (球形碰撞器)
    └── CapsuleCollider (胶囊碰撞器)

RigidActor (刚体基类)
    ├── RigidDynamic (动态刚体)
    └── RigidStatic (静态刚体)

CharacterController (角色控制器)
```

## 3. Physics - 物理核心 API

### 3.1 核心职责

- 初始化 PhysX 运行时
- 管理物理场景
- 创建物理对象
- 执行物理模拟

### 3.2 类定义

```cpp
class Physics {
public:
    // 初始化
    static void Init();
    static void FixedUpdate(float fixedDeltaTime);

    // 场景管理
    static void CreatePxScene();

    // 材质
    static PxMaterial* CreateMaterial(float static_friction, float dynamic_friction, float restitution);

    // 角色控制器
    static void CreateControllerManager();
    static void ReleaseControllerManager();
    static void PurgeControllers();
    static PxController* CreateDefaultCapsuleController(const Vector3& position, PxMaterial* shapeMaterial, 
                                                         const Vector3& shapePosition, const Quaternion& shapeRotation, 
                                                         float shapeRadius, float shapeHalfHeight);
    static void ReleaseController(PxController* controller);
    static int32_t MoveController(PxController* controller, const Vector3& displacement, float minDist, float elapsedTime);
    static PxRigidActor* GetControllerRigidActor(PxController* controller);
    static Vector3 GetControllerPosition(PxController* controller);
    static Vector3 GetControllerFootPosition(PxController* controller);
    static void SetControllerSize(PxController* controller, float radius, float height);
    static void SetControllerSlopeLimit(PxController* controller, float value);
    static void SetControllerNonWalkableMode(PxController* controller, int32_t value);
    static void SetControllerStepOffset(PxController* controller, float value);
    static Vector3 GetControllerUpDirection(PxController* controller);
    static void SetControllerUpDirection(PxController* controller, const Vector3& value);
    static void SetControllerPosition(PxController* controller, const Vector3& value);
    static Vector3 GetGravity();

    // Actor 创建
    static PxRigidDynamic* CreateRigidDynamic(const Vector3& position, const Quaternion& rotation, const char* name);
    static PxRigidStatic* CreateRigidStatic(const Vector3& position, const Quaternion& rotation, const char* name);
    static PxRigidDynamic* CreateRigidKinematic(const Vector3& position, const Quaternion& rotation, const char* name);
    static bool ReleaseRigidActor(PxRigidActor* rigidActor);
    static void UpdateRigidActorTransform(PxRigidActor* rigidActor, const Vector3& position, const Quaternion& rotation);
    static Vector3 GetRigidActorPosition(PxRigidActor* rigidActor);
    static Quaternion GetRigidActorRotation(PxRigidActor* rigidActor);
    static void SetMass(PxRigidBody* rigidBody, float mass);

    // Actor 模拟
    static void AddForce(PxRigidBody* actor, const Vector3& force, bool autoWake = true);
    static void AddLocalForce(PxRigidBody* actor, const Vector3& force, const Vector3& position, bool autoWake = true);
    static void SetLinearDamping(PxRigidBody* actor, float damping);
    static void SetAngularDamping(PxRigidBody* actor, float damping);
    static void SetLinearVelocity(PxRigidDynamic* actor, const Vector3& velocity);
    static void SetAngularVelocitySet(PxRigidDynamic* actor, const Vector3& velocity);
    static Vector3 GetLinearVelocity(PxRigidBody* actor);
    static Vector3 GetAngularVelocity(PxRigidBody* actor);

    // Shape 创建
    static PxShape* CreateSphereShape(float radius, PxMaterial* material, const Vector3& position, const Quaternion& rotation);
    static PxShape* CreateBoxShape(const Vector3& size, PxMaterial* material, const Vector3& position, const Quaternion& rotation);
    static PxShape* CreateCapsuleShape(PxF32 r, PxF32 halfHeight, PxMaterial* material, const Vector3& position, const Quaternion& rotation);
    static bool ReleaseShape(PxShape* shape);
    static void SetShapeFlag(PxShape* shape, int flags);
    static bool AttachShape(PxRigidActor* rigidActor, PxShape* shape);
    static bool DetachShape(PxRigidActor* rigidActor, PxShape* shape);
    static void UpdateBoxShapeSize(PxShape* shape, const Vector3& size);
    static void UpdateSphereShapeSize(PxShape* shape, float radius);

    // CCD
    static bool enable_ccd() { return enable_ccd_; }
    static void set_enable_ccd(bool enable_ccd) { enable_ccd_ = enable_ccd; }

    // 场景查询
    static bool RaycastSingle(Vector3& origin, Vector3& dir, float distance, RaycastHit* raycast_hit);

private:
    static PxDefaultAllocator px_allocator_;
    static PhysicErrorCallback physic_error_callback_;
    static SimulationEventCallback simulation_event_callback_;
    static SimulationFilterCallback simulation_filter_callback_;

    static PxFoundation* px_foundation_;
    static PxPhysics* px_physics_;
    static PxDefaultCpuDispatcher* px_cpu_dispatcher_;
    static PxScene* px_scene_;
    static PxControllerManager* px_controller_manager_;
    static PxPvd* px_pvd_;

    static bool enable_ccd_;
};
```

### 3.3 初始化流程

```cpp
void Physics::Init() {
    // 1. 创建 Foundation
    px_foundation_ = PxCreateFoundation(PX_PHYSICS_VERSION, px_allocator_, physic_error_callback_);

    // 2. 创建 PVD (PhysX Visual Debugger)
    px_pvd_ = PxCreatePvd(*px_foundation_);
    PxPvdTransport* transport = PxDefaultPvdSocketTransportCreate("127.0.0.1", 5425, 10);
    px_pvd_->connect(*transport, PxPvdInstrumentationFlag::eALL);

    // 3. 创建 Physics
    px_physics_ = PxCreatePhysics(PX_PHYSICS_VERSION, *px_foundation_, PxTolerancesScale(), true, px_pvd_);

    // 4. 创建 CPU Dispatcher
    px_cpu_dispatcher_ = PxDefaultCpuDispatcherCreate(4);

    // 5. 创建场景
    CreatePxScene();

    // 6. 创建控制器管理器
    CreateControllerManager();
}

void Physics::CreatePxScene() {
    PxSceneDesc sceneDesc(px_physics_->getTolerancesScale());
    sceneDesc.gravity = PxVec3(0.0f, -9.81f, 0.0f);
    sceneDesc.cpuDispatcher = px_cpu_dispatcher_;
    sceneDesc.filterShader = PxDefaultSimulationFilterShader;
    sceneDesc.simulationEventCallback = &simulation_event_callback_;

    px_scene_ = px_physics_->createScene(sceneDesc);
}
```

### 3.4 物理模拟

```cpp
void Physics::FixedUpdate(float fixedDeltaTime) {
    if (px_scene_) {
        px_scene_->simulate(fixedDeltaTime);
        px_scene_->fetchResults(true);
    }
}
```

## 4. Collider - 碰撞器基类

### 4.1 类定义

```cpp
class Collider : public Component {
public:
    Collider();
    ~Collider() override;

    // 物理材质
    void SetPhysicMaterial(PhysicMaterialRes physicMaterialRes);
    PhysicMaterialRes GetPhysicMaterial();

    // 触发器
    void SetIsTrigger(bool isTrigger);
    bool IsTrigger();

    // PhysX Shape
    PxShape* GetPxShape();

    // 偏移
    void SetOffset(Vector3 offset);
    Vector3 GetOffset();

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;
    void OnFixedUpdate() override;
    void PostResourceModify() override;
    void PostResourceLoaded() override;

protected:
    // 子类实现
    virtual void CreateShape() = 0;
    virtual void CreatePhysicMaterial();
    virtual void UpdateTriggerState();
    virtual void RegisterToRigidActor();
    virtual void UnRegisterToRigidActor();
    void UpdateShape();
    RigidActor* GetRigidActor();

    PxShape* m_pxShape = nullptr;
    PxMaterial* m_pxMaterial = nullptr;
    bool m_isTrigger = false;
    Vector3 m_offset{0.0f};
    RigidActor* m_rigidActor = nullptr;
    PhysicMaterialRes m_physicMaterial;

    RTTR_ENABLE(Component);
};
```

### 4.2 物理材质

```cpp
class PhysicMaterialRes {
public:
    PhysicMaterialRes() {}
    PhysicMaterialRes(float staticFriction, float dynamicFriction, float restitution)
        : m_staticFriction(staticFriction), m_dynamicFriction(dynamicFriction), m_restitution(restitution) {}

    void SetStaticFriction(float staticFriction) { m_staticFriction = staticFriction; }
    float GetStaticFriction() { return m_staticFriction; }

    void SetDynamicFriction(float dynamicFriction) { m_dynamicFriction = dynamicFriction; }
    float GetDynamicFriction() { return m_dynamicFriction; }

    void SetRestitution(float restitution) { m_restitution = restitution; }
    float GetRestitution() { return m_restitution; }

private:
    float m_staticFriction = 0.6f;   // 静摩擦系数
    float m_dynamicFriction = 0.6f;  // 动摩擦系数
    float m_restitution = 0.1f;      // 弹性系数
};
```

## 5. RigidActor - 刚体基类

### 5.1 类定义

```cpp
class RigidActor : public Component {
public:
    RigidActor();
    ~RigidActor() override;

    // 碰撞器管理
    virtual void AttachColliderShape(Collider* collider);
    virtual void DetachColliderShape(Collider* collider);

    // 生命周期
    void OnAwake() override;

protected:
    PxRigidActor* m_pxRigidActor;

    RTTR_ENABLE(Component)
};
```

### 5.2 RigidDynamic - 动态刚体

```cpp
class RigidDynamic : public RigidActor {
public:
    RigidDynamic();
    ~RigidDynamic() override;

    // 质量
    void SetMass(float mass);
    float GetMass();

    // 速度
    void SetLinearVelocity(const Vector3& velocity);
    Vector3 GetLinearVelocity();
    void SetAngularVelocity(const Vector3& velocity);
    Vector3 GetAngularVelocity();

    // 力
    void AddForce(const Vector3& force, bool autoWake = true);
    void AddLocalForce(const Vector3& force, const Vector3& position, bool autoWake = true);

    // 阻尼
    void SetLinearDamping(float damping);
    void SetAngularDamping(float damping);

    // 运动学模式
    void SetKinematic(bool kinematic);
    bool IsKinematic();

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;
    void OnFixedUpdate() override;

private:
    bool m_isKinematic = false;

    RTTR_ENABLE(RigidActor)
};
```

### 5.3 RigidStatic - 静态刚体

```cpp
class RigidStatic : public RigidActor {
public:
    RigidStatic();
    ~RigidStatic() override;

    void OnAwake() override;

    RTTR_ENABLE(RigidActor)
};
```

## 6. 具体碰撞器

### 6.1 BoxCollider

```cpp
class BoxCollider : public Collider {
public:
    BoxCollider();

    void SetSize(const Vector3& size);
    Vector3 GetSize() const { return m_size; }

protected:
    void CreateShape() override;

private:
    Vector3 m_size{1.0f, 1.0f, 1.0f};

    RTTR_ENABLE(Collider)
};
```

### 6.2 SphereCollider

```cpp
class SphereCollider : public Collider {
public:
    SphereCollider();

    void SetRadius(float radius);
    float GetRadius() const { return m_radius; }

protected:
    void CreateShape() override;

private:
    float m_radius = 0.5f;

    RTTR_ENABLE(Collider)
};
```

### 6.3 CapsuleCollider

```cpp
class CapsuleCollider : public Collider {
public:
    CapsuleCollider();

    void SetRadius(float radius);
    float GetRadius() const { return m_radius; }
    void SetHeight(float height);
    float GetHeight() const { return m_height; }

protected:
    void CreateShape() override;

private:
    float m_radius = 0.5f;
    float m_height = 2.0f;

    RTTR_ENABLE(Collider)
};
```

## 7. CharacterController - 角色控制器

### 7.1 类定义

```cpp
class CharacterController : public Component {
public:
    CharacterController();
    ~CharacterController() override;

    // 移动
    void Move(const Vector3& displacement);
    Vector3 GetPosition();
    void SetPosition(const Vector3& position);

    // 尺寸
    void SetSize(float radius, float height);

    // 参数
    void SetSlopeLimit(float value);
    void SetStepOffset(float value);

    // 生命周期
    void OnAwake() override;
    void OnFixedUpdate() override;

private:
    PxController* m_pxController = nullptr;
    float m_radius = 0.5f;
    float m_height = 2.0f;

    RTTR_ENABLE(Component)
};
```

## 8. 射线检测

### 8.1 RaycastHit

```cpp
struct RaycastHit {
    bool hit = false;
    Vector3 position;
    Vector3 normal;
    float distance;
    GameObject* gameObject = nullptr;
};
```

### 8.2 RaycastSingle

```cpp
bool Physics::RaycastSingle(Vector3& origin, Vector3& dir, float distance, RaycastHit* raycast_hit) {
    PxRaycastBuffer hitInfo;

    bool result = px_scene_->raycast(
        PxVec3(origin.x, origin.y, origin.z),
        PxVec3(dir.x, dir.y, dir.z),
        distance,
        hitInfo
    );

    if (result && hitInfo.hasBlock) {
        raycast_hit->hit = true;
        raycast_hit->position = Vector3(hitInfo.block.position.x, hitInfo.block.position.y, hitInfo.block.position.z);
        raycast_hit->normal = Vector3(hitInfo.block.normal.x, hitInfo.block.normal.y, hitInfo.block.normal.z);
        raycast_hit->distance = hitInfo.block.distance;

        // 获取碰撞的 GameObject
        PxRigidActor* actor = hitInfo.block.actor;
        // ...
    }

    return result;
}
```

## 9. 碰撞事件

### 9.1 SimulationEventCallback

```cpp
class SimulationEventCallback : public PxSimulationEventCallback {
public:
    // 触发器事件
    virtual void onTrigger(PxTriggerPair* pairs, PxU32 count) override;

    // 接触事件
    virtual void onContact(const PxContactPairHeader& pairHeader, const PxContactPair* pairs, PxU32 nbPairs) override;

    // 约束打破事件
    virtual void onConstraintBreak(PxConstraintInfo* constraints, PxU32 count) override;

    // 休眠事件
    virtual void onSleep(PxActor** actors, PxU32 count) override;

    // 唤醒事件
    virtual void onWake(PxActor** actors, PxU32 count) override;

    // 高级接触事件
    virtual void onAdvance(const PxRigidBody*const* bodyBuffer, const PxTransform* poseBuffer, const PxU32 count) override;
};
```

## 10. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 外观模式 | Physics | 封装 PhysX API |
| 模板方法模式 | Collider | 统一碰撞器流程 |
| 组合模式 | RigidActor + Collider | 刚体和碰撞器组合 |
| 工厂方法模式 | CreateShape | 子类创建 Shape |

## 11. 使用示例

### 11.1 创建动态刚体

```cpp
// 添加组件
auto* rigidDynamic = gameObject->AddComponent<RigidDynamic>();
auto* boxCollider = gameObject->AddComponent<BoxCollider>();

// 设置质量
rigidDynamic->SetMass(10.0f);

// 设置碰撞器尺寸
boxCollider->SetSize(Vector3(1.0f, 1.0f, 1.0f));

// 施加力
rigidDynamic->AddForce(Vector3(0, 100, 0));
```

### 11.2 创建角色控制器

```cpp
auto* controller = gameObject->AddComponent<CharacterController>();

// 设置尺寸
controller->SetSize(0.5f, 2.0f);

// 移动
controller->Move(Vector3(0, 0, 1) * speed * deltaTime);
```

### 11.3 射线检测

```cpp
RaycastHit hit;
if (Physics::RaycastSingle(origin, direction, 100.0f, &hit)) {
    if (hit.hit) {
        DEBUG_LOG_INFO("Hit: {}", hit.gameObject->GetName());
    }
}
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
