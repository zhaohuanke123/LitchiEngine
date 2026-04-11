# 资源管理系统分析

## 1. 模块概述

资源管理系统位于 `Engine/Source/Runtime/Resource/`，负责各类资源的加载、缓存和生命周期管理。采用模板基类设计，支持多种资源类型。

## 2. 架构设计

### 2.1 类关系

```
AResourceManager<T> (模板基类)
    ├── TextureManager (纹理管理器)
    ├── MaterialManager (材质管理器)
    ├── ModelManager (模型管理器)
    ├── ShaderManager (着色器管理器)
    ├── PrefabManager (预制体管理器)
    └── FontManager (字体管理器)

IResource (资源接口)
    ├── Material (材质)
    ├── Mesh (网格)
    ├── Prefab (预制体)
    └── ...

AssetManager (资产序列化)
    └── Serializer (序列化器)
```

## 3. IResource - 资源接口

### 3.1 核心职责

- 定义资源基类
- 管理资源路径
- 提供资源类型信息

### 3.2 类定义

```cpp
enum class ResourceType {
    Texture,
    Texture2d,
    Texture2dArray,
    TextureCube,
    Audio,
    Material,
    Mesh,
    Cubemap,
    Animation,
    Font,
    Shader,
    Prefab,
    Unknown,
};

class IResource : public Object {
public:
    IResource(ResourceType type);
    virtual ~IResource() = default;

    // 路径设置
    void SetResourceFilePath(const std::string& obsoultePath);

    // 路径获取
    ResourceType GetResourceType() const { return m_resource_type; }
    const char* GetResourceTypeCstr() const { return typeid(*this).name(); }
    bool HasFilePathNative() const { return !m_resource_file_path_native.empty(); }
    const std::string& GetResourceFilePath() const { return m_resource_file_path_foreign; }
    const std::string& GetResourceFilePathNative() const { return m_resource_file_path_native; }
    const std::string& GetResourceDirectory() const { return m_resource_directory; }
    const std::string& GetResourceFilePathAsset() const { return m_resource_file_path_asset; }

    // 标志
    void SetFlag(const uint32_t flag, bool enabled = true);
    uint32_t GetFlags() const { return m_flags; }
    void SetFlags(const uint32_t flags) { m_flags = flags; }

    // 状态
    bool IsReadyForUse() const { return m_is_ready_for_use; }

    // IO
    virtual bool SaveToFile(const std::string& file_path) { return true; }
    virtual bool LoadFromFile(const std::string& file_path) { return true; }

    // 类型转换
    template <typename T>
    static constexpr ResourceType TypeToEnum();

protected:
    ResourceType m_resource_type = ResourceType::Unknown;
    std::atomic<bool> m_is_ready_for_use = false;
    uint32_t m_flags = 0;

private:
    std::string m_resource_directory;
    std::string m_resource_file_path_native;
    std::string m_resource_file_path_foreign;
    std::string m_resource_file_path_asset;
};
```

### 3.3 路径管理

```cpp
void IResource::SetResourceFilePath(const std::string& obsoultePath) {
    // 检查是否为引擎原生文件
    const bool is_native_file = FileSystem::IsEngineMaterialFile(obsoultePath) ||
                                 FileSystem::IsEngineModelFile(obsoultePath) ||
                                 FileSystem::IsEnginePrefabFile(obsoultePath);

    // 验证文件存在
    if (!is_native_file) {
        if (!FileSystem::IsFile(obsoultePath)) {
            DEBUG_LOG_INFO("{} is not a valid file path", obsoultePath.c_str());
        }
    }

    // 设置路径
    m_resource_file_path_foreign = obsoultePath;
    m_resource_file_path_native = obsoultePath;
    m_resource_file_path_asset = FileSystem::GetRelativePathAssetFromNative(obsoultePath);

    // 设置名称和目录
    m_object_name = FileSystem::GetFileNameWithoutExtensionFromFilePath(obsoultePath);
    m_resource_directory = FileSystem::GetDirectoryFromFilePath(obsoultePath);
}
```

## 4. AResourceManager - 资源管理器模板基类

### 4.1 核心职责

- 提供资源加载、卸载、重载接口
- 管理资源缓存
- 处理路径转换

### 4.2 类定义

