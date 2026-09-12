# UE 数据绑定设计草案

> 设计目标与当前可调用能力分开说明；示例不是完整功能已验收的证明。本文不假定读者了解 RML 或 RmlUi 的源码。

> **AI 阅读与实施约束：** 本文主要表达设计理念、职责边界、层次结构、数据流和必须保持的语义，不是要求逐字落实的 API 规格。架构讨论不要求最终类名、拆分方式与草案一一对应，但标为当前用法的示例必须与现有公开 API 一致。已废弃的宏、Getter／Setter 注册重载和更新作用域不能再被当作待实现要求。实施前必须结合 MarkupUI 现有代码、已经封装的前端能力和当前明确决策重新核准设计，优先复用已有类型与抽象，不得为了匹配示例而重复定义类型或机械增加中间层。只要保持本文规定的职责边界和行为契约，实现可以调整命名、拆分方式、所有权表达和调用形式；若实现条件与草案假设冲突，应先指出差异并讨论，而不是擅自把草案示例当成既定代码结构。

## 一句话目标

让 RML 界面从明确登记的可绑定对象中读取数据；让输入框、复选框等控件把已验证的结果安全写回；让按钮等交互以命令通知游戏逻辑。无论界面放在 Slate（C++）还是 UMG（蓝图）中，RML 写法和结果都一致。

例如，角色金币变化后，`{{ gold }}` 自动刷新；玩家拖动音量滑条后，游戏设置中的音量被限制在合法范围并写回；玩家点击物品后，游戏收到“选择物品”的命令及物品 ID，而不是让 RML 任意调用游戏对象。

## 设计起点：C++ 原生优先，蓝图只是包装

数据绑定的第一步必须是可独立使用的 C++ 原生契约：它不要求 `UObject`、反射、UMG 或蓝图。Slate、游戏模块和任何普通 C++ 数据结构都能通过它创建模型、提供值、接受写入并注册命令。

第二步是 UE 反射数据上下文：文档通过 `SetDataContext(Name, UObject&)` 把业务对象注册到指定的 RML `data-model` 名称。第三步才是 UMG 控件：它调用相同的数据上下文、字段通知和集合操作节点，不拥有另一套 RML 语法、刷新时序或验证规则。

```text
C++ 原生模型、值、字段、写入、命令契约
                ↑                 ↑
      Slate / 普通 C++      UE 反射适配层
                                      ↑
                         UObject 数据上下文、UMG 与蓝图节点
```

这样做有三个结果：

- C++ 项目可在没有 UMG 的情况下完整使用数据绑定；
- 反射不能绕开字段权限、类型转换和命令参数契约；
- 蓝图用户得到的行为与 Slate 一致，后续也能平滑迁移部分高频逻辑到 C++。

## 来自 WPF 数据绑定的通用原则

WPF 的数据绑定建立在“可通知的数据对象”上：属性由 Getter/Setter 公开，Setter 只有在新旧值不同后才发送“此属性已变化”的通知；集合则在增加、删除、替换或重排时单独通知。这样，界面不是每帧轮询所有对象，而是在数据真正变化时更新。WPF 同时区分单向、双向、仅向源写入、一次性读取，以及每次属性变化、失去焦点、显式提交等写回时机。

MarkupUI 应采用 **C++ 数据对象 + 精确变化通知** 的思想，但不照搬 WPF 的 XAML 绑定语法或依赖属性体系：

| WPF 的通用思路 | MarkupUI 的对应设计 |
| --- | --- |
| 属性 Getter/Setter + 属性变化通知 | `FMarkupObservableObject`、`TMarkupProperty<T>` 与显式注册／通知。 |
| 可观察集合 | `TMarkupObservableArray` 与 `TMarkupObservableMap`；它们只能作为可观察对象的属性，由集合变化事件通知对应字段。 |
| 数据上下文 | 文档按 RML `data-model` 名称登记的可观察对象。只有文档支持设置数据上下文；元素、数组和 Map 均不能直接成为设置目标。 |
| 单向、双向等绑定模式 | 由元素上的绑定属性选择 FromSource、FromUI 或双向；字段权限只决定对应读取或写入请求是否被接受，不引入与 RML 原生语法冲突的另一套标记。 |
| 每次变化、失焦、显式提交 | 保持 RmlUi 控件原有 `change` 语义为默认；高频或延迟提交用明确命令实现，不隐式改变控件行为。 |

最重要的差异是刷新粒度。RmlUi 原生数据模型只接受顶层变量的脏标记；因此路径 `Player.Name.First` 最终标脏所属 `data-model` 的顶层字段 `Player`；业务侧仍可通知完整路径或子对象的本地属性。MarkupUI 文档按名称登记多个可绑定对象，RML 使用预先写好的 `data-model="..."` 为不同 DOM 子树选择模型；数组和 Map 只能作为对象属性参与绑定，不需要字段到文本、属性或样式的 Lambda 绑定 API。

## 三层分工

```text
游戏状态 / 设置 / 业务对象
        ↕  只公开声明过的字段与操作
UE 绑定模型
        ↕  值、校验结果、命令参数
RML 文档
```

- **游戏状态**是真正的数据所有者。例如角色属性、设置对象、背包组件。它决定什么字段可展示、什么字段可写、什么操作允许发生。
- **UE 绑定模型**是每个界面实例的受控入口。它把 UE 的值变为 RML 能显示的值，追踪需要刷新的字段，并处理来自控件的写入和事件。
- **RML 文档**只描述显示和交互意图。例如“显示金币”“当音量改变时写入 `music_volume`”“点击时执行 `select_item`”。它不持有游戏对象，也不拥有存档、网络或权限逻辑。

原生可观察对象只公开已登记属性和命令；UObject 走反射可见性筛选，不要求业务对象逐个调用原生 RegisterProperty。两条路径不能混为一谈。

## 术语

| 术语 | 含义 |
| --- | --- |
| 模型 | RML 用来读写的一组命名数据，例如 `inventory`。一份正在显示的文档可以按名称登记多个独立模型。 |
| 字段 | 模型中的顶层数据，例如 `gold`、`items`、`settings`。RML 可读取 `items[0].name`；原生数据模型以顶层字段刷新。 |
| 路径 | 在 RML 中访问数据的写法，例如 `settings.music_volume`。 |
| 只读字段 | RML 可以显示但不能通过输入控件写回的字段。 |
| 可写字段 | RML 可以通过指定控件写回的字段；每次写入均须类型转换和校验。 |
| 命令 | RML 发出的具名交互意图，例如“选择物品”“开始游戏”。它有固定的参数契约，由已登记的 C++ 成员函数接收。 |
| 脏标记 | 表示某个字段已变化、下一次界面更新应重新读取它的信号。 |

## 支持范围

以下演示 RML 语法与目标数据结构，不与后文各个独立 C++ 示例共用一份模型。字段大小写必须与实际登记名称一致：

```html
<div data-model="inventory">
  <h1>{{ player.display_name }}</h1>
  <p data-if="gold > 0">金币：{{ gold }}</p>

  <button data-for="item, index : items"
          data-class-selected="item.id == selected_item_id"
          data-event-click="select_item(item.id)">
    {{ index + 1 }}. {{ item.name }} × {{ item.count }}
  </button>

  <input type="range" min="0" max="1" step="0.05"
         data-value="settings.music_volume"/>
  <input type="checkbox" data-checked="show_minimap"/>
</div>
```

| RML 写法 | 计划 | 使用者可观察到的行为 |
| --- | --- | --- |
| `{{ ... }}`、`data-attr-*`、`data-class-*`、`data-style-*` | 支持 | 读取字段和表达式结果。 |
| `data-if`、`data-visible`、`data-alias-*` | 支持 | 控制显示、可见性和模板作用域。 |
| `data-for` | 支持 | 用数组生成列表；数组变化后列表自动更新。 |
| `data-value`、`data-checked` | 支持 | 对已声明为可写的标量执行双向绑定。 |
| `data-event-*` | 支持 | 发送受控的命令，可绑定已登记的 C++ 成员函数。 |
| 自定义转换函数 | 支持 | 格式化展示值；必须是无副作用的纯计算。 |
| `data-rml` | 延后 | 它可改变文档内容，需单独确定安全和重载规则。 |

`data-*` 和 `{{ ... }}` 是 RML 的保留语法，必须在文档加载前写好。运行中给现有元素动态添加这些属性，不应期待它们变成绑定。

## C++ 绑定方案

本节示例按当前公开 API 编写。省略的业务处理用注释标明，不再使用拟议的属性宏、Getter／Setter 注册重载或不存在的更新作用域。

### 可观察对象与 Property

- 原生模型契约是 `MarkupUI::DataBinding::IObservableObject`。开发者可以自行实现，不强制继承 `FMarkupObservableObject`。
- `FMarkupObservableObject` 提供属性登记、命令登记与变化通知的常用实现。
- 每个实例使用 `TMarkupProperty<T>` 字段保存自己的值；非模板基类 `FMarkupProperty` 用于统一登记和按名称查询。
- Property 构造时只传初始值；名称在 `RegisterProperty(Name, Property)` 时传入，使用区分大小写的 `FString`。
- 属性名、命令名分别不能重复；重复注册输出错误并保留原登记项，不以崩溃处理。
- `RegisterProperty` 不接收 Getter／Setter。下面的业务 Get/Set 是普通 C++ 包装，前端不会根据函数名自动调用它们。
- 业务代码通过对象的 `SetValue(Property, Value)` 修改值并发送通知。直接调用 Property 的 `SetValue` 不会自动发布所属对象的属性通知。
- 当前不提供属性访问权限枚举、属性声明宏、静态 `Make`、清除局部值或默认值回退机制。

