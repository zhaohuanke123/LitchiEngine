# LitchiEngine 序列化系统分析

## 概述

LitchiEngine 的序列化系统是一个基于 **RTTR (Run-Time Type Reflection)** 反射框架和 **RapidJSON** 构建的通用序列化解决方案。该系统实现了运行时对象的 JSON 序列化/反序列化，支持场景、预制体、材质等多种资源的持久化存储。

### 核心特性

- **反射驱动**: 基于 RTTR 实现运行时类型反射，无需手写序列化代码
- **JSON 格式**: 使用 RapidJSON 生成可读性强的 JSON 文本
- **多态支持**: 支持多态类型的序列化，自动记录派生类类型信息
- **容器支持**: 支持 `std::vector`、`std::map` 等标准容器的序列化
- **元数据控制**: 通过 RTTR 元数据控制序列化行为（如 `NO_SERIALIZE`）

## 架构设计

### 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         应用层 (Application Layer)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ SceneManager │  │ AssetManager │  │ PrefabSystem │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                        │
│         └─────────────────┼─────────────────┘                        │
│                           ▼                                         │
├─────────────────────────────────────────────────────────────────────┤
│                      序列化层 (Serialization Layer)                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Serializer                                │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐    │  │
│  │  │  SerializeToJson()  │  │  DeserializeFromJson()      │    │  │
│  │  └──────────┬──────────┘  └──────────────┬──────────────┘    │  │
│  │             │                             │                    │  │
│  │             ▼                             ▼                    │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐    │  │
│  │  │  to_json_recursively│  │  fromjson_recursively       │    │  │
│  │  └─────────────────────┘  └─────────────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                       反射层 (Reflection Layer)                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │    TypeRegister  │  │    TypeManager   │  │      Object      │   │
│  │  (RTTR 注册)      │  │  (类型查询)       │  │   (基类定义)      │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                       基础层 (Foundation Layer)                      │
│  ┌──────────────────┐  ┌──────────────────┐                         │
│  │   RTTR Library   │  │   RapidJSON      │                         │
│  │  (运行时反射)     │  │  (JSON 解析)      │                         │
│  └──────────────────┘  └──────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 核心类关系图

```
┌────────────────────────────────────────────────────────────────────┐
│                           Serializer                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  + SerializeToJson(rttr::instance obj) -> std::string        │  │
│  │  + DeserializeFromJson(json, rttr::instance obj) -> bool     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬────────────────────────────────────┘
                                │ uses
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                          RTTR Type System                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  rttr::type         - 类型信息                                │  │
│  │  rttr::instance     - 对象实例包装                            │  │
│  │  rttr::variant      - 值包装                                  │  │
│  │  rttr::property     - 属性访问                                │  │
│  │  rttr::metadata     - 元数据                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## 核心类设计

### 1. Serializer 类

`Serializer` 是序列化系统的核心类，提供静态方法实现对象与 JSON 的双向转换。

**文件位置**: `Engine/Source/Runtime/Core/Meta/Serializer/serializer.h`

```cpp
class Serializer {
public:
    // 将反射对象序列化为 JSON 文本
    static std::string SerializeToJson(rttr::instance obj);

    // 从 JSON 文本反序列化到反射对象
    static bool DeserializeFromJson(const std::string& json, rttr::instance obj);
};
```

#### 序列化流程 (SerializeToJson)

```
SerializeToJson(obj)
       │
       ▼
┌──────────────────────┐
│ 验证对象有效性        │
│ obj.is_valid()       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 创建 RapidJSON       │
│ StringBuffer/Writer  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ to_json_recursively  │◄─────────────────────┐
│ 递归遍历对象属性      │                      │
└──────────┬───────────┘                      │
           │                                   │
           ▼                                   │
    ┌──────────────┐                          │
    │ 检查类型      │                          │
    └──────┬───────┘                          │
           │                                   │
    ┌──────┴───────┬──────────┬──────────┐    │
    ▼              ▼          ▼          ▼    │
