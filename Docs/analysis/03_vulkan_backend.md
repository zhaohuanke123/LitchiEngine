# Vulkan 后端实现分析

## 1. 模块概述

Vulkan 后端位于 `Engine/Source/Runtime/Function/Renderer/RHI/Vulkan/`，是 RHI 抽象层的 Vulkan 实现。每个 RHI 接口类都有对应的 Vulkan 实现类。

## 2. 类对应关系

| RHI 抽象类 | Vulkan 实现类 |
|-----------|---------------|
| RHI_Device | Vulkan_Device |
| RHI_CommandList | Vulkan_CommandList |
| RHI_Pipeline | Vulkan_Pipeline |
| RHI_Shader | Vulkan_Shader |
| RHI_DescriptorSetLayout | Vulkan_DescriptorSetLayout |
| RHI_Texture | Vulkan_Texture |
| RHI_Semaphore | Vulkan_Semaphore |

## 3. Vulkan_Device - 设备管理

### 3.1 初始化流程

```
CreateInstance() → PickPhysicalDevice() → CreateLogicalDevice() →
CreateCommandPools() → CreateDescriptorPool() → CreateAllocator()
```

### 3.2 核心成员

```cpp
class Vulkan_Device {
public:
    // Vulkan 核心对象
    VkInstance m_instance = nullptr;
    VkPhysicalDevice m_physical_device = nullptr;
    VkDevice m_device = nullptr;
    VkSurfaceKHR m_surface = nullptr;

    // 队列
    std::array<VkQueue, RHI_Queue_Type::Max> m_queues;
    std::array<uint32_t, RHI_Queue_Type::Max> m_queue_family_indices;

    // 命令池
    std::array<VkCommandPool, RHI_Queue_Type::Max> m_command_pools;

    // 描述符池
    VkDescriptorPool m_descriptor_pool = nullptr;

    // 内存分配器
    VmaAllocator m_allocator = nullptr;

    // 管线缓存
    std::unordered_map<uint64_t, Vulkan_Pipeline*> m_pipeline_cache;
    std::unordered_map<uint64_t, Vulkan_DescriptorSetLayout*> m_layout_cache;

    // 延迟删除队列
    struct DeletionEntry {
        RHI_Resource_Type type;
        void* resource;
        uint64_t frame;
    };
    std::vector<DeletionEntry> m_deletion_queue;
};
```

### 3.3 设备创建细节

```cpp
void Vulkan_Device::CreateLogicalDevice() {
    // 1. 获取队列族属性
    std::vector<VkQueueFamilyProperties> queue_families;

    // 2. 选择队列族
    uint32_t graphics_family = ...;  // 图形队列
    uint32_t compute_family = ...;   // 计算队列
    uint32_t copy_family = ...;      // 拷贝队列

    // 3. 创建设备队列
    std::vector<VkDeviceQueueCreateInfo> queue_infos;
    float priority = 1.0f;

    // 4. 设置设备扩展
    std::vector<const char*> device_extensions = {
        VK_KHR_SWAPCHAIN_EXTENSION_NAME,
        VK_EXT_DESCRIPTOR_INDEXING_EXTENSION_NAME,  // Bindless
        VK_KHR_BUFFER_DEVICE_ADDRESS_EXTENSION_NAME,
        // ...
    };

    // 5. 设置物理设备特性
    VkPhysicalDeviceFeatures2 features2;
    features2.features.fillModeNonSolid = VK_TRUE;  // 线框模式
    features2.features.samplerAnisotropy = VK_TRUE;  // 各向异性过滤
    features2.features.geometryShader = VK_TRUE;     // 几何着色器
    features2.features.tessellationShader = VK_TRUE; // 曲面细分

    // 6. 创建逻辑设备
    vkCreateDevice(m_physical_device, &create_info, nullptr, &m_device);
}
```

### 3.4 VMA 内存分配器

```cpp
void Vulkan_Device::CreateAllocator() {
    VmaAllocatorCreateInfo allocator_info = {};
    allocator_info.physicalDevice = m_physical_device;
    allocator_info.device = m_device;
    allocator_info.instance = m_instance;
    allocator_info.flags = VMA_ALLOCATOR_CREATE_BUFFER_DEVICE_ADDRESS_BIT;

    vmaCreateAllocator(&allocator_info, &m_allocator);
}
```

