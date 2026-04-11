# UI 系统分析

## 1. 模块概述

UI 系统位于 `Engine/Source/Runtime/Function/Framework/Component/UI/`，基于 ImGui 实现编辑器 UI，同时提供游戏内 UI 组件。UI 组件继承自 Component，可以挂载到 GameObject 上。

## 2. 架构设计

### 2.1 类关系

```
Component (组件基类)
    └── UI 组件
            ├── UICanvas (UI 画布)
            ├── RectTransform (UI 变换)
            ├── UIImage (UI 图片)
            └── UIText (UI 文本)

UIManager (UI 管理器)
    └── Canvas (画布)
            └── APanel (面板基类)
                    ├── PanelWindow (窗口面板)
                    └── ... (其他面板)
```

## 3. UICanvas - UI 画布

### 3.1 核心职责

- 作为 UI 元素的容器
- 管理 UI 渲染空间
- 初始化 UI 相机

### 3.2 类定义

```cpp
class UICanvas : public Component {
public:
    UICanvas();
    ~UICanvas() override;

    // 资源回调
    void PostResourceLoaded() override;

    // 分辨率
    void SetResolution(Vector2 resolution) { m_resolution = resolution; }
    Vector2 GetResolution() { return m_resolution; }

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;

private:
    void InitCanvasCamera();
    void InitCanvasTransform();

    Vector2 m_resolution;

    RTTR_ENABLE(Component)
};
```

### 3.3 初始化

```cpp
void UICanvas::OnAwake() {
    InitCanvasCamera();
    InitCanvasTransform();
}

void UICanvas::InitCanvasCamera() {
    // 获取或添加相机组件
    Camera* camera = GetGameObject()->GetComponent<Camera>();
    if (!camera) {
        camera = GetGameObject()->AddComponent<Camera>();
    }

    // 设置正交投影
    camera->SetProjectionType(ProjectionType::Orthographic);
    camera->SetNearPlane(0.0f);
    camera->SetFarPlane(100.0f);
}

void UICanvas::InitCanvasTransform() {
    // 设置 RectTransform
    RectTransform* rectTransform = GetGameObject()->GetComponent<RectTransform>();
    if (!rectTransform) {
        rectTransform = GetGameObject()->AddComponent<RectTransform>();
    }
}
```

## 4. RectTransform - UI 变换

### 4.1 核心职责

- 管理 UI 元素的位置和尺寸
- 支持锚点和轴心点
- 处理 UI 布局

### 4.2 类定义

```cpp
class RectTransform : public Transform {
public:
    RectTransform();
    ~RectTransform() override;

    // 锚点
    void SetAnchor(Vector2 anchor) { m_anchor = anchor; }
    Vector2 GetAnchor() { return m_anchor; }

    // 轴心点
    void SetPivot(Vector2 pivot) { m_pivot = pivot; }
    Vector2 GetPivot() { return m_pivot; }

    // 尺寸
    void SetSize(Vector2 size) { m_size = size; }
    Vector2 GetSize() { return m_size; }

    // 偏移
    void SetOffset(Vector2 offset) { m_offset = offset; }
    Vector2 GetOffset() { return m_offset; }

private:
    Vector2 m_anchor{0.5f, 0.5f};
    Vector2 m_pivot{0.5f, 0.5f};
    Vector2 m_size{100.0f, 100.0f};
    Vector2 m_offset{0.0f, 0.0f};

    RTTR_ENABLE(Transform)
};
```

## 5. UIImage - UI 图片

### 5.1 核心职责

- 显示图片
- 管理图片颜色
- 提供顶点/索引缓冲

### 5.2 类定义

```cpp
class UIImage : public Component {
public:
    UIImage();
    ~UIImage() override;

    // 资源路径
    void SetImagePath(std::string fontPath) { m_imagePath = fontPath; }
    std::string GetImagePath() { return m_imagePath; }

    // 纹理
    void SetTexture(RHI_Texture* texture2D) {
        m_texture2D = texture2D;
        m_imagePath = texture2D->GetResourceFilePathAsset();
    }
    RHI_Texture* GetTexture() { return m_texture2D; }

    // 颜色
    void SetColor(Color color) { m_color = color; }
    Color GetColor() { return m_color; }

    // 缓冲
    std::shared_ptr<RHI_VertexBuffer> GetVertexBuffer() { return m_vertexBuffer; }
    std::shared_ptr<RHI_IndexBuffer> GetIndexBuffer() { return m_indexBuffer; }

    // 资源回调
    void PostResourceModify() override;
    void PostResourceLoaded() override;

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;

private:
    void CreateBuffer();

    std::string m_imagePath;
    RHI_Texture* m_texture2D;
    Color m_color;
    std::shared_ptr<RHI_VertexBuffer> m_vertexBuffer;
    std::shared_ptr<RHI_IndexBuffer> m_indexBuffer;

    RTTR_ENABLE(Component)
};
```