┌────────┐  ┌──────────┐ ┌────────┐ ┌────────┴───┐
│原子类型│  │ 枚举类型  │ │ 字符串 │ │ 复合对象   │
│写入JSON│  │ 写入名称  │ │ 写入值 │ │递归调用    │
└────────┘  └──────────┘ └────────┘ └────────────┘
           │
           ▼
┌──────────────────────┐
│ 返回 JSON 字符串      │
└──────────────────────┘
```

#### 反序列化流程 (DeserializeFromJson)

```
DeserializeFromJson(json, obj)
       │
       ▼
┌──────────────────────┐
│ RapidJSON 解析文本    │
│ document.Parse()     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 检查解析错误          │
│ HasParseError()      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ fromjson_recursively │◄─────────────────────┐
│ 递归填充对象属性      │                      │
└──────────┬───────────┘                      │
           │                                   │
           ▼                                   │
    ┌──────────────┐                          │
    │ 遍历属性列表  │                          │
    └──────┬───────┘                          │
           │                                   │
    ┌──────┴───────┬──────────┬──────────┐    │
    ▼              ▼          ▼          ▼    │
┌────────┐  ┌──────────┐ ┌────────┐ ┌────────┴───┐
│数组类型│  │ 对象类型  │ │基础类型│ │ 多态类型   │
│处理数组│  │递归处理  │ │类型转换│ │创建派生类  │
└────────┘  └──────────┘ └────────┘ └────────────┘
           │
           ▼
┌──────────────────────┐
│ 返回成功/失败         │
└──────────────────────┘
```

### 2. 类型系统支持

#### Object 基类

所有可序列化对象的基类：

```cpp
class Object {
public:
    std::string& GetObjectName();
    void SetObjectName(std::string& name);
    uint64_t GetObjectId() const;
    void SetObjectId(uint64_t id);

    // 资源加载后的回调
    virtual void PostResourceLoaded() {}
    virtual void PostResourceModify() {}

    RTTR_ENABLE()  // 启用 RTTR 反射
protected:
    std::string m_object_name;
    uint64_t m_object_id = 0;
};
```

#### 继承层次

```
Object
  │
  ├── ScriptObject (脚本对象基类)
  │     │
  │     ├── GameObject (游戏对象)
  │     ├── Component (组件基类)
  │     │     ├── Transform
  │     │     ├── MeshRenderer
  │     │     ├── Camera
  │     │     ├── Light
  │     │     └── ... (其他组件)
  │     └── Scene (场景)
  │
  └── IResource (资源基类)
        ├── Material (材质)
        ├── Prefab (预制体)
        └── ... (其他资源)
```

## RTTR 反射与序列化集成

### RTTR 注册机制

类型注册在 `TypeRegister.h` 中集中管理：

```cpp
RTTR_REGISTRATION {
    // 基础类型
    registration::class_<Vector3>("Vec3")
        .constructor()
        .property("x", &Vector3::x)
        .property("y", &Vector3::y)
        .property("z", &Vector3::z);

    // 多态组件类型
    registration::class_<Component>("Component")
        (rttr::metadata("Serializable", true),
         rttr::metadata("Polymorphic", true))
        .constructor<>()(rttr::policy::ctor::as_raw_ptr);

    // GameObject
    registration::class_<GameObject>("GameObject")
        (rttr::metadata("Polymorphic", true))
        .constructor<>()(rttr::policy::ctor::as_raw_ptr)
        .property("id", &GameObject::m_id)
        .property("parentId", &GameObject::m_parentId)
        .property("layer", &GameObject::GetLayer, &GameObject::SetLayer)
        .property("name", &GameObject::GetName, &GameObject::SetName)
        .property("componentList", &GameObject::m_componentList);
}
```

### 元数据标签

| 标签 | 用途 | 示例 |
|------|------|------|
| `Polymorphic` | 标记多态类型，序列化时记录派生类类型信息 | `rttr::metadata("Polymorphic", true)` |
| `Serializable` | 标记可序列化类型 | `rttr::metadata("Serializable", true)` |
| `NO_SERIALIZE` | 标记属性不参与序列化 | `rttr::metadata("NO_SERIALIZE", true)` |
| `AssetPath` | 标记属性为资源路径 | `rttr::metadata("AssetPath", true)` |
| `QuatToEuler` | 四元数转欧拉角显示 | `rttr::metadata("QuatToEuler", true)` |

### 多态序列化机制

序列化多态类型时，会自动记录实际类型信息：

```json
{
    "Type": "MeshRenderer",
    "materialPath": "Materials/default.lmat"
}
```

**实现原理**:

```cpp
// 序列化时
if (objType.get_metadata("Polymorphic").to_bool() == true) {
    writer.String("Type");
    auto objDerivedType = obj.get_derived_type();
    std::string typeName = objDerivedType.get_name().data();
    writer.String(typeName);
}

