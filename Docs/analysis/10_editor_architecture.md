# 编辑器架构分析

## 1. 模块概述

编辑器位于 `Engine/Source/Editor/`，基于 ImGui 构建，提供场景编辑、资源管理、游戏预览等功能。采用面板化设计，支持灵活的窗口布局。

## 2. 架构设计

### 2.1 类关系

```
ApplicationEditor (编辑器应用)
    │
    ├── UIManager (UI 管理器)
    │
    ├── PanelsManager (面板管理器)
    │       ├── MenuBar (菜单栏)
    │       ├── Hierarchy (层级面板)
    │       ├── Inspector (检视面板)
    │       ├── SceneView (场景视图)
    │       ├── GameView (游戏视图)
    │       ├── AssetBrowser (资源浏览器)
    │       ├── Console (控制台)
    │       ├── Profiler (性能分析)
    │       └── ... (其他面板)
    │
    ├── EditorActions (编辑器操作)
    │
    └── RendererPath (渲染路径)
            ├── m_rendererPath4SceneView
            ├── m_rendererPath4GameView
            └── m_rendererPath4AssetView
```

## 3. ApplicationEditor - 编辑器应用

### 3.1 核心职责

- 初始化编辑器环境
- 管理编辑器生命周期
- 协调 UI 和渲染

### 3.2 类定义

```cpp
class ApplicationEditor : public ApplicationBase {
public:
    ApplicationEditor();
    ~ApplicationEditor();

    // 应用类型
    LitchiApplicationType GetApplicationType() override;

    // 生命周期
    bool Initialize() override;
    void Run() override;
    void Update() override;

    // 窗口设置
    WindowSettings CreateWindowSettings() override;

    // 场景回调
    void OnSceneLoaded() override;

    // 状态
    bool IsRunning() const;
    static ApplicationEditor* Instance();

    // 路径
    const std::string& GetEditorAssetsPath() { return m_editorAssetsPath; }

    // 渲染
    void RenderViews(float p_deltaTime);
    void RenderUI();

    // 选择
    void SelectActor(GameObject* p_target);
    void MoveToTarget(GameObject* p_target);

public:
    std::unique_ptr<UIManager> uiManager;
    PanelsManager m_panelsManager;

    RendererPath* m_rendererPath4SceneView = nullptr;
    RendererPath* m_rendererPath4GameView = nullptr;
    RendererPath* m_rendererPath4AssetView = nullptr;

protected:
    std::string m_editorAssetsPath;

private:
    void RunProjectHub();
    void OnProjectOpen();
    void SetupEditorUI();
    void SetupRendererPath();

    float m_restFixedTime = 0.0f;
    uint64_t m_elapsedFrames = 0;
    Canvas m_canvas;
    static ApplicationEditor* instance_;
    EditorActions m_editorActions;
};
```

### 3.3 初始化流程

```cpp
bool ApplicationEditor::Initialize() {
    // 1. 初始化基类
    ApplicationBase::Initialize();

    // 2. 设置编辑器资源路径
    m_editorAssetsPath = "Editor/Assets/";

    // 3. 初始化 UI 管理器
    uiManager = std::make_unique<UIManager>();

    // 4. 设置编辑器 UI
    SetupEditorUI();

    // 5. 设置渲染路径
    SetupRendererPath();

    return true;
}

void ApplicationEditor::SetupEditorUI() {
    // 创建面板
    m_panelsManager.CreatePanel<MenuBar>("Menu Bar");
    m_panelsManager.CreatePanel<Hierarchy>("Hierarchy", true);
    m_panelsManager.CreatePanel<Inspector>("Inspector", true);
    m_panelsManager.CreatePanel<SceneView>("Scene View", true);
    m_panelsManager.CreatePanel<GameView>("Game View", true);
    m_panelsManager.CreatePanel<AssetBrowser>("Asset Browser", true);
    m_panelsManager.CreatePanel<Console>("Console", true);
    m_panelsManager.CreatePanel<Profiler>("Profiler", false);
    // ...
}

void ApplicationEditor::SetupRendererPath() {
    // 创建渲染路径
    m_rendererPath4SceneView = new RendererPath(RendererPathType::SceneView);
    m_rendererPath4GameView = new RendererPath(RendererPathType::GameView);
    m_rendererPath4AssetView = new RendererPath(RendererPathType::AssetView);
}
```