### 标量与嵌套对象

```cpp
#include "DataBinding/MarkupObservableObject.h"

class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        RegisterProperty(TEXT("Health"), HealthProperty);
        RegisterProperty(TEXT("DisplayName"), DisplayNameProperty);
    }

    int32 GetHealth() const { return GetValue(HealthProperty); }
    void SetHealth(int32 Value) { SetValue(HealthProperty, Value); }
    const FString& GetDisplayName() const { return GetValue(DisplayNameProperty); }
    void SetDisplayName(const FString& Value) { SetValue(DisplayNameProperty, Value); }

private:
    TMarkupProperty<int32> HealthProperty = TMarkupProperty<int32>(100);
    TMarkupProperty<FString> DisplayNameProperty = TMarkupProperty<FString>(FString(TEXT("Player")));
};

class FPartyUiState final : public FMarkupObservableObject
{
public:
    FPartyUiState()
    {
        RegisterProperty(TEXT("Player"), PlayerProperty);
    }

    void SetPlayer(const TSharedPtr<FPlayerUiState>& Value)
    {
        SetValue(PlayerProperty, Value);
    }

private:
    TMarkupProperty<TSharedPtr<FPlayerUiState>> PlayerProperty = TMarkupProperty<TSharedPtr<FPlayerUiState>>(TSharedPtr<FPlayerUiState>());
};
```

例如 `Party->SetPlayer(Player)` 后，模型内的 `Player.Health` 访问子对象属性。子对象自身调用 `SetHealth` 会发出自己的变化通知；父对象替换 `Player` 则通知父属性。业务对象的共享引用由业务侧管理，绑定本身不替业务侧长期保活对象。

### 集合属性与显式通知

普通 `TArray/TMap` 与 `TMarkupObservableArray/Map` 都可以放入 `TMarkupProperty<T>`。前者要求业务代码显式通知，后者的可通知修改操作会发布集合变化。可观察容器暴露可写存储时采用保守 Reset；长期保存引用并在以后修改，仍不能期待容器侦测每一次外部写入。普通容器的下标写入则需要显式通知。

```cpp
#include "DataBinding/MarkupObservableArray.h"
#include "DataBinding/MarkupObservableMap.h"
#include "DataBinding/MarkupNotifyUtilities.h"

class FInventoryUiState final : public FMarkupObservableObject
{
public:
    FInventoryUiState()
    {
        RegisterProperty(TEXT("Gold"), GoldProperty);
        RegisterProperty(TEXT("Items"), ItemsProperty);
        RegisterProperty(TEXT("ItemCounts"), ItemCountsProperty);
        RegisterProperty(TEXT("Tags"), TagsProperty);
    }

    int32 GetGold() const { return GetValue(GoldProperty); }
    void SetGold(int32 Value) { SetValue(GoldProperty, Value); }

    void AddItem(const FString& Item)
    {
        GetMutableValue(ItemsProperty).Add(Item);
    }

    void SetItemCount(const FString& Key, int32 Count)
    {
        GetMutableValue(ItemCountsProperty).Add(Key, Count);
    }

    void AddTag(const FString& Tag)
    {
        TArray<FString>& Tags = GetMutableValue(TagsProperty);
        Tags.Add(Tag);
        MarkupUI::NotifyArrayAdded(Tags);
    }

private:
    TMarkupProperty<int32> GoldProperty = TMarkupProperty<int32>(0);
    TMarkupProperty<TMarkupObservableArray<FString>> ItemsProperty = TMarkupProperty<TMarkupObservableArray<FString>>(TMarkupObservableArray<FString>());
    TMarkupProperty<TMarkupObservableMap<FString, int32>> ItemCountsProperty = TMarkupProperty<TMarkupObservableMap<FString, int32>>(TMarkupObservableMap<FString, int32>());
    TMarkupProperty<TArray<FString>> TagsProperty = TMarkupProperty<TArray<FString>>(TArray<FString>());
};
```

集合必须是已登记对象属性中的原集合，不能用复制出来的临时集合通知原属性。普通集合通知按已观察到的集合定位订阅；页面尚未读取的路径没有刷新需求。Map 的数据访问和通知能力不等于 RML 已有完整字典表达式语法，见后文边界。

### 命令登记与调用

`FromMember` 必须同时接收共享 Owner 和成员函数指针。因此属性可以在构造函数中注册，命令要在对象已经由共享指针管理后注册，不能在构造函数里调用 `AsShared()`。

```cpp
class FInventoryCommandUiState final : public FMarkupObservableObject
{
public:
    FInventoryCommandUiState()
    {
        RegisterProperty(TEXT("SelectedItem"), SelectedItemProperty);
    }

    static TSharedRef<FInventoryCommandUiState> Create()
    {
        TSharedRef<FInventoryCommandUiState> State = MakeShared<FInventoryCommandUiState>();
        State->RegisterCommand(TEXT("SelectItem"), FMarkupCommand::FromMember(State, &FInventoryCommandUiState::SelectItem));
        State->RegisterCommand(TEXT("RequestClose"), FMarkupCommand::FromMember(State, &FInventoryCommandUiState::RequestClose));
        return State;
    }

    void SelectItem(int32 ItemId)
    {
        SetValue(SelectedItemProperty, ItemId);
    }

    void RequestClose()
    {
        OnCloseRequested.ExecuteIfBound();
    }

    FSimpleDelegate OnCloseRequested;

private:
    TMarkupProperty<int32> SelectedItemProperty = TMarkupProperty<int32>(INDEX_NONE);
};

TSharedRef<FInventoryCommandUiState> Commands = FInventoryCommandUiState::Create();
if (const FMarkupCommand* Command = Commands->GetCommandByName(TEXT("SelectItem")))
{
    const FString& Signature = Command->GetSignature();
    TConstArrayView<FMarkupCommandParameter> Parameters = Command->GetParameters();
    Command->Exec(int32(42));
}
```

`GetAllCommands()`、`GetCommandByName()` 提供查询；命令自己提供参数描述、签名与 `Exec`。命令弱引用 Owner，不能借此长期保活模型。RML 的命令在设置上下文时登记，缺失命令不使用空槽延迟补造；替换上下文时旧命令清理后登记新命令。

#### 命令签名硬性契约

命令必须经过两阶段检查：

- **编译期签名检查：** `FromMember` 从成员函数指针推导所属类、返回类型和完整参数列表，并在编译期拒绝任何违反本节契约的函数签名。开发者不需要另外声明一份参数描述。
- **调用前参数检查：** 每次 RML 调用命令时，必须先检查实际参数数量、值类型以及转换结果是否与已确定的成员函数签名匹配。只有全部参数通过检查后才能进入成员函数。

- 同一个可观察对象的命令名必须唯一。`RegisterCommand` 不允许重复登记，也不采用覆盖或“最后一个生效”语义；基类和派生类最终形成的命令表同样不能重名。不同具名 DataModel 之间可以使用同名命令。
- 只有直接作为具名 DataContext 根对象的 `MarkupUI::DataBinding::IObservableObject` 所公开的命令才能被 RML 调用。嵌套对象、Array 元素或 Map 值中的可观察对象即使登记了命令，这些命令也不会暴露给该 DataModel。RML 可以从这些叶子对象中读取值作为根命令的参数，但不能通过对象路径调用叶子命令。
- 命令只能绑定非静态成员函数，返回类型必须是 `void`。RmlUi 原生事件回调没有命令返回值，因此不扩展额外的返回值语义。
- RML 传入的参数数量必须与成员函数签名完全一致。任何参数缺失或多余都拒绝调用。
- 不支持 C++ 默认参数。即使成员函数声明了默认值，RML 也必须显式传入全部参数。
- 不支持可变参数、输出参数、指针参数、非常量左值引用和右值引用。参数只能按值或按常量左值引用传入。
- 字符串值可以按成员函数签名转换为 `FString`、`FText` 或 `FName`，不支持字符串指针或其他隐式字符串类型。
- 数字参数使用 C++ `static_cast` 的转换语义。整数收窄、浮点精度收窄以及浮点数转整数可以产生截断或精度损失，框架不承诺保持原值精度。只有当转换会进入 C++ 未定义行为时才拒绝调用。
- 枚举参数只接受没有小数部分且位于枚举底层整数类型范围内的数值。存在 UE 枚举元数据时，还必须是已声明的枚举项；普通 C++ 枚举无法通用地判定稀疏枚举项，因此只保证底层整数范围正确。
- `FLinearColor`、`FVector2D`、`FVector3f` 和 `FVector` 只接受对应的 RmlUi 值类型，不从字符串或数字参数猜测构造。两种三维向量可相互赋值及传参；经过 RML 时使用 float 精度，因此 `FVector` 的 double 分量可能发生精度收窄。
- 重载成员函数必须在 `FromMember` 调用处使用 `static_cast` 明确选择签名，不提供按 RML 运行时参数自动选择 C++ 重载的机制。
- 参数数量或类型检查失败时，命令不执行；不使用默认值补齐，也不截断多余参数。

重载成员函数在创建命令时明确选定签名，例如：

```cpp
class FSelectionUiState final : public FMarkupObservableObject
{
public:
    void SelectItem(int32 ItemId) { /* 按编号处理选择。 */ }
    void SelectItem(const FString& ItemName) { /* 按名称处理选择。 */ }
};

TSharedRef<FSelectionUiState> Selection = MakeShared<FSelectionUiState>();
FMarkupCommand SelectById = FMarkupCommand::FromMember(
    Selection,
    static_cast<void (FSelectionUiState::*)(int32)>(&FSelectionUiState::SelectItem));
SelectById.Exec(int32(42));
```

### Property 与 Frontend 的边界

