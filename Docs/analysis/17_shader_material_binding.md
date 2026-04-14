# 着色器管理与材质绑定机制分析

## 1. 概述

本文档深入分析 LitchiEngine 中着色器的管理机制，包括 MaterialShader 类设计、SPIR-V 反射机制、Uniform 系统、描述符绑定约定以及材质参数传递流程。

## 2. 核心类关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Material (材质实例)                             │
│  - 持有 MaterialShader 引用                                              │
│  - 管理 Uniform 数据 (m_uniformDataList)                                 │
│  - 维护 GPU 常量缓冲区 (m_valueConstantBuffer)                           │
│  - 管理纹理资源 (m_textureMap)                                           │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 引用
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      MaterialShader (着色器组合)                          │
│  - 组合顶点/像素着色器                                                    │
│  - 缓存全局 Uniform 信息 (m_globalUniformDict)                           │
│  - 缓存纹理描述符 (m_textureDescriptorDict)                              │
│  - 合并所有描述符 (m_materialDescriptors)                                │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ 持有
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      RHI_Shader (着色器基类)                              │
│  - HLSL → SPIR-V 编译                                                    │
│  - SPIR-V 反射提取描述符                                                  │
│  - 管理描述符列表 (m_descriptors)                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. MaterialShader 类设计

### 3.1 类定义

```cpp
struct MaterialShader
{
    // 着色器资源
    std::string m_shaderPath;
    RHI_Shader* m_vertex_shader;
    RHI_Shader* m_pixel_shader;
    uint64_t m_hash = 0;

    // 描述符信息
    RHI_Descriptor m_globalMaterial;                           // 材质常量缓冲区描述符
    std::unordered_map<std::string, ShaderUniform> m_globalUniformDict;  // Uniform 名称 → 信息
    std::unordered_map<std::string, RHI_Descriptor> m_textureDescriptorDict; // 纹理名称 → 描述符
    std::vector<RHI_Descriptor> m_materialDescriptors;         // 合并后的所有描述符
};
```

### 3.2 加载流程

```
LoadFromFile(file_path)
    │
    ├── 1. 创建顶点着色器
    │       vertexShader->Compile(RHI_Shader_Vertex, file_path)
    │       └── HLSL → DXCompiler → SPIR-V
    │       └── Reflect() 提取描述符
    │
    ├── 2. 创建像素着色器
    │       pixelShader->Compile(RHI_Shader_Pixel, file_path)
    │       └── 同上
    │
    └── 3. LoadMaterialDescriptors()
            ├── GetGlobalDescriptor() 查找名为 "Material" 的 cbuffer
            ├── 提取 Uniform 信息到 m_globalUniformDict
            ├── 提取纹理描述符到 m_textureDescriptorDict
            └── 合并顶点/像素着色器的描述符
```

### 3.3 关键方法

#### GetGlobalDescriptor - 查找材质常量缓冲区

```cpp
bool RHI_Shader::GetGlobalDescriptor(RHI_Descriptor& globalDescriptor)
{
    for (const auto& descriptor : m_descriptors)
    {
        if (descriptor.name == "Material")  // 必须命名为 "Material"
        {
            globalDescriptor = descriptor;
            return true;
        }
    }
    return false;
}
```

**重要约定**：材质 cbuffer 必须命名为 `Material`，否则无法被识别。

## 4. SPIR-V 反射机制

### 4.1 反射流程

```cpp
void RHI_Shader::Reflect(const RHI_Shader_Stage shader_stage, const uint32_t* ptr, const uint32_t size)
{
    const CompilerHLSL compiler = CompilerHLSL(ptr, size);
    ShaderResources resources = compiler.get_shader_resources();

    // 提取各类资源
    spirv_resources_to_descriptors(compiler, m_descriptors, resources.separate_images, ...);    // 纹理
    spirv_resources_to_descriptors(compiler, m_descriptors, resources.uniform_buffers, ...);    // 常量缓冲区
    spirv_resources_to_descriptors(compiler, m_descriptors, resources.push_constant_buffers, ...);
    spirv_resources_to_descriptors(compiler, m_descriptors, resources.separate_samplers, ...);  // 采样器
}
```

### 4.2 描述符提取

