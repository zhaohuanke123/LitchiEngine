# 脚本系统架构分析

## 1. 模块概述

脚本系统位于 `Engine/Source/Runtime/Function/Scripting/`，使用 Mono 运行时实现 C# 脚本支持。C# 脚本核心位于 `Engine/Source/ScriptCore/`。

## 2. 架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    C# 脚本层 (ScriptCore)                     │
│  ScriptObject, GameObject, Component, ScriptComponent        │
│  InternalCalls (内部调用接口)                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ mono_add_internal_call
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    C++ 脚本引擎层                              │
│  ScriptEngine, ScriptClass, ScriptInstance, ScriptRegister   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Mono 运行时                                 │
│  MonoDomain, MonoAssembly, MonoClass, MonoObject, MonoMethod │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心类关系

```
ScriptEngine (引擎入口)
    │
    ├── ScriptClass (脚本类定义)
    │       └── MonoClass* (Mono运行时句柄)
    │
    ├── ScriptInstance (脚本实例)
    │       ├── MonoObject* (托管对象)
    │       └── ScriptClass* (类定义)
    │
    └── ScriptObject (C++ 端基类)
            └── m_unmanagedId (对象ID映射)
```

## 3. ScriptEngine - 脚本引擎核心

### 3.1 核心职责

- 初始化 Mono 运行时
- 加载核心程序集 (LitchiScriptCore.dll)
- 加载应用程序集 (用户脚本)
- 管理脚本类和实例
- 提供 C++/C# 互操作接口

### 3.2 初始化流程

```cpp
void ScriptEngine::Init(std::string dataPath) {
    // 1. 初始化数据结构
    s_data = new ScriptEngineData();

    // 2. 初始化 Mono 运行时
    std::string monoDllPath = dataPath + "Assets/Mono";
    InitMono(monoDllPath);

    // 3. 加载核心程序集
    std::string scriptCoreDllPath = dataPath + "LitchiScriptCore.dll";
    LoadCoreAssembly(scriptCoreDllPath);

    // 4. 加载程序集中的类型
    LoadAssemblyClasses();

    // 5. 注册内部调用
    ScriptRegister::RegisterFunctions();

    // 6. 注册组件映射
    ScriptRegister::RegisterComponents();

    // 7. 缓存引擎核心类
    s_data->EngineClass4ScriptObject = ScriptClass("LitchiEngine", "ScriptObject", true);
    s_data->EngineClass4Component = ScriptClass("LitchiEngine", "Component", true);
    s_data->EngineClass4ScriptComponent = ScriptClass("LitchiEngine", "ScriptComponent", true);
    s_data->EngineClass4Scene = ScriptClass("LitchiEngine", "Scene", true);
    s_data->EngineClass4GameObject = ScriptClass("LitchiEngine", "GameObject", true);
}
```

### 3.3 Mono 初始化

```cpp
void ScriptEngine::InitMono(std::string monoDllPath) {
    // 设置 Mono 运行时程序集目录
    mono_set_assemblies_path(monoDllPath.c_str());

    // 调试模式配置
    if (s_data->EnableDebugging) {
        const char* argv[2] = {
            "--debugger-agent=transport=dt_socket,address=127.0.0.1:2550,server=y,suspend=n",
            "--soft-breakpoints"
        };
        mono_jit_parse_options(2, (char**)argv);
        mono_debug_init(MONO_DEBUG_FORMAT_MONO);
    }

    // 初始化 JIT
    MonoDomain* rootDomain = mono_jit_init("LitchiMonoRuntime");
    s_data->RootDomain = rootDomain;

    // 标记主线程
    mono_thread_set_main(mono_thread_current());
}
```

### 3.4 程序集加载