// 反序列化时
if (rawType2.get_metadata("Polymorphic")) {
    auto realTypeName = json_object["Type"].GetString();
    rawType2 = rttr::type::get_by_name(realTypeName);
    wrapped_var = rawType2.create();
}
```

## 场景序列化

### Scene 类序列化

**Scene 类定义**:

```cpp
class Scene : public ScriptObject {
public:
    std::vector<GameObject*> m_gameObjectList{};
    int64_t m_availableID = 1;

    RTTR_ENABLE()
};
```

**RTTR 注册**:

```cpp
registration::class_<Scene>("Scene")
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("name", &Scene::GetName, &Scene::SetName)
    .property("availableID", &Scene::m_availableID)
    .property("gameObjects", &Scene::m_gameObjectList);
```

**JSON 输出示例**:

```json
{
    "name": "MainScene",
    "availableID": 5,
    "gameObjects": [
        {
            "Type": "GameObject",
            "id": 1,
            "parentId": 0,
            "layer": 0,
            "name": "MainCamera",
            "componentList": [
                {
                    "Type": "Transform",
                    "localPosition": { "x": 0.0, "y": 1.0, "z": -10.0 },
                    "localRotation": { "x": 0.0, "y": 0.0, "z": 0.0, "w": 1.0 },
                    "localScale": { "x": 1.0, "y": 1.0, "z": 1.0 }
                },
                {
                    "Type": "Camera",
                    "clearFlags": "SolidColor",
                    "projectionType": "Perspective",
                    "nearPlane": 0.1,
                    "farPlane": 1000.0
                }
            ]
        }
    ]
}
```

### GameObject 序列化

**GameObject 类定义**:

```cpp
class GameObject : public ScriptObject {
public:
    int64_t m_id{0};              // 场景内唯一 ID
    int64_t m_parentId{0};        // 父对象 ID (用于重建层级)
    std::vector<Component*> m_componentList;

    RTTR_ENABLE()
};
```

**序列化特点**:

1. **ID 引用**: 使用 `m_id` 和 `m_parentId` 记录对象关系，而非直接序列化指针
2. **组件列表**: 支持多态组件的序列化，自动记录组件实际类型
3. **延迟绑定**: 反序列化后通过 `PostResourceLoaded()` 重建对象引用关系

### 场景加载/保存流程

```
┌────────────────────────────────────────────────────────────────────┐
│                     场景加载流程                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  SceneManager::LoadScene(path)                                     │
│         │                                                          │
│         ▼                                                          │
│  ┌─────────────────────┐                                          │
│  │ 创建空场景           │                                          │
│  │ CreateEmptyScene()  │                                          │
│  └──────────┬──────────┘                                          │
│             │                                                      │
│             ▼                                                      │
│  ┌─────────────────────┐                                          │
│  │ AssetManager::      │                                          │
│  │ LoadAsset<Scene>()  │                                          │
│  └──────────┬──────────┘                                          │
│             │                                                      │
│             ▼                                                      │
│  ┌─────────────────────┐                                          │
│  │ Serializer::        │                                          │
│  │ DeserializeFromJson │                                          │
│  └──────────┬──────────┘                                          │
│             │                                                      │
│             ▼                                                      │
│  ┌─────────────────────┐                                          │
│  │ PostResourceLoaded  │                                          │
│  │ - 设置 Scene 引用    │                                          │
│  │ - 重建父对象关系     │                                          │
│  │ - 初始化组件         │                                          │
│  └─────────────────────┘                                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## 资源序列化

### AssetManager 模板类