```cpp
void spirv_resources_to_descriptors(
    const CompilerHLSL& compiler,
    vector<RHI_Descriptor>& descriptors,
    const SmallVector<Resource>& resources,
    const RHI_Descriptor_Type descriptor_type,
    const RHI_Shader_Stage shader_stage)
{
    for (const Resource& resource : resources)
    {
        uint32_t slot = compiler.get_decoration(resource.id, spv::DecorationBinding);
        auto name = compiler.get_name(resource.id);
        SPIRType type = compiler.get_type(resource.type_id);

        // 检查是否为材质描述符
        bool isMaterial = CheckIsMaterialDescriptor(compiler, resource);

        // 如果是材质常量缓冲区，提取成员变量信息
        if (isMaterial && type.basetype == SPIRType::Struct)
        {
            uniformList = spirv_constantBuffer_struct_uniformList(compiler, type);
        }

        descriptors.emplace_back(name, descriptor_type, layout, slot, ...);
    }
}
```

### 4.3 Uniform 信息提取

```cpp
vector<ShaderUniform>* spirv_constantBuffer_struct_uniformList(const CompilerHLSL& compiler, const SPIRType parentType)
{
    vector<ShaderUniform>* uniformList = new vector<ShaderUniform>();

    for (unsigned i = 0; i < parentType.member_types.size(); i++)
    {
        auto& member_type = compiler.get_type(parentType.member_types[i]);
        size_t member_size = compiler.get_declared_struct_member_size(parentType, i);
        const string& member_name = compiler.get_member_name(parentType.self, i);
        size_t member_offset = compiler.type_struct_member_offset(parentType, i);

        ShaderUniform uniformInfo;
        uniformInfo.name = member_name;      // 成员名称
        uniformInfo.size = member_size;       // 字节大小
        uniformInfo.location = member_offset; // 在 cbuffer 中的偏移
        uniformInfo.type = /* 根据类型推断 */;

        uniformList->push_back(uniformInfo);
    }

    return uniformList;
}
```

### 4.4 类型映射

| SPIR-V 类型 | UniformType |
|-------------|-------------|
| SPIRType::Boolean | UNIFORM_BOOL |
| SPIRType::Int | UNIFORM_INT |
| SPIRType::UInt | UNIFORM_UINT |
| SPIRType::Float (vecsize=1) | UNIFORM_FLOAT |
| SPIRType::Float (vecsize=2) | UNIFORM_FLOAT_VEC2 |
| SPIRType::Float (vecsize=3) | UNIFORM_FLOAT_VEC3 |
| SPIRType::Float (vecsize=4) | UNIFORM_FLOAT_VEC4 |
| SPIRType::Float (columns=4) | UNIFORM_FLOAT_MAT4 |
| SPIRType::Struct | UNIFORM_Struct (递归提取) |

## 5. ShaderUniform 结构

### 5.1 定义

```cpp
struct ShaderUniform
{
    UniformType type;              // 数据类型
    std::string name;              // 变量名
    uint32_t location;             // 在 cbuffer 中的偏移
    int size;                      // 字节大小
    std::vector<ShaderUniform>* memberUniform;  // 嵌套结构体成员
};
```

### 5.2 UniformType 枚举

```cpp
enum class UniformType : uint32_t
{
    UNIFORM_Unknown = 0x00,
    UNIFORM_BOOL,
    UNIFORM_INT,
    UNIFORM_UINT,
    UNIFORM_FLOAT,
    UNIFORM_FLOAT_VEC2,
    UNIFORM_FLOAT_VEC3,
    UNIFORM_FLOAT_VEC4,
    UNIFORM_FLOAT_MAT4,
    UNIFORM_DOUBLE_MAT4,
    UNIFORM_TEXTURE,
    UNIFORM_Struct,
};
```

## 6. 描述符绑定约定

### 6.1 寄存器槽位定义

```cpp
// shader register slot shifts (HLSL → SPIR-V 映射)
const uint32_t rhi_shader_shift_register_u = 100;    // UAV (u registers)
const uint32_t rhi_shader_shift_register_b = 200;    // ConstantBuffer (b registers)
const uint32_t rhi_shader_shift_register_s = 300;    // Sampler (s registers)
const uint32_t rhi_shader_shift_register_t = 400;    // Texture (t registers)
const uint32_t rhi_shader_shift_register_material_t = 500;    // 材质纹理
const uint32_t rhi_shader_shift_register_material_value = 10; // 材质 cbuffer 偏移
```

### 6.2 材质描述符判断

```cpp
bool CheckIsMaterialDescriptor(const CompilerHLSL& compiler, const Resource& resource)
{
    uint32_t slot = compiler.get_decoration(resource.id, spv::DecorationBinding);

    // 材质判定条件：
    // 1. slot >= 500 (材质纹理槽位)
    // 2. slot == 210 (200 + 10, 材质 cbuffer 槽位)
    if (slot >= rhi_shader_shift_register_material_t ||
        slot == rhi_shader_shift_register_b + rhi_shader_shift_register_material_value)
    {
        return true;
    }
    return false;
}
```

