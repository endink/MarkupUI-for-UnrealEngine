# UE 数据绑定设计草案

> 状态：讨论草案。本文描述计划提供的公开能力、用法与验收标准；名称均为拟议名称，可在实现前调整。本文不假定读者了解 RML 或 RmlUi 的源码。

## 一句话目标

让 RML 界面把 UE 对象中的、**明确授权公开**的数据读出来显示；让输入框、复选框等控件把**已验证**的结果安全写回；让按钮等交互以“命令”或 `UFUNCTION` 的形式通知游戏逻辑。无论界面放在 Slate（C++）还是 UMG（蓝图）中，RML 写法和结果都一致。

例如，角色金币变化后，`{{ gold }}` 自动刷新；玩家拖动音量滑条后，游戏设置中的音量被限制在合法范围并写回；玩家点击物品后，游戏收到“选择物品”的命令及物品 ID，而不是让 RML 任意调用游戏对象。

## 设计起点：C++ 原生优先，蓝图只是包装

数据绑定的第一步必须是可独立使用的 C++ 原生契约：它不要求 `UObject`、反射、UMG 或蓝图。Slate、游戏模块和任何普通 C++ 数据结构都能通过它创建模型、提供值、接受写入并注册命令。

第二步才是反射适配：把已白名单标记的 `UPROPERTY` 和 `UFUNCTION` 翻译成同一份 C++ 模型定义。第三步才是 UMG 与蓝图节点：它们调用相同的字段、写入和命令 API，不拥有另一套 RML 语法、刷新时序或验证规则。

```text
C++ 原生模型、值、字段、写入、命令契约
                ↑                 ↑
      Slate / 普通 C++      UE 反射适配层
                                      ↑
                              UMG 与蓝图节点
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
| 属性 Getter/Setter + 属性变化通知 | `FMarkupObservableObject` 与 `MARKUP_*` 属性宏。 |
| 可观察集合 | `FMarkupObservableArray`；任何增删改排都会通知字段路径，并由绑定策略决定标记顶层字段或局部更新条目。 |
| 数据上下文 | RML 的 `data-model`。一个文档实例在模型名下只看到自己的数据上下文。 |
| 单向、双向等绑定模式 | 由字段权限决定：只读字段为单向，可写字段允许双向；不引入与 RML 原生语法冲突的另一套标记。 |
| 每次变化、失焦、显式提交 | 保持 RmlUi 控件原有 `change` 语义为默认；高频或延迟提交用明确命令实现，不隐式改变控件行为。 |

最重要的差异是刷新粒度。RmlUi 原生数据模型只接受顶层变量的脏标记；因此只使用原生 `data-model` 时，`Player.Name.First` 变化必须通知最外层字段 `player`。但 RmlUi 也公开了完整的元素 DOM API，MarkupUI 可以在原生模型之外提供一层**显式局部 DOM 刷新**：将一个字段绑定到作者指定的元素目标，仅改写该目标的文本、属性、样式、类或子树。这个扩展能提供细粒度观察，但它必须与原生数据视图分工，不能同时争夺同一元素。

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

绝不能因为某个 `UObject` 被绑定，就让 RML 自动读取其全部 `UPROPERTY` 或执行其任意 `UFUNCTION`。每一项暴露都必须是显式的白名单。

## 术语

| 术语 | 含义 |
| --- | --- |
| 模型 | RML 用来读写的一组命名数据，例如 `inventory`。一份正在显示的文档拥有自己的模型实例。 |
| 字段 | 模型中的顶层数据，例如 `gold`、`items`、`settings`。RML 可读取 `items[0].name`；原生数据模型以顶层字段刷新，局部 DOM 绑定可针对路径更新指定元素。 |
| 路径 | 在 RML 中访问数据的写法，例如 `settings.music_volume`。 |
| 只读字段 | RML 可以显示但不能通过输入控件写回的字段。 |
| 可写字段 | RML 可以通过指定控件写回的字段；每次写入均须类型转换和校验。 |
| 命令 | RML 发出的具名交互意图，例如“选择物品”“开始游戏”。它有固定的参数契约，由 C++ 回调或公开的 `UFUNCTION` 接收。 |
| 脏标记 | 表示某个字段已变化、下一次界面更新应重新读取它的信号。 |

## 支持范围

第一阶段应支持以下 RML 标准写法：

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
  <input type="checkbox" data-checked="settings.show_minimap"/>
</div>
```

| RML 写法 | 计划 | 使用者可观察到的行为 |
| --- | --- | --- |
| `{{ ... }}`、`data-attr-*`、`data-class-*`、`data-style-*` | 支持 | 读取字段和表达式结果。 |
| `data-if`、`data-visible`、`data-alias-*` | 支持 | 控制显示、可见性和模板作用域。 |
| `data-for` | 支持 | 用数组生成列表；数组变化后列表自动更新。 |
| `data-value`、`data-checked` | 支持 | 对已声明为可写的标量执行双向绑定。 |
| `data-event-*` | 支持 | 发送受控的命令，可绑定 C++ 回调或 `UFUNCTION`。 |
| 自定义转换函数 | 支持 | 格式化展示值；必须是无副作用的纯计算。 |
| `data-rml` | 延后 | 它可改变文档内容，需单独确定安全和重载规则。 |

`data-*` 和 `{{ ... }}` 是 RML 的保留语法，必须在文档加载前写好。运行中给现有元素动态添加这些属性，不应期待它们变成绑定。

## C++ 原生方案

### 统一的运行时绑定对象：`FMarkupObservableObject`

原生 C++ 数据模型优先继承 `FMarkupObservableObject`，并通过 `MARKUP_*` 宏声明属性、嵌套对象、数组、只读字段、业务 Setter 和命令。

`IMarkupObservableObject` 是可观察对象接口；`FMarkupObservableObject` 是唯一推荐的基础实现。它维护访问器/命令字典和属性变化事件，根对象挂载到模型名后，RML 自动从该字典读取、写入并订阅变化。

`UObject` 也可直接绑定，不要求继承任何 MarkupUI 基类。反射/蓝图路径会为每个绑定上下文生成 `FMarkupBlueprintObject`：它继承 `FMarkupObservableObject`，以弱引用持有目标 `UObject`，并将明确公开的属性、Setter 与 `UFUNCTION` 命令映射为访问器。Slate、UMG、反射和蓝图最终消费完全相同的访问器字典、写入结果和变化事件。

### C++ 数据模型宏：把样板代码变成明确契约