### 3.5 描述符池 (Bindless)

```cpp
void Vulkan_Device::CreateDescriptorPool() {
    std::vector<VkDescriptorPoolSize> pool_sizes = {
        { VK_DESCRIPTOR_TYPE_SAMPLER, 1000 },
        { VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 100000 },  // 大量纹理槽
        { VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE, 100000 },
        { VK_DESCRIPTOR_TYPE_STORAGE_IMAGE, 1000 },
        { VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, 10000 },
        { VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, 1000 },
    };

    VkDescriptorPoolCreateInfo pool_info = {};
    pool_info.flags = VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT;  // Bindless 支持
    pool_info.maxSets = rhi_max_descriptor_set_count;
    pool_info.poolSizeCount = pool_sizes.size();
    pool_info.pPoolSizes = pool_sizes.data();

    vkCreateDescriptorPool(m_device, &pool_info, nullptr, &m_descriptor_pool);
}
```

## 4. Vulkan_CommandList - 命令列表

### 4.1 命令缓冲区管理

```cpp
class Vulkan_CommandList : public RHI_CommandList {
public:
    // 命令缓冲区
    VkCommandBuffer m_cmd_buffer = nullptr;

    // 渲染 Pass 状态
    VkRenderPassBeginInfo m_render_pass_info = {};
    VkRenderPass m_render_pass = nullptr;
    VkFramebuffer m_framebuffer = nullptr;
    bool m_is_render_pass_active = false;

    // 时间戳查询
    VkQueryPool m_query_pool_timestamp = nullptr;
    std::vector<uint64_t> m_timestamp_results;

    // 遮挡查询
    VkQueryPool m_query_pool_occlusion = nullptr;
};
```

### 4.2 命令录制流程

```cpp
void Vulkan_CommandList::Begin() {
    // 重置命令缓冲区
    vkResetCommandBuffer(m_cmd_buffer, 0);

    VkCommandBufferBeginInfo begin_info = {};
    begin_info.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    begin_info.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
    vkBeginCommandBuffer(m_cmd_buffer, &begin_info);

    m_state = RHI_CommandListState::Recording;
}

void Vulkan_CommandList::End() {
    // 结束渲染 Pass (如果活跃)
    if (m_is_render_pass_active) {
        vkCmdEndRenderPass(m_cmd_buffer);
        m_is_render_pass_active = false;
    }

    vkEndCommandBuffer(m_cmd_buffer);
    m_state = RHI_CommandListState::Ended;
}
```

### 4.3 渲染 Pass

```cpp
void Vulkan_CommandList::BeginRenderPass(RHI_PipelineState& pso) {
    // 1. 创建附件描述
    std::vector<VkAttachmentDescription> attachments;
    std::vector<VkAttachmentReference> color_refs;
    VkAttachmentReference depth_ref;

    // 2. 创建子 Pass
    VkSubpassDescription subpass = {};
    subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount = color_refs.size();
    subpass.pColorAttachments = color_refs.data();
    subpass.pDepthStencilAttachment = &depth_ref;

    // 3. 创建子 Pass 依赖
    VkSubpassDependency dependency = {};
    dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
    dependency.dstSubpass = 0;
    dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependency.srcAccessMask = 0;
    dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

    // 4. 创建 RenderPass
    VkRenderPassCreateInfo rp_info = {};
    rp_info.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
    rp_info.attachmentCount = attachments.size();
    rp_info.pAttachments = attachments.data();
    rp_info.subpassCount = 1;
    rp_info.pSubpasses = &subpass;
    rp_info.dependencyCount = 1;
    rp_info.pDependencies = &dependency;

    vkCreateRenderPass(device, &rp_info, nullptr, &m_render_pass);

    // 5. 创建 Framebuffer
    VkFramebufferCreateInfo fb_info = {};
    fb_info.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    fb_info.renderPass = m_render_pass;
    fb_info.attachmentCount = image_views.size();
    fb_info.pAttachments = image_views.data();
    fb_info.width = pso.m_width;
    fb_info.height = pso.m_height;
    fb_info.layers = 1;

    vkCreateFramebuffer(device, &fb_info, nullptr, &m_framebuffer);

    // 6. 开始渲染 Pass
    vkCmdBeginRenderPass(m_cmd_buffer, &m_render_pass_info, VK_SUBPASS_CONTENTS_INLINE);
    m_is_render_pass_active = true;
}
```

