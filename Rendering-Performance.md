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

性能数据行统一以 `[MarkupUI Statistics]` 开头。每份报告由 `commands`、`passes` 和 `resources` 三行组成，三行共享相同的 `report`、`target`、`vw` 和 `vh`。复制或解析日志时必须保留并合并同一 `report` 的三行，不能把其中一行当作完整报告。

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
| `report` | 同一次统计报告的编号 | 使用它合并 `commands`、`passes` 和 `resources` 三行 |
| `target` | 当前绘制目标在本次进程中的诊断标识 | 多个 Widget 同时输出日志时用于分组；不要把它当作跨进程稳定 ID |
| `vw` | 当前 Draw Target 的实际像素视口宽度 | Slate Widget 中对应 Widget 视口，而不是窗口或 BackBuffer 宽度 |
| `vh` | 当前 Draw Target 的实际像素视口高度 | 视口不同的原始像素、采样和带宽压力数据不能直接横向比较 |
| `msaaSamples` | 本次执行使用的 MarkupUI MSAA sample count | A/B 对比时必须保持一致 |
| `geometryBatching` | 相邻几何合批开关，`1` 为开启，`0` 为关闭 | 不再依赖文件名或拒绝计数反推测试配置 |
| `colorMatrixFusion` | 连续颜色矩阵融合开关，`1` 为开启，`0` 为关闭 | 开启后不再保证 Viewer 的逐 Pass UNORM clamp 语义 |

比较两份日志时，应先按 `target` 分组，再只比较 `vw × vh` 相同的报告。若必须比较不同分辨率，至少同时观察按视口面积归一化后的指标，例如：

```text
normalizedOffscreenSamples = offscreenSamples / (vw × vh)
```

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
| `resolves` | MSAA RenderTarget Resolve 操作数量 | Raster、Composite、Mask Snapshot 等带 resolve attachment 的写入 |
| `rasterPassPixels` | Raster Pass 目标 Layer 范围的累计面积 | 配合 Raster Pass 数判断 Pass 是否仍覆盖过大的 Layer |
| `compositePassPixels` | Composite 与最终 Base 回写范围的累计面积 | 判断合成次数减少是否同时缩小覆盖面积 |
| `resolvePixels` | 实际 Resolve 操作覆盖的累计面积 | 延迟 Resolve 的核心带宽趋势指标 |
| `clearPixels` | 显式 Clear 覆盖的累计面积 | 区分少量局部 Clear 与大范围清屏 |

分析顺序建议为：先看每类 Pass 数，再看对应像素范围。仅减少 Pass 数但扩大覆盖范围，可能使总成本反而上升。

### Layer、Stencil 与 MSAA

| 字段 | 含义 | 是否包含 Sample Count |
| --- | --- | --- |
| `layerPixels` | 所有有效离屏 Layer Bounds 的二维像素面积总和 | 否 |
| `layerSamples` | Layer Color 的 sample slot 总量；MSAA Color 按 sample count 计算，存在 Resolve 纹理时再加入一份单采样面积 | 是 |
| `stencilSamples` | Layer Stencil 的 sample slot 总量 | 是 |

例如 2× MSAA 且每个 Layer 都有 Resolve Texture 时，通常会观察到：

```text
layerSamples   ≈ layerPixels × 3
stencilSamples ≈ layerPixels × 2
```

这里的 `3` 是 `2× MSAA Color + 1× Resolve`，不是三倍过度绘制。它仍然表示真实的资源规模和潜在带宽压力。

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

`PerformanceLogs/2026-09-02-baseline.txt` 是优化前的一次页面滚动历史样本。该日志生成时 `vw`、`vh` 尚未记录 Widget 实际视口，因此只能用于观察同一份日志内部的阶段变化，不能依靠其中的视口字段与新日志做归一化对比。

`PerformanceLogs/2026-09-02_22-31-00-no-batch.txt` 与
`PerformanceLogs/2026-09-02_22-35-41-batch.txt` 使用修正前的合批统计公式：其中 `mergedDraws` 错误包含
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