这里保留设计关系，不复制私有辅助类或再声明一套假想接口：

- 类型化 Property 保存业务值，非模板 Property 让属性登记表不必模板化；类型擦除不是前端适配。
- Property 不实现、持有或返回 Frontend 接口。业务侧字符串使用 `FString`，不要求业务开发者提供 UTF-8。
- 通用 Frontend 的 `IValue`、`IScalar`、`IObject`、`IArray`、`IMap` 位于 `MarkupUI::DataBinding::Frontend`，不是 Property 的基类。
- Frontend 值按 Kind 区分 Scalar、Object、Array、Map，Scalar 再区分 ValueType；不另造重复的种类枚举。
- Adapter 的共享引用管理 Adapter 自身，背后的业务对象按弱引用实时解析；读取失败返回无值，不用读取作用域长期保活业务对象。
- 标量读取结果的有效期遵循 Scalar 接口约定，不把临时借用地址当作永久业务值；字符串转换属于前端通讯边界。
- 模型按名称保留稳定空槽，尚未提供对象时不伪造业务默认数据；替换同名对象后清理旧绑定，再允许字段按需解析并刷新。
- 数组与 Map 不能直接作为 DataContext。元素和值的支持范围见文末表格；不因为容器能存储某种 C++ 类型就推断该类型能绑定。
- 集合整体替换由类型化 `SetValue` 完成；前端标量元素写回与集合整体写回是两种不同能力，后者不作为通用入口。
- 任意原生类型可以作为 Property 的存储类型，但只有已支持的类型可以转换为前端可读写值。


### 文档的具名数据上下文

MarkupUI 文档使用 RmlUi 原生的具名 `data-model`。C++ 不声明字段到 DOM 表现的映射，而是把可绑定 Object 登记到文档中的模型名；RML 子树通过 `data-model="模型名"` 选择对应对象，并使用原生绑定表达式显示其属性。

RML 作者必须在文档加载前写好模型名：

```html
<div data-model="inventory">
  <span name="hud-gold"></span>

  <div name="player-health" class="health-bar">
    <span name="player-health-label"></span>
  </div>

  <div name="inventory-list"></div>
</div>
```

C++ 通过 `SRmlUiWidget` 为当前文档设置具名数据上下文，输入是原生可观察对象或 UObject。元素不提供 `SetDataContext`；数组和 Map 只能是对象的属性：

```cpp
// SRmlUiWidget 的公开成员声明摘录。两种输入都要求非空。
bool SetDataContext(const FString& Name, const TSharedRef<MarkupUI::DataBinding::IObservableObject>& Object);
bool SetDataContext(const FString& Name, UObject& Object);
void RemoveDataContext(const FString& Name);
void ClearDataContexts();
```

MarkupUI 的文档包装对象必须早于原生 RmlUi Document 存在。加载 RML 时，胶水层为文档中静态声明的每个 `data-model` 建立稳定的模型槽；尚未设置 Object 的槽为空，所有读取安全返回无值。因此文档加载不依赖业务对象是否已经创建，也不要求调用方预先登记模型名。

```cpp
Ui->SetDataContext(TEXT("inventory"), InventoryState);
```

一份文档可以登记多个不同名称的数据上下文。每个名称对应一个稳定模型槽，不同 RML 子树通过各自静态声明的 `data-model` 选择模型。

**RML 表现规则：**

- 文本、属性、样式和列表继续使用 RmlUi 原生绑定表达式，包括 `data-for`。
- 格式化后的显示值应作为可绑定属性提供，不由绑定 API 接收 Lambda。
- 文本、属性、样式、类、列表和 DOM 子树的具体表现都由 RML 描述；C++ 只负责提供具名的可观察数据上下文。
- 具名 DataModel 使用 RmlUi 的 `allow_missing_variables` 机制。文档首次加载时尚未绑定的变量显示为默认值，后续绑定并标脏后即可正常刷新。
- MarkupUI 对外以文档为数据刷新协调边界，不提供单个元素的 Update API；底层常规更新由 RmlUi Context 统一推进。
- 统一更新入口不等于每次重算整份文档。RmlUi 内部先检查 DataModel 的脏变量集合，只更新依赖这些变量的 DataView 和相关元素；样式、布局与位置也通过各自的 dirty 标记按需更新。

**运行时设置数据上下文：**

- `SetDataContext(Name, Object)` 只填充或替换同名模型槽背后的 Object Adapter，不替换模型槽。
- RML 元素已有的 `data-model` 属性保持不变。
- 设置完成后，该模型关联的顶层字段被标记为脏，页面在下一次更新时重新读取 Object。
- 如果当前 RML 尚未引用这个名称，胶水层仍可提前创建模型槽并保存 Adapter。该设置在当前页面没有可见效果，后续加载或重载的文档引用同名 `data-model` 时会直接使用它。

**静态 `data-model` 的边界：**

- “静态”只约束 RML 元素上的 `data-model` 属性，不约束 `SetDataContext` 的调用时机。
- 页面已经声明模型名时，可以在文档加载后再提供 Object；稳定模型槽会接管更新，不需要重载文档。
- 不支持在运行时给已经挂接的元素新增或修改 `data-model` 属性，因为 RmlUi 不会因此重新建立元素绑定。

**不提供的绑定入口：**

- 不提供从 C++ 直接绑定文本、属性、样式、类、列表或任意 DOM 子树的 API。
- 不提供以 Lambda 描述字段表现的另一套绑定机制。
- 默认不提供把任意字符串传给 `SetInnerRML` 的通用数据绑定。重新解析标记可能破坏焦点、输入状态和元素身份，也可能把未受信任文本当作界面标记。

### Object 的数组属性与 `data-for`

`data-for` 是 RmlUi 的原生列表方案。数组必须作为当前可观察 Object 已登记的 Property，页面通过该属性执行 `data-for`；C++ 不提供单独的列表创建、更新、删除或移动回调。

```cpp
Ui->SetDataContext(
    TEXT("inventory"),
    InventoryState);
```

每个条目仍应有稳定且唯一的业务标识，例如物品实例 ID，而不是数组下标。数组发生增删、移动或替换后，`data-for` 依照 RmlUi 的原生规则管理该子树；C++ 不得手工改写这个受管子树，也不应跨更新周期保存其中元素的裸指针。

### Object 的字典属性

普通 `TMap` 和 `TMarkupObservableMap` 可以作为 Property，Map 键支持 `FString`、`FName`、`int32`。通用数据层的 Map 访问、元素写回和通知能力，与 RML 的字典表达式支持分开评价。

当前不以 `data-for="Item : Map"` 或 `Map["key"]` 作为可用 RML 示例，也不承诺为页面提供键排序。完整 Map 语法与遍历是独立议题，不能因底层接口已有 Map 就标记为完成。

### 与原生 RmlUi 数据绑定的边界

一个 RML 元素通过静态 `data-model` 属性选择模型；子元素可以在 RML 中声明另一个 `data-model`，但 C++ 不能给元素动态设置数据上下文。下表是硬规则：

| 场景 | 是否允许 | 原因 |
| --- | --- | --- |
| 一份文档登记多个具名 Object | 允许 | 每个 RML 子树通过静态 `data-model` 选择需要的模型。 |
| 同一名称重复设置 Object | 允许 | 后一次设置替换该模型背后的业务对象，并触发模型重新读取。 |
| 文档加载后首次设置模型名 | 允许 | 胶水层填充稳定模型槽并触发重新读取；调用方无需在加载前预登记。 |
| 设置当前 RML 尚未引用的模型名 | 允许 | 先创建并保存模型槽；当前无可见效果，后续加载或重载可使用。 |
| 运行时新增或修改元素的 `data-model` 属性 | 不允许 | RmlUi 不会因属性变化重新建立元素绑定。 |
| 在 C++ 中给元素设置数据上下文 | 不允许 | 元素不提供 `SetDataContext`，模型选择由 RML 的静态 `data-model` 决定。 |
| `data-for` 容器由 C++ 手工增删子元素 | 不允许 | 列表结构只由 RmlUi 的 `data-for` 管理。 |
| Array 或 Map 直接设置为上下文 | 不允许 | 集合必须作为可观察 Object 的属性参与绑定。 |
| 可写字段用于 `data-value` 等输入绑定 | 允许 | 输入和展示均由该上下文内的原生 RML 绑定表达式管理。 |

严格模式应验证模型名非空且合法，并拒绝向已失效或已卸载的文档设置上下文。

### 数据上下文更新后的时序和限制

数据上下文的属性或集合通知会在正常的文档更新阶段统一驱动 RML 绑定重新求值；若业务代码立刻读取尺寸、偏移等布局结果，可能读到上一次结果。只有确实需要同步读取布局时，才允许显式更新文档，并应把它视为有性能代价的高级操作。

数据上下文不直接改写 RmlUi 维护的 `hover`、`active`、`focus`、`checked`、`disabled` 等内置伪类。项目可定义自己的伪类，例如 `low-health`，但表单控件的焦点、选择和可用状态仍由原生控件/绑定机制负责。

### 支持的数据类型与路径

首版属性值支持：布尔值、各种整数、浮点数、字符串、枚举、颜色、二维向量、对象、数组和字典。原生对象与反射对象通过各自的属性访问能力参与嵌套；不能用一套旧的假想值接口推断所有 C++ 类型都受支持。三维向量支持 `FVector3f` 与 `FVector`，进入 RML 后使用 float 精度。

路径由字段名、对象成员和数组下标构成：

```text
gold
player.display_name
settings.music_volume
items[0].count
```

字段名、成员名、命令名只允许字母、数字和下划线，且以字母或下划线开头。不要把用户输入、资产路径或对象名直接拼进路径。

