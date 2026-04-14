# 着色器与材质管理系统分析

## 1. 模块概述

材质管理系统负责管理着色器资源、材质属性和 Uniform 数据的绑定与传输。系统采用三层架构设计：

- **Material**: 材质实例，存储具体的属性值和纹理引用
- **MaterialShader**: 着色器组合器，管理 Vertex/Pixel Shader 的组合和描述符信息
- **MaterialManager**: 资源管理器，负责材质的加载、缓存和生命周期管理

核心设计理念是通过 `MaterialShader` 将 HLSL 着色器中的 `cbuffer Material` 和纹理绑定信息反射提取，然后在 `Material` 实例中存储具体的值，最终通过 `RHI_ConstantBuffer` 传递到 GPU。

## 2. 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        MaterialManager                          │
│  - CreateResource(path) -> Material*                            │
│  - DestroyResource(Material*)                                   │
│  - ReloadResource(Material*, path)                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 管理
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Material                               │
│  - m_materialRes: MaterialRes*           // 序列化数据          │
│  - m_shader: MaterialShader*             // 关联的着色器        │
│  - m_uniformDataList: map<string, any>   // Uniform 值存储      │
│  - m_valueConstantBuffer: RHI_ConstantBuffer  // GPU 缓冲       │
│  - m_textureMap: map<int, RHI_Texture*>  // 纹理绑定            │
│  - m_value: void*                        // 打包的 Uniform 数据  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 引用
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                       MaterialShader                            │
│  - m_vertex_shader: RHI_Shader*                                 │
│  - m_pixel_shader: RHI_Shader*                                  │
│  - m_globalMaterial: RHI_Descriptor      // Material cbuffer    │
│  - m_globalUniformDict: map<string, ShaderUniform>              │
│  - m_textureDescriptorDict: map<string, RHI_Descriptor>         │
│  - m_materialDescriptors: vector<RHI_Descriptor>                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 包含
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                          RHI_Shader                             │
│  - m_descriptors: vector<RHI_Descriptor> // 反射的描述符        │
│  - m_input_layout: RHI_InputLayout       // 顶点输入布局        │
│  - m_rhi_resource: VkShaderModule        // Vulkan 着色器模块   │
└─────────────────────────────────────────────────────────────────┘
```

## 3. Material 类详细分析

### 3.1 核心职责

`Material` 类是材质系统的核心，负责：

1. 存储 Uniform 变量值（标量、向量、纹理引用）
2. 管理 ConstantBuffer 用于 GPU 数据传输
3. 序列化/反序列化材质配置
4. 同步数据到渲染管线

### 3.2 关键数据结构

```cpp
class Material : public IResource {
private:
    // 序列化数据
    MaterialRes* m_materialRes;

    // 关联的着色器
    MaterialShader* m_shader;

    // Uniform 值存储 (使用 std::any 实现类型擦除)
    std::map<std::string, std::any> m_uniformDataList;

    // GPU 常量缓冲区
    std::shared_ptr<RHI_ConstantBuffer> m_valueConstantBuffer;

    // 纹理绑定映射 (slot -> texture)
    std::map<int, RHI_Texture*> m_textureMap;

    // 打包的 Uniform 数据内存
    void* m_value;
    int m_valueSize;
    bool m_isValueDirty;
};
```

### 3.3 UniformInfo 类型系统

材质系统使用 RTTR 反射支持的类型系统来存储序列化数据：

```cpp
// 基类
class UniformInfoBase {
public:
    std::string name;
    virtual UniformType GetUniformType();
    RTTR_ENABLE()
};

// 具体类型
class UniformInfoBool : public UniformInfoBase { float value; };
class UniformInfoInt : public UniformInfoBase { int value; };
class UniformInfoFloat : public UniformInfoBase { float value; };
class UniformInfoVector2 : public UniformInfoBase { Vector2 vector; };
class UniformInfoVector3 : public UniformInfoBase { Vector3 vector; };
class UniformInfoVector4 : public UniformInfoBase { Vector4 vector; };
class UniformInfoTexture : public UniformInfoBase { std::string path; };
```

### 3.4 核心方法分析

#### 3.4.1 SetValue / GetValue

```cpp
template<typename T>
void Material::SetValue(const std::string& name, const T& value) {
    if (m_uniformDataList.find(name) != m_uniformDataList.end()) {
        m_uniformDataList[name] = std::any(value);
        m_isValueDirty = true;  // 标记需要同步到 GPU
    }
}

