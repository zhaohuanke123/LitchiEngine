# 动画系统分析

## 1. 模块概述

动画系统位于 `Engine/Source/Runtime/Function/Renderer/Rendering/Animation.h` 和 `Engine/Source/Runtime/Function/Framework/Component/Animation/`，实现骨骼动画功能。

## 2. 架构设计

### 2.1 类关系

```
Animation (动画资源)
    └── AnimationClip (动画片段)
            └── BoneAnimation (骨骼动画)
                    ├── VectorKey (位置关键帧)
                    ├── QuatKey (旋转关键帧)
                    └── VectorKey (缩放关键帧)

Animator (动画控制器组件)
    └── AnimationClipInfo (动画片段信息)

SkinnedMeshRenderer (蒙皮网格渲染器)
    ├── BoneInfo (骨骼信息)
    └── Cb_Bone_Arr (骨骼常量缓冲)
```

## 3. Animation - 动画资源

### 3.1 数据结构

```cpp
// 顶点权重
struct AnimationVertexWeight {
    uint32_t vertexID;
    float weight;
};

// 骨骼
struct AnimationBone {
    std::string name;
    std::vector<AnimationVertexWeight> vertexWeights;
    Matrix offset;
};

// 向量关键帧
struct KeyVector {
    double time;
    Vector3 value;
};

// 四元数关键帧
struct KeyQuaternion {
    double time;
    Quaternion value;
};

// 动画节点
struct AnimationNode {
    std::string name;
    std::vector<KeyVector> positionFrames;
    std::vector<KeyQuaternion> rotationFrames;
    std::vector<KeyVector> scaleFrames;
};

// 向量键
struct VectorKey {
    float timePos;
    Vector3 value;
};

// 四元数键
struct QuatKey {
    float timePos;
    Quaternion value;
};
```

### 3.2 BoneAnimation - 骨骼动画

```cpp
class BoneAnimation {
public:
    float GetStartTime() const;
    float GetEndTime() const;
    void Interpolate(float t, Matrix& M);

    std::vector<VectorKey> translation;
    std::vector<VectorKey> scale;
    std::vector<QuatKey> rotationQuat;
    Matrix defaultTransform;

private:
    Vector3 LerpKeys(float t, const std::vector<VectorKey>& keys);
    Quaternion LerpKeys(float t, const std::vector<QuatKey>& keys);
};
```

### 3.3 AnimationClip - 动画片段

```cpp
class AnimationClip {
public:
    float GetClipStartTime() const;
    float GetClipEndTime() const;
    void Interpolate(float t, std::vector<Matrix>& boneTransform);

    std::vector<BoneAnimation> boneAnimations;
};
```

### 3.4 BoneInfo - 骨骼信息

```cpp
struct BoneInfo {
    bool isSkinned = false;
    Matrix boneOffset;     // 骨骼空间到模型空间的变换矩阵 (Bind Pose)
    Matrix defaultOffset;  // 默认骨骼的变换矩阵
    int parentIndex;
};
```

## 4. Animator - 动画控制器

### 4.1 类定义

```cpp
struct AnimationClipInfo {
    std::string m_clipName;
    std::string m_clipPath;
    std::string m_selectClipResName;
    float m_startTime;
    float m_endTime;
    bool m_isLoop;
};

class Animator : public Component {
public:
    Animator();
    ~Animator();

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;
    void PostResourceLoaded() override;
    void PostResourceModify() override;

    // 播放控制
    bool Play(std::string clipName);

    // 获取当前状态
    AnimationClip& GetCurrentClip();
    std::string GetCurrentClipName() { return m_clipName; }
    float GetCurrentTimePos() { return m_timePos; }

    // 设置动画片段
    void SetAnimationClipMap(std::unordered_map<std::string, AnimationClip>& clipMap);

    // 动画片段信息数组
    std::vector<AnimationClipInfo> m_animationClipInfoArr;

private:
    float m_timePos = 0.0f;
    std::string m_clipName = "";
    AnimationClip m_emptyAnimationClip{};
    std::unordered_map<std::string, AnimationClip> m_animationClipMap{};

    RTTR_ENABLE(Component)
};
```

### 4.2 播放动画

```cpp
bool Animator::Play(std::string clipName) {
    auto it = m_animationClipMap.find(clipName);
    if (it == m_animationClipMap.end()) {
        return false;
    }

    m_clipName = clipName;
    m_timePos = 0.0f;

    return true;
}
```