为了让 C++ 数据模型能够用简洁、声明式的方式定义属性，同时保留 UE 所需的权限、局部 DOM 和命令能力，应提供一组只依赖 C++ 的模型宏。它们不要求 `UObject`，也不依赖 `UPROPERTY`；因此可用于 Slate、子系统、普通游戏状态和自动化测试。

属性声明和构造函数登记处于两个不同的 C++ 作用域，因此采用两条明确、普通的宏语句：属性宏在类作用域生成值成员和 Getter/Setter；访问宏在构造函数中登记对应的访问器。没有隐藏登记成员、没有额外属性清单，也不需要 Begin/End 类宏。

属性名在 `MARKUP_PROPERTY` 与 `MARKUP_PROPERTY_ACCESS` 中各出现一次。这是刻意保留的、可审查的显式关联：开发者可以一眼看出哪个字段被公开给 UI，构造函数也与完全手写 `RegisterAccessor` 的代码一一对应。

`FMarkupObservableObject` 维护“名称 → 访问器”字典。`MARKUP_PROPERTY_ACCESS(Name)` 表示读写；`MARKUP_PROPERTY_ACCESS(Name, ReadOnly)` 表示只读；`MARKUP_PROPERTY_ACCESS(Name, WriteOnly)` 表示只写。对象和数组仍由各自的访问宏登记。

基类对模型和高级 C++ 使用者公开统一查询接口：

```cpp
class FMarkupObservableObject : public IMarkupObservableObject
{
public:
    const FMarkupAccessor* FindAccessor(FName Name) const;
    const TMap<FName, FMarkupAccessor>& GetAccessors() const;
    FMarkupPropertyChangedEvent& OnPropertyChanged();

protected:
    void RegisterAccessor(FName Name, FMarkupAccessor Accessor);
    void RegisterCommand(FName Name, FMarkupCommand Command);
    void NotifyPropertyChanged(FStringView Path);
};
```

`FMarkupAccessor` 统一保存读取函数、可选写入函数、值类型与访问权限；`FMarkupPropertyChangedEvent` 发送变化路径。模型绑定、局部 DOM 刷新、反射适配和蓝图包装都只通过这些接口工作，不需要了解派生类的成员布局。

根对象挂到具名模型时，插件从字典递归取得访问器：`player` 是模型名，`Name` 是根对象中的对象访问器，`First` 是其子对象中的属性访问器。因此 RML 在 `data-model="player"` 内使用 `{{ Name.First }}`、`{{ Weight }}` 和 `{{ PlayerId }}`。若内容规范要求小写名称，使用额外的命名变体宏即可；默认名称始终与 C++ 属性名一致。

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        MARKUP_PROPERTY_ACCESS(First)
        MARKUP_PROPERTY_ACCESS(Last)
    }

    MARKUP_PROPERTY(FString, First)
    MARKUP_PROPERTY(FString, Last)
};

class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        MARKUP_OBJECT_PROPERTY_ACCESS(Name)
        MARKUP_PROPERTY_ACCESS(Weight)
        MARKUP_PROPERTY_ACCESS(PlayerId, ReadOnly)
    }

    MARKUP_OBJECT_PROPERTY(TSharedPtr<FNameUiState>, Name)
    MARKUP_PROPERTY_COMPARE(float, Weight, FMath::IsNearlyEqual)
    MARKUP_PROPERTY_GETTER(FString, PlayerId)
};

TSharedRef<FPlayerUiState> Player = MakeShared<FPlayerUiState>();
Ui->SetDataModel(TEXT("player"), Player);
```

#### 普通可观察属性：宏写法与不用宏的对照

开发者在构造函数中显式登记访问器，再在类中声明属性：

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        MARKUP_PROPERTY_ACCESS(First)
        MARKUP_PROPERTY_ACCESS(Last)
    }

    MARKUP_PROPERTY(FString, First)
    MARKUP_PROPERTY(FString, Last)
};
```

不使用宏时，开发者必须手写以下等价 C++：

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        RegisterAccessor(
            TEXT("First"),
            FMarkupAccessor::MakeReadWrite(
                [this]() { return FMarkupValue(FirstValue); },
                [this](const FMarkupValue& InValue)
                {
                    SetFirst(InValue.GetString());
                    return FMarkupWriteResult::Accepted();
                }));

        RegisterAccessor(
            TEXT("Last"),
            FMarkupAccessor::MakeReadWrite(
                [this]() { return FMarkupValue(LastValue); },
                [this](const FMarkupValue& InValue)
                {
                    SetLast(InValue.GetString());
                    return FMarkupWriteResult::Accepted();
                }));
    }

    const FString& GetFirst() const
    {
        return FirstValue;
    }

    void SetFirst(FString InFirst)
    {
        if (FirstValue == InFirst)
        {
            return;
        }

        FirstValue = MoveTemp(InFirst);
        NotifyPropertyChanged(TEXT("First"));
    }

    const FString& GetLast() const
    {
        return LastValue;
    }

    void SetLast(FString InLast)
    {
        if (LastValue == InLast)
        {
            return;
        }

        LastValue = MoveTemp(InLast);
        NotifyPropertyChanged(TEXT("Last"));
    }

