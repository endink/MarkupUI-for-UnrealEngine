# Tracing 使用指南

MarkupUI Tracing 使用显式采集会话和自动传播的 Context。支持 OpenTelemetry 的平台使用真实后端；其他平台自动使用 Null Backend，业务代码不需要条件编译。每个有效 Span 必须属于一个已经开始且尚未停止的会话。

```cpp
#include "Tracing.h"
#include "Tracing.h"

using namespace MarkupUI::Tracing;
```

## 基本用法

先开始采集会话，再显式创建 Root Span，最后结束 Root，并异步停止会话以取得不可变快照：

```cpp
const TSharedRef<ITrace, ESPMode::ThreadSafe> Trace =
    StartTrace();

{
    FSpan Load = NewRootSpan(Trace, TEXT("Document Load"));

    {
        FSpan Parse = NewSpan(TEXT("Parse"));
        // 执行解析。
    }
}

TFuture<TSharedRef<const FSnapshot, ESPMode::ThreadSafe>> SnapshotFuture =
    StopTraceAsync(Trace);

SnapshotFuture.Next(
    [](const TSharedRef<const FSnapshot, ESPMode::ThreadSafe>& Snapshot)
    {
        // 使用不可变快照。
    });
```

`StartTrace` 只创建采集会话，不创建 Span，也不修改当前 Context。使用真实后端时，`NewRootSpan(Trace, Name)` 创建 Root Span 并产生 TraceId。Root Span 的名称会显示为甘特图根步骤；创建 Root 后可通过 `Trace->GetId()` 读取 TraceId。`NewSpan(Name)` 只能创建当前活动 Span 的子节点；找不到 Parent Span 时会返回无效 Span并输出警告。

通常可能在活动 Trace 外执行的公共路径可以使用 `TryNewSpan`。它只在当前 Context 属于 Trace 时创建子 Span，否则静默返回无效 Span：

```cpp
FSpan Paint = TryNewSpan(TEXT("Paint"));
Paint.SetFrameNumber(FrameNumber);
```

Null Backend 下，Trace、Span、Context 和 Snapshot 操作安全地退化为空结果；`AsyncTraced`、`AsyncTracedTask` 和 `EnqueueRenderCommand` 仍会执行原始 UE 任务，只是不传播追踪 Context。可以通过 `IsAvailable()` 查询当前构建是否使用真实后端。

可以在已有活动 Trace 中开始另一个 Trace。新 Trace 不会成为外层 Trace 的子级；停止内层 Trace 后，外层 Context 会恢复：

```cpp
const TSharedRef<ITrace, ESPMode::ThreadSafe> Outer = StartTrace();
FSpan OuterRoot = NewRootSpan(Outer, TEXT("Outer"));

const TSharedRef<ITrace, ESPMode::ThreadSafe> Inner = StartTrace();
FSpan InnerRoot = NewRootSpan(Inner, TEXT("Inner"));

InnerRoot.End();
StopTraceAsync(Inner);
FSpan OuterWork = NewSpan(TEXT("Outer Work"));
OuterWork.End();
OuterRoot.End();
StopTraceAsync(Outer);
```

`FSpan` 创建后自动成为当前 Span，离开作用域时自动结束并恢复此前 Context。需要提前结束时可以调用 `End`；重复调用是安全的：

```cpp
FSpan Load = NewRootSpan(Trace, TEXT("Load"));
// ...
Load.End();
```

长时间记录或作为成员变量跨帧保存时，可以在不结束 Span 的情况下解除和恢复当前 Context：

```cpp
// 开始记录，并默认成为当前 Span。
LoadSpan = NewRootSpan(Trace, TEXT("Load"));

// 暂时离开 Load Context；Load 仍继续计时。
LoadSpan.Deactivate();

// 后续调用或后续帧中重新成为当前 Span。
LoadSpan.Activate();
FSpan Paint = NewSpan(TEXT("Paint"));
Paint.End();
LoadSpan.Deactivate();

// 完成整个阶段。
LoadSpan.End();
```

`Activate` 和 `Deactivate` 可以重复调用。已经结束的 Span 不会再次激活。激活状态遵循栈式 Context 顺序：应先结束或解除当前子 Span，再解除它的父 Span。

`StopTraceAsync` 是硬截止点。它立即关闭 Trace 的写入，并返回一个在后台处理完成后兑现的 Future：

- 返回停止前已经完整结束的 Span。
- 丢弃尚未结束的 Span。
- 如果一个已结束 Span 的父级没有完整结束，该残缺子树同样被丢弃。
- 停止后才执行的异步工作不会继续向该 Trace 写入 Span。
- 不同 Trace 相互独立；停止一个 Trace 不会消费另一个 Trace 的数据。

## Root 与父级

Root 必须显式创建：

```cpp
FSpan Root = NewRootSpan(Trace, TEXT("Frame"));
```

普通 Span 必须存在当前 Parent Span：

```text
Frame (Root)
`-- Paint
```

```cpp
FSpan Paint = NewSpan(TEXT("Paint"));
```

显式 `Parent` 只接受同一 Trace 的 Context：

```cpp
FSpanOptions Options;
Options.Parent = ParentContext;

