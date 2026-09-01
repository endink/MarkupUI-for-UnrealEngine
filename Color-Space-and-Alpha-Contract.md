# Draw 色彩空间与 Alpha 契约

## 目标

本文规定插件默认 Draw 路径中的颜色、纹理、离屏 Layer 和最终 Target 语义。

当前首要目标是逐像素对齐 RmlUi 官方参考后端，并保持普通 UI 的直接绘制性能。默认路径不引入页面级线性 RenderTarget，也不增加最终全屏合成 pass。

Alpha 始终表示线性覆盖率，不参与 sRGB 或 Gamma 转换。

## 默认颜色模式：Encoded Direct

RmlUi 6.3 官方 DX12 后端使用 `DXGI_FORMAT_R8G8B8A8_UNORM`，不创建 sRGB SRV/RTV，也不进行显式 sRGB 与 Linear 转换。顶点颜色、图片、gradient、filter、blur 和 blend 均按 encoded UNORM 数值处理。

插件默认遵循相同语义：

```text
RmlUi encoded、Premultiplied RGBA
        ↓ 不进行 sRGB → Linear 转换
shader / gradient / filter / blur / layer composite
        ↓ Premultiplied Alpha 混合
Slate Output
```

默认路径必须满足：

1. RmlUi 顶点颜色保留原始字节，不在 Frontend、Pipeline 或 Target 中线性化。
2. RmlUi 图片和生成纹理按 encoded UNORM 数值采样。
3. 普通绘制直接写入 Slate Target，不创建页面级离屏目标。
4. 仅 RCSS 效果本身要求 Layer 时按需创建局部离屏目标。
5. Gradient、filter、blur 和 layer composite 首先复现官方后端的 encoded 运算语义。
6. 资源仍携带颜色空间和 Alpha 元数据，但 `EncodedDirect` 不得因 `ColorSpace` 自动改变采样结果。

## Alpha 契约

RmlUi 几何和生成纹理使用 Premultiplied Alpha。普通图片通常使用 Straight Alpha。

```cpp
enum class EDrawTextureAlphaMode : uint8
{
    Opaque,
    Straight,
    Premultiplied,
};
```

Straight Alpha 图片在 shader 中只预乘一次：

```hlsl
float4 texture_color = Texture.Sample(TextureSampler, uv);

if (TextureAlphaMode == Straight)
{
    texture_color.rgb *= texture_color.a;
}

return vertex_color * texture_color;
```

普通 Blend State 使用：

```text
RGB:   One / InverseSourceAlpha / Add
Alpha: One / InverseSourceAlpha / Add
```

即：

```text
Out = Source + Destination × (1 - SourceAlpha)
```

不得对 Premultiplied 输出再次使用 `SourceAlpha` 作为 RGB SourceFactor。

## 资源元数据

颜色空间和 Alpha 是两个独立维度：

```cpp
enum class EDrawTextureColorSpace : uint8
{
    Linear,
    SRGB,
};
```

```cpp
class IDrawTextureResource
{
public:
    virtual FIntPoint GetSize() const = 0;
    virtual EDrawTextureColorSpace GetColorSpace() const = 0;
    virtual EDrawTextureAlphaMode GetAlphaMode() const = 0;
};
```

元数据用于描述资源来源，并为未来 Material、Image、HDR 或可选线性 Target 提供依据。默认 `EncodedDirect` Slate 路径只依据 `AlphaMode` 决定是否预乘，不依据 `ColorSpace` 自动创建 sRGB 采样视图。

| 资源 | ColorSpace 元数据 | AlphaMode | 默认 Slate 解释 |
| --- | --- | --- | --- |
| RmlUi 顶点颜色 | encoded sRGB 语义 | Premultiplied | 原始字节 |
| RmlUi `GenerateTexture()` | SRGB | Premultiplied | encoded UNORM |
| 字体 atlas | SRGB 元数据；coverage 主要位于 Alpha | Premultiplied | encoded UNORM，Alpha 不转换 |
| PNG | SRGB | Straight | encoded 采样后预乘一次 |
| JPEG | SRGB | Opaque | encoded 采样 |
| `UTexture2D` | 遵守资产声明 | 默认 Straight | 默认兼容路径必须取得 encoded 数值 |
| Mask/data texture | Linear | Straight 或 Opaque | 按数据语义读取 |
| encoded 离屏 Layer | SRGB | Premultiplied | 与官方后端一致地保存和采样 |