UObject 引用不会作为指针值暴露给 RML。原生 C++ 可观察对象可以通过 `TWeakObjectPtr<T>` 或 `TStrongObjectPtr<T>` 属性将 UObject、Actor 或组件作为下级对象节点公开，页面只能继续读取该对象允许绑定的反射属性。`UClass*`、委托、接口对象和软引用不作为首版绑定值；若 UI 只需要物品图标或角色名，仍应优先公开稳定 ID、文本和图片 URI，而不是扩大整个业务对象的页面可见范围。

### 输入类型转换

属性写回和命令参数按实际输入类型处理，不因为目标属性是整数、枚举或字符串就提前改变输入值。

- UE 侧统一负责类型转换；枚举的具体成员合法性仍按目标枚举判断。
- 转换失败保留原值，命令参数有任何一项失败时不执行命令。
- 数值转换检查范围，普通浮点数转整数沿用截断语义；枚举额外要求没有小数部分。
- 整数文本必须完整、没有溢出，不接受部分解析后忽略余下字符。
- 字符串、Text、Name 可以相互转换；颜色与向量保持整体值语义，不自动拆分为参数。
- 自定义转换器尚未提供注册入口，作为后续扩展，不视为当前已支持功能。

### 枚举读写与命令参数

- 枚举向页面公开数值，不自动公开枚举名称或 DisplayName。
- UE 反射枚举写回和命令传参都检查底层整数范围与已声明枚举项；失败保留原值或不执行命令，并输出警告。
- 普通 C++ 枚举没有反射项清单，只检查数值可表示性与底层整数范围，不承诺发现未声明的枚举值。
- 小数、NaN、无穷大及越界数值被拒绝，不先截断成整数再接受。数值形式的 `2.0` 可表示整数；枚举名称字符串不转换为枚举值。
- 表单以字符串提交枚举数值时，仅接受完整的十进制整数字符串，例如 `"2"`；拒绝 `"2.5"`、`"2abc"` 等内容。
- 不自动组合 Flags。反射枚举组合值只有本身也是已声明项时才允许。
- 上述规则适用于直接属性、集合中的枚举元素，以及原生命令和 UObject 命令参数。枚举通过有符号 64 位数值通讯，不承诺传递大于 `INT64_MAX` 的无符号枚举值。

### 更新和脏标记

通知可携带相对于通知对象的属性路径；业务代码不需要自行缩减为 RML 顶层字段。已读取路径对应的通知在绑定更新中合并，最终由 RML 按顶层字段重新求值。

```cpp
// Ui 已创建；业务拥有者应持续持有这些模型。
TSharedRef<FInventoryUiState> InventoryState = MakeShared<FInventoryUiState>();
TSharedRef<FPlayerUiState> PlayerState = MakeShared<FPlayerUiState>();
Ui->SetDataContext(TEXT("inventory"), InventoryState);
Ui->SetDataContext(TEXT("player"), PlayerState);

InventoryState->AddItem(FString(TEXT("Potion")));
InventoryState->SetGold(100);
PlayerState->SetHealth(80);
```

上述调用可以连续执行，不需要 `BeginDataUpdate`、`EndDataUpdate` 或手工创建更新作用域。页面在下一次正常更新读取结果，不保证业务调用返回时布局已同步更新。通知合并不代表业务集合可以跨线程并发读写。

### 可观察集合与 RmlUi 桥接

- 可观察数组和 Map 的通知保留新增、删除、插入、替换、重置等动作；普通集合通过通知函数显式报告。
- 通用数据层不假设前端只能整组刷新，也不要求业务方先收拢路径。
- RML 绑定最终将变化收拢到当前模型的顶层字段。例如模型内路径为 `Inventory.Items` 时标脏 `Inventory`；路径直接为 `Items` 时标脏 `Items`。
- 不承诺 RML 对某个数组元素进行独立 DataView 刷新，也不承诺所有原始变更明细会永久保存在绑定层。

### C++ 原生命令

命令使用前文的“共享对象创建后登记”写法；不在构造函数里获取共享 Owner。它只负责接收 UI 意图，业务方法修改属性或集合后仍使用对应通知机制。命令自身持有弱 Owner，业务对象失效后调用不再执行。

## 双向绑定的完整规则

本节的基础标量读写示例使用当前 API；访问权限、声明式校验、拒绝恢复与编辑冲突策略仍是设计目标，不能将这些目标当作已经全部实现的行为。

### 什么是双向绑定

`data-value` 和 `data-checked` 不仅是“控件显示字段值”。双向绑定由两个方向组成：

- **FromSource**：可观察对象中的数据变化更新到页面。例如玩家掉血后，血条随之更新。
- **FromUI**：玩家在页面的输入更新到可观察对象。例如输入文字、拖动音量滑条、勾选复选框或选择单选框。
```cpp
class FMusicSettingsUiState final : public FMarkupObservableObject
{
public:
    FMusicSettingsUiState()
    {
        RegisterProperty(TEXT("music_volume"), MusicVolumeProperty);
        RegisterProperty(TEXT("enable_music"), EnableMusicProperty);
    }

    float GetMusicVolume() const { return GetValue(MusicVolumeProperty); }
    void SetMusicVolume(float Value) { SetValue(MusicVolumeProperty, Value); }
    bool GetEnableMusic() const { return GetValue(EnableMusicProperty); }
    void SetEnableMusic(bool Value) { SetValue(EnableMusicProperty, Value); }

private:
    TMarkupProperty<float> MusicVolumeProperty = TMarkupProperty<float>(0.8f);
    TMarkupProperty<bool> EnableMusicProperty = TMarkupProperty<bool>(true);
};
```

RML 使用这个公开属性：

```html
<div data-model="settings">
  <label for="music-volume">音乐音量</label>
  <input id="music-volume" type="range" min="0" max="1" step="0.01"
         data-value="music_volume"/>
  <span>{{ music_volume }}</span>
</div>
```

挂载时，`settings` 指向这个对象：

```cpp
TSharedRef<FMusicSettingsUiState> Settings = MakeShared<FMusicSettingsUiState>();
Ui->SetDataContext(TEXT("settings"), Settings);
```

| 谁先改变 | 示例代码或操作 | 结果 |
| --- | --- | --- |
| C++ / 游戏设置 | `Settings->SetMusicVolume(0.65f);` | `MusicVolume` 变化并通知；下一次文档更新时，滑条位置和 `<span>` 都显示 `0.65`。 |
| 玩家 | 将滑条拖到 `0.30`，控件按正常 `change` 时机提交 | 输入值转换为 `float` 后写入 `MusicVolume`；写入成功时，模型值变为 `0.30`，随后控件和 `<span>` 显示模型最终值。 |
| 非法输入与业务校验 | 类型不匹配、范围限制、只读约束 | 当前类型转换与未来业务校验需分别核查；上述示例没有声明范围或只读元数据，也不保证自动恢复控件。 |

前端写入这里登记的 Property，不会自动经过示例中的普通 C++ Setter。若需要业务校验，不能仅在该 Setter 中加入逻辑后就假定页面写回也会执行它。

`data-value` 只绑定单个可写标量路径；不要在这里写表达式，例如 `data-value="volume * 100"` 是不允许的。若 UI 显示的是百分比但存储的是 0 至 1，可增加只供展示的字段，或在字段 Write 中处理，而不是让 RML 推断反向公式。

### `data-value` 与 `data-checked`

以下片段均放在声明了相应字段的 `data-model` 子树内；模型名不是属性前缀。`data-value` 同步的是元素的 `value` 属性；`data-checked` 同步的是复选框或单选框的 `checked` 状态。两者不能互换。

复选框同时具有独立的 `value` 与 `checked`：前者表示该选项的提交值，通常是字符串；后者才表示玩家是否勾选。因此，不应使用 `data-value` 绑定布尔开关：

```html
<!-- 错误：绑定的是 value，不是勾选状态。 -->
<input type="checkbox" value="enable_music" data-value="enable_music"/>

<!-- 正确：true 为勾选，false 为取消勾选。 -->
<input type="checkbox" data-checked="enable_music"/>
```

当 `enable_music` 为 `true` 时，复选框勾选；为 `false` 时取消勾选。玩家切换复选框后，控件在正常 `change` 时机将新的 `bool` 写回模型。

单选组的 `checked` 状态由每个选项的 `value` 与同一个字段比较决定：

```html
<input type="radio" name="difficulty" value="easy"
       data-checked="difficulty"/> 简单
<input type="radio" name="difficulty" value="hard"
       data-checked="difficulty"/> 困难
```

这里的 `difficulty` 使用字符串；值等于某个单选框的 `value` 时，该选项被选中。这个例子不承诺字符串到枚举的自动映射。

### 字段方向和提交时机

当前 `RegisterProperty(Name, Property)` 只登记 Property；不存在 `EMarkupPropertyAccess` 或只读／只写属性宏，也不会把普通业务 Getter／Setter 自动登记为绑定访问器。

下表保留访问控制的产品设计目标，不表示当前已提供这些配置：

| 待讨论的访问控制 | 允许读取 | 允许写入 |
| --- | --- | --- |
| ReadOnly | 是 | 否 |
| ReadWrite | 是 | 是 |
| WriteOnly | 否 | 是 |

是否增加这类元数据需要单独决策，不能据此要求恢复旧的属性宏或 Getter／Setter 注册方案。普通 C++ Get/Set 的访问权限不等于前端读写权限。RML 的提交时机继续采用原生控件事件，不引入假想的 WPF 式配置参数。

### 页面绑定方向

页面通过元素上的 RML 数据绑定属性选择绑定方向：

