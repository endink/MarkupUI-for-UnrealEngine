# 蓝图 DOM

在 UMG 中添加 **MarkupUI** Widget，并设置 HTML Document 资产，或调用 **Set Html**。收到 **On Dom Ready** 后，再调用 **Get Root Element**。加载完成前返回的元素状态为 **DocNotReady**。

## 查找和修改

从根元素调用 QuerySelector 或 GetElementById。元素上的查询只查该元素范围内；GetClosest 向自身和祖先查找。把结果保存为 Element 变量，可以反复使用、复制和传给其他蓝图函数。

```text
On Dom Ready
  → Get Root Element
  → GetElementById ("title")
  → 保存 Title Element
  → SetInnerHtml (Title Element, "新的标题")
```

SetInnerHtml 保持文本自动转义，输入 "<b>标题</b>" 会显示这段文字。可用节点还包括 Get Children、Get Child Count、QuerySelectorAll、GetElementsByClassName、Matches Selector、GetClosest、Get Parent，以及属性、样式和子元素修改。

修改和取值节点使用输入变量自身记录错误。用 Is Element Valid 检查本地状态，用 Get Element State 获取具体错误。已有错误的元素会继续传递该错误。重新从有效根元素查询，可得到新的操作结果。Is Valid 不向文档发起查询；元素已被删除时，下一次实际操作会记录失效。

## 表单控件

```text
GetElementById ("name")
  → As Input
  → 保存 Name Input
  → Set Value (Input) (Name Input, "Alice")
```

Input 支持 Get Input Type、Get/Set Value 和 Get/Set Checked。TextArea 支持 Get/Set Value。Form 支持 Submit Form 和 Reset Form。

As 转换的标签不匹配时，对应的 Is Input/TextArea/Select/Form Valid 返回 false；原 Element 不受影响。转换本身不会报错，实际调用不支持的操作或绑定不支持的事件时会记录包含标签和操作名称的日志。

## 连接事件

公共鼠标、指针、焦点、滚轮等事件使用 **Bind Element Event**。Input 的 Input/Change、TextArea 的 Input/Change、Form 的 Submit/Reset 使用各自的绑定节点。

```text
On Dom Ready
  → Get Root Element
  → GetElementById ("name")
  → As Input
  → Bind Input Event
       Element  ← Name Input
       Callback ← Create Event (选择 OnNameInput)
       Return Value → 保存 Name Subscription

OnNameInput (Event)
  → Break Markup UI Dom Event
  → Value → 更新其他 UI
```

从 Callback 委托引脚拖出 **Create Event**，选择一个接收 Markup UI Dom Event 参数的函数；也可以连接具有同样参数的 Custom Event 委托引脚。事件参数包含 Type、Target、Current Target、Value、Checked、Position 和 Fields。表单 Fields 是保留顺序和重复名称的 Name/Value 数组。

回调在 Game Thread 异步执行，值和表单字段保存事件发生当时的快照。回调中再次读取元素会得到当前值，可能已经发生变化。

保存订阅返回值，可调用 **Unbind DOM Event** 取消。订阅值可以复制，取消其中一份会使其他副本也失效。丢弃订阅变量不会自动取消；重新加载、Clear Document 或移除 Widget 会取消该文档的全部订阅。重复绑定会增加监听，需主动取消旧订阅。

## 当前限制

- Select 的值操作和 Change 事件当前会报告 Unsupported，As Select 仍可检查标签是否匹配。
- 新 MarkupUI Widget 当前转发鼠标、指针、滚轮和焦点；键盘及 IME 文本输入尚未接入，因此不能将文本键入当作已支持的交互。
- 表单提交暂不覆盖完整浏览器表单规范；先使用普通 Input、Checkbox/Radio 和 TextArea，避免依赖文件上传或 Select 提交。
- Element 和订阅是运行时引用，请勿用于存档或网络复制。
