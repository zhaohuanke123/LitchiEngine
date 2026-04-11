# 渲染管线实现分析

## 1. 模块概述

渲染系统位于 `Engine/Source/Runtime/Function/Renderer/`，是引擎的核心渲染模块。采用分层架构设计：

```
Renderer (高层渲染器)
    └── RendererPath (渲染路径/视图)
            └── Pass_* (渲染 Pass)
                    └── RHI_Device (硬件抽象层)
```

## 2. Renderer - 高层渲染器

### 2.1 核心职责

- 管理所有渲染资源 (着色器、常量缓冲、渲染目标、采样器)
- 调度渲染管线执行
- 提供渲染 Pass 实现
- 管理帧资源和同步

### 2.2 类定义

```cpp
class Renderer {
public:
    // 生命周期
    static void Initialize();
    static void Shutdown();
    static void Tick(const uint64_t frame_count);

    // 渲染 Pass
    static void Pass_ShadowMaps(RendererPath* renderer_path);
    static void Pass_SkyBox(RendererPath* renderer_path);
    static void Pass_ForwardPass(RendererPath* renderer_path);
    static void Pass_UIPass(RendererPath* renderer_path);
    static void Pass_Debug(RendererPath* renderer_path);

    // 资源获取
    static RHI_Shader* GetShader(Renderer_Shader shader);
    static RHI_ConstantBuffer* GetCb(Renderer_BindingsCb slot);
    static RHI_RenderTarget* GetRenderTarget(Renderer_RenderTarget target);
    static RHI_Sampler* GetSampler(Renderer_Sampler sampler);

    // 常量缓冲更新
    static void UpdateCbFrame(const uint64_t frame_count);
    static void UpdateCbCamera(RendererPath* renderer_path);
    static void UpdateCbLights(RendererPath* renderer_path);

private:
    // 着色器资源
    static std::array<RHI_Shader*, Renderer_Shader::Max> m_shaders;

    // 常量缓冲
    static std::array<RHI_ConstantBuffer*, Renderer_BindingsCb::Max> m_cbs;

    // 渲染目标
    static std::array<RHI_RenderTarget*, Renderer_RenderTarget::Max> m_render_targets;

    // 采样器
    static std::array<RHI_Sampler*, Renderer_Sampler::Max> m_samplers;
};
```

### 2.3 初始化流程

```cpp
void Renderer::Initialize() {
    // 1. 初始化 RHI 设备
    RHI_Device::Initialize();

    // 2. 创建采样器
    m_samplers[Renderer_Sampler::PointClamp] = CreateSampler(...);
    m_samplers[Renderer_Sampler::BilinearClamp] = CreateSampler(...);
    m_samplers[Renderer_Sampler::TrilinearWrap] = CreateSampler(...);
    m_samplers[Renderer_Sampler::TrilinearWrapAnisotropy] = CreateSampler(...);
    m_samplers[Renderer_Sampler::ShadowMap] = CreateSampler(...);

    // 3. 创建渲染目标
    m_render_targets[Renderer_RenderTarget::SceneColor] = CreateRenderTarget(...);
    m_render_targets[Renderer_RenderTarget::SceneDepth] = CreateRenderTarget(...);
    m_render_targets[Renderer_RenderTarget::ShadowMap] = CreateRenderTarget(...);
    m_render_targets[Renderer_RenderTarget::ShadowMapCube] = CreateRenderTarget(...);

    // 4. 创建常量缓冲
    m_cbs[Renderer_BindingsCb::Frame] = CreateConstantBuffer(sizeof(Cb_Frame));
    m_cbs[Renderer_BindingsCb::Camera] = CreateConstantBuffer(sizeof(Cb_Camera));
    m_cbs[Renderer_BindingsCb::Lights] = CreateConstantBuffer(sizeof(Cb_Lights));
    m_cbs[Renderer_BindingsCb::Object] = CreateConstantBuffer(sizeof(Cb_Object));

    // 5. 加载着色器
    m_shaders[Renderer_Shader::ForwardPBR] = LoadShader("PBRTest.hlsl");
    m_shaders[Renderer_Shader::ShadowMap] = LoadShader("ShadowMap.hlsl");
    m_shaders[Renderer_Shader::SkyBox] = LoadShader("SkyBox.hlsl");
    m_shaders[Renderer_Shader::UI] = LoadShader("UI.hlsl");
    // ...
}
```

