# Go validator验证参数是否零值以及是否传递



## 问题场景

在Go中，当使用encoding/json包解码JSON数据到结构体时：

- 如果前端未传递某个字段，validator会将该字段设置为其类型的零值。
- 如果前端传递了该字段，并且是零值，validator同样会将其设置为相应的零值。

这意味着，仅凭字段的值，无法区分字段是**未传递**还是**传递了零值**。这在某些场景下可能导致问题。

## 解决方法

使用**指针**与`required`标签，可以解决上面的问题。



### 指针

将字段设置为指针类型，那么：

- 如果前端未传递该字段，validator会将指针设置为`nil`。
- 如果前端有传递该字段，validator会将指针指向相应的值。

这意味着，使用指针，可以区分参数**传递了**或**未传递**。



### `required`标签

`required`会验证字段值是否为**零值**，如果为零值，则验证不通过。



### 结合使用

对于接口中定义的字段参数，按照是否必传、是否可为零值，可分4种情况：

1. 必传，不可以为零值

   参数是必传的，并且不能是零值（比如购买数量）

   `required`

2. 必传，可为零值

   这种情况，参数是必传的，如果传的是零也要判断出来（比如传状态0或1）

   `required` + 指针

3. 非必传，不可以为零值

   这种情况，前端可以不传，但是传的参数不能是【零】（比如前端传筛选条件，要么不筛选，要么提供一个有效的值）

   非零条件

4. 非必传，可为零值

   这种情况，前端可以不传，但是传了【零】，我要能判断出来（比如把数量改为0，把描述文字改为空）

   指针

## 总结

如果前端传的【零】有效，就使用**指针**。

必选参数，加`required`，没什么好说的。