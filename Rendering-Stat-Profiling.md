# MarkupUI 渲染 Stat 与性能分析路线

## 目标

为 MarkupUI 建立可长期维护的 CPU、RenderThread 和 GPU 时间观测体系，使性能结论能够回答：

- 一帧时间花在前端更新、命令准备、RDG 调度还是 GPU 执行；
- Raster、Composite、Filter、Blur、Mask、Copy 和 Resolve 中哪一类工作最昂贵；
- 现有逻辑统计变化是否真的对应 CPU 或 GPU 时间变化；
- 优化是否降低真实耗时，而不只是减少命令数、像素范围或 sample slot。

这项工作属于性能可观测性建设，不直接改变渲染结果，也不应为了获得更漂亮的统计数字改变 RmlUi 语义。

## 工具能力边界

| 工具 | 适合回答的问题 | 不能单独证明的内容 |
| --- | --- | --- |
| Unreal Insights | Game、Render、RHI 线程耗时，CPU 等待关系，多帧 GPU Scope 时间分布 | Shader Occupancy、Cache 命中率和精确显存带宽 |
| GPU Visualizer | 单帧 GPU Pass 层级、各 Scope 聚合时间 | 长时间 CPU/GPU 调度关系和硬件级瓶颈 |
| RDG Insights | RDG Pass、资源生命周期和依赖关系 | GPU 核心利用率与驱动内部行为 |
| RenderDoc | 单帧 Draw、纹理、RenderTarget、Pipeline State 和 Shader 调试 | 代表性长时间性能分布 |
| PIX、Nsight、Radeon GPU Profiler | 硬件计数器、Occupancy、Cache、带宽和 Wave 分析 | 跨平台统一结论 |

Unreal Insights 可以精确记录启用 GPU Trace 后各个标记范围的 GPU 时间，但不能把这些时间自动解释成完整的
GPU 利用率、实际显存带宽或 Shader 执行效率。硬件级结论必须由对应平台分析器验证。

## 与现有统计的关系

现有 `[MarkupUI Statistics]` 日志继续承担“工作量和资源范围”诊断，字段契约见
[Rendering-Performance.md](Rendering-Performance.md)。它记录的是逻辑数量，不是计时器：

- `commands`、`rasterDrawCalls` 和各类 Pass 数用于解释调度规模；
- `declaredBounds`、`requiredBounds` 和 `allocatedBounds` 用于解释范围传播；
- `resolvePixels`、`filterPassPixels` 和 `offscreenSamples` 用于解释逻辑工作量；
- CPU/GPU Scope 用于证明这些变化是否产生真实耗时收益。

分析时必须同时保留视口、DPI、全局 MSAA、页面状态、滚动位置和动画状态。只减少逻辑计数、但没有降低对应
CPU 或 GPU 时间的改动，不能被认定为性能优化完成。

## 埋点分层

### CPU Trace Scope

使用稳定的 `TRACE_CPUPROFILER_EVENT_SCOPE` 名称覆盖主要阶段：

- `MarkupUI Frontend Update`
- `MarkupUI Frontend Render`
- `MarkupUI Pipeline Submit`
- `MarkupUI Layer Plan Build`
- `MarkupUI Bounds Propagation`
- `MarkupUI Command Preparation`
- `MarkupUI Geometry Upload Preparation`
- `MarkupUI RDG Pass Scheduling`
- `MarkupUI Saved Resource Processing`

Scope 应包围完整阶段，不为每条 Draw、每个 DOM Element 或动态资源名称创建事件。这样既能看出 CPU 优化的收益，
也能避免测试页上千条命令导致 Trace 数据量和采集开销反过来污染结果。

### RDG 与 GPU Scope

使用稳定的 `RDG_EVENT_SCOPE` 建立 RDG/RenderDoc 层级，并使用 `RDG_GPU_STAT_SCOPE` 为 GPU Visualizer 提供聚合：

- `MarkupUI Raster`
- `MarkupUI Composite`
- `MarkupUI Color Filter`
- `MarkupUI Blur`
- `MarkupUI Drop Shadow`
- `MarkupUI Backdrop`
- `MarkupUI Mask`
- `MarkupUI Copy`
- `MarkupUI Resolve`

可以按功能类别和必要的 Layer 生命周期分组，但不得把 Handle、URI 或逐 Draw 序号拼进 Scope 名称。动态名称会
破坏跨帧聚合，并扩大 Trace 和 GPU marker 成本。

### Trace Counter 与关联事件

优先把以下现有统计作为低频 Trace Counter，按一次 Executor 报告更新，而不是按命令累加：

- Raster Draw Calls
- Active Layers
- Resolve Pixels
- Filter Pass Pixels
- Offscreen Samples
- Stencil Pixels
- Vertex Upload Bytes
- Index Upload Bytes

