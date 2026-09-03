# MarkupUI 渲染性能诊断与优化路线

## 文档目标

本文是 MarkupUI 的内部开发文档，用于解释 `r.MarkupUI.RenderStatistics` 输出的每个字段、建立一致的日志分析方法，并记录按预期收益排序的渲染优化任务。

统计日志描述的是一次 Unreal RDG 绘制执行的逻辑工作量和资源规模。它适合发现异常趋势、比较改动前后差异，但不能单独替代 Unreal Insights、GPU Visualizer、RDG Insights 或 RenderDoc 的真实时间和带宽测量。

## 开启统计

在 Unreal Editor 的控制台中输入：

```text
r.MarkupUI.RenderStatistics 1
```

支持以下模式：

| 值 | 行为 | 推荐用途 |
| ---: | --- | --- |
| `0` | 禁用统计日志 | 日常运行 |
| `1` | 仅当完整统计值变化时输出 | 滚动、交互和状态切换分析 |
| `2` | 每个执行帧均输出 | 连续采样和外部脚本分析 |

模式 `1` 使用进程内上一条统计结果进行比较。多个 MarkupUI Widget 同时绘制时，不同 Widget 的结果可能交替出现，应使用 `target` 区分绘制目标。

性能数据行统一以 `[MarkupUI Statistics]` 开头。当前每份报告由 `commands`、`bounds`、
`antialiasing`、`passes` 和 `resources` 五行组成，五行共享相同的 `report`、`target`、`vw` 和 `vh`。
复制或解析日志时必须保留并合并同一 `report` 的全部行，不能把其中一行当作完整报告。旧基线可能只有
`commands`、`bounds`、`passes` 和 `resources` 四行；分析脚本继续接受这种旧格式，但不会为它推导抗锯齿诊断。

## 统计边界

阅读日志前必须先明确以下规则：

1. 每条日志对应一次 RDG Executor 执行，不是累计到当前时刻的全局总量。
2. `Pixels` 字段表示二维纹理面积或 Pass 覆盖面积；`Samples` 字段额外乘入 MSAA sample count。
3. 统计值是本次执行登记的累计工作量，不是 RDG transient resource 的峰值常驻显存。RDG 可以让生命周期不重叠的资源复用同一块物理内存。
4. 像素数和采样数不是字节数。实际字节数还取决于纹理格式、压缩、tile、硬件缓存和读写方式。
5. Pass 数量不能直接换算为 GPU 时间。一个小范围 Fullscreen Pass 可能比一个覆盖 4K 的 Pass 便宜很多。
6. 状态切换数表示相邻 Prepared Draw 的逻辑状态差异，不保证与驱动层最终产生的 PSO 或 Descriptor 绑定次数完全相同。
7. `vw`、`vh` 是当前 Draw Target（Slate 场景下即 Widget）的实际像素视口，不是显示器、窗口或 Slate BackBuffer 尺寸。
8. 比较性能时必须先确认 `vw`、`vh` 相同，并固定 DPI、MSAA、页面内容、滚动位置和动画状态。

## 字段参考

### 采样身份与视口

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `report` | 同一次统计报告的编号 | 使用它合并同一报告的全部分类行 |
| `target` | 当前绘制目标在本次进程中的诊断标识 | 多个 Widget 同时输出日志时用于分组；不要把它当作跨进程稳定 ID |
| `vw` | 当前 Draw Target 的实际像素视口宽度 | Slate Widget 中对应 Widget 视口，而不是窗口或 BackBuffer 宽度 |
| `vh` | 当前 Draw Target 的实际像素视口高度 | 视口不同的原始像素、采样和带宽压力数据不能直接横向比较 |
| `msaaSamples` | 本次执行使用的 MarkupUI MSAA sample count | A/B 对比时必须保持一致 |
| `adaptiveMSAA` | Raster Layer 内容自适应 MSAA 开关，`1` 表示允许经证明不需要边缘 AA 的 Layer 使用 1× | 位于 `category=antialiasing`；旧日志没有该字段 |
| `geometryBatching` | 相邻几何合批开关，`1` 为开启，`0` 为关闭 | 不再依赖文件名或拒绝计数反推测试配置 |
| `colorMatrixFusion` | 连续颜色矩阵融合开关，`1` 为开启，`0` 为关闭 | 开启后不再保证 Viewer 的逐 Pass UNORM clamp 语义 |

比较两份日志时，应先按 `target` 分组，再只比较 `vw × vh` 相同的报告。若必须比较不同分辨率，至少同时观察按视口面积归一化后的指标，例如：

```text
normalizedOffscreenSamples = offscreenSamples / (vw × vh)
```

### Base Layer Bounds

`category=bounds, layer=base` 专门记录 Base Layer 从初始可见区域到最终资源范围的变化。矩形统一使用
`MinX:MinY:MaxX:MaxY`，坐标位于当前 Draw Target 的像素空间。

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `baseOffscreen` | Base Layer 是否使用独立离屏目标 | `0` 表示直接写入 Draw Target，此时 Bounds 仍可用于诊断，但不存在 Base Color 离屏分配 |
| `regionStatus` | 初始可见区域判定 | `bounded` 表示可安全限定；`full-view-covered` 表示有界命令已经覆盖完整视口；`full-view-fallback` 表示存在无法安全限定的无 Scissor 命令 |
| `declaredBounds` | 在 Composite 反向传播前，由可见命令声明出的 Base 初始需求范围 | 若等于完整 `vw × vh` 且 `regionStatus=full-view-fallback`，优先寻找无 Scissor 命令 |
| `requiredBounds` | Composite 和 Filter 依赖反向传播后，Base 必须保留的范围 | 若明显大于 `declaredBounds`，优先检查 Composite、Backdrop 或 Filter 传播 |
| `allocatedBounds` | Base Layer 最终实际使用的 Render Target 范围 | 应尽量接近 `requiredBounds`；明显更大表示资源分配阶段仍有额外扩大 |

这组字段的主要用途不是计算 GPU 时间，而是寻找某条无 Scissor 命令或 Composite 依赖传播导致 Base Layer
扩张到完整 Widget 的情况。诊断顺序如下：

1. 使用 `vw × vh` 得到完整 Widget 范围。
2. 检查 `declaredBounds` 是否已经等于完整 Widget。
3. 若 `regionStatus=full-view-fallback`，定位无 Scissor 的 Draw 或 Composite 命令。
4. 若 `declaredBounds` 较小但 `requiredBounds` 扩大，定位 Composite、Backdrop 和 Filter 的反向需求传播。
5. 若只有 `allocatedBounds` 扩大，检查资源创建阶段；只有日志证明这里存在异常扩大后，才继续修改 Bounds 逻辑。