### 4.3 更新动画

```cpp
void Animator::OnUpdate() {
    if (m_clipName.empty()) return;

    auto it = m_animationClipMap.find(m_clipName);
    if (it == m_animationClipMap.end()) return;

    AnimationClip& clip = it->second;

    // 更新时间
    m_timePos += Time::GetDeltaTime();

    // 检查循环
    float endTime = clip.GetClipEndTime();
    if (m_timePos > endTime) {
        // 查找是否循环
        for (auto& info : m_animationClipInfoArr) {
            if (info.m_clipName == m_clipName && info.m_isLoop) {
                m_timePos = clip.GetClipStartTime();
                break;
            }
        }
    }
}
```

## 5. SkinnedMeshRenderer - 蒙皮网格渲染器

### 5.1 类定义

```cpp
class SkinnedMeshRenderer : public MeshRenderer {
public:
    SkinnedMeshRenderer();
    ~SkinnedMeshRenderer() override;

    // 骨骼常量缓冲
    std::shared_ptr<RHI_ConstantBuffer> GetBoneConstantBuffer() { return m_boneConstantBuffer; }

    // 类型
    MeshRendererType GetMeshRendererType() const override { return MeshRendererType::SkinMeshRenderer; }

    // 材质
    void SetDefaultMaterial() override;

    // 资源回调
    void PostResourceModify() override;
    void PostResourceLoaded() override;

    // 生命周期
    void OnAwake() override;
    void OnUpdate() override;

private:
    // 创建骨骼缓冲
    void CreateBoneBuffer();
    void CreateDefaultBoneBuffer();

    // 计算骨骼变换
    void CalcDefaultFinalTransform(std::vector<int>& boneHierarchy, std::vector<Matrix>& nodelDefaultTransforms);
    void CalcFinalTransform(float timePos, AnimationClip& clip, std::vector<int>& boneHierarchy, std::vector<Matrix>& boneOffsets);

    // 查找动画控制器
    Animator* FindAnimatorInHierarchy();

    bool m_isDirty = true;
    bool m_isFirstTick = true;
    Cb_Bone_Arr m_boneArr;
    std::shared_ptr<RHI_ConstantBuffer> m_boneConstantBuffer;

    RTTR_ENABLE(MeshRenderer)
};
```

### 5.2 计算骨骼变换

```cpp
void SkinnedMeshRenderer::CalcFinalTransform(float timePos, AnimationClip& clip, 
                                               std::vector<int>& boneHierarchy, 
                                               std::vector<Matrix>& boneOffsets) {
    // 获取骨骼动画变换
    std::vector<Matrix> boneTransforms;
    clip.Interpolate(timePos, boneTransforms);

    // 计算最终变换
    for (size_t i = 0; i < boneHierarchy.size(); i++) {
        Matrix toParent = boneTransforms[i];
        Matrix toRoot = toParent;

        // 应用父骨骼变换
        int parentIndex = boneHierarchy[i];
        if (parentIndex != -1) {
            toRoot = toParent * m_boneArr.boneTransformArr[parentIndex];
        }

        // 存储最终变换
        m_boneArr.boneTransformArr[i] = toRoot;

        // 应用骨骼偏移
        m_boneArr.boneTransformArr[i] = boneOffsets[i] * toRoot;
    }

    // 更新常量缓冲
    m_boneConstantBuffer->Update(&m_boneArr);
}
```

### 5.3 更新蒙皮

```cpp
void SkinnedMeshRenderer::OnUpdate() {
    // 查找动画控制器
    Animator* animator = FindAnimatorInHierarchy();
    if (!animator) return;

    // 获取当前动画片段
    AnimationClip& clip = animator->GetCurrentClip();
    if (clip.boneAnimations.empty()) return;

    // 计算骨骼变换
    float timePos = animator->GetCurrentTimePos();
    CalcFinalTransform(timePos, clip, m_boneHierarchy, m_boneOffsets);
}
```

## 6. 骨骼动画插值

### 6.1 位置插值

