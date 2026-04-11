# 着色器系统分析

## 1. 模块概述

着色器系统位于 `Engine/Data/Engine/Shaders/`，使用 HLSL 编写，通过 DXCompiler 编译为 SPIR-V 供 Vulkan 使用。着色器系统包含常量缓冲区定义、光照计算、PBR 材质实现等。

## 2. 目录结构

```
Engine/Data/Engine/Shaders/
├── Common/                     # 公共头文件
│   ├── common.hlsl             # 公共定义
│   ├── common_buffers.hlsl     # 常量缓冲区定义
│   ├── common_light.hlsl       # 光照计算
│   ├── common_structs.hlsl     # 结构体定义
│   ├── common_textures.hlsl    # 纹理定义
│   ├── common_samplers.hlsl    # 采样器定义
│   ├── common_colorspace.hlsl  # 颜色空间转换
│   ├── common_vertex_pixel.hlsl # 顶点/像素输入
│   └── shadow_mapping.hlsl     # 阴影映射
├── Forward/                    # 前向着色器
│   ├── Standard.hlsl           # 标准着色器
│   ├── Standard_Skin.hlsl      # 蒙皮着色器
│   └── PBR/
│       ├── PBRTest.hlsl        # PBR 测试着色器
│       └── PBRTest_Skin.hlsl   # PBR 蒙皮着色器
├── depth_light.hlsl            # 深度/阴影着色器
├── depth_light_skin.hlsl       # 蒙皮深度着色器
├── skyBox.hlsl                 # 天空盒着色器
├── grid.hlsl                   # 网格着色器
├── line.hlsl                   # 线条着色器
├── quad.hlsl                   # 四边形着色器
├── imgui.hlsl                  # ImGui 着色器
└── test.hlsl                   # 测试着色器
```

## 3. 常量缓冲区布局

### 3.1 FrameBufferData

```hlsl
struct FrameBufferData {
    float2 resolution_render;      // 渲染分辨率
    float2 resolution_output;      // 输出分辨率
    float2 taa_jitter_current;     // 当前帧 TAA 抖动
    float2 taa_jitter_previous;    // 上一帧 TAA 抖动
    float delta_time;              // 帧间隔
    uint frame;                    // 帧计数
    float gamma;                   // Gamma 值
    uint options;                  // 选项标志
};

cbuffer BufferFrame : register(b0) {
    FrameBufferData buffer_frame;
};
```

### 3.2 RendererPathBufferData

```hlsl
struct RendererPathBufferData {
    matrix view;                      // 视图矩阵
    matrix projection;                // 投影矩阵
    matrix view_projection;           // 视图投影矩阵
    matrix view_projection_inverted;  // 逆视图投影矩阵
    matrix view_projection_orthographic; // 正交投影
    matrix view_projection_unjittered;   // 无抖动视图投影
    matrix view_projection_previous;     // 上一帧视图投影
    
    float3 camera_position;  // 相机位置
    float camera_near;       // 近裁剪面
    
    float3 camera_direction; // 相机方向
    float camera_far;        // 远裁剪面
};

cbuffer BufferRendererPath : register(b5) {
    RendererPathBufferData buffer_rendererPath;
};
```

### 3.3 LightBufferData

```hlsl
struct LightBufferData {
    matrix view_projection[6];  // 光源视图投影矩阵 (点光源6面)
    
    float4 color;       // 光源颜色
    
    float3 position;    // 光源位置
    float intensity;    // 光源强度
    
    float3 forward;     // 光源方向
    float range;        // 光源范围
    
    float angle;        // 聚光灯角度
    uint flags;         // 光源标志
    float2 padding;
    
    // 光源类型判断
    bool light_is_directional() { return flags & uint(1U << 0); }
    bool light_is_point() { return flags & uint(1U << 1); }
    bool light_is_spot() { return flags & uint(1U << 2); }
    bool light_has_shadows() { return flags & uint(1U << 3); }
    
    // 计算光照方向
    float3 compute_direction(float3 fragment_position);
    
    // 计算衰减
    float compute_attenuation(const float3 surface_position);
};

RWStructuredBuffer<LightBufferData> buffer_lights : register(u1);
```

### 3.4 MaterialBufferData

```hlsl
struct MaterialBufferData {
    float4 color;       // 基础颜色
    
    float2 tiling;      // 纹理平铺
    float2 offset;      // 纹理偏移
    
    float roughness;    // 粗糙度
    float metallness;   // 金属度
    float normal;       // 法线强度
    float height;       // 高度
    
    uint properties;    // 材质属性标志
    float clearcoat;    // 清漆
    float clearcoat_roughness; // 清漆粗糙度
    float anisotropic;  // 各向异性
    
    float anisotropic_rotation; // 各向异性旋转
    float sheen;        // 光泽
    float sheen_tint;   // 光泽色调
    float padding3;
};

cbuffer BufferMaterial : register(b2) {
    MaterialBufferData buffer_material;
};
```