### 命令与 Draw Call

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `commands` | 本帧通用 Draw Command List 中的全部命令数，包含 raster、layer、composite、save 等命令 | 判断前端生成的总命令规模；它不等于 Draw Call |
| `rasterCommands` | 进入可见性裁剪和相邻合批前的 raster 源命令数量 | 与 `culledRasterCommands`、`rasterDrawCalls` 一起观察 |
| `culledRasterCommands` | 在上传顶点和索引前被证明不可见的普通绘制数量 | 滚动后应随不可见章节增加；Stencil 状态序列采用保守保留规则 |
| `preparedRasterCommands` | 可见性裁剪后、相邻合批前保留的普通 raster 命令数量 | 正常情况下等于 `rasterCommands - culledRasterCommands` |
| `rasterDrawCalls` | 实际加入 RDG Raster Pass 的普通 Draw 与 clip-mask replay Draw 总数 | 用于观察真实调度的几何 Draw Call；不包含 Composite fullscreen triangle |
| `clipReplayDrawCalls` | `rasterDrawCalls` 中为 Composite 重建目标 Stencil 而追加的 Draw Call | 可从总数中分离裁剪链 replay 成本 |
| `mergedDraws` | 合批器实际成功吸收到前一个 Draw 的普通命令数量 | 由合批器直接累计，不包含早期裁剪 |
| `mergeRate` | `mergedDraws / preparedRasterCommands` | 表示合批对裁剪后普通命令的实际 Draw Call 降幅 |
| `culledUploadBytes` | 被早期剔除的几何原本需要上传的逻辑字节数 | 用于估算 CPU 准备和上传节省趋势，不等于总线实测带宽 |
| `batchRejectClip` | 因裁剪写入模式不兼容而未合批的相邻 draw 数量 | 判断裁剪写入是否为主要边界 |
| `batchRejectTextureShader` | 因纹理、纹理 Alpha 模式或 Shader 不兼容而未合批的相邻 draw 数量 | 判断 atlas 或材质状态是否限制合批 |
| `batchRejectTransform` | 因 Transform 不兼容而未合批的相邻 draw 数量 | 判断几何变换是否为主要边界 |
| `batchRejectScissor` | 因 Scissor 状态或矩形不兼容而未合批的相邻 draw 数量 | 判断裁剪矩形变化密度 |
| `batchRejectStencil` | 因 Stencil 读取状态或 reference 不兼容而未合批的相邻 draw 数量 | 复杂嵌套裁剪中通常较高 |
| `batchRejectOther` | 因 Layer、Blend 或其他兼容条件未通过而未合批的相邻 draw 数量 | 用于覆盖上述类别之外的合批边界 |

以下关系可用于检查统计是否自洽：

```text
preparedRasterCommands = rasterCommands - culledRasterCommands
normalRasterDrawCalls = rasterDrawCalls - clipReplayDrawCalls
mergeRate = mergedDraws / max(preparedRasterCommands, 1)
```

`rasterDrawCalls` 还包含 `clipReplayDrawCalls`，因此不能再用 `rasterCommands - rasterDrawCalls` 推导合批数量。
合批成功数和实际执行的 replay Draw 均由执行路径直接记录。

当 `preparedRasterCommands` 和 `normalRasterDrawCalls` 在滚动过程中几乎不变时，说明不可见页面内容仍然进入
Target 的几何准备和绘制流程。此时应优先检查可见性裁剪，而不是先扩大合批规则。

### 几何复用与上传

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `geometryInstances` | 裁剪前 raster 源命令引用编译几何的次数 | 表示进入本帧准备阶段的几何实例工作量；当前通常与 `rasterCommands` 相同 |
| `uniqueGeometries` | 本帧引用的不同编译几何数量 | 与实例数比较，判断共享顶点数据的空间 |
| `geometryReuseRate` | `(geometryInstances - uniqueGeometries) / geometryInstances` | 越高表示同一编译几何被重复使用得越多 |
| `vertices` | 本帧上传到 RDG vertex buffer 的顶点数量 | 当前相同编译几何的顶点只追加一次，因此受几何复用影响 |
| `indices` | 本帧上传的索引数量 | 每个几何实例仍会追加对应索引，可能高于纯唯一几何所需数量 |
| `vertexUploadBytes` | `vertices × sizeof(FUnrealRDGDrawVertex)` | CPU 到 GPU 的逻辑顶点上传量 |
| `indexUploadBytes` | `indices × sizeof(uint32)` | CPU 到 GPU 的逻辑索引上传量 |

`geometryReuseRate` 高并不等于 Draw Call 已合并。它只说明顶点数据被复用；是否合批仍由纹理、Shader、Layer、Transform、Scissor、Stencil 和绘制顺序共同决定。

### 绘制状态变化

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `textureSwitches` | 同一 Raster Segment 内相邻 draw 的纹理或 alpha mode 发生变化的次数 | 判断纹理排序和 atlas 是否可能降低状态切换 |
| `shaderSwitches` | 相邻 draw 的 shader handle 发生变化的次数 | 判断渐变、普通纹理和其他 Shader 路径的切换密度 |
| `stencilSwitches` | clip write、clip operation、stencil reference 或 clip read 状态发生变化的次数 | 高值通常来自复杂嵌套裁剪；不能为了降低数字而改变绘制顺序或裁剪语义 |

这些字段只累计实际加入 RDG 的 Raster Segment，并在每个 Segment 开始时重新建立相邻关系，因此不统计
Segment 之间的边界切换。合批拒绝原因同样只来自实际执行的 Segment；关闭合批时所有拒绝计数均为零。

### Layer 与逻辑合成

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `layers` | 本次执行计划中建立的 RDG Layer 条目数，包含 Base Layer，也可能包含空 Layer | 不能单独用来判断实际分配数量；应结合 `activeLayers` 与 `layerPixels` |
| `activeLayers` | 本帧实际拥有有效 Color 资源的离屏 Layer 数 | 与 `layers` 比较可观察反向需求传播移除了多少空 Layer |
| `logicalComposites` | Draw Layer Plan 中的 Composite 命令总数 | 表示前端要求的逻辑合成数量，包括最终可能因不可见而未执行的命令 |
| `compositePasses` | 实际加入 RDG 的 Composite Raster Pass 数量 | 与 `logicalComposites` 比较可观察可见性裁剪；Base Layer 最终回写也会贡献一次 |

