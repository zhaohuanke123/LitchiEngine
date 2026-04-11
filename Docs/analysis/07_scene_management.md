# 场景管理系统分析

## 1. 模块概述

场景管理系统位于 `Engine/Source/Runtime/Function/Scene/`，负责场景的加载、卸载、保存和运行时管理。预制体系统位于 `Engine/Source/Runtime/Function/Prefab/`，提供可复用的游戏对象模板。

## 2. 类设计

### 2.1 类关系

```
SceneManager (场景管理器)
    │
    └── Scene (场景实例)
            │
            └── GameObject* (游戏对象列表)

Prefab (预制体)
    │
    └── GameObject* (游戏对象模板)
```

## 3. Scene - 场景类

### 3.1 核心职责

- 管理场景内的所有 GameObject
- 处理 GameObject 的创建和销毁
- 驱动 GameObject 的生命周期
- 支持预制体实例化

### 3.2 类定义

```cpp
class Scene : public ScriptObject {
public:
    Scene();
    Scene(std::string name);
    ~Scene();

    // 播放控制
    void Play();
    bool IsPlaying() const { return m_isPlaying; }

    // 更新
    void Update();
    void FixedUpdate();
    void LateUpdate();

    // GameObject 管理
    GameObject* CreateGameObject(const std::string& name, bool isUI = false);
    void RemoveGameObject(GameObject* go);

    // 预制体实例化
    GameObject* InstantiatePrefab(Prefab* prefab, GameObject* root);

    // 遍历
    void Foreach(std::function<void(GameObject* game_object)> func);

    // 名称
    std::string GetName() { return m_name; }
    void SetName(std::string name) { m_name = name; }

    // 解析标记
    void Resolve() { m_resolve = true; }
    bool IsNeedResolve() { return m_resolve; }
    void ResetResolve();

    // 查找
    GameObject* Find(const char* name);
    GameObject* Find(const int64_t id);
    GameObject* FindByUnmanagedId(const int64_t unmanagedId);

    // 获取列表
    std::vector<GameObject*> GetRootGameObjectList();
    std::vector<GameObject*>& GetAllGameObjectList() {
        return m_gameObjectList;
    }

    // 编辑器更新
    void OnEditorUpdate();

    // 事件
    static Event<GameObject*> InstantiatePrefabEvent;

    // 公开成员
    std::vector<GameObject*> m_gameObjectList;
    int64_t m_availableID = 1;

    void PostResourceLoaded() override;

private:
    bool m_isPlaying = false;
    std::string m_name;
    bool m_resolve = false;
};
```

### 3.3 GameObject 创建

```cpp
GameObject* Scene::CreateGameObject(const std::string& name, bool isUI) {
    // 1. 分配唯一 ID
    int64_t id = m_availableID++;

    // 2. 创建 GameObject
    auto* game_object = new GameObject(name, id, m_isPlaying, this);

    // 3. 添加默认 Transform
    if (!isUI) {
        game_object->AddComponent<Transform>();
    } else {
        game_object->AddComponent<RectTransform>();
    }

    // 4. 添加到列表
    m_gameObjectList.push_back(game_object);

    // 5. 触发创建事件
    GameObject::CreatedEvent.Invoke(game_object);

    // 6. 如果在播放模式，立即调用生命周期
    if (m_isPlaying) {
        game_object->SetSleeping(false);
        if (game_object->GetActive()) {
            game_object->OnAwake();
            game_object->OnEnable();
            game_object->OnStart();
        }
    }

    m_resolve = true;
    return game_object;
}
```

### 3.4 GameObject 移除

```cpp
void Scene::RemoveGameObject(GameObject* go) {
    // 1. 收集要移除的对象（包括子对象）
    std::vector<Transform*> entities_to_remove;
    auto tran = go->GetComponent<Transform>();
    entities_to_remove.push_back(tran);
    tran->GetDescendants(&entities_to_remove);

    // 2. 创建 ID 集合
    std::set<uint64_t> ids_to_remove;
    for (Transform* transform : entities_to_remove) {
        ids_to_remove.insert(transform->GetGameObject()->GetObjectId());
    }

    // 3. 从列表中移除
    m_gameObjectList.erase(
        std::remove_if(m_gameObjectList.begin(), m_gameObjectList.end(),
            [&](GameObject* entity) {
                return ids_to_remove.count(entity->GetObjectId()) > 0;
            }),
        m_gameObjectList.end());

    // 4. 更新父对象
    if (Transform* parent = tran->GetParent()) {
        parent->AcquireChildren();
    }

    // 5. 删除对象
    for (Transform* transform : entities_to_remove) {
        delete transform->GetGameObject();
    }

    m_resolve = true;
}
```

### 3.5 预制体实例化