```cpp
bool ScriptEngine::LoadCoreAssembly(const std::filesystem::path& filepath) {
    // 加载程序集
    s_data->CoreAssembly = Utils::LoadMonoAssembly(filepath, s_data->EnableDebugging);
    if (s_data->CoreAssembly == nullptr)
        return false;

    // 获取程序集镜像
    s_data->CoreAssemblyImage = mono_assembly_get_image(s_data->CoreAssembly);
    return true;
}

// 辅助函数：从文件加载程序集
static MonoAssembly* LoadMonoAssembly(const std::filesystem::path& assemblyPath, bool loadPDB) {
    // 读取文件数据
    ScopedBuffer fileData = FileSystem::ReadFileBinary(assemblyPath);

    // 打开镜像
    MonoImageOpenStatus status;
    MonoImage* image = mono_image_open_from_data_full(fileData.As<char>(), fileData.Size(), 1, &status, 0);

    // 加载 PDB 调试符号
    if (loadPDB) {
        std::filesystem::path pdbPath = assemblyPath;
        pdbPath.replace_extension(".pdb");
        if (std::filesystem::exists(pdbPath)) {
            ScopedBuffer pdbFileData = FileSystem::ReadFileBinary(pdbPath);
            mono_debug_open_image_from_memory(image, pdbFileData.As<const mono_byte>(), pdbFileData.Size());
        }
    }

    // 加载程序集
    MonoAssembly* assembly = mono_assembly_load_from_full(image, pathString.c_str(), &status, 0);
    mono_image_close(image);

    return assembly;
}
```

### 3.5 类型加载

```cpp
void ScriptEngine::LoadAssemblyClasses() {
    s_data->ScriptObjectClassDict.clear();

    // 找到引擎的脚本基类
    MonoClass* engineObjectClass = mono_class_from_name(s_data->CoreAssemblyImage, "LitchiEngine", "ScriptObject");

    // 遍历程序集中的所有类型
    const MonoTableInfo* typeDefinitionsTable = mono_image_get_table_info(s_data->CoreAssemblyImage, MONO_TABLE_TYPEDEF);
    int32_t numTypes = mono_table_info_get_rows(typeDefinitionsTable);

    for (int32_t i = 0; i < numTypes; i++) {
        // 获取类型信息
        const char* nameSpace = mono_metadata_string_heap(image, cols[MONO_TYPEDEF_NAMESPACE]);
        const char* className = mono_metadata_string_heap(image, cols[MONO_TYPEDEF_NAME]);

        // 检查是否是 ScriptObject 的子类
        MonoClass* monoClass = mono_class_from_name(image, nameSpace, className);
        bool isEngineObject = mono_class_is_subclass_of(monoClass, engineObjectClass, false);
        if (!isEngineObject)
            continue;

        // 创建脚本类并缓存
        Ref<ScriptClass> scriptClass = CreateRef<ScriptClass>(nameSpace, className, true);
        s_data->ScriptObjectClassDict[fullName] = scriptClass;

        // 加载字段信息
        void* iterator = nullptr;
        while (MonoClassField* field = mono_class_get_fields(monoClass, &iterator)) {
            const char* fieldName = mono_field_get_name(field);
            MonoType* type = mono_field_get_type(field);
            ScriptFieldType fieldType = Utils::MonoTypeToScriptFieldType(type);

            scriptClass->m_fields[fieldName] = { fieldType, fieldName, field };
        }
    }
}
```

## 4. ScriptClass - 脚本类句柄

### 4.1 类定义

```cpp
class ScriptClass {
public:
    ScriptClass() = default;
    ScriptClass(const std::string& classNamespace, const std::string& className, bool isCore = false);

    // 实例化
    MonoObject* Instantiate();

    // 方法操作
    MonoMethod* GetMethod(const std::string& name, int parameterCount);
    MonoObject* InvokeMethod(MonoObject* instance, MonoMethod* method, void** params = nullptr);

    // 字段访问
    const std::map<std::string, ScriptField>& GetFields() const { return m_fields; }

private:
    std::string m_classNamespace;
    std::string m_className;
    std::map<std::string, ScriptField> m_fields;
    MonoClass* m_monoClass = nullptr;
};
```