### 2.4 主渲染循环

```cpp
void Renderer::Tick(const uint64_t frame_count) {
    // 1. 更新帧常量缓冲
    UpdateCbFrame(frame_count);

    // 2. 渲染内置场景视图 (编辑器场景视图)
    Render4BuildInSceneView();

    // 3. 渲染内置游戏视图 (运行时游戏视图)
    Render4BuildInGameView();

    // 4. 渲染内置资产视图 (资产预览)
    Render4BuildInAssetView();

    // 5. 处理延迟删除队列
    RHI_Device::DeletionQueueParse();
}

void Renderer::Render4BuildInSceneView() {
    RendererPath* path = m_scene_view_path;

    // 更新相机数据
    UpdateCbCamera(path);

    // 更新光源数据
    UpdateCbLights(path);

    // 执行渲染 Pass 序列
    Pass_SkyBox(path);
    Pass_ShadowMaps(path);
    Pass_ForwardPass(path);
    Pass_UIPass(path);
    Pass_Debug(path);
}
```

## 3. RendererPath - 渲染路径

### 3.1 设计目的

RendererPath 代表一个独立的渲染视图/上下文，每个视图有自己的：
- 相机
- 渲染目标
- 可渲染对象列表
- 光源数据

### 3.2 类定义

```cpp
// 渲染路径类型
enum class RendererPathType : uint8_t {
    SceneView,   // 编辑器场景视图
    GameView,    // 游戏视图
    AssetView,   // 资产预览视图
    Custom       // 自定义视图
};

// 光源数据
struct RendererLightData {
    Light* light = nullptr;
    Matrix view_matrix;
    Matrix projection_matrix;
    BoundingBox bounding_box;
};

// 光源组
struct RendererLightGroup {
    std::vector<RendererLightData> directional_lights;
    std::vector<RendererLightData> point_lights;
    std::vector<RendererLightData> spot_lights;
};

// 渲染路径
class RendererPath {
public:
    // 类型
    RendererPathType GetType() const { return m_type; }

    // 相机
    Camera* GetCamera() const { return m_camera; }
    void SetCamera(Camera* camera) { m_camera = camera; }

    // 渲染目标
    RHI_RenderTarget* GetRenderTarget() const { return m_render_target; }
    void SetRenderTarget(RHI_RenderTarget* target) { m_render_target = target; }

    // 可渲染对象
    void AddRenderable(MeshRenderer* renderer);
    void RemoveRenderable(MeshRenderer* renderer);
    const std::vector<MeshRenderer*>& GetRenderables() const { return m_renderables; }

    // 光源管理
    void AddLight(Light* light);
    void RemoveLight(Light* light);
    RendererLightGroup* GetLightGroup() { return &m_light_group; }

    // 视锥剔除
    void PerformFrustumCulling();
    bool IsVisible(MeshRenderer* renderer) const;

    // 排序
    void SortRenderables();

private:
    RendererPathType m_type = RendererPathType::Custom;
    Camera* m_camera = nullptr;
    RHI_RenderTarget* m_render_target = nullptr;
    std::vector<MeshRenderer*> m_renderables;
    RendererLightGroup m_light_group;
    Frustum m_frustum;
};
```

### 3.3 视锥剔除

```cpp
void RendererPath::PerformFrustumCulling() {
    // 从相机构建视锥体
    m_frustum = Frustum(m_camera->GetViewMatrix() * m_camera->GetProjectionMatrix());

    // 遍历所有可渲染对象
    for (auto* renderer : m_renderables) {
        // 获取世界包围盒
        BoundingBox world_bounds = renderer->GetBoundsWorld();

        // 视锥体相交测试
        renderer->SetVisible(m_frustum.Intersects(world_bounds));
    }
}
```

### 3.4 排序策略

