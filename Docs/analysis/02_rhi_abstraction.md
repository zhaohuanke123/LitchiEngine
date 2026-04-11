# RHI 抽象层设计分析

## 1. 模块概述

RHI (Render Hardware Interface) 是渲染硬件抽象层，位于 `Engine/Source/Runtime/Function/Renderer/RHI/`，设计目标是屏蔽不同图形 API (Vulkan, D3D12) 的差异，提供统一的渲染接口。

## 2. 核心枚举定义

### 2.1 API 和设备类型

```cpp
enum class RHI_Api_Type {
    D3d12,
    Vulkan,
    Undefined
};

enum class RHI_PhysicalDevice_Type {
    Integrated,  // 集成显卡
    Discrete,    // 独立显卡
    Virtual,     // 虚拟显卡
    Cpu,
    Undefined
};

enum class RHI_Queue_Type {
    Graphics,  // 图形队列
    Compute,   // 计算队列
    Copy,      // 拷贝队列
    Max
};
```

### 2.2 格式定义

```cpp
enum class RHI_Format : uint32_t {
    // R
    R8_Unorm, R8_Uint, R16_Unorm, R16_Uint, R16_Float, R32_Uint, R32_Float,
    // Rg
    R8G8_Unorm, R16G16_Float, R32G32_Float,
    // Rgb
    R11G11B10_Float, R32G32B32_Float,
    // Rgba
    R8G8B8A8_Unorm, R10G10B10A2_Unorm, R16G16B16A16_Unorm,
    R16G16B16A16_Snorm, R16G16B16A16_Float, R32G32B32A32_Float,
    // Depth
    D16_Unorm, D32_Float, D32_Float_S8X24_Uint,
    // Compressed
    BC7, ASTC,
    // Surface
    B8R8G8A8_Unorm,
    Max
};
```

### 2.3 描述符类型

```cpp
enum class RHI_Descriptor_Type {
    Sampler,            // 采样器
    Texture,            // 纹理 (只读)
    TextureStorage,     // 存储纹理 (读写)
    PushConstantBuffer, // Push Constant
    ConstantBuffer,     // 常量缓冲
    StructuredBuffer,   // 结构化缓冲
    Max
};
```

### 2.4 图像布局

```cpp
enum class RHI_Image_Layout {
    General,               // 通用布局
    Preinitialized,        // 预初始化
    Attachment,            // 附件 (渲染目标)
    Shading_Rate_Attachment, // VRS 附件
    Shader_Read,           // 着色器读取
    Transfer_Source,       // 传输源
    Transfer_Destination,  // 传输目标
    Present_Source,        // 呈现源
    Max
};
```

## 3. RHI_Device - 设备管理

### 3.1 核心接口

```cpp
class RHI_Device {
public:
    // 生命周期
    static void Initialize();
    static void Tick(const uint64_t frame_count);
    static void Destroy();

    // 队列操作
    static void QueueSubmit(RHI_Queue_Type type, ...);
    static void QueuePresent(void* swapchain, ...);

    // 描述符管理
    static void CreateDescriptorPool();
    static void AllocateDescriptorSet(void*& resource, ...);

    // 管线管理 (带缓存)
    static void GetOrCreatePipeline(
        RHI_PipelineState& pso,
        RHI_Pipeline*& pipeline,
        RHI_DescriptorSetLayout*& layout
    );

    // 内存管理
    static void MemoryBufferCreate(void*& resource, uint64_t size, ...);
    static void MemoryTextureCreate(RHI_Texture* texture);

    // 立即执行
    static RHI_CommandList* CmdImmediateBegin(RHI_Queue_Type queue_type);
    static void CmdImmediateSubmit(RHI_CommandList* cmd_list);

    // 延迟删除队列
    static void DeletionQueuePush(RHI_Resource_Type type, void* resource, uint64_t frame);
    static void DeletionQueueParse();
};
```

### 3.2 关键特性

- **管线缓存**: 基于 PSO hash 的管线缓存机制
- **描述符池**: 支持 bindless 资源绑定
- **延迟删除**: 通过帧同步安全释放 GPU 资源
- **内存管理**: 使用 VMA (Vulkan Memory Allocator)