private:
    FString FirstValue;
    FString LastValue;
};
```

这里的 `RegisterAccessor` 是完全手写模式不可省略的一步：它把名称、Getter 和 Setter 显式登记到基类字典。`MARKUP_PROPERTY_ACCESS(First)` 是这段登记代码的简写，仍然明确地放在构造函数中。因此 `SetFirst(TEXT("Ada"))`：第一次调用改变值并发出一次通知；第二次传入相同值时不发通知。浮点数默认不应使用 `==`，而应使用带比较器的属性声明：

```cpp
MARKUP_PROPERTY_COMPARE(float, Weight, FMath::IsNearlyEqual)
```

不用宏时，开发者只需将 `FirstValue == InFirst` 替换为指定比较器；Getter、Setter 和通知语义不变。

#### 嵌套对象与数组：宏写法与不用宏的对照

嵌套对象和数组也在构造函数中显式登记访问器：

```cpp
class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        MARKUP_OBJECT_PROPERTY_ACCESS(Name)
        MARKUP_ARRAY_PROPERTY_ACCESS(Items)
    }

    MARKUP_OBJECT_PROPERTY(TSharedPtr<FNameUiState>, Name)
    MARKUP_ARRAY_PROPERTY(FItemView, Items)
};
```

不使用这些宏时，开发者必须手写以下等价 C++。`FMarkupPropertySubscription` 是由基础类提供的可释放观察句柄；它不是元素或 Widget 的所有权。

```cpp
class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        RegisterAccessor(
            TEXT("Name"),
            FMarkupAccessor::MakeObject(
                [this]() -> IMarkupObservableObject*
                {
                    return NameValue.Get();
                }));

        RegisterAccessor(
            TEXT("Weight"),
            FMarkupAccessor::MakeReadWrite(
                [this]() { return FMarkupValue(WeightValue); },
                [this](const FMarkupValue& InValue)
                {
                    SetWeight(InValue.GetFloat());
                    return FMarkupWriteResult::Accepted();
                }));

        RegisterAccessor(
            TEXT("PlayerId"),
            FMarkupAccessor::MakeReadOnly(
                [this]() { return FMarkupValue(PlayerIdValue); }));
    }

    const TSharedPtr<FNameUiState>& GetName() const
    {
        return NameValue;
    }

    void SetName(TSharedPtr<FNameUiState> InName)
    {
        if (NameValue == InName)
        {
            return;
        }

        NameSubscription.Reset();
        NameValue = MoveTemp(InName);

        if (NameValue.IsValid())
        {
            NameSubscription = NameValue->OnPropertyChanged().AddLambda(
                [this](FStringView ChildPath)
                {
                    NotifyPropertyChanged(
                        FString::Printf(TEXT("Name.%s"), *FString(ChildPath)));
                });
        }

        NotifyPropertyChanged(TEXT("Name"));
    }

    float GetWeight() const
    {
        return WeightValue;
    }

    void SetWeight(float InWeight)
    {
        if (FMath::IsNearlyEqual(WeightValue, InWeight))
        {
            return;
        }

        WeightValue = InWeight;
        NotifyPropertyChanged(TEXT("Weight"));
    }

    const FString& GetPlayerId() const
    {
        return PlayerIdValue;
    }

private:
    TSharedPtr<FNameUiState> NameValue;
    float WeightValue = 0.0f;
    FString PlayerIdValue;
    FMarkupPropertySubscription NameSubscription;
};
```

这里同样是完整的手写模式：构造函数显式登记三个访问器；`SetName` 显式订阅和解除订阅子对象的变化。`NameValue->SetFirst(...)` 会转发为 `Name.First`；替换整个对象时通知 `Name`。`PlayerId` 只登记 Getter，因此任何 RML 写入都会被拒绝。`MARKUP_OBJECT_PROPERTY_ACCESS` 与不同模式的 `MARKUP_PROPERTY_ACCESS` 分别简写对应的登记、订阅和通知代码。

`MARKUP_ARRAY_PROPERTY` 配合构造函数中的 `MARKUP_ARRAY_PROPERTY_ACCESS(Items)` 提供受管集合接口。不使用这些宏时，开发者必须手写以下等价 C++；它不暴露可随意改动的 `TArray`，只暴露只读遍历和受管修改：

```cpp
class FInventoryCollectionUiState final : public FMarkupObservableObject
{
public:
    FInventoryCollectionUiState()
    {
        RegisterAccessor(
            TEXT("Items"),
            FMarkupAccessor::MakeArray(
                [this]() -> const FMarkupObservableArray<FItemView>&
                {
                    return ItemsValue;
                }));
    }

    const FMarkupObservableArray<FItemView>& GetItems() const
    {
        return ItemsValue;
    }

    bool AddItem(FItemView Item)
    {
        return ItemsValue.Add(MoveTemp(Item));
    }

    bool RemoveItemByKey(const FString& ItemId)
    {
        return ItemsValue.RemoveByKey(ItemId);
    }

    bool MoveItem(int32 FromIndex, int32 ToIndex)
    {
        return ItemsValue.Move(FromIndex, ToIndex);
    }

private:
    FMarkupObservableArray<FItemView> ItemsValue{*this, TEXT("Items")};
};
```

`FMarkupObservableArray` 负责在 `Add`、`RemoveByKey`、`Move`、`Replace` 和 `Reset` 时发出自身的变化事件；但手写模式仍要显式把它登记为 `Items` 访问器，并订阅后转发到对象的变化事件。`MARKUP_ARRAY_PROPERTY_ACCESS(Items)` 简写这两部分代码。未启用局部列表时，这些通知最终聚合为顶层 `items`；启用稳定键局部列表时，可细化为 `items[物品键]`。

#### 挂载模型

将根对象挂载到模型名后，`FMarkupObservableObject` 的访问器字典提供其属性、对象和数组：

```cpp
TSharedRef<FPlayerUiState> Player = MakeShared<FPlayerUiState>();
Ui->SetDataModel(TEXT("player"), Player);

Player->GetName()->SetFirst(TEXT("Ada"));
Player->SetWeight(72.5f);
```

`MARKUP_OBJECT_PROPERTY` 要求对象本身也是可观察对象；构造函数中的 `MARKUP_OBJECT_PROPERTY_ACCESS(Name)` 负责登记和订阅。替换 `Name` 对象时，它通知 `player.name`；子对象的 `First` 改变时，父对象转发为 `player.name.first`。注册给原生 RmlUi 数据模型时，最终仍聚合为顶层 `player` 的脏标记；注册给局部 DOM 目标时，可以精确命中 `player.name.first`。

`MARKUP_PROPERTY` 的默认比较规则适用于可比较的值类型；浮点数应使用 `MARKUP_PROPERTY_COMPARE(float, Weight, NearlyEqual)` 指定比较器，避免浮点微小误差导致无意义刷新。不可比较的大对象必须使用自定义比较器或显式 `MarkChanged`，不能默认逐字节比较。

### 集合、对象和复杂写入的宏

直接暴露可写 `TArray` 会让调用者绕过通知；模型宏必须只提供受管集合接口：

```cpp
class FInventoryCollectionUiState final : public FMarkupObservableObject
{
public:
    FInventoryCollectionUiState()
    {
        MARKUP_ARRAY_PROPERTY_ACCESS(Items)
        MARKUP_PROPERTY_ACCESS(Gold)
    }

    MARKUP_ARRAY_PROPERTY(FItemView, Items)
    MARKUP_PROPERTY(int32, Gold)

    // 业务函数调用 GetItems().Add、GetItems().RemoveByKey、
    // GetItems().Move 或 GetItems().Replace。
};
```

物品键由 `FItemView` 的稳定 `Id` 成员提供。模型挂载时指定根对象即可：

```cpp
Ui->SetDataModel(TEXT("inventory"), InventoryState);
```

带业务规则的可写属性使用显式登记宏和自定义 Setter：

```cpp
class FSettingsUiState final : public FMarkupObservableObject
{
public:
    FSettingsUiState()
    {
        MARKUP_PROPERTY_ACCESS(MusicVolume, WriteOnly)
    }

