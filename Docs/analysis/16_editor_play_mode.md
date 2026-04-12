# 编辑器运行模式状态机分析

## 概述

LitchiEngine 编辑器实现了完整的运行模式状态机，支持 EDIT、PLAY、PAUSE、FRAME_BY_FRAME 四种模式。本文档详细分析 Play 按钮按下后的逻辑流程和状态变化。

## 编辑器状态定义

### EEditorMode 枚举

```cpp
// EditorActions.h
enum class EEditorMode {
    EDIT,           // 编辑模式：场景可编辑，不执行游戏逻辑
    PLAY,           // 播放模式：游戏正常运行
    PAUSE,          // 暂停模式：游戏逻辑暂停
    FRAME_BY_FRAME  // 单帧模式：执行一帧后暂停
};
```

### 状态转换图

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
┌──────┐  Play   ┌──────┐  Pause   ┌───────┐            │
│ EDIT │ ───────▶│ PLAY │ ────────▶│ PAUSE │            │
└──────┘         └──────┘          └───────┘            │
    ▲               │                   │                │
    │               │                   │ Resume         │
    │               │                   ▼                │
    │               │            ┌─────────────┐        │
    │               │            │ FRAME_BY_   │        │
    │               └───────────▶│   FRAME     │        │
    │                   Next     └─────────────┘        │
    │                   Frame           │                │
    │                                   │                │
    └───────────────────────────────────┴────────────────┘
                      Stop