### 4.2 实现

```cpp
ScriptClass::ScriptClass(const std::string& classNamespace, const std::string& className, bool isCore)
    : m_classNamespace(classNamespace), m_className(className)
{
    // 从程序集获取 MonoClass
    m_monoClass = mono_class_from_name(
        isCore ? s_data->CoreAssemblyImage : s_data->AppAssemblyImage,
        classNamespace.c_str(),
        className.c_str()
    );
}

MonoObject* ScriptClass::Instantiate() {
    return ScriptEngine::InstantiateClass(m_monoClass);
}

MonoMethod* ScriptClass::GetMethod(const std::string& name, int parameterCount) {
    return mono_class_get_method_from_name(m_monoClass, name.c_str(), parameterCount);
}

MonoObject* ScriptClass::InvokeMethod(MonoObject* instance, MonoMethod* method, void** params) {
    MonoObject* exception = nullptr;
    auto result = mono_runtime_invoke(method, instance, params, &exception);

    if (exception != nullptr) {
        mono_unhandled_exception(exception);
        DEBUG_LOG_ERROR("InvokeMethod Fail Exception");
        return nullptr;
    }
    return result;
}
```

## 5. ScriptInstance - 脚本实例

### 5.1 类定义

```cpp
class ScriptInstance {
public:
    ScriptInstance(Ref<ScriptClass> scriptClass, uint64_t unmanagedId);

    // 生命周期调用
    void InvokeOnCreate();
    void InvokeOnUpdate(float ts);

    // 方法调用
    void Invoke(std::string methodName, void* param, int paramCount = 1);
    void InvokeBaseClass(Ref<ScriptClass> baseScriptClass, std::string methodName, void* param, int paramCount = 1);

    // 字段访问
    template<typename T> T GetFieldValue(const std::string& name);
    template<typename T> void SetFieldValue(const std::string& name, T value);

    // 属性
    Ref<ScriptClass> GetScriptClass() { return m_scriptClass; }
    uint64_t GetUnmanagedId();
    MonoObject* GetManagedObject() { return m_managedObject; }

private:
    bool GetFieldValueInternal(const std::string& name, void* buffer);
    bool SetFieldValueInternal(const std::string& name, const void* value);

private:
    Ref<ScriptClass> m_scriptClass;
    uint64_t m_unmanagedId;
    MonoObject* m_managedObject = nullptr;

    // 缓存的方法
    MonoMethod* m_constructor = nullptr;
    MonoMethod* m_onCreateMethod = nullptr;
    MonoMethod* m_onUpdateMethod = nullptr;
};
```

### 5.2 实现

```cpp
ScriptInstance::ScriptInstance(Ref<ScriptClass> scriptClass, uint64_t unmanagedId)
    : m_scriptClass(scriptClass), m_unmanagedId(unmanagedId)
{
    // 实例化托管对象
    m_managedObject = scriptClass->Instantiate();

    // 缓存常用方法
    m_constructor = scriptClass->GetMethod(".ctor", 1);
    m_onCreateMethod = scriptClass->GetMethod("OnCreate", 0);
    m_onUpdateMethod = scriptClass->GetMethod("OnUpdate", 1);
}

void ScriptInstance::InvokeOnCreate() {
    if (m_onCreateMethod)
        m_scriptClass->InvokeMethod(m_managedObject, m_onCreateMethod);
}

void ScriptInstance::InvokeOnUpdate(float ts) {
    if (m_onUpdateMethod) {
        void* param = &ts;
        m_scriptClass->InvokeMethod(m_managedObject, m_onUpdateMethod, &param);
    }
}

bool ScriptInstance::GetFieldValueInternal(const std::string& name, void* buffer) {
    const auto& fields = m_scriptClass->GetFields();
    auto it = fields.find(name);
    if (it == fields.end())
        return false;

    const ScriptField& field = it->second;
    mono_field_get_value(m_managedObject, field.ClassField, buffer);
    return true;
}

bool ScriptInstance::SetFieldValueInternal(const std::string& name, const void* value) {
    const auto& fields = m_scriptClass->GetFields();
    auto it = fields.find(name);
    if (it == fields.end())
        return false;

    const ScriptField& field = it->second;
    mono_field_set_value(m_managedObject, field.ClassField, (void*)value);
    return true;
}
```

