# RCSS 渲染支持完成度与开发路线

## 目标

以当前集成的 RmlUi 6.3 为基准，补齐 UE 插件对内置 RCSS 视觉语义的支持，同时保持以下架构边界：

- `FRmlLanguageFrontend` 只负责 RmlUi C API、Context、Document 和回调适配。
- `FUnrealRHIDrawPipeline` 组织 Frontend、通用绘制状态、资源句柄和命令流。
- `IDrawTarget` 负责消费通用绘制帧；Slate 只是其中一种 Target。
- 高级视觉能力必须先进入通用 Draw 协议，不能直接写死在 Slate Widget 或 Slate 后端中。

## 当前结论

当前插件的 RCSS 解析、层叠、选择器和布局能力由完整的 RmlUi 6.3 核心提供。默认 Slate Target 已真实执行普通几何、字体、图片、scissor、stencil clip、预乘 Alpha、GPU 透视变换、RDG layer、Blend/Replace composite、保存纹理与 alpha mask、全部内置 filter、backdrop-filter、box-shadow 和六种内置 gradient。阶段 1–11 的视觉测试已与 RmlUi Document Viewer 对齐；当前明确未实现的视觉扩展仅剩自定义 `decorator: shader(...)`。

当前完成度估计：

| 范围 | 完成度 |
| --- | ---: |
| RCSS 解析、层叠、选择器、布局 | 由 RmlUi 6.3 核心完整提供 |
| 普通几何、字体、图片、scissor | 已完成 |
| 精确基础渲染语义 | 已完成并完成视觉对比 |
| RmlUi 6.3 内置高级视觉效果 | 已完成并完成视觉对比 |
| 自定义 Shader Decorator | 未实现 |
| 形式化 RCSS 一致性测试套件 | 尚未建立，不能据此宣称绝对 100% |

当前里程碑状态：

- 已关闭：任务 0.1–0.6、任务 0.8、阶段 1–11。
- 核心完成但保留后期优化：任务 0.7 编译几何复用。
- 阶段 12 的代码侧优化与自动化验证已完成；真实页面 GPU 时间仍待固定场景外部采样。
- 当前功能主线：阶段 13 自定义 Shader Decorator。

架构重构已经承载 RCSS 高级像素能力：局部离屏 Layer、Blend/Replace composite、颜色与空间 filter、backdrop-filter、Layer 保存、alpha mask、box-shadow 和六种内置 gradient 已进入真实 RDG 执行。资源路径完整性已经关闭，后续主要功能缺口转为生命周期与视觉回归收尾，以及最后阶段的自定义 Shader Decorator。

## 当前实现校对

### 已有能力

- RmlUi 6.3 完整解析 RCSS 的选择器、层叠、变量、普通属性、Flex、transition、animation 和 transform。
- Pipeline 已接收 geometry、texture、scissor、clip mask、transform、layer、filter 和 shader 回调。
- `FDrawFrame` 已使用 `EMarkupDrawCommand + IDrawCommand` typed command 表达 geometry、clip mask、layer、composite、save layer 和 shader draw。
- 每种具体 command 位于独立文件中，自行声明稳定的 command type，并通过统一接口提供克隆和可选 geometry 访问；命令流不再依赖 `TVariant` 运行时类型探测。
- Filter/Shader 参数已在 C ABI 边界转换为 string、float、bool、vector2、color 和 color-stop-list，不再向通用层泄漏 RmlUi Variant type tag。
- Target 能力契约保证只有当前 Target 可执行的 Filter/Shader 才会获得非零 handle。
- 通用绘制帧已经将命令和纹理资源快照提交给 `IDrawTarget`。
- Pipeline 与 Target 共享 `FMarkupRenderingHealth`；结构化错误按稳定键只报告一次，灾难性错误停止重复 GPU 提交，并在文档、命令或 Target 代次变化后按策略恢复。
- 默认 Slate Target 已能绘制普通带纹理三角形。
- 默认 Slate Target 已正确执行 premultiplied alpha、stencil `Set`、`SetInverse` 和 `Intersect`。
- Transform 使用完整 GPU 4×4 路径，保留 clip-space `W` 和透视正确插值。
- `FCompiledDrawGeometry` 以不可变共享资源表达编译几何，命令克隆不再复制完整顶点和索引。
- Slate 每帧打包时会复用相同编译几何，避免同一 Geometry 在一帧内重复转换和上传。
- `FMarkupFilterCompiler` 已从 Pipeline 中独立，负责参数校验、默认值、程序选择、颜色矩阵和阴影参数预计算。
- 磁盘资源 Provider 已支持直接读取 RML、RCSS、字体以及由 ImageWrapper 解码的图片。
- Unreal Asset Provider 已支持 `URmlDocument`、`URmlStyleSheet`、`UFontFace` 和 `UTexture2D`。

### 尚未完成的高级像素执行

Pipeline 已能记录完整逻辑描述和操作顺序。默认 Slate Target 当前真实执行 bounded RDG layer、
Blend/Replace composite、source-layer stencil 裁剪快照、全部内置 filter、保存纹理、保存 alpha mask，以及 linear、radial、conic 和对应 repeating gradient。
尚未注册的自定义 shader 请求仍返回失败，不会退化成普通 geometry 或产生“成功但无效果”的伪 handle。