### 3.5 PassBufferData (Push Constant)

```hlsl
struct PassBufferData {
    matrix transform;  // 世界变换矩阵
    matrix m_value;    // 附加数据
};

[[vk::push_constant]]
PassBufferData buffer_pass;
```

### 3.6 BoneDataArr

```hlsl
const static int MaxBone = 512;
struct BoneDataArr {
    matrix boneTransformArr[MaxBone]; // 骨骼变换矩阵
};

cbuffer BoneDataArr : register(b4) {
    BoneDataArr bone_data_arr;
}
```

## 4. PBR 着色器实现

### 4.1 PBRTest.hlsl

```hlsl
// 材质数据
struct MaterialData {
    float2 u_textureTiling;   // 纹理平铺
    float2 u_textureOffset;   // 纹理偏移
    float4 u_color;           // 基础颜色
};

cbuffer Material : register(b10) {
    MaterialData materialData;
};

// 纹理绑定
Texture2D u_albedo : register(t101);    // 反照率
Texture2D u_normal : register(t102);    // 法线
Texture2D u_metallic : register(t103);  // 金属度
Texture2D u_roughness : register(t104); // 粗糙度
Texture2D u_aO : register(t105);        // 环境光遮蔽

// 顶点着色器
Pixel mainVS(Vertex_PosUvNorTan input) {
    Pixel output;
    
    input.position.w = 1.0f;
    output.fragPos = mul(input.position, buffer_pass.transform).xyz;
    output.position = mul(input.position, buffer_pass.transform);
    output.position = mul(output.position, buffer_rendererPath.view_projection_unjittered);
    output.normal = mul(float4(input.normal, 0), buffer_pass.transform).xyz;
    output.tangent = mul(float4(input.tangent, 0), buffer_pass.transform).xyz;
    output.uv = input.uv;
    
    return output;
}

// 像素着色器
float4 mainPS(Pixel input) : SV_Target {
    // 1. 纹理采样
    float2 g_TexCoords = materialData.u_textureOffset + 
        float2((input.uv.x * materialData.u_textureTiling.x) % 1.0, 
               (input.uv.y * materialData.u_textureTiling.y) % 1.0);
    
    float4 albedo = u_albedo.Sample(samplers[sampler_point_wrap], g_TexCoords);
    float3 albedoLinear = srgb_to_linear(albedo.xyz);
    albedo = float4(albedoLinear * materialData.u_color.rgb, albedo.a * materialData.u_color.a);
    
    float metallic = u_metallic.Sample(samplers[sampler_point_wrap], g_TexCoords).x;
    float perceptualRoughness = 1 - u_roughness.Sample(samplers[sampler_point_wrap], g_TexCoords).x;
    float roughness = perceptualRoughness * perceptualRoughness;
    float squareRoughness = roughness * roughness;
    float lerpSquareRoughness = pow(lerp(0.002, 1, roughness), 2);
    
    // 2. 法线计算
    float3 sampleNormal = u_normal.Sample(samplers[sampler_point_wrap], g_TexCoords).xyz;
    float3 binormal = cross(normalize(input.normal), normalize(input.tangent));
    float3x3 rotation = float3x3(input.tangent, binormal, sampleNormal);
    float3 normal = mul(rotation, sampleNormal);
    normal = normalize(input.normal);
    
    // 3. 视角方向
    float3 viewDir = normalize(buffer_rendererPath.camera_position.xyz - input.fragPos.xyz);
    float nv = max(saturate(dot(normal, viewDir)), 0.000001);
    
    // 4. 光照计算
    uint lightCount = (uint)pass_get_f3_value2().z;
    float3 lightSum = float3(0, 0, 0);
    
    for (int index = 0; index < lightCount; index++) {
        lightSum += CalcOneLightColorPBR(input.fragPos, albedo, metallic, squareRoughness, 
                                          lerpSquareRoughness, normal, viewDir, nv, index);
    }
    
    // 5. 最终颜色
    float3 color = linear_to_srgb(lightSum + float3(0.03, 0.03, 0.03) * albedo.xyz);
    
    return float4(color.x, color.y, color.z, albedo.a);
}
```

### 4.2 PBR 光照计算