template<typename T>
const T& Material::GetValue(const std::string p_key) {
    if (m_uniformDataList.find(p_key) == m_uniformDataList.end()) {
        return T();
    }
    std::any a = m_uniformDataList.at(p_key);
    if (a.type() == typeid(T)) {
        return std::any_cast<T>(a);
    }
    return T();
}
```

**设计要点**：
- 使用 `std::any` 实现类型擦除，支持多种数据类型
- 设置值时自动标记 `m_isValueDirty`，触发延迟同步机制
- 类型安全检查，避免错误类型转换

#### 3.4.2 SetTexture

```cpp
void Material::SetTexture(const std::string& name, RHI_Texture* texture) {
    if (m_uniformDataList.find(name) != m_uniformDataList.end()) {
        m_uniformDataList[name] = std::make_any<RHI_Texture*>(texture);
    }
    m_isValueDirty = true;
}

// 便捷方法：通过枚举类型设置 PBR 纹理
void Material::SetTexture(MaterialTexture textureType, RHI_Texture* texture) {
    auto name = GetPBRUniformNameFromMaterialTextureType(textureType);
    SetTexture(name, texture);
}
```

**PBR 纹理映射**：

| MaterialTexture | Shader Uniform Name |
|-----------------|---------------------|
| Color | u_albedo |
| Roughness | u_roughness |
| Metalness | u_metallic |
| Normal | u_normal |
| Occlusion | u_aO |
| Emission | - |
| Height | - |
| AlphaMask | - |

#### 3.4.3 PostResourceLoaded

```cpp
void Material::PostResourceLoaded() {
    // 1. 清空并初始化 UniformDataList
    m_uniformDataList.clear();

    // 从着色器获取所有 Uniform 声明
    for (const auto& element : m_shader->GetGlobalShaderUniformDict()) {
        m_uniformDataList.emplace(element.first, std::any());
    }
    for (const auto& element : m_shader->GetTextureDescriptorDict()) {
        m_uniformDataList.emplace(element.first, std::any());
    }

    // 2. 从序列化数据加载值
    for (auto uniformInfo : m_materialRes->uniformInfoList) {
        // 根据类型加载数据...
    }

    // 3. 分配 GPU 缓冲区
    if (m_value == nullptr) {
        int size = CalcValueSize();  // 从着色器获取全局缓冲区大小
        m_value = malloc(size);
        m_valueConstantBuffer->Create(size, 1);
    }

    // 4. 同步到 GPU
    UpdateRenderData();
}
```

## 4. MaterialShader 类详细分析

### 4.1 核心职责

`MaterialShader` 是着色器组合器，负责：

1. 管理配对的 Vertex/Pixel Shader
2. 反射提取 Material cbuffer 结构
3. 收集材质纹理描述符
4. 生成用于描述符集创建的描述符列表

### 4.2 数据结构

```cpp
struct MaterialShader {
    // 着色器路径
    std::string m_shaderPath;

    // 着色器引用
    RHI_Shader* m_vertex_shader;
    RHI_Shader* m_pixel_shader;

    // 哈希值 (用于管道缓存)
    uint64_t m_hash;

    // 全局材质描述符 (cbuffer Material)
    RHI_Descriptor m_globalMaterial;

    // Uniform 名称 -> 信息映射
    std::unordered_map<std::string, ShaderUniform> m_globalUniformDict;

    // 纹理名称 -> 描述符映射
    std::unordered_map<std::string, RHI_Descriptor> m_textureDescriptorDict;