`logicalComposites` 是命令计划指标，`compositePasses` 是实际调度指标，两者不应被视为重复字段。

### RDG Pass

| 字段 | 含义 | 常见来源 |
| --- | --- | --- |
| `rasterPasses` | 实际加入 RDG 的几何 Raster Pass 数量 | Raster Segment 写入 Layer |
| `compositePasses` | Layer 合成 Pass 数量 | Blend、Replace、Backdrop 合成以及最终 Base 回写 |
| `filterPasses` | Filter 执行产生的 Pass 数量 | Color Matrix、Opacity、Mask Image、Blur、Drop Shadow、上下采样 |
| `fusedColorMatrixFilters` | 开启颜色矩阵融合后，被吸收到前一个矩阵中的后续 Filter 数量 | 默认多 Pass 路径通常为 `0`；用于确认可选融合是否实际命中 |
| `maskPasses` | 使用 Stencil 生成遮罩快照的 Pass 数量 | Source Layer stencil snapshot、保存 mask |
| `copyPasses` | 显式纹理复制 Pass 数量 | SaveLayerAsTexture、同 Layer composite scratch copy |
| `clearPasses` | 显式清理纹理或 Layer 的 Pass 数量 | Filter scratch 初始化、Snapshot 初始化、空源 Layer 初始化 |
| `stencilDrawCalls` | 实际启用 Stencil 写入或测试的 Draw Call 数量 | Clip Mask 写入、受 Clip Mask 约束的几何、Mask Replay 和受裁剪的 Composite |
| `stencilClearPasses` | 以 `EClear` Load Action 首次初始化 Stencil Attachment 的 Raster Pass 数量 | 该字段表示 Attachment Clear，不表示额外创建了独立 RDG Clear Pass |
| `resolves` | MSAA RenderTarget Resolve 操作数量 | Raster、Composite、Mask Snapshot 等带 resolve attachment 的写入 |
| `rasterPassPixels` | Raster Pass 目标 Layer 范围的累计面积 | 配合 Raster Pass 数判断 Pass 是否仍覆盖过大的 Layer |
| `compositePassPixels` | Composite 与最终 Base 回写范围的累计面积 | 判断合成次数减少是否同时缩小覆盖面积 |
| `resolvePixels` | 实际 Resolve 操作覆盖的累计面积 | 延迟 Resolve 的核心带宽趋势指标 |
| `clearPixels` | 显式 Clear 覆盖的累计面积 | 区分少量局部 Clear 与大范围清屏 |

分析顺序建议为：先看每类 Pass 数，再看对应像素范围。仅减少 Pass 数但扩大覆盖范围，可能使总成本反而上升。

### 抗锯齿需求

`category=antialiasing` 独立记录每个有效 Layer 是否必须使用多采样，以及实际 Prepared Draw 触发该结论的原因。
这些字段描述的是保守的边缘覆盖判定，不是对最终图像质量的评分。

全局 `msaaSamples` 始终是上限。若插件设置选择 1×，required 和 optional Layer 都强制使用 1×；自适应逻辑
不会把它们提升到 2×/4×。此时 AA 分类仍然输出，用来标记当前 1× 策略下哪些内容理论上需要边缘 AA。

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `adaptiveMSAA` | 是否启用 Raster Layer 内容自适应 MSAA | `0` 时所有 Layer 仍按全局 `msaaSamples` 分配，但 required/optional 统计仍用于估算潜在命中范围 |
| `aaRequiredLayers` | 至少包含一个需要采样边缘覆盖的 Draw，或需要 Clip Mask replay 的有效 Layer 数量 | 开启自适应后这些 Layer 保持全局 Sample Count |
| `aaOptionalLayers` | 所有直接 Raster Draw 均证明不需要采样边缘覆盖的有效 Layer 数量 | 开启自适应后这些 Layer 使用 1×；Composite-only Layer 也属于此类 |
| `aaRequiredPixels` | `aaRequiredLayers` 的 Bounds 面积总和 | 与 `aaOptionalPixels` 一起判断实际资源收益，不要只比较 Layer 数量 |
| `aaOptionalPixels` | `aaOptionalLayers` 的 Bounds 面积总和 | 表示可以安全使用 1× 的像素范围，不等于已经节省的物理带宽 |
| `aaGeometryEdgeDraws` | 边界包含非水平或非垂直边，或无法证明边界安全的 Prepared Draw 数量 | 常见于圆角、多边形和其他生成几何边缘 |
| `aaTransformDraws` | 使用任意非 Identity 2D 或 3D Transform 的 Prepared Draw 数量 | Translate、Scale、Rotate、Skew、Yaw、Pitch、Roll、Perspective 均保守要求 AA |
| `aaClipMaskDraws` | 写入或读取 Clip Mask 的 Prepared Draw 数量 | Stencil Set、SetInverse、Intersect、裁剪内容和 Clip replay 都要求 AA |
| `aaPixelMisalignedDraws` | 几何边界不能证明落在一致整数像素位置的 Prepared Draw 数量 | 包含分数像素平移、边界顶点像素偏移不一致等情况 |

同一 Draw 可以同时满足多个原因，因此四个 `aa*Draws` 不能相加后与 `rasterDrawCalls` 比较。合批后的一个
Prepared Draw 会保留被合并 Draw 的全部原因，所以这些计数描述实际提交批次的原因分布，而不是前端源命令数量。

内容自适应命中率建议按面积优先计算：

```text
aaOptionalLayerShare = aaOptionalLayers / (aaRequiredLayers + aaOptionalLayers)
aaOptionalPixelShare = aaOptionalPixels / (aaRequiredPixels + aaOptionalPixels)
```

`aaOptionalPixelShare` 比 Layer 数更能说明潜在收益。一个完整视口的 required Layer 可能远大于多个小型
optional Layer。比较开关前后时还必须保持 `vw`、`vh`、`msaaSamples`、页面状态和滚动路线一致，并同时观察
`layerSamples`、`stencilSamples`、`resolvePixels` 与视觉回归结果。

### Layer、Stencil 与 MSAA

