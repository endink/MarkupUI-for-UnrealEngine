# RHI 资源生命周期开发设计

> 本文是 MarkupUI 渲染后端的开发设计记录，用于实现、测试和代码审查时核对资源失效顺序。
> 它不是插件使用指南，也不承诺所有平台都支持 GPU 设备丢失后的进程内恢复。

## 目标

当 Unreal Engine 完整释放并重新初始化 RHI 时，MarkupUI 必须满足以下约束：

- 已失效的 GPU backing 不再被任何新旧绘制访问。
- 已经提交但尚未执行的旧 Frame 可以安全丢弃，不要求恢复其原始内容。
- 持有 GPU backing 的资源各自释放自身引用，不向 Pipeline 逐个发送回调。
- Target 对外发布一个统一的资源状态和代次。
- Pipeline 只协调 Frontend 与 Target，不依赖 Unreal RHI 类型。
- Frontend 在恢复确认后重新生成字体 atlas、生成纹理、Saved Texture 和 Saved Mask。
- 普通 resize、DPI、窗口化和交换链调整不推进资源代次。

## 职责边界

### 单个 GPU 资源

每个插件私有 GPU 资源只负责释放自己的 backing：

- Frontend 生成纹理释放其 `FTextureRHIRef`。
- Saved Texture 和 Saved Mask 释放其 `IPooledRenderTarget`。
- Unreal `UTexture` 继续由 Unreal 自身的纹理资源生命周期管理。
- RDG 临时 Layer 是帧内资源，不参与跨 RHI 恢复。

资源释放时不调用 Frontend、不修改 Pipeline，也不发布独立的恢复事件。

### Unreal RHI 资源状态

Unreal 后端维护唯一的进程级 RHI 资源 token。token 同时编码：

- 当前资源 Generation；
- RHI 是否已经完成自身的初始化回调。

全局状态通过一个 `FRenderResource` 接收 `ReleaseRHI()` 和 `InitRHI()`。插件 GPU 资源以弱引用监听者
登记到该全局资源，不再分别继承 `FRenderResource`。`ReleaseRHI()` 先推进 Generation 并令状态不可用，
再按监听表让仍然存活的资源释放各自 backing；`InitRHI()` 只表示 RHI
初始化流程已经到达该资源，不直接允许 Target 立即恢复绘制。

### Target

Target 将 Unreal 后端的全局 token 转换为通用资源生命周期状态：

- `Unavailable`：RHI 尚不可用；
- `Recovering`：RHI 已重新初始化，但 Target 的 RenderThread barrier 尚未确认；
- `Ready`：barrier 已完成，可以清理旧缓存并生成新 Frame。

Target 负责：

- 请求非阻塞 RenderThread 恢复 barrier；
- 清理 Target 私有的 Persistent Texture、Saved Texture 和 Saved Mask 缓存；
- 将当前资源 token 捕获到提交给 Slate 的绘制元素中。

### Pipeline

Pipeline 通过 Target 的可选资源生命周期支持接口读取状态，不直接包含 `FRenderResource`、RHI
引用或 RenderThread 代码。

Pipeline 负责：

- `Unavailable` 时停止产生 Frame；
- `Recovering` 时请求 barrier 和下一次更新；
- 新 Generation 首次变为 `Ready` 时，通知 Frontend 释放语言层资源缓存并要求 Target 清理旧缓存；
- 跳过恢复确认帧，下一帧才提交新 Generation 的完整命令与资源快照。

### Frontend

Frontend 实现统一的渲染资源失效入口。RmlUi Frontend 在该入口中请求当前 Context 释放纹理，
后续 `Update()` / `Render()` 自然重新生成所需资源。

该入口不负责 Unreal GPU backing，也不理解 Target 的资源实现。

## 顺序保证

不能依赖 `FRenderResource` 的注册顺序。UE 可能逆序释放、正序初始化资源，但动态资源注册时机会随
模块加载、Widget 创建和平台配置变化。

### ReleaseRHI

无论全局状态资源在其他资源之前还是之后收到 `ReleaseRHI()`，RenderThread 都会串行执行这些回调。
GameThread 在中间窗口提交的 Frame 仍携带旧 token，并在真正访问纹理、RDG Layer 或 Saved backing
之前被 RenderThread 拒绝。

### InitRHI

全局状态资源可能早于其他资源收到 `InitRHI()`。因此 Target 不把该回调直接解释为 `Ready`，而是
排入一个 RenderThread barrier。barrier 只能在当前 RHI 初始化工作之后执行，完成时再次核对
Generation 和初始化状态，然后确认 Target 的 Ready Generation。

Pipeline 不阻塞 GameThread 等待 barrier；它请求下一次更新并在后续帧重新查询状态。

## Frame 拒绝规则

Slate 绘制元素在提交时捕获完整资源 token。RenderThread 执行入口必须先完成以下检查：

1. 当前 RHI 状态可用；
2. Frame token 与当前 token 完全一致；
3. 命令和资源快照有效。

前两项失败时整帧安静跳过，不进入 RDG，不报告文档渲染错误，也不尝试使用部分仍然可用的资源。
Frame 持有的强引用继续保证 C++ 对象活到 RenderThread 消费或丢弃完成，但强引用不代表 GPU backing
仍然有效。

## 恢复状态序列

```text
Ready, Generation N
  -> ReleaseRHI
Unavailable, Generation N + 1
  -> InitRHI
Recovering, Generation N + 1
  -> RenderThread barrier
Ready, Generation N + 1
  -> Pipeline 令 Frontend 和 Target 失效旧缓存
  -> 跳过当前恢复帧
  -> 下一帧重新生成并提交资源
```

如果 barrier 执行前再次发生 `ReleaseRHI()`，barrier 捕获的旧 Generation 不得确认成功；Target 必须
重新等待最新 Generation。

## 非触发事件

以下变化继续走现有 viewport 和布局更新路径，不触发 RHI 资源恢复：

- Widget resize；
- DPI 或 pixel ratio 改变；
- 窗口移动、裁剪和最小化；
- 普通窗口化、全屏切换；
- 仅重建 swapchain 或 back buffer。

只有 Unreal 调用完整的 `FRenderResource` RHI 释放与初始化周期时，资源 Generation 才会推进。

## 验证要求

自动化测试至少覆盖：

- 初始 Ready Generation 不触发无意义的 Frontend 重建；
- Unavailable 状态不调用 Frontend Render；
- Recovering 状态只请求 barrier，不提交 Frame；
- 新 Generation Ready 后，Frontend 和 Target 各失效一次；
- 恢复确认帧不提交，下一帧才恢复绘制；
- 旧 token 的在途 Slate 绘制在进入 RDG 前被拒绝；
- 重复读取同一 Generation 不重复清缓存或重建 Frontend。