### 5.3 创建缓冲

```cpp
void UIImage::CreateBuffer() {
    // 创建四边形顶点
    std::vector<RHI_Vertex_PosUv> vertices = {
        {{-0.5f, -0.5f, 0.0f}, {0.0f, 1.0f}},  // 左下
        {{ 0.5f, -0.5f, 0.0f}, {1.0f, 1.0f}},  // 右下
        {{ 0.5f,  0.5f, 0.0f}, {1.0f, 0.0f}},  // 右上
        {{-0.5f,  0.5f, 0.0f}, {0.0f, 0.0f}},  // 左上
    };

    // 创建索引
    std::vector<uint32_t> indices = {
        0, 1, 2,
        0, 2, 3
    };

    // 创建缓冲
    m_vertexBuffer = RHI_Device::CreateVertexBuffer(vertices);
    m_indexBuffer = RHI_Device::CreateIndexBuffer(indices);
}
```

## 6. UIText - UI 文本

### 6.1 核心职责

- 显示文本
- 管理字体和颜色
- 生成文本网格

### 6.2 类定义

```cpp
class UIText : public Component {
public:
    UIText();
    ~UIText() override;

    // 字体
    void SetFontPath(std::string fontPath) { m_fontPath = fontPath; }
    std::string GetFontPath() { return m_fontPath; }
    void SetFont(Font* font) { m_font = font; }
    Font* GetFont() { return m_font; }

    // 文本
    void SetText(std::string text);
    std::string GetText() { return m_text; }

    // 颜色
    void SetColor(Color color) { m_color = color; }
    Color GetColor() { return m_color; }

    // 缓冲
    std::shared_ptr<RHI_VertexBuffer> GetVertexBuffer() { return m_vertex_buffer; }
    std::shared_ptr<RHI_IndexBuffer> GetIndexBuffer() { return m_index_buffer; }

    // 资源回调
    void PostResourceModify() override;
    void PostResourceLoaded() override;

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;
    void OnEditorUpdate() override;
    void OnPreRender() override;
    void OnPostRender() override;

private:
    void CreateTextBuffer();
    void UpdateTextContent();

    std::string m_fontPath;
    Font* m_font{};
    std::string m_text;
    bool m_dirty{false};
    Color m_color;
    std::shared_ptr<RHI_VertexBuffer> m_vertex_buffer;
    std::shared_ptr<RHI_IndexBuffer> m_index_buffer;

    RTTR_ENABLE(Component)
};
```

### 6.3 文本网格生成

```cpp
void UIText::UpdateTextContent() {
    if (!m_font || m_text.empty()) return;

    std::vector<RHI_Vertex_PosUv> vertices;
    std::vector<uint32_t> indices;

    float x = 0.0f;
    float y = 0.0f;

    for (char c : m_text) {
        // 获取字符信息
        const FontChar& charInfo = m_font->GetChar(c);

        // 添加四边形顶点
        float x0 = x + charInfo.offsetX;
        float y0 = y + charInfo.offsetY;
        float x1 = x0 + charInfo.width;
        float y1 = y0 + charInfo.height;

        float u0 = charInfo.texCoordX;
        float v0 = charInfo.texCoordY;
        float u1 = u0 + charInfo.texWidth;
        float v1 = v0 + charInfo.texHeight;

        uint32_t baseIndex = static_cast<uint32_t>(vertices.size());

        vertices.push_back({{x0, y0, 0.0f}, {u0, v1}});
        vertices.push_back({{x1, y0, 0.0f}, {u1, v1}});
        vertices.push_back({{x1, y1, 0.0f}, {u1, v0}});
        vertices.push_back({{x0, y1, 0.0f}, {u0, v0}});

        indices.push_back(baseIndex + 0);
        indices.push_back(baseIndex + 1);
        indices.push_back(baseIndex + 2);
        indices.push_back(baseIndex + 0);
        indices.push_back(baseIndex + 2);
        indices.push_back(baseIndex + 3);

        // 移动到下一个字符
        x += charInfo.advanceX;
    }

    // 更新缓冲
    m_vertex_buffer->Update(vertices);
    m_index_buffer->Update(indices);
}
```