```

## Play 按钮按下后的完整流程

### 1. 入口点：EditorActions::StartPlaying()

```cpp
// EditorActions.cpp:467-495
void EditorActions::StartPlaying()
{
    if (m_editorMode == EEditorMode::EDIT)
    {
        // 1. 刷新 Inspector 面板
        EDITOR_PANEL(Inspector, "Inspector").Refresh();

        // 2. 触发 PlayEvent
        PlayEvent.Invoke();

        // 3. 获取当前场景
        auto currScene = ApplicationEditor::Instance()->sceneManager->GetCurrentScene();

        // 4. 序列化场景备份 (关键！用于停止时恢复)
        m_sceneBackup = Serializer::SerializeToJson(currScene);

        // 5. 聚焦 GameView 窗口
        m_panelsManager.GetPanelAs<GameView>("Game View").Focus();

        // 6. 调用 Scene::Play() 启动场景
        currScene->Play();

        // 7. 切换编辑器模式
        SetEditorMode(EEditorMode::PLAY);

        // 8. 激活 GameView 渲染路径
        ApplicationEditor::Instance()->m_rendererPath4GameView->SetScene(currScene);
        ApplicationEditor::Instance()->m_rendererPath4GameView->SetActive(true);
    }
    else
    {
        // 从暂停恢复播放
        SetEditorMode(EEditorMode::PLAY);
    }
}
```

### 2. 场景备份机制

**关键设计**：在进入播放模式前，编辑器会将当前场景序列化为 JSON 字符串保存到 `m_sceneBackup`。

```cpp
// EditorActions.h:394
std::string m_sceneBackup;  // 场景备份字符串
```

**目的**：
- 运行时修改场景后，停止时可以恢复到播放前的状态
- 保证编辑器模式下场景数据不丢失

### 3. Scene::Play() - 启动场景

```cpp
// SceneManager.cpp:245-259
void Scene::Play()
{
    // 1. 设置播放标志
    m_isPlaying = true;

    // 2. 唤醒所有 GameObject
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) { p_element->SetSleeping(false); });

    // 3. 调用所有组件的 OnAwake
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->ForeachComponent([](Component* comp) { comp->OnAwake(); });
        });

    // 4. 调用所有组件的 OnEnable
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->ForeachComponent([](Component* comp) { comp->OnEnable(); });
        });

    // 5. 调用所有 GameObject 的 OnStart
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->OnStart();
        });
}
```

### 4. 主循环中的更新逻辑

```cpp
// ApplicationEditor.cpp:173-211
void ApplicationEditor::Update()
{
    ApplicationBase::Update();
    auto scene = this->sceneManager->GetCurrentScene();

    if (scene)
    {
        auto editorMode = m_editorActions.GetCurrentEditorMode();

        if (editorMode == EditorActions::EEditorMode::PLAY ||
            editorMode == EditorActions::EEditorMode::FRAME_BY_FRAME)
        {
            // 检查 GameView 是否聚焦
            auto& gameView = m_panelsManager.GetPanelAs<GameView>("Game View");
            if (!gameView.IsFocused() || !gameView.IsOpened())
            {
                return;
            }

            // 物理固定更新
            m_restFixedTime += Time::GetDeltaTime();
            float fixedDeltaTime = Time::GetFixedUpdateTime();
            while (m_restFixedTime > fixedDeltaTime)
            {
                Physics::FixedUpdate(fixedDeltaTime);
                scene->FixedUpdate();
                m_restFixedTime -= fixedDeltaTime;
            }

            // 游戏逻辑更新
            scene->Update();
            scene->LateUpdate();
        }
        else
        {
            // 编辑模式：只调用 OnEditorUpdate
            scene->OnEditorUpdate();
        }
    }
}
```

## 组件生命周期详解

### 生命周期顺序

```
OnAwake() → OnEnable() → OnStart() → OnUpdate() → OnDisable() → OnDestroy()
```

### 触发时机

| 方法 | 触发时机 | 说明 |
|------|----------|------|
| `OnAwake()` | Scene::Play() 或 GameObject 创建时 | 初始化组件状态，设置 `m_awaked = true` |
| `OnEnable()` | OnAwake 之后，或 SetActive(true) | 组件激活时调用 |
| `OnStart()` | OnEnable 之后 | 首帧初始化，设置 `m_started = true` |
| `OnUpdate()` | 每帧 | 游戏逻辑更新 |
| `OnFixedUpdate()` | 固定时间步长 | 物理更新 |
| `OnLateUpdate()` | OnUpdate 之后 | 后处理更新 |
| `OnDisable()` | SetActive(false) 或停止播放 | 组件禁用 |
| `OnDestroy()` | GameObject 销毁或停止播放 | 清理资源 |

### GameObject 状态标志

```cpp
// GameObject 内部状态
bool m_active = true;      // 是否激活
bool m_awaked = false;     // 是否已调用 OnAwake
bool m_started = false;    // 是否已调用 OnStart
bool m_sleeping = true;    // 是否休眠（编辑模式下为 true）
bool m_isPlaying = false;  // 是否在播放模式
```

### RecursiveActiveUpdate - 激活状态变化处理

```cpp
// GameObject.cpp:285-308
void GameObject::RecursiveActiveUpdate()
{
    bool isActive = GetActive();

    if (!m_sleeping)
    {
        if (!m_wasActive && isActive)
        {
            // 从禁用变为激活
            if (!m_awaked)
                OnAwake();
            OnEnable();
            if (!m_started)
                OnStart();
        }

        if (m_wasActive && !isActive)
        {
            // 从激活变为禁用
            OnDisable();
        }
    }

    // 递归处理子对象
    for (auto child : GetChildren())
        child->RecursiveActiveUpdate();
}
```

## 暂停模式 (PAUSE)

### PauseGame() 实现

```cpp
// EditorActions.cpp:497-501
void EditorActions::PauseGame()
{
    // 当前实现为空，注释掉了音频暂停逻辑
    // SetEditorMode(EEditorMode::PAUSE);
    // LitchiEditor::ApplicationEditor::Instance()->audioEngine->Suspend();
}
```

**注意**：当前实现中 `PauseGame()` 被注释掉了，暂停功能未完全实现。

### 暂停模式下的更新

在 `ApplicationEditor::Update()` 中：
- `PAUSE` 模式下不会调用 `scene->Update()`
- 游戏逻辑被冻结

## 单帧模式 (FRAME_BY_FRAME)

### NextFrame() 实现

```cpp
// EditorActions.cpp:537-541
void EditorActions::NextFrame()
{
    if (m_editorMode == EEditorMode::PLAY || m_editorMode == EEditorMode::PAUSE)
        SetEditorMode(EEditorMode::FRAME_BY_FRAME);
}
```

### 单帧执行逻辑

在 `ApplicationEditor::Update()` 中：
- `FRAME_BY_FRAME` 模式与 `PLAY` 模式一样会执行 `scene->Update()`
- 执行一帧后需要手动切换回 `PAUSE` 模式

## 停止播放 (StopPlaying)

### 完整流程

```cpp
// EditorActions.cpp:503-535
void EditorActions::StopPlaying()
{
    if (m_editorMode != EEditorMode::EDIT)
    {
        // 1. 切换到编辑模式
        SetEditorMode(EEditorMode::EDIT);

        // 2. 保存场景路径信息
        bool loadedFromDisk = ApplicationEditor::Instance()->sceneManager->IsCurrentSceneLoadedFromPath();
        std::string sceneSourcePath = ApplicationEditor::Instance()->sceneManager->GetCurrentSceneSourcePath();

        // 3. 保存当前选中的 GameObject ID
        int64_t focusedActorID = -1;
        if (auto targetActor = EDITOR_PANEL(Inspector, "Inspector").GetTargetActor())
            focusedActorID = targetActor->m_id;

        // 4. 从备份恢复场景 (关键！)
        ApplicationEditor::Instance()->sceneManager->LoadSceneFromMemory(m_sceneBackup);

        // 5. 恢复场景路径
        if (loadedFromDisk)
            ApplicationEditor::Instance()->sceneManager->StoreCurrentSceneSourcePath(sceneSourcePath);

        // 6. 清空备份
        m_sceneBackup.clear();

        // 7. 聚焦 SceneView
        EDITOR_PANEL(SceneView, "Scene View").Focus();

        // 8. 恢复选中的 GameObject
        if (auto actorInstance = ApplicationEditor::Instance()->sceneManager->GetCurrentScene()->Find(focusedActorID))
            EDITOR_PANEL(Inspector, "Inspector").FocusActor(actorInstance);

        // 9. 解析场景
        ApplicationEditor::Instance()->sceneManager->GetCurrentScene()->Resolve();

        // 10. 刷新 Hierarchy
        EDITOR_PANEL(Hierarchy, "Hierarchy").Refresh();

        // 11. 更新渲染路径
        ApplicationEditor::Instance()->m_rendererPath4SceneView->SetScene(
            ApplicationEditor::Instance()->sceneManager->GetCurrentScene());
        ApplicationEditor::Instance()->m_rendererPath4GameView->SetActive(false);
    }
}
```

### 场景恢复机制

```cpp
// SceneManager.cpp:351-366
bool SceneManager::LoadSceneFromMemory(std::string& p_doc)
{
    CreateEmptyScene();  // 创建空场景

    // 从 JSON 字符串反序列化
    if (!Serializer::DeserializeFromJson(p_doc, m_currScene))
    {
        UnloadCurrentScene();
        return false;
    }

    m_currScene->PostResourceLoaded();  // 后处理：重建层级关系
    SceneLoadEvent.Invoke();

    return true;
}
```

## Toolbar 按钮状态管理

### 按钮启用/禁用逻辑

```cpp
// Toolbar.cpp:52-89
EDITOR_EVENT(EditorModeChangedEvent) += [this](EditorActions::EEditorMode p_newMode)
{
    const auto enable = [](LitchiRuntime::Button* p_button, bool p_enable)
    {
        p_button->disabled = !p_enable;
        p_button->textColor = p_enable
            ? LitchiRuntime::Color{ 1.0f, 1.0f, 1.0f, 1.0f }
            : LitchiRuntime::Color{ 1.0f, 1.0f, 1.0f, 0.15f };
    };

    switch (p_newMode)
    {
    case EditorActions::EEditorMode::EDIT:
        enable(m_playButton, true);   // 可用
        enable(m_pauseButton, false); // 禁用
        enable(m_stopButton, false);  // 禁用
        enable(m_nextButton, false);  // 禁用
        break;
    case EditorActions::EEditorMode::PLAY:
        enable(m_playButton, false);  // 禁用
        enable(m_pauseButton, true);  // 可用
        enable(m_stopButton, true);   // 可用
        enable(m_nextButton, true);   // 可用
        break;
    case EditorActions::EEditorMode::PAUSE:
        enable(m_playButton, true);   // 可用（恢复播放）
        enable(m_pauseButton, false); // 禁用
        enable(m_stopButton, true);   // 可用
        enable(m_nextButton, true);   // 可用
        break;
    case EditorActions::EEditorMode::FRAME_BY_FRAME:
        enable(m_playButton, true);   // 可用
        enable(m_pauseButton, false); // 禁用
        enable(m_stopButton, true);   // 可用
        enable(m_nextButton, true);   // 可用
        break;
    }
};
```

## 渲染路径切换

### SceneView vs GameView

```cpp
// ApplicationEditor.cpp:440-467
void ApplicationEditor::SetupRendererPath()
{
    // SceneView 渲染路径 - 始终激活
    m_rendererPath4SceneView = new RendererPath(RendererPathType_SceneView);
    Renderer::UpdateRendererPath(RendererPathType_SceneView, m_rendererPath4SceneView);
    m_rendererPath4SceneView->SetActive(true);

    // GameView 渲染路径 - 仅播放时激活
    m_rendererPath4GameView = new RendererPath(RendererPathType_GameView);
    Renderer::UpdateRendererPath(RendererPathType_GameView, m_rendererPath4GameView);
    // 默认不激活

    // AssetView 渲染路径
    m_rendererPath4AssetView = new RendererPath(RendererPathType_AssetView);
    Renderer::UpdateRendererPath(RendererPathType_AssetView, m_rendererPath4AssetView);
}
```

### 播放时的渲染路径切换

```cpp
// StartPlaying() 中
ApplicationEditor::Instance()->m_rendererPath4GameView->SetScene(currScene);
ApplicationEditor::Instance()->m_rendererPath4GameView->SetActive(true);

