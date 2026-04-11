# Catlike Coding 教学风格调研报告

## 概述

Catlike Coding (https://catlikecoding.com/) 是由 Jasper Flick 创建的编程教程网站，专注于 Unity 和 Godot 引擎的高质量技术教程。该网站以其独特的教学风格著称，在游戏开发社区享有盛誉。

### 教程系列概览

网站目前提供以下 Unity 教程系列（共 16 个）：

| 系列名称 | 描述 | 章节数 |
|---------|------|--------|
| **Basics** | Unity 基础入门 | 7 章 |
| **Pseudorandom Noise** | 伪随机噪声生成 | 7 章 |
| **Procedural Meshes** | 程序化网格生成 | - |
| **Pseudorandom Surfaces** | 伪随机表面生成 | - |
| **Prototypes** | 游戏原型开发 | - |
| **Movement** | 角色移动控制 | - |
| **Object Management** | 对象管理系统 | - |
| **Tower Defense** | 塔防游戏开发 | - |
| **Flow** | 流动效果（水面等） | 4 章 |
| **Custom SRP** | 自定义渲染管线 | 17 章 |
| **Rendering** | 渲染管线原理 | 20 章 |
| **Advanced Rendering** | 高级渲染技术 | 7 章 |
| **Hex Map** | 六边形地图系统 | - |
| **Marching Squares** | Marching Squares 算法 | - |
| **Mesh Basics** | 网格基础（已过时） | 4 章 |
| **Old Tutorials** | 旧版教程 | - |

## 教程结构分析

### 章节组织

Catlike Coding 的教程采用**严格的层级结构**：

```
系列 (Series)
  └── 教程 (Tutorial)
        ├── 主要章节 (Section)
        │     ├── 子章节 (Subsection)
        │     │     ├── 概念说明
        │     │     ├── 代码实现
        │     │     └── 验证结果
        │     └── Q&A 侧边栏
        └── 资源链接 (unitypackage/PDF)
```

**示例：Custom Render Pipeline 教程结构**

```
A New Render Pipeline
├── Project Setup
├── Pipeline Asset
└── Render Pipeline Instance

Rendering
├── Camera Renderer
├── Drawing the Skybox
├── Command Buffers
├── Clearing the Render Target
├── Culling
├── Drawing Geometry
└── Drawing Opaque and Transparent Geometry Separately

Editor Rendering
├── Drawing Legacy Shaders
├── Error Material
├── Partial Class
├── Drawing Gizmos
└── Drawing Unity UI

Multiple Cameras
├── Two Cameras
├── Dealing with Changing Buffer Names
├── Layers
└── Clear Flags
```

### 知识点划分

知识点遵循**原子化原则**，每个子章节专注于单一概念：

1. **单一职责**：每个子章节解决一个具体问题
2. **最小依赖**：新知识点只依赖已讲解内容
3. **即时验证**：每个改动后立即展示结果

**示例：Pseudorandom Noise - Hashing 教程的知识点分解**

| 子章节 | 知识点 | 前置依赖 |
|--------|--------|----------|
| Grid Visualization | 基础网格显示 | 无 |
| Job System | 并行计算基础 | Grid Visualization |
| Shader Basics | 着色器入门 | Job System |
| Small xxHash | 哈希函数实现 | 以上全部 |
| Bit Manipulation | 位运算技巧 | Small xxHash |
| RGB Visualization | 多通道可视化 | Bit Manipulation |

### 渐进式设计

渐进式设计体现在三个层面：

#### 1. 系列级别

教程系列之间存在明确的**前置关系**：

```
Basics → Rendering → Advanced Rendering
        ↓
      Custom SRP → (后续高级主题)
```

每个系列开篇明确标注前置要求：
- "These tutorials build on the work done in the Rendering series."
- "Upgraded to Unity 2020 on 18 May 2021."

#### 2. 教程级别

单个教程采用**功能累积模式**：

**Custom Render Pipeline 教程的渐进过程**：

| 阶段 | 功能 | 复杂度 |
|------|------|--------|
| 1 | 创建空的 Pipeline Asset | 最低 |
| 2 | 渲染天空盒 | 低 |
| 3 | 相机属性设置 | 低 |
| 4 | 不透明几何体渲染 | 中 |
| 5 | 透明几何体分离渲染 | 中 |
| 6 | 编辑器支持（Gizmos/UI） | 高 |
| 7 | 多相机支持 | 高 |

#### 3. 代码级别

代码以**增量方式**演进，使用注释标记变更：

```hlsl
// 注释掉的旧代码
//CBUFFER_START(UnityPerMaterial)
//    float4 _BaseColor;
//CBUFFER_END

// 新代码
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
```

## 代码讲解方式

### 代码展示策略

#### 1. 增量片段展示

**不展示完整文件**，而是展示变化的关键部分：

- 新增代码：完整展示新函数或代码块
- 修改代码：使用注释标记旧代码，新代码紧随其后
- 删除代码：用注释保留，便于理解变化

**优点**：
- 减少阅读负担
- 变更一目了然
- 保留演进历史

#### 2. 上下文锚定

代码片段总是包含足够的上下文：

```csharp
public class Graph : MonoBehaviour {

    [SerializeField]
    Transform pointPrefab = default;

    // 新增代码在清晰的上下文中
    [SerializeField, Range(10, 100)]
    int resolution = 10;
}
```

#### 3. 分层展示

复杂功能分层展示：

1. **接口/结构定义**：先展示数据结构
2. **核心实现**：展示主要逻辑
3. **细节补充**：展示辅助函数

### 注释风格

#### 1. 教学性注释

代码注释用于**解释意图**，而非重复代码：

```hlsl
// We subtract because that makes the flow go in the direction of the vector
float2 flowVector = tex2D(_FlowMap, i.uv).rg * 2 - 1;
```

#### 2. 变更标记注释

使用 `//` 标记被替换的代码：

```csharp
// transform.localPosition = Vector3.right * i;
transform.localPosition = Vector3.right * (i + 0.5f);
```

#### 3. Q&A 侧边栏

复杂概念通过**专门的 Q&A 块**解释：

```
┌─────────────────────────────────────┐
│ Why not use int for the hash type?  │
│                                     │
│ Using uint avoids potential issues  │
│ with negative numbers and makes     │
│ bit manipulation more intuitive.    │
└─────────────────────────────────────┘
```

常见 Q&A 主题：
- 数据类型选择（float vs half, int vs uint）
- API 设计决策
- 性能考量
- 数学原理

### 解释深度

#### 三层解释模型

| 层次 | 内容 | 示例 |
|------|------|------|
| **What** | 这段代码做什么 | "This creates a new pipeline asset" |
| **Why** | 为什么要这样做 | "Without this, nothing would render" |
| **How** | 内部如何工作 | "The SRP Batcher reduces CPU overhead by..." |

#### 深度控制原则

- **基础概念**：详细解释原理和背景
- **中间层**：解释设计决策和权衡
- **高级主题**：假设读者有基础，直接讲解实现

**示例：Shader Fundamentals 教程中的深度控制**

```markdown
### What is a Shader?

[A detailed explanation of shader pipeline, GPU architecture, etc.]

### Creating Our First Shader

[Minimal explanation - assumes understanding from previous section]

### Space Transformation

[Mathematical foundation + implementation details]
```

## 图文配合分析

### 示意图的使用

#### 1. 概念示意图

用于解释抽象概念：

- **数学函数图**：Desmos 生成的函数曲线
- **流程图**：渲染管线流程
- **结构图**：类继承关系、模块依赖

**示例：Perlin Noise 教程**

使用数学图形展示：
- 梯度向量构造过程（线 → 正方形 → 八面体）
- 插值函数的连续性分析
- 噪声函数的导数曲线

#### 2. 对比示意图

展示不同参数的效果对比：

- 左右并列展示不同配置
- 参数变化对结果的影响
- Before/After 效果对比

### 效果图的使用

#### 1. 渲染结果截图

每个主要步骤后展示渲染结果：

- **场景视图**：整体效果
- **材质预览**：着色器效果
- **帧调试器**：验证渲染流程

**示例：Custom Render Pipeline 教程**

```
[Scene View: Empty] → [Scene View: Skybox Only] → [Scene View: Full Scene]
[Frame Debugger: No Draw Calls] → [Frame Debugger: Skybox Draw] → [Frame Debugger: All Geometry]
```

#### 2. 性能指标截图

展示优化效果：

- Stats 面板：顶点数、三角形数、Draw Calls
- Profiler 截图：CPU/GPU 时间
- Memory Profiler：内存分配

### 动画/GIF 的使用

#### 使用原则

- **极少使用**：只在最终效果展示时使用
- **小尺寸**：不喧宾夺主
- **自动播放**：无需用户交互

**示例：Game Objects and Scripts 教程**

末尾展示一个小的时钟动画 GIF，验证教程最终成果。

#### 动画替代方案

静态图像 + 文字说明通常足以传达信息，动画仅用于：
- 时间相关效果（水面流动、粒子系统）
- 交互效果预览
- 最终成果展示

### 图像组织方式

#### 1. 并列展示

相关图像左右并列，便于对比：

```
[Inspector: Material Settings] [Scene: Render Result]
```

#### 2. 流程展示

按执行顺序排列图像：

```
[1. Initial State] → [2. Intermediate] → [3. Final Result]
```

#### 3. 分层展示

同一场景的不同视角/模式：

```
[Scene View] [Game View] [Frame Debugger]
```

## 难度递进设计

### 从简单到复杂的过渡方式

#### 1. 空白起步

大多数教程从**最小可行实现**开始：

**Draw Calls 教程的起步**：
```hlsl
Shader "Custom/Unlit" {
    SubShader {
        Pass {}
    }
}
```

然后逐步添加功能：
1. 添加 HLSL 代码块
2. 实现顶点着色器
3. 实现片段着色器
4. 添加属性
5. 支持批处理

#### 2. 问题驱动

每个复杂功能都由**实际问题**引入：

```
问题：所有物体都是同一颜色
原因：材质属性没有传递到着色器
解决：添加 Per-Material 属性
验证：[截图展示不同颜色的物体]
```

#### 3. 迭代改进

功能不是一次性实现完整，而是**逐步完善**：

**Perlin Noise 的迭代过程**：

| 迭代 | 功能 | 复杂度 |
|------|------|--------|
| 1 | 重构 Value Noise 为接口 | 低 |
| 2 | 实现泛型格子系统 | 低 |
| 3 | 添加 1D 梯度 | 中 |
| 4 | 添加 2D 梯度 | 中 |
| 5 | 添加归一化 | 高 |
| 6 | 添加 3D 梯度 | 高 |

### 每个教程的起点和终点

#### 起点：明确前置要求

每个教程开篇列出：

1. **前置教程**："This tutorial follows Building a Graph."
2. **Unity 版本**："Made with Unity 2020.3."
3. **知识假设**："Assumes familiarity with C# and Unity basics."

#### 终点：明确成果

教程结束时读者将拥有：

1. **可工作的代码**：完整实现某功能
2. **可下载的项目**：unitypackage 文件
3. **下一步指引**：链接到下一个教程

### 系列教程之间的衔接

#### 1. 知识传递

前一系列的成果作为后一系列的基础：

```
Rendering 系列：
  └── Matrices → Shader Fundamentals → ... → Parallax

Advanced Rendering 系列（依赖 Rendering）：
  └── Flat and Wireframe Shading → Tessellation → ...

Custom SRP 系列（独立的现代实现）：
  └── Custom Render Pipeline → Draw Calls → Directional Lights → ...
```

#### 2. 版本演进

某些系列有更新版本：

```
Mesh Basics (旧版, Unity 4/5)
      ↓
Procedural Meshes (新版, 现代实现)
```

#### 3. 概念复用

相同概念在不同系列中重复出现，但应用场景不同：

- **噪声函数**：Pseudorandom Noise（基础）→ Pseudorandom Surfaces（应用）
- **着色器基础**：Rendering（理解原理）→ Custom SRP（实现管线）

## 实践设计

### 练习题设计

**核心发现：Catlike Coding 教程不包含显式练习题。**

#### 替代策略：跟随实践

教程本身就是一次完整的实践过程：

1. **逐步跟随**：读者同步编写代码
2. **即时验证**：每个步骤后立即验证结果
3. **资源支持**：提供素材（纹理、模型）

#### 下载包设计

每个教程/章节提供 **unitypackage** 下载：

- 包含完整的阶段性成果
- 读者可以检查自己的实现
- 可以跳过某些步骤直接从中间开始

### 挑战任务设计

#### 内嵌挑战

虽然无显式练习，但教程中包含**隐式挑战**：

**Draw Calls 教程中的挑战**：

> "Create a MeshBall component to draw 1023 instances"
> "Implement PerObjectMaterialProperties for per-object customization"

这些挑战是教程的一部分，读者必须完成才能继续。

#### 可选扩展

教程末尾有时建议扩展方向：

> "Now our triplanar shader is fully functional. You can use it as a basis for your own work, extending, tweaking, and tuning it as desired."

### 验证方法

#### 1. 视觉验证

每步代码后展示预期结果：

```
[代码块]
↓
[结果截图]
↓
[说明：If you see X, everything is working correctly]
```

#### 2. 数据验证

使用 Unity 内置工具：

- **Frame Debugger**：验证渲染流程
- **Profiler**：验证性能指标
- **Stats Panel**：验证 Draw Calls

#### 3. 问题诊断

提供常见问题和解决方案：

```
问题：物体显示为粉色
原因：着色器编译错误
解决：检查 Console 面板的错误信息
```

## 核心特征总结

### 教学设计原则

通过深入分析，提炼出以下核心原则：

#### 1. 渐进式构建原则 (Progressive Construction)

- 从最小可行实现开始
- 每步只添加一个功能
- 保持代码始终可运行

#### 2. 问题驱动原则 (Problem-Driven)

- 先提出问题/需求
- 再解释解决方案
- 最后验证结果

#### 3. 增量展示原则 (Incremental Disclosure)

- 不展示完整文件
- 只展示变化部分
- 使用注释保留历史

#### 4. 即时验证原则 (Immediate Verification)

- 每个改动后立即展示结果
- 使用截图/数据验证
- 提供预期行为描述

#### 5. 概念分层原则 (Conceptual Layering)

- What/Why/How 三层解释
- Q&A 侧边栏处理细节问题
- 深度根据概念重要性调整

#### 6. 系列连贯原则 (Series Continuity)

- 明确前置要求
- 知识点跨系列复用
- 版本演进有清晰路径

### 内容组织模式

```
标题（明确目标）
    ↓
前置要求（读者自检）
    ↓
章节（主内容）
    ├── 概念引入
    ├── 代码实现
    ├── 结果验证
    └── Q&A 补充
    ↓
资源下载（实践支持）
    ↓
下一步（系列衔接）
```

### 代码展示模式

```
[概念说明]
    ↓
[最小代码片段]
    ↓
[结果截图]
    ↓
[问题/局限]
    ↓
[改进代码]
    ↓
[新结果截图]
    ↓
[重复...]
```

## 对 LitchiEngine 的建议

基于 Catlike Coding 的教学风格分析，针对 LitchiEngine 的教程设计提出以下建议：

### 1. 教程结构建议

#### 系列规划

建议按模块划分教程系列：

| 系列 | 内容 | 前置 |
|------|------|------|
| **Engine Basics** | 引擎基础架构、应用生命周期 | 无 |
| **RHI Fundamentals** | RHI 抽象层设计 | Engine Basics |
| **Vulkan Backend** | Vulkan 后端实现 | RHI Fundamentals |
| **Rendering Pipeline** | 渲染管线实现 | Vulkan Backend |
| **Scene Management** | 场景系统设计 | Engine Basics |
| **Script System** | C# 脚本系统 | Engine Basics |
| **Animation System** | 骨骼动画系统 | Rendering Pipeline |

#### 章节模板

建议采用以下章节结构：

```markdown
# 教程标题

## 概述
简要说明本教程的目标和成果。

## 前置要求
- 前置教程链接
- 所需知识背景
- 开发环境要求

## 正文

### 第一节：[主题]
#### 子节：[具体内容]
[概念说明]
[代码片段]
[结果验证]

### 第二节：[主题]
...

## 总结
回顾本教程的关键知识点。

## 下一步
链接到下一个教程。

## 资源
- 示例代码下载
- 参考文档链接
```

### 2. 代码展示建议

#### 采用增量展示

```cpp
// 第一步：基础结构
class Renderer {
public:
    void Render();
};

// 第二步：添加相机支持
class Renderer {
public:
    void Render();
    void SetCamera(Camera* camera);  // 新增

private:
    Camera* m_camera = nullptr;      // 新增
};
```

#### 提供完整上下文

每个代码片段应包含：
- 所在文件路径
- 相关的头文件引用
- 命名空间上下文

#### 标记变更

使用注释标记被替换的代码：

```cpp
// void Draw();  // 旧版本
void Draw(RenderTarget* target);  // 新版本
```

### 3. 图文配合建议

#### 必备图像类型

| 类型 | 用途 | 工具 |
|------|------|------|
| 架构图 | 展示模块关系 | draw.io, Excalidraw |
| 流程图 | 展示执行流程 | draw.io |
| 渲染结果 | 验证实现效果 | 截图 |
| Frame Debugger | 验证渲染过程 | RenderDoc, 截图 |
| 性能数据 | 验证优化效果 | Profiler 截图 |

#### 图像组织原则

- 每个主要步骤后必须有验证图像
- 对比图像应并列展示
- 图像应有清晰的标题/说明

### 4. 实践设计建议

#### 建议添加练习题

与 Catlike Coding 不同，建议为 LitchiEngine 添加显式练习：

```markdown
## 练习

1. **基础**：实现一个简单的三角形渲染。
2. **进阶**：添加纹理支持。
3. **挑战**：实现多 Pass 渲染。

## 验证标准

- [ ] 三角形正确显示
- [ ] 纹理正确采样
- [ ] 多 Pass 正确执行
```

#### 提供完整示例项目

每个教程提供：
- 起始项目（如果需要）
- 完成项目
- 中间检查点（可选）

### 5. 难度递进建议

#### 三级难度设计

| 级别 | 内容 | 示例 |
|------|------|------|
| **入门** | 概念讲解 + 最小实现 | RHI 接口定义 |
| **进阶** | 完整实现 + 优化 | Vulkan 后端完整实现 |
| **高级** | 扩展功能 + 性能优化 | 多线程命令缓冲 |

#### 知识点分解原则

每个知识点应满足：
- 单一概念
- 可独立验证
- 有明确的输入输出

### 6. 本地化建议

考虑到 LitchiEngine 面向中文用户：

- 使用中文撰写教程
- 保留关键英文术语（如 RHI, Vulkan）
- 提供术语对照表

## 参考教程

本报告基于以下教程的深入分析：

### Unity Basics 系列
1. **Game Objects and Scripts** - 项目创建、编辑器基础、C# 脚本入门
2. **Building a Graph** - Prefab、实例化、Shader Graph 基础

### Custom SRP 系列
3. **Custom Render Pipeline** - 渲染管线架构、Camera Renderer、Command Buffers
4. **Draw Calls** - HLSL 着色器、SRP Batcher、GPU Instancing

### Pseudorandom Noise 系列
5. **Hashing** - 哈希函数实现、Job System、可视化
6. **Perlin Noise** - 梯度噪声、数学基础、多维实现

### Flow 系列
7. **Texture Distortion** - UV 动画、流动效果、Shader 优化

### Advanced Rendering 系列
8. **Triplanar Mapping** - 无 UV 纹理、投影技术、自定义 GUI

---

*报告日期：2026-04-11*
*调研工具：WebFetch, Claude AI*
*参考资料：https://catlikecoding.com/*
