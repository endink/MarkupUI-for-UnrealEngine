# 资源路径与根目录

MarkupUI 使用与 HTML/CSS 相近的资源路径规则。相对路径相对于当前引用文件，`/` 开头的路径相对于当前资源来源对应的 MarkupUI 资源根目录。

资源根目录同时也是强制访问边界。RML、RCSS、模板、字体和图片等资源都不能通过 URI 访问根目录之外的位置。

## 配置资源根目录

MarkupUI 提供两个相互独立的资源根目录：

- `AssetResourceRootDirectory`：Unreal 资产文档使用的资源根目录；
- `DesignResourceRootDirectory`：设计文件及磁盘资源使用的根目录。

资源域由入口文档的加载方式确定：

- 入口文档从 Unreal 资产加载时，该文档属于 Unreal 资产资源域；
- 入口文档从磁盘文件加载时，该文档属于磁盘资源域；
- 文档引用的 RML、RCSS、模板、字体和图片继承入口文档的资源域；
- 嵌套加载的资源继续继承同一资源域，不会因文件扩展名或路径形式改变来源；
- 资源 URI 不能主动切换资源域；
- 当前资源域查找失败后，不会自动前往另一资源域回退查找。

## 设计文件导入与资产名称

设计目录导入保留子目录结构，并按资源类型添加资产名前缀。可在 **Project Settings → Plugins → MarkupUI → Resources → Asset Naming** 分别设置 Document、Style Sheet、SVG、Font Face 和 Texture 前缀。页面中的引用保持原始文件名，不需要添加前缀。以下为默认配置：

| 设计文件 | 导入后的 UE 资产名 |
| --- | --- |
| `Menu.rml` | `RML_Menu` |
| `Menu.rcss` | `RCSS_Menu` |
| `Logo.svg` | `SVG_Logo` |
| `Default.ttf` / `Default.otf` | `Default`（默认不加前缀） |
| `Logo.png` / `Logo.jpg` | `T_Logo` |

- 资产域优先查找规范化名称，找不到时回退到无前缀的原始名称。两者都存在时使用规范化资产；规范化资产类型不正确时应修正资产，而不是依赖旧资产兜底。
- 导入与资产域 URI 查找始终在源文件名之前添加配置前缀，不将文件名自身的前缀视为已经规范化。例如 `Logo.png` 对应 `T_Logo`，`T_Logo.png` 对应 `T_T_Logo`，两者不冲突。磁盘域仍按实际文件名访问，不使用这些资产前缀。
- 只有内容浏览器的 **Normalize MarkupUI Asset Names** 操作会保留已有正确前缀，避免对同一资产重复操作时不断添加前缀。
- 前缀可以留空，例如 Font Face Prefix 留空时，`Default.ttf` 导入为 `Default`。空前缀仍参与最终名称冲突检查。
- 前缀必须使用合法的 UE 资产名称字符，不能包含路径分隔符。非法配置会阻止导入和资产查找，并给出错误。
- 修改配置不会自动改名已有资产，也不会记忆之前使用的前缀；查找只尝试当前规范化名称和原始名称。修改后重新加载文档；已有旧前缀资产需要自行调整名称，右键规范化不会猜测并删除旧前缀。
- 一次导入使用点击开始时的配置，期间修改设置只影响后续导入。
- 点击开始后先检查全部目标名称。若同一目标目录存在冲突，整次导入取消，并显示全部冲突文件；不会先导入一部分。
- `Menu.rml` 与 `Menu.rcss` 可以共存；`Logo.png` 与 `Logo.jpg` 都对应 `T_Logo`，不能同时导入同一目标目录。`Logo.png` 与 `T_Logo.png` 同样冲突。判断忽略大小写。
- 不同子目录中的同名文件可以共存。
- 不自动迁移已有资产。在内容浏览器选择资产后，使用 **Normalize MarkupUI Asset Names** 右键菜单规范化名称。只处理 RML、RCSS、SVG、Font Face、Texture2D；已规范化或不支持的资产跳过，目标名称被占用时提示并跳过，不覆盖、不自动加编号。
- 导入资产不自动保存；确认结果后由用户保存。

以下路径示例中的无前缀 UE 资产路径表示兼容的旧资产位置。例如 `/images/logo.png` 优先查找 `/Game/UI/images/T_logo`，不存在时才查找示例中的 `/Game/UI/images/logo`。目录访问边界不变，回退不会跨资源域或跨根目录。

## 相对路径

不以 `/` 开头的路径相对于直接引用它的文件所在目录。

假设 Unreal 资产资源根目录为：

```text
/Game/UI
```

当前样式表位于：

```text
/Game/UI/styles/pages/MainStyle
```

以下 RCSS：

```css
.logo {
    decorator: image("../../images/logo.png");
}

@font-face {
    font-family: "Interface";
    src: "../../fonts/Interface-Regular.ttf";
}
```

分别引用：

```text
/Game/UI/images/logo
/Game/UI/fonts/Interface-Regular
```

RmlUi 6.3 通过 RML 中的 `<link>` 加载外部 RCSS，不支持浏览器 CSS 的 `@import`。`<link>` 的相对路径从当前 RML 所在目录开始计算：

```xml
<link type="text/rcss" href="../styles/page.rcss" />
```

RCSS 加载后，其中的字体和图片路径继续相对于该 RCSS 自身所在目录，而不是相对于最初的 RML 文档：

```css
@font-face {
    font-family: "Interface";
    src: "../fonts/Interface-Regular.ttf";
}

.logo {
    decorator: image("../images/logo.png");
}
```

## 根路径