// StopPlaying() 中
ApplicationEditor::Instance()->m_rendererPath4SceneView->SetScene(
    ApplicationEditor::Instance()->sceneManager->GetCurrentScene());
ApplicationEditor::Instance()->m_rendererPath4GameView->SetActive(false);
```

## 事件系统

### EditorActions 事件

```cpp
// EditorActions.h:380-384
LitchiRuntime::Event<LitchiRuntime::GameObject*> ActorSelectedEvent;
LitchiRuntime::Event<LitchiRuntime::GameObject*> ActorUnselectedEvent;
LitchiRuntime::Event<EEditorMode> EditorModeChangedEvent;  // 模式变化事件
LitchiRuntime::Event<> PlayEvent;                          // 播放事件
```

### EditorModeChangedEvent 触发

```cpp
// EditorActions.cpp:461-465
void EditorActions::SetEditorMode(EEditorMode p_newEditorMode)
{
    m_editorMode = p_newEditorMode;
    EditorModeChangedEvent.Invoke(m_editorMode);  // 通知所有监听者
}
```

### 监听者

- **Toolbar**: 更新按钮状态
- **其他面板**: 可能需要根据模式调整行为

## 运行时创建 GameObject

### CreateGameObject 在播放模式下的行为

```cpp
// SceneManager.cpp:36-68
GameObject* Scene::CreateGameObject(const std::string& name, bool isUI)
{
    int64_t id = m_availableID++;
    auto* game_object = new GameObject(name, id, m_isPlaying, this);

    // 添加默认 Transform
    if (!isUI)
        game_object->AddComponent<Transform>();
    else
        game_object->AddComponent<RectTransform>();

    m_gameObjectList.push_back(game_object);
    GameObject::CreatedEvent.Invoke(game_object);

    // 如果在播放模式下，立即调用生命周期方法
    if (m_isPlaying)
    {
        game_object->SetSleeping(false);
        if (game_object->GetActive())
        {
            game_object->OnAwake();
            game_object->OnEnable();
            game_object->OnStart();
        }
    }

    m_resolve = true;
    return game_object;
}
```

## 设计模式总结

| 模式 | 应用位置 |
|------|----------|
| 状态模式 | EEditorMode 状态切换 |
| 观察者模式 | EditorModeChangedEvent, PlayEvent |
| 备忘录模式 | m_sceneBackup 场景备份恢复 |
| 命令模式 | DelayAction 延迟执行 |

## 关键文件路径

| 文件 | 职责 |
|------|------|
| `Editor/Source/Core/EditorActions.h/cpp` | 编辑器操作控制，状态管理 |
| `Editor/Source/Panels/Toolbar.cpp` | 播放控制按钮 UI |
| `Editor/Source/ApplicationEditor.cpp` | 主循环，模式判断 |
| `Runtime/Function/Scene/SceneManager.h/cpp` | 场景管理，Play() 方法 |
| `Runtime/Function/Framework/GameObject/GameObject.cpp` | 生命周期方法 |

## 总结

LitchiEngine 编辑器的运行模式状态机设计清晰：

1. **状态分离**：EDIT/PLAY/PAUSE/FRAME_BY_FRAME 四种状态明确分离
2. **场景备份**：进入播放前备份场景，停止时恢复，保证编辑数据不丢失
3. **生命周期完整**：组件生命周期 OnAwake → OnEnable → OnStart → OnUpdate → OnDisable → OnDestroy 完整实现
4. **渲染路径切换**：SceneView 和 GameView 独立渲染路径，播放时正确切换
5. **事件驱动**：通过 EditorModeChangedEvent 通知状态变化，解耦各模块