## 6. C++/C# 对象映射

### 6.1 映射机制

```
C++ 端                          C# 端
─────────────────────────────────────────────
ScriptObject                    ScriptObject
  └── m_unmanagedId (ID)    ←→    └── m_umanagedId (ID)

ScriptObjectInstanceDict[ID] → ScriptInstance → MonoObject*
```

### 6.2 ScriptObject (C++ 端)

```cpp
class ScriptObject : public Object {
public:
    ScriptObject() : Object() {}

    uint64_t GetUnmanagedId() {
        return m_unmanagedId;
    }

    RTTR_ENABLE(Object)

protected:
    uint64_t m_unmanagedId;
};
```

### 6.3 ScriptObject (C# 端)

```csharp
public abstract class ScriptObject {
    /// <summary>
    /// 设置非托管对象的Id
    /// </summary>
    public void SetUnmanagedIdFromEngine(ulong unmanagedId) {
        m_umanagedId = unmanagedId;
    }

    /// <summary>
    /// 非托管层定义的id
    /// </summary>
    protected internal ulong UnmanagedId => m_umanagedId;
    private ulong m_umanagedId;
}
```

## 7. 内部调用 (InternalCalls)

### 7.1 注册机制

```cpp
// C++ 端注册
#define LitchiEngine_ADD_INTERNAL_CALL(Name) \
    mono_add_internal_call("LitchiEngine.InternalCalls::" #Name, Name)

void ScriptRegister::RegisterFunctions() {
    LitchiEngine_ADD_INTERNAL_CALL(GetScriptInstance);
    // ... 更多内部调用
}
```

### 7.2 C# 端声明

```csharp
internal static class InternalCalls {
    // 获取脚本实例
    [MethodImplAttribute(MethodImplOptions.InternalCall)]
    internal extern static ScriptObject Internal_GetScriptInstance(ulong scriptObjectUnmanagedId);

    // GameObject 操作
    [MethodImplAttribute(MethodImplOptions.InternalCall)]
    internal extern static string Internal_GetGameObjectName(ulong gameObjectUnmanagedId);

    // Component 操作
    [MethodImplAttribute(MethodImplOptions.InternalCall)]
    internal extern static Component Internal_GetOrCreateComponent(ulong sceneUnmanagedId, ulong gameObjectUnmanagedId, string componentName);

    [MethodImplAttribute(MethodImplOptions.InternalCall)]
    internal extern static ScriptComponent Internal_GetOrCreateScriptComponent(ulong sceneUnmanagedId, ulong gameObjectUnmanagedId, string className);
}
```

### 7.3 典型实现

```cpp
// C++ 端实现
static MonoObject* GetScriptInstance(uint64_t unmanagedId) {
    return ScriptEngine::GetManagedInstance(unmanagedId);
}

// C# 端使用
public GameObject Scene => InternalCalls.Internal_GetScriptInstance(m_sceneUnmanageId) as GameObject;
```

## 8. C# 脚本核心类

### 8.1 类继承关系

```
ScriptObject (抽象基类)
    ├── Scene (场景)
    ├── GameObject (游戏对象)
    └── Component (组件基类)
            ├── Transform
            ├── ScriptComponent (脚本组件基类)
            │       └── TestScriptComponent (用户脚本)
            └── ... (其他内置组件)
```

### 8.2 Component 生命周期

```csharp
public abstract class Component : ScriptObject {
    /// <summary>脚本实例化时</summary>
    protected abstract void OnAwake();

    /// <summary>脚本启动时</summary>
    protected abstract void OnStart();

    /// <summary>脚本刷新</summary>
    protected abstract void OnUpdate(float deltaTime);
}
```