`AssetManager` 提供统一的资源加载/保存接口：

```cpp
class AssetManager {
public:
    // 从文件加载资源
    template<typename AssetType>
    static bool LoadAsset(const std::string& asset_url, AssetType& out_asset) {
        std::ifstream asset_json_file(asset_url);
        std::stringstream buffer;
        buffer << asset_json_file.rdbuf();
        std::string asset_json_text(buffer.str());
        return Serializer::DeserializeFromJson(asset_json_text, out_asset);
    }

    // 保存资源到文件
    template<typename AssetType>
    static bool SaveAsset(const AssetType& out_asset, const std::string& asset_url) {
        std::ofstream asset_json_file(asset_url);
        auto asset_json_text = Serializer::SerializeToJson(out_asset);
        asset_json_file << asset_json_text;
        return true;
    }

    // 内存序列化
    template<typename AssetType>
    static bool Deserialize(const std::string& data, AssetType& out_asset);
    template<typename AssetType>
    static std::string Serialize(const AssetType& out_asset);
};
```

### Material 序列化

**MaterialRes 数据结构**:

```cpp
class UniformInfoBase {
    std::string name;
    RTTR_ENABLE()
};

class UniformInfoFloat : public UniformInfoBase {
    float value;
    RTTR_ENABLE(UniformInfoBase)
};

class MaterialRes {
    RHI_Vertex_Type vertexType;
    std::string shaderPath;
    MaterialResSetting materialSetting;
    std::vector<UniformInfoBase*> uniformInfoList;  // 多态数组

    RTTR_ENABLE()
};
```

**RTTR 注册**:

```cpp
// 多态 Uniform 基类
registration::class_<UniformInfoBase>("UniformInfoBase")
    (rttr::metadata("Serializable", true),
     rttr::metadata("Polymorphic", true))
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("name", &UniformInfoBase::name);

// 派生类
registration::class_<UniformInfoFloat>("UniformInfoFloat")
    (rttr::metadata("Serializable", true))
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("value", &UniformInfoFloat::value);

// Material 资源
registration::class_<MaterialRes>("MaterialRes")
    (rttr::metadata("Serializable", true))
    .constructor<>()
    .property("vertexType", &MaterialRes::vertexType)
    .property("shaderPath", &MaterialRes::shaderPath)
    .property("materialSetting", &MaterialRes::materialSetting)
    .property("uniformInfoList", &MaterialRes::uniformInfoList);
```

**JSON 输出示例**:

```json
{
    "vertexType": "PosUvNorTan",
    "shaderPath": "Shaders/PBR.shader",
    "materialSetting": {
        "isTransparent": false
    },
    "uniformInfoList": [
        {
            "Type": "UniformInfoFloat",
            "name": "metallic",
            "value": 0.5
        },
        {
            "Type": "UniformInfoVector3",
            "name": "albedo",
            "vector": { "x": 1.0, "y": 0.8, "z": 0.6 }
        },
        {
            "Type": "UniformInfoTexture",
            "name": "baseMap",
            "path": "Textures/base.png"
        }
    ]
}
```

### Prefab 序列化

**Prefab 类定义**:

```cpp
class Prefab : public ScriptObject, public IResource {
public:
    std::vector<GameObject*> m_gameObjectList;
    int64_t m_availableID = 1;
    int64_t m_root_entity_id{-1};  // 根实体 ID

    bool LoadFromFile(const std::string& path);
    bool SaveToFile(const std::string& path);

    RTTR_ENABLE()
};
```

**预制体实例化流程**:

```
Scene::InstantiatePrefab(prefab, parent)
       │
       ▼
┌──────────────────────┐
│ 深拷贝预制体          │
│ 通过序列化/反序列化   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ PostResourceLoaded   │
│ 初始化所有 GameObject │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 分配新 ID            │
│ 设置 Scene 引用       │
│ 添加到场景列表        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 重建父子关系          │
│ 设置根对象父节点      │
└──────────────────────┘
```

## 数据流图