默认 Slate 普通 geometry 像素着色器仍然只有以下语义：

```hlsl
return Input.Color * Texture.Sample(TextureSampler, Input.UV);
```

因此当前主要尚未完成的 shader 能力是自定义 `decorator: shader(...)`；内置 gradient 已使用独立像素着色路径执行。

## 对旧结论的修正

### 资源系统限制已经减小

标准 ResourceHost 当前同时注册 Core Style 字体、磁盘和 Unreal Asset Provider。磁盘模式已经能读取普通文件、字体、PNG、JPEG 等资源，不再局限于 Unreal Asset。

Unreal Asset 模式支持配置项目或插件内容挂载点作为资产资源根目录。磁盘与资产资源域均限制在各自配置的根目录内，且不会自动跨域回退查找。

### Clip mask 已完成基础语义

当前 stencil 路径已实现 RmlUi 的三种 clip mask 操作：

- `Set`：清除旧 mask 后写入新区域。
- `SetInverse`：清除为有效区域，再将几何覆盖区域写为无效。
- `Intersect`：通过递增 stencil reference 保留连续 mask 的交集。

连续圆角、inverse 和 intersect 已进入视觉测试页；基础 saved mask 也已通过 L07/L08 的 UE 与 Viewer 对比。后续仍需验证 mask 与 scissor、圆角 clip、transform 的组合行为。

### Premultiplied alpha 已按官方语义执行

RmlUi 顶点颜色和生成纹理使用 premultiplied alpha。普通颜色混合当前使用：

```text
SourceFactor      = One
DestinationFactor = InverseSourceAlpha
```

默认 Slate 路径采用 encoded-UNORM 采样，不额外执行 sRGB 到 Linear 转换；磁盘纹理通过禁用 SRV 的隐式 sRGB 解码与 RmlUi 参考后端保持一致。

### Transform 已使用 GPU 透视路径

Pipeline 将 RmlUi 的 translation 与 4×4 transform 保留在通用 DrawCommand 中。Slate Target 将自己的二维 PaintTransform 与其组合，并在顶点 Shader 中输出完整 clip-space `XYZW`，由 GPU 完成齐次除法以及透视正确的 UV、颜色插值。

## 实现 Layer 前的前置任务

不能直接从 Slate 离屏 RenderTarget 开始实现。应先补齐通用 Draw 数据模型，否则高级效果会重新耦合到 Slate。

### 任务 0.1：修正基础绘制正确性（已完成）

- [x] 将普通绘制改为正确的 premultiplied alpha blend。
- [x] 正确实现 stencil `Set`。
- [x] 正确实现 stencil `SetInverse`。
- [x] 正确实现 stencil `Intersect`。
- [x] 默认绘制采用与 RmlUi 官方一致的 encoded-UNORM + Premultiplied Alpha 契约；资源保留 ColorSpace/AlphaMode 元数据，但默认 Slate 路径不触发线性转换。线性/HDR 工作流仅作为未来可选策略。
- [x] 实现 GPU 4×4 transform 路径，保留 clip-space `W`，同时支持二维 affine 与 perspective transform。

验收标准：

- 半透明文字、图片边缘和圆角色块不出现二次乘 alpha 导致的暗边。
- 连续嵌套圆角裁剪、inverse clip 和 intersect clip 与 RmlUi 参考后端截图一致。

### 任务 0.2：扩展通用绘制命令协议（已完成）

命令流至少需要表达：

```cpp
enum class EMarkupDrawCommand : uint8
{
    DrawGeometry,
    SetClipMask,
    PushLayer,
    PopLayer,
    CompositeLayers,
    SaveLayerAsTexture,
    SaveLayerAsMask,
    DrawShader,
};
```

每种操作使用独立 command 类型并实现轻量 `IDrawCommand` 接口。具体 command 通过静态
`CommandType` 标记自身类型，消费者按 `EMarkupDrawCommand` 分派；不把所有可选字段堆入
单一结构，也不使用 `TVariant::IsType/Get` 链进行运行时类型探测。

- [x] layer 操作进入 `FDrawFrame`，而不是只保存在 Pipeline 的临时集合中。
- [x] 每条 geometry 命令能确定目标 layer。
- [x] composite 命令保存 source、destination、blend mode 和 filter 列表。
- [x] shader draw 命令保存 shader handle、geometry、translation 和 texture。
- [x] 帧快照包含执行命令所需的 filter、shader 和持久资源描述。
- [x] 每种 command 拆分到 `Rendering/Commands` 独立文件，并提供英文职责注释。
- [x] 命令列表通过 `TUniquePtr<IDrawCommand>` 表达所有权，Target 适配通过 `Clone()` 创建独立快照。

验收标准：

- 一个非 Slate 的测试 Target 可以只读取 `FDrawFrame`，完整重放 layer/filter/shader 操作顺序。
- Target 不需要访问 `FRmlLanguageFrontend` 或任何 RmlUi 类型。

### 任务 0.3：保存 Filter 和 Shader 的真实描述（已完成）

将当前集合：

```cpp
TSet<FDrawHandle> Filters;
TSet<FDrawHandle> Shaders;
```