- [x] 增加 `activeLayers`，区分逻辑 Layer 条目和真正创建 Color/Stencil 的 Layer。
- [x] 增加 `resolvePixels`，不能只统计 Resolve 次数。
- [x] 增加 `clearPixels`，区分小区域 Clear 和完整纹理 Clear。
- [x] 增加 `rasterPassPixels` 与 `compositePassPixels`。
- [x] 增加稳定的 Widget/Target 标识，支持多 Widget 日志分析。
- [x] 将一条超长日志拆成命令、Pass、资源三行，并使用相同 frame/report id 关联。
- [x] 明确统计是逻辑累计量还是 RDG 峰值；如需要峰值，使用 RDG/Insights 数据单独记录，不在现有字段上偷换语义。
- [x] 将早期裁剪与真实合批成功数分离，`mergedDraws` 不再通过源命令数和 Draw Call 数反推。
- [x] 将实际执行的 clip-mask replay 纳入 `rasterDrawCalls`，并通过 `clipReplayDrawCalls` 单独报告。
- [x] 合批拒绝原因与状态切换只累计实际加入 RDG 的 Raster Segment。
- [x] 在日志中显式记录 MSAA sample count、几何合批和颜色矩阵融合开关。

验收条件：同一滚动样本可以回答“多少 Layer 实际分配”“Resolve 覆盖多少像素”“Clear 覆盖多少像素”，且日志不会被常用复制路径截断。

### 任务 1：Layer 延迟 Resolve 与 Dirty Tracking

预期收益最高，且不需要降低 MSAA 质量。

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

- [x] 从最终可见 Composite、Save 和 Base 输出建立 Layer 需求区域。
- [x] 沿 Source/Destination 关系反向传播实际所需 Bounds。
- [x] Filter 根据 blur sigma、drop-shadow offset 和采样核扩展必要 halo。
- [x] 不再仅因 Layer Description 存在就把整个描述范围视为本帧必需区域。
- [x] 对完全不可见且无跨帧保存需求的 Layer 标记为空，不创建 RDG 纹理。
- [x] 保持无 scissor 命令的安全规则；无法证明边界时宁可使用 ViewRect，也不能错误裁剪。

主要观测：`activeLayers`、`layerPixels`、`layerSamples`、`stencilSamples`、`offscreenSamples`。

验收条件：页面顶部没有高级效果可见时，Layer Pixels 应明显低于当前约 35.65M；滚动离开效果卡片后，对应 Layer 成本应下降而不是保持峰值。

风险：反向需求传播错误会造成阴影、Blur halo、Transform 或 inverse mask 被截断。

### 任务 3：Filter Scratch 使用局部纹理与局部 Clear

当前 Filter Texture Pool 使用 Source Layer 的完整 extent。对于只覆盖小卡片的 Filter，这会创建和清理远大于实际 Viewport 的临时纹理。

- [x] Filter scratch extent 改为 `FilterBounds + required halo`。
- [x] 为局部纹理保存独立 Origin，统一修正采样和输出坐标。
- [x] Blur 的降采样、双轴卷积和上采样全部在局部坐标中执行。
- [x] Drop Shadow offset 参与 Bounds 扩展。
- [x] 后续只读取已写区域时使用 `ENoAction`，避免无意义的完整纹理 Clear。
- [x] 必须清透明边界时仅清理必要区域，并计入 `clearPixels`。

主要观测：`filterTexturePixels`、`filterPassPixels`、`clearPasses`、`clearPixels`、Filter GPU 时间。

验收条件：`filterTexturePixels / filterPassPixels` 比例显著降低；当前峰值约 67.8M 的 Filter 临时纹理面积不再由完整 Source Layer 主导。

风险：局部纹理坐标、Blur 边界和 Shadow halo 任何一处错误都会产生接缝、矩形边缘或偏移。