### 8.3 GameObject 实现

```csharp
public class GameObject : ScriptObject {
    /// <summary>添加组件</summary>
    public T AddComponent<T>() where T : Component {
        return GetOrCreateComponent<T>();
    }

    /// <summary>获取组件</summary>
    public T GetComponent<T>() where T : Component {
        return GetOrCreateComponent<T>();
    }

    private T GetOrCreateComponent<T>() where T : Component {
        var componentType = typeof(T);

        // 内置组件
        if (componentType.BaseType == typeof(Component)) {
            return InternalCalls.Internal_GetOrCreateComponent(
                Scene.UnmanagedId, UnmanagedId, componentType.Name) as T;
        }
        // 脚本组件
        else if (typeof(T).BaseType == typeof(ScriptComponent)) {
            return InternalCalls.Internal_GetOrCreateScriptComponent(
                Scene.UnmanagedId, UnmanagedId,
                $"{componentType.Namespace}.{componentType.Name}") as T;
        }
        return null;
    }

    public string Name => InternalCalls.Internal_GetGameObjectName(UnmanagedId);
    public Scene Scene => InternalCalls.Internal_GetScriptInstance(m_sceneUnmanageId) as Scene;
}
```

## 9. 脚本字段系统

### 9.1 字段类型

```cpp
enum class ScriptFieldType {
    None = 0,
    Float, Double,
    Bool, Char, Byte, Short, Int, Long,
    UByte, UShort, UInt, ULong,
    Vector2, Vector3, Vector4
};
```

### 9.2 类型映射

```cpp
static std::unordered_map<std::string, ScriptFieldType> s_ScriptFieldTypeMap = {
    { "System.Single", ScriptFieldType::Float },
    { "System.Double", ScriptFieldType::Double },
    { "System.Boolean", ScriptFieldType::Bool },
    { "System.Int32", ScriptFieldType::Int },
    { "System.Int64", ScriptFieldType::Long },
    { "LitchiEngine.Vector2", ScriptFieldType::Vector2 },
    { "LitchiEngine.Vector3", ScriptFieldType::Vector3 },
    { "LitchiEngine.Vector4", ScriptFieldType::Vector4 },
    // ...
};
```

### 9.3 字段实例

```cpp
struct ScriptFieldInstance {
    ScriptField Field;

    template<typename T>
    T GetValue() {
        static_assert(sizeof(T) <= 16, "Type too large!");
        return *(T*)m_Buffer;
    }

    template<typename T>
    void SetValue(T value) {
        static_assert(sizeof(T) <= 16, "Type too large!");
        memcpy(m_Buffer, &value, sizeof(T));
    }

private:
    uint8_t m_Buffer[16];
};
```

## 10. 引擎数据结构

### 10.1 ScriptEngineData

```cpp
struct ScriptEngineData {
    // Mono 运行时
    MonoDomain* RootDomain = nullptr;
    MonoAssembly* CoreAssembly = nullptr;
    MonoImage* CoreAssemblyImage = nullptr;
    MonoAssembly* AppAssembly = nullptr;
    MonoImage* AppAssemblyImage = nullptr;

    // 程序集路径
    std::filesystem::path CoreAssemblyFilepath;
    std::filesystem::path AppAssemblyFilepath;

    // 引擎核心类
    ScriptClass EngineClass4ScriptObject;
    ScriptClass EngineClass4Component;
    ScriptClass EngineClass4ScriptComponent;
    ScriptClass EngineClass4Scene;
    ScriptClass EngineClass4GameObject;

    // 脚本类和实例映射
    std::unordered_map<std::string, Ref<ScriptClass>> ScriptObjectClassDict;
    std::unordered_map<uint64_t, Ref<ScriptInstance>> ScriptObjectInstanceDict;
    std::unordered_map<uint64_t, ScriptFieldMap> ScriptObjectFieldDict;

    // ID 分配
    uint64_t m_unmanagedIdFactory;

    // 调试
    bool EnableDebugging = false;
};
```