| 字段 | 含义 | 是否包含 Sample Count |
| --- | --- | --- |
| `layerPixels` | 所有有效离屏 Layer Bounds 的二维像素面积总和 | 否 |
| `layerSamples` | Layer Color 的 sample slot 总量；MSAA Color 按 sample count 计算，存在 Resolve 纹理时再加入一份单采样面积 | 是 |
| `stencilLayers` | 本帧实际分配了 Stencil 纹理的 Layer 数量 | 不适用 |
| `stencilRequiredLayers` | 根据实际 Clip Mask 写入、测试、Replay 和 Composite 裁剪需求判断必须使用 Stencil 的 Layer 数量 | 不适用 |
| `stencilPixels` | 已分配 Stencil 纹理的二维像素面积总和 | 否 |
| `stencilRequiredPixels` | 真正需要 Stencil 的 Layer Bounds 二维像素面积总和 | 否 |
| `stencilSamples` | Layer Stencil 的 sample slot 总量 | 是 |

例如 2× MSAA 且每个 Layer 都有 Resolve Texture 时，通常会观察到：

```text
layerSamples   ≈ layerPixels × 3
stencilSamples ≈ layerPixels × 2
```

这里的 `3` 是 `2× MSAA Color + 1× Resolve`，不是三倍过度绘制。它仍然表示真实的资源规模和潜在带宽压力。

对于按需分配实施前的旧日志，使用下面的差值观察当时理论上可以避免的资源范围：

```text
avoidableStencilLayers = stencilLayers - stencilRequiredLayers
avoidableStencilPixels = stencilPixels - stencilRequiredPixels
potentialStencilPixelSavingRate = 1 - stencilRequiredPixels / stencilPixels
```

`stencilRequiredLayers` 只表示数量，不能单独用于估算资源收益。一个全屏 Layer 可能比多个小 Layer 占用更多
Stencil 面积，因此评估时必须同时比较 `stencilPixels` 与 `stencilRequiredPixels`。按需分配完成后，正常情况下
`stencilLayers` 应接近 `stencilRequiredLayers`，`stencilPixels` 应接近 `stencilRequiredPixels`；
`stencilDrawCalls` 则主要用于确认优化没有改变实际裁剪工作量。

### Filter、Snapshot 与 Scratch

| 字段 | 含义 | 诊断方式 |
| --- | --- | --- |
| `filterTexturePixels` | 本帧创建的 Filter ping-pong 临时纹理面积累计值 | 与 `filterPassPixels` 比较；前者远大于后者通常表示临时纹理范围过大 |
| `filterPassPixels` | 所有 Filter Pass 实际 Viewport 面积累计值 | 更接近 Filter Shader 实际处理的像素数量，但不含采样 tap 数量 |
| `snapshotPixels` | 保存 Layer 纹理和 Stencil 遮罩快照所创建纹理的面积累计值 | 高值表示 Save/Mask 快照范围可能过大或重复生成 |
| `scratchPixels` | Source 与 Destination 为同一 Layer 时，为避免读写冲突而创建的 Composite Scratch 面积 | 非零是正确性保护，不应直接删除；应检查范围是否最小化 |

Blur 的实际纹理采样数会受到 sigma、降采样层级和每轴采样核影响，因此 `filterPassPixels` 仍不是完整的 texture fetch 数量。

### 离屏资源总量

| 字段 | 含义 | 注意事项 |
| --- | --- | --- |
| `offscreenPixels` | 本次执行登记的所有离屏纹理二维面积累计值 | 包括 Layer Color、Stencil、Resolve、Filter、Snapshot 和 Scratch；不同类别之间会重叠统计 |
| `offscreenSamples` | 上述离屏纹理面积乘以各自 sample count 后的累计值 | 适合比较 MSAA 改动前后趋势，不是显存字节数 |

#### 不要把 `offscreenSamples` 直接视为带宽

`offscreenSamples ≈ 5.5 × viewportPixels` 表示离屏纹理的逻辑 sample slot 总量，不表示 GPU 每帧实际读写了
5.5 次完整视口。Fast Clear、Tile Cache、RDG Pass 合并、附件压缩和具体 GPU 架构都会影响真实物理流量。

在 2× MSAA 下，一个覆盖视口且确实需要 Stencil 的 AA-required 基础 Layer 通常包含以下逻辑资源：

| 资源 | 每视口像素的 sample slot 数 |
| --- | ---: |
| MSAA Color | 2 |
| MSAA Depth/Stencil | 2 |
| Resolve Color | 1 |
| 合计 | **5** |
| 额外 Layer、Filter、Snapshot | 视页面而定；当前测试样本约为 0.5 |

因此，旧实现中接近 `5×` 的基础值首先说明启用了 2× MSAA Color、匹配的 Depth/Stencil 和 Resolve，不应自动判定为
重复绘制、异常带宽或待优化问题。只有 GPU 时间、硬件带宽计数器、Pass 覆盖范围或严格 A/B 测试证明这里是
实际瓶颈时，才应针对它制定优化任务；不能仅凭 `offscreenSamples / (vw × vh)` 的倍数得出优化结论。

当前按需分配和内容自适应路径会让不需要 Stencil 的 Layer 省去 Depth/Stencil，让 AA-optional Layer 使用 1×
Color 且不创建 Resolve。因此新日志不再预期固定接近 `5×`；应结合 `category=antialiasing`、Stencil 字段和
各类 sample 统计解释变化。

## 带宽压力估算

现有统计可以估算逻辑资源规模和每帧带宽压力趋势，但不能直接给出显卡实际带宽。粗略估算某一种 RGBA8 单采样纹理的逻辑容量时可以使用：

```text
bytes ≈ pixels × 4
```

对于已知为 RGBA8 的颜色操作，可以建立以下每帧理论流量近似值：

```text
colorWriteBytes ≈ passPixels × sampleCount × 4
resolveBytes ≈ resolvePixels × (MSAASampleCount + 1) × 4
logicalBytesPerSecond ≈ logicalBytesPerFrame × frameRate
```

这里的 Resolve 公式把多采样颜色读取和单采样结果写入都计入。它只适合固定页面、视口、DPI、MSAA 和帧率下进行 A/B 对比。

不能把全部 `offscreenSamples` 统一乘以 4 当作真实显存或真实带宽，原因包括：

- Color、Stencil 和其他资源格式的每 sample 字节数不同；
- `offscreenSamples` 描述登记过的逻辑 sample slot，不包含每个 Pass 的重复读写次数；
- Blend 可能读取目标颜色，Filter 会读取输入纹理，当前统计没有完整记录这些读取字节；
- Blur 的实际纹理读取次数取决于 sigma、降采样层级和采样核，不能仅由 `filterPassPixels` 推导；
- Fast Clear、颜色压缩、Tile Cache、纹理缓存和 RDG 资源别名都会改变物理流量；
- `clearPixels`、`copyPasses` 和 Pass 数量也不能脱离资源格式与覆盖范围直接换算为带宽。