### 任务 4：不可见 Raster Command 早期裁剪

该任务主要降低 CPU 准备、Buffer 上传和 Draw Call，而不是首先解决离屏带宽。

- [x] 在追加 Vertex/Index 和建立 Prepared Draw 前计算有效 scissor 与目标 Layer Bounds 的交集。
- [x] 正面积为空的普通绘制不进入本帧上传和 Draw Call。
- [x] Stencil write 不能孤立裁剪；当前保留所有仍有 Layer 消费者的 Stencil 状态序列。
- [x] Transform 后 Bounds 当前始终使用保守范围；CPU 投影估算曾导致滚动误裁剪，已明确禁止用于正确性剔除。
- [x] 记录 `culledRasterCommands` 和 `culledUploadBytes`。

主要观测：`rasterCommands`、`rasterDrawCalls`、`vertices`、`indices`、上传字节数和 RenderThread 时间。

验收条件：滚动到页面顶部或底部时，不可见章节不再维持约 1200 个 Draw Call；裁剪结果不得破坏连续 Stencil、Transform 和 Layer 顺序。

风险：错误跳过 Stencil Set/Intersect 会影响后续所有几何；这是本任务最重要的正确性边界。

### 任务 5：减少 Snapshot、Mask 与 Same-Layer Scratch 范围

- [x] SaveLayerAsTexture 只保存调用时语义要求的 Bounds。
- [x] Masked Layer Snapshot 使用反向收缩后的 Source Layer 必要范围。
- [x] Source 与 Destination 相同 Layer 时，Scratch Copy 只复制存在读写冲突的区域。
- [x] 同一 Layer 写入版本与 Stencil reference 的快照在同一帧被多个消费者读取时复用。
- [x] 跨帧持久纹理保持现有安全生命周期，不以弱引用换取表面上的内存下降。

主要观测：`snapshotPixels`、`scratchPixels`、`copyPasses`、`maskPasses` 和 copy GPU 时间。

验收条件：保存纹理、mask-image、box-shadow 和 backdrop-filter 的视觉与生命周期测试全部通过，同时 Snapshot/Scratch Pixels 随实际 Bounds 缩小。

### 任务 6：提高几何合批命中率

合批只能处理绘制顺序中相邻且状态完全兼容的命令。不能跨越 Stencil、Layer、Blend 或透明顺序边界排序。

- [x] 统计导致 `CanMerge` 失败的原因分布。
- [x] 在不改变顺序的前提下合并相同 Texture、Shader、Layer、Alpha、Transform、Scissor 和 Stencil 状态的连续绘制。
- [x] 评估把平移烘焙进顶点或改为实例数据是否能扩大兼容范围；当前保留顶点复用与精确 Transform 边界，不引入额外实例流。
- [x] 评估文字 atlas、图片 atlas 或 bindless 路径；它们属于资源架构改造，不并入基础透明顺序合批器。
- [x] 保留合批开关，便于视觉和性能 A/B。

主要观测：`mergedDraws`、`mergeRate`、`rasterDrawCalls`、RenderThread 时间和 GPU submission 时间。

验收条件：真实页面出现稳定命中并降低 Draw Call；若测试页仍接近 0%，必须由失败原因统计证明是命令状态天然不兼容，而不是合批器失效。

风险：透明 UI 不能任意重排；错误扩大兼容条件会改变 Alpha、Transform、Scissor 或 Stencil 语义。

### 任务 7：按 Layer 需求选择 MSAA

这是潜在收益很高但会影响视觉质量契约的后期任务，应在延迟 Resolve 和 Bounds 收缩完成后再评估。