替换为带真实内容的描述表，例如：

```cpp
TMap<FDrawHandle, FCompiledDrawFilter> Filters;
TMap<FDrawHandle, FDrawShaderDescriptor> Shaders;
```

Filter 描述至少需要表达：

- opacity
- brightness
- contrast
- grayscale
- invert
- sepia
- hue-rotate
- saturate
- blur sigma
- drop-shadow color、offset、sigma
- mask image

Shader 描述至少需要表达：

- linear gradient
- radial gradient
- conic gradient
- repeating
- center、radius、p0、p1、angle
- 完整 color stop list
- 自定义 shader 名称和参数

- [x] Filter handle 映射到 `FCompiledDrawFilter`，包含标准化执行程序、预计算颜色矩阵、
  opacity、blur sigma、预乘 drop-shadow 参数或 mask source layer。
- [x] Shader handle 映射到内置渐变或自定义 shader 的真实描述。
- [x] 帧快照保留执行本帧命令所需的描述，Release 不会破坏已提交帧。
- [x] Filter 编译成功和 Release 使用 Debug 日志记录；解析、参数或 Target 能力失败使用 Error 日志。

### 任务 0.4：结构化 C API 渲染参数（已完成）

旧 C API 曾把 `Rml::Variant` 转换成字符串并附带内部 type tag；ABI v15 已改为结构化值。

- [x] 为 string、float、bool、vector2、color 和 color-stop-list 定义明确的 C ABI 数据结构。
- [x] 在 C API 边界完成 RmlUi 类型到通用 Draw 类型的转换。
- [x] 通用 Pipeline 不保存或解释 RmlUi 的 `Variant` type tag。
- [x] 复杂参数数组只在同步回调期间借用，Adapter 在回调内完成深拷贝。

验收标准：

- 多 color stop 渐变经过 C API 后，stop 数量、位置、颜色和顺序完全不丢失。

### 任务 0.5：建立能力与失败规则（已完成）

非零 handle 现在表示当前 Target 已明确承诺能够执行对应效果。

- [x] 只有已知、参数有效且当前 Target 声明可执行的 filter 才返回有效 handle。
- [x] 只有已知、参数有效且当前 Target 声明可执行的 shader 才返回有效 handle。
- [x] 未知、参数无效或 Target 不支持时返回 `0` 并输出一次明确诊断。
- [x] Release 回调忽略零 handle，并对无效或重复释放保持安全且只诊断一次。

### 任务 0.6：标准化 Unreal Imported Asset 集成（已完成）

该任务属于编辑器资源工作流，不直接提高 RCSS 像素效果完成度，但保证 RML 和 RCSS 资产
遵循 Unreal Content Browser 的标准导入、源文件和重导入契约。

- [x] RML Document 和 RCSS Style Sheet 的 Asset Definition 通过 `CanImport()` 声明为导入型资产。
- [x] 两种资产将 `AssetImportData` 发布为标准 `SourceFile` Asset Registry 标签。
- [x] `GetSourceFiles()` 优先读取 Asset Registry，并兼容尚未重存标签的旧资产。
- [x] 删除插件自定义 Content Browser Reimport 菜单，由 `FAssetFileContextMenu` 自动接管。
- [x] 保留 Factory 的 `FReimportHandler`，作为 `FReimportManager` 调用的实际重导入后端。
- [x] 移除不再需要的 `ContentBrowser` 和 `ToolMenus` 模块依赖。

验收标准：

- RML 和 RCSS 资产在 Content Browser 中与其他 Unreal imported asset 一致地提供 Reimport、
  Reimport With New File、Open Source Location 和批量操作。
- 菜单层不包含 MarkupUI 自定义重导入分发逻辑。

### 任务 0.7：标准化编译几何复用（核心完成，跨帧 Cache 待性能评估）

该任务不增加 RCSS 属性范围，但确保 RmlUi 的 `CompileGeometry` 真正具有跨 Draw 复用语义，
避免静态页面在命令构造和 Slate 快照阶段反复复制同一份顶点和索引。

- [x] 使用不可变 `FCompiledDrawGeometry` 保存顶点、索引和局部 Bounds。
- [x] Pipeline、Draw Command 和 Slate 快照通过共享引用维持异步渲染生命周期。
- [x] `ReleaseGeometry` 只解除 Pipeline 持有关系，不破坏已提交 Frame。
- [x] Slate 在单帧聚合 Buffer 中按编译几何去重，相同 Geometry 多次 Draw 只上传一次。
- [ ] 根据性能统计决定是否增加 Target 侧跨帧持久 GPU Geometry Cache。

验收标准：

- 同一编译 Geometry 在一帧中以不同 translation、transform、texture 或 clip 状态绘制时，
  只生成一份打包顶点和索引。
- Frontend 在 Render 回调结束前释放 Geometry 时，RenderThread 仍能安全消费已提交 Frame。

### 任务 0.8：渲染健康、错误隔离与恢复（已完成）

渲染错误不能使用包含动态 handle 的完整日志文本去重，也不能让同一灾难性命令流每帧重复创建 GPU 工作。