### 3.4 主循环

```cpp
void ApplicationEditor::Run() {
    while (!m_window->ShouldClose()) {
        // 1. 计算帧时间
        float deltaTime = CalculateDeltaTime();

        // 2. 处理输入
        ProcessInput();

        // 3. 更新编辑器
        Update();

        // 4. 渲染视图
        RenderViews(deltaTime);

        // 5. 渲染 UI
        RenderUI();

        // 6. 呈现
        Present();
    }
}

void ApplicationEditor::Update() {
    // 更新场景
    if (IsRunning()) {
        m_sceneManager->GetCurrentScene()->Update();
        m_sceneManager->GetCurrentScene()->FixedUpdate();
        m_sceneManager->GetCurrentScene()->LateUpdate();
    } else {
        m_sceneManager->GetCurrentScene()->OnEditorUpdate();
    }
}
```

## 4. PanelsManager - 面板管理器

### 4.1 核心职责

- 创建和管理面板
- 注册面板到菜单栏
- 添加面板到画布

### 4.2 类定义

```cpp
class PanelsManager {
public:
    PanelsManager(LitchiRuntime::Canvas& p_canvas);

    // 创建面板
    template<typename T, typename... Args>
    void CreatePanel(const std::string& p_id, Args&&... p_args) {
        if constexpr (std::is_base_of<PanelWindow, T>::value) {
            m_panels.emplace(p_id, std::make_unique<T>(p_id, std::forward<Args>(p_args)...));
            T& instance = *static_cast<T*>(m_panels.at(p_id).get());
            GetPanelAs<LitchiEditor::MenuBar>("Menu Bar").RegisterPanel(instance.name, instance);
        } else {
            m_panels.emplace(p_id, std::make_unique<T>(std::forward<Args>(p_args)...));
        }

        m_canvas.AddPanel(*m_panels.at(p_id));
    }

    // 获取面板
    template<typename T>
    T& GetPanelAs(const std::string& p_id) {
        return *static_cast<T*>(m_panels[p_id].get());
    }

private:
    std::unordered_map<std::string, std::unique_ptr<LitchiRuntime::APanel>> m_panels;
    LitchiRuntime::Canvas& m_canvas;
};
```

## 5. Inspector - 检视面板

### 5.1 核心职责

- 显示选中 GameObject 的属性
- 编辑组件属性
- 添加/移除组件

### 5.2 类定义