因此，现有字段适合回答“哪类操作的压力更大”和“改动前后逻辑工作量变化多少”。实际 GPU 时间、峰值显存和物理带宽仍应使用 Unreal Insights、GPU Visualizer、RDG Insights、RenderDoc 或硬件厂商分析工具确认。

## 测试页滚动样本

以下数据来自优化前的页面滚动历史样本。原始录制文件当前已不在 `PerformanceLogs` 目录中；该日志生成时
`vw`、`vh` 尚未记录 Widget 实际视口，因此只能用于保留历史观察结论，不能与新日志做归一化对比。

另两份已移出当前 `PerformanceLogs` 目录的 no-batch 与 batch 历史录制使用修正前的合批统计公式：其中
`mergedDraws` 错误包含
`culledRasterCommands`，且两次录制的主要视口高度分别为 1271 和 1304。它们只能作为历史录制保留，不能作为
正式的合批 A/B 结论。修正统计后必须在相同视口、DPI、页面状态和滚动路线下重新录制两条基线。

该样本从页面顶部滚动到底部，共 96 帧，呈现以下阶段：

| 阶段 | Raster Pass | Composite | Filter | Mask | Clear | Resolve | Filter Texture Pixels |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 页面顶部 | 43 | 1 | 0 | 0 | 0 | 43 | 0 |
| 开始进入效果区 | 46 | 4 | 3 | 3 | 3 | 52 | 0.49M |
| 中部效果区 | 52 | 17 | 35 | 16 | 22 | 84 | 48.4M |
| 峰值 | 74 | 34 | 55 | 33 | 47 | 140 | 67.8M |
| 页面底部稳定 | 74 | 25 | 46 | 24 | 29 | 122 | 66.3M |

同时观察到：

- `commands` 保持在 1323–1350；
- `rasterDrawCalls` 保持在 1199–1226，并始终接近 `rasterCommands`；
- `layers` 始终为 45；
- `layerPixels` 在 35.65M–37.53M 之间波动；
- 当时的 `mergedDraws=0`，`mergeRate=0%`；
- `stencilSwitches` 约为 561；
- `filterPasses`、`maskPasses` 和 `compositePasses` 在峰值后下降。

### 样本结论

没有观察到随帧持续累积的资源泄漏：命令、Layer 和像素规模保持有界，Filter/Mask Pass 在离开峰值区域后也会下降。

主要问题按影响排序为：

1. MSAA Resolve 与 Raster/Composite 写入绑定过紧，顶部已有 `43 rasterPasses / 43 resolves`，峰值达到 140 次 Resolve。
2. 不可见区域仍然产生约 1200 个 Raster Draw Call；滚动没有显著降低几何提交量。
3. Filter 临时纹理按照 Source Layer 的完整 extent 创建，峰值累计 67.8M 像素，并伴随 47 次 Clear。
4. 即使顶部没有执行 Filter/Mask，`layerPixels` 已约 35.65M，说明 Layer 有效范围和生命周期仍有收缩空间。
5. 当前相邻合批没有命中，但在可见性、Layer 和 Resolve 问题解决前，0% 合批不是最大带宽来源。

## 科学的性能验证流程

每个优化任务均使用以下流程，避免同时修改多个变量后无法归因：

1. 固定测试页面版本、分辨率、DPI、MSAA、字体、窗口尺寸和滚动位置。
2. 使用同一构建配置，完成预热后再采样。
3. 至少记录页面顶部、效果区入口、峰值区域和页面底部四个稳定位置。
4. 记录优化前后统计字段的中位数和峰值，不使用单帧偶然值作为结论。
5. 使用 GPU Visualizer、Unreal Insights 或 RenderDoc 核对实际 GPU 时间、Pass extent 和资源生命周期。
6. 每项优化单独提交，并保留能够关闭该优化的诊断开关，直到视觉回归稳定。
7. 对 UE 和 RmlUi Document Viewer 重新截图比较，确保优化没有改变 RCSS 语义、Alpha、Blur、Shadow、Mask、Transform 和 Stencil 结果。
8. 自动化测试最后集中运行，但视觉正确性必须在进入下一项优化前确认。

## 按收益排序的优化任务

### 任务 0：补齐可归因的统计指标

这不是直接的运行时优化，而是所有后续优化的测量前置条件。

状态：**已完成**。

- [x] 增加 `activeLayers`，区分逻辑 Layer 条目和真正拥有有效离屏 Color 的 Layer；Stencil 使用独立字段统计。
- [x] 增加 `resolvePixels`，不能只统计 Resolve 次数。
- [x] 增加 `clearPixels`，区分小区域 Clear 和完整纹理 Clear。
- [x] 增加 `rasterPassPixels` 与 `compositePassPixels`。
- [x] 增加稳定的 Widget/Target 标识，支持多 Widget 日志分析。
- [x] 将一条超长日志拆成命令、Bounds、抗锯齿、Pass、资源五类，并使用相同 frame/report id 关联。
- [x] 明确统计是逻辑累计量还是 RDG 峰值；如需要峰值，使用 RDG/Insights 数据单独记录，不在现有字段上偷换语义。
- [x] 将早期裁剪与真实合批成功数分离，`mergedDraws` 不再通过源命令数和 Draw Call 数反推。
- [x] 将实际执行的 clip-mask replay 纳入 `rasterDrawCalls`，并通过 `clipReplayDrawCalls` 单独报告。
- [x] 合批拒绝原因与状态切换只累计实际加入 RDG 的 Raster Segment。
- [x] 在日志中显式记录 MSAA sample count、几何合批和颜色矩阵融合开关。
- [x] 增加 Base Layer 的 `declaredBounds`、`requiredBounds`、`allocatedBounds` 与可见区域状态，定位无 Scissor 回退和 Composite 反向传播导致的范围扩张。
- [x] 增加 Stencil 实际分配、语义需求、像素范围、Draw Call 与 Attachment Clear 统计，为按需分配建立基线。
- [x] 增加 AA required/optional Layer、像素范围和四类重叠原因统计，为内容自适应 MSAA 建立可归因数据。

