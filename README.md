# MarkupUI for Unreal Engine

> 让 RML/RCSS 的表达力，成为 Unreal Engine 原生 UI 工作流的一部分。

MarkupUI 是一个面向 Unreal Engine 的运行时与编辑器插件，将 [RmlUi](https://github.com/mikke89/RmlUi) 的 HTML/CSS 风格界面系统带入 Unreal，并以原生 Slate 为落点完成绘制、输入和资源协作。它希望为游戏 UI 提供一种更接近“文档 + 样式”的创作方式：界面结构清晰、样式可复用、交互语义明确，同时保有 Unreal 工程所需要的性能、资源生命周期与渲染可靠性。

项目当前以 RmlUi 6.3 为基础，重点不是简单地把 Web 语法嵌入引擎，而是在 Unreal 的渲染与资产体系中建立一条可验证、可演进的 UI 管线。

## 为什么是 MarkupUI

传统游戏 UI 往往在布局、视觉样式与交互逻辑之间不断来回穿梭。MarkupUI 选择以 RML 描述界面结构、以 RCSS 表达视觉规则，并将它们接入 Unreal 的 Slate、资源和编辑器工作流。

这使 UI 更接近一份可阅读、可维护的界面文档：

- 用 RML 组织层级、文本、图片和交互元素；
- 用 RCSS 管理选择器、层叠、变量、布局、动画与变换；
- 用 Slate 承接 Unreal 原生的绘制、窗口、输入与焦点行为；
- 用 Unreal 资产与磁盘资源共同服务于内容制作和迭代。

MarkupUI 的目标不是替代 Unreal 的所有 UI 技术，而是为需要高表达力、可维护样式和文档化工作流的界面，提供一条扎实的原生路径。

## 当前能力

### RML / RCSS 驱动的界面

- 基于 RmlUi 6.3，支持 RCSS 的选择器、层叠、变量和常规属性；
- 支持 Flex 布局、transition、animation 与 2D/透视 transform；
- 支持普通几何、文本、图片与矩形裁剪；
- 支持 stencil `Set`、`SetInverse` 与 `Intersect` 裁剪语义。

### 与 Unreal 的原生协作

- 以 `SRmlUiWidget` 融入 Slate 绘制体系；
- 转发鼠标、滚轮、键盘、文本输入、焦点与鼠标光标行为；
- 支持 `.rml` 文档和 `.rcss` 样式表导入为 Unreal 资产，并遵循标准的 Source File / Reimport 契约；
- 支持 `URmlDocument`、`URmlStyleSheet`、`UFontFace` 与 `UTexture2D` 资源；
- 支持磁盘中的文档、样式、字体与 PNG/JPEG 等图片资源。

### 为正确性而生的渲染管线

- 使用预乘 Alpha 混合，避免半透明边缘出现暗边；
- 保留完整 4×4 GPU 变换路径，支持透视正确插值；
- 通过通用的 typed draw command 协议隔离 RmlUi 前端、渲染管线与具体 Target；
- 已实现局部 RDG Layer、`Blend` / `Replace` 合成和 `opacity` filter；
- 可将 Layer 保存为同帧或跨帧可采样纹理，并安全管理异步渲染与资源释放；
- 将 Target 能力与失败契约前置：尚未真正实现的效果会明确拒绝，而不是静默降级为错误的画面。

## 架构理念

MarkupUI 把“界面描述”和“像素执行”分开处理：RmlUi 负责解析、布局与生成绘制意图；通用绘制帧负责保存稳定的命令与资源快照；Target 负责在具体平台上重放它们。当前默认 Target 是 Slate，但协议并不依赖 Slate 或 RmlUi 类型。

```text
RML + RCSS
    ↓
RmlUi Frontend
    ↓
通用 Draw Frame / 命令流
    ↓
Slate Target + RDG
    ↓
Unreal Engine UI
```

这种边界让高级视觉能力可以先成为通用协议的一部分，再由不同 Target 逐步实现；也让错误隔离、跨线程资源生命周期和未来的多 Target 支持有清晰的落点。

## 进展与边界

基础 RML/RCSS 解析、布局与常规绘制已具备较高完成度，当前工作的重心是补齐 RmlUi 内置 RCSS 的高级视觉语义。

已可用的高级路径包括局部 Layer、普通/覆盖合成、`opacity` filter，以及保存 Layer 为可复用纹理。以下能力的描述和调度基础已经进入通用管线，但 Slate Target 尚未提供完整的真实像素执行：

- 颜色矩阵 filter、blur、drop-shadow 与 backdrop-filter；
- `mask-image` 与保存 Layer 为 mask；
- 带模糊的 box-shadow；
- linear / radial / conic（及 repeating）gradient；
- 自定义 shader decorator。

这不是功能缺席时的沉默降级。MarkupUI 将“声明支持”视作渲染契约：只有当前 Target 确实能正确执行的效果才会被启用。

## Roadmap

接下来的工作将沿着一条清晰的主线推进：

1. **完成高级像素能力**：颜色 filter、blur、drop-shadow、backdrop-filter、mask 与完整 box-shadow。
2. **实现内置渐变与可扩展 Shader**：覆盖 RmlUi 内置 gradient，并提供受控的 Unreal Shader 注册机制。
3. **完善资源路径与生命周期**：增强插件内容资源、相对路径、热更新和 RHI 重建场景下的可靠性。
4. **建立视觉回归基线**：以截图与像素对比覆盖裁剪、变换、Layer、Filter、Shadow、Mask 与 Gradient。
5. **在正确性稳定后持续优化**：以真实指标驱动连续几何批处理、GPU 资源复用和性能分析，绝不以改变绘制顺序换取表面上的性能数字。

完整的技术路线与完成标准请阅读：[RCSS 渲染支持完成度与开发路线](RCSS-Rendering-Support-Roadmap.md)。

## 文档

- [资源路径与根目录](Usage/Resource-Paths.md)
- [使用 Unreal 资产字体](Usage/Unreal-Asset-Fonts.md)
- [RCSS 渲染支持完成度与开发路线](RCSS-Rendering-Support-Roadmap.md)
- [渲染性能诊断与优化路线](Rendering-Performance.md)
- [Slate RDG Layer 执行与资源复用设计](Slate-RDG-Layer-Design.md)
- [色彩空间与 Alpha 契约](Color-Space-and-Alpha-Contract.md)
- [设计讨论归档](Discussions/README.md)

## 致谢

MarkupUI 建立在优秀的开源 UI 中间件 [RmlUi](https://github.com/mikke89/RmlUi) 之上。感谢 RmlUi 社区为轻量、高表达力的 C++ UI 系统所做的长期投入。

---

MarkupUI 正在把 Web 风格的界面表达，转化为 Unreal 中可控、可验证、可长期维护的生产力。