```cpp
template<typename T>
class AResourceManager {
public:
    // 资源操作
    T* LoadResource(const std::string& p_path);
    void UnloadResource(const std::string& p_path);
    bool MoveResource(const std::string& p_previousPath, const std::string& p_newPath);
    void ReloadResource(const std::string& p_path);
    bool IsResourceRegistered(const std::string& p_path);
    void UnloadResources();

    // 注册/注销
    T* RegisterResource(const std::string& p_path, T* p_instance);
    void UnregisterResource(const std::string& p_path);

    // 获取资源
    T* GetResource(const std::string& p_path, bool p_tryToLoadIfNotFound = true);
    T* operator[](const std::string& p_path);

    // 获取资源映射
    std::unordered_map<std::string, T*>& GetResources();

protected:
    // 子类实现
    virtual T* CreateResource(const std::string& p_path) = 0;
    virtual void DestroyResource(T* p_resource) = 0;
    virtual void ReloadResource(T* p_resource, const std::string& p_path) = 0;

    // 路径转换
    std::string GetRealPath(const std::string& p_path) const;
    std::string GetRelativePath(const std::string& p_path) const;

private:
    std::unordered_map<std::string, T*> m_resources;
};
```

### 4.3 资源加载

```cpp
template<typename T>
inline T* AResourceManager<T>::LoadResource(const std::string& p_path) {
    // 1. 转换为相对路径
    std::string resPath = GetRelativePath(p_path);

    // 2. 检查缓存
    if (auto resource = GetResource(resPath, false); resource) {
        return resource;
    }

    // 3. 创建资源
    auto newResource = CreateResource(resPath);
    if (newResource) {
        return RegisterResource(resPath, newResource);
    }

    return nullptr;
}
```

### 4.4 资源卸载

```cpp
template<typename T>
inline void AResourceManager<T>::UnloadResource(const std::string& p_path) {
    std::string resPath = GetRelativePath(p_path);

    if (auto resource = GetResource(resPath, false); resource) {
        auto tempPath = resPath;
        DestroyResource(resource);
        UnregisterResource(tempPath);
    }
}
```

### 4.5 资源注册

```cpp
template<typename T>
inline T* AResourceManager<T>::RegisterResource(const std::string& p_path, T* p_instance) {
    // 如果已存在，先销毁
    if (auto resource = GetResource(p_path, false); resource) {
        DestroyResource(resource);
    }

    DEBUG_LOG_INFO("RegisterResource path:{}", p_path);

    m_resources[p_path] = p_instance;
    return p_instance;
}
```

### 4.6 路径转换

```cpp
template<typename T>
inline std::string AResourceManager<T>::GetRealPath(const std::string& p_path) const {
    std::string result;

    if (FileSystem::IsFullPath(p_path)) {
        return p_path;
    }

    if (p_path[0] == ':') {
        // 引擎路径 (:Textures/default.png)
        result = FileSystem::GetEngineAssetDirectoryPath() +
                 std::string(p_path.data() + 1, p_path.data() + p_path.size());
    } else {
        // 项目路径 (Textures/my_texture.png)
        result = FileSystem::GetProjectAssetDirectoryPath() + p_path;
    }

    return result;
}
```

## 5. 具体资源管理器

### 5.1 TextureManager

```cpp
class TextureManager : public AResourceManager<RHI_Texture2D> {
public:
    virtual RHI_Texture2D* CreateResource(const std::string& p_path) override;
    virtual void DestroyResource(RHI_Texture2D* p_resource) override;
    virtual void ReloadResource(RHI_Texture2D* p_resource, const std::string& p_path) override;
};
```

### 5.2 MaterialManager

```cpp
class MaterialManager : public AResourceManager<Material> {
public:
    virtual Material* CreateResource(const std::string& p_path) override;
    virtual void DestroyResource(Material* p_resource) override;
    virtual void ReloadResource(Material* p_resource, const std::string& p_path) override;

    // 创建新材质
    Material* CreateMaterial(const std::string& p_path);
};
```

### 5.3 ModelManager

```cpp
class ModelManager : public AResourceManager<Model> {
public:
    virtual Model* CreateResource(const std::string& p_path) override;
    virtual void DestroyResource(Model* p_resource) override;
    virtual void ReloadResource(Model* p_resource, const std::string& p_path) override;
};
```

## 6. AssetManager - 资产序列化

### 6.1 核心职责

- 提供资产加载/保存接口
- 封装序列化操作

### 6.2 类定义

