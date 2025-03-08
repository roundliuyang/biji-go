# map

## 创建map的几种方式



###  使用var 方式

 使用 **var m map[string]int** 来声明一个 **map[string]int** 类型的变量，我们来打印一下该变量的内存地址,为 0x0

```go
func main() {
    var m map[string]int
    if m == nil {
        fmt.Println("m is nil")
    }

    fmt.Printf("%p\n", m)
}
```

当判断是否为nil时，打印出来了 `m is nil`。

当尝试给nil的map 进行赋值时，会直接崩溃 `m["age"] = 18`, 当执行这段代码时就会panic。



### new 方式

`new(Type)` 方式返回的是`*Type`， 返回的是一个指针，这里的指针已经不是空指针了，是有具体的内存地址了

```go
func main() {

    m := new(map[string]int)
    if m == nil {
        fmt.Println("m is nil")
    }

    fmt.Printf("%p\n", m)
    fmt.Println(*m)
}
```

这里判断`m==nil` 失败.

但是依然不能给这个 m 赋值, `(*m)["age"]=18` 依然提示是向nil map 赋值, 需要将 m 指向一个具体的值后才可以正常的赋值

```go
func main() {

    m := new(map[string]int)
    *m = map[string]int{}

    (*m)["name"] = 18
    fmt.Println(*m)
}
```



### make 方式

```go
func main() {

    m := make(map[string]int)
    fmt.Printf("%p\n", m)
    fmt.Println(m)
    m["age"] = 9
    fmt.Println(m)
    fmt.Printf("%p\n", m)
}
```

使用make方式可以返回一个声明并初始化的变量，对于这个变量，我们可以对其进行直接操作。 上面代码输出为

```shell
0xc00006e180
map[]
map[age:9]
0xc00006e180
```



### 声明的时候并初始化

这种方式是比较传统的，可以将其想象成 `var s string = "name"`

```go
func main(){
  var m = map[string]int{} //这里的{} 不能少
  m["age"] = 18
  fmt.Pringln(m)
}
```



## json 中的应用

有很多时候，我们将json进行反序列化的时候，会转成结构体或者map，

```go
func main() {
    jstr := `{"name":"yangyanxing", "age": 18}`
    var m map[string]interface{}
    mm := make(map[string]interface{})
    var mmm = map[string]interface{}{}
    var mmmm = new(map[string]interface{})
    json.Unmarshal([]byte(jstr), &m)
    json.Unmarshal([]byte(jstr), &mm)
    json.Unmarshal([]byte(jstr), &mmm)
    json.Unmarshal([]byte(jstr), mmmm)
    fmt.Println(m)
    fmt.Println(mm)
    fmt.Println(mmm)
    fmt.Println(*mmmm)
}
```

上面四种方式初始化map对象，在`json.Unmarshl` 方法都可以将json 转为map,注意使用new 返回的本身就是指针了，就不用再用`&` 取地址符了，其实即使用的取地址符，也可以得到相同的结果。