    MARKUP_PROPERTY_SETTER(float, MusicVolume, 0.8f, SetMusicVolumeFromUi)

    FMarkupWriteResult SetMusicVolumeFromUi(float RequestedValue)
    {
        const float AcceptedValue = FMath::Clamp(RequestedValue, 0.0f, 1.0f);
        if (!FMath::IsNearlyEqual(GetMusicVolume(), AcceptedValue))
        {
            SetStoredMusicVolume(AcceptedValue);
        }
        ApplyAudioVolume(AcceptedValue);
        return FMarkupWriteResult::Corrected(AcceptedValue);
    }
};
```

`MARKUP_PROPERTY_ACCESS(MusicVolume, WriteOnly)` 在访问器字典中登记 `MusicVolume`，并把来自 RML 的写入交给 `MARKUP_PROPERTY_SETTER` 声明的 `SetMusicVolumeFromUi`。不使用这些宏时，开发者必须手写包含读访问器、写访问器、通知和字典登记的属性：

```cpp
class FSettingsUiState final : public FMarkupObservableObject
{
public:
    FSettingsUiState()
    {
        RegisterAccessor(
            TEXT("MusicVolume"),
            [this]
            {
                return FMarkupValue(MusicVolume);
            },
            [this](const FMarkupValue& Value)
            {
                return SetMusicVolumeFromUi(Value.GetFloat());
            });
    }

    float GetMusicVolume() const
    {
        return MusicVolume;
    }

    FMarkupWriteResult SetMusicVolumeFromUi(float RequestedValue)
    {
        const float AcceptedValue = FMath::Clamp(RequestedValue, 0.0f, 1.0f);
        if (!FMath::IsNearlyEqual(MusicVolume, AcceptedValue))
        {
            MusicVolume = AcceptedValue;
            NotifyPropertyChanged(TEXT("MusicVolume"));
        }
        ApplyAudioVolume(AcceptedValue);
        return FMarkupWriteResult::Corrected(AcceptedValue);
    }

private:
    float MusicVolume = 0.8f;
};
```

`MARKUP_PROPERTY_ACCESS(MusicVolume, WriteOnly)` 在编译期检查 `MARKUP_PROPERTY_SETTER` 指定的 Setter 签名。Setter 必须接收一个可转换值，并返回 `FMarkupWriteResult`；因此它能明确表达接受、拒绝或修正，且不会把任意成员函数误注册为 UI 写入入口。

### 命令宏

命令在构造函数中显式登记，避免字符串名称、参数表和成员函数各写一遍：

```cpp
class FInventoryCommandUiState final : public FMarkupObservableObject
{
public:
    FInventoryCommandUiState()
    {
        MARKUP_COMMAND_ACCESS("select_item", SelectItem)
        MARKUP_COMMAND_ACCESS("request_close", RequestClose)
    }

    void SelectItem(const FString& ItemId)
    {
        SelectedItemId = ItemId;
        NotifyPropertyChanged("SelectedItemId");
    }

    void RequestClose()
    {
        OnCloseRequested.ExecuteIfBound();
    }
};
```

不使用命令宏时，开发者必须手写：

```cpp
class FInventoryCommandUiState final : public FMarkupObservableObject
{
public:
    FInventoryCommandUiState()
    {
        RegisterCommand(
            TEXT("select_item"),
            [this](const TArray<FMarkupValue>& Arguments)
            {
                SelectItem(Arguments[0].GetString());
            });
        RegisterCommand(
            TEXT("request_close"),
            [this](const TArray<FMarkupValue>&)
            {
                RequestClose();
            });
    }

    void SelectItem(const FString& ItemId)
    {
        SelectedItemId = ItemId;
        NotifyPropertyChanged(TEXT("SelectedItemId"));
    }

    void RequestClose()
    {
        OnCloseRequested.ExecuteIfBound();
    }
};
```

`MARKUP_COMMAND_ACCESS` 由成员函数签名推导参数数量和类型，并在编译期拒绝输出参数、非常量引用、`UObject` 指针、潜伏任务和不支持的值类型。反射命令与蓝图命令都必须转换成同一份已验证的命令描述，不能跳过这层检查。

### 宏总表

| 构造函数宏 | 类定义宏 | 解决的问题 | 是否可被反射/蓝图复用 |
| --- | --- | --- | --- |
| `MARKUP_PROPERTY_ACCESS(Name)` | `MARKUP_PROPERTY(Type, Name)` | 声明私有值、Getter、Setter、变更比较和读写访问器。 | 是；反射层可将其描述为普通字段。 |
| `MARKUP_PROPERTY_ACCESS(Name, ReadOnly)` | `MARKUP_PROPERTY_GETTER(Type, Name)` | 声明只读 Getter 属性并登记只读访问器。 | 是。 |
| `MARKUP_OBJECT_PROPERTY_ACCESS(Name)` | `MARKUP_OBJECT_PROPERTY(Type, Name)` | 声明嵌套对象属性，并登记替换追踪和子属性通知。 | 是。 |
| `MARKUP_ARRAY_PROPERTY_ACCESS(Name)` | `MARKUP_ARRAY_PROPERTY(Type, Name)` | 声明受管集合属性，并登记集合变化转发。 | 是。 |
| `MARKUP_PROPERTY_ACCESS(Name, WriteOnly)` | `MARKUP_PROPERTY_SETTER(Type, Name, DefaultValue, Setter)` | 声明带业务 Setter 的值，并登记只写访问器。 | 是；可映射到反射 Setter 或蓝图函数。 |
| `MARKUP_COMMAND_ACCESS(RmlName, Function)` | — | 在构造函数中登记成员函数命令。 | 是；可映射到 `UFUNCTION`。 |

构造函数宏**只能写在派生类自己的构造函数体内**，不能写在普通成员函数、静态函数或类定义体中。类定义宏**只能写在类定义体内**，不能写进构造函数体或任何普通函数体。两类宏刻意分离：前者完成运行时访问器登记，后者声明 C++ 属性或成员接口。

宏应生成可阅读的 C++ 接口，而不是藏起业务逻辑。开发者仍能手写 Getter、Setter、比较器、验证逻辑和命令函数；不能用宏把网络、存档、权限检查或复杂副作用塞进属性赋值。

### 局部 DOM 刷新：突破顶层字段粒度

局部 DOM 刷新是 MarkupUI 在 RmlUi 数据绑定之上提供的第二条更新路径。它不是把任意字段变化都自动反射到整个文档，而是由 C++ 显式声明“某个字段变化时，更新哪个作者命名的元素、更新它的什么部分”。

RML 作者只需为可局部刷新的目标提供稳定、唯一的 `id`：

```html
<div data-model="inventory">
  <span id="hud-gold"></span>

  <div id="player-health" class="health-bar">
    <span id="player-health-label"></span>
  </div>

  <div id="inventory-list"></div>
