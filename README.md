[中文](./README_cn.md)
<br>
<br>
<p align="center">
  <img alt="MarkupUI" src="./Logo.svg" width="152">
</p>
<p align="center">
  An HTML and CSS-style UI plugin for Unreal Engine.
  <br>
  <br>
  <a href="#overview"><img alt="Unreal Engine UI" src="https://img.shields.io/badge/Unreal%20Engine-1C2A45"></a>
</p>
<p align="center">
  <a href="#overview">Overview</a>
  | <a href="#documentation">Documentation</a>
  | <a href="#roadmap">Roadmap</a>
</p>
<br>

![MarkupUI preview](preview.jpg)

# Overview

**MarkupUI** brings HTML and CSS-style UI authoring to Unreal Engine. It is designed for UI that benefits from readable document structure, reusable styles, and a clear separation between interface content and presentation.

Build interfaces with markup and stylesheets, then keep them close to the Unreal workflow your project already uses.

## Highlights

- **HTML/CSS-style authoring** — Use markup, selectors, cascading styles, variables, Flex layout, transitions, animations, and transforms to describe UI.
- **Native Unreal workflow** — Work with RML documents and RCSS stylesheets as Unreal assets, with standard import and reimport support.
- **Slate integration** — Use MarkupUI as part of an Unreal UI, with mouse, keyboard, text input, focus, and cursor behavior available to authored interfaces.
- **Flexible resources** — Load documents, styles, fonts, and images from Unreal assets or a disk-based content layout.
- **Visual foundation** — Support practical UI rendering features such as clipping, transforms, compositing, filters, and saved visual resources, while documenting any unsupported visual syntax clearly.

## Documentation

- [Resource paths and root directory](Usage/Resource-Paths.md)
- [Using Unreal asset fonts](Usage/Unreal-Asset-Fonts.md)
- [RCSS rendering support and roadmap](Development/RCSS-Rendering-Support-Roadmap.md)
- [Data binding support roadmap](Development/DataBinding-Support-Roadmap.md)
- [Rendering performance diagnostics](Development/Rendering-Performance.md)
- [Rendering stats and profiling](Development/Rendering-Stat-Profiling.md)

## Roadmap

MarkupUI is focused on making advanced RCSS visuals dependable in Unreal projects. Current work continues across mask composition, box shadows, gradients, custom shaders, visual regression coverage, and performance measurement.

The project favors accurate, explicit behavior: features that are not ready for reliable output should be documented as unavailable instead of silently producing a different result. See the [RCSS rendering support and roadmap](Development/RCSS-Rendering-Support-Roadmap.md) for the current scope and direction.

## Acknowledgements

MarkupUI is built with [RmlUi](https://github.com/mikke89/RmlUi). Thank you to its community for creating a lightweight and expressive C++ UI middleware.
