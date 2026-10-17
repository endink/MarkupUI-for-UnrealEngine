# Tracing 使用指南

在 HTML 文档资产编辑器的 Tracing 面板中，可以采集页面帧耗时、查看时间线并比较不同预览分辨率下的结果。

## 采集

1. 打开 HTML 文档资产，并确认预览能够显示页面。
2. 在 Tracing 面板选择采样时长和预览模式；非当前编辑器模式可以选择 Resolution。
3. 点击 **Sample** 开始采集；点击 **Cancel** 可以取消当前采集。
4. 采集结束后选择帧或汇总视图，检查时间线和耗时统计。

**Fit** 调整时间线显示范围；**Collapse All**、**Expand All** 控制层级显示；选中条目后可使用 **Focus**。

## CPU Profile

勾选 **CPU Profile** 后，点击 **Sample** 会同时录制 Unreal Insights CPU 时序，无需手动输入控制台命令。采样结束或点击 **Cancel** 时，CPU 录制也会停止。

录制文件与 SDK 的 `.jsonl` 文件位于同一目录，文件主名相同，扩展名为 `.utrace`。使用 Unreal Insights 打开 `.utrace`，查看 CPU、帧、任务、渲染命令和 RHI 命令时序，重点比较异常帧与附近正常帧。

**Unreal Insights** 标签页与 Timeline、Statistics 并列。面板刚打开时列表为空，点击 **Refresh** 后异步列出采样目录中的 `.utrace` 文件，按创建时间从新到旧排列，每批显示 50 条，每次查询 51 条判断是否还有更多；有更多时列表最后一行显示 **Load more** 按钮，点击后追加下一批 50 条。文件路径前的绿点表示本次同步采样文件；路径使用小字体，过长时显示省略号，悬停可查看完整路径；**Copy** 复制完整路径，**Open** 使用当前 UE 的 Unreal Insights 打开文件。

刷新期间左侧面板显示加载遮罩，底下的控件暂时不可操作；采样期间继续显示原有进度界面，不提供刷新入口。采样完成或取消后自动刷新并回到第一页。

如果已有其他 Unreal Insights 录制，CPU Profile 会报告冲突并取消本次采样；它不会接管现有录制。录制期间会启用 Named Events，结束后恢复原来的设置。CPU 时序不能直接代表 GPU 执行耗时。

## 结果解释

Mean 和 P90 表示所采样帧的平均耗时与第 90 百分位耗时。Estimated FPS 是根据采样耗时计算的估计值，不能直接当作完整游戏的帧率。

比较结果时，应使用相同文档、分辨率、资源状态和运行环境。首次加载与后续稳定帧可能有明显差异，应分别观察。

MarkupUI Settings 中的 Tracing 默认关闭；编辑器采样可以临时为当前文档启用追踪。**Rebuild Cache** 当前不可用。
