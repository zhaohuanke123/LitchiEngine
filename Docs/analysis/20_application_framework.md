# 应用程序框架分析

## 1. 概述

LitchiEngine 的应用程序框架采用经典的多态设计，通过 `ApplicationBase` 基类定义统一的应用程序生命周期接口，由编辑器应用 (`ApplicationEditor`) 和独立应用 (`ApplicationStandalone`) 分别实现具体行为。框架使用服务定位器模式 (`ServiceLocator`) 管理全局服务访问，实现了引擎核心模块之间的解耦。

## 2. 核心类设计

### 2.1 类继承层次

```
ApplicationBase (抽象基类)
    ├── ApplicationEditor (编辑器应用)
    └── ApplicationStandalone (独立游戏应用)

Application (静态入口类)
    └── 管理 ApplicationBase 实例的生命周期
```

### 2.2 ApplicationBase 类

**文件**: `Engine/Source/Runtime/Core/App/ApplicationBase.h`

```cpp
class ApplicationBase {
public:
    ApplicationBase() {}
    virtual ~ApplicationBase() {}

    // 应用类型标识
    virtual LitchiApplicationType GetApplicationType() = 0;

    // 生命周期方法
    virtual bool Initialize();
    virtual void Run();
    virtual void Update();
    virtual void Exit();

    // 窗口配置
    virtual WindowSettings CreateWindowSettings() = 0;

    // 回调钩子
    virtual void OnSceneLoaded();
    virtual void OnApplyProjectSettings();
    virtual void OnResetProjectSettings();

    // 资源管理器 (所有权属于 ApplicationBase)
    std::unique_ptr<ConfigManager> configManager;
    std::unique_ptr<Window> window;
    std::unique_ptr<ModelManager> modelManager;
    std::unique_ptr<ShaderManager> shaderManager;
    std::unique_ptr<MaterialManager> materialManager;
    std::unique_ptr<FontManager> fontManager;
    std::unique_ptr<TextureManager> textureManager;
    std::unique_ptr<SceneManager> sceneManager;
    std::unique_ptr<PrefabManager> prefabManager;

    // 单例访问
    static ApplicationBase* Instance() { return s_instance; }
    static ApplicationBase* s_instance;

protected:
    std::string m_engineAssetsPath;   // 引擎资源路径
    std::string m_engineRootPath;     // 引擎根目录
    std::string m_projectPath;        // 项目路径
    std::string m_projectName;        // 项目名称
    std::string m_title;              // 应用标题
};
```

**应用类型枚举**:

```cpp
enum class LitchiApplicationType {
    Editor,  // 编辑器模式
    Game     // 游戏运行模式
};
```

### 2.3 Application 入口类

**文件**: `Engine/Source/Runtime/Core/App/application.h`

```cpp
class Application {
public:
    // 初始化应用实例
    static void Initialize(ApplicationBase* instance);

    // 运行应用
    static void Run();

private:
    static ApplicationBase* s_instance;
};
```

**实现**:

```cpp
void Application::Initialize(ApplicationBase* instance) {
    s_instance = instance;
    if(!s_instance->Initialize()) {
        throw std::runtime_error("Failed to Initialization Application");
    }
}

void Application::Run() {
    s_instance->Run();
}
```

### 2.4 ServiceLocator 服务定位器

**文件**: `Engine/Source/Runtime/Core/Global/ServiceLocator.h`

```cpp
#define OVSERVICE(Type) LitchiRuntime::ServiceLocator::Get<Type>()

class ServiceLocator {
public:
    // 注册服务
    template<typename T>
    static void Provide(T& p_service) {
        __SERVICES[typeid(T).hash_code()] = std::any(&p_service);
    }

    // 获取服务
    template<typename T>
    static T& Get() {
        return *std::any_cast<T*>(__SERVICES[typeid(T).hash_code()]);
    }

private:
    static std::unordered_map<size_t, std::any> __SERVICES;
};
```