## 4. RHI_CommandList - 命令列表

### 4.1 状态机设计

```
Idle → Recording → Ended → Submitted → (回到 Idle)
```

```cpp
enum class RHI_CommandListState : uint8_t {
    Idle,
    Recording,
    Ended,
    Submitted
};
```

### 4.2 核心方法

```cpp
class RHI_CommandList : public Object {
public:
    // 生命周期
    void Begin();
    void End();
    void Submit();
    void WaitForExecution();

    // 渲染 Pass
    void SetPipelineState(RHI_PipelineState& pso);
    void SetPipelineState(RHI_PipelineState& pso, bool needBeginRenderPass);

    // 绘制
    void Draw(uint32_t vertex_count, uint32_t vertex_start_index = 0);
    void DrawIndexed(uint32_t index_count, uint32_t index_offset = 0,
                     uint32_t vertex_offset = 0, uint32_t instance_start_index = 0,
                     uint32_t instance_count = 1);

    // 计算
    void Dispatch(uint32_t x, uint32_t y, uint32_t z = 1, bool async = false);
    void Dispatch(RHI_Texture* texture);

    // 资源绑定
    void SetBufferVertex(const RHI_VertexBuffer* buffer, uint32_t binding = 0);
    void SetBufferIndex(const RHI_IndexBuffer* buffer);
    void SetConstantBuffer(uint32_t slot, RHI_ConstantBuffer* buffer);
    void SetTexture(uint32_t slot, RHI_Texture* texture, uint32_t mip_index, ...);
    void SetStructuredBuffer(uint32_t slot, RHI_StructuredBuffer* buffer);
    void SetSampler(uint32_t slot, RHI_Sampler* sampler);

    // Push Constants
    void PushConstants(uint32_t offset, uint32_t size, const void* data);
    template<typename T>
    void PushConstants(const T& data) { PushConstants(0, sizeof(T), &data); }

    // 内存屏障
    void InsertBarrierTexture(RHI_Texture* texture, uint32_t mip_start,
                              uint32_t mip_range, uint32_t array_length,
                              RHI_Image_Layout layout_old, RHI_Image_Layout layout_new);

    // 调试标记
    void BeginMarker(const char* name);
    void EndMarker();

    // 时间戳查询
    uint32_t BeginTimestamp();
    void EndTimestamp();
    float GetTimestampResult(uint32_t index);

    // 遮挡查询
    void BeginOcclusionQuery(uint64_t entity_id);
    void EndOcclusionQuery();
    bool GetOcclusionQueryResult(uint64_t entity_id);
};
```

## 5. RHI_PipelineState - 管线状态

### 5.1 结构定义

```cpp
class RHI_PipelineState {
public:
    //= 静态状态 - 影响 PSO 生成
    MaterialShader* material_shader = nullptr;
    RHI_Shader* shader_vertex = nullptr;
    RHI_Shader* shader_hull = nullptr;
    RHI_Shader* shader_domain = nullptr;
    RHI_Shader* shader_pixel = nullptr;
    RHI_Shader* shader_compute = nullptr;
    RHI_RasterizerState* rasterizer_state = nullptr;
    RHI_BlendState* blend_state = nullptr;
    RHI_DepthStencilState* depth_stencil_state = nullptr;
    RHI_PrimitiveTopology primitive_topology = RHI_PrimitiveTopology::TriangleList;
    bool instancing = false;

    //= 渲染目标
    std::array<RHI_Texture*, rhi_max_render_target_count> render_target_color_textures;
    RHI_Texture* render_target_depth_texture = nullptr;
    RHI_Texture* vrs_input_texture = nullptr;

    //= 清除参数
    float clear_depth = rhi_depth_load;
    uint32_t clear_stencil = rhi_stencil_load;
    std::array<Color, rhi_max_render_target_count> clear_color;

    //= 方法
    void Prepare();
    bool HasClearValues() const;
    uint64_t GetHash() const { return m_hash; }
    bool IsGraphics() const;
    bool IsCompute() const;
    bool HasTessellation() const;

private:
    uint32_t m_width = 0;
    uint32_t m_height = 0;
    uint64_t m_hash = 0;
    uint64_t m_hash_dynamic = 0;
};
```