- [x] 引入 Pipeline/Target 共享且线程安全的 `FMarkupRenderingHealth`。
- [x] 使用 fault、command index 和 command type 组成稳定错误键，不使用每帧变化的 Layer handle。
- [x] 每个文档内相同稳定错误只输出一次 Error，后续重复错误保持静默。
- [x] 为错误声明 Frame、Command、Target、Command-or-Target 和 Document 生命周期。
- [x] 灾难性错误锁定当前故障代次，后续绘制在进入 RDG/GPU 前停止。
- [x] 文档重载、Frontend 更新、viewport/DPI、Widget clip 和 Rendering Settings 变化推进对应恢复代次。
- [x] Render Thread 使用 Frame 捕获的 generation token 报告成功或失败，过期 Frame 不能覆盖新状态。
- [x] 恢复后的第一帧成功时只输出一条 Debug 诊断。
- [x] 零面积或完全位于可见区域外的 Layer 作为正常空工作跳过，不记为错误。
- [x] 完全裁剪的 mask-image 保留有效逻辑 filter handle，但不创建零尺寸 GPU 资源或保存命令，
  后续空区域 composite 正常跳过且不输出错误。

验收标准：

- 同一非法 Layer 命令流持续存在时，只在首次失败时输出一条 Error，后续帧不重复提交 GPU 绘制。
- 修正文档、更新命令或改变相关 Target 状态后能够自动重试，不要求重启 Editor。
- 文档级结构损坏在重新加载文档前保持隔离，其他 Widget 和 Pipeline 不受影响。
- `MarkupUI.Rendering.Pipeline.EmptyMaskImage` Automation Test 已验证空 mask 不产生保存命令、
  持久资源或错误日志。

## 高级视觉支持开发阶段

### 阶段 1：通用 Layer Stack

- [x] Pipeline 使用保留 handle `0` 表达 base layer。
- [x] `PushLayer` / `PopLayer` 已进入命令流，并记录 layer 与 parent layer。
- [x] Pipeline 检测 Pop 下溢和 Frame 结束时的栈不平衡；非法帧不会提交给 Target。
- [x] Slate Target 将 `PushLayer` / `PopLayer` 解析为经过二次校验的 Layer 表与有序 raster segments，
  并实际切换对应 RDG color/resolve/stencil。
- [x] `PushLayer` 捕获当前有效 Bounds；有 scissor 时取 `Scissor ∩ Viewport`，否则使用完整 Viewport。
- [x] Layer 命令记录 Bounds 和 Origin，使 Target 能在页面坐标与局部纹理坐标间转换。
- [x] 新 layer 首次写入或读取前在有效 Bounds 内初始化为透明黑。
- [x] Slate RDG Layer 记录 bounds、尺寸、坐标原点和空区域状态，窗口裁剪后同步修正局部原点。
- [x] 完成 Slate RDG 临时 Layer 与 RenderTarget Pool 设计；帧内资源使用 RDG transient pool，
  只有 save 结果 extraction 为跨帧 `IPooledRenderTarget`。详见
  [Slate RDG Layer 执行与资源复用设计](Slate-RDG-Layer-Design.md)。
- [x] 按设计建立 Target 侧 RDG Layer executor；不重复实现 RDG 已有的帧内纹理池。
- [x] 支持窗口 resize、DPI 改变和 Target 销毁；Pipeline 只在 viewport 或 pixel ratio 实际变化时更新 Frontend，
  Target 的弱引用失效后停止生成新 Frame，已提交 Frame 继续由 RenderThread 安全消费。
- [x] 支持 RHI 资源失效与重建协调。进程级 `FRenderResource` 先推进 packed generation token，再通过
  弱监听表让插件 GPU 资源释放自身 backing；Target 通过非阻塞 RenderThread barrier 确认恢复，Pipeline
  统一协调 Frontend 与 Target 重建，旧 token 的在途 Slate Frame 在进入 RDG 前整帧跳过。详见
  [RHI 资源生命周期开发设计](RHI-Resource-Lifecycle.md)。

验收标准：

- 嵌套 layer 的绘制顺序、区域和透明背景正确。
- 非法或不平衡的 layer 调用能被检测，不产生悬挂 GPU 资源。

### 阶段 2：Layer 合成（已完成）

- [x] 按命令原始顺序实现 `CompositeLayers(Source, Destination, Blend, Filters)` 调度。
- [x] 实现普通 premultiplied-alpha `Blend`。
- [x] 实现严格覆盖的 `Replace`，不退化为普通 alpha blend。
- [x] source 和 destination 为同一 layer 时先复制到 RDG scratch texture，避免 SRV/RTV 冲突。
- [x] composite 输出遵守命令捕获的 scissor。
- [x] composite 使用 source layer 中命令捕获的 clip-mask 生成透明裁剪快照，再写入 destination layer；
  该规则适配 Slate Target 的每 Layer 独立 stencil，不依赖参考后端的全 Layer 共享 stencil。
- [x] Slate Target 已开放 `bLayerOperations` 和 `bSaveLayerAsTexture`；mask 和尚未实现的 filter
  仍使用独立能力拒绝。

验收状态：

- L01 的 `opacity(100%)` Layer 与直接绘制结果均可见，已验证基础 Layer raster 和 composite 链路。
- L02 覆盖 opacity filter 的整体透明度合成。
- L03 覆盖 source-layer rounded stencil snapshot 与 destination composite 的组合。