```cpp
Vector3 BoneAnimation::LerpKeys(float t, const std::vector<VectorKey>& keys) {
    if (keys.empty()) return Vector3::Zero;
    if (keys.size() == 1) return keys[0].value;

    // 找到相邻的两个关键帧
    size_t i = 0;
    while (i < keys.size() - 1 && keys[i + 1].timePos < t) {
        i++;
    }

    // 计算插值因子
    float t0 = keys[i].timePos;
    float t1 = keys[i + 1].timePos;
    float lerpFactor = (t - t0) / (t1 - t0);

    // 线性插值
    return Vector3::Lerp(keys[i].value, keys[i + 1].value, lerpFactor);
}
```

### 6.2 旋转插值

```cpp
Quaternion BoneAnimation::LerpKeys(float t, const std::vector<QuatKey>& keys) {
    if (keys.empty()) return Quaternion::Identity;
    if (keys.size() == 1) return keys[0].value;

    // 找到相邻的两个关键帧
    size_t i = 0;
    while (i < keys.size() - 1 && keys[i + 1].timePos < t) {
        i++;
    }

    // 计算插值因子
    float t0 = keys[i].timePos;
    float t1 = keys[i + 1].timePos;
    float lerpFactor = (t - t0) / (t1 - t0);

    // 球面线性插值 (SLERP)
    return Quaternion::Slerp(keys[i].value, keys[i + 1].value, lerpFactor);
}
```

### 6.3 骨骼变换插值

```cpp
void BoneAnimation::Interpolate(float t, Matrix& M) {
    // 插值位置、旋转、缩放
    Vector3 position = LerpKeys(t, translation);
    Quaternion rotation = LerpKeys(t, rotationQuat);
    Vector3 scale = LerpKeys(t, this->scale);

    // 构建变换矩阵
    M = Matrix::CreateScale(scale) *
        Matrix::CreateFromQuaternion(rotation) *
        Matrix::CreateTranslation(position);
}
```

## 7. 着色器集成

### 7.1 骨骼常量缓冲

```hlsl
const static int MaxBone = 512;
struct BoneDataArr {
    matrix boneTransformArr[MaxBone];
};

cbuffer BoneDataArr : register(b4) {
    BoneDataArr bone_data_arr;
}
```

### 7.2 蒙皮着色器

```hlsl
// 顶点输入
struct Vertex_PosUvNorTanBone {
    float3 position : POSITION;
    float2 uv : TEXCOORD;
    float3 normal : NORMAL;
    float3 tangent : TANGENT;
    int4 boneIndices : BONEINDICES;
    float4 boneWeights : BONEWEIGHTS;
};

// 顶点着色器
Pixel mainVS(Vertex_PosUvNorTanBone input) {
    Pixel output;

    // 蒙皮变换
    float4 pos = float4(input.position, 1.0f);
    float4 skinnedPos = float4(0, 0, 0, 0);
    
    for (int i = 0; i < 4; i++) {
        int boneIndex = input.boneIndices[i];
        float weight = input.boneWeights[i];
        skinnedPos += mul(pos, bone_data_arr.boneTransformArr[boneIndex]) * weight;
    }

    // 变换到裁剪空间
    output.position = mul(skinnedPos, buffer_pass.transform);
    output.position = mul(output.position, buffer_rendererPath.view_projection_unjittered);
    output.fragPos = skinnedPos.xyz;
    output.uv = input.uv;
    output.normal = input.normal;
    output.tangent = input.tangent;

    return output;
}
```

## 8. 设计模式总结

| 设计模式 | 应用位置 | 说明 |
|----------|----------|------|
| 组件模式 | Animator, SkinnedMeshRenderer | 分离动画逻辑和渲染 |
| 策略模式 | AnimationClip | 不同动画片段切换 |
| 插值模式 | LerpKeys | 平滑动画过渡 |

## 9. 使用示例

### 9.1 添加动画组件

```cpp
// 添加 SkinnedMeshRenderer
auto* skinnedRenderer = gameObject->AddComponent<SkinnedMeshRenderer>();

// 添加 Animator
auto* animator = gameObject->AddComponent<Animator>();

// 设置动画片段
std::unordered_map<std::string, AnimationClip> clips;
clips["Idle"] = idleClip;
clips["Walk"] = walkClip;
clips["Run"] = runClip;
animator->SetAnimationClipMap(clips);
```

### 9.2 播放动画

```cpp
// 播放动画
animator->Play("Walk");

// 切换动画
animator->Play("Run");
```

---

**分析完成时间**: 2026-04-10
**分析者**: AI Agent