## 11. 对象创建流程

### 11.1 创建 Scene

```cpp
uint64_t ScriptEngine::CreateScene() {
    auto scriptInstance = CreateScriptInstance(
        CreateRef<ScriptClass>(s_data->EngineClass4Scene));
    return scriptInstance->GetUnmanagedId();
}
```

### 11.2 创建 GameObject

```cpp
uint64_t ScriptEngine::CreateGameObject(uint64_t sceneUnmanagedId) {
    auto scriptInstance = CreateScriptInstance(
        CreateRef<ScriptClass>(s_data->EngineClass4GameObject));

    uint64_t id = scriptInstance->GetUnmanagedId();
    scriptInstance->Invoke("SetSceneUnmanagedIdFromEngine", &sceneUnmanagedId, 1);
    return id;
}
```

### 11.3 创建 Component

```cpp
uint64_t ScriptEngine::CreateComponent(uint64_t gameObjectUnmanagedId, const std::string& componentTypeName) {
    auto fullName = fmt::format("LitchiEngine.{}", componentTypeName);
    auto scriptClass = s_data->ScriptObjectClassDict[fullName];

    auto scriptInstance = CreateScriptInstance(scriptClass);

    uint64_t id = scriptInstance->GetUnmanagedId();
    scriptInstance->InvokeBaseClass(
        CreateRef<ScriptClass>(s_data->EngineClass4Component),
        "SetGameObjectUnmanagedIdFromEngine", &gameObjectUnmanagedId, 1);

    return id;
}
```

## 12. 调试支持

### 12.1 调试配置

```cpp
if (s_data->EnableDebugging) {
    const char* argv[2] = {
        "--debugger-agent=transport=dt_socket,address=127.0.0.1:2550,server=y,suspend=n,loglevel=3",
        "--soft-breakpoints"
    };
    mono_jit_parse_options(2, (char**)argv);
    mono_debug_init(MONO_DEBUG_FORMAT_MONO);
}
```

### 12.2 PDB 加载

```cpp
if (loadPDB) {
    std::filesystem::path pdbPath = assemblyPath;
    pdbPath.replace_extension(".pdb");
    if (std::filesystem::exists(pdbPath)) {
        ScopedBuffer pdbFileData = FileSystem::ReadFileBinary(pdbPath);
        mono_debug_open_image_from_memory(image, pdbFileData.As<const mono_byte>(), pdbFileData.Size());
    }
}
```

## 13. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 代理模式 | ScriptObject | C++/C# 对象桥接 |
| 工厂模式 | ScriptEngine | 创建脚本实例 |
| 注册表模式 | ScriptRegister | 注册内部调用 |
| 句柄模式 | ScriptClass | 封装 MonoClass |
| 缓存模式 | ScriptInstance | 缓存常用方法 |

## 14. 最佳实践

### 14.1 脚本编写规范

```csharp
// 继承 ScriptComponent 创建自定义脚本
public class PlayerController : ScriptComponent {
    // 公共字段可在编辑器中编辑
    public float MoveSpeed = 5.0f;

    protected override void OnAwake() {
        // 初始化
    }

    protected override void OnStart() {
        // 启动逻辑
    }

    protected override void OnUpdate(float deltaTime) {
        // 每帧更新
    }
}
```

### 14.2 C++ 端扩展

```cpp
// 1. 声明内部调用
static MonoObject* MyInternalCall(uint64_t id) {
    return ScriptEngine::GetManagedInstance(id);
}

// 2. 注册内部调用
void ScriptRegister::RegisterFunctions() {
    LitchiEngine_ADD_INTERNAL_CALL(MyInternalCall);
}

// 3. C# 端声明
// [MethodImplAttribute(MethodImplOptions.InternalCall)]
// internal extern static ScriptObject MyInternalCall(ulong id);
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
