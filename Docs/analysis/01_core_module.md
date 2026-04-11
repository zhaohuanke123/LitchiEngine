# Core 模块架构分析

## 1. 模块概述

Core 模块是 LitchiEngine 的基础设施层，位于 `Engine/Source/Runtime/Core/`，提供引擎运行所需的核心功能：

- 数学库 (Math)
- 反射系统 (Meta/Reflection)
- 事件系统 (Tools/Eventing)
- 窗口管理 (Window)
- 应用程序框架 (App)
- 服务定位器 (Global)
- 日志系统 (Log)
- 时间管理 (Time)
- 文件系统工具 (Tools/FileSystem)

## 2. 目录结构

```
Engine/Source/Runtime/Core/
├── App/                    # 应用程序框架
│   ├── ApplicationBase.h   # 应用基类
│   └── application.cpp
├── Global/                 # 全局服务
│   └── ServiceLocator.h    # 服务定位器
├── Log/                    # 日志系统
│   └── debug.h/cpp
├── Math/                   # 数学库
│   ├── Vector2/3/4.h/cpp   # 向量
│   ├── Matrix.h/cpp        # 矩阵
│   ├── Quaternion.h/cpp    # 四元数
│   ├── BoundingBox.h/cpp   # 包围盒
│   ├── Frustum.h/cpp       # 视锥体
│   ├── Plane.h/cpp         # 平面
│   ├── Ray.h/cpp           # 射线
│   └── MathHelper.h        # 数学辅助函数
├── Meta/                   # 元数据系统
│   ├── Reflection/         # 反射
│   │   ├── object.h        # 基础对象
│   │   └── type.h          # 类型管理
│   └── Serializer/         # 序列化
│       └── serializer.h
├── Screen/                 # 屏幕管理
├── Time/                   # 时间管理
├── Tools/                  # 工具类
│   ├── Eventing/           # 事件系统
│   ├── FileSystem/         # 文件系统
│   └── Utils/              # 工具函数
└── Window/                 # 窗口管理
    ├── Window.h/cpp        # 窗口类
    ├── Inputs/             # 输入管理
    ├── Cursor/             # 光标
    ├── Dialogs/            # 对话框
    └── Settings/           # 设置
```

## 3. 数学库 (Math)

### 3.1 Vector3 类

**文件**: `Math/Vector3.h`

```cpp
class LC_CLASS Vector3 {
public:
    float x, y, z;

    // 构造函数
    Vector3();
    Vector3(float x, float y, float z);
    Vector3(float f);  // 统一值

    // 核心方法
    void Normalize();
    Vector3 Normalized() const;
    float Length() const;
    float LengthSquared() const;

    // 静态方法
    static float Dot(const Vector3& v1, const Vector3& v2);
    static Vector3 Cross(const Vector3& v1, const Vector3& v2);
    static float Distance(const Vector3& a, const Vector3& b);
    static Vector3 Lerp(const Vector3& a, const Vector3& b, float t);

    // 运算符重载
    Vector3 operator+(const Vector3& b) const;
    Vector3 operator-(const Vector3& b) const;
    Vector3 operator*(float value) const;
    // ...

    // 预定义常量
    static const Vector3 Zero, Left, Right, Up, Down, Forward, Backward, One;
};
```

**设计特点**:
- 值类型设计，支持拷贝
- 提供静态常量便于使用
- 内联实现优化性能
- 使用 `[[nodiscard]]` 属性防止忽略返回值

### 3.2 Matrix 类

**文件**: `Math/Matrix.h`

```cpp
class LC_CLASS Matrix {
public:
    // Column-major 内存布局 (兼容 DirectX/GPU)
    float m00, m10, m20, m30;
    float m01, m11, m21, m31;
    float m02, m12, m22, m32;
    float m03, m13, m23, m33;

    // 工厂方法
    static Matrix CreateTranslation(const Vector3& translation);
    static Matrix CreateRotation(const Quaternion& rotation);
    static Matrix CreateScale(const Vector3& scale);
    static Matrix CreateLookAtLH(const Vector3& camera_position, const Vector3& target, const Vector3& up);
    static Matrix CreatePerspectiveFieldOfViewLH(float fieldOfView, float aspectRatio, float near_plane, float far_plane);
    static Matrix CreateOrthographicLH(float width, float height, float zNearPlane, float zFarPlane);

    // 变换方法
    Vector3 GetTranslation() const;
    Vector3 GetScale() const;
    Quaternion GetRotation() const;
    void Decompose(Vector3& scale, Quaternion& rotation, Vector3& translation) const;

    // 矩阵运算
    Matrix Transposed() const;
    Matrix Inverted() const;
    Matrix operator*(const Matrix& rhs) const;
    Vector3 operator*(const Vector3& rhs) const;
};
```