</div>
```

随后由 C++ 注册目标。下面的名称为伪代码，表达的是公开契约：

```cpp
Model.BindDomText("gold", "hud-gold", [](const FInventoryUiState& State)
{
    return FString::FromInt(State.GetGold());
});

Model.BindDomAttribute("player.health", "player-health", "aria-valuenow",
    [](const FInventoryUiState& State)
    {
        return LexToString(State.GetHealth());
    });

Model.BindDomStyle("player.health", "player-health", "width",
    [](const FInventoryUiState& State)
    {
        return FString::Printf(TEXT("%.1f%%"), State.GetHealthPercent() * 100.0f);
    });

Model.BindDomText("player.health", "player-health-label", FormatHealthLabel);
```

当 `player.health` 改变时，只有上例中四个目标会被更新；金币、背包列表和无关布局不重新求值。文本目标更新文本内容，属性目标调用元素属性 API，样式目标调用元素样式 API；类目标和自定义伪类目标同理。所有 DOM 更改仍在当前文档正常更新周期内生效。

局部刷新使用以下目标类别，避免一个万能的“执行 DOM 脚本”接口：

| 目标类别 | 可改动内容 | 典型用途 |
| --- | --- | --- |
| 文本 | 一个文本节点的内容 | 金币、名称、倒计时。 |
| 属性 | 指定属性的值或存在状态 | 图片地址、无障碍属性、`disabled`。 |
| 样式 | 一个内联 RCSS 属性 | 生命条宽度、颜色、透明度。 |
| 类 / 自定义伪类 | 指定类或由项目定义的伪类开关 | 危险状态、选中状态、冷却完成。 |
| 列表 | 一个受管容器的插入、删除、移动、替换 | 背包、任务、聊天记录。 |
| 安全子树 | 由受管模板生成的子元素 | 少量固定结构的状态面板。 |

默认不提供“把任意字符串传给 `SetInnerRML`”的通用字段绑定。它会重新解析标记，可能破坏焦点、输入状态和元素身份，也可能把未受信任文本当作界面标记。只有显式注册的安全子树模板可以使用结构更新；普通文本一律按文本写入，而不是按 RML 解析。

### 局部列表与稳定身份

`data-for` 是 RmlUi 的原生列表方案；其子元素可被复用，且官方明确不支持在数据模型中手工改变该结构。因此，想获得单条目刷新、插入、删除和移动的列表不能同时把同一容器交给 `data-for` 与 DOM 列表绑定。

局部列表使用单独的空容器与模板呈现器：

```html
<div id="inventory-list"></div>
```

```cpp
Model.BindDomList("items", "inventory-list")
    .Key([](const FItemView& Item) { return Item.Id; })
    .Create(CreateInventoryRow)
    .Update(UpdateInventoryRow)
    .Remove(RemoveInventoryRow);
```

每个条目必须提供稳定且唯一的键，例如物品实例 ID，而不是数组下标。更新器只改动该条目的已命名后代元素；新增、移除和排序只作用于该容器的直接子项。这样 `items[3].count` 变化可以只更新该物品的数量文本，而不会重建其他行或丢失滚动位置。

作者不得把同一个元素同时作为 `data-for` 的结构节点和局部列表容器，也不得在局部列表容器中手动保留外部元素指针。条目被移除、替换或文档重载后，旧元素引用不再有效；公共 API 应使用稳定键和临时元素观察句柄，而不是要求用户保存裸元素指针。

### 与原生 RmlUi 数据绑定的边界

同一元素的同一表现属性只能有一个写入者。下表是硬规则：

| 场景 | 是否允许 | 原因 |
| --- | --- | --- |
| `{{ gold }}` 与局部文本绑定同时写同一文本节点 | 不允许 | 两个绑定会互相覆盖。 |
| `data-class-danger` 与局部类绑定同时管理 `danger` 类 | 不允许 | 类状态没有唯一来源。 |
| `data-for` 与局部列表绑定同一容器 | 不允许 | 两者都会增删、复用子元素。 |
| 原生字段更新一个元素，局部 DOM 更新另一个元素 | 允许 | 目标不重叠。 |
| 局部 DOM 绑定读取可写字段，输入仍用 `data-value` | 允许但须分目标 | 输入元素由 `data-value` 管理，展示元素由局部绑定管理。 |

注册局部目标时，严格模式应验证目标 `id` 存在且唯一，并检查目标是否位于 `data-for` 的受管子树中。冲突会在文档加载时报告，而不是运行时随机覆盖。

### DOM 更新后的时序和限制

RmlUi 的元素 DOM 支持更新属性、样式、类、文本和子树，也支持按 `id` 或选择器检索元素。修改 DOM 后，布局和计算值会在正常的文档更新阶段统一完成；若业务代码立刻读取尺寸、偏移等布局结果，可能读到上一次结果。只有确实需要同步读取布局时，才允许显式更新文档，并应把它视为有性能代价的高级操作。

局部 DOM 刷新不应改写 RmlUi 维护的 `hover`、`active`、`focus`、`checked`、`disabled` 等内置伪类。项目可定义自己的伪类，例如 `low-health`，但表单控件的焦点、选择和可用状态仍由原生控件/绑定机制负责。

### 支持的数据类型与路径

首版属性值支持：布尔值、各种整数、浮点数、字符串、枚举、颜色、二维向量、对象和数组。对象的成员与数组元素通过 `FMarkupValue` 表达；故嵌套深度不依赖 UE 反射。

路径由字段名、对象成员和数组下标构成：

```text
gold
player.display_name
settings.music_volume
items[0].count
```

字段名、成员名、命令名只允许字母、数字和下划线，且以字母或下划线开头。不要把用户输入、资产路径或对象名直接拼进路径。

不会直接暴露 `UObject*`、`UClass*`、组件、世界、Actor、委托、接口对象、软引用或硬引用。若 UI 需要显示物品图标或角色名，应公开稳定 ID、文本、图片 URI 等表现数据，而非对象本身。

### 更新和脏标记

RmlUi 原生模型只支持顶层字段的脏标记。修改 `items[3].count` 后，受管数组或对象属性会自动通知 `items`，而不是要求业务代码手工通知：

```cpp
InventoryState->EditItems().Replace(ItemId, UpdatedItem);
```

`FMarkupObservableObject` 提供批量更新。事务结束前，同一顶层字段无论改变多少次都只刷新一次：

```cpp
InventoryState->BeginUpdate();
InventoryState->EditItems().Add(NewItem);
InventoryState->SetGold(NewGold);
InventoryState->EndUpdate();
```

视图会忽略没有实际变化的子值，但插件不承诺单个数组元素的独立刷新。背包、任务列表这类数据建议以顶层数组为刷新边界。

### C++ 原生命令

命令通过构造函数中的 `MARKUP_COMMAND_ACCESS` 登记在 `FMarkupObservableObject` 派生类中。它适合将 UI 意图交给控制器、子系统或游戏功能模块，而不要求它们是 `UObject`。命令参数由成员函数签名确定；对象挂载到模型时会自动出现在命令字典中。回调捕获应使用弱引用，Widget、模型或目标对象关闭后，命令必须安全地不执行。

## 双向绑定的完整规则

### 什么是双向绑定

`data-value` 和 `data-checked` 不仅是“控件显示字段值”。它们有两个方向：

```text
UE 改变 MusicVolume
    → 通知 settings 顶层字段
    → 滑条移到新位置