### 阶段 3：保存 Layer 与跨帧资源生命周期

- [x] `SaveLayerAsTexture()` 返回可供普通 geometry 使用的 texture handle。
- [x] `SaveLayerAsMaskImage()` 返回 mask filter handle，而不是普通纹理 handle。
- [x] 保存结果在命令出现的位置生成独立快照；同帧通过 `FRDGTextureRef` 使用，跨帧通过
  RDG extraction 转换为 `IPooledRenderTarget`，下一帧重新注册为 external RDG texture。
- [x] 保存范围严格使用命令捕获的 scissor；超出当前 Layer 可见范围的部分保持透明。
- [x] Pipeline 管理逻辑 texture handle 生命周期。
- [x] Target 管理保存纹理的 GPU backing 生命周期。
- [x] `FDrawFrame` 仅持有 RenderThread 消费本帧所需的强引用。
- [x] `ReleaseTexture` 从 Target 活跃表移除 backing，在途 Frame 通过快照强引用安全完成；
  未知或重复释放为 no-op。
- [x] `SaveLayerAsMaskImage` 与 `ReleaseFilter` 的持久资源路径：同帧使用 RDG 快照，跨帧使用
  Target 持有的 pooled render target；释放后由在途 Frame 快照保持 GPU backing 存活。

RmlUi 的 box-shadow 可能缓存 `SaveLayerAsTexture()` 的结果，因此保存结果不能只存在于当前帧。

验收状态：

- GPU Automation Test 已覆盖 Save 后同帧采样、保存后继续修改源 Layer、跨帧采样、
  `ReleaseTexture` 与在途 Frame、1×/4× MSAA 一致性以及 Target 销毁后的引用安全。
- mask 对应的同帧快照、源 Layer 后续修改隔离、跨帧复用、局部 Bounds、1×/4× MSAA、
  `ReleaseFilter` 在途释放和 Target cache 销毁均已通过 GPU Automation Test。
- 2026-09-01 使用 `MarkupUI` 过滤器执行完整插件 C++ 测试集：发现 14 项，14 项成功，0 项失败。
- 全功能测试页 L07 `SaveLayerAsMaskImage` alpha silhouette 与 L08 persistent mask reuse
  已确认 UE 与 RmlUi Document Viewer 视觉一致。
- 视觉测试页 L04 已确认 sharp outer shadow 的保存纹理、偏移、颜色和圆角轮廓与 RmlUi Viewer 一致。
- 视觉测试页 L05 已确认正负偏移的 zero-blur inset shadow 与内部裁剪一致。
- 视觉测试页 L06 已确认多重 sharp shadow 的顺序和跨帧复用稳定，无闪烁、白色 fallback 或纹理丢失。

### 阶段 4：基础颜色 Filter（已完成）

优先实现可用单 pass 颜色矩阵完成的 filter：

- [x] opacity
- [x] brightness
- [x] contrast
- [x] grayscale
- [x] invert
- [x] sepia
- [x] hue-rotate
- [x] saturate

Pipeline 将上述效果编译为通用颜色矩阵描述，Target 使用统一 pixel shader 执行，以减少 shader permutation。
Slate Target 按 RmlUi 官方 DX12 后端的语义逐个执行声明顺序中的 filter：两个帧内 transient
texture 交替作为输入和输出，每个颜色矩阵都是独立 pass。中间结果遵循 encoded RGBA8 UNORM
的 clamp 与量化行为，不合并相邻矩阵，避免改变官方多 pass 的像素结果。

验收状态：

- CPU Automation Test 覆盖七种颜色矩阵的编译结果和非法参数拒绝。
- GPU Automation Test 覆盖七种颜色效果、声明顺序、中间 pass clamp、透明像素、opacity 与
  color matrix 混合链，以及 1×/4× MSAA 一致性。
- 2026-09-01 使用 `MarkupUI` 过滤器执行完整插件 C++ 测试集：发现 33 项，33 项成功，0 项失败。
- 全功能测试页 F01–F08 已使用相同字体完成 UE 与 RmlUi Document Viewer 对比，颜色、透明度、
  filter 顺序和位移结果一致；此前观察到的文字清晰度差异来自测试字体不同，不是 filter 像素差异。

### 阶段 5：Blur 与 Drop Shadow（已完成）

- [x] separable Gaussian blur。
- [x] 水平和垂直两个 pass。
- [x] sigma 到采样半径和 kernel 的稳定规则。
- [x] filter ink overflow 对 layer/scissor bounds 的扩展。
- [x] 临时 RenderTarget 复用。
- [x] drop-shadow 的 offset、颜色、blur 和原图组合。

Blur 是 `filter: blur`、`backdrop-filter: blur`、drop-shadow 和模糊 box-shadow 的共同依赖。

Slate Target 保留 RmlUi DX12 参考后端的有序多 Pass 语义：大 sigma 先按稳定规则逐级降采样，
再使用 7-tap separable Gaussian kernel 执行垂直和水平 blur，最后恢复到原始分辨率。Drop Shadow
先从源 alpha 生成带 offset 的预乘颜色图像，按需执行相同 blur，再以 source-over 规则将原图合成到阴影上。
各步骤不融合，继续保留 encoded RGBA8 UNORM 的中间 clamp 与量化行为。

