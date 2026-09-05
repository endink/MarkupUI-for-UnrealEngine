# UE 数据绑定设计草案

> 状态：讨论草案。本文描述计划提供的公开能力、用法与验收标准；名称均为拟议名称，可在实现前调整。本文不假定读者了解 RML 或 RmlUi 的源码。

## 一句话目标

让 RML 界面从明确登记的可绑定对象中读取数据；让输入框、复选框等控件把已验证的结果安全写回；让按钮等交互以命令通知游戏逻辑。无论界面放在 Slate（C++）还是 UMG（蓝图）中，RML 写法和结果都一致。

例如，角色金币变化后，`{{ gold }}` 自动刷新；玩家拖动音量滑条后，游戏设置中的音量被限制在合法范围并写回；玩家点击物品后，游戏收到“选择物品”的命令及物品 ID，而不是让 RML 任意调用游戏对象。

## 设计起点：C++ 原生优先，蓝图只是包装

数据绑定的第一步必须是可独立使用的 C++ 原生契约：它不要求 `UObject`、反射、UMG 或蓝图。Slate、游戏模块和任何普通 C++ 数据结构都能通过它创建模型、提供值、接受写入并注册命令。

第二步是 UE 反射数据上下文：`SetDataContext(UObject*)` 直接把业务对象挂到文档或元素。第三步才是 UMG 控件：它调用相同的数据上下文、字段通知和集合操作节点，不拥有另一套 RML 语法、刷新时序或验证规则。

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
| 属性 Getter/Setter + 属性变化通知 | `FMarkupObservableObject` 与 `MARKUP_*` 属性宏。 |
| 可观察集合 | `FMarkupObservableArray`；任何增删改排都会通知字段路径，作为元素数据上下文时由该子树内的 `data-for` 更新。 |
| 数据上下文 | 文档或元素持有的可观察对象/数组。文档上下文是默认值，元素上下文可覆盖其子树。 |
| 单向、双向等绑定模式 | 由元素上的绑定属性选择 FromSource、FromUI 或双向；字段权限只决定对应读取或写入请求是否被接受，不引入与 RML 原生语法冲突的另一套标记。 |
| 每次变化、失焦、显式提交 | 保持 RmlUi 控件原有 `change` 语义为默认；高频或延迟提交用明确命令实现，不隐式改变控件行为。 |

最重要的差异是刷新粒度。RmlUi 原生数据模型只接受顶层变量的脏标记；因此只使用原生 `data-model` 时，`Player.Name.First` 变化必须通知最外层字段 `player`。MarkupUI 通过文档和元素的数据上下文把可绑定对象或可绑定数组挂到对应 DOM 子树；每个子树只订阅自己的上下文，从而缩小受影响的绑定范围，不需要字段到文本、属性或样式的 Lambda 绑定 API。

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

RML 不自动读取对象成员或执行任意函数。每一项可见属性和命令都必须显式登记。

## 术语

| 术语 | 含义 |
| --- | --- |
| 模型 | RML 用来读写的一组命名数据，例如 `inventory`。一份正在显示的文档拥有自己的模型实例。 |
| 字段 | 模型中的顶层数据，例如 `gold`、`items`、`settings`。RML 可读取 `items[0].name`；原生数据模型以顶层字段刷新，元素数据上下文可将不同子树拆分为独立更新范围。 |
| 路径 | 在 RML 中访问数据的写法，例如 `settings.music_volume`。 |
| 只读字段 | RML 可以显示但不能通过输入控件写回的字段。 |
| 可写字段 | RML 可以通过指定控件写回的字段；每次写入均须类型转换和校验。 |
| 命令 | RML 发出的具名交互意图，例如“选择物品”“开始游戏”。它有固定的参数契约，由已登记的 C++ 成员函数接收。 |
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
| `data-event-*` | 支持 | 发送受控的命令，可绑定已登记的 C++ 成员函数。 |
| 自定义转换函数 | 支持 | 格式化展示值；必须是无副作用的纯计算。 |
| `data-rml` | 延后 | 它可改变文档内容，需单独确定安全和重载规则。 |

`data-*` 和 `{{ ... }}` 是 RML 的保留语法，必须在文档加载前写好。运行中给现有元素动态添加这些属性，不应期待它们变成绑定。

## C++ 原生方案

### 统一的运行时绑定对象：`FMarkupObservableObject`

原生 C++ 数据模型优先继承 `FMarkupObservableObject`，并通过 `MARKUP_*` 宏声明属性、嵌套对象、数组、只读字段、业务 Setter 和命令。

`IMarkupObservableObject` 是可观察对象接口；`FMarkupObservableObject` 是唯一推荐的基础实现。它维护属性/命令字典和属性变化事件，根对象通过 `SetDataContext` 设置到文档或元素后，RML 自动从该字典读取、写入并订阅变化。

UE 反射路径直接使用 `UObject`。`SetDataContext(UObject*)` 以 UE 反射读取已公开的属性和函数；蓝图的 Set、数组和 Map 通知节点只作用于这个 UObject 上真实存在的成员，不创建第二个代理 Context 或镜像属性。

### 可绑定对象

为了让 C++ 数据模型能够用简洁、声明式的方式定义属性，同时保留 UE 所需的权限、元素数据上下文和命令能力，应提供一组只依赖 C++ 的模型宏。它们不要求 `UObject`，也不依赖 `UPROPERTY`；因此可用于 Slate、子系统、普通游戏状态和自动化测试。

属性声明和构造函数登记处于两个不同的 C++ 作用域：类定义宏生成受保护的 `FMarkupProperty` 字段，以及按权限裁剪的公开 Getter/Setter；构造函数调用 `RegisterProperty` 登记字段，并可选择登记该对象的 Getter、Setter 成员函数。没有隐藏登记成员、没有额外属性清单，也不需要 Begin/End 类宏。

属性名在类定义宏与 `RegisterProperty` 中各出现一次。这是刻意保留的、可审查的显式关联：开发者可以一眼看出哪个字段被公开给 UI，以及页面读写会经过哪些成员函数；构造函数也与完全手写代码一一对应。

`FMarkupObservableObject` 维护“名称 → 属性”字典。读写权限由属性字段本身决定：无后缀属性宏表示读写，`_READONLY` 表示只读，`_WRITEONLY` 表示只写；对象和数组遵循同一规则。读写权限是 `FMarkupProperty` 的模型标识，与页面刷新方向无关。

基类对模型和高级 C++ 使用者公开统一查询接口：

```cpp
class FMarkupCommand;

class FMarkupObservableObject : public IMarkupObservableObject
{
public:
    const FMarkupProperty* FindProperty(FName Name) const;
    const TMap<FName, FMarkupProperty>& GetProperties() const;
    const FMarkupCommand* FindCommand(FName Name) const;
    const TMap<FName, FMarkupCommand>& GetCommands() const;
    FMarkupPropertyChangedEvent& OnPropertyChanged();

protected:
    // 以下四种形式均可用；GetterMemberFunction、SetterMemberFunction 为伪代码占位。
    // 最终以模板重载实现，且只接受当前对象的非静态成员函数指针。
    void RegisterProperty(FName Name, FMarkupProperty& Property);
    void RegisterProperty(FName Name, FMarkupProperty& Property, GetterMemberFunction Getter);
    void RegisterProperty(FName Name, FMarkupProperty& Property, nullptr_t, SetterMemberFunction Setter);
    void RegisterProperty(FName Name, FMarkupProperty& Property, GetterMemberFunction Getter, SetterMemberFunction Setter);
    void RegisterCommand(FName Name, FMarkupCommand Command);
    void NotifyPropertyChanged(FStringView Path);
};
```

`FMarkupProperty` 统一保存类型化存储、值类型与访问权限；`FMarkupCommand` 统一保存参数契约和成员函数调用描述；`FMarkupPropertyChangedEvent` 发送变化路径。`RegisterProperty` 可将页面读写接到已登记的 Getter、Setter 成员函数：未登记的一侧直接使用属性字段，登记的一侧必须经过该成员函数。Lambda、自由函数、静态成员函数及裸 `this` 回调都不是属性访问器的登记形式。模型绑定、元素数据上下文、反射适配和蓝图包装都只通过这些接口工作，不需要了解派生类的成员布局。

#### 可绑定属性