```cpp
class Inspector : public LitchiRuntime::PanelWindow {
public:
    Inspector(const std::string& p_title, bool p_opened,
              const LitchiRuntime::PanelWindowSettings& p_windowSettings);
    ~Inspector();

    // 焦点控制
    void FocusActor(GameObject* p_target);
    void UnFocus();
    void SoftUnFocus();

    // 获取目标
    GameObject* GetTargetActor() const;

    // 创建检视器
    void CreateActorInspector(GameObject* p_target);

    // 绘制组件
    void DrawComponent(std::string name, Component* p_component);

    // 刷新
    void Refresh();

private:
    void OnDraw() override;

    // RTTR 属性绘制
    void DrawInstance(WidgetContainer& p_root, rttr::instance ins, Object* obj);
    void DrawInstanceInternalRecursively(WidgetContainer& p_root, const rttr::instance& inputIns, 
                                          Object* obj, std::vector<std::string> propertyPathList);
    bool DrawCustomInstanceInternal(WidgetContainer& p_root, rttr::property prop, rttr::variant prop_value, 
                                     const rttr::string_view name, Object* obj, std::vector<std::string> propertyPathList);
    void DrawArray(WidgetContainer& p_root, const rttr::variant_sequential_view& view, 
                   const rttr::string_view propertyName, Object* obj, std::vector<std::string> propertyPathList);
    bool DrawProperty(WidgetContainer& p_root, const rttr::variant& var, const rttr::string_view propertyName, 
                      Object* obj, std::vector<std::string> propertyPathList);
    bool DrawAtomicTypeObject(WidgetContainer& p_root, const rttr::type& t, const rttr::variant& var, 
                               const rttr::string_view propertyName, Object* obj, std::vector<std::string> propertyPathList);

    void NeedRefresh() { m_needRefresh = true; }
    void ResetNeedRefresh() { m_needRefresh = false; }

private:
    GameObject* m_targetActor = nullptr;
    Group* m_actorInfo;
    Group* m_inspectorHeader;
    ComboBox* m_componentSelectorWidget;
    InputText* m_scriptSelectorWidget;

    // 事件监听器
    uint64_t m_componentAddedListener = 0;
    uint64_t m_componentRemovedListener = 0;
    uint64_t m_behaviourAddedListener = 0;
    uint64_t m_behaviourRemovedListener = 0;
    uint64_t m_destroyedListener = 0;

    bool m_needRefresh;
};
```

### 5.3 属性绘制

```cpp
void Inspector::CreateActorInspector(GameObject* p_target) {
    m_targetActor = p_target;

    // 清空内容
    Clear();

    // 绘制 GameObject 信息
    m_actorInfo = &CreateWidget<Group>();
    m_actorInfo->CreateWidget<InputText>("Name", p_target->GetName());

    // 绘制所有组件
    for (auto& component : p_target->GetComponents()) {
        DrawComponent(component->GetObjectName(), component);
    }

    // 组件选择器
    m_componentSelectorWidget = &CreateWidget<ComboBox>();
    m_componentSelectorWidget->valueChangedEvent += [this](int choice) {
        // 添加组件
    };
}

void Inspector::DrawComponent(std::string name, Component* p_component) {
    // 创建折叠组
    auto& group = CreateWidget<Group>();

    // 绘制组件属性 (使用 RTTR 反射)
    rttr::instance ins = *p_component;
    DrawInstance(group, ins, p_component);
}
```

## 6. Hierarchy - 层级面板

### 6.1 核心职责

- 显示场景中的 GameObject 层级
- 支持选择、删除、重命名
- 拖拽重排层级

### 6.2 功能

```cpp
class Hierarchy : public LitchiRuntime::PanelWindow {
public:
    Hierarchy(const std::string& p_title, bool p_opened,
              const LitchiRuntime::PanelWindowSettings& p_windowSettings);

    // 刷新层级
    void Refresh();

private:
    void OnDraw() override;

    // 绘制 GameObject 节点
    void DrawGameObject(GameObject* p_gameObject);

    // 处理拖拽
    void HandleDragDrop();

    // 上下文菜单
    void ShowContextMenu(GameObject* p_gameObject);
};
```

## 7. SceneView - 场景视图

### 7.1 核心职责

- 渲染场景预览
- 处理编辑器相机控制
- 支持 Gizmo 操作

### 7.2 类定义

```cpp
class SceneView : public AViewControllable {
public:
    SceneView(const std::string& p_title, bool p_opened,
              const LitchiRuntime::PanelWindowSettings& p_windowSettings);

    // 更新
    void Update() override;

    // 渲染
    void Render() override;

    // Gizmo
    void DrawGizmo();

private:
    // 编辑器相机
    CameraController m_cameraController;

    // Gizmo 行为
    GizmoBehaviour m_gizmoBehaviour;

    // 渲染路径
    RendererPath* m_rendererPath;
};
```

## 8. GameView - 游戏视图

### 8.1 核心职责

- 渲染游戏画面
- 支持游戏预览
- 分辨率设置

### 8.2 类定义