Executor 使用三个按帧惰性创建的 RDG transient scratch texture，并保证输出纹理不与当前输入或第二输入
发生 SRV/RTV alias。Composite scissor 现在参与页面离屏可见区域计算；无 scissor 的 composite 会明确
使用完整 ViewRect，避免阴影等 filter ink overflow 被普通 geometry bounds 提前裁掉。

验收状态：

- CPU Automation Test 已覆盖 blur sigma 和 drop-shadow offset、颜色预乘参数的编译结果。
- GPU Automation Test 已覆盖小 sigma blur、大 sigma 降采样 blur、带 blur 的彩色 drop-shadow，
  blur/drop-shadow 的 1×/4× MSAA 一致性，以及 Slate `PF_A2B10G10R10` 输出目标下的 Alpha 精度回归。
- 2026-09-02 使用插件名 `MarkupUI` 执行完整插件 C++ 测试集：发现 41 项，41 项成功，0 项失败。
- 全功能测试页 F09–F11 已完成 UE 与 RmlUi Document Viewer 的最终视觉对比；RGBA8 UNORM
  离屏中间格式消除了低位 Alpha 导致的分层轮廓、八边形大 blur 和 drop-shadow 矩形边界，
  小 blur、大 blur、彩色 drop-shadow 的颜色、轮廓和衰减现已一致。

### 阶段 6：Filter 与 Backdrop Filter 调度（已完成）

- [x] opacity `filter`：元素及其子内容先进入离屏 layer，再应用 opacity 并合成到父 layer。
- [x] 多个 opacity filter 在 composite 阶段合并为等价的累计 opacity。
- [x] 将同一调度扩展到颜色矩阵和 mask filter。
- [x] 将同一调度扩展到 blur 和 drop-shadow。
- [x] `backdrop-filter`：读取元素后方已有内容，过滤并写回，再继续绘制元素。
- [x] 支持当前已实现的多个不同类型 filter 按声明顺序串联。
- [x] 处理 filter 输入区域和输出裁剪区域不同的情况。

RmlUi 已经通过 layer 和 composite 调用顺序描述调度，Target 应忠实执行命令流，不应重新解释 RCSS。

验收状态：

- 全功能测试页 B01–B03 已完成 UE 与 RmlUi Document Viewer 对比，背景过滤、前景隔离、圆角 blur
  和多 Filter 顺序的视觉结果一致。
- GPU Automation Test 已覆盖前景隔离、不同输入/输出区域和 Backdrop 多 Filter 顺序。
- 2026-09-02 使用无参数默认模式执行完整插件 C++ 测试集：发现 63 项，63 项成功，0 项失败。

### 阶段 7：Mask Image（已完成）

- [x] 将 mask decorator 绘制到临时 layer。
- [x] `SaveLayerAsMaskImage()` 生成 mask filter。
- [x] 使用 mask filter 的 alpha 通道合成元素内容 layer。
- [x] 多个 mask decorator 由 RmlUi 绘制到同一个 mask layer 后生成单一快照。
- [x] mask 明确只使用保存图像的 alpha 通道；RGB 不参与遮罩权重。
- [x] mask 与 scissor、圆角 clip、transform 同时使用时保持正确（M03–M09，UE 与 RmlUi Viewer 对比通过）。

### 阶段 8：Box Shadow（已完成）

- [x] 验证无 blur 的 inset/outset shadow（L04、L05，UE 与 RmlUi Viewer 对比通过）。
- [x] 验证带 blur 的 inset/outset shadow（S01–S03，UE 与 RmlUi Viewer 对比通过）。
- [x] 验证多重 sharp shadow 顺序（L06）。
- [x] 验证 shadow texture 的跨帧缓存与释放（L06 视觉验证与 GPU Automation Test）。

不应在 UE 侧另写一套 box-shadow 语义。Layer、blur、clip、save texture 和 composite 完整后，应让 RmlUi 已生成的 shadow geometry 自然工作。

### 阶段 9：内置 Gradient Shader（已完成）

- [x] linear-gradient
- [x] repeating-linear-gradient
- [x] radial-gradient
- [x] repeating-radial-gradient
- [x] conic-gradient
- [x] repeating-conic-gradient
- [x] color stop 数量上限及超限行为：与参考后端一致，最多使用前 16 个 stop。
- [x] premultiplied color interpolation
- [x] Pipeline 生成 Target 无关的标准 gradient 描述；默认 Slate Target 已按该描述执行。
- [x] 六种 Gradient GPU Automation Test 与 Pipeline 参数标准化测试。
- [x] G01–G06 完成 UE 与 RmlUi Viewer 的人工视觉对比，结果一致。

颜色 stop 可通过 structured buffer 或受控上限的 shader 参数传递。

### 阶段 10：资源路径完整性（已完成）