`FMarkupProperty` 是非模板的统一字段类型。属性宏生成它的受保护字段；反射与蓝图适配也创建同一类型的字段。字段以 `DataType` 固定其值类型，以 `Access` 固定可读写能力，而不是由页面或某个控件决定。

以下是拟议的伪代码定义。它只描述公开形状，不代表最终头文件名、include、模块导出规则或 ABI。所有类型化 Get/Set 都要求与 `DataType` 匹配；不匹配时在开发环境报告明确错误，写入返回失败，不进行隐式转换。

```cpp
// 伪代码：FMarkupValue、FMarkupWriteResult 与 UE 基础类型在此处均已可用。

class IMarkupObservableArray;
class IMarkupObservableObject;
class IMarkupPropertyStore;

enum class EMarkupDataType : uint8
{
    None,
    Bool,
    Int32,
    Int64,
    Float,
    Double,
    String,
    Text,
    Name,
    Enum,
    Color,
    Vector2D,
    Object,
    Array,
};

enum class EMarkupPropertyAccess : uint8
{
    ReadOnly,
    WriteOnly,
    ReadWrite,
};

class FMarkupProperty
{
public:
    FMarkupProperty() = default;

    EMarkupDataType GetDataType() const;
    EMarkupPropertyAccess GetAccess() const;
    bool CanRead() const;
    bool CanWrite() const;

    // 仅供宏生成代码和内部存储层使用；手写模型使用下方的类型化 Get/Set。
    const void* GetValuePtr() const;
    FMarkupWriteResult SetValuePtr(const void* Value);

    FMarkupValue GetValue() const;
    FMarkupWriteResult SetValue(const FMarkupValue& Value);

    bool GetBool() const;
    int32 GetInt32() const;
    int64 GetInt64() const;
    float GetFloat() const;
    double GetDouble() const;
    FString GetString() const;
    FText GetText() const;
    FName GetName() const;
    int64 GetEnum() const;
    FLinearColor GetColor() const;
    FVector2D GetVector2D() const;
    IMarkupObservableObject* GetObject() const;

    template<typename TObject>
    TSharedPtr<TObject> GetObject() const;
    IMarkupObservableArray* GetArray() const;

    template<typename TItem>
    TSharedPtr<FMarkupObservableArray<TItem>> GetArray() const;

    FMarkupWriteResult SetBool(bool Value);
    FMarkupWriteResult SetInt32(int32 Value);
    FMarkupWriteResult SetInt64(int64 Value);
    FMarkupWriteResult SetFloat(float Value);
    FMarkupWriteResult SetDouble(double Value);
    FMarkupWriteResult SetString(FStringView Value);
    FMarkupWriteResult SetText(const FText& Value);
    FMarkupWriteResult SetName(FName Value);
    FMarkupWriteResult SetEnum(int64 Value);
    FMarkupWriteResult SetColor(FLinearColor Value);
    FMarkupWriteResult SetVector2D(FVector2D Value);
    FMarkupWriteResult SetObject(IMarkupObservableObject* Value);

    template<typename TObject>
    FMarkupWriteResult SetObject(TSharedPtr<TObject> Value);
    FMarkupWriteResult SetArray(IMarkupObservableArray* Value);

    template<typename TItem>
    FMarkupWriteResult SetArray(TSharedPtr<FMarkupObservableArray<TItem>> Value);

private:
    // 属性字典中的副本与模型字段共享同一个存储。
    TSharedPtr<IMarkupPropertyStore> Store;
};
```

`FMarkupProperty` 自身不带模板参数。`GetObject<T>` / `SetObject<T>` 与 `GetArray<T>` / `SetArray<T>` 只是 UE C++ 侧的类型化便捷成员，用于免除手写模型的对象、数组指针转换；它们不改变字段类型，也不跨越 C API。为同时保留 C++ 的类型安全与 C API 的稳定边界，具体值存储放在内部的类型擦除接口之后：

```cpp
class IMarkupPropertyStore
{
public:
    virtual ~IMarkupPropertyStore() = default;

    virtual EMarkupDataType GetDataType() const = 0;
    virtual EMarkupPropertyAccess GetAccess() const = 0;
    virtual bool CanRead() const = 0;
    virtual bool CanWrite() const = 0;

    virtual const void* GetValuePtr() const = 0;
    virtual FMarkupWriteResult SetValuePtr(const void* Value) = 0;
    virtual FMarkupValue ToMarkupValue() const = 0;
    virtual FMarkupWriteResult FromMarkupValue(
        const FMarkupValue& Value) = 0;
};

template<typename T>
class TMarkupPropertyStore final : public IMarkupPropertyStore
{
public:
    explicit TMarkupPropertyStore(
        T InDefaultValue,
        EMarkupPropertyAccess InAccess);

    virtual EMarkupDataType GetDataType() const override;
    virtual EMarkupPropertyAccess GetAccess() const override;
    virtual bool CanRead() const override;
    virtual bool CanWrite() const override;

    virtual const void* GetValuePtr() const override;
    virtual FMarkupWriteResult SetValuePtr(const void* Value) override;
    virtual FMarkupValue ToMarkupValue() const override;
    virtual FMarkupWriteResult FromMarkupValue(
        const FMarkupValue& Value) override;

private:
    T Value;
    EMarkupPropertyAccess Access;
};

template<typename T>
FMarkupProperty MakeMarkupProperty(
    T InDefaultValue,
    EMarkupPropertyAccess InAccess);

```

属性宏根据其 `Type` 参数在内部创建对应的 `TMarkupPropertyStore<Type>`，但生成给模型类的字段始终是非模板的 `FMarkupProperty`。例如 `MARKUP_PROPERTY(FString, First, TEXT(""))` 生成 `FMarkupProperty FirstProperty`，其内部存储为 `TMarkupPropertyStore<FString>`。属性字典保存 `FMarkupProperty` 的登记项时，共享同一份内部存储；登记项还可保存指向对象成员 Getter、Setter 的调用描述。构造函数中的 `RegisterProperty` 只登记已有字段及其可选访问成员，不重复类型。

模板实例绝不穿过 C API 或动态库边界。对外读写统一经过 `FMarkupProperty`、`FMarkupValue` 和带类型标签的 C 值；`TMarkupPropertyStore` 只存在于编译它的 C++ 模块内。

字段使用内部类型化存储；`GetXxx` / `SetXxx` 供模型类自身使用，登记后的 `GetValue` / `SetValue` 是来自 RML、反射或蓝图的受控入口。若 `RegisterProperty` 登记了 Getter，读取会调用 Getter；若登记了 Setter，写入会调用 Setter。未登记的一侧才直接读写属性字段。属性宏负责生成字段、配置 `DataType` / `Access`，并决定类对外公开哪一侧接口。每个类定义宏的最后一个参数均为可选的 `DefaultValue`；省略时按该类型的默认构造值初始化。开发者不应手写 `XxxProperty` 字段：

| 类定义宏 | 宏生成的受保护字段 | 宏生成的公开 C++ 接口 |
| --- | --- | --- |
| `MARKUP_PROPERTY(float, MusicVolume[, DefaultValue])` | `FMarkupProperty MusicVolumeProperty` | `GetMusicVolume()`、`SetMusicVolume(float)` |
| `MARKUP_PROPERTY_READONLY(FString, PlayerId[, DefaultValue])` | `FMarkupProperty PlayerIdProperty` | `GetPlayerId()` |
| `MARKUP_PROPERTY_WRITEONLY(FString, SearchQuery[, DefaultValue])` | `FMarkupProperty SearchQueryProperty` | `SetSearchQuery(FString)` |

类自身及其派生类可通过受保护字段调用与 `DataType` 对应的 Get/Set；对外暴露的 Getter、Setter 则严格由宏声明决定。`MARKUP_PROPERTY_READONLY` 不自动生成 `SetXxx` 成员函数，`MARKUP_PROPERTY_WRITEONLY` 不自动生成 `GetXxx` 成员函数；开发者仍可自行声明和实现 `SetXxx` 或 `GetXxx` 作为额外包装。`ReadOnly`、`WriteOnly` 与 `ReadWrite` 是模型字段能力，不描述页面是否刷新。

