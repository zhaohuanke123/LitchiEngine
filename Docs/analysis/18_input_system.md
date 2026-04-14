# 输入系统分析

## 概述

LitchiEngine 的输入系统负责处理键盘和鼠标输入，采用事件驱动架构，通过 GLFW 窗口系统接收底层输入事件，并通过 InputManager 类提供统一的输入查询接口。

## 核心类设计

### 1. InputManager 类

`InputManager` 是输入系统的核心管理类，采用静态类设计，提供全局访问点。

**文件位置**: `Engine/Source/Runtime/Core/Window/Inputs/InputManager.h/cpp`

```cpp
class InputManager
{
public:
    // 生命周期管理
    static void Initialize(Window* p_window);
    static void UnInitialize();
    static void Tick();
    static void ClearEvents();

    // 键盘输入查询
    static EKeyState GetKeyState(EKey p_key);
    static bool IsKeyPressed(EKey p_key);
    static bool IsKeyReleased(EKey p_key);

    // 鼠标输入查询
    static EMouseButtonState GetMouseButtonState(EMouseButton p_button);
    static bool IsMouseButtonPressed(EMouseButton p_button);
    static bool IsMouseButtonReleased(EMouseButton p_button);

    // 鼠标位置和状态
    static const Vector2& GetMousePosition();
    static void SetMousePosition(const Vector2& position);
    static const Vector2& GetMouseDelta();
    static const Vector2& GetMouseWheelDelta();

    // 光标控制
    static void SetMouseCursorVisible(const bool visible);
    static bool GetMouseCursorVisible();

    // 视口相关
    static void SetMouseIsInViewport(const bool is_in_viewport);
    static bool GetMouseIsInViewport();
    static void SetEditorViewportOffset(const Vector2& offset);
    static const Vector2 GetMousePositionRelativeToWindow();
    static const Vector2 GetMousePositionRelativeToEditorViewport();

private:
    // 事件回调
    static void OnKeyPressed(int p_key);
    static void OnKeyReleased(int p_key);
    static void OnMouseButtonPressed(int p_button);
    static void OnMouseButtonReleased(int p_button);
    static void OnScrollMove(int16_t x, int16_t y);

private:
    static Window* m_window;

    // 事件监听器 ID
    static ListenerID m_keyPressedListener;
    static ListenerID m_keyReleasedListener;
    static ListenerID m_mouseButtonPressedListener;
    static ListenerID m_mouseButtonReleasedListener;
    static ListenerID m_scrollMoveListener;

    // 事件存储
    static std::unordered_map<EKey, EKeyState> m_keyEvents;
    static std::unordered_map<EMouseButton, EMouseButtonState> m_mouseButtonEvents;

    // 鼠标状态
    static Vector2 m_mousePosition;
    static Vector2 m_mouseDelta;
    static Vector2 m_mouseWheelDelta;
    static Vector2 m_editorViewportOffset;
    static bool m_mouseIsInViewport;
};
```

### 2. 枚举定义

#### EKey - 键盘按键枚举

**文件位置**: `Engine/Source/Runtime/Core/Window/Inputs/EKey.h`

```cpp
enum class EKey
{
    KEY_UNKNOWN = -1,
    KEY_SPACE = 32,
    KEY_APOSTROPHE = 39,
    // ... 字母键
    KEY_A = 65, KEY_B = 66, /* ... */ KEY_Z = 90,
    // ... 数字键
    KEY_0 = 48, /* ... */ KEY_9 = 57,
    // ... 功能键
    KEY_ESCAPE = 256,
    KEY_ENTER = 257,
    KEY_TAB = 258,
    KEY_BACKSPACE = 259,
    // ... 方向键
    KEY_RIGHT = 262, KEY_LEFT = 263, KEY_DOWN = 264, KEY_UP = 265,
    // ... F 键
    KEY_F1 = 290, /* ... */ KEY_F25 = 314,
    // ... 修饰键
    KEY_LEFT_SHIFT = 340, KEY_LEFT_CONTROL = 341, KEY_LEFT_ALT = 342,
    KEY_RIGHT_SHIFT = 344, KEY_RIGHT_CONTROL = 345, KEY_RIGHT_ALT = 346,
    // ... 小键盘
    KEY_KP_0 = 320, /* ... */ KEY_KP_EQUAL = 336,
    KEY_MENU = 348
};
```

按键枚举值与 GLFW 键码一致，便于直接转换。

#### EKeyState - 按键状态枚举

**文件位置**: `Engine/Source/Runtime/Core/Window/Inputs/EKeyState.h`

```cpp
enum class EKeyState
{
    KEY_UP = 0,
    KEY_DOWN = 1
};
```