    // 合并的描述符列表 (用于创建描述符集布局)
    std::vector<RHI_Descriptor> m_materialDescriptors;
};
```

### 4.3 描述符加载流程

```cpp
bool MaterialShader::LoadMaterialDescriptors() {
    // 1. 获取全局材质描述符
    bool isGlobalMaterial = m_vertex_shader->GetGlobalDescriptor(m_globalMaterial);
    if (!isGlobalMaterial) {
        isGlobalMaterial = m_pixel_shader->GetGlobalDescriptor(m_globalMaterial);
    }

    // 2. 提取 cbuffer 成员
    auto uniformList = *(m_globalMaterial.uniformList->at(0).memberUniform);
    for (auto uniform : uniformList) {
        m_globalUniformDict[uniform.name] = uniform;
    }

    // 3. 收集材质纹理 (slot >= 500 表示材质纹理)
    for (auto& rhi_descriptor : m_vertex_shader->GetDescriptors()) {
        if (rhi_descriptor.type == RHI_Descriptor_Type::Texture &&
            rhi_descriptor.slot >= rhi_shader_shift_register_material_t) {
            m_textureDescriptorDict[rhi_descriptor.name] = rhi_descriptor;
        }
    }
    // 同样处理像素着色器...

    // 4. 合并描述符 (Vertex + Pixel)
    m_materialDescriptors = m_vertex_shader->GetDescriptors();
    // 合并像素着色器描述符，更新 stage 标志...
}
```

### 4.4 寄存器槽位分配

系统使用槽位偏移策略区分不同类型的描述符：

| 常量 | 值 | 用途 |
|------|-----|------|
| `rhi_shader_shift_register_u` | 100 | UAV (u 寄存器) |
| `rhi_shader_shift_register_b` | 200 | ConstantBuffer (b 寄存器) |
| `rhi_shader_shift_register_s` | 300 | Sampler (s 寄存器) |
| `rhi_shader_shift_register_t` | 400 | Texture (t 寄存器) |
| `rhi_shader_shift_register_material_t` | 500 | 材质纹理起始槽位 |
| `rhi_shader_shift_register_material_value` | 10 | 材质常量缓冲槽位 (b10) |

## 5. MaterialManager 分析

### 5.1 继承结构

```cpp
class MaterialManager : public AResourceManager<Material> {
public:
    virtual Material* CreateResource(const std::string& p_path) override;
    virtual void DestroyResource(Material* p_resource) override;
    virtual void ReloadResource(Material* p_resource, const std::string& p_path) override;

    Material* CreateMaterial(const std::string& p_path);
};
```

### 5.2 资源生命周期

```
LoadResource(path)
    │
    ├── 检查缓存
    │   └── 如果存在，返回缓存实例
    │
    ├── CreateResource(path)
    │   └── new Material() + LoadFromFile()
    │
    ├── RegisterResource(path, material)
    │   └── m_resources[path] = material
    │
    └── 返回 Material*
```

### 5.3 核心方法实现

```cpp
Material* MaterialManager::CreateResource(const std::string& p_path) {
    std::string realPath = GetRealPath(p_path);

    Material* material = new Material();
    if (!material->LoadFromFile(realPath)) {
        delete material;
        return nullptr;
    }

    return material;
}