根对象设置为数据上下文后，插件从字典递归取得属性：`Name` 是根对象中的对象属性，`First` 是其子对象中的属性。因此页面可使用 `{{ Name.First }}`、`{{ Weight }}` 和 `{{ PlayerId }}`。若内容规范要求小写名称，使用额外的命名变体宏即可；默认名称始终与 C++ 属性名一致。

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        RegisterProperty(TEXT("First"), FirstProperty, &FNameUiState::GetFirst, &FNameUiState::SetFirst);
        RegisterProperty(TEXT("Last"), LastProperty, &FNameUiState::GetLast, &FNameUiState::SetLast);
    }

    MARKUP_PROPERTY(FString, First, TEXT(""))
    MARKUP_PROPERTY(FString, Last, TEXT(""))
};

class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        RegisterProperty(TEXT("Name"), NameProperty, &FPlayerUiState::GetName, &FPlayerUiState::SetName);
        RegisterProperty(TEXT("Weight"), WeightProperty, &FPlayerUiState::GetWeight, &FPlayerUiState::SetWeight);
        RegisterProperty(TEXT("PlayerId"), PlayerIdProperty, &FPlayerUiState::GetPlayerId);
    }

    MARKUP_OBJECT_PROPERTY(TSharedPtr<FNameUiState>, Name, nullptr)
    MARKUP_PROPERTY(float, Weight, 0.0f)
    MARKUP_PROPERTY_READONLY(FString, PlayerId, TEXT(""))
};

TSharedRef<FPlayerUiState> Player = MakeShared<FPlayerUiState>();
Ui->GetDocument()->SetDataContext(Player);
```

##### 宏写法与不用宏的对照

开发者在构造函数中显式登记属性，再在类中声明属性：

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        RegisterProperty(TEXT("First"), FirstProperty, &FNameUiState::GetFirst, &FNameUiState::SetFirst);
        RegisterProperty(TEXT("Last"), LastProperty, &FNameUiState::GetLast, &FNameUiState::SetLast);
    }

    MARKUP_PROPERTY(FString, First, TEXT(""))
    MARKUP_PROPERTY(FString, Last, TEXT(""))
};
```

以下是宏的概念展开。它展示宏生成的 `FMarkupProperty` 字段、构造函数登记和公开 Getter/Setter；它不是要求开发者手写的第二套数据字段：

```cpp
class FNameUiState final : public FMarkupObservableObject
{
public:
    FNameUiState()
    {
        RegisterProperty(TEXT("First"), FirstProperty, &FNameUiState::GetFirst, &FNameUiState::SetFirst);
        RegisterProperty(TEXT("Last"), LastProperty, &FNameUiState::GetLast, &FNameUiState::SetLast);
    }

    FString GetFirst() const
    {
        return FirstProperty.GetString();
    }

    void SetFirst(FString InFirst)
    {
        FirstProperty.SetString(InFirst);
    }

    FString GetLast() const
    {
        return LastProperty.GetString();
    }

    void SetLast(FString InLast)
    {
        LastProperty.SetString(InLast);
    }

protected:
    FMarkupProperty FirstProperty = MakeMarkupProperty<FString>(TEXT(""), EMarkupPropertyAccess::ReadWrite);
    FMarkupProperty LastProperty = MakeMarkupProperty<FString>(TEXT(""), EMarkupPropertyAccess::ReadWrite);
};
```

`RegisterProperty(TEXT("First"), FirstProperty, &FNameUiState::GetFirst, &FNameUiState::SetFirst)` 是读写属性的标准登记方式；它把名称、已定义的属性字段和对象成员访问函数登记到基类字典。基类在此处连接属性变化回调；`FMarkupProperty` 负责类型检查、固定的类型比较和写入。因此无论 C++ 调用还是页面写入，都会经过 `SetFirst`：第一次调用改变值并发出一次通知；第二次传入相同值时不发通知。手写 Setter 无需自行比较或调用 `NotifyPropertyChanged`。

若 Getter、Setter 均未登记，页面读取和写回都会跳过类中同名的 `GetFirst`、`SetFirst`，直接读写 `FirstProperty`：

```cpp
RegisterProperty(TEXT("First"), FirstProperty, nullptr, nullptr);
```

#### 嵌套对象与数组：宏写法与不用宏的对照

嵌套对象和数组也在构造函数中显式登记属性：

对象和数组与标量属性保持相同的声明选择：

```cpp
MARKUP_OBJECT_PROPERTY(Type, Name[, DefaultValue])
MARKUP_OBJECT_PROPERTY_READONLY(Type, Name[, DefaultValue])
MARKUP_OBJECT_PROPERTY_WRITEONLY(Type, Name[, DefaultValue])

MARKUP_ARRAY_PROPERTY(ElementType, Name[, DefaultValue])
MARKUP_ARRAY_PROPERTY_READONLY(ElementType, Name[, DefaultValue])
MARKUP_ARRAY_PROPERTY_WRITEONLY(ElementType, Name[, DefaultValue])
```

无后缀宏表示读写；`_READONLY` 与 `ReadOnly` 访问登记配对，只自动公开读取和页面读取；`_WRITEONLY` 与 `WriteOnly` 访问登记配对，只自动公开写入。`_READONLY` 不生成 `SetXxx`，`_WRITEONLY` 不生成 `GetXxx`；如业务代码仍需要另一侧的便利接口，可以自行包装。对象 Setter 的含义是替换整个对象引用；数组 Setter 的含义是替换整个受管数组。数组内部的增加、删除、移动和替换仍应通过受管集合接口或命令完成，不向页面暴露可任意修改的裸 `TArray`。纯 `WriteOnly` 对象或数组不能作为需要先读取当前值的 RML 绑定目标，例如对象路径读取或 `data-for`；它们应通过事件命令提交完整的新值。

```cpp
class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        RegisterProperty(TEXT("Name"), NameProperty, &FPlayerUiState::GetName, &FPlayerUiState::SetName);
        RegisterProperty(TEXT("Items"), ItemsProperty, &FPlayerUiState::GetItems, &FPlayerUiState::SetItems);
    }

    MARKUP_OBJECT_PROPERTY(TSharedPtr<FNameUiState>, Name, nullptr)
    MARKUP_ARRAY_PROPERTY(FItemView, Items)
};
```

不使用这些宏时，手写模式仍只声明 `FMarkupProperty` 字段并在构造函数登记它们：

```cpp
class FPlayerUiState final : public FMarkupObservableObject
{
public:
    FPlayerUiState()
    {
        RegisterProperty(TEXT("Name"), NameProperty, &FPlayerUiState::GetName, &FPlayerUiState::SetName);
        RegisterProperty(TEXT("Weight"), WeightProperty, &FPlayerUiState::GetWeight, &FPlayerUiState::SetWeight);
        RegisterProperty(TEXT("PlayerId"), PlayerIdProperty, &FPlayerUiState::GetPlayerId);
    }

    TSharedPtr<FNameUiState> GetName() const
    {
        return NameProperty.GetObject<FNameUiState>();
    }

    void SetName(TSharedPtr<FNameUiState> InName)
    {
        NameProperty.SetObject(MoveTemp(InName));
    }

    float GetWeight() const
    {
        return WeightProperty.GetFloat();
    }

    void SetWeight(float InWeight)
    {
        WeightProperty.SetFloat(FMath::Clamp(InWeight, 0.0f, 100.0f));
    }

    FString GetPlayerId() const
    {
        return PlayerIdProperty.GetString();
    }

protected:
    FMarkupProperty NameProperty = MakeMarkupProperty<TSharedPtr<FNameUiState>>(nullptr, EMarkupPropertyAccess::ReadWrite);
    FMarkupProperty WeightProperty = MakeMarkupProperty<float>(0.0f, EMarkupPropertyAccess::ReadWrite);
    FMarkupProperty PlayerIdProperty = MakeMarkupProperty<FString>(TEXT(""), EMarkupPropertyAccess::ReadOnly);
};
```

手写对象属性使用 `GetObject<T>` / `SetObject<T>`，数组属性使用 `GetArray<T>` / `SetArray<T>`，标量属性使用对应的 `GetXxx` / `SetXxx`。读写属性应将相应的成员 Getter、Setter 传给 `RegisterProperty`；页面读写与 C++ 调用因而共用同一条访问路径。`RegisterProperty` 负责把属性接入基类的变化转发；替换对象、子对象变化、以及数组的受管变更都不要求模型类手写订阅、解除订阅或 `NotifyPropertyChanged`。`PlayerIdProperty` 为只读属性，因此只登记 Getter，RML 写入会被拒绝。