### 6.3 HLSL 绑定约定

| 资源类型 | HLSL 寄存器 | 实际 slot | 说明 |
|----------|-------------|-----------|------|
| 材质常量缓冲区 | `register(b10)` | 210 | 必须命名为 `Material` |
| 材质纹理 | `register(t100+)` | 500+ | 未使用纹理会被优化掉 |
| 引擎常量缓冲区 | `register(b0-b9)` | 200-209 | Frame, Light, MaterialBufferData |
| 引擎纹理 | `register(t0-t99)` | 400-499 | 深度图、阴影图等 |

### 6.4 DXCompiler 槽位偏移

编译时通过参数指定槽位偏移：

```cpp
arguments.emplace_back("-fvk-b-shift"); arguments.emplace_back("200"); arguments.emplace_back("all");
arguments.emplace_back("-fvk-t-shift"); arguments.emplace_back("400"); arguments.emplace_back("all");
```

这意味着：
- HLSL `register(b0)` → SPIR-V binding 200
- HLSL `register(b10)` → SPIR-V binding 210
- HLSL `register(t100)` → SPIR-V binding 500

## 7. Material 类与 Uniform 绑定

### 7.1 Material 核心成员

```cpp
class Material : public IResource
{
private:
    MaterialRes* m_materialRes;                      // 序列化数据
    MaterialShader* m_shader;                        // 着色器引用
    std::map<std::string, std::any> m_uniformDataList;  // Uniform 数据缓存
    std::shared_ptr<RHI_ConstantBuffer> m_valueConstantBuffer;  // GPU 常量缓冲区
    std::map<int, RHI_Texture*> m_textureMap;        // 纹理槽位映射
    void* m_value;                                   // CPU 端数据缓冲
    int m_valueSize;                                 // 缓冲大小
    bool m_isValueDirty;                             // 脏标记
};
```

### 7.2 材质加载流程

```
Material::LoadFromFile(file_path)
    │
    ├── 1. 加载 JSON 文件到 MaterialRes
    │
    ├── 2. 加载着色器
    │       shaderManager->LoadResource(shaderPath)
    │
    └── 3. PostResourceLoaded()
            │
            ├── 初始化 m_uniformDataList
            │       遍历 shader 的 m_globalUniformDict 和 m_textureDescriptorDict
            │       为每个 Uniform 创建空条目
            │
            ├── 填充 Uniform 数据
            │       遍历 materialRes->uniformInfoList
            │       根据 UniformType 转换数据类型
            │       存入 m_uniformDataList
            │
            ├── 分配 GPU 缓冲
            │       m_value = malloc(shader->GetGlobalSize())
            │       m_valueConstantBuffer->Create(size, 1)
            │
            └── UpdateRenderData()
                    遍历 m_uniformDataList
                    SyncToDataBuffer() 将数据写入 m_value
                    构建 m_textureMap
```

### 7.3 Uniform 数据同步

```cpp
void Material::SyncToDataBuffer(const std::string& name)
{
    const auto& value = m_uniformDataList[name];
    const auto& uniformInfo = m_shader->GetGlobalUniformInfo(name);

    int offset = uniformInfo.location;  // 从 SPIR-V 反射获取
    int size = uniformInfo.size;

    // 根据 UniformType 处理不同类型
    switch (uniformInfo.type)
    {
    case UniformType::UNIFORM_FLOAT:
        auto floatValue = std::any_cast<float>(value);
        memcpy(reinterpret_cast<std::byte*>(m_value) + offset, &floatValue, size);
        break;
    case UniformType::UNIFORM_FLOAT_VEC3:
        auto vec3Value = std::any_cast<Vector3>(value);
        memcpy(reinterpret_cast<std::byte*>(m_value) + offset, &vec3Value, size);
        break;
    // ... 其他类型
    }
}
```

### 7.4 GPU 数据提交

```cpp
void Material::UpdateRenderData()
{
    // 1. 同步所有 Uniform 到 CPU 缓冲
    for (auto& uniform : m_uniformDataList)
    {
        SyncToDataBuffer(uniform.first);
    }

    // 2. 更新 GPU 常量缓冲区
    m_valueConstantBuffer->UpdateWithReset(m_value);

    // 3. 构建纹理映射
    m_textureMap.clear();
    for (auto& uniform : m_uniformDataList)
    {
        auto& descriptor = m_shader->GetTextureDescriptor(uniform.first);
        if (descriptor.type == RHI_Descriptor_Type::Texture)
        {
            m_textureMap[descriptor.slot] = std::any_cast<RHI_Texture*>(uniform.second);
        }
    }
}
```