Material* MaterialManager::CreateMaterial(const std::string& p_path) {
    auto relativePath = GetRelativePath(p_path);
    std::string realPath = GetRealPath(relativePath);
    Material* prefab = new Material(realPath);
    RegisterResource(relativePath, prefab);
    return prefab;
}
```

## 6. Uniform 数据流分析

### 6.1 数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                       编辑器 / 脚本层                            │
│   material->SetValue<float>("u_roughness", 0.5f);               │
│   material->SetTexture("u_albedo", texture);                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Material::m_uniformDataList               │
│   map<string, any> = {                                          │
│       {"u_roughness", 0.5f},                                    │
│       {"u_albedo", RHI_Texture*},                               │
│       ...                                                       │
│   }                                                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │ Tick() / UpdateRenderData()
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Material::SyncToDataBuffer                │
│   根据 ShaderUniform 信息打包到 m_value 内存块                   │
│   - offset = uniformInfo.location                               │
│   - size = uniformInfo.size                                     │
│   memcpy(m_value + offset, &value, size);                       │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 RHI_ConstantBuffer::UpdateWithReset             │
│   memcpy(m_mapped_data + m_offset, data_cpu, m_stride);         │
│   (持久映射，直接写入 GPU 可见内存)                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                          GPU                                    │
│   cbuffer Material : register(b10) {                            │
│       MaterialData materialData;                                │
│   }                                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 SyncToDataBuffer 详细实现

```cpp
void Material::SyncToDataBuffer(const std::string& name) {
    // 1. 检查名称有效性
    if (m_uniformDataList.find(name) == m_uniformDataList.end()) return;

    const auto& value = m_uniformDataList[name];
    if (!value.has_value()) return;

    // 2. 获取着色器反射信息
    const auto& uniformInfo = m_shader->GetGlobalUniformInfo(name);
    int offset = uniformInfo.location;
    int size = uniformInfo.size;
    UniformType uniformType = uniformInfo.type;

    // 3. 根据类型写入内存
    switch (uniformType) {
    case UniformType::UNIFORM_BOOL: {
        auto boolValue = std::any_cast<bool>(value);
        memcpy(reinterpret_cast<std::byte*>(m_value) + offset,
               reinterpret_cast<std::byte*>(&boolValue), size);
        break;
    }
    case UniformType::UNIFORM_FLOAT: {
        auto floatValue = std::any_cast<float>(value);
        memcpy(reinterpret_cast<std::byte*>(m_value) + offset,
               reinterpret_cast<std::byte*>(&floatValue), size);
        break;
    }
    case UniformType::UNIFORM_FLOAT_VEC4: {
        auto vec4Value = std::any_cast<Vector4>(value);
        memcpy(reinterpret_cast<std::byte*>(m_value) + offset,
               reinterpret_cast<std::byte*>(&vec4Value), size);
        break;
    }
    // ... 其他类型
    }
}
```

### 6.3 常量缓冲区对齐

Vulkan 要求 Uniform Buffer 按 `minUniformBufferOffsetAlignment` 对齐：

```cpp
void RHI_ConstantBuffer::RHI_CreateResource() {
    // 计算对齐后的步长
    size_t min_alignment = RHI_Device::PropertyGetMinUniformBufferOffsetAllignment();
    if (min_alignment > 0) {
        m_stride = static_cast<uint32_t>(
            (m_stride + min_alignment - 1) & ~(min_alignment - 1)
        );
    }

    // 创建缓冲区
    RHI_Device::MemoryBufferCreate(m_rhi_resource, m_object_size_gpu,
        VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT,  // 可映射
        nullptr, m_object_name.c_str());

    // 获取持久映射指针
    m_mapped_data = RHI_Device::MemoryGetMappedDataFromBuffer(m_rhi_resource);
}
```

## 7. 纹理绑定机制

### 7.1 纹理描述符收集

```cpp
// MaterialShader 中收集材质纹理
for (auto& rhi_descriptor : m_vertex_shader->GetDescriptors()) {
    if (rhi_descriptor.type == RHI_Descriptor_Type::Texture &&
        rhi_descriptor.slot >= rhi_shader_shift_register_material_t) {
        m_textureDescriptorDict[rhi_descriptor.name] = rhi_descriptor;
    }
}
```

### 7.2 纹理更新流程

```cpp
void Material::UpdateRenderData() {
    // 1. 更新值缓冲区
    for (auto& uniform : m_uniformDataList) {
        SyncToDataBuffer(uniform.first);
    }
    m_valueConstantBuffer->UpdateWithReset(m_value);

    // 2. 更新纹理映射
    m_textureMap.clear();
    for (auto& uniform : m_uniformDataList) {
        auto& name = uniform.first;
        auto& descriptor = m_shader->GetTextureDescriptor(name);

        if (descriptor.type == RHI_Descriptor_Type::Texture) {
            int slot = descriptor.slot;
            if (uniform.second.type() == typeid(RHI_Texture*)) {
                m_textureMap[slot] = any_cast<RHI_Texture*>(uniform.second);
            }
        }
    }
}
```

### 7.3 着色器中的纹理绑定

```hlsl
// PBRTest.hlsl
Texture2D u_albedo : register(t101);    // slot = 500 + 101 = 601
Texture2D u_normal : register(t102);
Texture2D u_metallic : register(t103);
Texture2D u_roughness : register(t104);
Texture2D u_aO : register(t105);