以下代码展示**不使用** `MARKUP_ARRAY_PROPERTY` 的手写受管集合属性。它登记属性字段及可选访问成员；无论采用手写方式还是 `MARKUP_ARRAY_PROPERTY` 宏，都不暴露可随意改动的 `TArray`，只暴露受管集合接口：

```cpp
class FInventoryCollectionUiState final : public FMarkupObservableObject
{
public:
    FInventoryCollectionUiState()
    {
        RegisterProperty(TEXT("Items"), ItemsProperty, &FInventoryCollectionUiState::GetItems, &FInventoryCollectionUiState::SetItems);
    }

    TSharedPtr<FMarkupObservableArray<FItemView>> GetItems() const
    {
        return ItemsProperty.GetArray<FItemView>();
    }

    void SetItems(TSharedPtr<FMarkupObservableArray<FItemView>> InItems)
    {
        ItemsProperty.SetArray(MoveTemp(InItems));
    }

protected:
    FMarkupProperty ItemsProperty = MakeMarkupProperty<TSharedPtr<FMarkupObservableArray<FItemView>>>(MakeShared<FMarkupObservableArray<FItemView>>(), EMarkupPropertyAccess::ReadWrite);
};
```

`FMarkupObservableArray` 在 `Add`、`RemoveByKey`、`Move`、`Replace` 和 `Reset` 时发出自身的变化事件。将 `ItemsProperty` 登记给基类后，基类负责把这些变化转发为对象属性变化。通知最终聚合为顶层 `items`；当该数组被设为元素数据上下文时，由该元素子树内的 `data-for` 更新其列表。

#### 设置数据上下文

将根对象设为文档数据上下文后，`FMarkupObservableObject` 的访问器字典提供其属性、对象和数组：

```cpp
TSharedRef<FPlayerUiState> Player = MakeShared<FPlayerUiState>();
Ui->GetDocument()->SetDataContext(Player);

Player->GetName()->SetFirst(TEXT("Ada"));
Player->SetWeight(72.5f);
```

`MARKUP_OBJECT_PROPERTY` 要求对象本身也是可观察对象；构造函数中的 `RegisterProperty` 负责登记属性及其 Getter、Setter 成员函数。替换 `Name` 对象时，它通知 `Name`；子对象的 `First` 改变时，父对象转发为 `Name.First`。注册给 RmlUi 的内部数据模型时，最终仍聚合为数据上下文顶层字段的脏标记；设为元素数据上下文时，该元素子树可订阅 `Name.First`。

可绑定属性不提供自定义比较器。标量属性使用框架固定的同类型相等判断；对象属性与集合属性只比较共享引用身份。替换为同一引用时不通知；对象内部属性变化和集合内部受管变更分别由对象、集合自身的变化事件转发。

### 集合、对象和复杂写入的宏

直接暴露可写 `TArray` 会让调用者绕过通知；模型宏必须只提供受管集合接口：

```cpp
class FInventoryCollectionUiState final : public FMarkupObservableObject
{
public:
    FInventoryCollectionUiState()
    {
        RegisterProperty(TEXT("Items"), ItemsProperty, &FInventoryCollectionUiState::GetItems, &FInventoryCollectionUiState::SetItems);
        RegisterProperty(TEXT("Gold"), GoldProperty, &FInventoryCollectionUiState::GetGold, &FInventoryCollectionUiState::SetGold);
    }

    MARKUP_ARRAY_PROPERTY(FItemView, Items)
    MARKUP_PROPERTY(int32, Gold, 0)

    // 业务函数调用 GetItems().Add、GetItems().RemoveByKey、
    // GetItems().Move 或 GetItems().Replace。
};
```

物品键由 `FItemView` 的稳定 `Id` 成员提供。模型挂载时指定根对象即可：

```cpp
Ui->GetDocument()->SetDataContext(InventoryState);
```

若属性的 Setter 包含钳制、校验或联动其他状态，页面写入会经过这个 Setter，与 C++ 业务调用保持一致。需要执行“保存”“购买”等独立意图时仍应使用可绑定命令。纯 `WriteOnly` 字段不可作为原生 `data-value` 或 `data-checked` 的目标，因为两者都需要读取字段值；这类输入应使用 `data-event-*` 命令提交。

### 可绑定命令

命令与属性一样是模型中的受控描述，而不是把成员函数直接暴露给页面。命令在构造函数中显式登记，避免字符串名称、参数表和成员函数各写一遍；`FMarkupCommand` 保存命令名、参数契约与调用函数，只有登记后的命令才能从 RML 调用。

```cpp
class FInventoryCommandUiState final : public FMarkupObservableObject
{
public:
    FInventoryCommandUiState()
    {
        RegisterCommand(TEXT("select_item"), SelectItemCommand);
        RegisterCommand(TEXT("request_close"), RequestCloseCommand);
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

protected:
    FMarkupCommand SelectItemCommand = FMarkupCommand::FromMember(&FInventoryCommandUiState::SelectItem);
    FMarkupCommand RequestCloseCommand = FMarkupCommand::FromMember(&FInventoryCommandUiState::RequestClose);
};
```

命令先作为 `FMarkupCommand` 成员字段声明，再由构造函数中的 `RegisterCommand` 登记。字段只能通过 `FMarkupCommand::FromMember` 绑定非静态成员函数，不提供 Lambda、自由函数或裸 `this` 回调入口。它由成员函数签名生成命令描述，推导参数数量和类型，并在编译期拒绝输出参数、非常量引用、`UObject` 指针、潜伏任务和不支持的值类型。命令描述本身只保存成员函数信息；模型挂载后，基类以弱引用关联模型，在调用时先锁定 `TSharedPtr`，成功后才调用成员函数。反射命令与蓝图命令都必须转换成同一份已验证的命令描述，不能跳过这层检查。

### 宏总表

| 类定义宏 | 解决的问题 | 是否可被反射/蓝图复用 |
| --- | --- | --- |
| `MARKUP_PROPERTY(Type, Name[, DefaultValue])` | 生成受保护的属性字段、Getter、Setter。 | 是；反射层可将其描述为普通字段。 |
| `MARKUP_PROPERTY_READONLY(Type, Name[, DefaultValue])` | 生成受保护的属性字段和公开 Getter；不生成 Setter。 | 是。 |
| `MARKUP_OBJECT_PROPERTY(Type, Name[, DefaultValue])` | 声明嵌套对象属性。 | 是。 |
| `MARKUP_OBJECT_PROPERTY_READONLY(Type, Name[, DefaultValue])` | 声明只读嵌套对象属性；不生成 Setter。 | 是。 |
| `MARKUP_OBJECT_PROPERTY_WRITEONLY(Type, Name[, DefaultValue])` | 声明仅可替换整个对象的属性；不生成 Getter。 | 是；可映射到反射 Setter 或蓝图函数。 |
| `MARKUP_ARRAY_PROPERTY(Type, Name[, DefaultValue])` | 声明受管集合属性。 | 是。 |
| `MARKUP_ARRAY_PROPERTY_READONLY(Type, Name[, DefaultValue])` | 声明只读受管集合属性；不生成 Setter。 | 是。 |
| `MARKUP_ARRAY_PROPERTY_WRITEONLY(Type, Name[, DefaultValue])` | 声明仅可替换整个受管数组的属性；不生成 Getter。 | 是；可映射到反射 Setter 或蓝图函数。 |
| `MARKUP_PROPERTY_WRITEONLY(Type, Name[, DefaultValue])` | 生成受保护的属性字段和公开 Setter；不生成 Getter。 | 是；可映射到反射 Setter 或蓝图函数。 |
| `FMarkupCommand SelectItemCommand = FMarkupCommand::FromMember(...)` | 声明成员函数命令。 | 是；可映射到 `UFUNCTION`。 |

`RegisterProperty` 与 `RegisterCommand` **只能写在派生类自己的构造函数体内**，不能写在普通成员函数、静态函数或类定义体中。类定义宏**只能写在类定义体内**，不能写进构造函数体或任何普通函数体。所有类定义属性宏的末尾可选参数都是 `DefaultValue`；它必须与属性类型匹配，且始终位于最后。两者刻意分离：前者完成运行时登记，后者声明 C++ 属性接口。

宏应生成可阅读的 C++ 接口，而不是藏起业务逻辑。需要校验或联动时，开发者可手写属性字段、Getter、Setter，并将这些成员函数传给 `RegisterProperty`；页面写入将调用该 Setter。不能用属性 Setter 承载“保存”“购买”等独立意图，仍应使用命令。