### 序列化数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                        序列化数据流                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │   C++ 对象   │───▶│  RTTR 反射  │───▶│  Serializer │             │
│  │             │    │  instance   │    │             │             │
│  └─────────────┘    └─────────────┘    └──────┬──────┘             │
│                                               │                     │
│                                               ▼                     │
│                                        ┌─────────────┐              │
│                                        │ RapidJSON   │              │
│                                        │ PrettyWriter│              │
│                                        └──────┬──────┘              │
│                                               │                     │
│                                               ▼                     │
│                                        ┌─────────────┐              │
│                                        │ JSON String │              │
│                                        └──────┬──────┘              │
│                                               │                     │
│                                               ▼                     │
│                                        ┌─────────────┐              │
│                                        │   .json文件 │              │
│                                        └─────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 反序列化数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                       反序列化数据流                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │   .json文件 │───▶│ RapidJSON   │───▶│  Document   │             │
│  │             │    │  Parse      │    │             │             │
│  └─────────────┘    └─────────────┘    └──────┬──────┘             │
│                                               │                     │
│                                               ▼                     │
│                                        ┌─────────────┐              │
│                                        │ Serializer  │              │
│                                        │ Deserialize │              │
│                                        └──────┬──────┘              │
│                                               │                     │
│                                               ▼                     │
│                                        ┌─────────────┐              │
│                                        │  RTTR 反射  │              │
│                                        │ property    │              │
│                                        └──────┬──────┘              │
│                                               │                     │
│                                               ▼                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │PostResource │◀───│  C++ 对象    │◀───│  创建对象   │             │
│  │  Loaded()   │    │             │    │  (构造函数)  │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 使用示例

### 基本使用

```cpp
// 1. 序列化对象到文件
Scene scene;
// ... 填充场景数据 ...
AssetManager::SaveAsset(scene, "Scenes/main.lscene");

// 2. 从文件加载对象
Scene loadedScene;
AssetManager::LoadAsset("Scenes/main.lscene", loadedScene);
loadedScene.PostResourceLoaded();  // 重建引用关系

// 3. 内存序列化 (用于深拷贝等场景)
std::string json = AssetManager::Serialize(prefab);
Prefab* deepCopy = new Prefab();
AssetManager::Deserialize(json, deepCopy);
```

### 添加可序列化组件

```cpp
// 1. 定义组件类
class MyComponent : public Component {
public:
    float m_speed = 1.0f;
    std::string m_tag;

    RTTR_ENABLE(Component)
};

// 2. RTTR 注册 (在 TypeRegister.h 中)
RTTR_REGISTRATION {
    registration::class_<MyComponent>("MyComponent")
        .constructor<>()(rttr::policy::ctor::as_raw_ptr)
        .property("speed", &MyComponent::m_speed)
        .property("tag", &MyComponent::m_tag);
}

// 3. 实现可选的 PostResourceLoaded
void MyComponent::PostResourceLoaded() {
    Component::PostResourceLoaded();
    // 执行资源加载后的初始化逻辑
}
```

### 排除属性序列化

```cpp
RTTR_REGISTRATION {
    registration::class_<GameObject>("GameObject")
        .property("Id", &GameObject::GetObjectId, &GameObject::SetObjectId)
            (rttr::metadata("NO_SERIALIZE", true))  // 该属性不参与序列化
        .property("Name", &GameObject::GetObjectName, &GameObject::SetObjectName)
            (rttr::metadata("NO_SERIALIZE", true));
}
```

### 标记资源路径属性

```cpp
RTTR_REGISTRATION {
    registration::class_<MeshRenderer>("MeshRenderer")
        .property("materialPath", &MeshRenderer::GetMaterialPath, &MeshRenderer::SetMaterialPath)
            (rttr::metadata("AssetPath", true),
             rttr::metadata("AssetType", PathParser::EFileType::MATERIAL));
}
```

## 扩展指南

### 添加新的可序列化类型

1. **定义类并继承适当的基类**

```cpp
class MyResource : public IResource {
public:
    std::string m_data;
    int m_value;

    RTTR_ENABLE()  // 必须添加此宏
};
```

2. **在 TypeRegister.h 中注册**

```cpp
RTTR_REGISTRATION {
    registration::class_<MyResource>("MyResource")
        (rttr::metadata("Serializable", true))  // 标记可序列化
        .constructor<>()
        .property("data", &MyResource::m_data)
        .property("value", &MyResource::m_value);
}
```