```cpp
void RendererPath::SortRenderables() {
    std::sort(m_renderables.begin(), m_renderables.end(),
        [this](MeshRenderer* a, MeshRenderer* b) {
            // 1. 按材质排序 (减少状态切换)
            if (a->GetMaterial() != b->GetMaterial()) {
                return a->GetMaterial()->GetID() < b->GetMaterial()->GetID();
            }

            // 2. 按距离排序 (透明物体从后往前)
            float dist_a = Vector3::DistanceSquared(m_camera->GetPosition(), a->GetPosition());
            float dist_b = Vector3::DistanceSquared(m_camera->GetPosition(), b->GetPosition());

            return dist_a > dist_b;  // 远到近
        });
}
```

## 4. 渲染 Pass 实现

### 4.1 Pass_ShadowMaps - 阴影渲染

```cpp
void Renderer::Pass_ShadowMaps(RendererPath* renderer_path) {
    RendererLightGroup* light_group = renderer_path->GetLightGroup();

    // 遍历所有方向光
    for (auto& light_data : light_group->directional_lights) {
        Light* light = light_data.light;
        if (!light->CastShadows()) continue;

        // 1. 设置阴影贴图渲染目标
        RHI_RenderTarget* shadow_target = m_render_targets[Renderer_RenderTarget::ShadowMap];
        shadow_target->SetRenderTarget();

        // 2. 配置 PSO
        RHI_PipelineState pso;
        pso.render_target_depth_texture = shadow_target->GetDepthTexture();
        pso.shader_vertex = m_shaders[Renderer_Shader::ShadowMap];
        pso.depth_stencil_state = m_depth_stencil_shadow;
        pso.rasterizer_state = m_rasterizer_shadow;

        // 3. 设置光源视图投影矩阵
        Cb_Shadow cb_shadow;
        cb_shadow.view_projection = light_data.view_matrix * light_data.projection_matrix;
        m_cbs[Renderer_BindingsCb::Shadow]->Update(&cb_shadow);

        // 4. 渲染深度
        RHI_CommandList* cmd = RHI_Device::CmdImmediateBegin(RHI_Queue_Type::Graphics);
        cmd->Begin();
        cmd->SetPipelineState(pso);

        for (auto* renderer : renderer_path->GetRenderables()) {
            if (!renderer->IsVisible() || !renderer->CastShadows()) continue;

            // 更新对象常量缓冲
            Cb_Object cb_object;
            cb_object.model = renderer->GetTransform()->GetWorldMatrix();
            m_cbs[Renderer_BindingsCb::Object]->Update(&cb_object);

            // 绘制
            DrawMesh(cmd, renderer->GetMesh());
        }

        cmd->End();
        RHI_Device::CmdImmediateSubmit(cmd);
    }
}
```

### 4.2 Pass_SkyBox - 天空盒渲染

```cpp
void Renderer::Pass_SkyBox(RendererPath* renderer_path) {
    Camera* camera = renderer_path->GetCamera();

    // 1. 配置 PSO
    RHI_PipelineState pso;
    pso.render_target_color_textures[0] = renderer_path->GetRenderTarget()->GetColorTexture();
    pso.render_target_depth_texture = renderer_path->GetRenderTarget()->GetDepthTexture();
    pso.shader_vertex = m_shaders[Renderer_Shader::SkyBox];
    pso.shader_pixel = m_shaders[Renderer_Shader::SkyBox];
    pso.depth_stencil_state = m_depth_stencil_skybox;  // 深度测试 LessEqual
    pso.rasterizer_state = m_rasterizer_skybox;        // CullFront

    // 2. 更新相机数据
    Cb_SkyBox cb_skybox;
    cb_skybox.view = Matrix::RemoveTranslation(camera->GetViewMatrix());
    cb_skybox.projection = camera->GetProjectionMatrix();
    m_cbs[Renderer_BindingsCb::SkyBox]->Update(&cb_skybox);

    // 3. 绘制立方体
    RHI_CommandList* cmd = RHI_Device::CmdImmediateBegin(RHI_Queue_Type::Graphics);
    cmd->Begin();
    cmd->SetPipelineState(pso);
    cmd->SetTexture(0, m_skybox_cubemap);
    cmd->Draw(36);  // 立方体 6 面 * 2 三角形 * 3 顶点
    cmd->End();
    RHI_Device::CmdImmediateSubmit(cmd);
}
```