// 使用全局采样器数组
float4 albedo = u_albedo.Sample(samplers[sampler_point_wrap], g_TexCoords);
```

## 8. 设计模式总结

### 8.1 架构模式

| 模式 | 应用位置 | 说明 |
|------|----------|------|
| 资源管理模式 | MaterialManager + AResourceManager | 统一的资源加载、缓存、生命周期管理 |
| 类型擦除 | std::any 存储 Uniform 值 | 支持多种数据类型的统一存储 |
| 反射机制 | SPIR-V 反射提取描述符 | 运行时获取着色器结构信息 |
| 脏标记模式 | m_isValueDirty | 延迟更新，避免不必要的 GPU 同步 |
| 组合模式 | MaterialShader 组合 VS/PS | 灵活的着色器组合 |

### 8.2 关键设计决策

1. **分离 Material 和 MaterialShader**
   - Material 存储实例数据，MaterialShader 存储类型信息
   - 多个 Material 可以共享同一个 MaterialShader

2. **使用 std::any 存储 Uniform 值**
   - 优点：灵活，支持任意类型
   - 缺点：运行时类型检查开销

3. **持久映射 ConstantBuffer**
   - 避免每帧映射/取消映射开销
   - 直接写入 GPU 可见内存

4. **槽位偏移策略**
   - 通过槽位范围区分描述符类型
   - 材质纹理使用 t100+ 槽位

## 9. 使用示例

### 9.1 创建材质

```cpp
// 加载材质
Material* material = materialManager->LoadResource("Materials/PBR.mat");

// 设置属性
material->SetValue<Vector4>("u_color", Vector4(1.0f, 0.0f, 0.0f, 1.0f));
material->SetValue<Vector2>("u_textureTiling", Vector2(2.0f, 2.0f));

// 设置纹理
RHI_Texture* albedoTex = textureManager->LoadResource("Textures/brick.png");
material->SetTexture(MaterialTexture::Color, albedoTex);

// 或直接使用名称
material->SetTexture("u_albedo", albedoTex);
```

### 9.2 创建自定义着色器材质

```hlsl
// CustomShader.hlsl
struct MaterialData {
    float2 u_textureTiling;
    float2 u_textureOffset;
    float4 u_color;
    float u_time;  // 自定义属性
};

cbuffer Material : register(b10) {
    MaterialData materialData;
};

Texture2D u_mainTex : register(t101);
```

```cpp
// C++ 端
Material* material = new Material();
material->SetShader(shaderManager->LoadResource("Shaders/CustomShader.hlsl"));
material->SetValue<float>("u_time", 0.0f);

// 每帧更新
material->SetValue<float>("u_time", Time::GetTime());
```

### 9.3 序列化与加载

```cpp
// 保存材质
material->SaveToFile("Materials/MyMaterial.mat");

// 加载材质
Material* loadedMat = materialManager->LoadResource("Materials/MyMaterial.mat");
```

## 10. 扩展指南

### 10.1 添加新的 Uniform 类型

1. **在 UniformType.h 中添加枚举**：
```cpp
enum class UniformType : uint32_t {
    // ...
    UNIFORM_FLOAT_MAT3,  // 新增类型
};
```

2. **在 UniformInfo 类族中添加**：
```cpp
class UniformInfoMatrix3x3 : public UniformInfoBase {
public:
    UniformType GetUniformType() override {
        return UniformType::UNIFORM_FLOAT_MAT3;
    }
    Matrix3x3 matrix;
    RTTR_ENABLE(UniformInfoBase)
};
```

3. **在 Material::SyncToDataBuffer 中处理**：
```cpp
case UniformType::UNIFORM_FLOAT_MAT3: {
    auto mat3Value = std::any_cast<Matrix3x3>(value);
    memcpy(reinterpret_cast<std::byte*>(m_value) + offset,
           reinterpret_cast<std::byte*>(&mat3Value), size);
    break;
}
```

4. **在 Vulkan_Shader.cpp 的 SPIR-V 反射中处理**：
```cpp
// 在 spirv_constantBuffer_struct_uniformList 中添加类型识别
```

### 10.2 支持自定义着色器

材质系统已支持自定义着色器，只需：

1. 编写 HLSL 着色器，定义 `cbuffer Material : register(b10)`
2. 材质纹理使用 `register(t100+)`
3. 通过 ShaderManager 加载着色器
4. Material 会自动反射获取所有 Uniform

### 10.3 多 Pass 渲染支持

当前系统设计为单 Material cbuffer，如需多 Pass 支持：

1. 扩展 MaterialShader 支持多个描述符集
2. 或使用 Push Constant 传递 Pass 相关数据

---

**分析完成时间**: 2026-04-14
**分析者**: AI Agent