## 5. Vulkan_Pipeline - 管线

### 5.1 管线创建

```cpp
class Vulkan_Pipeline {
public:
    VkPipeline m_pipeline = nullptr;
    VkPipelineLayout m_pipeline_layout = nullptr;
    Vulkan_DescriptorSetLayout* m_descriptor_set_layout = nullptr;

    static Vulkan_Pipeline* Create(RHI_PipelineState& pso) {
        Vulkan_Pipeline* pipeline = new Vulkan_Pipeline();

        // 1. 着色器阶段
        std::vector<VkPipelineShaderStageCreateInfo> stages;
        if (pso.shader_vertex) stages.push_back(CreateShaderStage(VK_SHADER_STAGE_VERTEX_BIT, pso.shader_vertex));
        if (pso.shader_pixel) stages.push_back(CreateShaderStage(VK_SHADER_STAGE_FRAGMENT_BIT, pso.shader_pixel));
        if (pso.shader_compute) stages.push_back(CreateShaderStage(VK_SHADER_STAGE_COMPUTE_BIT, pso.shader_compute));

        // 2. 顶点输入
        VkPipelineVertexInputStateCreateInfo vertex_input = {};
        // 从着色器反射获取顶点布局

        // 3. 输入装配
        VkPipelineInputAssemblyStateCreateInfo input_assembly = {};
        input_assembly.topology = ToVkPrimitiveTopology(pso.primitive_topology);

        // 4. 视口和裁剪
        VkPipelineViewportStateCreateInfo viewport_state = {};

        // 5. 光栅化
        VkPipelineRasterizationStateCreateInfo rasterizer = {};
        rasterizer.polygonMode = ToVkPolygonMode(pso.rasterizer_state->fill_mode);
        rasterizer.cullMode = ToVkCullMode(pso.rasterizer_state->cull_mode);
        rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;

        // 6. 多重采样
        VkPipelineMultisampleStateCreateInfo multisampling = {};

        // 7. 深度/模板
        VkPipelineDepthStencilStateCreateInfo depth_stencil = {};
        depth_stencil.depthTestEnable = pso.depth_stencil_state->depth_enable;
        depth_stencil.depthWriteEnable = pso.depth_stencil_state->depth_write;
        depth_stencil.depthCompareOp = ToVkCompareOp(pso.depth_stencil_state->depth_compare);

        // 8. 颜色混合
        VkPipelineColorBlendStateCreateInfo color_blending = {};
        // 从 blend_state 配置

        // 9. 动态状态
        std::vector<VkDynamicState> dynamic_states = {
            VK_DYNAMIC_STATE_VIEWPORT,
            VK_DYNAMIC_STATE_SCISSOR
        };

        // 10. 创建管线
        if (pso.IsCompute()) {
            VkComputePipelineCreateInfo compute_info = {};
            compute_info.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
            compute_info.stage = stages[0];
            compute_info.layout = pipeline->m_pipeline_layout;
            vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &compute_info, nullptr, &pipeline->m_pipeline);
        } else {
            VkGraphicsPipelineCreateInfo graphics_info = {};
            graphics_info.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
            graphics_info.stageCount = stages.size();
            graphics_info.pStages = stages.data();
            graphics_info.pVertexInputState = &vertex_input;
            graphics_info.pInputAssemblyState = &input_assembly;
            graphics_info.pViewportState = &viewport_state;
            graphics_info.pRasterizationState = &rasterizer;
            graphics_info.pMultisampleState = &multisampling;
            graphics_info.pDepthStencilState = &depth_stencil;
            graphics_info.pColorBlendState = &color_blending;
            graphics_info.layout = pipeline->m_pipeline_layout;
            graphics_info.renderPass = render_pass;
            vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &graphics_info, nullptr, &pipeline->m_pipeline);
        }

        return pipeline;
    }
};
```

## 6. Vulkan_Shader - 着色器

### 6.1 编译流程

```
HLSL 源码 → DXCompiler (dxc) → SPIR-V → VkShaderModule
```

### 6.2 着色器反射