### 4.3 Pass_ForwardPass - 前向渲染

```cpp
void Renderer::Pass_ForwardPass(RendererPath* renderer_path) {
    Camera* camera = renderer_path->GetCamera();
    RendererLightGroup* light_group = renderer_path->GetLightGroup();

    // 1. 更新光源常量缓冲
    UpdateCbLights(renderer_path);

    // 2. 配置 PSO
    RHI_PipelineState pso;
    pso.render_target_color_textures[0] = renderer_path->GetRenderTarget()->GetColorTexture();
    pso.render_target_depth_texture = renderer_path->GetRenderTarget()->GetDepthTexture();
    pso.shader_vertex = m_shaders[Renderer_Shader::ForwardPBR];
    pso.shader_pixel = m_shaders[Renderer_Shader::ForwardPBR];
    pso.depth_stencil_state = m_depth_stencil_default;
    pso.rasterizer_state = m_rasterizer_default;

    // 3. 执行视锥剔除和排序
    renderer_path->PerformFrustumCulling();
    renderer_path->SortRenderables();

    // 4. 渲染所有可见对象
    RHI_CommandList* cmd = RHI_Device::CmdImmediateBegin(RHI_Queue_Type::Graphics);
    cmd->Begin();
    cmd->SetPipelineState(pso);

    for (auto* renderer : renderer_path->GetRenderables()) {
        if (!renderer->IsVisible()) continue;

        // 更新对象常量缓冲
        Cb_Object cb_object;
        cb_object.model = renderer->GetTransform()->GetWorldMatrix();
        cb_object.normal = cb_object.model.Inverted().Transposed();
        m_cbs[Renderer_BindingsCb::Object]->Update(&cb_object);

        // 绑定材质
        Material* material = renderer->GetMaterial();
        if (material) {
            BindMaterial(cmd, material);
        }

        // 绘制网格
        DrawMesh(cmd, renderer->GetMesh());
    }

    cmd->End();
    RHI_Device::CmdImmediateSubmit(cmd);
}
```

### 4.4 Pass_UIPass - UI 渲染

```cpp
void Renderer::Pass_UIPass(RendererPath* renderer_path) {
    // 1. 配置 PSO (正交投影)
    RHI_PipelineState pso;
    pso.render_target_color_textures[0] = renderer_path->GetRenderTarget()->GetColorTexture();
    pso.shader_vertex = m_shaders[Renderer_Shader::UI];
    pso.shader_pixel = m_shaders[Renderer_Shader::UI];
    pso.blend_state = m_blend_alpha;  // Alpha 混合
    pso.depth_stencil_state = m_depth_stencil_off;  // 无深度测试

    // 2. 渲染 UI 元素
    RHI_CommandList* cmd = RHI_Device::CmdImmediateBegin(RHI_Queue_Type::Graphics);
    cmd->Begin();
    cmd->SetPipelineState(pso);

    // 渲染文本
    for (auto* text : GetUITexts()) {
        DrawUIText(cmd, text);
    }

    // 渲染图片
    for (auto* image : GetUIImages()) {
        DrawUIImage(cmd, image);
    }

    cmd->End();
    RHI_Device::CmdImmediateSubmit(cmd);
}
```

## 5. 光源系统

### 5.1 光源类型

```cpp
enum class LightType : uint8_t {
    Directional,  // 方向光
    Point,        // 点光源
    Spot          // 聚光灯
};
```

### 5.2 光源数据结构