玩家拖动滑条
    → 把控件值转换为浮点数
    → 校验、调用字段 Write
    → 刷新 settings 顶层字段
    → 滑条显示最终有效值
```

`data-value` 只绑定单个可写标量路径；不要在这里写表达式，例如 `data-value="volume * 100"` 是不允许的。若 UI 显示的是百分比但存储的是 0 至 1，可增加只供展示的字段，或在字段 Write 中处理，而不是让 RML 推断反向公式。

`data-checked` 的布尔复选框写回 `bool`；单选组写回字符串或枚举的明确值。对枚举，必须在字段定义中给出允许项，不能接受任意字符串。

### 字段方向和提交时机

WPF 的“绑定模式”和“写回时机”是很有价值的产品概念，但 MarkupUI 不应为了模仿 XAML 而给 RML 另造一套绑定属性。公开 C++ 字段定义应直接表达能力和策略：

| 字段策略 | RML 可见行为 | 适用场景 |
| --- | --- | --- |
| 只读 | UE 变化后更新控件；RML 写入被拒绝。 | 金币、生命值、物品列表。 |
| 可写，普通提交 | `data-value` / `data-checked` 按控件的标准 `change` 语义写回。 | 设置滑条、复选框、普通表单。 |
| 可写，显式提交 | RML 只能展示草稿；点击已注册的“提交”命令后才写业务字段。 | 改名、购买数量、敏感设置。 |
| 仅接收输入 | RML 可以产生输入，但 UE 后续变化不主动覆盖控件。 | 搜索词、临时筛选条件。 |
| 一次性快照 | 仅在文档加载时读取一次。 | 文档标题、不可变说明。 |

“显式提交”和“仅接收输入”是原生字段策略，不是对 `data-value` 表达式规则的放宽。它们通过单独的草稿字段和命令组织，例如 `draft.player_name` 配合 `commit_player_name()`；不能要求 RML 从 `name | to_upper` 反推出应该写回何处。

### 输入转换和校验顺序

1. 读取控件提出的字符串、数字或布尔值。
2. 根据字段已声明类型转换。空字符串、非数字、溢出和非法枚举均为转换失败。
3. 执行声明式约束，例如最小值、最大值、长度、正则表达式、枚举集合。
4. 调用字段 Write；它执行最终业务校验，并返回接受、拒绝或修正。
5. 仅在接受或修正后标记所属顶层字段；拒绝时不改变 UE 数据。

`FMarkupWriteResult` 是拟议公开结果：接受、拒绝、接受但已修正。拒绝时控件恢复为上一次有效值；修正时，例如输入 1.5 被限制为 1.0，控件立刻显示 1.0。

```html
<input type="number" min="0" max="99" data-value="purchase.quantity"/>
<p data-if="purchase.error_message != ''">{{ purchase.error_message }}</p>
```

不要把错误提示依赖为插件自动生成的英文文本。业务 Write 可同时更新 `purchase.error_message`，从而实现本地化和产品一致的提示。

### 文本、输入法与外部更新冲突

文本输入在编辑状态时，输入法合成文本、选择范围和光标位置都必须优先保留。默认策略如下：

- 玩家尚未提交编辑时，外部更新同一文本字段不覆盖正在输入的内容。
- 控件触发 `change` 后提交；提交成功时显示已接受或修正后的值。
- 提交失败时恢复最后一次有效值，并保留由业务模型提供的错误状态。
- 焦点离开、文档重载或 Widget 销毁时，尚未提交的文本按 RmlUi 控件的正常 `change` 语义处理；游戏不应假设每个按键都会写回。

若产品确实需要每个按键同步，RML 可使用 `data-event-input` 调用专用命令。该命令应自行节流，不能把昂贵查询、网络请求或存档写入绑定在每个字符上。

### 赋值与命令的顺序

RML 可先对可写字段赋值，再触发命令：

```html
<button data-event-click="settings.show_minimap = !settings.show_minimap; save_settings()">
  保存设置
</button>
```

赋值只会作用于可写字段；`save_settings` 必须预先注册。UI 触发的写入完成后，命令才执行，因此命令能读取已接受的最终状态。未知命令、错误数量的参数或不匹配的参数类型都不会执行任何游戏逻辑。

### 批量更新与循环预防

从 RML 写回字段后，模型会把该字段标记为已处理。游戏状态随后发出的通知仅用于把最终值同步回控件，不应再次触发同一命令或产生无限循环。业务代码应只修改状态，不要在字段通知中模拟一次新的按钮点击。

## 反射绑定：把 `UObject` 适配为原生模型

反射绑定不是第二套绑定系统。任意 `UObject` 可直接作为绑定目标；每次绑定会生成一个 `FMarkupBlueprintObject`，以弱引用持有该对象，并将已授权的 UE 属性、Setter 和函数映射为访问器。其访问器字典、写入结果和命令字典与 C++ 宏模型完全相同。

### 推荐的视图模型写法

推荐创建专供界面使用的 `UObject` 或 `USTRUCT` 视图模型。它可以从游戏对象收集数据，但不直接把角色、背包组件等大型业务对象整体暴露给 RML。

```cpp
UCLASS(BlueprintType)
class UInventoryViewModel : public UObject
{
    GENERATED_BODY()

public:
    // RML：{{ player.display_name }}
    UPROPERTY(BlueprintReadOnly, FieldNotify,
        meta=(MarkupBind, MarkupPath="PlayerDisplayName"))
    FText PlayerDisplayName;