```cpp
class Vulkan_Shader : public RHI_Shader {
public:
    VkShaderModule m_shader_module = nullptr;

    // 反射信息
    std::vector<RHI_Descriptor> m_descriptors;      // 描述符列表
    std::vector<RHI_PushConstantRange> m_push_constants;  // Push Constant 范围
    RHI_VertexBufferLayout m_vertex_layout;          // 顶点布局

    // 从 SPIR-V 反射
    void Reflect(const std::vector<uint32_t>& spirv);
};
```

### 6.3 SPIR-V 反射

```cpp
void Vulkan_Shader::Reflect(const std::vector<uint32_t>& spirv) {
    spv_reflect::ShaderModule reflection(spirv.data(), spirv.size());

    // 1. 提取描述符绑定
    uint32_t count = 0;
    reflection.EnumerateDescriptorBindings(&count, nullptr);
    std::vector<SpvReflectDescriptorBinding*> bindings(count);
    reflection.EnumerateDescriptorBindings(&count, bindings.data());

    for (auto* binding : bindings) {
        RHI_Descriptor descriptor;
        descriptor.slot = binding->binding;
        descriptor.stage = binding->stage;
        descriptor.type = SpvToRhiDescriptorType(binding->descriptor_type);
        // ...
        m_descriptors.push_back(descriptor);
    }

    // 2. 提取 Push Constants
    // 3. 提取顶点输入布局
}
```

## 7. Vulkan_DescriptorSetLayout - 描述符集布局

### 7.1 创建流程

```cpp
class Vulkan_DescriptorSetLayout : public RHI_DescriptorSetLayout {
public:
    VkDescriptorSetLayout m_layout = nullptr;
    VkDescriptorSet m_descriptor_set = nullptr;

    void Create() {
        // 1. 从描述符列表创建绑定
        std::vector<VkDescriptorSetLayoutBinding> bindings;
        for (auto& desc : m_descriptors) {
            VkDescriptorSetLayoutBinding binding = {};
            binding.binding = desc.slot;
            binding.descriptorType = ToVkDescriptorType(desc.type);
            binding.descriptorCount = desc.array_length;
            binding.stageFlags = ToVkShaderStage(desc.stage);
            bindings.push_back(binding);
        }

        // 2. 创建布局
        VkDescriptorSetLayoutCreateInfo layout_info = {};
        layout_info.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
        layout_info.flags = VK_DESCRIPTOR_SET_LAYOUT_CREATE_UPDATE_AFTER_BIND_POOL_BIT;  // Bindless
        layout_info.bindingCount = bindings.size();
        layout_info.pBindings = bindings.data();

        vkCreateDescriptorSetLayout(device, &layout_info, nullptr, &m_layout);

        // 3. 分配描述符集
        VkDescriptorSetAllocateInfo alloc_info = {};
        alloc_info.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
        alloc_info.descriptorPool = descriptor_pool;
        alloc_info.descriptorSetCount = 1;
        alloc_info.pSetLayouts = &m_layout;

        vkAllocateDescriptorSets(device, &alloc_info, &m_descriptor_set);
    }
};
```

### 7.2 资源更新

```cpp
void Vulkan_DescriptorSetLayout::UpdateDescriptorSet() {
    std::vector<VkWriteDescriptorSet> writes;

    for (auto& desc : m_descriptors) {
        VkWriteDescriptorSet write = {};
        write.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
        write.dstSet = m_descriptor_set;
        write.dstBinding = desc.slot;
        write.descriptorCount = desc.array_length;

        switch (desc.type) {
        case RHI_Descriptor_Type::ConstantBuffer:
            write.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
            write.pBufferInfo = &buffer_info;
            break;
        case RHI_Descriptor_Type::Texture:
            write.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
            write.pImageInfo = &image_info;
            break;
        // ...
        }

        writes.push_back(write);
    }

    vkUpdateDescriptorSets(device, writes.size(), writes.data(), 0, nullptr);
}
```

## 8. Vulkan_Texture - 纹理

### 8.1 创建流程