## 8. 材质文件格式

### 8.1 JSON 结构

```json
{
  "vertexType": "PosUvNorTan",
  "shaderPath": ":Shaders/Forward/PBR/PBRTest.hlsl",
  "uniformInfoList": [
    {
      "Type": "UniformInfoTexture",
      "name": "u_albedo",
      "path": ":Textures/PBRTest/F3_Green.png"
    },
    {
      "Type": "UniformInfoFloat",
      "name": "u_roughness",
      "value": 0.5
    },
    {
      "Type": "UniformInfoVector3",
      "name": "u_baseColor",
      "vector": { "x": 1.0, "y": 0.0, "z": 0.0 }
    }
  ]
}
```

### 8.2 UniformInfo 类型

| 类型名 | 字段 | 说明 |
|--------|------|------|
| UniformInfoBool | `value` | 布尔值 |
| UniformInfoInt | `value` | 整数值 |
| UniformInfoFloat | `value` | 浮点值 |
| UniformInfoVector2 | `vector: {x, y}` | 2D 向量 |
| UniformInfoVector3 | `vector: {x, y, z}` | 3D 向量 |
| UniformInfoVector4 | `vector: {x, y, z, w}` | 4D 向量 |
| UniformInfoTexture | `path` | 纹理路径 |

## 9. 完整数据流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           材质文件 (.mat)                                │
│  { shaderPath, uniformInfoList: [{ name, type, value }] }              │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ LoadFromFile
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          MaterialRes (反序列化)                          │
│  - shaderPath: string                                                    │
│  - uniformInfoList: vector<UniformInfoBase*>                            │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      MaterialShader (着色器加载)                         │
│  - 编译 HLSL → SPIR-V                                                   │
│  - SPIR-V 反射 → 描述符列表                                              │
│  - 提取 Material cbuffer 成员 → m_globalUniformDict                     │
│  - 提取纹理描述符 → m_textureDescriptorDict                              │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Material (运行时实例)                                │
│  - m_uniformDataList: map<string, any>  // Uniform 名称 → 值            │
│  - m_value: void*  // CPU 端 cbuffer 数据                               │
│  - m_valueConstantBuffer: GPU cbuffer                                   │
│  - m_textureMap: map<slot, Texture*>  // 纹理槽位 → 纹理                 │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ UpdateRenderData
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          GPU 渲染                                        │
│  - DescriptorSet 绑定 cbuffer (slot 210)                                │
│  - DescriptorSet 绑定纹理 (slot 500+)                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## 10. 常见问题与约定

### 10.1 材质 cbuffer 必须命名为 "Material"

```hlsl
// ✅ 正确
cbuffer Material : register(b10)
{
    MaterialData materialData;
};

// ❌ 错误 - 无法被识别为材质
cbuffer MyMaterialParams : register(b10)
{
    MaterialData materialData;
};
```

### 10.2 未使用的纹理会被优化掉

```hlsl
Texture2D u_normalMap : register(t101);  // 如果未使用，编译器会移除

// 确保纹理被使用
float3 normal = u_normalMap.Sample(sampler, uv).xyz;
```

### 10.3 材质纹理槽位约定

| 用途 | 推荐 slot | 说明 |
|------|-----------|------|
| 漫反射/Albedo | t101 | |
| 法线 | t102 | |
| 金属度 | t103 | |
| 粗糙度 | t104 | |
| AO | t105 | |

### 10.4 错误诊断

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `MaterialShader::LoadMaterialDescriptors Fail` | cbuffer 未命名为 "Material" | 重命名 cbuffer |
| `Not Found Uniform uniformName:xxx` | 材质文件定义了着色器中不存在的 uniform | 移除材质文件中的多余定义，或在着色器中使用该变量 |
| `UniformList = null or Size = 0` | 纹理类型描述符无成员列表 | 正常现象，纹理不需要 UniformList |

## 11. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 组合模式 | MaterialShader 组合 RHI_Shader | 顶点/像素着色器组合 |
| 缓存模式 | m_globalUniformDict, m_textureDescriptorDict | 避免重复反射查询 |
| 脏标记模式 | m_isValueDirty | 延迟更新 GPU 数据 |
| 类型擦除 | std::any 存储不同类型 Uniform | 统一存储接口 |
| 反射模式 | SPIR-V 反射 | 运行时获取着色器结构 |

---

**分析完成时间**: 2026-04-13
**分析者**: AI Agent