**设计特点**:
- Column-major 内存布局，直接映射到 GPU
- 支持 TRS (Translation-Rotation-Scale) 分解
- 提供完整的视图/投影矩阵生成

### 3.3 Quaternion 类

**文件**: `Math/Quaternion.h`

```cpp
class LC_CLASS Quaternion {
public:
    float x, y, z, w;

    // 工厂方法
    static Quaternion FromAngleAxis(float angle, const Vector3& axis);
    static Quaternion FromYawPitchRoll(float yaw, float pitch, float roll);
    static Quaternion FromEulerAngles(const Vector3& rotation);
    static Quaternion FromLookRotation(const Vector3& direction, const Vector3& up = Vector3::Up);
    static Quaternion Lerp(const Quaternion& a, const Quaternion& b, float t);

    // 核心方法
    void Normalize();
    Quaternion Normalized() const;
    Quaternion Inverse() const;
    Vector3 ToEulerAngles() const;

    // 运算符
    Quaternion operator*(const Quaternion& rhs) const;
    Vector3 operator*(const Vector3& rhs) const;  // 旋转向量

    static const Quaternion Identity;
};
```

### 3.4 BoundingBox 类

**文件**: `Math/BoundingBox.h`

```cpp
class LC_CLASS BoundingBox {
public:
    BoundingBox();
    BoundingBox(const Vector3& min, const Vector3& max);
    BoundingBox(const Vector3* vertices, const uint32_t point_count);

    Vector3 GetCenter() const { return (m_max + m_min) * 0.5f; }
    Vector3 GetSize() const { return m_max - m_min; }
    Vector3 GetExtents() const { return (m_max - m_min) * 0.5f; }

    Intersection IsInside(const Vector3& point) const;
    Intersection IsInside(const BoundingBox& box) const;
    BoundingBox Transform(const Matrix& transform) const;
    void Merge(const BoundingBox& box);

private:
    Vector3 m_min, m_max;
};
```

### 3.5 Frustum 类

**文件**: `Math/Frustum.h`

```cpp
class Frustum {
public:
    Frustum(const Matrix& mView, const Matrix& mProjection, float screenDepth);
    bool IsVisible(const Vector3& center, const Vector3& extent, bool ignore_near_plane = false) const;

private:
    Plane m_planes[6];  // 6 个裁剪平面
    Intersection CheckCube(const Vector3& center, const Vector3& extent) const;
    Intersection CheckSphere(const Vector3& center, float radius) const;
};
```

### 3.6 MathHelper

**文件**: `Math/MathHelper.h`

```cpp
namespace LitchiRuntime::Math::Helper {
    // 常量
    constexpr float EPSILON = std::numeric_limits<float>::epsilon();
    constexpr float PI = 3.14159265359f;
    constexpr float DEG_TO_RAD = PI / 180.0f;
    constexpr float RAD_TO_DEG = 180.0f / PI;

    // 模板函数
    template <typename T> constexpr T Clamp(T x, T a, T b);
    template <typename T> constexpr T Lerp(T lhs, T rhs, U t);
    template <class T> constexpr T Abs(T value);
    template <class T> constexpr bool Equals(T lhs, T rhs, T error = EPSILON);
    template <class T> constexpr T Max(T a, T b);
    template <class T> constexpr T Min(T a, T b);
    template <class T> constexpr T Sqrt(T x);
}
```

## 4. 反射系统 (Meta/Reflection)

### 4.1 Object 基类

**文件**: `Meta/Reflection/object.h`