    // RML：{{ gold }}。未标记 MarkupWritable，因此只能展示。
    UPROPERTY(BlueprintReadOnly, FieldNotify, meta=(MarkupBind))
    int32 Gold = 0;

    // RML：data-for="item : items"。
    UPROPERTY(BlueprintReadOnly, FieldNotify, meta=(MarkupBind))
    TArray<FInventoryItemView> Items;

    // RML：data-value="settings.music_volume"。
    UPROPERTY(BlueprintReadWrite, FieldNotify,
        meta=(MarkupBind, MarkupPath="MusicVolume", MarkupWritable,
              ClampMin="0.0", ClampMax="1.0"))
    float MusicVolume = 0.8f;
};

USTRUCT(BlueprintType)
struct FInventoryItemView
{
    GENERATED_BODY()

    UPROPERTY(meta=(MarkupBind))
    FString Id;

    UPROPERTY(meta=(MarkupBind))
    FText Name;

    UPROPERTY(meta=(MarkupBind))
    int32 Count = 0;
};
```

这里的 `MarkupBind` 是公开白名单标记，`MarkupPath` 用来给字段指定 RML 路径；没有 `MarkupPath` 时使用属性名。`MarkupWritable` 是额外授权，绝不能因为属性有 `BlueprintReadWrite` 就自动允许 RML 写入。

`FieldNotify` 用于让 UE 告诉映射对象“这个字段变了”。若项目不采用该机制，仍可在修改字段后调用视图模型的 `NotifyMarkupChanged("字段")`。两种方式最终都通知同一个 `FMarkupObservableObject`。

### 反射公开的递归规则

- 顶层 `UObject` 只读取带 `MarkupBind` 的属性。
- 已公开的 `USTRUCT` 或数组元素，也只读取其带 `MarkupBind` 的成员。
- 任何一级遇到未标记成员、对象指针、循环引用或不支持的类型，均在开发环境报告清晰错误，并将该字段视为不可用；不能悄悄穿透到更多属性。
- `TMap` 的键必须可安全转换为字符串且没有歧义。面向 UI 的排序、筛选、分页通常应在视图模型中先转成数组。
- `Transient`、`Deprecated`、编辑器专用属性默认拒绝，除非另有明确的运行时公开标记。

### 反射可写字段与 Setter

写回必须同时满足：属性有 `MarkupBind`、有 `MarkupWritable`、类型可逆转换、当前对象有效，且通过全部校验。写回不会调用任意属性的隐式副作用逻辑。

对于有业务规则的字段，推荐声明专门的 Setter，而不是允许控件直接改属性：

```cpp
UFUNCTION(meta=(RmlSetter="settings.music_volume"))
FMarkupWriteResult SetMusicVolumeFromUi(float RequestedValue)
{
    const float AcceptedValue = FMath::Clamp(RequestedValue, 0.0f, 1.0f);
    MusicVolume = AcceptedValue;
    ApplyAudioVolume(AcceptedValue);
    NotifyFieldValueChanged(GET_MEMBER_NAME_CHECKED(UInventoryViewModel, MusicVolume));
    return FMarkupWriteResult::Corrected(AcceptedValue);
}
```

Setter 不可使用输出参数、引用参数、潜伏执行或依赖编辑器世界。它返回的 `FMarkupWriteResult` 直接成为原生字段写入结果；因此 C++、反射和蓝图对拒绝与修正有完全一致的表现。

## `UFUNCTION` 反射命令

除了字段，反射层还可将视图模型上明确标记的 `UFUNCTION` 转换为原生命令：

```cpp
UCLASS(BlueprintType)
class UInventoryViewModel : public UObject
{
    GENERATED_BODY()

public:
UFUNCTION(BlueprintCallable, meta=(MarkupCommand, MarkupName="select_item"))
    void SelectItem(const FString& ItemId)
    {
        OnItemSelected.Broadcast(ItemId);
    }

UFUNCTION(BlueprintCallable, meta=(MarkupCommand, MarkupName="request_close"))
    void RequestClose()
    {
        OnCloseRequested.Broadcast();
    }
};
```

函数参数必须是可从 RML 安全转换的值类型，例如布尔值、数字、字符串、`FName`、已声明的枚举、颜色、二维向量或受限的值对象。函数必须满足：

- 不含返回引用、输出参数或非常量引用参数；
- 不是潜伏函数、网络远程调用、控制台命令或编辑器专用函数；
- 不接受 `UObject`、`UClass`、Actor、组件、委托或任意结构化文本；
- 参数数量、顺序和类型与 RML 调用严格匹配；
- 执行目标是当前模型绑定的对象，不能从 RML 指定另一个对象或函数名。

默认不支持命令返回值。需要显示结果时，命令修改视图模型字段并通知刷新，例如设置 `purchase.error_message` 或更新 `items`。这让蓝图和 C++ 都遵循同一种、易追踪的状态变化方式。

## Slate（C++）使用草案

Slate 使用者创建 `FMarkupObservableObject` 派生模型，并直接挂载到 Widget。

```cpp
TSharedRef<SRmlUiWidget> Ui = SNew(SRmlUiWidget)
    .DocumentContent(InventoryRml)
    .DocumentSourceUri(TEXT("/Game/UI/Inventory.rml"));

TSharedRef<FInventoryCollectionUiState> InventoryState =
    MakeShared<FInventoryCollectionUiState>();