验收条件：同一滚动样本可以回答“多少 Layer 实际分配 Color/Stencil”“Resolve 覆盖多少像素”“Clear
覆盖多少像素”“哪些 Layer 可以安全使用 1×”，并能通过相同 `report` 合并五类日志。

### 任务 1：Layer 延迟 Resolve 与 Dirty Tracking

预期收益最高，且不需要降低 MSAA 质量。

状态：**已完成**。

- [x] Raster 或 Composite 写入 MSAA Color 后仅标记 Layer 为 `ResolveDirty`。
- [x] 后续继续写入同一 Layer 时不执行 Resolve。
- [x] 仅在 Composite、Filter、Save、Mask 或最终输出第一次读取 Layer 时执行 Resolve。
- [x] Resolve 后再次写入必须重新标记 dirty。
- [x] 同一 read-after-write epoch 最多 Resolve 一次。
- [x] Masked Snapshot 和 Base Final Composite 使用同一套读取前 Resolve 规则。

主要观测：`resolves`、`resolvePixels`、`offscreenSamples`、GPU Resolve 时间。

验收条件：页面顶部不再出现 Raster Pass 与 Resolve 一一对应；峰值 Resolve 数和 Resolve Pixels 显著下降，所有视觉用例与 Viewer 保持一致。

风险：遗漏 dirty 转换会读取旧 Resolve 内容；过早复用 Resolve 会在同 Layer 多次写入后产生时序错误。

### 任务 2：从最终消费者反向收缩 Layer Bounds

该任务同时减少 Color、Stencil、Resolve、Mask 和后续 Filter 的资源范围。

状态：**已完成保守实现**。当前实现只使用能够证明安全的命令、Scissor、Layer、Save、Composite 和 Filter
范围，不使用 CPU 投影几何 Bounds 进行正确性裁剪。

- [x] 从最终可见 Composite、Save 和 Base 输出建立 Layer 需求区域。
- [x] 沿 Source/Destination 关系反向传播实际所需 Bounds。
- [x] Filter 根据 blur sigma、drop-shadow offset 和采样核扩展必要 halo。
- [x] 不再仅因 Layer Description 存在就把整个描述范围视为本帧必需区域。
- [x] 对完全不可见且无跨帧保存需求的 Layer 标记为空，不创建 RDG 纹理。
- [x] 保持无 scissor 命令的安全规则；无法证明边界时宁可使用 ViewRect，也不能错误裁剪。

主要观测：`activeLayers`、`layerPixels`、`layerSamples`、`stencilSamples`、`offscreenSamples`。

验收条件：在相同 `vw × vh`、MSAA、页面内容和滚动位置下，`requiredBounds` 与 `allocatedBounds` 应保持
一致；滚动离开效果卡片后，对应 `activeLayers`、`layerPixels` 和 `offscreenSamples` 应下降。不得继续使用
缺少正确视口信息的旧日志绝对值作为验收阈值。

风险：反向需求传播错误会造成阴影、Blur halo、Transform 或 inverse mask 被截断。

### 任务 3：Filter Scratch 使用局部纹理与局部 Clear

旧实现曾让 Filter Texture Pool 使用 Source Layer 的完整 extent。当前实现改为按局部 Filter Bounds 建立
临时纹理，本任务记录该优化的已落地边界。

状态：**已完成**。

- [x] Filter scratch extent 改为 `FilterBounds + required halo`。
- [x] 为局部纹理保存独立 Origin，统一修正采样和输出坐标。
- [x] Blur 的降采样、双轴卷积和上采样全部在局部坐标中执行。
- [x] Drop Shadow offset 参与 Bounds 扩展。
- [x] 后续只读取已写区域时使用 `ENoAction`，避免无意义的完整纹理 Clear。
- [x] 必须清透明边界时仅清理必要区域，并计入 `clearPixels`。

主要观测：`filterTexturePixels`、`filterPassPixels`、`clearPasses`、`clearPixels`、Filter GPU 时间。

验收条件：在相同视口和页面状态的 A/B 数据中，`filterTexturePixels / filterPassPixels` 不再因为完整
Source Layer extent 而异常放大；Filter 视觉结果必须与 Viewer 一致。

风险：局部纹理坐标、Blur 边界和 Shadow halo 任何一处错误都会产生接缝、矩形边缘或偏移。

### 任务 4：基于 Scissor 与 Layer Bounds 的 Raster 早期裁剪

该任务主要降低 CPU 准备、Buffer 上传和 Draw Call，而不是首先解决离屏带宽。

状态：**已完成保守实现**。

- [x] 在追加 Vertex/Index 和建立 Prepared Draw 前计算有效 scissor 与目标 Layer Bounds 的交集。
- [x] 正面积为空的普通绘制不进入本帧上传和 Draw Call。
- [x] Stencil write 不能孤立裁剪；当前保留所有仍有 Layer 消费者的 Stencil 状态序列。
- [x] 记录 `culledRasterCommands` 和 `culledUploadBytes`。

当前限制：Transform 后几何不参与精确 CPU Bounds 推导。无 Scissor 命令仍使用保守范围；曾尝试的 CPU
投影估算会在滚动和复杂效果中误裁剪，不能把“已禁用不安全优化”标记为已完成优化。

评估结论：**当前决定不实现 Transform 后 CPU 精确裁剪**。只有新的 Bounds 日志证明完整 ViewRect 回退
构成实际瓶颈，并且能够建立覆盖透视、Filter、Backdrop 和 Clip 的保守边界契约时，才重新开启该任务。

主要观测：`rasterCommands`、`rasterDrawCalls`、`vertices`、`indices`、上传字节数和 RenderThread 时间。

验收条件：滚动导致章节离开有效 Layer/Scissor 范围时，`culledRasterCommands` 与 `culledUploadBytes` 增加，
`preparedRasterCommands` 和 `rasterDrawCalls` 相应下降；裁剪结果不得破坏连续 Stencil、Transform 和 Layer 顺序。

风险：错误跳过 Stencil Set/Intersect 会影响后续所有几何；这是本任务最重要的正确性边界。

### 任务 5：减少 Snapshot、Mask 与 Same-Layer Scratch 范围

状态：**部分完成**。

- [x] SaveLayerAsTexture 只保存调用时语义要求的 Bounds。
- [x] Masked Layer Snapshot 使用反向收缩后的 Source Layer 必要范围。
- [x] Source 与 Destination 相同 Layer 时，Scratch Copy 只复制存在读写冲突的区域。
- [x] 跨帧持久纹理保持现有安全生命周期，不以弱引用换取表面上的内存下降。
- [ ] 同一 Layer、相同写入版本、相同 Stencil reference 和相同 Bounds 的 Snapshot 在同一帧复用。目前
  `WriteVersion` 只随 Layer 写入递增，尚未建立 Snapshot cache key 与复用表。

