# Go编程模式：委托和反转控制

反转控制[IoC – Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control) 是一种软件设计的方法，其主要的思想是把控制逻辑与业务逻辑分享，不要在业务逻辑里写控制逻辑，这样会让控制逻辑依赖于业务逻辑，而是反过来，让业务逻辑依赖控制逻辑。在《[IoC/DIP其实是一种管理思想](https://coolshell.org/articles/9949.html)》中的那个开关和电灯的示例一样，开关是控制逻辑，电器是业务逻辑，不要在电器中实现开关，而是把开关抽象成一种协议，让电器都依赖之。这样的编程方式可以有效的降低程序复杂度，并提升代码重用。



## 嵌入和委托



### 结构体嵌入

在Go语言中，我们可以很方便的把一个结构体给嵌到另一个结构体中。如下所示：

```go
type Widget struct {
    X, Y int
}
type Label struct {
    Widget        // Embedding (delegation)
    Text   string // Aggregation
}
```

上面的示例中，我们把 `Widget`嵌入到了 `Label` 中，于是，我们可以这样使用：

```go
label := Label{Widget{10, 10}, "State:"}

label.X = 11
label.Y = 12
```

如果在 `Label` 结构体里出现了重名，就需要解决重名，例如，如果 成员 `X` 重名，用 `label.X`表明 是自己的`X` ，用 `label.Wedget.X` 表示嵌入过来的。

有了这样的嵌入，就可以像UI组件一样的在结构构的设计上进行层层分解。比如，我可以新出来两个结构体 `Button` 和 `ListBox`：

```go
type Button struct {
    Label // Embedding (delegation)
}

type ListBox struct {
    Widget          // Embedding (delegation)
    Texts  []string // Aggregation
    Index  int      // Aggregation
}
```



### 方法重写

然后，我们需要两个接口 `Painter` 用于把组件画出来，`Clicker` 用于表明点击事件：

```go
type Painter interface {
    Paint()
}
 
type Clicker interface {
    Click()
}
```

当然，

- 对于 `Lable` 来说，只有 `Painter` ，没有`Clicker`
- 对于 `Button` 和 `ListBox`来说，`Painter` 和`Clicker`都有。

下面是一些实现：

```go
func (label Label) Paint() {
  fmt.Printf("%p:Label.Paint(%q)\n", &label, label.Text)
}

//因为这个接口可以通过 Label 的嵌入带到新的结构体，
//所以，可以在 Button 中可以重载这个接口方法以
func (button Button) Paint() { // Override
    fmt.Printf("Button.Paint(%s)\n", button.Text)
}
func (button Button) Click() {
    fmt.Printf("Button.Click(%s)\n", button.Text)
}


func (listBox ListBox) Paint() {
    fmt.Printf("ListBox.Paint(%q)\n", listBox.Texts)
}
func (listBox ListBox) Click() {
    fmt.Printf("ListBox.Click(%q)\n", listBox.Texts)
}
```

这里，需要重点提示一下，**`Button.Paint()` 接口可以通过 Label 的嵌入带到新的结构体，如果 `Button.Paint()` 不实现的话，会调用 `Label.Paint()` ，所以，在 `Button` 中声明 `Paint()` 方法，相当于Override**。



### 嵌入结构多态

通过下面的程序可以看到，整个多态是怎么执行的。

```go
button1 := Button{Label{Widget{10, 70}, "OK"}}
button2 := NewButton(50, 70, "Cancel")
listBox := ListBox{Widget{10, 40}, 
    []string{"AL", "AK", "AZ", "AR"}, 0}

for _, painter := range []Painter{label, listBox, button1, button2} {
    painter.Paint()
}
 
for _, widget := range []interface{}{label, listBox, button1, button2} {
  widget.(Painter).Paint()
  if clicker, ok := widget.(Clicker); ok {
    clicker.Click()
  }
  fmt.Println() // print a empty line 
}
```

我们可以看到，我们可以使用接口来多态，也可以使用 泛型的 `interface{}` 来多态，但是需要有一个类型转换。



## 反转控制

我们再来看一个示例，我们有一个存放整数的数据结构，如下所示：

```go
type IntSet struct {
    data map[int]bool
}
func NewIntSet() IntSet {
    return IntSet{make(map[int]bool)}
}
func (set *IntSet) Add(x int) {
    set.data[x] = true
}
func (set *IntSet) Delete(x int) {
    delete(set.data, x)
}
func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}
```

其中实现了 `Add()` 、`Delete()` 和 `Contains()` 三个操作，前两个是写操作，后一个是读操作。



### 实现Undo功能

现在我们想实现一个 Undo 的功能。我们可以把把 `IntSet` 再包装一下变成 `UndoableIntSet` 代码如下所示：

```go
type UndoableIntSet struct { // Poor style
    IntSet    // Embedding (delegation)
    functions []func()
}
 
func NewUndoableIntSet() UndoableIntSet {
    return UndoableIntSet{NewIntSet(), nil}
}
 

func (set *UndoableIntSet) Add(x int) { // Override
    if !set.Contains(x) {
        set.data[x] = true
        set.functions = append(set.functions, func() { set.Delete(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}


func (set *UndoableIntSet) Delete(x int) { // Override
    if set.Contains(x) {
        delete(set.data, x)
        set.functions = append(set.functions, func() { set.Add(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}

func (set *UndoableIntSet) Undo() error {
    if len(set.functions) == 0 {
        return errors.New("No functions to undo")
    }
    index := len(set.functions) - 1
    if function := set.functions[index]; function != nil {
        function()
        set.functions[index] = nil // For garbage collection
    }
    set.functions = set.functions[:index]
    return nil
}
```

在上面的代码中，我们可以看到

- 我们在 `UndoableIntSet` 中嵌入了`IntSet` ，然后Override了 它的 `Add()`和 `Delete()` 方法。
- `Contains()` 方法没有Override，所以，会被带到 `UndoableInSet` 中来了。
- 在Override的 `Add()`中，记录 `Delete` 操作
- 在Override的 `Delete()` 中，记录 `Add` 操作
- 在新加入 `Undo()` 中进行Undo操作。

通过这样的方式来为已有的代码扩展新的功能是一个很好的选择，这样，可以在重用原有代码功能和重新新的功能中达到一个平衡。但是，这种方式最大的问题是，Undo操作其实是一种控制逻辑，并不是业务逻辑，所以，在复用 Undo这个功能上是有问题。因为其中加入了大量跟 `IntSet` 相关的业务逻辑。



### 反转依赖

现在我们来看另一种方法：

我们先声明一种函数接口，表现我们的Undo控制可以接受的函数签名是什么样的：

```go
type Undo []func()
```

有了上面这个协议后，我们的Undo控制逻辑就可以写成如下：

```go
func (undo *Undo) Add(function func()) {
  *undo = append(*undo, function)
}

func (undo *Undo) Undo() error {
  functions := *undo
  if len(functions) == 0 {
    return errors.New("No functions to undo")
  }
  index := len(functions) - 1
  if function := functions[index]; function != nil {
    function()
    functions[index] = nil // For garbage collection
  }
  *undo = functions[:index]
  return nil
}
```

这里你不必觉得奇怪， `Undo` 本来就是一个类型，不必是一个结构体，是一个函数数组也没什么问题。

然后，我们在我们的IntSet里嵌入 Undo，然后，再在 `Add()` 和 `Delete()` 里使用上面的方法，就可以完成功能。

```go
type IntSet struct {
    data map[int]bool
    undo Undo
}
 
func NewIntSet() IntSet {
    return IntSet{data: make(map[int]bool)}
}

func (set *IntSet) Undo() error {
    return set.undo.Undo()
}
 
func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}

func (set *IntSet) Add(x int) {
    if !set.Contains(x) {
        set.data[x] = true
        set.undo.Add(func() { set.Delete(x) })
    } else {
        set.undo.Add(nil)
    }
}
 
func (set *IntSet) Delete(x int) {
    if set.Contains(x) {
        delete(set.data, x)
        set.undo.Add(func() { set.Add(x) })
    } else {
        set.undo.Add(nil)
    }
}
```

这个就是控制反转，不再由 控制逻辑 `Undo` 来依赖业务逻辑 `IntSet`，而是由业务逻辑 `IntSet` 来依赖 `Undo` 。其依赖的是其实是一个协议，这个协议是一个没有参数的函数数组。我们也可以看到，我们 Undo 的代码就可以复用了。