### 文档与元素数据上下文

元素数据上下文是 MarkupUI 在 RmlUi 数据绑定之上的局部更新路径。C++ 不声明字段到 DOM 表现的映射，而是把可绑定对象或数组设置为文档或元素的上下文，由该子树内的原生 RML 绑定表达式显示数据。

RML 作者为需要单独设置上下文的元素提供稳定、唯一的 `name`：

```html
<div data-model="inventory">
  <span name="hud-gold"></span>

  <div name="player-health" class="health-bar">
    <span name="player-health-label"></span>
  </div>

  <div name="inventory-list"></div>
</div>
```

C++ 只设置文档或元素的数据上下文；上下文只能是 `FMarkupObservableObject` 或 `FMarkupObservableArray`：

```cpp
// 伪代码：文档和元素都提供这两类重载；数组重载对任意 Item 类型可用。
void SetDataContext(TSharedPtr<FMarkupObservableObject> Context);
template<typename Item>
void SetDataContext(TSharedPtr<FMarkupObservableArray<Item>> Context);
```

```cpp
SRmlUiWidget->GetDocument()->SetDataContext(InventoryState);

SRmlUiWidget->GetDocument()
    ->GetElementByName(TEXT("player-health"))
    .SetDataContext(PlayerHealthState);
```

文档上下文是默认上下文；元素上下文覆盖该元素及其子树继承到的上下文。RML 内仍使用原生文本、属性、样式和 `data-for` 绑定表达式；格式化值应作为可绑定属性提供，而不是由绑定 API 传入 Lambda。替换元素上下文时，只有该元素子树重新计算绑定。

不提供从 C++ 直接绑定文本、属性、样式、类、列表或任意 DOM 子树的 API。这些表现完全由 RML 元素自身的绑定表达式描述；C++ 的职责仅是设置可观察数据上下文。

默认也不提供“把任意字符串传给 `SetInnerRML`”的通用数据绑定。它会重新解析标记，可能破坏焦点、输入状态和元素身份，也可能把未受信任文本当作界面标记。

### 数组数据上下文与 `data-for`

`data-for` 是 RmlUi 的原生列表方案。将 `FMarkupObservableArray` 设置为列表元素的数据上下文后，该元素子树用 `data-for` 渲染数组；C++ 不提供单独的列表创建、更新、删除或移动回调。

```cpp
SRmlUiWidget->GetDocument()
    ->GetElementByName(TEXT("inventory-list"))
    .SetDataContext(InventoryState->GetItems());
```

每个条目仍应有稳定且唯一的业务标识，例如物品实例 ID，而不是数组下标。数组发生增删、移动或替换后，`data-for` 依照 RmlUi 的原生规则管理该子树；C++ 不得手工改写这个受管子树，也不应跨更新周期保存其中元素的裸指针。

### 与原生 RmlUi 数据绑定的边界

一个元素子树只能有一个直接数据上下文；子元素可显式设置新的上下文以覆盖继承值。下表是硬规则：

| 场景 | 是否允许 | 原因 |
| --- | --- | --- |
| 文档上下文与元素上下文同时存在 | 允许 | 元素上下文覆盖该元素及其子树继承到的文档上下文。 |
| 同一元素重复设置上下文 | 允许 | 后一次设置替换前一次设置，并解除旧上下文订阅。 |
| `data-for` 容器由 C++ 手工增删子元素 | 不允许 | 列表结构只由 RmlUi 的 `data-for` 管理。 |
| 子元素需要独立刷新范围 | 允许 | 为该子元素设置自己的可观察对象或数组上下文。 |
| 可写字段用于 `data-value` 等输入绑定 | 允许 | 输入和展示均由该上下文内的原生 RML 绑定表达式管理。 |

严格模式应验证通过 `GetElementByName` 查询的目标存在且名称唯一，并拒绝向已失效或已卸载文档中的元素设置上下文。

### 数据上下文更新后的时序和限制

数据上下文的属性或集合通知会在正常的文档更新阶段统一驱动 RML 绑定重新求值；若业务代码立刻读取尺寸、偏移等布局结果，可能读到上一次结果。只有确实需要同步读取布局时，才允许显式更新文档，并应把它视为有性能代价的高级操作。

数据上下文不直接改写 RmlUi 维护的 `hover`、`active`、`focus`、`checked`、`disabled` 等内置伪类。项目可定义自己的伪类，例如 `low-health`，但表单控件的焦点、选择和可用状态仍由原生控件/绑定机制负责。

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

批量更新的边界属于文档，不属于单个 `FMarkupObservableObject`。这样一次业务操作可以同时改动多个对象、多个数组和多个元素数据上下文：

```cpp
auto UpdateScope = SRmlUiWidget->GetDocument()->BeginDataUpdate();

InventoryState->EditItems().Add(NewItem);
InventoryState->SetGold(NewGold);
PlayerState->SetHealth(NewHealth);
```

`UpdateScope` 在离开作用域时提交。文档收集这段期间来自其已挂载数据上下文的变化，并按“数据模型 + 顶层字段”去重后提交给 RmlUi。例如 `Gold` 连续修改三次，或多个对象都影响同一顶层字段 `inventory`，在本次提交中都只标记一次。

更新作用域可以嵌套；只有最外层作用域结束时才提交。它不立即绘制或强制调用 RmlUi 更新，RML 仍在正常的文档更新阶段统一重新求值。作用域只合并当前文档的 RML 刷新请求，不阻止 `FMarkupObservableObject` 向游戏逻辑、存档或其他订阅者发送自身的变化通知。

不使用 `BeginDataUpdate()` 时，属性变化仍会自动排入当前文档下一次正常更新；显式作用域只用于希望将一组跨对象改动作为一次 RML 提交的场合。视图会忽略没有实际变化的子值，但插件不承诺单个数组元素的独立刷新。背包、任务列表这类数据建议以顶层数组为刷新边界。

### C++ 原生命令

命令通过构造函数中的 `RegisterCommand` 与 `FMarkupCommand::FromMember` 登记在 `FMarkupObservableObject` 派生类中。它适合将 UI 意图交给控制器、子系统或游戏功能模块，而不要求它们是 `UObject`。命令参数由成员函数签名确定；对象挂载到模型时会自动出现在命令字典中。命令不捕获裸指针：调用前必须先锁定模型的 `TSharedPtr`，模型或目标对象关闭后命令安全地不执行。

## 双向绑定的完整规则

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
        RegisterProperty(TEXT("music_volume"), MusicVolumeProperty, &FMusicSettingsUiState::GetMusicVolume, &FMusicSettingsUiState::SetMusicVolume);
        RegisterProperty(TEXT("enable_music"), EnableMusicProperty, &FMusicSettingsUiState::GetEnableMusic, &FMusicSettingsUiState::SetEnableMusic);
    }

    MARKUP_PROPERTY(float, MusicVolume, 0.8f)
    MARKUP_PROPERTY(bool, EnableMusic, true)
};
```

RML 使用这个公开属性：

```html
<label for="music-volume">音乐音量</label>
<input id="music-volume" type="range" min="0" max="1" step="0.01"
       data-value="settings.music_volume"/>
<span>{{ settings.music_volume }}</span>
```

挂载时，`settings` 指向这个对象：

```cpp
TSharedRef<FMusicSettingsUiState> Settings = MakeShared<FMusicSettingsUiState>();
Ui->GetDocument()->SetDataContext(Settings);
```

| 谁先改变 | 示例代码或操作 | 结果 |
| --- | --- | --- |
| C++ / 游戏设置 | `Settings->SetMusicVolume(0.65f);` | `MusicVolume` 变化并通知；下一次文档更新时，滑条位置和 `<span>` 都显示 `0.65`。 |
| 玩家 | 将滑条拖到 `0.30`，控件按正常 `change` 时机提交 | 输入值转换为 `float` 后写入 `MusicVolume`；写入成功时，模型值变为 `0.30`，随后控件和 `<span>` 显示模型最终值。 |
| 玩家输入非法值 | 例如非数字、超出字段允许范围，或属性只读 | 写入被拒绝或修正；控件不保留未接受的值，而是显示模型中的有效值。 |

因此，双向绑定不是“控件永远相信自己的输入”，而是控件向可写属性提出写入请求，模型决定接受、拒绝或修正；模型值始终是最终来源。

`data-value` 只绑定单个可写标量路径；不要在这里写表达式，例如 `data-value="volume * 100"` 是不允许的。若 UI 显示的是百分比但存储的是 0 至 1，可增加只供展示的字段，或在字段 Write 中处理，而不是让 RML 推断反向公式。

### `data-value` 与 `data-checked`

`data-value` 同步的是元素的 `value` 属性；`data-checked` 同步的是复选框或单选框的 `checked` 状态。两者不能互换。

复选框同时具有独立的 `value` 与 `checked`：前者表示该选项的提交值，通常是字符串；后者才表示玩家是否勾选。因此，不应使用 `data-value` 绑定布尔开关：

```html
<!-- 错误：绑定的是 value，不是勾选状态。 -->
<input type="checkbox" value="enable_music" data-value="settings.enable_music"/>