```cpp
// 方向光数据 (GPU)
struct LightDirectional {
    Vector4 direction;      // 方向 (世界空间)
    Vector4 color;          // 颜色 + 强度
    Matrix view_projection; // 阴影视图投影矩阵
    Vector4 shadow_params;  // 阴影参数
};

// 点光源数据 (GPU)
struct LightPoint {
    Vector4 position;       // 位置 (世界空间)
    Vector4 color;          // 颜色 + 强度
    Vector4 params;         // 范围, 衰减参数
};

// 聚光灯数据 (GPU)
struct LightSpot {
    Vector4 position;       // 位置 (世界空间)
    Vector4 direction;      // 方向
    Vector4 color;          // 颜色 + 强度
    Vector4 params;         // 内角, 外角, 范围
};
```

### 5.3 光源常量缓冲

```cpp
// HLSL 定义
cbuffer Cb_Lights : register(b2)
{
    uint light_directional_count;
    uint light_point_count;
    uint light_spot_count;
    uint padding;

    LightDirectional lights_directional[4];
    LightPoint lights_point[64];
    LightSpot lights_spot[16];
};
```

### 5.4 光源更新

```cpp
void Renderer::UpdateCbLights(RendererPath* renderer_path) {
    RendererLightGroup* group = renderer_path->GetLightGroup();
    Cb_Lights cb_lights = {};

    // 方向光
    cb_lights.light_directional_count = std::min((uint32_t)group->directional_lights.size(), 4u);
    for (uint32_t i = 0; i < cb_lights.light_directional_count; i++) {
        auto& light_data = group->directional_lights[i];
        Light* light = light_data.light;

        cb_lights.lights_directional[i].direction = Vector4(-light->GetForward(), 0.0f);
        cb_lights.lights_directional[i].color = Vector4(light->GetColor(), light->GetIntensity());
        cb_lights.lights_directional[i].view_projection = light_data.view_matrix * light_data.projection_matrix;
    }

    // 点光源
    cb_lights.light_point_count = std::min((uint32_t)group->point_lights.size(), 64u);
    for (uint32_t i = 0; i < cb_lights.light_point_count; i++) {
        auto& light_data = group->point_lights[i];
        Light* light = light_data.light;

        cb_lights.lights_point[i].position = Vector4(light->GetPosition(), 1.0f);
        cb_lights.lights_point[i].color = Vector4(light->GetColor(), light->GetIntensity());
        cb_lights.lights_point[i].params = Vector4(light->GetRange(), light->GetAttenuation(), 0, 0);
    }

    // 更新常量缓冲
    m_cbs[Renderer_BindingsCb::Lights]->Update(&cb_lights);
}
```

## 6. PBR 材质系统

### 6.1 材质属性

```cpp
class Material : public ScriptObject {
public:
    // PBR 属性
    Color GetAlbedo() const;
    float GetMetallic() const;
    float GetRoughness() const;
    float GetAO() const;

    // 纹理
    RHI_Texture* GetAlbedoMap() const;
    RHI_Texture* GetNormalMap() const;
    RHI_Texture* GetMetallicMap() const;
    RHI_Texture* GetRoughnessMap() const;
    RHI_Texture* GetAOMap() const;

    // 着色器
    MaterialShader* GetShader() const;

    // 渲染状态
    BlendMode GetBlendMode() const;
    bool IsDoubleSided() const;
};
```

### 6.2 PBR 着色器参数

```cpp
// HLSL 材质常量缓冲
cbuffer Cb_Material : register(b3)
{
    float4 albedo_color;
    float metallic;
    float roughness;
    float ao;
    float padding;
};

// 纹理绑定
Texture2D albedoMap    : register(t0);
Texture2D normalMap    : register(t1);
Texture2D metallicMap  : register(t2);
Texture2D roughnessMap : register(t3);
Texture2D aoMap        : register(t4);
```

### 6.3 PBR 光照计算