| 绑定方向 | RML 写法 | 页面行为 |
| --- | --- | --- |
| 单向：模型 → 页面 | `data-attr-value`；checkbox 使用 `data-attrif-checked`。 | 只读取模型，不在控件事件时写回。 |
| 单向：页面 → 模型 | `data-event-change` 等事件控制器。 | 只在事件发生时提交值，不从模型读取初始值或后续变化。 |
| 双向 | `data-value`；checkbox/radio 使用 `data-checked`。 | 读取模型更新控件，并在 `change` 时写回。 |
| 一次性 | 首版不提供页面绑定语法。 | 若要支持，文档加载时读取一次后必须解除订阅。 |

`data-value` 与 `data-checked` 是 RmlUi 固定的双向绑定；它们不能指定单向模式。需要单向行为时，页面应使用表中对应的数据绑定属性。`data-value` 始终只能绑定单个标量路径，不能从 `name | to_upper` 等表达式反推出写回目标。

### 页面方向与模型访问控制冲突

这一部分属于待确认的访问控制目标，不是当前 API 的演示：

- 若将来提供只读属性，双向控件提出的写入必须被拒绝。
- 若将来提供只写属性，页面的读取不能绕过权限。
- 拒绝后是否立即恢复控件、如何显示错误，需要单独实现和验证；不能仅由 `RegisterProperty` 没有传 Setter 推断为只读。
- 目前不要用已删除的 `MARKUP_PROPERTY_READONLY/WRITEONLY` 或虚构 Getter／Setter 重载编写例子。

### 输入转换和校验顺序

1. 读取控件提出的字符串、数字或布尔值。
2. 根据字段已声明类型转换。空字符串、非数字、溢出和非法枚举均为转换失败。
3. 执行声明式约束，例如最小值、最大值、长度、正则表达式、枚举集合。
4. 调用字段 Write；它执行最终业务校验，并返回接受、拒绝或修正。
5. 仅在接受或修正后标记所属顶层字段；拒绝时不改变 UE 数据。

以上是待验证的业务校验流程，并非现有注册接口的承诺。当前没有 `FMarkupWriteResult` 公共类型，也没有注册声明式约束的参数；不再以虚构结果类型作为实现要求。拒绝恢复、修正后重读的具体策略应另行确认。

```html
<input type="number" min="0" max="99" data-value="purchase.quantity"/>
<p data-if="purchase.error_message != ''">{{ purchase.error_message }}</p>
```

不要把错误提示依赖为插件自动生成的英文文本。业务 Write 可同时更新 `purchase.error_message`，从而实现本地化和产品一致的提示。

### 文本、输入法与外部更新冲突

文本输入在编辑状态时，输入法合成文本、选择范围和光标位置都必须优先保留。以下为待验证的目标策略，不表示已完成输入冲突处理：

- 玩家尚未提交编辑时，外部更新同一文本字段不覆盖正在输入的内容。
- 控件触发 `change` 后提交；提交成功时显示已接受或修正后的值。
- 提交失败时恢复最后一次有效值，并保留由业务模型提供的错误状态。
- 焦点离开、文档重载或 Widget 销毁时，尚未提交的文本按 RmlUi 控件的正常 `change` 语义处理；游戏不应假设每个按键都会写回。

若产品确实需要每个按键同步，RML 可使用 `data-event-input` 调用专用命令。该命令应自行节流，不能把昂贵查询、网络请求或存档写入绑定在每个字符上。

### 事件中的赋值表达式与命令

常见的“修改设置后保存”应将字段绑定和保存命令分开：

```html
<input type="checkbox" data-checked="show_minimap"/>
<button data-event-click="save_settings()">保存设置</button>
```

`data-checked` 在 checkbox 的 `change` 时写回 `show_minimap`；点击“保存设置”时只执行已登记的 `save_settings()` 命令。

`data-event-*` 触发时也可以应用赋值表达式。若产品需要单独的“切换小地图”按钮，可以这样写：

```html
<button data-event-click="show_minimap = !show_minimap">
  切换小地图
</button>
```

这条赋值表达式会将当前布尔值取反：开启变关闭，关闭变开启。赋值只会作用于模型允许写入的字段；页面也可在同一事件属性中用分号按顺序组合多个赋值表达式与已登记命令。未知命令、错误数量的参数或不匹配的参数类型都不会执行任何游戏逻辑。

### 批量更新与循环预防

从 RML 写回字段后，模型会把该字段标记为已处理。游戏状态随后发出的通知仅用于把最终值同步回控件，不应再次触发同一命令或产生无限循环。业务代码应只修改状态，不要在字段通知中模拟一次新的按钮点击。

## 蓝图支持

本节 UObject 的 C++ 示例使用当前接口；蓝图节点清单属于目标设计／原型，不能据此认定任意 UObject 的正式蓝图工作流或 UMG 包装已经完成。

蓝图与 UMG 可按模型名将业务 `UObject` 设置为 RML 文档的数据上下文。绑定路径使用 `UPROPERTY` 的反射名称；页面事件调用数据上下文上的 `UFUNCTION` 成员。

例如，以下对象可直接作为数据上下文：

```cpp
// MarkupInventoryData.h
#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MarkupInventoryData.generated.h"

UCLASS(BlueprintType)
class UMarkupInventoryData : public UObject
{
    GENERATED_BODY()

public:
    // 本例业务函数只改数据，由后文调用者显式通知。
    void SetGold(int32 InGold) { Gold = InGold; }
    void RemoveInventoryItemAt(int32 Index) { InventoryItems.RemoveAt(Index); }
    TArray<FName>& GetInventoryItems() { return InventoryItems; }

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    int32 Gold = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    TArray<FName> InventoryItems;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    TMap<FName, int32> ItemCounts;

    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void RequestClose() { /* 由项目接入关闭请求。 */ }
};
```

### 设置数据上下文完成绑定

```cpp
#include "UObject/StrongObjectPtr.h"

// 由业务拥有者持有此强引用；不要把它当作绑定提供的保活。
TStrongObjectPtr<UMarkupInventoryData> InventoryData(NewObject<UMarkupInventoryData>());
Ui->SetDataContext(TEXT("inventory"), *InventoryData.Get());
```

```html
<div data-model="inventory">
  <span>{{ Gold }}</span>
  <div data-for="InventoryItem : InventoryItems">{{ InventoryItem }}</div>
</div>
```

### 普通 `UPROPERTY` 的更新

普通 `UPROPERTY` 的读取和写入都作用于该 `UObject` 上的原始属性。`USTRUCT` 是属性值的一部分，页面可使用 `Attributes.Health` 访问嵌套成员；结构内部被业务代码改动时，通知其顶层属性。

普通属性的蓝图节点如下：

| 节点 | 作用 |
| --- | --- |
| Set Data Context | 输入 `Model Name` 和 `Object`，按 `data-model` 名称将该对象设置为 RML 文档的具名数据上下文。 |
| `Set <属性名> (Markup UI)` | 写入该对象的普通 `UPROPERTY`，再通知 RML 刷新依赖此属性的绑定。 |
| Notify Markup Property Changed | 原生 Set 或 C++ 已经改值后，显式通知指定属性。 |

`Set <属性名> (Markup UI)` 在写入成功后通知 MarkupUI 刷新该属性的绑定；普通原生 Set 不自动触发此通知。当前通知会自动合并，不提供 `BeginDataUpdate`／`EndDataUpdate`。

### `TArray` 属性的更新

通过以下蓝图节点来代替原生的数组编辑蓝图节点，可以实现数组变化后自动更新界面。

| 节点 | 作用 |
| --- | --- |
| Add (Markup UI) | 在末尾加入一个元素，并通知数组属性。 |
| Add Unique (Markup UI) | 不存在相同元素时加入，并通知数组属性。 |
| Append (Markup UI) | 追加另一数组的元素，并通知数组属性。 |
| Insert (Markup UI) | 在指定索引插入元素，并通知数组属性。 |
| Remove (Markup UI) | 按索引移除元素，并通知数组属性。 |
| Remove Item (Markup UI) | 移除匹配元素，并通知数组属性。 |
| Clear (Markup UI) | 清空数组，并通知数组属性。 |
| Resize (Markup UI) | 调整数组大小，并通知数组属性。 |
| Reverse (Markup UI) | 反转数组，并通知数组属性。 |
| Set Array Elem (Markup UI) | 修改指定索引的元素，并通知数组属性。 |
| Swap Array Elements (Markup UI) | 交换两个元素，并通知数组属性。 |
| Shuffle (Markup UI) | 打乱数组，并通知数组属性。 |

数组变更节点必须能够追溯到具体 `UObject` 上的数组属性，以确定变更后应通知的对象与属性名。
> 如果数组不是 `UObject` 上的成员变量（例如本地临时变量、函数返回新创建的 Array），无法使用以上节点。

操作完成后，节点通知整个数组属性；随后 `data-for` 等页面绑定将按 RmlUi 的原生规则更新子树。只读查询节点不产生通知。

### `TMap` 属性的更新

通过以下蓝图节点来代替原生的 **Map** 编辑蓝图节点，可以实现数组变化后自动更新界面。

| 节点 | 作用 |
| --- | --- |
| Add (Markup UI) | 保持原生 Add 的使用方式；新键记录新增，已有键记录替换，并通知 Map 属性。 |
| Remove (Markup UI) | 按键移除一个键值对，并通知 Map 属性。 |
| Clear (Markup UI) | 清空 Map，并通知 Map 属性。 |