```hlsl
float3 CalcOneLightColorPBR(
    float3 fragPos,
    float4 albedo, float metallic, float squareRoughness, float lerpSquareRoughness,
    float3 normal, float3 viewDir, float nv,
    int light_index)
{
    float PI = 3.14;
    float3 colorSpaceDielectricSpecRgb = float3(0.04, 0.04, 0.04);
    
    LightBufferData lightBufferData = buffer_lights[light_index];
    float3 light_to_pixel = lightBufferData.compute_direction(fragPos);
    float3 lightDir = -light_to_pixel;
    float attenuation = lightBufferData.compute_attenuation(fragPos);
    float3 halfVector = normalize(lightDir + viewDir);
    
    float nl = max(saturate(dot(normal, lightDir)), 0.000001);
    float lh = max(saturate(dot(lightDir, halfVector)), 0.000001);
    float vh = max(saturate(dot(viewDir, halfVector)), 0.000001);
    float nh = max(saturate(dot(normal, halfVector)), 0.000001);
    
    float3 randiance = lightBufferData.color.xyz * lightBufferData.intensity * attenuation * nl;
    
    // 阴影计算
    float shadow = ShadowCalculation2(normal, fragPos, light_index);
    randiance *= shadow;
    
    // D (法线分布函数)
    float D = lerpSquareRoughness / (pow((pow(nh, 2) * (lerpSquareRoughness - 1) + 1), 2) * PI);
    
    // F (Fresnel)
    float3 F0 = lerp(colorSpaceDielectricSpecRgb, albedo.xyz, metallic);
    float3 F = F0 + (1 - F0) * exp2((-5.55473 * vh - 6.98316) * vh);
    
    // G (几何遮蔽)
    float kInDirectLight = pow(squareRoughness + 1, 2) / 8;
    float GLeft = nl / lerp(nl, 1, kInDirectLight);
    float GRight = nv / lerp(nv, 1, kInDirectLight);
    float G = GLeft * GRight;
    
    // 漫反射
    float3 kd = (1 - F) * (1 - metallic);
    float3 diffColor = kd * albedo.xyz / PI;
    
    // 镜面反射
    float3 specColor = (D * G * F) / (4 * nv * nl);
    
    // 最终颜色
    float3 directLightResult = (diffColor + specColor) * randiance;
    
    return directLightResult;
}
```

## 5. 阴影映射

### 5.1 阴影计算

```hlsl
float ShadowCalculation2(float3 fragWorldNormal, float3 fragWorldPos, int lightIndex) {
    float shadow = 1.0f;
    LightBufferData light = buffer_lights[lightIndex];
    
    if (light.light_has_shadows()) {
        float distance_to_pixel = length(fragWorldPos - light.position);
        if (distance_to_pixel < light.range) {
            float3 light_to_pixel = light.compute_direction(fragWorldPos);
            float light_n_dot_l = saturate(dot(fragWorldNormal, -light_to_pixel));
        
            float2 resolution = light.compute_resolution();
            float2 texel_size = 1.0f / resolution;
            float3 normal_offset_bias = fragWorldNormal * (1.0f - saturate(light_n_dot_l)) * texel_size.x;
            float3 position_world = fragWorldPos + normal_offset_bias;
        
            if (light.light_is_point()) {
                // 点光源阴影
                uint light_slice_index = dot(light.forward, light_to_pixel) < 0.0f;
                uint slice_index = 2 * lightIndex + light_slice_index;
                float3 pos_view = mul(float4(position_world, 1.0f), light.view_projection[light_slice_index]).xyz;
                float3 ndc = project_onto_paraboloid(pos_view, 0.01, light.range);
                float3 sample_coords = float3(ndc_to_uv(ndc.xy), slice_index);
                shadow = SampleShadowMap(light, sample_coords, ndc.z);
            } else {
                // 方向光/聚光灯阴影
                uint slice_index = 2 * lightIndex;
                float3 pos_ndc = world_to_ndc(position_world, light.view_projection[0]);
                float2 pos_uv = ndc_to_uv(pos_ndc);
                
                if (is_valid_uv(pos_uv)) {
                    shadow = SampleShadowMap(light, float3(pos_uv, slice_index), pos_ndc.z);
                    
                    // 级联阴影混合
                    float cascade_fade = saturate((max(abs(pos_ndc.x), abs(pos_ndc.y)) - g_shadow_cascade_blend_threshold) * 4.0f);
                    if (light.light_is_directional() && cascade_fade > 0.0f) {
                        float3 pos_ndc_far = world_to_ndc(position_world, light.view_projection[1]);
                        float2 pos_uv_far = ndc_to_uv(pos_ndc_far);
                        float shadow_far = SampleShadowMap(light, float3(pos_uv_far, slice_index + 1), pos_ndc_far.z);
                        shadow = lerp(shadow, shadow_far, cascade_fade);
                    }
                }
            }
        }
    }
    
    return shadow;
}
```