**设计特点**:
- 使用 `std::any` 实现类型擦除，支持任意类型的服务
- 通过 `typeid(T).hash_code()` 作为键，实现类型安全的访问
- 提供宏 `OVSERVICE(Type)` 简化调用
- 仅存储指针，所有权仍由 `ApplicationBase` 持有

## 3. 应用程序生命周期

### 3.1 生命周期流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        main() 入口                               │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  创建具体应用实例 (ApplicationEditor / ApplicationStandalone)    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              Application::Initialize(instance)                   │
│  ├─ 保存实例指针                                                 │
│  └─ 调用 instance->Initialize()                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  ApplicationBase::Initialize()                   │
│  ├─ 设置单例实例 s_instance                                      │
│  ├─ 初始化路径 (engineRootPath, engineAssetsPath)               │
│  ├─ 初始化 ConfigManager (如果需要)                             │
│  ├─ 启动 Profiler 服务                                          │
│  ├─ 初始化 Debug 系统                                           │
│  ├─ 设置 FileSystem 资源路径                                    │
│  ├─ 创建资源管理器 (Scene, Shader, Material, etc.)              │
│  ├─ 注册服务到 ServiceLocator                                   │
│  ├─ 初始化 Time 系统                                            │
│  ├─ 初始化资源导入器 (Font, Model, Image)                       │
│  ├─ 创建窗口 (Window)                                           │
│  ├─ 初始化 InputManager                                         │
│  ├─ 初始化 Renderer                                             │
│  ├─ 初始化 Physics                                              │
│  └─ 注册场景加载事件回调                                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              子类::Initialize() 扩展初始化                        │
│  Editor: 创建 UIManager, 设置编辑器资源路径, 运行项目选择器      │
│  Standalone: 设置渲染路径, 加载场景, 隐藏控制台                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Application::Run()                            │
│  └─ 调用 instance->Run()                                        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       主循环                                     │
│  while (IsRunning()) {                                          │
│      ├─ window->PollEvents()        // 事件轮询                 │
│      ├─ Update()                    // 逻辑更新                 │
│      ├─ Renderer::Tick()            // 渲染帧                   │
│      ├─ RenderViews() / RenderUI()  // 视图/UI渲染              │
│      └─ InputManager::ClearEvents() // 清理输入事件             │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    应用退出                                       │
│  ├─ 析构函数释放资源                                             │
│  └─ delete application_instance                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 初始化顺序详解

`ApplicationBase::Initialize()` 的初始化顺序经过精心设计：

| 顺序 | 步骤 | 说明 |
|------|------|------|
| 1 | 路径初始化 | 设置引擎根目录和资源目录 |
| 2 | ConfigManager | 项目配置加载（仅 Game 模式或项目路径有效时） |
| 3 | Profiler | Easy Profiler 启动，支持性能分析 |
| 4 | Debug | 调试系统初始化 |
| 5 | FileSystem | 设置资源搜索路径 |
| 6 | 资源管理器 | 创建 Scene/Shader/Material/Texture/Model/Font/Prefab 管理器 |
| 7 | ServiceLocator | 注册所有管理器为全局服务 |
| 8 | Time | 时间系统初始化 |
| 9 | 资源导入器 | Font/Model/Image 导入器初始化 |
| 10 | Window | 创建窗口并设置图标 |
| 11 | InputManager | 输入管理器初始化（依赖 Window） |
| 12 | Renderer | 渲染系统初始化 |
| 13 | Physics | 物理引擎初始化 |
| 14 | 事件注册 | 场景加载事件回调 |

## 4. 主循环分析

### 4.1 编辑器主循环 (ApplicationEditor::Run)

```cpp
void ApplicationEditor::Run() {
    while (IsRunning()) {
        EASY_BLOCK("Frame") {
            // 1. 事件轮询
            window->PollEvents();

            // 2. 逻辑更新
            EASY_BLOCK("Update") {
                Update();
            } EASY_END_BLOCK;

            // 3. 渲染 (窗口未最小化时)
            if (!ApplicationBase::Instance()->window->IsMinimized()) {
                EASY_BLOCK("Renderer") {
                    Renderer::Tick();
                } EASY_END_BLOCK;

                EASY_BLOCK("RenderViews") {
                    RenderViews(Time::GetDeltaTime());
                } EASY_END_BLOCK;

                EASY_BLOCK("RenderUI") {
                    RenderUI();
                } EASY_END_BLOCK;
            }

            // 4. 帧后处理
            InputManager::ClearEvents();
            ++m_elapsedFrames;
        } EASY_END_BLOCK;
    }
}
```

### 4.2 独立应用主循环 (ApplicationStandalone::Run)

```cpp
void ApplicationStandalone::Run() {
    while (IsRunning()) {
        EASY_BLOCK("Frame") {
            // 1. 事件轮询
            window->PollEvents();

            // 2. 逻辑更新
            EASY_BLOCK("Update") {
                Update();
            } EASY_END_BLOCK;

            // 3. 渲染
            EASY_BLOCK("Renderer") {
                if (!ApplicationBase::Instance()->window->IsMinimized()) {
                    Renderer::Tick();
                }
            } EASY_END_BLOCK;

            // 4. 帧后处理
            InputManager::ClearEvents();
            ++m_elapsedFrames;
        } EASY_END_BLOCK;
    }
}
```

### 4.3 主循环对比

| 特性 | Editor | Standalone |
|------|--------|------------|
| 渲染视图 | SceneView + GameView + AssetView | 仅 GameView |
| UI 渲染 | ImGui 编辑器界面 | 无 |
| ProjectHub | 启动时显示项目选择器 | 直接加载项目 |
| 窗口装饰 | 标准窗口 | 可选无边框/全屏 |

### 4.4 Update 方法详解

**基类 Update**:

```cpp
void ApplicationBase::Update() {
    Time::Update();              // 更新时间系统
    InputManager::Tick();        // 处理输入
    configManager->Tick(Time::GetDeltaTime());  // 配置热重载
}
```

**编辑器 Update**:

```cpp
void ApplicationEditor::Update() {
    ApplicationBase::Update();

    auto scene = sceneManager->GetCurrentScene();
    if (scene) {
        if (m_editorActions.GetCurrentEditorMode() == EEditorMode::PLAY ||
            m_editorActions.GetCurrentEditorMode() == EEditorMode::FRAME_BY_FRAME) {
            // 游戏模式：执行完整的游戏循环
            m_restFixedTime += Time::GetDeltaTime();
            float fixedDeltaTime = Time::GetFixedUpdateTime();

            // 固定时间步长物理更新
            while (m_restFixedTime > fixedDeltaTime) {
                Physics::FixedUpdate(fixedDeltaTime);
                scene->FixedUpdate();
                m_restFixedTime -= fixedDeltaTime;
            }

            scene->Update();
            scene->LateUpdate();
        } else {
            // 编辑模式：仅更新编辑器视图
            scene->OnEditorUpdate();
        }
    }
}
```

**独立应用 Update**:

```cpp
void ApplicationStandalone::Update() {
    ApplicationBase::Update();

    auto scene = sceneManager->GetCurrentScene();
    if (scene->IsPlaying()) {
        // 固定时间步长物理更新
        m_restFixedTime += Time::GetDeltaTime();
        float fixedDeltaTime = Time::GetFixedUpdateTime();

        while (m_restFixedTime > fixedDeltaTime) {
            Physics::FixedUpdate(fixedDeltaTime);
            scene->FixedUpdate();
            m_restFixedTime -= fixedDeltaTime;
        }

        scene->Update();
        scene->LateUpdate();
    }
}
```

## 5. 编辑器与独立应用差异

### 5.1 ApplicationEditor 特有功能

| 功能 | 说明 |
|------|------|
| ProjectHub | 项目选择/创建界面 |
| UIManager | ImGui 编辑器 UI 管理 |
| PanelsManager | 编辑器面板管理 |
| EditorActions | 编辑器操作（播放/暂停/步进） |
| 多渲染视图 | SceneView/GameView/AssetView |
| 编辑模式 | 支持编辑/播放/帧步进三种模式 |

**初始化流程**:

```cpp
bool ApplicationEditor::Initialize() {
    // 1. 调用基类初始化
    if (!ApplicationBase::Initialize()) return false;

    // 2. 设置编辑器资源路径
    m_editorAssetsPath = PathParser::MakeNonWindowsStyle(
        std::filesystem::canonical("Data/Editor").string() + "/");

    // 3. 初始化 UIManager
    uiManager = std::make_unique<UIManager>(window->GetGlfwWindow(), EStyle::DUNE_DARK);
    ServiceLocator::Provide<UIManager>(*uiManager.get());

    // 4. 运行项目选择器
    RunProjectHub();

    return true;
}
```

**项目打开流程**:

```cpp
void ApplicationEditor::OnProjectOpen() {
    // 1. 初始化项目配置
    configManager = std::make_unique<ConfigManager>();
    configManager->Initialize(m_projectPath);

    // 2. 更新资源路径
    FileSystem::SetAssetDirectoryPath(projectAssetsPath, m_engineAssetsPath);

    // 3. 应用项目设置
    OnApplyProjectSettings();

    // 4. 设置渲染路径
    SetupRendererPath();

    // 5. 创建编辑器 UI
    SetupEditorUI();

    // 6. 加载默认场景
    if (!configManager->GetDefaultScenePath().empty()) {
        EDITOR_EXEC(LoadSceneFromDisk(defaultScene));
    } else {
        EDITOR_EXEC(LoadEmptyScene());
    }
}
```

### 5.2 ApplicationStandalone 特有功能

| 功能 | 说明 |
|------|------|
| 无项目选择器 | 直接从工作目录加载项目 |
| 隐藏控制台 | 游戏运行时隐藏控制台窗口 |
| 单一渲染视图 | 仅 GameView 渲染 |
| 全屏支持 | 支持无边框/全屏模式 |

**初始化流程**:

```cpp
bool ApplicationStandalone::Initialize() {
    // 1. 调用基类初始化
    if (!ApplicationBase::Initialize()) return false;

    // 2. 设置渲染路径
    SetupRendererPath();

    // 3. 加载默认场景
    sceneManager->LoadScene("Scenes\\New Scene4.scene");
    sceneManager->GetCurrentScene()->Resolve();
    sceneManager->GetCurrentScene()->Play();

    // 4. 创建相机
    auto cameraObject = sceneManager->GetCurrentScene()->CreateGameObject("Camera");
    // ... 设置相机参数

    // 5. 隐藏控制台
    ConsoleHelper::HideConsole();

    return true;
}
```

### 5.3 窗口设置对比

**编辑器窗口**:

```cpp
WindowSettings ApplicationEditor::CreateWindowSettings() {
    WindowSettings settings;
    settings.title = "Litchi Editor";
    settings.width = 1000;
    settings.height = 580;
    settings.minimumWidth = 1;
    settings.minimumHeight = 1;
    settings.maximized = true;
    return settings;
}
```

**独立应用窗口**:

```cpp
WindowSettings ApplicationStandalone::CreateWindowSettings() {
    WindowSettings settings;
    settings.title = "Litchi Standalone";
    settings.width = 1920;
    settings.height = 1080;
    settings.decorated = false;  // 无边框

    // 从项目配置读取分辨率和全屏设置
    if (configManager) {
        auto size = configManager->GetResolutionSize();
        settings.width = size.first;
        settings.height = size.second;
        settings.fullscreen = configManager->IsFullScreen();
    }
    return settings;
}
```

## 6. 服务定位器使用

### 6.1 服务注册

在 `ApplicationBase::Initialize()` 中注册所有核心服务：

```cpp
// 创建资源管理器
sceneManager = std::make_unique<SceneManager>();
shaderManager = std::make_unique<ShaderManager>();
materialManager = std::make_unique<MaterialManager>();
textureManager = std::make_unique<TextureManager>();
modelManager = std::make_unique<ModelManager>();
fontManager = std::make_unique<FontManager>();
prefabManager = std::make_unique<PrefabManager>();

// 注册到服务定位器
ServiceLocator::Provide(*sceneManager.get());
ServiceLocator::Provide(*shaderManager.get());
ServiceLocator::Provide(*materialManager.get());
ServiceLocator::Provide(*textureManager.get());
ServiceLocator::Provide(*modelManager.get());
ServiceLocator::Provide(*fontManager.get());
ServiceLocator::Provide(*prefabManager.get());
```

编辑器额外注册 UIManager:

```cpp
uiManager = std::make_unique<UIManager>(...);
ServiceLocator::Provide<UIManager>(*uiManager.get());
```

### 6.2 服务获取

**使用宏获取服务**:

```cpp
// 获取 SceneManager
auto& sceneManager = OVSERVICE(SceneManager);

// 获取 TextureManager
auto& textureManager = OVSERVICE(TextureManager);
```

**直接调用**:

```cpp
auto& sceneManager = ServiceLocator::Get<SceneManager>();
```

### 6.3 设计优势

| 优势 | 说明 |
|------|------|
| 解耦 | 模块间通过接口访问，而非直接依赖 |
| 灵活性 | 可在运行时替换服务实现 |
| 简洁性 | 通过宏简化访问代码 |
| 类型安全 | 使用模板确保类型正确 |

## 7. 设计模式

### 7.1 模板方法模式 (Template Method)

`ApplicationBase` 定义了应用程序生命周期的骨架，子类可以重写特定步骤：

```cpp
class ApplicationBase {
public:
    // 骨架方法
    virtual bool Initialize() {
        // 固定流程
        SetupPaths();
        InitializeManagers();
        CreateWindow();
        // 调用子类扩展
        return true;
    }

    // 钩子方法
    virtual void OnSceneLoaded() {}  // 默认空实现
    virtual void OnApplyProjectSettings() {}
};

class ApplicationEditor : public ApplicationBase {
    void OnSceneLoaded() override {
        // 编辑器特定处理：更新渲染路径
        m_rendererPath4SceneView->SetScene(scene);
    }
};
```

### 7.2 单例模式 (Singleton)

`ApplicationBase` 提供全局访问点：

```cpp
class ApplicationBase {
public:
    static ApplicationBase* Instance() { return s_instance; }
    static ApplicationBase* s_instance;
};

// 使用
auto* app = ApplicationBase::Instance();
```

### 7.3 服务定位器模式 (Service Locator)

```cpp
// 注册服务
ServiceLocator::Provide<SceneManager>(sceneManager);

// 获取服务
auto& sceneManager = OVSERVICE(SceneManager);
```

### 7.4 工厂方法模式 (Factory Method)

`CreateWindowSettings()` 由子类实现具体的窗口配置创建：

```cpp
class ApplicationBase {
    virtual WindowSettings CreateWindowSettings() = 0;
};

class ApplicationEditor : public ApplicationBase {
    WindowSettings CreateWindowSettings() override {
        // 创建编辑器窗口配置
    }
};

class ApplicationStandalone : public ApplicationBase {
    WindowSettings CreateWindowSettings() override {
        // 创建游戏窗口配置
    }
};
```

### 7.5 策略模式 (Strategy)

编辑器通过 `EditorActions` 支持不同的运行模式：

```cpp
enum class EEditorMode {
    EDIT,          // 编辑模式
    PLAY,          // 播放模式
    FRAME_BY_FRAME // 帧步进模式
};

void ApplicationEditor::Update() {
    switch (m_editorActions.GetCurrentEditorMode()) {
        case EEditorMode::PLAY:
            // 执行游戏逻辑
            scene->Update();
            break;
        case EEditorMode::EDIT:
            // 执行编辑器逻辑
            scene->OnEditorUpdate();
            break;
    }
}
```

## 8. 关键文件路径

| 模块 | 路径 |
|------|------|
| ApplicationBase | `Engine/Source/Runtime/Core/App/ApplicationBase.h` |
| ApplicationBase.cpp | `Engine/Source/Runtime/Core/App/ApplicationBase.cpp` |
| Application | `Engine/Source/Runtime/Core/App/application.h` |
| Application.cpp | `Engine/Source/Runtime/Core/App/application.cpp` |
| ServiceLocator | `Engine/Source/Runtime/Core/Global/ServiceLocator.h` |
| ApplicationEditor | `Engine/Source/Editor/include/ApplicationEditor.h` |
| ApplicationEditor.cpp | `Engine/Source/Editor/source/ApplicationEditor.cpp` |
| Editor main | `Engine/Source/Editor/source/main.cpp` |
| ApplicationStandalone | `Engine/Source/Standalone/include/ApplicationStandalone.h` |
| ApplicationStandalone.cpp | `Engine/Source/Standalone/source/ApplicationStandalone.cpp` |
| Standalone main | `Engine/Source/Standalone/source/main.cpp` |

## 9. 扩展指南

### 9.1 添加新的应用程序类型

1. 创建新类继承 `ApplicationBase`:

```cpp
class ApplicationServer : public ApplicationBase {
public:
    LitchiApplicationType GetApplicationType() override {
        return LitchiApplicationType::Game;
    }

    bool Initialize() override {
        if (!ApplicationBase::Initialize()) return false;
        // 服务器特定初始化
        return true;
    }

    void Run() override {
        while (IsRunning()) {
            Update();
            // 服务器逻辑
        }
    }

    WindowSettings CreateWindowSettings() override {
        return WindowSettings{};  // 无头服务器可能不需要窗口
    }
};
```

2. 创建入口文件:

```cpp
int main(int argc, char** argv) {
    Application application;
    auto server = new ApplicationServer();
    application.Initialize(server);
    application.Run();
    delete server;
    return 0;
}
```

### 9.2 添加新服务

1. 创建服务类:

```cpp
class NetworkManager {
public:
    void Initialize();
    void Shutdown();
};
```

2. 在 `ApplicationBase` 中添加成员:

```cpp
std::unique_ptr<NetworkManager> networkManager;
```

3. 在 `Initialize()` 中注册:

```cpp
networkManager = std::make_unique<NetworkManager>();
ServiceLocator::Provide(*networkManager.get());
```

4. 使用服务:

```cpp
auto& network = OVSERVICE(NetworkManager);
```

## 10. 总结

LitchiEngine 的应用程序框架设计具有以下特点：

| 设计特点 | 实现方式 |
|----------|----------|
| 统一生命周期 | `ApplicationBase` 定义标准初始化/运行/退出流程 |
| 多态扩展 | 子类重写虚方法实现 Editor/Standalone 差异化 |
| 服务解耦 | `ServiceLocator` 提供全局服务访问 |
| 资源集中管理 | 所有资源管理器由 `ApplicationBase` 持有 |
| 模式应用 | 模板方法、单例、服务定位器、工厂方法、策略模式 |
| 性能可观测 | 集成 Easy Profiler 支持性能分析 |

框架遵循开闭原则，对扩展开放（通过继承添加新应用类型），对修改封闭（核心生命周期流程稳定）。服务定位器模式有效解耦了各模块间的依赖，使得引擎具有良好的可测试性和可维护性。

---

**分析完成时间**: 2026-04-14
**分析者**: AI Agent