```cpp
class Object {
public:
    Object();
    virtual ~Object() = default;

    // 名称和 ID
    std::string& GetObjectName();
    void SetObjectName(std::string& name);
    uint64_t GetObjectId() const;
    static uint64_t GenerateObjectId();

    // 资源生命周期回调
    virtual void PostResourceModify() {}
    virtual void PostResourceLoaded() {}

    RTTR_ENABLE()  // 启用 RTTR 反射

protected:
    std::string m_object_name;
    uint64_t m_object_id = 0;
    uint64_t m_object_size_cpu = 0;
    uint64_t m_object_size_gpu = 0;
};
```

**设计特点**:
- 所有引擎对象的基类
- 使用 RTTR 库实现运行时类型反射
- 提供 `PostResourceLoaded` 和 `PostResourceModify` 钩子

### 4.2 TypeManager

**文件**: `Meta/Reflection/type.h`

```cpp
class TypeManager {
public:
    template <class T>
    rttr::type GetType() {
        type t = type::get<T>();
        return t;
    }

    template <class T>
    bool IsSerializable() {
        type t = type::get<T>();
        return t.get_metadata("Serializable").to_bool();
    }
};
```

### 4.3 RTTR 使用示例

```cpp
// 类定义
class MyComponent : public Component {
public:
    float speed = 1.0f;
    bool enabled = true;

    RTTR_ENABLE(Component)
};

// 注册
RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .property("speed", &MyComponent::speed)
        .property("enabled", &MyComponent::enabled);
}
```

## 5. 事件系统 (Tools/Eventing)

### 5.1 Event 类

**文件**: `Tools/Eventing/Event.h`

```cpp
using ListenerID = uint64_t;

template<class... ArgTypes>
class Event {
public:
    using Callback = std::function<void(ArgTypes...)>;

    ListenerID AddListener(Callback p_callback);
    ListenerID operator+=(Callback p_callback);
    bool RemoveListener(ListenerID p_listenerID);
    bool operator-=(ListenerID p_listenerID);
    void RemoveAllListeners();
    uint64_t GetListenerCount();
    void Invoke(ArgTypes... p_args);

private:
    std::unordered_map<ListenerID, Callback> m_callbacks;
    ListenerID m_availableListenerID = 0;
};
```

**使用示例**:

```cpp
// 定义事件
Event<int, float> myEvent;

// 添加监听器
ListenerID id = myEvent += [](int a, float b) {
    // 处理事件
};

// 触发事件
myEvent.Invoke(42, 3.14f);

// 移除监听器
myEvent -= id;
```

## 6. 窗口管理 (Window)

### 6.1 Window 类

**文件**: `Window/Window.h`

```cpp
class Window {
public:
    Window(const WindowSettings& p_windowSettings);
    ~Window();

    // 窗口操作
    void SetSize(uint16_t p_width, uint16_t p_height);
    void SetPosition(int16_t p_x, int16_t p_y);
    void SetTitle(const std::string& p_title);
    void SetFullscreen(bool p_value);
    void Minimize() const;
    void Maximize() const;

    // 状态查询
    bool ShouldClose() const;
    bool IsFullscreen() const;
    bool IsFocused() const;
    uint32_t GetWidth();
    uint32_t GetHeight();

    // 输入相关
    void SetCursorMode(ECursorMode p_cursorMode);
    void SetCursorShape(ECursorShape p_cursorShape);
    void PollEvents() const;

    // 事件
    Event<int> KeyPressedEvent;
    Event<int> KeyReleasedEvent;
    Event<int> MouseButtonPressedEvent;
    Event<uint16_t, uint16_t> ResizeEvent;
    Event<> CloseEvent;
    // ...

private:
    GLFWwindow* m_glfwWindow;
};
```

### 6.2 InputManager

**文件**: `Window/Inputs/InputManager.h`

```cpp
class InputManager {
public:
    static void Initialize(Window* p_window);
    static void Tick();
    static void ClearEvents();

    // 键盘
    static EKeyState GetKeyState(EKey p_key);
    static bool IsKeyPressed(EKey p_key);
    static bool IsKeyReleased(EKey p_key);

    // 鼠标
    static EMouseButtonState GetMouseButtonState(EMouseButton p_button);
    static bool IsMouseButtonPressed(EMouseButton p_button);
    static const Vector2& GetMousePosition();
    static const Vector2& GetMouseDelta();
    static const Vector2& GetMouseWheelDelta();
};
```