Map 变更节点同样必须能够追溯到具体 `UObject` 上的 Map 属性；执行完成后通知整个 Map 属性。
> Add 保持 UE 原生节点行为，但内部会区分新增与替换，以保留与 C++ 原生集合一致的细节记录；
> Remove 未找到键时仍可发出一次无害通知。
> Find、Contains、Length、Keys、Values 等只读操作不通知。
> 如果一个 Map 不是 `UOjbect` 上的成员变量（例如本地临时变量、函数返回新创建的 Map），无法使用以上节点。

### 命令

页面事件可以调用数据上下文上允许绑定的 `UFUNCTION` 成员，例如：

```rml
<button data-event-click="SaveSettings()">保存设置</button>
<button data-event-click="ShowMinimap = !ShowMinimap">切换小地图</button>
```

第一个例子调用对象成员函数；第二个例子只应用一个属性表达式。命令执行后如修改了属性或集合，应使用对应的通知机制刷新页面。

## 在 C++ 中进行 `UObject` 变化通知

`UObject` 实例（指针）可以通过 `MarkupUI` 提供的 C++ 通知函数通知属性变化来更新界面。


### 原生 C++ 通知函数

`MarkupUI` 命名空间中的原生通知函数用于报告业务代码已经完成的数据修改。通知函数只负责发送变化信息，不会把任意 C++ 字段注册成可绑定属性，也不会代替实际的数据修改。

使用这些函数时必须满足以下前提：

- 普通属性、Array 和 Map 都必须是可绑定 UObject 对象图中可由反射访问的 `UPROPERTY`，并满足 MarkupUI 的属性可见性规则。
- Array 或 Map 可以位于数据上下文 UObject 上，也可以位于页面已经读取到的下级 UObject 或 `USTRUCT` 路径中，但最终集合本身必须对应一个 `UPROPERTY`。
- 临时变量、局部容器、普通非反射 C++ 成员，以及尚未被页面读取的集合，不会仅因为调用通知函数而成为可绑定数据。
- 普通属性通过所属 UObject 和相对于该对象的成员名称或属性路径通知。
- Array 和 Map 通过实际集合实例通知；调用方不需要再次提供所属 UObject、成员名称或完整绑定路径。集合与属性路径之间的关系在页面读取该集合时建立。

#### 普通属性函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyPropertyChanged(Object, MemberName) | 通知普通属性或属性路径变化。 |

#### Array 函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyArrayAdded(Array) | 在末尾添加一个元素后调用；末尾索引由通知函数取得。 |
| MarkupUI::NotifyArrayAppended(Array, AppendCount) | 在末尾追加多个元素后调用；`AppendCount` 是本次新增的元素数量。 |
| MarkupUI::NotifyArrayInserted(Array, Index, Count) | 在指定索引插入一个或多个元素后调用。 |
| MarkupUI::NotifyBeginArrayDelete(Array, Index, Count) | 删除指定范围前调用。 |
| MarkupUI::NotifyEndArrayDelete(Array) | 删除完成后调用。 |
| MarkupUI::NotifyBeginArrayReplace(Array, Index, Count) | 替换指定范围前调用。 |
| MarkupUI::NotifyEndArrayReplace(Array) | 替换完成后调用。 |
| MarkupUI::NotifyArrayChanged(Array) | 清空、排序、交换、反转、打乱或无法准确描述的整体变化完成后调用。 |

#### Map 函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyMapAdded(Map, Key) | 新增指定键后调用。 |
| MarkupUI::NotifyMapReplaced(Map, Key) | 替换指定键对应的值后调用。 |
| MarkupUI::NotifyMapRemoved(Map, Key) | 移除指定键后调用。 |
| MarkupUI::NotifyMapChanged(Map) | 清空或无法准确描述的整体变化完成后调用。 |

**注意**

- 单步通知函数（没有 `Begin` / `End`）在集合更新完成后调用。
- 两步通知函数（`Begin` / `End`）必须成对出现：修改集合前调用 `Begin`，修改完成后调用 `End`。
- 集合实例只有在被页面成功读取后才会建立观察关系。未被任何活动数据上下文读取的集合不会产生页面刷新，也不需要从集合反向搜索所属 Object。
- 同一个集合可能被多个模型或多个文档观察；一次集合通知会送达所有仍然有效的观察关系。
- 语言无关的数据绑定层必须保留集合通知中的操作种类、索引、数量和 Map Key，不得因为某个前端当前只支持整体验证就提前丢失这些信息。
- 具体前端的 Bridge 在提交刷新时，才根据目标语言真实具备的刷新粒度决定是否将细粒度集合变化收拢为整个集合或顶层变量变化。该限制不得反向污染公共通知 API 和语言无关层。
- 集合地址仅用于定位候选观察关系，不代表集合的所有权，也不会被保存后直接解引用。通知派发前必须依据弱宿主对象、已读取属性路径和当前集合地址重新验证身份，防止容器移动或内存地址复用造成误通知。
- 通知函数可以从任意线程调用，但这不表示容器修改本身自动具备线程安全性；调用方仍须保证集合写入符合其业务线程模型。

例如，已有业务接口修改受保护属性后，可直接通知：

```cpp
void UpdateGoldThroughBusinessApi(
    UMarkupInventoryData* InventoryData,
    int32 NewGold)
{
    if (!InventoryData)
    {
        return;
    }

    InventoryData->SetGold(NewGold);
    MarkupUI::NotifyPropertyChanged(
        InventoryData,
        GET_MEMBER_NAME_CHECKED(UMarkupInventoryData, Gold));
}
```

删除受保护数组中的元素时，先由业务接口完成删除前准备，再在删除后提交：

```cpp
void RemoveInventoryItemThroughBusinessApi(
    UMarkupInventoryData* InventoryData,
    int32 Index)
{
    if (!InventoryData)
    {
        return;
    }

    const TArray<FName>& InventoryItems = InventoryData->GetInventoryItems();
    if (!InventoryItems.IsValidIndex(Index))
    {
        return;
    }
    MarkupUI::NotifyBeginArrayDelete(InventoryItems, Index);

    InventoryData->RemoveInventoryItemAt(Index);

    MarkupUI::NotifyEndArrayDelete(InventoryItems);
}
```

集合通知函数与容器操作相互独立：业务代码先执行修改，再按前述调用时机报告变化。通知会保留新增、追加、插入、删除、替换等集合变更明细。当前 RML Bridge 可以根据 RmlUi 的实际刷新能力收拢这些变化；将来其他前端支持更细粒度的集合更新时，仍可使用同一份变更记录，不要求游戏代码改写。

受保护的 Array 和 Map 属性仍然可以使用集合实例通知。属性本身必须是满足绑定可见性要求的 `UPROPERTY`；C++ Getter 可以向业务代码返回集合引用，调用方修改该引用后，直接将同一个集合实例传给通知函数：

```cpp
UCLASS()
class UInventoryData final : public UObject
{
    GENERATED_BODY()

public:
    TArray<FName>& GetItems() { return Items; }
    TMap<FName, int32>& GetItemCounts() { return ItemCounts; }

protected:
    UPROPERTY(BlueprintReadOnly, Category = "MarkupUI|DataBinding")
    TArray<FName> Items;

    UPROPERTY(BlueprintReadOnly, Category = "MarkupUI|DataBinding")
    TMap<FName, int32> ItemCounts;
};
```

此时虽然我们无法访问受保护属性 `Items` 和 `ItemCounts`， 仍然可以用值引用来通知属性变化。

```cpp
void AddInventoryItem(UInventoryData& InventoryData, const FName& ItemName)
{
    TArray<FName>& Items = InventoryData.GetItems();
    Items.Add(ItemName);
    MarkupUI::NotifyArrayAdded(Items);
}

void RemoveInventoryItem(UInventoryData& InventoryData, int32 Index)
{
    TArray<FName>& Items = InventoryData.GetItems();
    if (!Items.IsValidIndex(Index))
    {
        return;
    }

    MarkupUI::NotifyBeginArrayDelete(Items, Index);
    Items.RemoveAt(Index);
    MarkupUI::NotifyEndArrayDelete(Items);
}


void SetInventoryItemCount(UInventoryData& InventoryData, const FName& ItemName, int32 Count)
{
    TMap<FName, int32>& ItemsCountFromGet = InventoryData.GetItemCounts();
    const bool bAlreadyExists = ItemsCountFromGet.Contains(ItemName);
    ItemsCountFromGet.Add(ItemName, Count);

    if (bAlreadyExists)
    {
        MarkupUI::NotifyMapReplaced(ItemsCountFromGet, ItemName);
    }
    else
    {
        MarkupUI::NotifyMapAdded(ItemsCountFromGet, ItemName);
    }
}
```

Getter 的可见性只影响 C++ 调用方式，不改变绑定前提：`Items` 和 `ItemCounts` 仍是反射属性。局部变量或普通非 `UPROPERTY` 容器即使也能传入同名通知函数，也没有可供页面匹配的观察关系，因此不会触发数据绑定刷新。

## Slate（C++）使用草案

以下沿用前文 `FInventoryUiState`，`InventoryRml` 是已取得的文档内容；资源域沿用 Widget 的资源上下文，`/Inventory.rml` 是该资源根下的逻辑 URI。

Slate 使用者创建 `FMarkupObservableObject` 派生模型，并直接挂载到 Widget。

```cpp
TSharedRef<SRmlUiWidget> Ui = SNew(SRmlUiWidget)
    .DocumentContent(InventoryRml)
    .DocumentSourceUri(TEXT("/Inventory.rml"));

TSharedRef<FInventoryUiState> InventoryState = MakeShared<FInventoryUiState>();
Ui->SetDataContext(
    TEXT("inventory"),
    InventoryState);
```

每个 `SRmlUiWidget` 的文档必须拥有单独的 Rml 模型实例。多个文档可以引用同一个业务数据对象，但不能共享 RmlUi 内部的模型 handle。