- [x] Unreal Asset Provider 支持项目与插件内容挂载点作为 `AssetResourceRootDirectory`。
- [x] 磁盘与资产相对路径均不能逃出各自配置的资源根目录；绝对磁盘路径、UNC 与 URI Scheme 会被拒绝。
- [x] RML `<link>` 以及 RCSS 内字体、图片等嵌套引用统一相对于直接引用者解析。
- [x] Unreal Asset 与磁盘资源分别保持独立资源域；`url()` 不会跨域回退查找。
- [x] 资源根目录在文档 Context 创建时固定；设置变化仅影响新建或 Full Reload 后的 Context，正在显示的 Context 不会被静默切换。
- [x] 自动化测试覆盖根路径、相对路径、嵌套引用、目录逃逸、外部路径拒绝、插件 Mount Point 与资源域隔离。

RmlUi 6.3 不提供 CSS `@import`；外部样式表通过 RML `<link type="text/rcss">` 引用。

### 阶段 11：视觉回归与生命周期测试

至少建立以下截图或像素对比用例：

- [x] premultiplied alpha 半透明边缘（A01–A03）
- [x] 连续圆角 clip（C01–C02）
- [x] inverse clip（C03、S02）
- [x] intersect clip（C02）
- [x] 二维 transform（T04）
- [x] 透视 transform（T01–T03）
- [x] opacity filter（L01–L03）
- [x] 其余每一种颜色 filter（F01–F08，使用相同字体后 UE 与 Viewer 视觉一致）
- [x] blur（F09–F10）
- [x] drop-shadow（F11）
- [x] backdrop-filter（B01–B03，UE 与 Viewer 视觉一致；GPU Automation Test）
- [x] 基础 mask-image（L07、L08）
- [x] inset/outset box-shadow（S01–S03）
- [x] 六种 gradient（G01–G06，UE 与 RmlUi Viewer 视觉一致；GPU Automation Test）
- [x] layer Blend（L01–L03）
- [x] layer Replace（GPU Automation Test）
- [x] source layer 与 destination layer 相同（1× scratch-copy 与 4× MSAA）
- [x] Save 后同帧作为纹理绘制，且保存后修改源 Layer 不污染快照（1× 与 4× MSAA）
- [x] 保存纹理在下一帧继续使用
- [x] 多 Widget、多 Context；两个真实 `FRmlLanguageFrontend` 分别创建 Context、加载 Document 并通过独立
  Pipeline/Target 生成 Frame，关闭第一个 Context 不影响第二个继续更新和绘制。
- [x] Target 资源表销毁时 RenderThread Frame 尚未消费资源快照
- [x] 文档关闭时已提交的 Slate 绘制仍在 RenderThread 等待消费
- [x] resize、DPI 改变和窗口销毁；自动化测试覆盖初始 viewport、无变化去重、尺寸与 pixel ratio 更新、
  零尺寸拒绝、Target 弱引用失效以及在途 Frame 的资源寿命。
- [x] RHI 资源失效与重建协调；自动化测试覆盖 Unavailable、Recovering、Ready、新代次单次失效、
  恢复确认帧跳过和下一帧恢复。真实 GPU Device Removed 的进程内恢复仍由 Unreal 平台能力决定。
- [x] 在途 Frame 尚未消费时 `ReleaseTexture`
- [x] mask-image `ReleaseFilter` 与 Gradient `ReleaseShader`；已提交 Frame 继续持有执行所需快照。

### 阶段 12：渲染性能分析与连续几何批处理

该阶段只在像素语义和视觉回归稳定后进行，不能为减少 Draw Call 改变 RmlUi 原始绘制顺序、
Premultiplied Alpha、clip、layer 或 filter 结果。

- [x] 分别统计 command 数、合批前后 raster draw、geometry 实例/复用、顶点与索引上传量、
  texture/shader/stencil state switch、layer 数和逻辑 composite 数，避免仅用 command 总数判断 GPU 压力。
- [x] 从真实 RDG 调度点统计 raster、composite、filter、mask、copy、clear pass 与 resolve 操作，
  并按 Layer、Stencil、Filter 临时纹理、Filter 覆盖区域、Snapshot 和 Scratch 分类统计离屏像素与采样量。
  以上计数由 `r.MarkupUI.RenderStatistics` 输出；模式 `1` 仅在数据变化时报告，模式 `2` 每帧报告。
- [x] 完成 Layer 反向需求区域传播、Filter halo 扩展、不可见普通几何早期剔除、局部 Filter scratch、
  延迟 MSAA Resolve、局部 same-layer scratch 和同帧 mask snapshot 复用。
- [x] 增加 active layer、pass pixel、resolve/clear pixel、剔除上传量、稳定 target/report id 与批处理拒绝原因统计；
  日志按 commands、passes、resources 三行输出。
- [ ] 使用 Unreal Insights、RDG Insights 或 RenderDoc 对典型页面采集 CPU 提交时间、GPU 时间和实际显存峰值。
  这些时间与驱动级资源峰值属于外部性能采样，不由同步渲染日志估算。
- [x] 为普通连续 geometry 建立严格的 batch key，至少包含 texture、shader、blend、stencil、
  scissor、transform、目标 layer 和纹理 Alpha 语义。
- [x] 只合并命令流中相邻且 batch key 完全兼容的 geometry，不跨越 clip mask、layer、composite、
  filter 或其他顺序屏障重新排序。
- [x] 合并兼容 geometry 的顶点和索引范围，减少 `DrawIndexedPrimitive` 调用与 PSO/资源切换。
  该路径由全局 `FUIRenderingSettings::bEnableGeometryBatching` 控制，当前默认关闭，便于与参考路径
  进行像素和性能 A/B 对比。