```cpp
GameObject* Scene::InstantiatePrefab(Prefab* prefab, GameObject* root) {
    // 1. 深拷贝预制体
    auto prefab_data = AssetManager::Serialize(prefab);
    auto deep_copy_prefab = new Prefab();
    AssetManager::Deserialize(prefab_data, deep_copy_prefab);
    deep_copy_prefab->PostResourceLoaded();

    // 2. 获取根实体
    auto rootObject = deep_copy_prefab->GetRootEntity();
    if (root != nullptr) {
        rootObject->SetParent(root);
    }

    // 3. DFS 遍历添加到场景
    std::stack<GameObject*> stack;
    stack.push(rootObject);

    while (stack.size() > 0) {
        auto current = stack.top();
        if (current == nullptr) break;
        stack.pop();

        // 分配新 ID
        auto newId = m_availableID++;
        current->m_id = newId;
        current->SetScene(this);
        m_gameObjectList.push_back(current);

        // 播放模式下调用生命周期
        if (m_isPlaying) {
            current->SetSleeping(false);
            if (current->GetActive()) {
                current->OnAwake();
                current->OnEnable();
                current->OnStart();
            }
        }

        // 添加子对象到栈
        auto childs = current->GetChildren();
        for (auto data : childs) {
            data->m_parentId = newId;
            stack.push(data);
        }
    }

    // 4. 清理临时预制体
    deep_copy_prefab->OnlyClearOnDeepCopy();
    delete deep_copy_prefab;

    m_resolve = true;
    InstantiatePrefabEvent.Invoke(rootObject);

    return rootObject;
}
```

### 3.6 场景播放

```cpp
void Scene::Play() {
    m_isPlaying = true;

    // 1. 唤醒所有 GameObject
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            p_element->SetSleeping(false);
        });

    // 2. 调用 OnAwake
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->ForeachComponent([](Component* comp) {
                    comp->OnAwake();
                });
        });

    // 3. 调用 OnEnable
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->ForeachComponent([](Component* comp) {
                    comp->OnEnable();
                });
        });

    // 4. 调用 OnStart
    std::for_each(m_gameObjectList.begin(), m_gameObjectList.end(),
        [](GameObject* p_element) {
            if (p_element->GetActive())
                p_element->OnStart();
        });
}
```

## 4. SceneManager - 场景管理器

### 4.1 核心职责

- 管理当前场景
- 加载/卸载场景
- 保存场景
- 提供场景事件

### 4.2 类定义

```cpp
class SceneManager {
public:
    SceneManager();
    ~SceneManager();

    // 场景操作
    void LoadEmptyScene();
    void CreateEmptyScene();
    bool LoadScene(const std::string& path);
    bool LoadSceneFromMemory(std::string& p_doc);
    void UnloadCurrentScene();
    bool HasCurrentScene() const;

    // 保存
    void SaveCurrentScene(const std::string& path);

    // 当前场景
    Scene* GetCurrentScene() { return m_currScene; }

    // 路径管理
    std::string GetCurrentSceneSourcePath() const {
        return m_currentSceneSourcePath;
    }
    bool IsCurrentSceneLoadedFromPath() {
        return m_currentSceneLoadedFromPath;
    }
    void StoreCurrentSceneSourcePath(const std::string& path);
    void ForgetCurrentSceneSourcePath();

    // 遍历
    void Foreach(std::function<void(GameObject* game_object)> func);

    // 事件
    Event<> SceneLoadEvent;
    Event<> SceneUnloadEvent;
    Event<const std::string&> CurrentSceneSourcePathChangedEvent;

private:
    Scene* m_currScene{ nullptr };
    bool m_currentSceneLoadedFromPath{ false };
    std::string m_currentSceneSourcePath{};
};
```

### 4.3 场景加载

```cpp
bool SceneManager::LoadScene(const std::string& path) {
    // 1. 创建空场景
    CreateEmptyScene();

    // 2. 从文件加载
    if (!AssetManager::LoadAsset(
            ApplicationBase::Instance()->configManager->GetAssetFolderFullPath() + path,
            m_currScene)) {
        UnloadCurrentScene();
        return false;
    }

    // 3. 存储路径
    StoreCurrentSceneSourcePath(path);

    // 4. 资源加载后处理
    m_currScene->PostResourceLoaded();

    // 5. 触发事件
    SceneLoadEvent.Invoke();

    return true;
}
```

### 4.4 场景卸载

```cpp
void SceneManager::UnloadCurrentScene() {
    if (m_currScene) {
        delete m_currScene;
        m_currScene = nullptr;
        SceneUnloadEvent.Invoke();
    }

    ForgetCurrentSceneSourcePath();
}
```

### 4.5 场景保存