以 `/` 开头的 URI 相对于当前资源域的资源根目录。这里的 `/` 是 MarkupUI 资源根，不是 Unreal Package Mount Point，也不是操作系统文件系统根目录。

当 `AssetResourceRootDirectory` 为 `/Game/UI` 时：

```text
/images/logo.png  -> /Game/UI/images/logo
/fonts/main.ttf   -> /Game/UI/fonts/main
/Game/a.png       -> /Game/UI/Game/a
/Engine/a.png     -> /Game/UI/Engine/a
```

因此，`/Game/a.png` 不会绕过配置直接访问 Unreal 的 `/Game/a`。

当 `DesignResourceRootDirectory` 为 `D:/Project/MarkupUIResources` 时：

```text
/images/logo.png -> D:/Project/MarkupUIResources/images/logo.png
/fonts/main.ttf  -> D:/Project/MarkupUIResources/fonts/main.ttf
```

同一份 RML/RCSS 可以使用一致的相对路径和根路径组织方式；具体访问 Unreal 资产还是磁盘文件，由入口文档的资源来源决定。

## 根目录访问边界

所有路径在使用前都会统一分隔符并折叠 `.` 和 `..`。最终位置必须仍然位于对应的资源根目录内。

`..` 并非一律禁止。相对路径可以返回上级目录，只要规范化后的完整路径仍位于当前资源根目录内。

### 示例 1：相对路径与根目录边界

本示例使用这些资源根目录：

```text
AssetResourceRootDirectory = /Game/UI
DesignResourceRootDirectory = D:/Project/MarkupUIResources
```

并假设正在进行资源引用的 RCSS 分别位于：

```text
UE 资产：/Game/UI/styles/pages/MainStyle
磁盘文件：D:/Project/MarkupUIResources/styles/pages/main.rcss
```

相对路径从当前 RCSS 所在目录开始计算；根路径从对应的资源根目录开始计算。

| 访问方式 | RCSS 中的引用 | 规范化后的完整路径 | 允许 | 原因 |
| --- | --- | --- | --- | --- |
| UE 资产 | `../../shared/colors.rcss` | `/Game/UI/shared/colors` | ✅ |  |
| UE 资产 | `../../../secret.rcss` | `/Game/secret` | ❌ | 最终路径逃出了 `/Game/UI`。 |
| UE 资产 | `images\..\..\..\..\secret.png` | `/Game/secret` | ❌ | 统一分隔符并折叠 `..` 后逃出了 `/Game/UI`。 |
| UE 资产 | `/images/logo.png` | `/Game/UI/images/logo` | ✅ |  |
| 磁盘文件 | `../../shared/colors.rcss` | `D:/Project/MarkupUIResources/shared/colors.rcss` | ✅ |  |
| 磁盘文件 | `../../../secret.rcss` | `D:/Project/secret.rcss` | ❌ | 最终路径逃出了 `D:/Project/MarkupUIResources`。 |
| 磁盘文件 | `/images/logo.png` | `D:/Project/MarkupUIResources/images/logo.png` | ✅ |  |
| 磁盘文件 | `C:\External\secret.png` | `C:/External/secret.png` | ❌ | URI 直接指定了磁盘绝对路径。 |
| 磁盘文件 | `//server/share/secret.png` | `//server/share/secret.png` | ❌ | URI 直接指定了 UNC 网络路径。 |
| 磁盘文件 | `file:///C:/External/secret.png` | `C:/External/secret.png` | ❌ | 不允许使用 `file://` 绕过设计资源根目录。 |

### 示例 2：重新解释 Unreal Mount Point 形式

Unreal 资产 URI 不能使用 `/Game`、`/Engine` 或其他插件 Mount Point 绕过 `AssetResourceRootDirectory`。所有 `/...` 引用一律从配置的资产资源根目录开始。

本示例假设：

```text
AssetResourceRootDirectory = /Game/UI
```

因此，RCSS 中看似 Unreal 全局资产路径的引用会被重新解释为资产资源根目录下的普通子路径：

| 访问方式 | RCSS 中的引用 | 规范化后的完整路径 | 允许 |
| --- | --- | --- | --- |
| UE 资产 | `/Game/a.png` | `/Game/UI/Game/a` | ✅ |
| UE 资产 | `/Engine/a.png` | `/Game/UI/Engine/a` | ✅ |
| UE 资产 | `/MarkupUI/images/a.png` | `/Game/UI/MarkupUI/images/a` | ✅ |

## 推荐的目录组织

建议让 Unreal 资产与磁盘资源采用相同的内部布局：

```text
MarkupUIResources/
├── documents/
├── styles/
├── templates/
├── fonts/
└── images/
```

推荐默认使用 `/` 根路径引用 RML、RCSS、模板、字体和图片。根路径不依赖当前文档所在目录，因此移动 RML 或 RCSS 后，已有资源引用仍能指向资源根目录内的相同位置。

例如：

```xml
<link type="text/rcss" href="/styles/theme.rcss" />
<link type="text/rcss" href="/styles/page.rcss" />
```

```css
@font-face {
    font-family: "Interface";
    src: "/fonts/Interface-Regular.ttf";
}

.logo {
    decorator: image("/images/logo.png");
}
```

只有当资源与引用它的文件具有明确的局部从属关系，并且预计始终一起移动时，才建议使用相对路径。例如，一个页面专属 RCSS 引用同目录下的页面专属图片：

```css
.page-banner {
    decorator: image("page-banner.png");
}
```

如果该图片是多个页面共享的资源，应改用 `/images/page-banner.png`，避免移动 RCSS 后需要同步修改引用路径。

修改资源根目录或资源布局后，应重新加载文档，使文档及其相关样式、模板、字体和图片按新路径重新解析。
