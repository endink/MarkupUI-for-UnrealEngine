# 使用 Unreal 资产字体

本文说明当 MarkupUI 文档以 Unreal Asset 作为资源来源时，如何导入字体并在 RCSS 中通过 `@font-face` 使用它。

## 应该使用哪一种 Unreal 字体资产

MarkupUI 使用 `UFontFace`，而不是 `UFont`。

Unreal Editor 导入一个 `.ttf` 或 `.otf` 文件时，可能同时生成两类资产：

- **Font Face（`UFontFace`）**：保存字体文件的原始字节与单个字体面的数据；
- **Font（`UFont`）**：供 Unreal、Slate 或 UMG 使用的字体配置资产，可以组合多个 Font Face。

MarkupUI 使用 Font Face 中保存的字体数据生成页面文字。`UFont` 是 Unreal 字体系统的配置资产，不能代替 RCSS 中声明的具体字体面。

结论：

- 用于 MarkupUI/RCSS 的字体必须保留对应的 **Font Face** 资产；
- 自动生成的 **Font** 资产不是 MarkupUI 的必需资源；
- 如果该 Font 没有被 Slate、UMG 或其他 Unreal 功能使用，可以将其删除；
- 一个 `UFont` 可以引用多个 `UFontFace`，但 MarkupUI 不会从 `UFont` 中猜测应该使用哪一个字体面。

## 资产名称与 RCSS 路径

假设项目中存在以下 Font Face 资产：

```text
/Game/Assets/MarkupUI/DingTalkJinBu.DingTalkJinBu
```

当 RCSS 与字体使用相同的 Unreal 资源根目录时，可以按相对路径声明：

```css
@font-face {
    font-family: "DingTalk JinBu Test";
    src: "DingTalkJinBu.ttf";
}

body {
    font-family: "DingTalk JinBu Test";
}
```

这里的 `DingTalkJinBu.ttf` 是 RCSS 字体资源 URI，并不表示运行时会读取磁盘中的同名文件。在 Unreal Asset 资源模式下，它表示相对于当前 RCSS 资产查找同名 Font Face。上面的相对 URI 对应：

```text
/Game/Assets/MarkupUI/DingTalkJinBu.DingTalkJinBu
```

也可以使用相对于 `AssetResourceRootDirectory` 的根路径。假设资产资源根目录配置为：

```text
/Game/Assets/MarkupUI
```

则可以这样声明：

```css
@font-face {
    font-family: "DingTalk JinBu Test";
    src: "/DingTalkJinBu.ttf";
}
```

`/` 表示配置的 MarkupUI 资产资源根目录，不表示 Unreal 的全局 `/Game` 路径。完整规则请参阅[资源路径与根目录](Resource-Paths.md)。

建议让 Font Face 的资产名与源字体文件名一致。这样 RCSS 可以继续使用熟悉的 `.ttf` 或 `.otf` URI，同时稳定地映射到同名 Unreal 资产。

## 一个字体文件与粗体、斜体

很多浏览器页面只声明一个 `.ttf`，随后仍可以使用 `font-weight: bold` 或 `font-style: italic`。浏览器可能在找不到对应字体面时合成粗体或斜体，也可能正在使用包含多个变化轴的可变字体。

不要默认 MarkupUI/RmlUi 会完全复制浏览器的字体合成行为：

- 同一 `font-style` 下缺少指定字重时，RmlUi 可以选择最接近的已加载字重，但字形不一定真的变粗；
- 缺少 `italic` 字体面时，不应假设一定会自动将普通字体倾斜；
- 可变字体可能在一个文件中包含多个字重，但实际结果取决于字体数据以及当前 RmlUi/FreeType 对变化实例的识别；
- 需要与设计稿或 RmlUi Viewer 稳定一致时，应显式提供实际使用的字体面。

只使用一个普通字体面的最小配置仍然有效：

```css
@font-face {
    font-family: "MyFont";
    src: "MyFont-Regular.ttf";
    font-weight: 400;
    font-style: normal;
}

body {
    font-family: "MyFont";
    font-weight: 400;
    font-style: normal;
}
```

但如果页面确实使用粗体和斜体，推荐分别导入对应的 Font Face 资产，并逐一声明：

```css
@font-face {
    font-family: "MyFont";
    src: "MyFont-Regular.ttf";
    font-weight: 400;
    font-style: normal;
}

@font-face {
    font-family: "MyFont";
    src: "MyFont-Bold.ttf";
    font-weight: 700;
    font-style: normal;
}

@font-face {
    font-family: "MyFont";
    src: "MyFont-Italic.ttf";
    font-weight: 400;
    font-style: italic;
}

@font-face {
    font-family: "MyFont";
    src: "MyFont-BoldItalic.ttf";
    font-weight: 700;
    font-style: italic;
}

body {
    font-family: "MyFont";
}

strong {
    font-weight: 700;
}

em {
    font-style: italic;
}
```

对应的 Unreal 资产可以组织为：

```text
/Game/UI/Fonts/MyFont-Regular.MyFont-Regular
/Game/UI/Fonts/MyFont-Bold.MyFont-Bold
/Game/UI/Fonts/MyFont-Italic.MyFont-Italic
/Game/UI/Fonts/MyFont-BoldItalic.MyFont-BoldItalic
```

如果 RCSS 不在该目录中，则使用相对于 RCSS 的路径，或者使用相对于 `AssetResourceRootDirectory` 的根路径，例如 `/Fonts/MyFont-Regular.ttf`。

## `UFont` 为什么不能直接替代 `UFontFace`

一个 `UFont` 可以通过 Composite Font 配置引用多个字体面，例如：

```text
UFont
├── Regular  → MyFont-Regular (UFontFace)
├── Bold     → MyFont-Bold (UFontFace)
├── Italic   → MyFont-Italic (UFontFace)
└── Fallback → MyFont-CJK (UFontFace)
```

但一条 RCSS `@font-face` 表示一个明确的字体族、字重和样式。`UFont` 中组合的字体面、字符范围与 fallback 规则不能直接替代 RCSS 的字体声明。

因此 MarkupUI 将两者的职责明确分开：

- `UFontFace`：MarkupUI/RCSS 的字体资源；
- `UFont`：Unreal、Slate 和 UMG 的字体配置资源。

## 常见错误排查

### 日志提示期望 `UFontFace`

如果日志包含：

```text
Expected a UFontFace asset.
```

请检查 RCSS 引用的资产是否为 Font Face，而不是同名 Font。导入时生成两个资产的情况下，RCSS 必须引用 Font Face 的资产名称。

### 日志提示无法打开 `.ttf`

先根据日志中的资源来源检查对应位置：

- `Source=UnrealAsset`：检查资产目录和 Font Face 名称；
- `Source=Disk`：检查磁盘中是否存在对应字体文件。

不能只根据 `.ttf` 扩展名判断字体来自磁盘还是 Unreal 资产，应以当前文档使用的资源模式为准。

### 修改字体或 RCSS 后仍显示旧结果

重新导入相关 Font Face 与 RCSS 资产后，对文档执行 Full Reload，以完整应用字体和样式变化。