3. **实现 IResource 接口方法** (可选)

```cpp
bool MyResource::LoadFromFile(const std::string& path) {
    return AssetManager::LoadAsset(path, *this);
}

bool MyResource::SaveToFile(const std::string& path) {
    return AssetManager::SaveAsset(*this, path);
}
```

### 添加多态类型支持

1. **标记基类为多态**

```cpp
registration::class_<BaseComponent>("BaseComponent")
    (rttr::metadata("Polymorphic", true),
     rttr::metadata("Serializable", true))
    .constructor<>()(rttr::policy::ctor::as_raw_ptr);
```

2. **派生类使用 RTTR_ENABLE 宏**

```cpp
class DerivedComponent : public BaseComponent {
    RTTR_ENABLE(BaseComponent)  // 指定基类
};
```

3. **注册派生类**

```cpp
registration::class_<DerivedComponent>("DerivedComponent")
    (rttr::metadata("Serializable", true))
    .constructor<>()(rttr::policy::ctor::as_raw_ptr)
    .property("customValue", &DerivedComponent::m_customValue);
```

### 自定义容器序列化

Serializer 已内置支持标准容器：

- `std::vector` - 序列化为 JSON 数组
- `std::map` - 序列化为键值对数组
- `std::unordered_map` - 同 `std::map`

对于自定义容器，需要提供 RTTR 迭代器支持或转换为标准容器。

## 关键文件路径

| 模块 | 文件路径 |
|------|----------|
| Serializer | `Engine/Source/Runtime/Core/Meta/Serializer/serializer.h` |
| Serializer 实现 | `Engine/Source/Runtime/Core/Meta/Serializer/serializer.cpp` |
| 类型注册 | `Engine/Source/Runtime/AutoGen/Type/TypeRegister.h` |
| Object 基类 | `Engine/Source/Runtime/Core/Meta/Reflection/object.h` |
| TypeManager | `Engine/Source/Runtime/Core/Meta/Reflection/type.h` |
| AssetManager | `Engine/Source/Runtime/Resource/AssetManager.h` |
| Scene | `Engine/Source/Runtime/Function/Scene/SceneManager.h` |
| Prefab | `Engine/Source/Runtime/Function/Prefab/Prefab.h` |
| Material | `Engine/Source/Runtime/Function/Renderer/Rendering/Material.h` |
| Component 基类 | `Engine/Source/Runtime/Function/Framework/Component/Base/component.h` |
| GameObject | `Engine/Source/Runtime/Function/Framework/GameObject/GameObject.h` |

## 设计模式

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| 模板方法 | Serializer | 序列化流程固定，具体类型处理可扩展 |
| 访问者 | RTTR property 遍历 | 遍历对象属性进行读写 |
| 策略 | 多态类型处理 | 根据类型元数据选择处理策略 |
| 代理 | rttr::instance | 包装对象实例进行反射操作 |
| 工厂 | 多态类型创建 | 通过 RTTR 类型名创建派生类实例 |

## 注意事项

1. **指针序列化**: 仅序列化指针指向对象的 ID，不直接序列化指针值
2. **循环引用**: 通过 ID 引用机制避免循环引用问题
3. **资源路径**: 使用相对路径，支持跨平台资源管理
4. **PostResourceLoaded**: 反序列化后必须调用以重建对象引用
5. **构造函数策略**: 多态类型通常使用 `ctor::as_raw_ptr` 策略
6. **线程安全**: 序列化操作本身非线程安全，需在主线程执行

## 总结

LitchiEngine 的序列化系统通过 RTTR 反射框架实现了类型无关的通用序列化方案。该系统的核心优势在于：

1. **低侵入性**: 只需添加 RTTR 注册代码，无需修改类实现
2. **多态支持**: 自动处理继承和多态类型
3. **可扩展性**: 通过元数据机制灵活控制序列化行为
4. **统一接口**: AssetManager 提供统一的资源管理 API

该设计模式可广泛应用于游戏引擎的资源管理、场景编辑器、网络同步等场景。