## UMG（蓝图）使用草案

以下是未来 UMG 包装的设计目标，`URmlUiWidget` 不是当前已经提供的公开控件。Widget Blueprint 或拥有者在构造、初始化时取得实际业务 `UObject`，并在文档上调用具名 `Set Data Context`。

推荐工作流：

1. 取得当前角色、组件或其他业务 `UObject`。
2. 使用与 RML `data-model` 相同的名称将该对象设置给文档；可以在文档加载前或加载后调用。
3. 写入普通属性时使用 `Set <属性名> (Markup UI)`；修改数组或 Map 时使用对应的 `(Markup UI)` 集合操作节点。
4. 业务对象通过原生 Set 或 C++ 修改属性后，显式发送通知；多次通知自动合并，不设计另一套更新作用域。
5. 目标对象销毁后，同名模型槽自动回到空状态，无需手动保活；页面读取安全返回无值。

Designer 预览使用安全的预览对象，不得保存设置、触发命令或修改游戏世界。蓝图用户不需要通过 JSON 字符串填充数据；属性、数组和 Map 都沿用原始 `UObject` 属性的实际类型。

普通属性、数组和 Map 节点的完整清单及其通知行为见“蓝图支持”中的对应小节。节点基于实际 `UObject` 的反射属性生成；从对象的属性引脚或右键菜单进入，分类、名称和引脚规则均与对应原生节点一致。

## 生命周期、重载与线程

对使用者可见的生命周期是：

```text
创建控件并取得文档配置
  → 加载 RML 文档并为静态 data-model 建立模型槽
  → 尚未挂载 Object 的模型槽以空值参与绑定
  → 取得业务 UObject
  → SetDataContext(Name, Object) 填充或替换同名模型槽
  → 游戏状态与用户输入双向同步
  → 文档重载、控件销毁或 UObject 销毁时，旧绑定和命令失效
```

`SetDataContext(Name, Object)` 与文档加载没有强制先后顺序。页面先加载时，缺少业务对象的模型名由空模型槽承接；对象随后到达时，填充同名槽并在下一次更新重新读取。对象先到达时，槽可以先保存 Adapter，页面加载后直接使用。文档重载会重建 RmlUi 元素绑定，但文档包装层保留仍然有效的具名模型槽；旧元素回调和旧控件输入状态均失效。若需要保留搜索词、滚动位置或草稿，应由游戏业务对象保存这些值。

自动数据读取、输入处理和命令回调在界面的执行线程中串行发生，不允许同一份活动文档被多个线程同时求值。异步加载、网络回调或后台任务可以提交数据上下文或文档变更请求，但请求必须先转交给界面的执行线程，不能从后台线程直接进入文档求值。业务对象和可观察集合若允许后台写入，仍由业务层保证写入与界面读取不会并发，并将变化通知安全地交给界面。

Frontend 不设置统一的读取生命周期管理器。每个 Adapter 都独立负责解析自己的弱数据源，并在每次同步接口调用开始时重新检查它：

- 面向原生 C++ 模型的 Adapter 保存 `TWeakPtr<MarkupUI::DataBinding::IObservableObject>`。读取函数开始时调用 `Pin()`，将得到的局部 `TSharedPtr` 保持到本次函数返回，然后立即释放。
- 面向 `UObject` 的反射 Adapter 保存 `TWeakObjectPtr<UObject>`。它在游戏线程内检查对象仍然有效后立即完成本次反射访问，不缓存解析出的 `UObject*`，也不把 `UObject` 转换为 `MarkupUI::DataBinding::IObservableObject`。
- Scalar、Object、Array 和 Map Adapter 遵循同一规则，但不会共同登记到一个跨调用保活集合中。一个 Adapter 读取成功，不代表下一次调用或另一个 Adapter 的读取仍然成功。
- 弱引用已经失效时，本次操作直接返回无值或未处理，当前求值路径停止，页面使用安全默认值。
- `MarkupUI::DataBinding::Frontend::FValue` 只通过 `TSharedPtr<IValue>` 保持 Adapter 自身有效。它不会因此保活业务对象，也不缓存 `UObject*`、原生对象裸指针或之前读取到的属性值。
- Scalar 返回的当前值必须在同一次同步调用中被消费。String、Text、Name 的读写统一使用 `const FString*`，不另传长度；开发者不需要提供 UTF-8 数据。返回值在下一次读取或 Scalar 释放前有效。UTF-8 转换只属于外部 API 的适配边界。

### 命令回调中的生命周期安全

命令回调与自动数据读取必须分开理解。自动数据读取只负责同步取得当前值，Getter 必须保持只读，不得关闭文档、重载界面或结束宿主视图的生命周期。业务对象已经失效时，本次读取返回无值；正常读取本身不需要延迟。

命令可以同步改变业务状态和文档内容，但可能让当前求值所依赖的文档或宿主视图失效的操作只能作为请求提交。数据读取和命令胶水只负责识别并投递这类请求，不在每次回调外建立通用生命周期 Scope。安全规则如下：

- 正常输入、更新、绘制和数据读取沿既有前端调用链同步完成，不因进入某个回调而统一切换成延迟模式。
- 回调期间提出的关闭、替换或完整重载文档请求不会立即执行，也不会递归进入新的文档更新；胶水层将其投递给宿主目标的延迟队列。
- 延迟请求只在宿主目标后续的常规 Tick 中消费；执行这一批挂起操作时，目标自身由执行作用域保持有效。
- 执行作用域面向抽象宿主目标，不依赖某一种窗口或控件实现，也不会散布在输入、数据读取或命令回调路径中。
- 从非界面线程提出的宿主变更请求只负责排队，真正的文档和数据模型操作统一在界面的执行线程完成。
- 宿主视图已经释放时，尚未执行的请求全部作废，不得为了完成旧请求重新创建或长期保活界面。
- 排队成功只表示请求已被接受，不代表文档加载、样式解析或数据上下文更新已经成功；最终结果必须由对应操作的完成状态或错误报告表达。

**命令回调期间允许：**

- 修改业务对象的普通属性，并按属性契约执行校验、钳制或联动更新。
- 通过可观察集合的受管接口新增、删除、替换或重排元素，并发出对应变化通知。
- 将一个或多个模型字段标记为 Dirty；文档在后续常规更新中只重新读取关联的脏数据。
- 使用受支持的 DOM 操作新增、删除或修改元素。
- 发出关闭、替换、完整重载文档或重新加载样式的请求；这些请求在后续目标 Tick 中执行。
- 发出设置或替换具名数据上下文的请求；它不在当前求值栈中立即改变数据来源。
- 请求从父界面移除当前宿主视图；当前回调仍保持有效，实际生命周期在回调返回后结束。

**命令回调期间不允许：**

- 立即销毁当前回调依赖的文档运行环境或宿主视图。
- 在当前文档尚未返回时递归执行新的顶层更新、绘制或输入处理。更新请求只能触发后续常规更新。
- 通过销毁并重建整个文档运行环境来完成普通的文档重载。
- 绕开宿主界面的共享生命周期管理，强制释放当前正在执行回调的对象。
- 缓存并在回调结束后继续使用事件参数、借用元素、临时字符串视图或其他仅在本次回调中有效的数据。
- 删除当前正在使用的稳定模型槽。移除业务对象时只清空槽中的数据来源，使现有绑定安全地读取为空值。

### 延迟请求与文档版本

延迟请求必须区分“针对某一份文档的操作”和“与具体文档无关的操作”。每个宿主视图维护单调递增的文档版本，用于判断请求是否仍有执行意义：

- 关闭文档、替换文档和完整重载文档在请求提出时推进文档版本。
- 重新加载样式不会推进文档版本，但它记录提出请求时的版本；若执行前文档已经变化，该样式请求作废。
- 设置或替换数据上下文不绑定到某个文档版本。稳定模型槽可以跨文档重载继续存在，因此此类请求保持原有顺序执行。
- 没有文档语义的普通延迟操作既不参与文档版本判断，也不与其他普通操作自动合并。
- 同一语义的重复文档请求可以让旧请求失效，但不得把不同数据上下文名称或不同业务操作误合并。

开始处理一批延迟请求前，系统只读取一次当前文档版本，并先以该快照排除针对旧文档的请求。筛选完成后再执行本批请求；本批执行过程中产生的新版本和新请求属于下一批，不反向改变已经完成的筛选结果。

普通字段写入、集合通知和 Dirty 标记仍遵循同步数据模型语义。安全设计只延迟可能破坏当前求值环境的宿主操作，不应把所有 Getter、Setter、集合修改或命令统一异步化，也不应通过回调 Scope 推断操作是否危险。

## 错误、调试与安全

开发环境应提供绑定检查器，按界面实例显示：数据上下文、公开字段、读写权限、字段类型、最近刷新字段、最近的输入拒绝原因和最近命令。Shipping 不显示实际敏感数据。

日志或检查器中的错误必须包含文档标识、数据上下文和字段/命令名。至少覆盖：

- 已挂载业务 Object 后，RML 使用了该 Object 不存在或不允许绑定的字段或命令；
- RML 尝试写不允许写入的字段；
- 值转换、范围校验、枚举校验或 Setter 拒绝；
- `UFUNCTION` 参数不符合命令契约；
- 不支持的反射类型或循环引用；
- 数据上下文已失效、目标对象已销毁，或业务数据在界面读取期间被其他线程并发修改；
- 模型名为空或非法。

空模型槽是正常生命周期状态，不记作“缺少数据上下文”错误，也不因其中的字段读取输出警告。业务 Object 已经挂载后，开发环境推荐把“RML 引用了该 Object 不存在的字段”作为明显警告，并以安全默认值显示；Shipping 保持安全默认值且不暴露对象信息。是否让后者在严格模式下直接阻止文档加载，应作为项目设置提供。