## 7. 应用程序框架 (App)

### 7.1 ApplicationBase

**文件**: `App/ApplicationBase.h`

```cpp
class ApplicationBase {
public:
    ApplicationBase();
    virtual ~ApplicationBase();

    // 生命周期
    virtual bool Initialize();
    virtual void Run();
    virtual void Update();
    virtual void Exit();

    // 配置
    virtual WindowSettings CreateWindowSettings() = 0;
    virtual LitchiApplicationType GetApplicationType() = 0;

    // 资源管理器
    std::unique_ptr<ModelManager> modelManager;
    std::unique_ptr<ShaderManager> shaderManager;
    std::unique_ptr<MaterialManager> materialManager;
    std::unique_ptr<TextureManager> textureManager;
    std::unique_ptr<SceneManager> sceneManager;
    std::unique_ptr<PrefabManager> prefabManager;
    std::unique_ptr<Window> window;

    // 单例访问
    static ApplicationBase* Instance();

protected:
    std::string m_engineAssetsPath;
    std::string m_projectPath;
    std::string m_title;
};
```

## 8. 服务定位器 (Global)

### 8.1 ServiceLocator

**文件**: `Global/ServiceLocator.h`

```cpp
#define OVSERVICE(Type) LitchiRuntime::ServiceLocator::Get<Type>()

class ServiceLocator {
public:
    template<typename T>
    static void Provide(T& p_service) {
        __SERVICES[typeid(T).hash_code()] = std::any(&p_service);
    }

    template<typename T>
    static T& Get() {
        return *std::any_cast<T*>(__SERVICES[typeid(T).hash_code()]);
    }

private:
    static std::unordered_map<size_t, std::any> __SERVICES;
};
```

**使用示例**:

```cpp
// 提供服务
ServiceLocator::Provide<SceneManager>(sceneManager);

// 获取服务
auto& sceneManager = OVSERVICE(SceneManager);
```

## 9. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | ApplicationBase, TypeManager | 全局访问点 |
| 服务定位器 | ServiceLocator | 解耦服务依赖 |
| 观察者模式 | Event 系统 | 解耦事件通知 |
| 值对象模式 | Vector3, Matrix, Quaternion | 数学类型设计 |
| 模板方法模式 | ApplicationBase 生命周期 | 统一流程，子类扩展 |
| 工厂方法 | Matrix::CreateTranslation 等 | 创建复杂对象 |

## 10. 代码规范

### 10.1 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `Vector3`, `BoundingBox` |
| 函数名 | PascalCase | `Normalize()`, `GetCenter()` |
| 变量名 | snake_case with m_ prefix | `m_object_name`, `m_min` |
| 常量 | UPPER_CASE | `EPSILON`, `PI` |
| 模板参数 | T 或大写 | `template<typename T>` |

### 10.2 文件组织

- 头文件使用 `#pragma once`
- 包含顺序：标准库 → 第三方库 → 项目内部
- 使用 `//= INCLUDES ==========` 注释分隔

### 10.3 注释规范

```cpp
/**
 * @brief 简短描述
 * @param p_name 参数说明
 * @return 返回值说明
 */
```

### 10.4 宏定义

- `LC_CLASS`: 用于 RTTR 反射的类标记
- `RTTR_ENABLE(BaseClass)`: 启用反射继承
- `OVSERVICE(Type)`: 服务定位器快捷访问

## 11. 扩展建议

### 11.1 添加新的数学类型

1. 在 `Math/` 目录创建头文件和实现文件
2. 遵循现有命名和设计规范
3. 添加静态常量（如 Zero, Identity）
4. 实现必要的运算符重载

### 11.2 添加新的事件类型

1. 在需要事件的地方声明 `Event<Args...>` 成员
2. 使用 `Invoke()` 触发事件
3. 外部使用 `+=` 添加监听器

### 11.3 创建新的资源管理器

1. 继承 `AResourceManager<T>` 模板
2. 实现 `CreateResource` 和 `DestroyResource` 方法
3. 在 `ApplicationBase` 中注册

---

**分析完成时间**: 2026-04-09
**分析者**: AI Agent
