# Slate RDG Layer 执行与资源复用设计

## 目标与边界

Slate Target 必须忠实重放通用 Draw 命令流，而不是重新解释 RCSS。Layer 的逻辑 handle、
页面坐标区域和命令顺序由 Pipeline 决定；RDG texture、stencil、resolve 和 GPU 生命周期由
Slate Target 决定。

本设计区分两类资源：

- 帧内临时 Layer：只在一次 `Draw_RenderThread` 的 RDG graph 中存活。
- 跨帧保存结果：由 `SaveLayerAsTexture` 或 `SaveLayerAsMask` 创建，后续帧仍可引用。

二者不能共用同一种生命周期管理方式。

## Layer 坐标契约

`FPushLayerCommand` 携带：

- `Bounds`：Frontend viewport 坐标中的有效矩形。
- `Origin`：局部纹理 `(0, 0)` 对应的 Frontend viewport 坐标。
- `Layer`：新 Layer 的逻辑 handle。
- `ParentLayer`：Push 前的活动 Layer。

当前 Pipeline 使用以下规则生成区域：

```text
Scissor enabled: Bounds = Scissor intersect Viewport
Scissor disabled: Bounds = Viewport
Origin = Bounds.Min
```

Target 分配的纹理尺寸为 `Bounds.Size()`。页面点转换到 Layer 局部像素时使用：

```text
LocalPosition = PagePosition - Origin
```

空 Bounds 是合法的空 Layer，不分配 GPU texture，后续 draw 和 composite 为 no-op；不能为了
方便而回退到完整 Viewport，因为这会隐藏上游区域错误并增加显存与带宽。

## RenderThread 数据模型

Slate executor 在每帧建立仅属于当前 RDG graph 的 Layer 表：

```cpp
struct FSlateRDGLayer
{
	FDrawHandle Handle = 0;
	FDrawHandle ParentHandle = 0;
	FIntRect Bounds;
	FIntPoint Origin = FIntPoint::ZeroValue;
	FRDGTextureRef Color = nullptr;
	FRDGTextureRef Resolve = nullptr;
	FRDGTextureRef Stencil = nullptr;
	uint32 StencilReference = 0;
	bool bEmpty = false;
	bool bResolveDirty = false;
};
```

这是设计形状，不进入通用协议，也不暴露给 Pipeline。Base Layer `0` 映射到 Slate 输出目标；
开启 MSAA 时，base layer 与普通 child layer 一样使用 color/resolve 对。

## 命令执行

Target 按原顺序遍历命令，维护自己的严格活动栈和 Layer 表：

1. `PushLayer`
   - 校验 `ParentLayer` 等于当前栈顶，且 `Layer` 尚不存在。
   - 为非空 Bounds 创建透明黑 color target 和独立 stencil。
   - MSAA 启用时同时创建 resolve target。
   - 将新 Layer 压栈。
2. Geometry / clip mask / shader draw
   - 校验命令的 `TargetLayer` 等于当前栈顶。
   - 把连续、写入同一 Layer 的 draw 聚合为一个 raster segment。
   - viewport 使用 Layer 本地尺寸，矩阵额外应用 `Origin` 对应的 clip adjustment。
3. `PopLayer`
   - 校验不能弹出 base layer，且命令的 layer/parent 与栈一致。
   - 结束当前 raster segment；需要作为 SRV 读取时先 resolve。
   - 弹栈，但保留 Layer 表中的纹理，供后续 composite/save 使用。
4. `CompositeLayers`
   - 结束所有相关 raster segment。
   - 读取 source 的 resolve/color SRV，写入 destination color target。
   - 使用 source/destination 的 Origin 将相交页面区域换算为各自局部 UV 和 viewport。
5. 帧结束
   - Target 再次要求活动栈严格为 `[0]`。
   - 最终 base resolve 合成到 Slate 输出。

Pipeline 的校验负责阻止非法通用帧提交，Target 的二次校验负责防御其他 command producer 或
损坏快照。Target 遇到结构错误时丢弃该帧并输出 Error，不能尝试猜测父 Layer。

## RDG 临时资源池

帧内 Layer 直接使用 `FRDGBuilder::CreateTexture`。RDG 已根据 descriptor、pass lifetime 和资源
依赖执行 transient allocation、aliasing 与底层 pooled resource 复用，因此 MarkupUI 不再维护
一份 `TMap<Size, FTextureRHIRef>` 式的手工池。

每个 Layer 的 descriptor 由以下字段决定：

- `Bounds.Size()`
- Slate 输出像素格式
- sample count
- color / resolve / depth-stencil usage flags

Color target 使用透明黑 clear value。Stencil 必须是每个 Layer 独立的资源，因为 Pop 后恢复父
Layer 时必须同时恢复父 Layer 的 stencil 内容和 reference；不同 Layer 不能只共享一个 stencil。

这种方式自动覆盖：

- Widget resize 和 DPI 导致的尺寸变化
- 一帧内不同 Bounds 的 Layer
- Widget / Target 销毁
- RHI 重建后的临时资源重新分配

## 跨帧 RenderTarget Pool

`SaveLayerAsTexture` 和 `SaveLayerAsMask` 不能跨 graph 引用裸 `FRDGTextureRef`。当前 texture 保存路径：

1. 将要保存的 Layer resolve 成单采样 texture。
2. 使用 RDG extraction 得到 `TRefCountPtr<IPooledRenderTarget>`。
3. 把 pooled target 放入 Slate Target 的持久 handle 表。
4. 同帧后续 draw 通过 handle 对应的 `FRDGTextureRef` 直接读取快照；后续帧通过
   `RegisterExternalTexture` 导入新的 RDG graph。
5. 收到对应 Release 或 Target 销毁时移除强引用，资源返回引擎 RenderTarget Pool。

持久表的 key 是通用 texture/filter handle；descriptor 与 Bounds/Origin 一并保存。Resize 或 DPI
变化不应无条件清空仍被 RmlUi 引用的保存纹理；只有语义要求重建或 handle 被释放时才回收。

## MSAA 与 Resolve

- Color target 的 sample count 使用 `FUIRenderingSettings`。
- Resolve target 始终单采样，供 composite、filter、save 和最终 Slate 合成读取。
- 一个 Layer 只有在写入后首次被读取时才 resolve；后续再次写入会重新标记 dirty。
- Stencil sample count必须与当前 color target 一致。
- Filter ping-pong texture 默认单采样，因为它们由采样 pass 产生，不需要再次进行几何 MSAA。

## 分阶段落地

1. 建立 Target 侧 Layer 表、严格校验和 raster segment 调度。
2. 实现 Push/Pop 的临时 color/resolve/stencil，并保持能力表暂不开放。
3. 实现无 Filter 的 Blend 与 Replace composite。
4. 完成像素回归后，Slate Target 才声明 `bLayerOperations = true`。
5. [已完成] 实现 texture save/extraction 持久表；mask save 保留为后续独立 filter 资源路径。

能力声明是提交契约。只完成资源分配而没有 composite 时，不能提前将
`bLayerOperations` 设为 `true`，否则 RmlUi 会选择一个 Target 实际无法完成的渲染路径。