```hlsl
// PBRTest.hlsl - 核心 PBR 计算

// 法线分布函数 (GGX/Trowbridge-Reitz)
float DistributionGGX(float3 N, float3 H, float roughness) {
    float a = roughness * roughness;
    float a2 = a * a;
    float NdotH = max(dot(N, H), 0.0);
    float NdotH2 = NdotH * NdotH;

    float nom = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;

    return nom / denom;
}

// 几何遮蔽函数 (Schlick-GGX)
float GeometrySchlickGGX(float NdotV, float roughness) {
    float r = (roughness + 1.0);
    float k = (r * r) / 8.0;

    float nom = NdotV;
    float denom = NdotV * (1.0 - k) + k;

    return nom / denom;
}

// Fresnel 方程 (Schlick 近似)
float3 FresnelSchlick(float cosTheta, float3 F0) {
    return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}

// PBR 光照
float3 CalculatePBR(LightDirectional light, float3 N, float3 V, float3 albedo, float metallic, float roughness) {
    float3 L = normalize(-light.direction.xyz);
    float3 H = normalize(V + L);

    // Fresnel 反射率
    float3 F0 = lerp(float3(0.04, 0.04, 0.04), albedo, metallic);
    float3 F = FresnelSchlick(max(dot(H, V), 0.0), F0);

    // 法线分布
    float D = DistributionGGX(N, H, roughness);

    // 几何遮蔽
    float G = GeometrySchlickGGX(max(dot(N, V), 0.0), roughness) *
              GeometrySchlickGGX(max(dot(N, L), 0.0), roughness);

    // 镜面反射
    float3 numerator = D * G * F;
    float denominator = 4.0 * max(dot(N, V), 0.0) * max(dot(N, L), 0.0) + 0.0001;
    float3 specular = numerator / denominator;

    // 漫反射
    float3 kD = (1.0 - F) * (1.0 - metallic);
    float3 diffuse = kD * albedo / PI;

    // 最终颜色
    float NdotL = max(dot(N, L), 0.0);
    return (diffuse + specular) * light.color.rgb * light.color.a * NdotL;
}
```

## 7. 常量缓冲布局

### 7.1 Frame 常量缓冲

```cpp
// Cb_Frame - 每帧更新一次
struct Cb_Frame {
    float time;              // 游戏时间
    float delta_time;        // 帧间隔
    uint32_t frame_count;    // 帧计数
    uint32_t padding;
};
```

### 7.2 Camera 常量缓冲

```cpp
// Cb_Camera - 每视图更新一次
struct Cb_Camera {
    Matrix view;
    Matrix projection;
    Matrix view_projection;
    Matrix view_projection_inv;
    Vector3 position;
    float near_plane;
    float far_plane;
    Vector2 screen_size;
};
```

### 7.3 Object 常量缓冲

```cpp
// Cb_Object - 每对象更新一次
struct Cb_Object {
    Matrix model;            // 世界矩阵
    Matrix normal;           // 法线矩阵 (模型矩阵逆转置)
};
```

## 8. 渲染目标管理

### 8.1 预定义渲染目标

```cpp
enum class Renderer_RenderTarget : uint8_t {
    SceneColor,       // 场景颜色
    SceneDepth,       // 场景深度
    ShadowMap,        // 阴影贴图 (2D)
    ShadowMapCube,    // 阴影贴图 (立方体)
    Max
};
```

### 8.2 渲染目标创建

```cpp
void Renderer::CreateRenderTargets() {
    // 场景颜色
    RHI_Texture_Desc color_desc = {};
    color_desc.width = m_screen_width;
    color_desc.height = m_screen_height;
    color_desc.format = RHI_Format::R16G16B16A16_Float;
    color_desc.flags = RHI_Texture_Rtv | RHI_Texture_Srv;
    m_render_targets[Renderer_RenderTarget::SceneColor] = CreateRenderTarget(color_desc);

    // 场景深度
    RHI_Texture_Desc depth_desc = {};
    depth_desc.width = m_screen_width;
    depth_desc.height = m_screen_height;
    depth_desc.format = RHI_Format::D32_Float;
    depth_desc.flags = RHI_Texture_Dsv;
    m_render_targets[Renderer_RenderTarget::SceneDepth] = CreateRenderTarget(depth_desc);

    // 阴影贴图
    RHI_Texture_Desc shadow_desc = {};
    shadow_desc.width = 2048;
    shadow_desc.height = 2048;
    shadow_desc.format = RHI_Format::D32_Float;
    shadow_desc.flags = RHI_Texture_Dsv | RHI_Texture_Srv;
    m_render_targets[Renderer_RenderTarget::ShadowMap] = CreateRenderTarget(shadow_desc);
}
```