## 6. 颜色空间转换

### 6.1 sRGB ↔ Linear

```hlsl
// sRGB 到 Linear
float3 srgb_to_linear(float3 srgb) {
    return pow(srgb, float3(2.2, 2.2, 2.2));
}

// Linear 到 sRGB
float3 linear_to_srgb(float3 linear) {
    return pow(linear, float3(1.0/2.2, 1.0/2.2, 1.0/2.2));
}
```

## 7. 材质属性标志

### 7.1 材质属性检查

```hlsl
// G-Buffer 纹理属性
bool has_single_texture_roughness_metalness() { return buffer_material.properties & uint(1U << 0); }
bool has_texture_height() { return buffer_material.properties & uint(1U << 1); }
bool has_texture_normal() { return buffer_material.properties & uint(1U << 2); }
bool has_texture_albedo() { return buffer_material.properties & uint(1U << 3); }
bool has_texture_roughness() { return buffer_material.properties & uint(1U << 4); }
bool has_texture_metalness() { return buffer_material.properties & uint(1U << 5); }
bool has_texture_alpha_mask() { return buffer_material.properties & uint(1U << 6); }
bool has_texture_emissive() { return buffer_material.properties & uint(1U << 7); }
bool has_texture_occlusion() { return buffer_material.properties & uint(1U << 8); }
bool material_is_terrain() { return buffer_material.properties & uint(1U << 9); }
bool material_is_water() { return buffer_material.properties & uint(1U << 10); }
```

### 7.2 帧选项检查

```hlsl
bool is_taa_enabled() { return any(buffer_frame.taa_jitter_current); }
bool is_ssr_enabled() { return buffer_frame.options & uint(1U << 0); }
bool is_ssgi_enabled() { return buffer_frame.options & uint(1U << 1); }
bool is_screen_space_shadows_enabled() { return buffer_frame.options & uint(1U << 2); }
bool is_fog_enabled() { return buffer_frame.options & uint(1U << 3); }
bool is_fog_volumetric_enabled() { return buffer_frame.options & uint(1U << 4); }
```

## 8. 着色器编译流程

### 8.1 编译流程

```
HLSL 源码 (.hlsl)
    │
    ├── DXCompiler (dxc)
    │       ├── -spirv                    // 输出 SPIR-V
    │       ├── -fspv-target-env=vulkan1.3 // Vulkan 1.3
    │       └── -E main                   // 入口点
    │
    └── SPIR-V (.spv)
            │
            ├── VkShaderModule 创建
            └── SPIR-V 反射 (描述符绑定)
```

### 8.2 DXCompiler 参数

```cpp
// 典型编译参数
std::vector<LPCWSTR> arguments = {
    L"-spirv",                        // 生成 SPIR-V
    L"-fspv-target-env=vulkan1.3",    // Vulkan 1.3 目标
    L"-E", L"mainVS",                 // 入口点
    L"-T", L"vs_6_0",                 // 着色器模型
    L"-Zi",                           // 调试信息
    L"-Od",                           // 禁用优化 (调试)
};
```

## 9. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 模块化设计 | Common/ 目录 | 公共代码复用 |
| 常量缓冲区布局 | common_buffers.hlsl | 数据传递标准化 |
| 标志位模式 | properties/flags | 紧凑的选项存储 |
| 函数封装 | CalcOneLightColorPBR | 光照计算复用 |

## 10. 着色器编写规范

### 10.1 命名约定

```hlsl
// 常量缓冲区: buffer_xxx
cbuffer BufferFrame : register(b0) { FrameBufferData buffer_frame; };

// 纹理: u_xxx 或 tex_xxx
Texture2D u_albedo : register(t101);

// 采样器: samplers[枚举]
SamplerState samplers[] : register(s0);

// 结构体: XxxData
struct MaterialData { ... };
```

### 10.2 寄存器分配

| 寄存器类型 | 范围 | 用途 |
|------------|------|------|
| b0-b9 | 引擎常量缓冲 | Frame, Light, Material |
| b10+ | 材质常量缓冲 | 自定义材质数据 |
| t0-t99 | 引擎纹理 | 系统纹理 |
| t100+ | 材质纹理 | 材质贴图 |
| s0-s9 | 采样器 | 预定义采样器 |
| u0-u9 | UAV | 结构化缓冲 |

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