- [x] 将全局 MSAA 设置解释为质量上限，而不是所有离屏资源必须使用相同 sample count。
- [x] 直接几何、圆角边缘、Transform 和 Stencil Layer 保留所需 MSAA。
- [x] 纯 Filter ping-pong、颜色矩阵、Composite-only Layer 和已经被采样重建的中间纹理保持 1×。
- [x] 评估仅含轴对齐不透明几何的 Layer；现有命令契约不能可靠证明边缘覆盖，暂不降为 1×。
- [x] 使用现有 1×/4× MSAA GPU 像素用例验证 Layer 间转换。

主要观测：`layerSamples`、`stencilSamples`、`offscreenSamples`、Resolve 时间和视觉差异。

验收条件：在不降低测试页视觉一致性的前提下减少 sample slot 总量；不得以全局关闭 MSAA 作为任务完成标准。

### 任务 8：状态与 Pass 级后期优化

仅在前述主要资源范围问题解决后进行。

- [x] 分析 `stencilSwitches` 的来源；严格保留 Set/SetInverse/Intersect 与 reference 变化，不跨语义边界合并。
- [x] 合并兼容的连续几何；Layer Plan 中 Raster Segment 均由有序副作用切分，不跨 Composite/Save 边界合并 Pass。
- [x] 实现连续颜色矩阵 Filter 的可选融合模式。
- [x] 默认仍保持官方多 Pass、逐步 UNORM 写回和 clamp 语义；融合由显式设置开启。
- [x] 对融合路径建立 RGBA8 单通道 1/255 的像素误差阈值，并记录它不保证与 Viewer 逐步 clamp 一致。

主要观测：Pass 数、状态切换、Filter GPU 时间和像素回归结果。

风险：颜色矩阵融合会移除官方每一步的中间 UNORM clamp 与量化，不属于完全无损的内部优化。

## 优化优先级总结

| 顺序 | 工作项 | 主要收益 | 正确性风险 |
| ---: | --- | --- | --- |
| 0 | 补齐统计 | 让后续结果可归因 | 低 |
| 1 | 延迟 Resolve | 大幅减少 MSAA 带宽和 Resolve Pass | 中高 |
| 2 | 反向收缩 Layer Bounds | 同时减少 Color、Stencil、Resolve、Mask 和 Filter 范围 | 高 |
| 3 | 局部 Filter Scratch/Clear | 大幅减少 Filter 临时纹理和清屏带宽 | 高 |
| 4 | Raster 早期裁剪 | 降低 Draw Call、上传和 RenderThread 工作 | 高 |
| 5 | Snapshot/Mask/Scratch 收缩 | 减少复制和临时纹理 | 中 |
| 6 | 几何合批 | 降低 Draw Call 和提交开销 | 中高 |
| 7 | Layer 自适应 MSAA | 进一步减少 sample 数和带宽 | 高，涉及质量策略 |
| 8 | 状态与 Pass 融合 | 收尾优化 | 高，可能改变 Viewer 一致性 |

排序依据是当前测试页日志暴露出的资源规模和可复用范围，不是永久不变的架构优先级。每完成一个任务都应重新采样；如果主要瓶颈发生变化，后续顺序应由新数据调整。

## 本轮实现状态

任务 0–8 的代码侧项目已经完成。默认路径继续保持 RmlUi 官方的多 Pass、逐步 UNORM 写回语义；
`bEnableColorMatrixFilterFusion` 默认关闭，只用于显式性能 A/B。

已完成的本地验证：

- Development Editor 构建通过。
- MarkupUI 完整 C++ Automation Test 套件 69/69 通过，覆盖 1×/4× MSAA、Layer 保存与生命周期、Mask、
  Backdrop、Blur、Drop Shadow、同 Layer scratch、几何合批及可选矩阵融合。
- 当前代码需要重新录制包含正确 Widget `vw`、`vh` 的真实页面性能日志。旧基线位于
  `PerformanceLogs/2026-09-02-baseline.txt`，由于缺少正确视口语义，只能作为历史趋势参考；外部 GPU 时间、
  实际显存带宽和 RDG 峰值仍需 Insights、GPU Visualizer、RDG Insights、RenderDoc 或硬件厂商分析工具。