主要观测：`snapshotPixels`、`scratchPixels`、`copyPasses`、`maskPasses` 和 copy GPU 时间。

验收条件：保存纹理、mask-image、box-shadow 和 backdrop-filter 的视觉与生命周期测试全部通过，同时 Snapshot/Scratch Pixels 随实际 Bounds 缩小。

### 任务 6：严格相邻几何合批

合批只能处理绘制顺序中相邻且状态完全兼容的命令。不能跨越 Stencil、Layer、Blend 或透明顺序边界排序。

状态：**基础实现已完成**。它只保证安全地合并严格兼容的相邻 Draw，不承诺测试页具有较高命中率。

- [x] 统计导致 `CanMerge` 失败的原因分布。
- [x] 在不改变顺序的前提下合并相同 Texture、Shader、Layer、Alpha、Transform、Scissor 和 Stencil 状态的连续绘制。
- [x] 保留合批开关，便于视觉和性能 A/B。

当前限制：平移烘焙、实例数据、跨纹理 atlas 和 bindless 没有实现。它们属于扩大命中率的后续工作，
不能因为已经讨论或评估过就标记为完成。

评估结论：**当前决定不实现这些合批扩展**。现有样本的合批率约为 `0.08%`，尚无证据证明扩大兼容范围
能够抵消资源架构、实例数据和透明顺序验证成本；对应工作保留在未完成任务 4。

主要观测：`mergedDraws`、`mergeRate`、`rasterDrawCalls`、RenderThread 时间和 GPU submission 时间。

验收条件：真实页面出现稳定命中并降低 Draw Call；若测试页仍接近 0%，必须由失败原因统计证明是命令状态天然不兼容，而不是合批器失效。

风险：透明 UI 不能任意重排；错误扩大兼容条件会改变 Alpha、Transform、Scissor 或 Stencil 语义。

### 任务 7：Raster Layer 内容自适应 MSAA

状态：**已完成保守的整 Layer 自适应路径**。

- [x] 将全局 MSAA 设置解释为质量上限，而不是所有离屏资源必须使用相同 sample count。
- [x] 编译几何记录边界是否轴对齐、边界顶点是否具有一致像素偏移等不可变事实。
- [x] 每个 Prepared Draw 统一分类 Geometry Edge、Transform、Clip Mask 和 Pixel Misalignment AA 需求。
- [x] 只要一个直接 Draw 需要 AA，整个 Layer 保持全局 sample count；只有全部 Draw 均证明安全时才使用 1×。
- [x] 任意非 Identity 2D/3D Transform、Clip Mask 写入/读取和 Clip replay 均保守要求 AA。
- [x] 不含 Raster Draw 的 Composite-only Layer 使用 1×，且 Source Layer 的 AA 需求不传播到 Destination。
- [x] Filter ping-pong、颜色矩阵、Snapshot、Scratch 和其他采样后处理纹理保持 1×。
- [x] 增加独立 `category=antialiasing` 统计和 1×/4× GPU 像素回归。

当前限制：没有在同一 Layer 内建立不同 SampleCount 的局部区域；无法证明边界安全的几何会产生保守的
false positive 并继续使用 MSAA。当前也不会根据纹理内容猜测透明 padding 是否足以替代几何边缘覆盖。

开关关闭时，所有有效 Layer 继续使用全局 sample count，但 AA required/optional 统计仍然输出，便于做严格 A/B。

主要观测：`layerSamples`、`stencilSamples`、`offscreenSamples`、Resolve 时间和视觉差异。

验收条件：在不降低测试页视觉一致性的前提下减少 sample slot 总量；不得以全局关闭 MSAA 作为任务完成标准。

### 任务 8：可选连续颜色矩阵融合

状态：**可选路径已完成，默认路径未改变**。它不能被描述为默认渲染已经减少 Filter Pass。

- [x] 实现连续颜色矩阵 Filter 的可选融合模式。
- [x] 默认仍保持官方多 Pass、逐步 UNORM 写回和 clamp 语义；融合由显式设置开启。
- [x] 对融合路径建立 RGBA8 单通道 1/255 的像素误差阈值，并记录它不保证与 Viewer 逐步 clamp 一致。

主要观测：Pass 数、状态切换、Filter GPU 时间和像素回归结果。

风险：颜色矩阵融合会移除官方每一步的中间 UNORM clamp 与量化，不属于完全无损的内部优化。

### 任务 9：Stencil 按需分配

状态：**已完成**。

- [x] 仅为实际执行 Clip Mask 写入、Stencil 测试、Mask Replay 或受裁剪 Composite 的 Layer 分配 Stencil。
- [x] 不需要 Stencil 的 Raster Layer 不创建 Depth/Stencil Texture，也不绑定或清理 Stencil Attachment。
- [x] `stencilLayers` / `stencilPixels` 记录实际分配，`stencilRequiredLayers` / `stencilRequiredPixels` 记录语义需求。
- [x] Set、SetInverse、Intersect、SaveLayerAsMask、Backdrop 和 Composite Destination Clip 由完整 GPU 回归覆盖。

正常情况下，实际分配与语义需求应相等；若新日志出现差异，优先视为资源规划或统计契约回归。

## 未完成优化任务

下面只列尚未落地的代码工作。“完成分析”“决定暂不实现”或“存在实验开关”均不算完成。顺序依据当前
预期收益、实现风险和现有日志证据排列；每次取得新日志后应重新排序。

### 未完成 1：同帧 Snapshot 复用

- [ ] 建立包含 Layer handle、Layer write version、Stencil reference、Bounds 和 Snapshot 类型的完整 cache key。
- [ ] 相同 key 的 Save、Mask 或 Composite 消费者复用同一张 RDG Snapshot。
- [ ] 任意 Layer 写入、Stencil reference 变化或 Bounds 不同都必须产生新 Snapshot。
- [ ] 增加命中次数、避免的 Snapshot Pixels 和 Copy Pass 统计后再评估收益。

### 未完成 2：扩大几何合批兼容范围