```cpp
class AssetManager {
public:
    // 加载资产
    template<typename AssetType>
    static bool LoadAsset(const std::string& asset_url, AssetType& out_asset) {
        const std::filesystem::path asset_path = asset_url;
        std::ifstream asset_json_file(asset_path);
        if (!asset_json_file) {
            DEBUG_LOG_ERROR("open file: {} failed!", asset_path.generic_string());
            return false;
        }

        std::stringstream buffer;
        buffer << asset_json_file.rdbuf();
        std::string asset_json_text(buffer.str());

        return Serializer::DeserializeFromJson(asset_json_text, out_asset);
    }

    // 保存资产
    template<typename AssetType>
    static bool SaveAsset(const AssetType& out_asset, const std::string& asset_url) {
        std::ofstream asset_json_file(asset_url);
        if (!asset_json_file) {
            if (!std::filesystem::create_directory(asset_url)) {
                DEBUG_LOG_ERROR("open file {} failed!", asset_url);
                return false;
            }
        }

        auto asset_json_text = Serializer::SerializeToJson(out_asset);

        asset_json_file << asset_json_text;
        asset_json_file.flush();
        asset_json_file.close();

        return true;
    }

    // 序列化/反序列化
    template<typename AssetType>
    static bool Deserialize(const std::string& data, AssetType& out_asset) {
        return Serializer::DeserializeFromJson(data, out_asset);
    }

    template<typename AssetType>
    static std::string Serialize(const AssetType& out_asset) {
        auto asset_json_text = Serializer::SerializeToJson(out_asset);
        return asset_json_text;
    }
};
```

## 7. 资源类型

### 7.1 资源类型枚举

```cpp
enum class ResourceType {
    Texture,        // 纹理
    Texture2d,      // 2D 纹理
    Texture2dArray, // 2D 纹理数组
    TextureCube,    // 立方体纹理
    Audio,          // 音频
    Material,       // 材质
    Mesh,           // 网格
    Cubemap,        // 立方体贴图
    Animation,      // 动画
    Font,           // 字体
    Shader,         // 着色器
    Prefab,         // 预制体
    Unknown,        // 未知
};
```

### 7.2 资源管理器对应关系

| 资源类型 | 管理器类 | 资源类 |
|----------|----------|--------|
| Texture | TextureManager | RHI_Texture2D |
| Material | MaterialManager | Material |
| Mesh | ModelManager | Model |
| Shader | ShaderManager | RHI_Shader |
| Prefab | PrefabManager | Prefab |
| Font | FontManager | Font |

## 8. 资源路径约定

### 8.1 路径格式

```cpp
// 引擎资源路径 (以 : 开头)
":Textures/default.png"
":Shaders/PBRTest.hlsl"

// 项目资源路径 (相对路径)
"Textures/my_texture.png"
"Materials/my_material.mat"
"Models/my_model.model"
```

### 8.2 路径转换

```cpp
// 相对路径 → 绝对路径
std::string GetRealPath(const std::string& p_path) {
    if (p_path[0] == ':') {
        // 引擎路径
        return FileSystem::GetEngineAssetDirectoryPath() + (p_path + 1);
    } else {
        // 项目路径
        return FileSystem::GetProjectAssetDirectoryPath() + p_path;
    }
}
```

## 9. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 模板方法模式 | AResourceManager | 统一资源管理流程 |
| 工厂方法模式 | CreateResource | 子类实现资源创建 |
| 缓存模式 | m_resources | 避免重复加载 |
| 单例模式 | 各 Manager | 全局资源管理 |

## 10. 使用示例

### 10.1 加载纹理

```cpp
TextureManager textureManager;

// 加载纹理
auto* texture = textureManager.LoadResource("Textures/diffuse.png");

// 获取已加载的纹理
auto* cached = textureManager.GetResource("Textures/diffuse.png");

// 卸载纹理
textureManager.UnloadResource("Textures/diffuse.png");
```

### 10.2 加载材质

```cpp
MaterialManager materialManager;

// 加载材质
auto* material = materialManager.LoadResource("Materials/PBR.mat");

// 创建新材质
auto* newMaterial = materialManager.CreateMaterial("Materials/NewMaterial.mat");
```

### 10.3 加载场景

```cpp
// 通过 AssetManager 加载
Scene scene;
AssetManager::LoadAsset("Scenes/MainLevel.scene", scene);

// 保存场景
AssetManager::SaveAsset(scene, "Scenes/MainLevel.scene");
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