#### EMouseButton - 鼠标按钮枚举

**文件位置**: `Engine/Source/Runtime/Core/Window/Inputs/EMouseButton.h`

```cpp
enum class EMouseButton
{
    MOUSE_BUTTON_1 = 0,
    MOUSE_BUTTON_2 = 1,
    MOUSE_BUTTON_3 = 2,
    MOUSE_BUTTON_4 = 3,
    // ...
    MOUSE_BUTTON_LEFT = 0,
    MOUSE_BUTTON_RIGHT = 1,
    MOUSE_BUTTON_MIDDLE = 2
};
```

#### EMouseButtonState - 鼠标按钮状态枚举

**文件位置**: `Engine/Source/Runtime/Core/Window/Inputs/EMouseButtonState.h`

```cpp
enum class EMouseButtonState
{
    MOUSE_UP = 0,
    MOUSE_DOWN = 1
};
```

## 初始化流程

### 1. InputManager 初始化

```cpp
void InputManager::Initialize(Window* p_window)
{
    m_window = p_window;

    // 订阅 Window 的事件
    m_keyPressedListener = m_window->KeyPressedEvent.AddListener(InputManager::OnKeyPressed);
    m_keyReleasedListener = m_window->KeyReleasedEvent.AddListener(InputManager::OnKeyReleased);
    m_mouseButtonPressedListener = m_window->MouseButtonPressedEvent.AddListener(InputManager::OnMouseButtonPressed);
    m_mouseButtonReleasedListener = m_window->MouseButtonReleasedEvent.AddListener(InputManager::OnMouseButtonReleased);
    m_scrollMoveListener = m_window->ScrollMoveEvent.AddListener(InputManager::OnScrollMove);
}
```

初始化时，InputManager 订阅 Window 的各种输入事件。

### 2. Window 事件绑定

Window 类在构造时绑定 GLFW 回调：

```cpp
void Window::BindKeyCallback() const
{
    auto keyCallback = [](GLFWwindow* p_window, int p_key, int p_scancode, int p_action, int p_mods)
    {
        Window* windowInstance = FindInstance(p_window);
        if (windowInstance)
        {
            if (p_action == GLFW_PRESS)
                windowInstance->KeyPressedEvent.Invoke(p_key);
            if (p_action == GLFW_RELEASE)
                windowInstance->KeyReleasedEvent.Invoke(p_key);
        }
    };
    glfwSetKeyCallback(m_glfwWindow, keyCallback);
}

void Window::BindMouseCallback() const
{
    auto mouseCallback = [](GLFWwindow* p_window, int p_button, int p_action, int p_mods)
    {
        Window* windowInstance = FindInstance(p_window);
        if (windowInstance)
        {
            if (p_action == GLFW_PRESS)
                windowInstance->MouseButtonPressedEvent.Invoke(p_button);
            if (p_action == GLFW_RELEASE)
                windowInstance->MouseButtonReleasedEvent.Invoke(p_button);
        }
    };
    glfwSetMouseButtonCallback(m_glfwWindow, mouseCallback);
}
```

## 输入处理机制

### 事件驱动 vs 轮询

输入系统支持两种查询方式：

#### 1. 事件驱动查询（帧内事件）

用于检测"本帧内发生的事件"：

```cpp
bool InputManager::IsKeyPressed(EKey p_key)
{
    return m_keyEvents.find(p_key) != m_keyEvents.end() &&
           m_keyEvents.at(p_key) == EKeyState::KEY_DOWN;
}

bool InputManager::IsKeyReleased(EKey p_key)
{
    return m_keyEvents.find(p_key) != m_keyEvents.end() &&
           m_keyEvents.at(p_key) == EKeyState::KEY_UP;
}
```

事件在帧结束时被清除：

```cpp
void InputManager::ClearEvents()
{
    m_mouseWheelDelta = Vector2::Zero;
    m_keyEvents.clear();
    m_mouseButtonEvents.clear();
}
```

#### 2. 状态轮询查询（当前状态）

用于检测"当前的持续状态"：

```cpp
EKeyState InputManager::GetKeyState(EKey p_key)
{
    switch (glfwGetKey(m_window->GetGlfwWindow(), static_cast<int>(p_key)))
    {
    case GLFW_PRESS:   return EKeyState::KEY_DOWN;
    case GLFW_RELEASE: return EKeyState::KEY_UP;
    }
    return EKeyState::KEY_UP;
}
```

### 两种查询方式的区别

| 查询方式 | 方法 | 特点 | 使用场景 |
|----------|------|------|----------|
| 事件驱动 | `IsKeyPressed` / `IsKeyReleased` | 只在事件发生的当帧返回 true | 跳跃、射击等一次性动作 |
| 状态轮询 | `GetKeyState` | 返回按键的当前物理状态 | 移动、持续按住等 |