<!-- 正确：true 为勾选，false 为取消勾选。 -->
<input type="checkbox" data-checked="settings.enable_music"/>
```

当 `settings.enable_music` 为 `true` 时，复选框勾选；为 `false` 时取消勾选。玩家切换复选框后，控件在正常 `change` 时机将新的 `bool` 写回模型。

单选组的 `checked` 状态由每个选项的 `value` 与同一个字段比较决定：

```html
<input type="radio" name="difficulty" value="easy"
       data-checked="settings.difficulty"/> 简单
<input type="radio" name="difficulty" value="hard"
       data-checked="settings.difficulty"/> 困难
```

这里的 `settings.difficulty` 应是字符串或已声明的枚举；其值等于某个单选框的 `value` 时，该选项被选中。对枚举，字段定义必须给出允许项，不能接受任意字符串。

### 字段方向和提交时机

WPF 的“绑定模式”和“写回时机”是很有价值的产品概念，但 MarkupUI 不应为了模仿 XAML 而给 RML 另造一套绑定属性。每个可绑定属性创建时都具有属性访问控制。手写属性时显式传入控制值；使用宏时，由选择的宏确定控制值：

```cpp
FMarkupProperty MusicVolumeProperty = MakeMarkupProperty<float>(0.8f, EMarkupPropertyAccess::ReadWrite);
MARKUP_PROPERTY_READONLY(int32, Gold, 0)
```

| 属性访问控制 | RML 能读取 | RML 能写入 | 适用场景 |
| --- | --- | --- | --- |
| `ReadOnly` | 是 | 否；写入请求被拒绝。 | 金币、生命值、物品列表。 |
| `ReadWrite` | 是 | 是。 | 设置滑条、复选框、普通表单。 |
| `WriteOnly` | 否 | 是；页面可在事件中提交值。 | 搜索词、临时筛选条件、仅接受提交的数据。 |

属性访问控制只属于模型：它决定读取或写入请求是否被接受，**不决定页面选择何种绑定方向**。页面可声明任意方向；若方向请求的读取或写入不被模型允许，该请求被拒绝，模型值不会改变。

属性访问控制只约束 RML 绑定过程。游戏 C++ 代码或蓝图仍可按自身业务逻辑直接修改模型属性；修改后，具有读取方向的页面绑定会收到更新。

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

以下模型分别公开一个只读开关和一个只写开关：

```cpp
class FBindingAccessUiState final : public FMarkupObservableObject
{
public:
    FBindingAccessUiState()
    {
        RegisterProperty(TEXT("locked_option"), LockedOptionProperty, &FBindingAccessUiState::GetLockedOption);
        RegisterProperty(TEXT("incoming_option"), IncomingOptionProperty, nullptr, &FBindingAccessUiState::SetIncomingOption);
    }

    MARKUP_PROPERTY_READONLY(bool, LockedOption, true)
    MARKUP_PROPERTY_WRITEONLY(bool, IncomingOption, false)
};
```

页面仍可请求与模型访问控制冲突的方向，但模型始终拒绝越权请求：

```html
<!-- 冲突一：data-checked 是双向的，但 locked_option 是 ReadOnly。 -->
<input type="checkbox" data-checked="settings.locked_option"/>

<!-- 冲突二：该绑定属性要读取值，但 incoming_option 是 WriteOnly。 -->
<input type="checkbox" data-attrif-checked="settings.incoming_option"/>

<!-- 合法：只在 change 时把值写入 WriteOnly 属性。 -->
<input type="checkbox"
       data-event-change="settings.incoming_option = ev.checked"/>
```

第一个控件会先显示 `locked_option` 的当前值；玩家切换后，写入被拒绝，模型值保持不变。由于该控件仍具有读取方向，MarkupUI 随后重新读取模型值并恢复控件显示。

第二个控件的读取请求被拒绝：它不会从 `incoming_option` 获得初始或后续状态，开发环境应报告访问控制冲突。第三个控件没有读取方向，`change` 时的写入被接受；它适合只接收玩家输入、而不允许模型反向覆盖控件内容的场景。

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

### 事件中的赋值表达式与命令

常见的“修改设置后保存”应将字段绑定和保存命令分开：

```html
<input type="checkbox" data-checked="settings.show_minimap"/>
<button data-event-click="save_settings()">保存设置</button>
```

`data-checked` 在 checkbox 的 `change` 时写回 `settings.show_minimap`；点击“保存设置”时只执行已登记的 `save_settings()` 命令。

`data-event-*` 触发时也可以应用赋值表达式。若产品需要单独的“切换小地图”按钮，可以这样写：

```html
<button data-event-click="settings.show_minimap = !settings.show_minimap">
  切换小地图
</button>
```

这条赋值表达式会将当前布尔值取反：开启变关闭，关闭变开启。赋值只会作用于模型允许写入的字段；页面也可在同一事件属性中用分号按顺序组合多个赋值表达式与已登记命令。未知命令、错误数量的参数或不匹配的参数类型都不会执行任何游戏逻辑。

### 批量更新与循环预防

从 RML 写回字段后，模型会把该字段标记为已处理。游戏状态随后发出的通知仅用于把最终值同步回控件，不应再次触发同一命令或产生无限循环。业务代码应只修改状态，不要在字段通知中模拟一次新的按钮点击。

## 蓝图支持

蓝图与 UMG 可直接将业务 `UObject` 设为 RML 文档或元素的数据上下文。绑定路径使用 `UPROPERTY` 的反射名称；页面事件调用数据上下文上的 `UFUNCTION` 成员。

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
    void SetGold(int32 InGold);
    void RemoveInventoryItemAt(int32 Index);

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    int32 Gold = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    TArray<FName> InventoryItems;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Inventory")
    TMap<FName, int32> ItemCounts;

    UFUNCTION(BlueprintCallable, Category = "Inventory")
    void RequestClose();
};
```

### 设置数据上下文完成绑定

```cpp
UMarkupInventoryData* InventoryData = NewObject<UMarkupInventoryData>();
Ui->GetDocument()->SetDataContext(InventoryData);
```

```html
<span>{{ Gold }}</span>
<div data-for="InventoryItem : InventoryItems">{{ InventoryItem }}</div>
<div data-for="ItemCount : ItemCounts">{{ ItemCount }}</div>
```

### 普通 `UPROPERTY` 的更新

普通 `UPROPERTY` 的读取和写入都作用于该 `UObject` 上的原始属性。`USTRUCT` 是属性值的一部分，页面可使用 `Attributes.Health` 访问嵌套成员；结构内部被业务代码改动时，通知其顶层属性。

普通属性的蓝图节点如下：

| 节点 | 作用 |
| --- | --- |
| Set Data Context | 将一个 `UObject` 设置为 RML 文档或指定元素的数据上下文。 |
| `Set <属性名> (Markup UI)` | 写入该对象的普通 `UPROPERTY`，再通知 RML 刷新依赖此属性的绑定。 |
| Notify Markup Property Changed | 原生 Set 或 C++ 已经改值后，显式通知指定属性。 |
| `BeginDataUpdate` | `URmlUiWidget` 开始合并其当前 Document 中的属性刷新。 |
| `EndDataUpdate` | `URmlUiWidget` 提交当前 Document 合并后的属性刷新。 |

