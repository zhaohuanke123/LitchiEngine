# 项目代码规范总结

## 1. 概述

本文档总结了 LitchiEngine 项目的代码规范、命名约定、设计模式和最佳实践，基于对项目各模块的分析。

## 2. 命名规范

### 2.1 类名

- **PascalCase**: 所有类名使用 PascalCase（大驼峰命名法）
- **前缀**: 无特殊前缀，直接使用语义化名称
- **示例**:
  ```cpp
  class GameObject { };
  class Transform { };
  class MeshRenderer { };
  class RHI_Device { };
  ```

### 2.2 成员变量

- **m_ 前缀**: 类成员变量使用 `m_` 前缀
- **camelCase**: 前缀后使用小驼峰命名法
- **示例**:
  ```cpp
  class Component {
  private:
      GameObject* m_gameObject;
      uint64_t m_unmanagedId;
      bool m_isPlaying;
  };
  ```

### 2.3 局部变量

- **camelCase**: 局部变量使用小驼峰命名法
- **示例**:
  ```cpp
  void Update() {
      float deltaTime = Time::GetDeltaTime();
      Vector3 position = GetPosition();
  }
  ```

### 2.4 函数名

- **PascalCase**: 公共函数使用 PascalCase
- **动词开头**: 函数名以动词开头表示操作
- **示例**:
  ```cpp
  void OnAwake();
  void SetPosition(const Vector3& position);
  GameObject* CreateGameObject(const std::string& name);
  ```

### 2.5 常量

- **UPPER_CASE**: 常量使用全大写下划线分隔
- **示例**:
  ```cpp
  const uint32_t rhi_max_render_target_count = 8;
  const float PI = 3.14159f;
  ```

### 2.6 枚举

- **PascalCase**: 枚举类型和枚举值使用 PascalCase
- **示例**:
  ```cpp
  enum class RHI_Format {
      R8_Unorm,
      R16_Float,
      R32_Uint,
  };
  ```

### 2.7 命名空间

- **PascalCase**: 命名空间使用 PascalCase
- **示例**:
  ```cpp
  namespace LitchiRuntime { }
  namespace LitchiEditor { }
  ```

## 3. 代码组织结构

### 3.1 目录结构

```
Engine/Source/
├── Runtime/           # 核心运行时
│   ├── Core/          # 核心系统 (App, Math, Log, Time, Window, Screen)
│   ├── Function/      # 功能模块
│   │   ├── Renderer/  # 渲染系统
│   │   ├── Scene/     # 场景管理
│   │   ├── Physics/   # 物理系统
│   │   ├── Scripting/ # 脚本系统
│   │   ├── UI/        # UI系统
│   │   └── Framework/ # 组件框架
│   ├── Resource/      # 资源管理器
│   └── Platform/      # 平台相关代码
├── Editor/            # 编辑器
├── Standalone/        # 独立运行程序
└── ScriptCore/        # C# 脚本核心
```

### 3.2 文件命名

- **类名一致**: 文件名与主要类名一致
- **示例**:
  - `GameObject.h` / `GameObject.cpp`
  - `MeshRenderer.h` / `MeshRenderer.cpp`

### 3.3 头文件组织

```cpp
#pragma once

//= INCLUDES ==================
#include <memory>
#include <string>
#include "Runtime/Core/Math/Vector3.h"
//=============================

namespace LitchiRuntime
{
    class MyClass {
    public:
        // 公共方法

    private:
        // 私有成员

        RTTR_ENABLE()
    };
}
```

## 4. 设计模式

### 4.1 常用设计模式

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | Renderer, RHI_Device, SceneManager | 全局访问点 |
| 服务定位器 | ServiceLocator | 解耦服务依赖 |
| 组合模式 | GameObject-Component | 灵活的对象组合 |
| 观察者模式 | Event 系统 | 解耦事件通知 |
| 模板方法模式 | Component 生命周期 | 统一流程，子类扩展 |
| 工厂模式 | AResourceManager | 资源创建抽象 |
| 策略模式 | RendererPath | 不同渲染路径切换 |
| 代理模式 | ScriptObject | C++/C# 对象桥接 |
| 缓存模式 | Pipeline 缓存, DescriptorSet 缓存 | 减少 GPU 资源创建开销 |
| 外观模式 | Physics | 封装 PhysX API |
| 桥接模式 | RHI ↔ Vulkan 实现 | 分离抽象和实现 |

### 4.2 组件扩展模式

```cpp
// 1. 继承 Component 或其子类
class MyComponent : public Component {
public:
    MyComponent();
    ~MyComponent() override;

    // 2. 重写生命周期方法
    void OnAwake() override;
    void OnStart() override;
    void OnUpdate() override;

private:
    // 3. 添加成员变量
    float m_speed = 1.0f;

    // 4. 启用 RTTR 反射
    RTTR_ENABLE(Component)
};

// 5. 注册 RTTR
RTTR_REGISTRATION {
    rttr::registration::class_<MyComponent>("MyComponent")
        .constructor<>()
        .property("speed", &MyComponent::m_speed);
}
```

## 5. 反射系统 (RTTR)

### 5.1 基本使用