## 7. UI 渲染流程

### 7.1 渲染 Pass

```cpp
void Renderer::Pass_UIPass(RendererPath* renderer_path) {
    // 配置 PSO
    RHI_PipelineState pso;
    pso.render_target_color_textures[0] = renderer_path->GetRenderTarget()->GetColorTexture();
    pso.shader_vertex = m_shaders[Renderer_Shader::UI];
    pso.shader_pixel = m_shaders[Renderer_Shader::UI];
    pso.blend_state = m_blend_alpha;
    pso.depth_stencil_state = m_depth_stencil_off;

    RHI_CommandList* cmd = RHI_Device::CmdImmediateBegin(RHI_Queue_Type::Graphics);
    cmd->Begin();
    cmd->SetPipelineState(pso);

    // 渲染所有 UI 元素
    for (auto* canvas : GetUICanvases()) {
        for (auto* image : GetUIImages(canvas)) {
            DrawUIImage(cmd, image);
        }

        for (auto* text : GetUITexts(canvas)) {
            DrawUIText(cmd, text);
        }
    }

    cmd->End();
    RHI_Device::CmdImmediateSubmit(cmd);
}
```

### 7.2 绘制 UI 图片

```cpp
void DrawUIImage(RHI_CommandList* cmd, UIImage* image) {
    // 设置变换
    Matrix transform = image->GetGameObject()->GetComponent<RectTransform>()->GetMatrix();
    cmd->PushConstants(transform);

    // 绑定纹理
    cmd->SetTexture(0, image->GetTexture());

    // 绘制
    cmd->SetBufferVertex(image->GetVertexBuffer());
    cmd->SetBufferIndex(image->GetIndexBuffer());
    cmd->DrawIndexed(6);
}
```

## 8. ImGui 集成

### 8.1 UIManager

```cpp
class UIManager {
public:
    void Initialize();
    void Shutdown();
    void Render();

private:
    Canvas m_canvas;
};
```

### 8.2 Canvas

```cpp
class Canvas {
public:
    void AddPanel(APanel& p_panel);
    void RemovePanel(APanel& p_panel);
    void Draw();

private:
    std::vector<APanel*> m_panels;
};
```

### 8.3 APanel

```cpp
class APanel {
public:
    virtual void Draw() = 0;

    std::string name;
    bool opened = true;
};

class PanelWindow : public APanel {
public:
    void Draw() override {
        if (ImGui::Begin(name.c_str(), &opened)) {
            OnDraw();
        }
        ImGui::End();
    }

protected:
    virtual void OnDraw() {}
};
```

## 9. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 组件模式 | UI 组件 | UI 元素作为组件挂载 |
| 组合模式 | Canvas/Panel | UI 层级结构 |
| 模板方法模式 | PanelWindow::Draw | 统一绘制流程 |
| 外观模式 | UIManager | 封装 ImGui 操作 |

## 10. 使用示例

### 10.1 创建 UI 画布

```cpp
// 创建 UI 画布
auto* canvasObject = scene->CreateGameObject("UICanvas");
auto* canvas = canvasObject->AddComponent<UICanvas>();
canvas->SetResolution(Vector2(1920, 1080));
```

### 10.2 添加 UI 图片

```cpp
// 创建图片对象
auto* imageObject = scene->CreateGameObject("Image", true);  // isUI = true
imageObject->SetParent(canvasObject);

auto* image = imageObject->AddComponent<UIImage>();
image->SetTexture(textureManager->LoadResource("Textures/logo.png"));
image->SetColor(Color(1, 1, 1, 1));
```

### 10.3 添加 UI 文本

```cpp
// 创建文本对象
auto* textObject = scene->CreateGameObject("Text", true);
textObject->SetParent(canvasObject);

auto* text = textObject->AddComponent<UIText>();
text->SetFont(fontManager->LoadResource("Fonts/Arial.ttf"));
text->SetText("Hello World");
text->SetColor(Color(1, 1, 1, 1));
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