## 离屏 Layer

默认路径不会仅为颜色空间创建页面级离屏目标。

只有这些功能本身需要时才创建 Layer：

- `filter`
- `backdrop-filter`
- `mask-image`
- `box-shadow` / drop-shadow
- RmlUi 的显式 layer 保存与合成

基础规则：

- 使用 encoded `RGBA8/BGRA8`、Premultiplied Alpha。
- 清除为透明黑 `(0, 0, 0, 0)`。
- 尺寸限制为实际效果 bounds，不默认覆盖整个页面。
- RenderTarget 通过池复用。
- Mask 优先使用 stencil 或 coverage 目标，不无条件创建彩色 Layer。

## 可选线性模式

未来可以增加 `LinearComposited`，但它不是默认路径，也不是完成 RCSS 支持的前置条件。

```cpp
enum class EDrawColorMode : uint8
{
    EncodedDirect,
    LinearComposited,
};
```

线性模式需要完整链路：

```text
encoded 输入
    ↓ 解码
Linear Premultiplied 中间目标
    ↓ 所有效果和混合
最终显示编码
    ↓
Target
```

不得再次实现只有输入解码、没有最终编码的半条链路。线性模式还必须具备独立视觉基准、RenderTarget 池、资源采样视图和 HDR/输出约定。

## 各层职责

### Frontend

- 原样转发 RmlUi 顶点颜色和生成纹理。
- 声明资源的 ColorSpace 和 AlphaMode。
- 不决定 Slate 的 GPU 纹理格式或采样视图。

### Pipeline

- 保留命令和资源元数据。
- 默认使用 `EncodedDirect`。
- 不在 geometry compile 阶段量化或转换颜色。
- 未来若支持双模式，在不可变帧快照中记录模式。

### Target

- 默认直接写 Slate 输出。
- 将 Straight Alpha 转为 Premultiplied。
- 使用 Premultiplied Blend State。
- 仅在效果需要时创建局部 Layer。
- 未来的线性/HDR 模式必须显式选择，不能改变默认兼容结果。

## 验收

- [ ] UE 输出与 RmlUi 官方 Viewer 的纯色、渐变、图片和文字颜色一致。
- [ ] 50% Alpha 色块的混合结果一致。
- [ ] PNG 半透明边缘无重复乘 Alpha 导致的暗边。
- [ ] 字体 atlas 在不同背景上无异常明暗变化。
- [ ] 普通页面不创建页面级颜色 RenderTarget。
- [ ] 未使用 filter/layer 的页面不增加全屏合成 pass。
- [ ] Layer 保存再采样与官方参考后端一致。
- [ ] ColorSpace 元数据不会改变默认 `EncodedDirect` 的像素结果。

## 当前完成状态

- [x] 资源携带 `EDrawTextureColorSpace`。
- [x] 资源携带 `EDrawTextureAlphaMode`。
- [x] Slate shader 支持 Straight → Premultiplied。
- [x] 普通 Blend State 使用 `One / InverseSourceAlpha`。
- [x] RmlUi 顶点颜色恢复为原始 encoded 字节。
- [x] 默认原始像素纹理不创建 sRGB RHI 采样资源。
- [x] 默认路径不创建页面级线性离屏目标。
- [x] `UTexture2D` 在默认路径中通过禁用 sRGB 解码的 SRV 取得 encoded 采样值，不修改资产自身的 `SRGB` 设置。
- [ ] 建立与官方 Viewer 的截图回归测试。
- [ ] 按官方 encoded 语义实现 Layer、filter 和 gradient。