**示例**：

```cpp
// 一次性动作：跳跃（只在按下瞬间触发一次）
if (InputManager::IsKeyPressed(EKey::KEY_SPACE))
{
    Jump();
}

// 持续动作：移动（按住时持续移动）
if (InputManager::GetKeyState(EKey::KEY_W) == EKeyState::KEY_DOWN)
{
    MoveForward();
}
```

## 鼠标输入处理

### 鼠标位置和增量

```cpp
void InputManager::Tick()
{
    double x, y;
    glfwGetCursorPos(m_window->GetGlfwWindow(), &x, &y);

    Vector2 position = Vector2(static_cast<int>(x), static_cast<int>(y));

    // 计算增量
    m_mouseDelta = position - m_mousePosition;

    // 更新位置
    m_mousePosition = position;
}
```

每帧调用 `Tick()` 更新鼠标位置和移动增量。

### 鼠标滚轮

滚轮事件通过回调处理：

```cpp
void InputManager::OnScrollMove(int16_t x, int16_t y)
{
    m_mouseWheelDelta = Vector2(x, y);
}
```

### 编辑器视口支持

为支持编辑器场景视口，提供了视口偏移计算：

```cpp
void InputManager::SetEditorViewportOffset(const Vector2& offset)
{
    m_editorViewportOffset = offset;
}

const Vector2 InputManager::GetMousePositionRelativeToEditorViewport()
{
    return GetMousePositionRelativeToWindow() - m_editorViewportOffset;
}
```

### 光标可见性控制

```cpp
void InputManager::SetMouseCursorVisible(const bool visible)
{
    if (visible == GetMouseCursorVisible())
        return;
    ApplicationBase::Instance()->window->SetCursorMode(
        visible ? ECursorMode::NORMAL : ECursorMode::DISABLED);
}
```

用于第一人称相机等需要锁定光标的场景。

## Window 类的事件系统

Window 类提供了丰富的输入事件：

```cpp
class Window
{
public:
    // 输入事件
    Event<int> KeyPressedEvent;
    Event<int> KeyReleasedEvent;
    Event<int> MouseButtonPressedEvent;
    Event<int> MouseButtonReleasedEvent;

    // 鼠标事件
    Event<int16_t, int16_t> ScrollMoveEvent;
    Event<int16_t, int16_t> CursorMoveEvent;

    // 窗口事件
    Event<uint16_t, uint16_t> ResizeEvent;
    Event<uint16_t, uint16_t> FramebufferResizeEvent;
    Event<int16_t, int16_t> MoveEvent;
    Event<> MinimizeEvent;
    Event<> MaximizeEvent;
    Event<> GainFocusEvent;
    Event<> LostFocusEvent;
    Event<> CloseEvent;

    static Event<EWindowError, std::string> ErrorEvent;
};
```

## 数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户输入                                      │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         GLFW 窗口系统                                  │
│  - glfwSetKeyCallback()                                              │
│  - glfwSetMouseButtonCallback()                                      │
│  - glfwSetScrollCallback()                                           │
│  - glfwSetCursorPosCallback()                                        │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Window 类                                     │
│  - KeyPressedEvent / KeyReleasedEvent                                │
│  - MouseButtonPressedEvent / MouseButtonReleasedEvent                │
│  - ScrollMoveEvent                                                   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         InputManager                                  │
│  - m_keyEvents (帧内键盘事件)                                         │
│  - m_mouseButtonEvents (帧内鼠标事件)                                 │
│  - m_mousePosition / m_mouseDelta                                    │
│  - m_mouseWheelDelta                                                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         游戏逻辑 / 编辑器                              │
│  - IsKeyPressed() / IsKeyReleased()                                  │
│  - GetKeyState()                                                     │
│  - GetMousePosition() / GetMouseDelta()                              │
└─────────────────────────────────────────────────────────────────────┘
```

## 主循环集成

输入系统在主循环中的典型使用：

```cpp
void ApplicationBase::Run()
{
    while (!window->ShouldClose())
    {
        // 1. 处理窗口事件
        window->PollEvents();

        // 2. 更新输入状态
        InputManager::Tick();

        // 3. 游戏逻辑处理输入
        if (InputManager::IsKeyPressed(EKey::KEY_SPACE))
        {
            // 处理按键事件
        }
        if (InputManager::GetKeyState(EKey::KEY_W) == EKeyState::KEY_DOWN)
        {
            // 处理持续按键
        }

        // 4. 渲染
        // ...

        // 5. 清除帧内事件
        InputManager::ClearEvents();

        // 6. 交换缓冲区
        window->SwapBuffers();
    }
}
```

## C# 脚本绑定

**文件位置**: `Engine/Source/ScriptCore/Source/Input.cs`

目前 C# 端的 Input 类尚未完全实现：

```csharp
namespace LitchiEngine
{
    public class Input
    {
        //public static bool IsKeyDown(KeyCode keycode)
        //{
        //    return InternalCalls.Input_IsKeyDown(keycode);
        //}
    }
}
```

要完善 C# 脚本输入支持，需要：

1. 定义 `KeyCode` 枚举（与 C++ `EKey` 对应）
2. 在 InternalCalls 中注册输入相关内部调用
3. 实现 `Input_IsKeyDown` 等 C++ 端函数

## 设计模式

### 观察者模式

InputManager 通过订阅 Window 的事件来接收输入事件，实现了松耦合：

```cpp
// 订阅事件
m_keyPressedListener = m_window->KeyPressedEvent.AddListener(InputManager::OnKeyPressed);