- [ ] 在命中收益得到证明后，实现纯平移顶点烘焙或实例数据，使仅 Translation 不同的相邻 Draw 可以合并。
- [ ] 在纹理切换被证明是主要边界后，实现文字 atlas、图片 atlas 或 bindless 纹理路径。
- [ ] 不允许跨越透明顺序、Layer、Stencil、Scissor、Composite、Save 或 Filter 屏障重排。

开始条件：现有日志中基础相邻合批率长期约为 `0.08%`，因此该任务优先级低于资源范围优化。只有新的
拒绝原因统计表明某一种兼容扩展能够稳定命中时才应实施。

### 未完成 3：更精确的 Transform Bounds

- [ ] 使用 `category=bounds` 日志确认是否经常出现 `declaredBounds`、`requiredBounds` 或
  `allocatedBounds` 异常扩大到完整 Widget。
- [ ] 若 `regionStatus=full-view-fallback`，先定位具体无 Scissor Draw 或 Composite，不直接恢复旧的 CPU
  投影裁剪。
- [ ] 若确需精确 Bounds，必须覆盖透视 W、Blur/Shadow halo、Backdrop 目标采样、inverse clip 和
  Source/Destination 反向依赖。
- [ ] 没有日志证据和完整视觉回归前，不使用 Transform 后 CPU Bounds 进行正确性剔除。

### 未完成 4：同 Layer 局部混合采样

- [ ] 在一个逻辑 Layer 内把可证明安全的局部区域拆到 1×，只为其他区域保留 MSAA。
- [ ] 保持透明顺序、Clip Mask、Filter、Composite 和 Resolve 语义不变。
- [ ] 证明新增 RenderTarget、Composite 和同步成本低于减少的 sample slot 成本。

当前决定：**不实现**。整 Layer 自适应已经覆盖低风险命中；局部拆分会增加资源、Pass 与合成边界，只有新日志
证明大面积 Layer 长期因少量 AA Draw 被迫保留 MSAA，并且 GPU 工具确认它是实际瓶颈时才重新评估。

### 未完成 5：默认路径的状态与 Pass 优化

- [ ] 使用 GPU Visualizer、Unreal Insights、RDG Insights 或 RenderDoc 确认实际昂贵的 Pass 和状态边界。
- [ ] 寻找不移除中间 UNORM clamp、不改变透明顺序且能默认启用的无损 Pass 合并。
- [ ] Stencil Set、SetInverse、Intersect 和 reference 变化仍是严格语义边界，不能为了降低
  `stencilSwitches` 强行合并。
- [ ] 连续颜色矩阵融合保持可选，除非未来明确修改“与 Viewer 逐 Pass 一致”的默认契约。

## 优化优先级总结

| 顺序 | 工作项 | 当前状态 | 主要收益 | 正确性风险 |
| ---: | --- | --- | --- | --- |
| 0 | 补齐统计 | 已完成 | 让后续结果可归因 | 低 |
| 1 | 延迟 Resolve | 已完成 | 大幅减少 MSAA 带宽和 Resolve Pass | 中高 |
| 2 | 反向收缩 Layer Bounds | 已完成保守实现 | 同时减少 Color、Stencil、Resolve、Mask 和 Filter 范围 | 高 |
| 3 | 局部 Filter Scratch/Clear | 已完成 | 大幅减少 Filter 临时纹理和清屏带宽 | 高 |
| 4 | Raster 早期裁剪 | 已完成保守实现 | 降低 Draw Call、上传和 RenderThread 工作 | 高 |
| 5 | Snapshot/Mask/Scratch 收缩 | 部分完成 | 减少复制和临时纹理 | 中 |
| 6 | 严格相邻几何合批 | 基础实现已完成 | 在严格兼容时降低 Draw Call | 中高 |
| 7 | Raster Layer 内容自适应 MSAA | 已完成保守整 Layer 分流 | 安全 Layer 使用 1×，其余保持全局质量 | 中 |
| 8 | 可选颜色矩阵融合 | 可选路径已完成 | 显式 A/B 时减少颜色 Filter Pass | 高，可能改变 Viewer 一致性 |
| 9 | Stencil 按需分配 | 已完成 | 避免无裁剪 Layer 的 Stencil 资源与 Clear | 中 |

未完成工作的当前优先级：

| 优先级 | 未完成工作 | 当前决定 |
| ---: | --- | --- |
| 1 | 同帧 Snapshot 复用 | 尚未实现；先增加命中与避免像素统计 |
| 2 | 扩大几何合批兼容范围 | 当前决定不实现；现有约 `0.08%` 命中率不足以证明架构改造收益 |
| 3 | 更精确的 Transform Bounds | 当前决定不实现；只有 Bounds 日志证明异常完整视口扩张后重新开启 |
| 4 | 同 Layer 局部混合采样 | 当前决定不实现；需证明收益能够覆盖新增资源和 Composite 成本 |
| 5 | 默认路径的状态与 Pass 优化 | 尚无具体无损方案；等待 GPU 工具定位真实热点 |

排序依据是当前测试页日志暴露出的资源规模和可复用范围，不是永久不变的架构优先级。每完成一个任务都应重新采样；如果主要瓶颈发生变化，后续顺序应由新数据调整。

## 本轮实现状态

任务 0–4 已完成当前定义的保守实现；任务 5 仍缺少同帧 Snapshot 复用；任务 6 完成严格相邻合批，但扩大
兼容范围尚未实现；任务 7 完成保守的整 Layer 内容自适应 MSAA；任务 8 只完成默认关闭的可选颜色矩阵融合；
任务 9 完成 Stencil 按需分配。默认 Filter 路径继续保持 RmlUi 官方的多 Pass、逐步 UNORM 写回语义。

已完成的本地验证：

- Development Editor 构建通过。
- MarkupUI 完整 C++ Automation Test 套件 79/79 通过，覆盖 AA 原因分类、整 Layer 自适应 1×/4× 路径、
  Layer 保存与生命周期、Mask、Backdrop、Blur、Drop Shadow、同 Layer scratch、几何合批及可选矩阵融合。
- 当前旧格式基线为 `PerformanceLogs/2026-09-03_00-47-31-baseline.txt`，包含 Bounds、Stencil 和四类统计行，
  可用于后续相同视口、DPI、页面状态与滚动路线下的严格 A/B；外部 GPU 时间、实际显存带宽和 RDG 峰值
  仍需 Insights、GPU Visualizer、RDG Insights、RenderDoc 或硬件厂商分析工具。它不包含新
  `category=antialiasing`，因此不能用于分析 AA 命中率；需要用当前版本重新录制五类统计的新基线。