```cpp
// 类定义
class Player : public ScriptObject {
public:
    std::string name;
    int level;
    float health;

    RTTR_ENABLE(ScriptObject)
};

// 注册
RTTR_REGISTRATION {
    rttr::registration::class_<Player>("Player")
        .constructor<>()
        .property("name", &Player::name)
        .property("level", &Player::level)
        .property("health", &Player::health);
}
```

### 5.2 属性访问

```cpp
// 获取类型
rttr::type t = rttr::type::get<Player>();

// 获取属性
rttr::property prop = t.get_property("health");

// 设置值
Player player;
prop.set_value(player, 100.0f);

// 获取值
float health = prop.get_value(player).get_value<float>();
```

## 6. 事件系统

### 6.1 定义事件

```cpp
class GameObject {
public:
    Event<Component*> ComponentAddedEvent;
    Event<GameObject*> DestroyedEvent;
};
```

### 6.2 订阅事件

```cpp
// 使用 += 订阅
gameObject->ComponentAddedEvent += [](Component* component) {
    DEBUG_LOG_INFO("Component added: {}", component->GetObjectName());
};

// 使用 ListenerID 管理订阅
ListenerID id = gameObject->DestroyedEvent += [](GameObject* go) {
    DEBUG_LOG_INFO("GameObject destroyed: {}", go->GetName());
};

// 取消订阅
gameObject->DestroyedEvent -= id;
```

## 7. 资源管理

### 7.1 资源路径约定

```cpp
// 引擎资源路径 (以 : 开头)
":Textures/default.png"
":Shaders/PBRTest.hlsl"

// 项目资源路径 (相对路径)
"Textures/my_texture.png"
"Materials/my_material.mat"
```

### 7.2 资源加载

```cpp
// 通过资源管理器加载
TextureManager textureManager;
auto* texture = textureManager.LoadResource("Textures/diffuse.png");

// 通过 AssetManager 加载
Scene scene;
AssetManager::LoadAsset("Scenes/MainLevel.scene", scene);
```

## 8. 常量缓冲区布局

### 8.1 寄存器分配

| 寄存器类型 | 范围 | 用途 |
|------------|------|------|
| b0-b9 | 引擎常量缓冲 | Frame, Light, Material |
| b10+ | 材质常量缓冲 | 自定义材质数据 |
| t0-t99 | 引擎纹理 | 系统纹理 |
| t100+ | 材质纹理 | 材质贴图 |
| s0-s9 | 采样器 | 预定义采样器 |
| u0-u9 | UAV | 结构化缓冲 |

### 8.2 着色器命名

```hlsl
// 常量缓冲: buffer_xxx
cbuffer BufferFrame : register(b0) { FrameBufferData buffer_frame; };

// 纹理: u_xxx 或 tex_xxx
Texture2D u_albedo : register(t101);

// 采样器: samplers[枚举]
SamplerState samplers[] : register(s0);
```

## 9. 生命周期管理

### 9.1 GameObject 生命周期

```
Created → Active → Inactive → Destroyed
    │        │         │
    │        │         └── OnDestroy()
    │        │
    │        └── OnDisable()
    │
    └── OnAwake() → OnEnable() → OnStart()
```

### 9.2 Component 生命周期

```cpp
OnAwake()     // 场景开始时调用一次
OnEnable()    // 组件启用时调用
OnStart()     // OnAwake 后调用一次
OnUpdate()    // 每帧调用
OnFixedUpdate() // 物理帧调用
OnLateUpdate()  // OnUpdate 后调用
OnDisable()   // 组件禁用时调用
OnDestroy()   // 组件销毁时调用
```

## 10. 最佳实践

### 10.1 内存管理

- 使用智能指针 (`std::shared_ptr`, `std::unique_ptr`) 管理资源
- 使用 `Ref<T>` 别名简化 `std::shared_ptr<T>`
- GPU 资源通过延迟删除队列安全释放

### 10.2 线程安全

- 使用 `std::mutex` 保护共享数据
- 使用 `std::atomic` 进行原子操作
- 主线程负责渲染，工作线程处理资源加载

### 10.3 错误处理

- 使用 `DEBUG_LOG_ERROR` 记录错误
- 使用断言检查前置条件
- 返回 `nullptr` 或 `false` 表示失败

### 10.4 性能优化

- 使用对象池缓存 GPU 资源
- 使用视锥剔除减少绘制调用
- 按材质排序减少状态切换
- 使用实例化合并相同网格

## 11. 代码风格

### 11.1 大括号风格

```cpp
// 函数和类定义
class MyClass {
public:
    void Method() {
        // ...
    }
};

// 控制语句
if (condition) {
    // ...
} else {
    // ...
}
```

### 11.2 注释风格

```cpp
/**
 * @brief 简要描述
 * @param paramName 参数描述
 * @return 返回值描述
 */
void Method(int paramName);

// 单行注释
int value = 0;  // 行尾注释
```

### 11.3 包含顺序

```cpp
// 1. 标准库
#include <memory>
#include <string>

// 2. 第三方库
#include "mono/jit/jit.h"

// 3. 引擎头文件
#include "Runtime/Core/Math/Vector3.h"

// 4. 相对路径头文件
#include "Component.h"
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