## 9. 采样器配置

### 9.1 预定义采样器

```cpp
enum class Renderer_Sampler : uint8_t {
    PointClamp,               // 点采样, 钳制
    BilinearClamp,            // 双线性, 钳制
    TrilinearWrap,            // 三线性, 环绕
    TrilinearWrapAnisotropy,  // 三线性 + 各向异性, 环绕
    ShadowMap,                // 阴影贴图采样
    Max
};
```

### 9.2 采样器创建

```cpp
void Renderer::CreateSamplers() {
    // 点采样
    RHI_Sampler_Desc point_desc = {};
    point_desc.filter = RHI_Filter::Min_Mag_Mip_Point;
    point_desc.address_u = RHI_Address_Mode::Clamp;
    point_desc.address_v = RHI_Address_Mode::Clamp;
    m_samplers[Renderer_Sampler::PointClamp] = CreateSampler(point_desc);

    // 三线性各向异性
    RHI_Sampler_Desc aniso_desc = {};
    aniso_desc.filter = RHI_Filter::Min_Mag_Mip_Linear;
    aniso_desc.address_u = RHI_Address_Mode::Wrap;
    aniso_desc.address_v = RHI_Address_Mode::Wrap;
    aniso_desc.max_anisotropy = 16;
    m_samplers[Renderer_Sampler::TrilinearWrapAnisotropy] = CreateSampler(aniso_desc);

    // 阴影贴图
    RHI_Sampler_Desc shadow_desc = {};
    shadow_desc.filter = RHI_Filter::Min_Mag_Mip_Point;
    shadow_desc.address_u = RHI_Address_Mode::Clamp_Border;
    shadow_desc.address_v = RHI_Address_Mode::Clamp_Border;
    shadow_desc.border_color = Color(1, 1, 1, 1);  // 阴影边界
    shadow_desc.comparison_func = RHI_Comparison_Func::Less_Equal;
    m_samplers[Renderer_Sampler::ShadowMap] = CreateSampler(shadow_desc);
}
```

## 10. 渲染流程总结

### 10.1 完整渲染流程

```
Frame Start
    │
    ├── UpdateCbFrame()
    │
    ├── For each RendererPath:
    │       │
    │       ├── UpdateCbCamera()
    │       ├── UpdateCbLights()
    │       │
    │       ├── Pass_SkyBox()        // 天空盒
    │       │       └── Draw cube with cubemap
    │       │
    │       ├── Pass_ShadowMaps()    // 阴影贴图
    │       │       └── For each shadow-casting light:
    │       │               └── Render depth from light view
    │       │
    │       ├── Pass_ForwardPass()   // 前向渲染
    │       │       ├── Frustum culling
    │       │       ├── Sort renderables
    │       │       └── For each visible object:
    │       │               └── Draw with PBR material
    │       │
    │       ├── Pass_UIPass()        // UI
    │       │       └── Draw UI elements
    │       │
    │       └── Pass_Debug()         // 调试绘制
    │               └── Draw debug geometry
    │
    └── DeletionQueueParse()
        │
Frame End
```

### 10.2 性能优化策略

| 优化技术 | 应用位置 | 说明 |
|----------|----------|------|
| 视锥剔除 | RendererPath | 跳过不可见对象 |
| 排序 | RendererPath | 按材质排序减少状态切换 |
| 实例化 | ForwardPass | 相同网格合并绘制 |
| 管线缓存 | RHI_Device | 避免重复创建 PSO |
| 延迟删除 | RHI_Device | 安全释放 GPU 资源 |
| 常量缓冲更新 | 各 Pass | 只更新变化的部分 |

## 11. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 单例模式 | Renderer | 全局渲染管理 |
| 策略模式 | RendererPath | 不同渲染路径切换 |
| 命令模式 | RHI_CommandList | 延迟执行 GPU 命令 |
| 对象池 | 管线缓存 | 减少 PSO 创建开销 |
| 观察者模式 | 光源/相机更新 | 数据变化通知 |

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