```cpp
void SceneManager::SaveCurrentScene(const std::string& path) {
    AssetManager::SaveAsset<Scene>(
        *m_currScene,
        ApplicationBase::Instance()->configManager->GetAssetFolderFullPath() + path);

    StoreCurrentSceneSourcePath(path);
}
```

## 5. Prefab - 预制体

### 5.1 核心职责

- 作为可复用的 GameObject 模板
- 支持保存和加载
- 提供实例化接口

### 5.2 类定义

```cpp
class Prefab : public ScriptObject, public IResource {
public:
    Prefab();
    Prefab(const std::string& path);
    ~Prefab();

    // 文件操作
    bool LoadFromFile(const std::string& string);
    bool SaveToFile(const std::string& string);

    // 名称
    std::string GetName() { return m_name; }
    void SetName(std::string name) { m_name = name; }

    // 深拷贝清理
    void OnlyClearOnDeepCopy();

    // GameObject 管理
    GameObject* CreateGameObject(const std::string& name, bool isUI = false);
    void RemoveGameObject(GameObject* go);

    // 查找
    GameObject* Find(const char* name);
    GameObject* Find(const int64_t id);
    GameObject* FindByUnmanagedId(const int64_t unmanagedId);

    // 获取列表
    std::vector<GameObject*> GetRootGameObjectList();
    std::vector<GameObject*>& GetAllGameObjectList() {
        return m_gameObjectList;
    }

    // 根实体
    void SetRootEntity(GameObject* entity);
    GameObject* GetRootEntity() { return m_root_entity; }

    // 公开成员
    std::vector<GameObject*> m_gameObjectList;
    int64_t m_availableID = 1;
    int64_t m_root_entity_id{ -1 };

    void PostResourceLoaded() override;

private:
    GameObject* m_root_entity{ nullptr };
    std::string m_name;
};
```

## 6. 序列化机制

### 6.1 场景序列化

场景通过 AssetManager 进行序列化：

```cpp
// 保存
AssetManager::SaveAsset<Scene>(*m_currScene, path);

// 加载
AssetManager::LoadAsset(path, m_currScene);
```

### 6.2 序列化内容

场景序列化包含：

- 场景名称
- 所有 GameObject 列表
- 每个 GameObject 的：
  - ID 和父 ID
  - 名称和激活状态
  - 组件列表及组件属性

### 6.3 资源加载后处理

```cpp
void Scene::PostResourceLoaded() {
    // 1. 设置所有 GameObject 的场景引用
    for (auto go : m_gameObjectList) {
        go->SetScene(this);
        go->PostResourceLoaded();
    }

    // 2. 重建层级关系
    for (auto go : m_gameObjectList) {
        auto* parentGO = Find(go->m_parentId);
        go->SetParent(parentGO);
    }
}
```

## 7. 事件系统

### 7.1 Scene 事件

```cpp
// 预制体实例化事件
static Event<GameObject*> InstantiatePrefabEvent;
```

### 7.2 SceneManager 事件

```cpp
// 场景加载事件
Event<> SceneLoadEvent;

// 场景卸载事件
Event<> SceneUnloadEvent;

// 场景路径变更事件
Event<const std::string&> CurrentSceneSourcePathChangedEvent;
```

### 7.3 事件使用

```cpp
// 订阅场景加载事件
sceneManager.SceneLoadEvent += []() {
    DEBUG_LOG_INFO("Scene loaded");
};

// 订阅预制体实例化事件
Scene::InstantiatePrefabEvent += [](GameObject* go) {
    DEBUG_LOG_INFO("Prefab instantiated: {}", go->GetName());
};
```

## 8. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | SceneManager | 全局场景管理 |
| 工厂模式 | Scene::CreateGameObject | 创建 GameObject |
| 原型模式 | Prefab | 克隆预制体实例 |
| 观察者模式 | Event 系统 | 场景事件通知 |
| 迭代器模式 | Foreach | 遍历 GameObject |

## 9. 使用示例

### 9.1 创建场景

```cpp
// 创建空场景
sceneManager.CreateEmptyScene();

// 创建 GameObject
auto* go = sceneManager.GetCurrentScene()->CreateGameObject("Player");

// 添加组件
go->AddComponent<MeshRenderer>();
go->AddComponent<ScriptComponent>();
```

### 9.2 加载场景

```cpp
// 从文件加载
sceneManager.LoadScene("Scenes/MainLevel.scene");

// 获取场景中的对象
auto* player = sceneManager.GetCurrentScene()->Find("Player");
```

### 9.3 实例化预制体

```cpp
// 加载预制体
Prefab* prefab = AssetManager::LoadAsset<Prefab>("Prefabs/Enemy.prefab");

// 实例化到场景
auto* enemy = sceneManager.GetCurrentScene()->InstantiatePrefab(prefab, nullptr);
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