// 取消订阅
m_window->KeyPressedEvent.RemoveListener(m_keyPressedListener);
```

### 单例/静态类模式

InputManager 使用静态类设计，提供全局访问点，无需实例化。

### 门面模式

InputManager 为底层 GLFW 输入提供了简化的统一接口，隐藏了 GLFW 的复杂性。

## 扩展指南

### 添加新的输入类型

1. **定义枚举**: 在对应的枚举文件中添加新类型
2. **添加事件处理**: 在 InputManager 中添加对应的事件回调
3. **添加查询接口**: 提供便捷的查询方法

### 支持手柄输入

当前系统仅支持键盘和鼠标，要添加手柄支持：

1. 定义 `EGamepadButton` 和 `EGamepadAxis` 枚举
2. 在 Window 中绑定 `glfwSetJoystickCallback`
3. 在 InputManager 中添加手柄状态管理和查询接口
4. 实现 `GetGamepadButtonState`、`GetGamepadAxisValue` 等方法

### 完善 C# 脚本支持

1. **定义 KeyCode 枚举**:

```csharp
public enum KeyCode
{
    Space = 32,
    A = 65, B = 66, C = 67, /* ... */
    // 与 C++ EKey 枚举值对应
}
```

2. **添加 InternalCalls**:

```cpp
// C++ 端
bool Input_IsKeyDown(int keycode)
{
    return InputManager::GetKeyState(static_cast<EKey>(keycode)) == EKeyState::KEY_DOWN;
}

bool Input_IsKeyPressed(int keycode)
{
    return InputManager::IsKeyPressed(static_cast<EKey>(keycode));
}

// 注册
mono_add_internal_call("LitchiEngine.InternalCalls::Input_IsKeyDown", &Input_IsKeyDown);
mono_add_internal_call("LitchiEngine.InternalCalls::Input_IsKeyPressed", &Input_IsKeyPressed);
```

3. **完善 C# Input 类**:

```csharp
public class Input
{
    public static bool IsKeyDown(KeyCode keycode)
    {
        return InternalCalls.Input_IsKeyDown((int)keycode);
    }

    public static bool IsKeyPressed(KeyCode keycode)
    {
        return InternalCalls.Input_IsKeyPressed((int)keycode);
    }
}
```

## 关键文件路径

| 文件 | 路径 |
|------|------|
| InputManager 头文件 | `Engine/Source/Runtime/Core/Window/Inputs/InputManager.h` |
| InputManager 源文件 | `Engine/Source/Runtime/Core/Window/Inputs/InputManager.cpp` |
| EKey 枚举 | `Engine/Source/Runtime/Core/Window/Inputs/EKey.h` |
| EKeyState 枚举 | `Engine/Source/Runtime/Core/Window/Inputs/EKeyState.h` |
| EMouseButton 枚举 | `Engine/Source/Runtime/Core/Window/Inputs/EMouseButton.h` |
| EMouseButtonState 枚举 | `Engine/Source/Runtime/Core/Window/Inputs/EMouseButtonState.h` |
| Window 类 | `Engine/Source/Runtime/Core/Window/Window.h/cpp` |
| C# Input 类 | `Engine/Source/ScriptCore/Source/Input.cs` |

## 总结

LitchiEngine 的输入系统具有以下特点：

1. **事件驱动架构**: 通过 GLFW 回调 + 事件系统实现解耦
2. **双重查询模式**: 支持事件查询和状态轮询两种方式
3. **编辑器友好**: 提供视口偏移计算支持编辑器场景视图
4. **易于扩展**: 枚举定义清晰，接口设计简洁
5. **待完善**: C# 脚本支持尚未完全实现，手柄输入未支持