- [x] 根据统计结果决定暂不实现 Target 侧跨帧持久 GPU Geometry Cache：当前几何上传量相对离屏采样量很小，
  生命周期、失效和显存常驻复杂度的收益优先级低于 Layer/Filter 带宽优化。
- [x] 为批处理前后建立像素一致性测试，并记录合批命中率与拒绝原因；真实页面的 GPU 时间仍由外部采样验证。

#### 遗留优化：连续颜色 Filter 融合

该优化只在全部 RCSS 视觉功能完成并稳定后评估，不能作为颜色 Filter 的默认实现提前启用。

RmlUi 官方参考后端不会预先合并多个颜色矩阵，而是按 Filter 列表的声明顺序逐个执行
fullscreen pass，并在两张 post-process texture 之间 ping-pong。每一步都会写回 UNORM
RenderTarget，因此会发生 clamp、截断和量化。当前 Slate Target 保留了这一官方多 Pass 语义，
它是 UE 与 RmlUi Document Viewer 像素结果一致的默认契约。

可选优化方案是将连续的 `opacity + color-matrix` 预合并为单个 pass。它可以减少中间纹理带宽、
RenderPass 数量和 fullscreen draw call，尤其有利于大量元素串联多个颜色 Filter 的页面。但是，
当 brightness、contrast 等中间步骤产生超出 UNORM 范围的 RGB 时，融合路径不会执行官方每一步
的中间 clamp 与量化，后续矩阵可能重新使用这些越界值，因此最终像素可能与 Viewer 不同。

这不是纯内部、结果等价的性能优化。启用前必须明确产品契约，并完成以下事项：

- [ ] 统计真实页面中连续颜色 Filter 的数量、覆盖像素、pass 带宽和 GPU 时间，确认优化收益值得增加双路径复杂度。
- [x] 保持官方多 Pass 路径为默认且长期可用的参考路径。
- [x] 将连续 color-matrix 融合作为 `bEnableColorMatrixFilterFusion` 显式可选策略，默认关闭。
- [x] 为无越界矩阵链、中间 clamp 差异链和透明像素建立融合路径 GPU 像素测试；官方路径原有测试继续覆盖
  opacity 混合链和 1×/4× MSAA。
- [x] 文档明确融合路径只保证近似视觉一致，不承诺与 RmlUi Viewer 逐像素一致。
- [x] 生产配置只有显式开启设置才使用融合路径。

验收标准：

- 开启批处理后，现有视觉回归结果不发生像素差异。
- L01–L03 等 Layer/Filter 用例的命令顺序和 pass 边界不被合并破坏。
- 性能文档同时报告 CPU 提交时间、Draw Call、离屏像素和 GPU 时间，不以单一指标宣称优化完成。

### 阶段 13：自定义 Shader Decorator

自定义 `decorator: shader(...)` 是完成 RmlUi 内置 RCSS 语义后的最后一项扩展功能，不阻塞内置 RCSS
完整支持的完成判定。其字符串内容由渲染后端解释，不等价于自动执行任意 Web shader。

- [ ] 定义 Engine 级 shader provider/registry 公共扩展契约。
- [ ] 内置或示例实现 RmlUi 参考后端的 `creation` shader。
- [ ] 未注册 shader 明确失败，不退化为普通 geometry。
- [ ] 明确自定义 shader 可使用的纹理、尺寸、时间和参数类型。
- [ ] 明确 Shader Platform、Cook、模块注册和注销规则。
- [ ] 建立参数校验、GPU 像素、在途 Frame 与模块生命周期测试。

## 推荐实施顺序

```text
0.1 基础正确性
  ↓
0.2 通用命令协议
  ↓
0.3 Filter/Shader 描述 + 0.4 typed C API 参数
  ↓
0.5 能力失败规则
  ↓
0.6 Unreal Imported Asset 标准化
  ↓
0.7 编译几何复用
  ↓
1 Layer Stack
  ↓
2 Composite
  ↓
3 Save Layer 与跨帧资源
  ↓
4 颜色 Filter
  ↓
5 Blur / Drop Shadow
  ↓
6 Filter / Backdrop Filter
  ↓
7 Mask Image
  ↓
8 Box Shadow
  ↓
9 Gradient
  ↓
10 资源路径完整性
  ↓
11 生命周期与自动化视觉回归收尾
  ↓
12 性能分析与连续几何批处理
  ↓
13 自定义 Shader Decorator
```

## 完成定义

只有同时满足以下条件，才能称为“完整支持 RmlUi 6.3 内置 RCSS 视觉语义”：

- RmlUi 6.3 内置 filter 和 gradient 均有真实像素实现。
- layer、mask、backdrop、shadow 和 replace blend 语义完整。
- 未支持的自定义 shader 能明确失败，而不是静默退化为普通 geometry。
- 通用 `FDrawFrame` 不依赖 RmlUi 或 Slate 类型，可由其他 Target 重放。
- GPU 资源生命周期在跨帧、异步 RenderThread、Widget 销毁和 RHI 重建情况下安全。
- 每项高级效果都有稳定的视觉回归测试。