### 5.2 Hash 机制

PSO 使用 hash 进行缓存查找，避免重复创建管线：

```cpp
uint64_t GetHash() const { return m_hash; }
```

## 6. RHI_Descriptor - 描述符

### 6.1 结构定义

```cpp
class RHI_Descriptor {
public:
    RHI_Descriptor(
        const std::string& name,
        const RHI_Descriptor_Type type,
        const RHI_Image_Layout layout,
        const uint32_t slot,
        const uint32_t array_length,
        const uint32_t stage,
        const uint32_t struct_size,
        const bool as_array,
        const uint32_t isMaterial,
        std::vector<ShaderUniform>* uniformList
    );

    // 影响描述符 hash (静态 - 反射)
    uint32_t slot = 0;
    uint32_t stage = 0;

    // 影响描述符集 hash (动态 - 渲染器)
    uint32_t mip = 0;
    uint32_t mip_range = 0;
    void* data = nullptr;

    // 不影响 hash
    RHI_Descriptor_Type type = RHI_Descriptor_Type::Max;
    RHI_Image_Layout layout = RHI_Image_Layout::Max;
    uint64_t range = 0;
    uint32_t array_length = 0;
    uint32_t dynamic_offset = 0;
    uint32_t struct_size = 0;
    bool as_array = false;

    // 材质相关
    bool isMaterial;
    std::vector<ShaderUniform>* uniformList = nullptr;

    uint64_t ComputeHash();
    bool IsStorage() const { return type == RHI_Descriptor_Type::TextureStorage; }
};
```

## 7. RHI_DescriptorSetLayout - 描述符集布局

### 7.1 设计说明

描述符集布局从着色器反射信息创建，用于管线创建和资源绑定。

```cpp
class RHI_DescriptorSetLayout : public Object {
public:
    RHI_DescriptorSetLayout(const std::vector<RHI_Descriptor>& descriptors, const std::string& name);

    // 资源设置
    void SetConstantBuffer(uint32_t slot, RHI_ConstantBuffer* buffer);
    void SetStructuredBuffer(uint32_t slot, RHI_StructuredBuffer* buffer);
    void SetSampler(uint32_t slot, RHI_Sampler* sampler);
    void SetTexture(uint32_t slot, RHI_Texture* texture, uint32_t mip_index, uint32_t mip_range);
    void SetMaterialGlobalBuffer(RHI_ConstantBuffer* buffer);

    // 动态偏移
    void GetDynamicOffsets(std::array<uint32_t, 10>* offsets, uint32_t* count);

    // 获取描述符集
    RHI_DescriptorSet* GetDescriptorSet();
    const std::vector<RHI_Descriptor>& GetDescriptors() const { return m_descriptors; }
    uint64_t GetHash() const { return m_hash; }

private:
    void* m_rhi_resource = nullptr;
    uint64_t m_hash = 0;
    std::vector<RHI_Descriptor> m_descriptors;
};
```

## 8. RHI_Texture - 纹理资源

### 8.1 纹理标志

```cpp
enum RHI_Texture_Flags : uint32_t {
    RHI_Texture_Srv = 1U << 0,           // 着色器资源视图
    RHI_Texture_Uav = 1U << 1,           // 无序访问视图
    RHI_Texture_Rtv = 1U << 2,           // 渲染目标视图
    RHI_Texture_Vrs = 1U << 3,           // 可变着色率
    RHI_Texture_ClearBlit = 1U << 4,     // 支持清除/Blit
    RHI_Texture_PerMipViews = 1U << 5,   // 每 Mip 视图
    RHI_Texture_Greyscale = 1U << 6,     // 灰度
    RHI_Texture_Transparent = 1U << 7,   // 透明
    RHI_Texture_Srgb = 1U << 8,          // sRGB
    RHI_Texture_Mips = 1U << 9,          // 有 Mip
    RHI_Texture_Compressed = 1U << 10,   // 压缩格式
    RHI_Texture_Mappable = 1U << 11      // 可映射
};
```

### 8.2 核心接口