```cpp
class GameView : public AView {
public:
    GameView(const std::string& p_title, bool p_opened,
             const LitchiRuntime::PanelWindowSettings& p_windowSettings);

    void Render() override;

private:
    RendererPath* m_rendererPath;
    Vector2 m_resolution;
};
```

## 9. AssetBrowser - 资源浏览器

### 9.1 核心职责

- 浏览项目资源
- 拖拽导入资源
- 资源预览

### 9.2 功能

```cpp
class AssetBrowser : public LitchiRuntime::PanelWindow {
public:
    AssetBrowser(const std::string& p_title, bool p_opened,
                 const LitchiRuntime::PanelWindowSettings& p_windowSettings);

    // 刷新资源列表
    void Refresh();

    // 导入资源
    void ImportAsset(const std::string& p_path);

private:
    void OnDraw() override;

    // 绘制资源项
    void DrawAssetItem(const std::string& p_path);

    // 处理拖拽
    void HandleDragDrop();

    std::string m_currentDirectory;
    std::vector<std::string> m_assets;
};
```

## 10. EditorActions - 编辑器操作

### 10.1 核心职责

- 提供编辑器操作接口
- 封装常用操作

### 10.2 功能

```cpp
class EditorActions {
public:
    // 场景操作
    void LoadScene(const std::string& p_path);
    void SaveScene();
    void SaveSceneAs(const std::string& p_path);

    // GameObject 操作
    void CreateEmptyGameObject();
    void DeleteGameObject(GameObject* p_gameObject);
    void DuplicateGameObject(GameObject* p_gameObject);

    // 播放控制
    void Play();
    void Pause();
    void Stop();

    // 编辑操作
    void Undo();
    void Redo();

    // 相机操作
    void FocusOnSelection();
    void FrameSelected();
};
```

## 11. 面板列表

| 面板 | 文件位置 | 职责 |
|------|----------|------|
| MenuBar | Panels/MenuBar.h | 菜单栏 |
| Hierarchy | Panels/Hierarchy.h | 层级面板 |
| Inspector | Panels/Inspector.h | 检视面板 |
| SceneView | Panels/SceneView.h | 场景视图 |
| GameView | Panels/GameView.h | 游戏视图 |
| AssetBrowser | Panels/AssetBrowser.h | 资源浏览器 |
| Console | Panels/Console.h | 控制台 |
| Profiler | Panels/Profiler.h | 性能分析 |
| MaterialEditor | Panels/MaterialEditor.h | 材质编辑器 |
| AssetView | Panels/AssetView.h | 资源预览 |
| Toolbar | Panels/Toolbar.h | 工具栏 |
| ProjectSettings | Panels/ProjectSettings.h | 项目设置 |

## 12. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | ApplicationEditor | 全局编辑器访问 |
| 工厂模式 | PanelsManager::CreatePanel | 创建面板实例 |
| 观察者模式 | Inspector 事件监听 | 组件变化通知 |
| 模板方法模式 | AView::Render | 统一渲染流程 |
| 组合模式 | Canvas/Panel | UI 层级结构 |

## 13. ImGui 集成

### 13.1 初始化

```cpp
void UIManager::Initialize() {
    // 创建 ImGui 上下文
    ImGui::CreateContext();

    // 设置样式
    ImGui::StyleColorsDark();

    // 初始化平台/渲染后端
    ImGui_ImplGlfw_InitForVulkan(m_window->GetHandle(), true);
    ImGui_ImplVulkan_Init(m_device, m_renderPass);
}
```

### 13.2 渲染

```cpp
void UIManager::Render() {
    // 开始帧
    ImGui_ImplVulkan_NewFrame();
    ImGui_ImplGlfw_NewFrame();
    ImGui::NewFrame();

    // 绘制所有面板
    m_canvas.Draw();

    // 结束帧
    ImGui::Render();
    ImGui_ImplVulkan_RenderDrawData(ImGui::GetDrawData(), m_commandBuffer);
}
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