`Set <属性名> (Markup UI)` 在写入成功后通知 MarkupUI 刷新该属性的绑定；普通原生 Set 不自动触发此通知。C++ 直接通过 `SRmlUiWidget->GetDocument()->BeginDataUpdate()` 创建更新作用域；蓝图无法直接持有该 C++ Document 对象，因此 `BeginDataUpdate` 和 `EndDataUpdate` 作为 `URmlUiWidget` 的蓝图函数转发到其当前 Document。

### `TArray` 属性的更新

会修改数组的蓝图节点如下：

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

数组变更节点必须能够追溯到具体 `UObject` 上的数组属性，以确定变更后应通知的对象与属性名。临时数组、函数返回数组或来源不明的数组不属于可通知的数据上下文属性，不应被当作 MarkupUI 绑定集合处理。

操作完成后，节点通知整个数组属性；随后 `data-for` 按 RmlUi 的原生规则更新子树。只读查询节点不产生通知。

### `TMap` 属性的更新

首版提供以下会改变 Map 的节点：

| 节点 | 作用 |
| --- | --- |
| Add (Markup UI) | 新增或覆盖一个键值对，并通知 Map 属性。 |
| Remove (Markup UI) | 按键移除一个键值对，并通知 Map 属性。 |
| Clear (Markup UI) | 清空 Map，并通知 Map 属性。 |

Map 变更节点同样必须能够追溯到具体 `UObject` 上的 Map 属性；执行完成后通知整个 Map 属性。Add 覆盖已有键、Remove 未找到键时仍可发出一次无害通知。Find、Contains、Length、Keys、Values 等只读操作不通知。

### 命令

页面事件可以调用数据上下文上允许绑定的 `UFUNCTION` 成员，例如：

```rml
<button data-event-click="SaveSettings()">保存设置</button>
<button data-event-click="ShowMinimap = !ShowMinimap">切换小地图</button>
```

第一个例子调用对象成员函数；第二个例子只应用一个属性表达式。命令执行后如修改了属性或集合，应使用对应的通知机制刷新页面。

## 在 C++ 中进行 `UObject` 变化通知

`UObject` 实例（指针）可以通过宏修改 `UPROPERTY` 属性，宏修改会自动发送属性变化通知来更新已绑定到当前对象的页面内容。

所有宏都以 `Object` 和 `Member` 为前两个参数；例如 `InventoryData` 与 `&UMarkupInventoryData::InventoryItems`。`Member` 传入成员指针，而不是属性名字符串。

### 普通属性宏

| 宏 | 作用 |
| --- | --- |
| MARKUP_PROPERTY_SET(Object, Member, Value) | 写入普通属性；值实际变化后通知该属性。 |
| MARKUP_NOTIFY_PROPERTY_CHANGED(Object, Member) | 属性已经由业务代码写入后，通知该属性。 |

### 数组宏

| 宏 | 作用 |
| --- | --- |
| MARKUP_ARRAY_ADD(Object, Member, Item) | 在末尾添加元素，并记录新增变更。 |
| MARKUP_ARRAY_ADD_UNIQUE(Object, Member, Item) | 不存在相同元素时添加，并记录新增变更。 |
| MARKUP_ARRAY_APPEND(Object, Member, AppendedItems) | 追加传入集合中的元素，并记录新增变更。 |
| MARKUP_ARRAY_INSERT(Object, Member, Item, Index) | 在指定位置插入元素，并记录插入变更。 |
| MARKUP_ARRAY_REMOVE_AT(Object, Member, Index) | 移除指定索引的元素，并记录删除变更。 |
| MARKUP_ARRAY_REMOVE_ITEM(Object, Member, Item) | 移除匹配元素，并记录删除变更。 |
| MARKUP_ARRAY_REPLACE(Object, Member, Index, Item) | 替换指定索引的元素，并记录替换变更。 |
| MARKUP_ARRAY_RESIZE(Object, Member, Size) | 调整数组长度，并记录新增或删除变更。 |

### Map 宏

| 宏 | 作用 |
| --- | --- |
| MARKUP_MAP_ADD(Object, Member, Key, Value) | 新增键值对，并记录新增变更。 |
| MARKUP_MAP_REPLACE(Object, Member, Key, Value) | 替换已有键的值，并记录替换变更。 |
| MARKUP_MAP_REMOVE(Object, Member, Key) | 移除指定键，并记录删除变更。 |

例如：

```cpp
void UpdateInventory(
    UMarkupInventoryData* InventoryData,
    int32 NewGold,
    const FName& NewItem,
    const TArray<FName>& NewItems,
    int32 Index,
    FName Key,
    int32 Count)
{
    MARKUP_PROPERTY_SET(InventoryData, &UMarkupInventoryData::Gold, NewGold);

    MARKUP_ARRAY_ADD(InventoryData, &UMarkupInventoryData::InventoryItems, NewItem);
    MARKUP_ARRAY_APPEND(InventoryData, &UMarkupInventoryData::InventoryItems, NewItems);
    MARKUP_ARRAY_REPLACE(InventoryData, &UMarkupInventoryData::InventoryItems, Index, NewItem);

    MARKUP_MAP_ADD(InventoryData, &UMarkupInventoryData::ItemCounts, Key, Count);
    MARKUP_MAP_REPLACE(InventoryData, &UMarkupInventoryData::ItemCounts, Key, Count);
}
```

### 原生 C++ 通知函数

当无法通过 `UPROPERTY` 触发通知时（例如 `UPROPERTY` 字段**不是** public），还可以使用 `MarkupUI` 命名空间中的通知函数。函数通过成员名称（优先使用 UE 的 `GET_MEMBER_NAME_CHECKED` 获取）通知成员变化。

> `UPROPERTY` 字段**不是** public 时，你无法使用 `GET_MEMBER_NAME_CHECKED` 获取名称, 也可以传递字符串名称，字符串不是类型安全的，当字段名修改后不要忘记更新你的通知代码。

#### 普通属性函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyPropertyChanged(Object, MemberName) | 通知普通属性变化；数组或字典进行清空、排序、交换、反转或打乱等整体变更后，也使用此函数。 |

#### Array 函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyArrayAdded(Object, MemberName, Index) | 通知在末尾添加了一个元素。 |
| MarkupUI::NotifyArrayAppended(Object, MemberName, NewCount) | 通知在末尾追加了多个元素。 |
| MarkupUI::NotifyArrayInserted(Object, MemberName, Index, Count) | 通知从指定索引插入了一个或多个元素。 |
| MarkupUI::NotifyBeginArrayDelete(Object, MemberName, Index) | 在删除指定索引处元素前调用。 |
| MarkupUI::NotifyEndArrayDelete(Object, MemberName) | 删除完成后调用。 |
| MarkupUI::NotifyBeginArrayReplace(Object, MemberName, Index) | 在替换指定索引处元素前调用。 |
| MarkupUI::NotifyEndArrayReplace(Object, MemberName) | 替换完成后调用。 |

#### Map 函数

| 函数 | 作用 |
| --- | --- |
| MarkupUI::NotifyMapAdded(Object, MemberName, Key) | 通知新增了指定键。 |
| MarkupUI::NotifyMapReplaced(Object, MemberName, Key) | 通知替换了指定键对应的值。 |
| MarkupUI::NotifyMapRemoved(Object, MemberName, Key) | 通知移除了指定键。 |

**注意**

 - 单步通知函数：（没有 `Begin` / `End`）更新结束后调用。
 - 两步通知函数： `Begin` / `End` 成对出现的通知函数：应该再更新数据前调用 `Begin`, 更新完成后调用 `End`， 

例如，已有业务接口修改受保护属性后，可直接通知：

```cpp
void UpdateGoldThroughBusinessApi(
    UMarkupInventoryData* InventoryData,
    int32 NewGold)
{
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
    MarkupUI::NotifyBeginArrayDelete(
        InventoryData,
        GET_MEMBER_NAME_CHECKED(UMarkupInventoryData, InventoryItems),
        Index);

    InventoryData->RemoveInventoryItemAt(Index);

    MarkupUI::NotifyEndArrayDelete(
        InventoryData,
        GET_MEMBER_NAME_CHECKED(UMarkupInventoryData, InventoryItems));
}
```

所有集合宏均在一次调用中完成容器操作和变更记录。数组和 Map 宏保留新增、插入、删除、替换等集合变更明细。当前 RML 绑定层可以将其作为整个属性变化处理；将来 UI 层支持局部集合更新时，仍可直接使用同一份变更记录，不要求游戏代码改写。

## Slate（C++）使用草案