FSpan Child = NewSpan(TEXT("Child"), Options);
```

跨 Trace 的因果关系应使用 Span Link。

## 异步任务

`AsyncTracedTask`、`AsyncTraced` 和 `EnqueueRenderCommand` 捕获调用点的当前 Context，并在任务实际执行期间临时激活。线程池复用不会改变 Trace 归属。

```cpp
const TSharedRef<ITrace, ESPMode::ThreadSafe> Trace =
    StartTrace();

FSpan Load = NewRootSpan(Trace, TEXT("Image Load"));

TFuture<FDecodeResult> Future = AsyncTraced(
    EAsyncExecution::ThreadPool,
    []
    {
        FSpan Decode = NewSpan(TEXT("Decode"));
        return DecodeImage();
    });

const FDecodeResult Result = Future.Get();
Load.End();
TFuture<TSharedRef<const FSnapshot, ESPMode::ThreadSafe>> SnapshotFuture =
    StopTraceAsync(Trace);
```

包装函数只传播 Context，不会自动创建 Span，也不会延长父 Span 的持续时间。调用 `StopTraceAsync` 前是否等待异步任务由调用者决定；未完成的记录会被丢弃。

直接使用原生 `Async`、`AsyncTask` 或 `ENQUEUE_RENDER_COMMAND` 不保证传播 MarkupUI Tracing Context。

### 渲染线程命令

```cpp
FSpan Prepare = NewSpan(TEXT("Prepare Upload"));

EnqueueRenderCommand(
    [Texture](FRHICommandListImmediate& RHICmdList)
    {
        FSpan Upload = NewSpan(TEXT("Upload Texture"));
        UploadTexture(RHICmdList, Texture);
    });
```

## 线程信息

每个 Span 创建时都会按实际执行线程记录线程信息：

- `Game`
- `Slate`
- `Render`
- `RHI`
- `ThreadPool`
- `Background`
- `IO`

需要强制标准线程类别时：

```cpp
FSpanOptions Options;
Options.ThreadKind = EThreadKind::Io;

FSpan Read = NewSpan(TEXT("Read Resource"), Options);
```

`ThreadKind` 默认为 `Auto`，只影响当前 Span；`thread.id` 始终来自实际执行线程。

## 类型化属性

Trace Root 的公共属性可在开始时提供，后续 Span 自动继承文档、资源、Runtime 和帧信息：

```cpp
const TArray<FAttribute> Attributes =
{
    FAttribute(TEXT("document.path"), DocumentPath),
    FAttribute(TEXT("runtime.id"), RuntimeId),
    FAttribute(TEXT("frame.number"), FrameNumber),
};

const TSharedRef<ITrace, ESPMode::ThreadSafe> Trace =
    StartTrace(Attributes);

FSpan Frame = NewRootSpan(Trace, TEXT("Frame"));
```

Span 常用属性优先使用类型化接口：

```cpp
FSpan Decode = NewSpan(TEXT("Decode Image"));

Decode.SetThreadRole(EThreadRole::Worker);
Decode.SetThreadName(TEXT("Image Decoder"));
Decode.SetDocumentPath(DocumentPath);
Decode.SetResourcePath(ResourcePath);
Decode.SetRuntimeId(RuntimeId);
Decode.SetFrameNumber(FrameNumber);
Decode.SetUploadBytes(UploadBytes);
Decode.AddUploadBytes(AdditionalUploadBytes);
Decode.SetVertexCount(VertexCount);
```

其他属性使用 `SetAttribute` 或 `FSpanOptions::Attributes`。

## 事件和状态

```cpp
FSpan Load = NewSpan(TEXT("Load"));

Load.AddEvent(
    TEXT("Response Received"),
    {
        FAttribute(TEXT("http.status_code"), 200),
        FAttribute(TEXT("response.bytes"), ResponseBytes),
    });

Load.SetStatus(bSucceeded ? EStatus::Ok : EStatus::Error, ErrorMessage);
```

## Span Links

共享操作由多个请求共同触发时使用 Link：

```cpp
FSpanOptions Options;
Options.Links = {DocumentAContext, DocumentBContext};

FSpan SharedLoad = NewSpan(TEXT("Shared Resource Load"), Options);
```

## 读取快照

`StopTraceAsync` 返回 Future；Future 完成后的 Snapshot 可以长期持有和重复读取，但同一 Trace 只能停止和消费一次。不要在 Game Thread 或 Render Thread 上阻塞等待 Future，应通过 `Next` 处理结果：

```cpp
StopTraceAsync(Trace).Next(
    [](const TSharedRef<const FSnapshot, ESPMode::ThreadSafe>& Snapshot)
    {
        for (int32 Index = 0; Index < Snapshot->GetSpanCount(); ++Index)
        {
            const FSpanSnapshot Span = Snapshot->GetSpan(Index);
            const FString Name = Span.GetName();
            const FTimespan Duration = Span.GetDuration();
        }
    });
```

类型化属性有对应读取接口，例如 `GetThreadRole`、`GetResourcePath`、`GetRuntimeId`、`GetUploadBytes` 和 `GetVertexCount`。

当前随插件提供的 OpenTelemetry 静态库仅支持 Win64。OTLP、Collector、Jaeger 和 Grafana Exporter尚未作为默认工作流启用。