Ui->SetDataModel(TEXT("inventory"), InventoryState);
```

每个 `SRmlUiWidget` 必须拥有单独的模型实例。多个玩家同时打开同一份背包界面时，不能共用同一模型，除非该模型明确只读且不会保存任何界面局部状态。

## UMG（蓝图）使用草案

计划新增 `URmlUiWidget`，作为可放入 Widget Blueprint 的控件。它选择 RML Document、模型名和视图模型类别，并在运行时为每个控件创建自己的模型对象。

推荐工作流：

1. 创建 `BP_InventoryViewModel`，它可以是任意 `UObject` 蓝图；使用 `MarkupBind`/`MarkupCommand` 标记需要公开的成员。
2. 在其中定义需要展示的字段；只把允许控件更改的字段标为 `MarkupWritable`，为业务写入提供 Setter。
3. 在 Widget Blueprint 中加入“RML 界面”控件，指定 RML Document、模型名 `inventory` 和该视图模型类别。
4. 在“构造”或拥有者初始化时，调用视图模型的“设置字段”或“从背包刷新”函数；可使用“开始模型更新/结束模型更新”合并多项变化。
5. 在 `select_item`、`request_close` 等命令的蓝图事件中调用游戏逻辑。
6. 在“销毁”时解除与库存、设置、任务系统等外部对象的委托绑定。

Designer 预览只能使用视图模型的设计时默认值或专用预览数据。它不得保存设置、触发命令或修改游戏世界。

蓝图节点只是原生 API 的包装：

| 节点 | 对应原生能力 | 作用 |
| --- | --- | --- |
| 创建 RML 模型 | 创建模型 | 为当前 RML 控件创建具名模型。 |
| 设置 RML 值 | `SetValue` | 写入标量、对象或数组，并标记顶层字段。 |
| 通知 RML 字段变化 | `NotifyChanged` | 适用于外部已直接修改视图模型属性的情况。 |
| 开始/结束模型更新 | `BeginUpdate` / `EndUpdate` | 合并多次状态变化。 |
| 绑定 RML 命令 | `DefineCommand` | 按命令名接收已校验参数。 |
| 接受/拒绝/修正输入 | `FMarkupWriteResult` | 用于蓝图 Setter 的结果。 |

蓝图不应靠字符串形式的 JSON 填充数据。应提供“构造 RML 对象”“构造 RML 数组”“设置字符串/数字/布尔/颜色”节点，使类型错误可在编辑期或运行期明确报告。

## 生命周期、重载与线程

对使用者可见的生命周期是：

```text
创建控件
  → 创建/配置模型和命令
  → 加载 RML 文档
  → 游戏状态与用户输入双向同步
  → 文档重载或控件销毁时，旧模型和命令失效
```

模型必须在文档首次使用其 `data-model` 前准备完成。文档重载会创建新的绑定上下文；旧模型句柄、旧命令回调和旧控件输入状态均失效。若需要保留搜索词、滚动位置或草稿，应由游戏/视图模型保存这些值，再在新模型建立后提交。

所有公开的模型写入、反射读取、输入处理和命令回调都要求在游戏线程调用。异步加载、网络回调或后台任务应切回游戏线程后再更新视图模型。调用已销毁模型应返回失败结果而非崩溃。

## 错误、调试与安全

开发环境应提供绑定检查器，按界面实例显示：模型名、公开字段、读写权限、字段类型、最近刷新字段、最近的输入拒绝原因和最近命令。Shipping 不显示实际敏感数据。

日志或检查器中的错误必须包含文档标识、模型名和字段/命令名。至少覆盖：

- RML 使用了未声明的模型、字段或命令；
- RML 尝试写只读字段；
- 值转换、范围校验、枚举校验或 Setter 拒绝；
- `UFUNCTION` 参数不符合命令契约；
- 不支持的反射类型、循环引用或未标记成员；
- 模型已失效、目标对象已销毁、或从非游戏线程调用；
- 同一控件中重复模型名。

开发环境推荐把“RML 引用了不存在字段”作为明显警告，并以安全默认值显示；Shipping 保持安全默认值且不暴露对象信息。是否让该问题在严格模式下直接阻止文档加载，应作为项目设置提供。

## 自动化验收清单

- 同一份 RML 在 Slate 与 UMG 中，对同一模型数据得到一致的文字、属性、可见性和列表内容。
- 模型宏生成的 Getter、Setter、只读权限、浮点比较、嵌套对象通知、集合通知和命令签名，与手写原生模型具有相同结果。
- 标量、对象、嵌套结构、数组、枚举、`FText` 与受限映射的转换正确；不支持的值明确失败。
- `FieldNotify` 和手动 `NotifyRmlChanged` 都会刷新正确的顶层字段；数组成员变化要求刷新数组字段。
- `data-value`、`data-checked` 覆盖正常写入、类型错误、范围修正、业务拒绝和外部更新冲突。
- 输入法合成、文本选择与焦点期间不被外部刷新错误打断。
- 局部 DOM 文本、属性、样式、类和自定义伪类只更新声明的目标，不覆盖原生数据视图管理的元素。
- 局部列表覆盖稳定键的单条目更新、插入、删除、移动、文档重载和失效元素观察句柄；不得重建无关条目或丢失容器滚动状态。
- 严格模式拒绝重复目标 `id`、`data-for` 与局部列表共管、以及两个写入者管理同一元素表现属性。
- C++ 回调与 `UFUNCTION` 命令均覆盖参数成功、参数不匹配、目标销毁和重复点击。
- 文档重载、控件销毁、多个 Widget、多本地玩家不会串用模型、输入状态或命令。
- UMG Designer 不触发命令或写回真实游戏对象。

## 分阶段实施建议

1. **模型基础与宏**：统一值类型、属性访问器、顶层脏标记、批量更新、可观察对象/集合宏和 Slate 显式模型；先验证展示与原生 `data-for` 列表。
2. **局部 DOM 刷新**：实现作者命名目标、文本/属性/样式/类绑定、稳定键列表、冲突检查和局部更新回归；它是实现细粒度刷新的关键阶段。
3. **反射和双向写入**：实现 `MarkupBind`、`MarkupWritable`、Setter、`FieldNotify`、值校验和文本输入冲突规则。
4. **命令系统**：实现 C++ 命令回调和受限 `UFUNCTION` 命令，并完成安全审计与错误呈现。
5. **UMG 支持**：提供 `UMarkupUiWidget`、任意 `UObject` 的反射映射、蓝图节点、事件委托和 Designer 预览；与 Slate 共用回归用例。
6. **可选互操作**：评估与 UE MVVM 字段通知的更深整合、可复用数据源、多模型协调和 `data-rml`。这些均不能破坏当前白名单和双向写入契约。

## 尚待产品确认的选择

- `FMarkupBlueprintObject` 对 `UObject` 的反射映射是否只接受显式 `MarkupBind`/`MarkupCommand` 标记？建议始终保持显式白名单，不提供自动暴露全部属性的选项。
- 数值超出范围时，默认修正还是默认拒绝？建议由字段声明决定：设置类字段通常修正，交易、权限和数量类字段通常由 Setter 决定。
- 严格模式下，未声明字段是否阻止文档加载？建议开发/测试环境可配置为阻止，Shipping 使用安全默认值并记录汇总诊断。
- 是否提供每键同步的蓝图便捷节点？建议只提供受节流的专用命令，不把它作为普通 `data-value` 的默认行为。

## 参考资料

- [RmlUi：数据绑定](https://mikke89.github.io/RmlUiDoc/pages/data_bindings.html)
- [RmlUi：元素与 DOM 接口](https://mikke89.github.io/RmlUiDoc/pages/cpp_manual/elements.html)
- [Noesis UI：数据绑定](https://www.noesisengine.com/docs/Gui.Core.DataBindingTutorial.html)