多个 Widget 同时绘制时，全局 Counter 不能独立表示单个 Target。因此还需要一个轻量关联事件携带 `report`、
`target`、`vw`、`vh`、`msaaSamples`，用于把时间线与同一执行报告对应起来。不得通过为每个 Target 动态创建
Counter 名称来解决分组问题。

## 开销控制

- Trace 未启用时，埋点不得改变渲染命令、资源生命周期或像素结果。
- 不创建逐 Draw、逐顶点或逐资源 URI 的 CPU/GPU Scope。
- GPU timestamp 本身存在开销；性能结论应包含一次关闭 GPU Trace 的对照采样。
- `r.MarkupUI.PerformanceLog=2` 会持续输出日志，不应与最终 GPU 基准同时长期开启；需要字段关联时优先使用
  Trace Counter 和关联事件。
- Shipping 构建不依赖诊断数据维持任何正确性状态。

## 采样规范

每次正式比较至少记录：

1. Unreal Engine、RHI、GPU、驱动和构建配置；
2. 页面资产版本、视口、DPI、全局 MSAA 和 MarkupUI 性能开关；
3. 固定的交互路线、滚动范围、预热时间与采样时长；
4. CPU Game/Render/RHI 线程时间和 MarkupUI CPU Scope；
5. 总 GPU Frame 时间及 MarkupUI 各类 GPU Scope；
6. 同场景 `[MarkupUI Statistics]` 逻辑统计；
7. 至少三次独立采样，并报告中位数和高分位，而不是只选择最好的一帧。

若要比较 RmlUi 官方后端，必须使用相同 GPU、分辨率、页面内容、字体、图片、MSAA 和动画时间点。不同窗口系统、
交换链和 Present 策略的总帧时间不能直接归因于 MarkupUI 或 RmlUi 后端本身。

## 实施任务

### 任务 S1：建立可重复基线

- [ ] 固定视觉全功能页、AA 页和一个接近真实游戏 UI 的页面版本。
- [ ] 固定三种采样路线：静止、连续滚动、动画与 Filter 活跃。
- [ ] 记录视口、DPI、MSAA、构建配置和采样时长。
- [ ] 保存未加入新 Trace Scope 前的 CPU、GPU 与现有逻辑统计基线。

### 任务 S2：CPU 阶段埋点

- [ ] 加入 Frontend、Pipeline、计划构建、Bounds、命令准备、上传准备和 RDG 调度 Scope。
- [ ] 验证 RenderThread 与调用线程的 Scope 落在正确线程轨道。
- [ ] 确认关闭 Trace 时没有可测量的帧时间回退。

### 任务 S3：RDG 与 GPU 阶段埋点

- [ ] 为 Raster、Composite、各类 Filter、Mask、Copy 和 Resolve 建立稳定层级。
- [ ] 验证 GPU Visualizer、Unreal Insights GPU Track 和 RenderDoc 中名称与层级一致。
- [ ] 避免逐 Draw 动态 Scope，并检查采集开启后的 timestamp 开销。

### 任务 S4：Counter 与报告关联

- [ ] 将高价值逻辑指标接入 Trace Counter。
- [ ] 加入包含 `report`、`target`、视口和 MSAA 的关联事件。
- [ ] 验证多个 Widget 同时绘制时不会把某个 Target 的 Counter 误读为全局累计量。

### 任务 S5：建立分析报告模板

- [ ] 同时报告 CPU、GPU、逻辑工作量和资源范围，不用单一指标宣布收益。
- [ ] 对比中报告中位数、P95、样本数量和 Trace 开销对照。
- [ ] 明确区分已测量结果、从统计推导的趋势和仍需硬件工具验证的假设。

### 任务 S6：决定是否继续优化

- [ ] 只有某个 Scope 在代表性页面中构成稳定热点时，才为它增加新的优化任务。
- [ ] 若 MarkupUI 的 CPU/GPU 时间已满足预算，结束性能阶段，不继续增加推测性复杂度。
- [ ] 若需要宣称优于官方后端，使用严格等价场景完成独立 A/B，并保留原始捕获数据。

## 完成条件

- Unreal Insights 能区分 MarkupUI 的主要 CPU 和 GPU 阶段。
- GPU Visualizer 和 RenderDoc 中具有稳定、可聚合的 MarkupUI RDG 层级。
- 时间指标能够通过 `report` 和 `target` 与现有逻辑统计关联。
- 多 Widget、滚动、Filter、Mask、Backdrop 和 MSAA 场景均能被可靠采样。
- 关闭诊断时不改变像素结果、资源生命周期和渲染错误处理。
- 文档能够明确回答某个结论来自计时、逻辑统计还是硬件计数器。