```cpp
class RHI_Texture : public IResource, public std::enable_shared_from_this<RHI_Texture> {
public:
    // 属性
    uint32_t GetWidth() const;
    uint32_t GetHeight() const;
    RHI_Format GetFormat() const;
    uint32_t GetMipCount() const;
    uint32_t GetArrayLength() const;

    // 标志查询
    bool IsSrv() const;
    bool IsUav() const;
    bool IsRtv() const;
    bool IsDsv() const;
    bool HasMips() const;

    // 布局管理
    void SetLayout(RHI_Image_Layout layout, RHI_CommandList* cmd_list, ...);
    RHI_Image_Layout GetLayout(uint32_t mip) const;

    // GPU 资源
    void* GetRhiSrv() const;
    void* GetRhiUav() const;
    void* GetRhiRtv(uint32_t i = 0) const;
    void* GetRhiDsv(uint32_t i = 0) const;

protected:
    uint32_t m_width = 0;
    uint32_t m_height = 0;
    uint32_t m_mip_count = 1;
    RHI_Format m_format = RHI_Format::Max;
    std::array<RHI_Image_Layout, rhi_max_mip_count> m_layout;

    // API 资源
    void* m_rhi_resource = nullptr;
    void* m_rhi_srv = nullptr;
    void* m_rhi_uav = nullptr;
    std::array<void*, rhi_max_mip_count> m_rhi_srv_mips;
    std::array<void*, rhi_max_mip_count> m_rhi_uav_mips;
};
```

## 9. 常量和配置

```cpp
// 着色器寄存器槽位偏移 (HLSL 到 SPIR-V 转换需要)
const uint32_t rhi_shader_shift_register_u = 100;  // UAV
const uint32_t rhi_shader_shift_register_b = 200;  // ConstantBuffer
const uint32_t rhi_shader_shift_register_s = 300;  // Sampler
const uint32_t rhi_shader_shift_register_t = 400;  // Texture

// 限制
const uint8_t rhi_max_render_target_count = 8;
const uint8_t rhi_max_constant_buffer_count = 8;
const uint32_t rhi_max_descriptor_set_count = 512;
const uint8_t rhi_max_mip_count = 13;
const uint32_t rhi_max_queries_occlusion = 4096;
const uint32_t rhi_max_queries_timestamps = 512;

// 特殊值
const uint32_t rhi_all_mips = std::numeric_limits<uint32_t>::max();
const Color rhi_color_load = Color(std::numeric_limits<float>::infinity(), ...);
const float rhi_depth_load = std::numeric_limits<float>::infinity();
```

## 10. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 抽象工厂 | RHI_Device | 创建不同 API 的资源 |
| 状态模式 | RHI_CommandList | 命令列表状态机 |
| 缓存模式 | Pipeline 缓存 | 减少 PSO 创建开销 |
| 代理模式 | RHI_Texture | 封装底层 GPU 资源 |
| 值对象 | RHI_Descriptor | 描述符数据结构 |

## 11. 抽象原则

### 11.1 统一格式

- 使用 `RHI_Format` 枚举统一不同 API 的格式
- 提供 `rhi_format_to_bits_per_channel` 等辅助函数

### 11.2 资源生命周期

1. 创建: `RHI_Device::MemoryXxxCreate()`
2. 使用: 通过 `RHI_CommandList` 绑定和操作
3. 销毁: 通过延迟删除队列安全释放

### 11.3 状态追踪

- `RHI_CommandListState`: 命令列表状态
- `RHI_Image_Layout`: 纹理布局状态
- `RHI_Sync_State`: 同步状态

## 12. 与 Vulkan 的对应关系

| RHI 概念 | Vulkan 概念 |
|----------|-------------|
| RHI_Device | VkDevice + VkPhysicalDevice |
| RHI_CommandList | VkCommandBuffer |
| RHI_PipelineState | VkGraphicsPipelineCreateInfo |
| RHI_Pipeline | VkPipeline |
| RHI_DescriptorSet | VkDescriptorSet |
| RHI_DescriptorSetLayout | VkDescriptorSetLayout |
| RHI_Texture | VkImage + VkImageView |
| RHI_ConstantBuffer | VkBuffer |
| RHI_Semaphore | VkSemaphore |
| RHI_Fence | VkFence |

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
