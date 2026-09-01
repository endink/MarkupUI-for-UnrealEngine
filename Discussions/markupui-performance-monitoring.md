# MarkupUI Performance Monitoring

## Document Purpose

This is a temporary design decision for AI-assisted implementation planning.

It intentionally contains no source paths, code, API signatures, type names, function names, or details of the current implementation.

## Status

Approved direction for future work. No performance-monitoring implementation exists yet.

## Problem

MarkupUI needs performance monitoring that:

- uses Unreal Engine's native diagnostic tooling;
- provides current-frame diagnostics and time-series investigation;
- does not require a custom in-game graph widget;
- has minimal overhead when monitoring is disabled;
- distinguishes CPU work, render-thread work, rendering complexity, and actual GPU execution.

## Chosen Solution

Use two native Unreal Engine reporting surfaces backed by one shared set of instrumentation data:

1. A plugin-specific runtime stats group named `MarkupUI`.
2. Unreal Insights timing events and numeric counters.

Do not implement a custom chart.

## User Workflows

### Fast runtime diagnosis

1. Run a development-capable build.
2. Enable the `MarkupUI` runtime stats group.
3. Inspect current CPU timings and rendering-complexity counters.
4. Identify whether the immediate issue is update/layout work, command building, render-thread work, or unusually high rendering complexity.

### Time-series and spike diagnosis

1. Start or connect an Unreal Insights trace session.
2. Reproduce the slow interaction, screen, or document.
3. Inspect the plugin's CPU and render-thread timing events.
4. Add relevant numeric counters to an Unreal Insights Graph Track.
5. Correlate spikes in timing, draw complexity, layers, clipping, and resource work.

## Metrics

### Required timings

| Metric | Definition |
|---|---|
| `Update` | CPU time spent updating markup state, style, and layout. |
| `BuildDrawCommands` | CPU time spent traversing markup for rendering and producing draw commands. |
| `SlatePrepare` | CPU time spent preparing the UE UI rendering submission. |
| `RenderThread` | CPU time spent executing MarkupUI rendering work on the render thread. |
| `ResourceUpload` | CPU time spent creating or uploading UI resources, when applicable. |

### Required per-frame counters

| Metric | Definition |
|---|---|
| `Contexts` | Number of active MarkupUI contexts. |
| `GeometryCompiles` | Geometry compilation events in the frame. |
| `Vertices` | Vertices used by newly compiled geometry in the frame. |
| `Indices` | Indices used by newly compiled geometry in the frame. |
| `GeometryDraws` | Geometry draw submissions in the frame. |
| `ShaderDraws` | Shader draw submissions in the frame. |
| `ClipMaskOps` | Clip-mask operations in the frame. |
| `LayerPushes` | Offscreen-layer creation or push operations in the frame. |
| `LayerComposites` | Layer composition operations in the frame. |
| `GeneratedTextureBytes` | Byte count of dynamically generated textures in the frame. |

### Later, optional metrics

- Texture, font, filter, and shader cache hit/miss counts.
- Persistent CPU and GPU memory attributed to MarkupUI resources.
- Per-document or per-widget metrics, limited to a bounded top-N list.
- Markup element count and layout-dirty count, only if their semantics are reliable and stable.

## Measurement Rules

1. Each frame counter must define whether it is a delta, current value, or lifetime total.
2. Use aggregate plugin values by default. Per-context names must be bounded to avoid unbounded telemetry cardinality.
3. Collection must add negligible cost when the corresponding Unreal stats or tracing channel is disabled.
4. Resource allocation/upload cost must be reported separately from normal frame rendering cost.
5. Do not label CPU command generation, draw counts, or vertex counts as GPU time.
6. Actual GPU execution time must come from Unreal Engine GPU profiling data, not from plugin-side CPU instrumentation.

## C ABI Decision

### First version

Do not add or change the portable C ABI.

Reasoning:

- The UE host can measure the relevant high-level frame phases itself.
- The UE host can already observe renderer-facing activity needed for the required complexity counters.
- Adding portable ABI surface now would increase compatibility and maintenance cost without unlocking the UE monitoring feature.

### Future cross-host capability

Consider a portable API only when non-UE hosts need a standard way to retrieve the same metrics.

The preferred shape is a versioned, pull-based, last-completed-frame snapshot. It must:

- include structure size and version fields;
- define frame ownership, reset behavior, and accumulation behavior;
- expose only host-neutral metrics;
- remain separate from Unreal-specific tooling;
- omit GPU duration unless actual GPU completion can be measured.

Avoid a high-frequency profiling callback API unless a future host has a demonstrated need for it.

## Non-goals

- No custom runtime chart, graph widget, HUD, or overlay.
- No modification of Unreal Engine's built-in unit graph.
- No promise that plugin counters measure actual GPU execution duration.
- No portable C ABI change in the first implementation.

## Acceptance Criteria

The work is complete when all of the following are true:

1. The `MarkupUI` runtime stats group shows the required current-frame timings and counters.
2. The same timing and counter data is visible in Unreal Insights.
3. At least one numeric metric can be displayed in an Unreal Insights Graph Track.
4. A known expensive UI case produces an identifiable increase in the relevant timing and complexity metrics.
5. Disabled monitoring has no material impact on normal MarkupUI behavior or performance.
6. Documentation explains the user workflows without requiring a custom graph UI.

## Implementation Order When Approved

1. Add aggregate timing instrumentation.
2. Add aggregate per-frame complexity counters.
3. Publish all required data to runtime stats and Unreal Insights.
4. Add render-thread and resource-work separation.
5. Validate against intentionally expensive UI scenarios.
6. Add optional cache, memory, and bounded per-context metrics only when they answer a real diagnostic question.

## References

- [Unreal Engine Stats System Overview](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-stats-system-overview)
- [Stat Commands in Unreal Engine](https://dev.epicgames.com/documentation/unreal-engine/stat-commands-in-unreal-engine)
- [Using the Timers and Counters Tabs in Unreal Insights](https://dev.epicgames.com/documentation/unreal-engine/using-the-timers-and-counters-tabs-in-unreal-insights-for-unreal-engine)