Slate 使用者创建 `FMarkupObservableObject` 派生模型，并直接挂载到 Widget。

```cpp
TSharedRef<SRmlUiWidget> Ui = SNew(SRmlUiWidget)
    .DocumentContent(InventoryRml)
    .DocumentSourceUri(TEXT("/Game/UI/Inventory.rml"));

TSharedRef<FInventoryCollectionUiState> InventoryState =
    MakeShared<FInventoryCollectionUiState>();
Ui->GetDocument()->SetDataContext(InventoryState);
```

每个 `SRmlUiWidget` 必须拥有单独的模型实例。多个玩家同时打开同一份背包界面时，不能共用同一模型，除非该模型明确只读且不会保存任何界面局部状态。

## UMG（蓝图）使用草案

`URmlUiWidget` 是可放入 Widget Blueprint 的控件。Widget Blueprint 或拥有者在构造、初始化时取得实际业务 `UObject`，直接调用 `Set Data Context`。

推荐工作流：

1. 取得当前角色、组件或其他业务 `UObject`。
2. 将该对象设置给 RML 文档或指定元素。
3. 写入普通属性时使用 `Set <属性名> (Markup UI)`；修改数组或 Map 时使用对应的 `(Markup UI)` 集合操作节点。
4. 业务对象通过原生 Set 或 C++ 修改属性后，调用 `Notify Markup Property Changed`；跨多个对象或多个属性合并更新时，在 `URmlUiWidget` 上调用 `BeginDataUpdate` 和 `EndDataUpdate`。
5. 目标对象销毁后，数据上下文自动失效，无需手动保活。

Designer 预览使用安全的预览对象，不得保存设置、触发命令或修改游戏世界。蓝图用户不需要通过 JSON 字符串填充数据；属性、数组和 Map 都沿用原始 `UObject` 属性的实际类型。

普通属性、数组和 Map 节点的完整清单及其通知行为见“蓝图支持”中的对应小节。节点基于实际 `UObject` 的反射属性生成；从对象的属性引脚或右键菜单进入，分类、名称和引脚规则均与对应原生节点一致。

## 生命周期、重载与线程

对使用者可见的生命周期是：

```text
创建控件并加载 RML 文档
  → 取得业务 UObject
  → SetDataContext 设置文档或元素的数据上下文
  → 游戏状态与用户输入双向同步
  → 文档重载、控件销毁或 UObject 销毁时，旧绑定和命令失效
```

数据上下文必须在文档首次计算依赖它的绑定前设置完成。文档重载会创建新的绑定关系；旧命令回调和旧控件输入状态均失效。若需要保留搜索词、滚动位置或草稿，应由游戏保存这些值，再在新文档设置 `UObject` 后提交。

所有公开的模型写入、反射读取、输入处理和命令回调都要求在游戏线程调用。异步加载、网络回调或后台任务应切回游戏线程后再更新视图模型。调用已销毁模型应返回失败结果而非崩溃。

## 错误、调试与安全

开发环境应提供绑定检查器，按界面实例显示：数据上下文、公开字段、读写权限、字段类型、最近刷新字段、最近的输入拒绝原因和最近命令。Shipping 不显示实际敏感数据。

日志或检查器中的错误必须包含文档标识、数据上下文和字段/命令名。至少覆盖：

- RML 没有数据上下文、或使用了不存在、不允许绑定的字段或命令；
- RML 尝试写不允许写入的字段；
- 值转换、范围校验、枚举校验或 Setter 拒绝；
- `UFUNCTION` 参数不符合命令契约；
- 不支持的反射类型或循环引用；
- 数据上下文已失效、目标对象已销毁、或从非游戏线程调用；
- 同一元素重复设置互相冲突的数据上下文。

开发环境推荐把“RML 引用了不存在字段”作为明显警告，并以安全默认值显示；Shipping 保持安全默认值且不暴露对象信息。是否让该问题在严格模式下直接阻止文档加载，应作为项目设置提供。

## 自动化验收清单

- 同一份 RML 在 Slate 与 UMG 中，对同一模型数据得到一致的文字、属性、可见性和列表内容。
- 模型宏生成的 Getter、Setter、只读权限、浮点比较、嵌套对象通知、集合通知和命令签名，与手写原生模型具有相同结果。
- 标量、对象、嵌套结构、数组、枚举、`FText` 与受限映射的转换正确；不支持的值明确失败。
- `UObject` 数据上下文能通过字段通知或手动 `Notify Markup Property Changed` 刷新正确的属性；数组成员变化要求通知数组属性。
- `Set Data Context` 直接接受角色、组件或其他业务 `UObject`；不创建代理对象、镜像属性或第二份需要维护的蓝图资产。
- `Set <属性名> (Markup UI)` 与原生 Set Var 在标题、输入/输出顺序、类型颜色、属性分类和连接方式上保持一致；选择器名称与右上角标识可明确区分它。
- 数组与 Map 的 `(Markup UI)` 修改节点尊重原生函数的名称、分类、默认输入、泛型推导和引用方向；只接受同一 `UObject` 属性的 Get 或 Set 输出，并在完成后只通知该属性。
- 同一 `UObject` 可安全设置到多个文档或元素；目标对象销毁后，所有关联绑定安全失效且不保活该对象。
- `data-value`、`data-checked` 覆盖正常写入、类型错误、范围修正、业务拒绝和外部更新冲突。
- 输入法合成、文本选择与焦点期间不被外部刷新错误打断。
- 文档与元素的 `SetDataContext` 覆盖、继承、替换和卸载后解除订阅均符合预期；元素上下文变化不影响无关子树。
- 数组数据上下文与 `data-for` 覆盖条目变更、插入、删除、移动、文档重载和滚动状态；C++ 不得手工修改其受管子树。
- 严格模式拒绝不存在或重复名称的元素目标，以及向失效文档或元素设置数据上下文。
- C++ 回调与 `UFUNCTION` 命令均覆盖参数成功、参数不匹配、目标销毁和重复点击。
- 文档重载、控件销毁、多个 Widget、多本地玩家不会串用模型、输入状态或命令。
- UMG Designer 不触发命令或写回真实游戏对象。

## 分阶段实施建议

1. **模型基础与宏**：统一值类型、属性访问器、顶层脏标记、可观察对象/集合宏和 Slate 显式模型；先验证展示与原生 `data-for` 列表。
2. **文档与元素数据上下文**：实现 `SRmlUiWidget` 文档上下文、元素上下文覆盖、子树订阅、跨对象数据更新作用域和数组 `data-for` 回归；它是实现细粒度刷新的关键阶段。
3. **UE 反射数据上下文**：实现 `SetDataContext(UObject*)`、反射属性读写、嵌套结构转换、字段通知和显式属性通知。
4. **蓝图通知节点与命令**：实现普通属性 Set、Array／Map 修改节点、原生节点转换、受限反射成员函数命令、值校验和文本输入冲突规则。
5. **UMG 支持**：提供 `UMarkupUiWidget` 与 `SetDataContext` 的蓝图节点、事件委托和 Designer 预览；与 Slate 共用回归用例。
6. **可选互操作**：评估与 UE MVVM 字段通知的更深整合、可复用数据源、多模型协调和 `data-rml`。这些均不能破坏当前白名单和双向写入契约。

## 尚待产品确认的选择

- 哪些 `UPROPERTY` 与 `UFUNCTION` 可由反射暴露给页面？建议采用明确的 MarkupUI 元数据或白名单；运行时绝不自动暴露目标对象的全部成员。
- 数值超出范围时，默认修正还是默认拒绝？建议由字段声明决定：设置类字段通常修正，交易、权限和数量类字段通常由 Setter 决定。
- 严格模式下，不存在或未允许的字段是否阻止文档加载？建议开发/测试环境可配置为阻止，Shipping 使用安全默认值并记录汇总诊断。
- 是否提供每键同步的蓝图便捷节点？建议只提供受节流的专用命令，不把它作为普通 `data-value` 的默认行为。

## 参考资料

- [RmlUi：数据绑定](https://mikke89.github.io/RmlUiDoc/pages/data_bindings.html)
- [RmlUi：元素与 DOM 接口](https://mikke89.github.io/RmlUiDoc/pages/cpp_manual/elements.html)
- [Noesis UI：数据绑定](https://www.noesisengine.com/docs/Gui.Core.DataBindingTutorial.html)