## 自动化验收清单

这是一份目标清单，不是已通过测试的报告；UMG、正式蓝图节点、访问控制和输入冲突策略应在相应功能实施后验收。

- 同一份 RML 在 Slate 与 UMG 中，对同一模型数据得到一致的文字、属性、可见性和列表内容。
- 当前 `TMarkupProperty<T>`、两参数 `RegisterProperty`、`SetValue`、嵌套对象与集合通知、命令签名按真实公开用法验收，不再要求复活旧属性宏或 Getter／Setter 注册重载。
- 标量、对象、嵌套结构、数组、枚举、`FText` 与受限映射的转换正确；不支持的值明确失败。
- `UObject` 数据上下文能通过字段通知或手动 `Notify Markup Property Changed` 刷新正确的属性；数组成员变化要求通知数组属性。
- `Set Data Context` 接受模型名以及角色、组件或其他业务 `UObject`；不创建代理对象、镜像属性或第二份需要维护的蓝图资产。
- `Set <属性名> (Markup UI)` 与原生 Set Var 在标题、输入/输出顺序、类型颜色、属性分类和连接方式上保持一致；选择器名称与右上角标识可明确区分它。
- 数组与 Map 的 `(Markup UI)` 修改节点尊重原生函数的名称、分类、默认输入、泛型推导和引用方向；只接受同一 `UObject` 属性的 Get 或 Set 输出，并在完成后只通知该属性。
- 同一 `UObject` 可安全登记到多个文档或同一文档的多个模型名；目标对象销毁后，所有关联绑定安全失效且不保活该对象。
- `data-value`、`data-checked` 覆盖正常写入、类型错误、范围修正、业务拒绝和外部更新冲突。
- 输入法合成、文本选择与焦点期间不被外部刷新错误打断。
- 文档先加载时，未挂载 Object 的 `data-model` 使用空模型槽且读取安全返回无值；随后设置同名 Object 能在下一次更新恢复全部关联绑定。
- 文档的具名 `SetDataContext` 覆盖、同名 Object 替换、加载前预设和卸载后解除订阅均符合预期；替换一个模型不影响其他模型。
- Object 数组属性与 `data-for` 覆盖条目变更、插入、删除、移动、文档重载和滚动状态；C++ 不得手工修改其受管子树。
- 严格模式拒绝空模型名、非法模型名以及向失效文档设置数据上下文，但允许在文档加载后首次设置合法名称。
- C++ 回调与 `UFUNCTION` 命令均覆盖参数成功、参数不匹配、目标销毁和重复点击。
- 文档重载、控件销毁、多个 Widget、多本地玩家不会串用模型、输入状态或命令。
- UMG Designer 不触发命令或写回真实游戏对象。

## 后续实施与复核

1. **现有原生模型**：以 `TMarkupProperty<T>`、实例注册、属性／集合通知和共享 Owner 命令为基准，核查功能；不把旧宏或旧 Store 示例当作缺口。
2. **具名上下文与反射模型**：复核同名替换、清除、弱引用、混合路径与多模型通知；当前入口为 `SRmlUiWidget::SetDataContext`。
3. **写回体验**：单独确认业务校验、访问控制、拒绝恢复和输入冲突是否需要新增能力。
4. **蓝图与 UMG**：区分现有原型和正式对外功能，再安排节点、控件与预览验收。
5. **Map 与其他扩展**：完整 RML Map 语法、局部数据上下文等单独讨论，不从通用数据层的能力推导为前端已支持。

## 尚待产品确认的选择

- 哪些 `UPROPERTY` 与 `UFUNCTION` 可由反射暴露给页面？建议采用明确的 MarkupUI 元数据或白名单；运行时绝不自动暴露目标对象的全部成员。
- 数值超出范围时，默认修正还是默认拒绝？建议由字段声明决定：设置类字段通常修正，交易、权限和数量类字段通常由 Setter 决定。
- 严格模式下，不存在或未允许的字段是否阻止文档加载？建议开发/测试环境可配置为阻止，Shipping 使用安全默认值并记录汇总诊断。
- 是否提供每键同步的蓝图便捷节点？建议只提供受节流的专用命令，不把它作为普通 `data-value` 的默认行为。

## 混合可观测对象

同一份数据上下文对象图可以混合使用原生 C++ 可观察对象和 `UObject`，但首版只支持能够通过各自公开属性契约发现的组合：

| 父对象 | 子属性 | 首版支持 |
| --- | --- | --- |
| 原生 C++ 可观察对象 | 原生 C++ 可观察对象 | 支持 |
| 原生 C++ 可观察对象 | `UObject` | 支持 |
| `UObject` | `UObject` | 支持 |
| `UObject` | 原生 C++ 可观察对象 | 暂不支持 |

原生 C++ 可观察对象一旦作为数据上下文或已登记属性参与绑定，其自身的属性变化通知即可继续驱动页面更新，不需要通过 UE 反射观察这套通知。

原生 C++ 可观察对象通过 `TWeakObjectPtr<T>` 或 `TStrongObjectPtr<T>` 登记 UObject 子属性。两种引用都在每次读取时重新检查：弱引用不会保活 UObject，强引用明确表示业务模型需要保持该对象存活。普通 `UObject*` 和 `TObjectPtr<T>` 不属于这条原生属性契约，避免在非 UObject、非 GC 扫描的数据模型中形成含义不明确的对象所有权。

同样的规则适用于原生可观察对象的直接属性、Array 元素和 Map Value。路径进入 UObject 后由反射属性规则继续解析，并订阅该 UObject 的属性变化通知；路径不会把 UObject 转换成原生可观察对象，也不会把对象指针值传递给 RML。

首版不尝试从任意 `UObject` 中发现由普通 C++ 字段、`TSharedPtr` 或其他非反射成员持有的原生 C++ 可观察对象。需要这种方向的混合关系时，应先调整公开的数据上下文结构；后续可再评估通过显式属性访问入口开放，而不要求 `UObject` 实现 MarkupUI 协议。

### 集合元素支持范围

以下表格**仅针对 TArray 元素和 TMap 的 Value**。UObject 侧集合须为可见的 `UPROPERTY`；ObservableObject 侧集合须登记为 Property。ObservableObject 侧同样适用于 `TMarkupObservableArray` 和 `TMarkupObservableMap`。Map 键限定为 `FString`、`FName`、`int32`，页面表达式仍受当前 RML Map 能力限制。

#### 元素类型支持对比

| 元素 / Value 类型 | UObject | ObservableObject |
| --- | --- | --- |
| `bool`、`int32`、`int64`、`float`、`double` | ✅ | ✅ |
| `FString`、`FText`、`FName` | ✅ | ✅ |
| UE 反射枚举 | ✅ | ✅ |
| 普通 C++ 枚举 | ❌ | ✅ |
| `FLinearColor`、`FVector2D` | ✅ | ✅ |
| `FVector3f`、`FVector` | ✅ | ✅ |
| `UObject*` | ✅ | ❌ |
| `TObjectPtr<UObject>` | ✅ | ❌ |
| `TWeakObjectPtr<UObject>` | ✅ | ✅ |
| `TStrongObjectPtr<UObject>` | ❌ | ✅ |
| `TSharedPtr<IObservableObject>` | ❌ | ✅ |
| USTRUCT | ✅ | ✅ |
| `TSharedPtr<USTRUCT>` | ❌ | ✅ |
| `TWeakPtr<USTRUCT>` | ❌ | ✅ |

#### 集合操作支持对比

| 集合操作 | UObject | ObservableObject |
| --- | --- | --- |
| 读取数量与元素 | ✅ | ✅ |
| 读取对象元素的属性 | ✅ | ✅ |
| 读取 USTRUCT 元素的成员 | ✅ | ✅ |
| 访问 USTRUCT 内的 Array / Map | ✅ | ✅ |
| 前端直接写回标量元素 | ✅ | ✅ |
| 前端写回对象元素的标量属性 | ✅ | ✅ |
| 前端写回 USTRUCT 元素的标量成员 | ✅ | ✅ |
| 普通集合显式变化通知 | ✅ | ✅ |

USTRUCT 值及上述两种 USTRUCT 智能指针也可作为原生可观察对象的直接登记属性。结构体成员遵循相同的反射可见性要求，使用 `UPROPERTY(BlueprintReadOnly)` 或 `UPROPERTY(BlueprintReadWrite)` 暴露需要绑定的成员；结构体内可以继续包含受支持的 UObject、USTRUCT、Array 和 Map 成员。

- `TSharedPtr<FMyStruct>` 由业务属性或集合保持结构体存活；`TWeakPtr<FMyStruct>` 不保活结构体，失效后读取返回无值。
- 结构体没有独立的属性通知身份。修改结构体成员后，通过所属可观察对象发送属性路径通知，或通过所在集合发送对应变更通知。
- 弱引用失效本身不会自动发送通知。业务若要求页面及时显示失效结果，仍需发出所属属性或集合的通知。
- 原生结构体及其 `TSharedPtr` 不会自动让结构体中的 UObject 强引用参与 GC；需要由业务另外保证 UObject 生命周期，或使用可检测失效的 UObject 弱引用。

## 参考资料

- [RmlUi：数据绑定](https://mikke89.github.io/RmlUiDoc/pages/data_bindings.html)
- [RmlUi：元素与 DOM 接口](https://mikke89.github.io/RmlUiDoc/pages/cpp_manual/elements.html)
- [Noesis UI：数据绑定](https://www.noesisengine.com/docs/Gui.Core.DataBindingTutorial.html)