```cpp
class Vulkan_Texture : public RHI_Texture {
public:
    VkImage m_image = nullptr;
    VmaAllocation m_allocation = nullptr;
    std::vector<VkImageView> m_image_views;

    void Create() {
        // 1. 创建 Image
        VkImageCreateInfo image_info = {};
        image_info.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
        image_info.imageType = VK_IMAGE_TYPE_2D;
        image_info.format = ToVkFormat(m_format);
        image_info.extent = { m_width, m_height, 1 };
        image_info.mipLevels = m_mip_count;
        image_info.arrayLayers = m_array_length;
        image_info.samples = VK_SAMPLE_COUNT_1_BIT;
        image_info.tiling = VK_IMAGE_TILING_OPTIMAL;
        image_info.usage = ToVkImageUsageFlags(m_flags);
        image_info.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
        image_info.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;

        // 2. 分配内存 (VMA)
        VmaAllocationCreateInfo alloc_info = {};
        alloc_info.usage = VMA_MEMORY_USAGE_GPU_ONLY;

        vmaCreateImage(m_allocator, &image_info, &alloc_info,
                       &m_image, &m_allocation, nullptr);

        // 3. 创建 ImageView
        VkImageViewCreateInfo view_info = {};
        view_info.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
        view_info.image = m_image;
        view_info.viewType = VK_IMAGE_VIEW_TYPE_2D;
        view_info.format = ToVkFormat(m_format);
        view_info.subresourceRange = { aspect_mask, 0, m_mip_count, 0, m_array_length };

        vkCreateImageView(m_device, &view_info, nullptr, &m_image_views[0]);
    }
};
```

## 9. 同步机制

### 9.1 Vulkan_Semaphore

```cpp
class Vulkan_Semaphore : public RHI_Semaphore {
public:
    VkSemaphore m_semaphore = nullptr;
    VkFence m_fence = nullptr;

    void Create() {
        VkSemaphoreCreateInfo sem_info = {};
        sem_info.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;
        vkCreateSemaphore(device, &sem_info, nullptr, &m_semaphore);

        VkFenceCreateInfo fence_info = {};
        fence_info.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
        vkCreateFence(device, &fence_info, nullptr, &m_fence);
    }

    void Wait() { vkWaitForFences(device, 1, &m_fence, VK_TRUE, UINT64_MAX); }
    void Reset() { vkResetFences(device, 1, &m_fence); }
};
```

### 9.2 帧同步

```
Frame N:
  1. 等待上一帧完成 (Fence)
  2. 录制命令
  3. 提交命令 (Semaphore 信号)
  4. 呈现 (等待 Semaphore)
  5. 延迟删除队列清理
```

## 10. 关键优化策略

### 10.1 Bindless 资源

- 使用 `VK_EXT_descriptor_indexing` 扩展
- 描述符池设置 `UPDATE_AFTER_BIND` 标志
- 纹理通过索引访问，无需频繁更新描述符集

### 10.2 管线缓存

- 基于 PSO hash 缓存已创建的管线
- 避免每帧重复创建管线对象

### 10.3 延迟删除

```cpp
void Vulkan_Device::DeletionQueueParse() {
    // 只删除帧号 <= 当前帧 - 帧数 的资源
    // 确保GPU不再使用该资源
    auto it = std::remove_if(m_deletion_queue.begin(), m_deletion_queue.end(),
        [this](const DeletionEntry& entry) {
            if (entry.frame <= m_frame_count - frames_in_flight) {
                // 安全删除
                switch (entry.type) {
                case RHI_Resource_Type::Buffer:   vmaDestroyBuffer(...); break;
                case RHI_Resource_Type::Image:    vmaDestroyImage(...); break;
                case RHI_Resource_Type::Pipeline: vkDestroyPipeline(...); break;
                // ...
                }
                return true;
            }
            return false;
        });
    m_deletion_queue.erase(it, m_deletion_queue.end());
}
```

### 10.4 VMA 内存管理

- 使用 VMA 自动管理 GPU 内存分配
- 支持 `GPU_ONLY`、`CPU_TO_GPU`、`GPU_TO_CPU` 等内存类型
- 支持缓冲区设备地址 (Buffer Device Address)

## 11. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 桥接模式 | RHI ↔ Vulkan 实现 | 分离抽象和实现 |
| 对象池 | 管线缓存、描述符集缓存 | 减少创建开销 |
| 命令模式 | CommandList | 延迟执行 GPU 命令 |
| 状态机 | CommandList 状态 | 管理命令录制生命周期 |
| 延迟删除 | DeletionQueue | 安全释放 GPU 资源 |

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
