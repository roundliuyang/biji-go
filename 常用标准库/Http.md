# Http

Go语言内置的net/http包十分的优秀，提供了HTTP客户端和服务端的实现。



## net/http介绍

Go语言内置的net/http包提供了HTTP客户端和服务端的实现。



## HTTP客户端

基本的HTTP/HTTPS请求 Get、Head、Post和PostForm函数发出HTTP/HTTPS请求。

```go
resp, err := http.Get("http://5lmh.com/")
...
resp, err := http.Post("http://5lmh.com/upload", "image/jpeg", &buf)
...
resp, err := http.PostForm("http://5lmh.com/form",
    url.Values{"key": {"Value"}, "id": {"123"}})
```

程序在使用完response后必须关闭回复的主体。

```go
resp, err := http.Get("http://5lmh.com/")
if err != nil {
    // handle error
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
// ...
```



## GET请求示例

使用net/http包编写一个简单的发送HTTP请求的Client端，代码如下：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "net/http"
)

func main() {
    resp, err := http.Get("https://www.5lmh.com/")
    if err != nil {
        fmt.Println("get failed, err:", err)
        return
    }
    defer resp.Body.Close()
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("read from resp.Body failed,err:", err)
        return
    }
    fmt.Print(string(body))
}
```

将上面的代码保存之后编译成可执行文件，执行之后就能在终端打印liwenzhou.com网站首页的内容了，我们的浏览器其实就是一个发送和接收HTTP协议数据的客户端，我们平时通过浏览器访问网页其实就是从网站的服务器接收HTTP数据，然后浏览器会按照HTML、CSS等规则将网页渲染展示出来。





## 带参数的GET请求示例

关于GET请求的参数需要使用Go语言内置的net/url这个标准库来处理。

```go
func main() {
    apiUrl := "http://127.0.0.1:9090/get"
    // URL param
    data := url.Values{}
    data.Set("name", "枯藤")
    data.Set("age", "18")
    u, err := url.ParseRequestURI(apiUrl)
    if err != nil {
        fmt.Printf("parse url requestUrl failed,err:%v\n", err)
    }
    u.RawQuery = data.Encode() // URL encode
    fmt.Println(u.String())
    resp, err := http.Get(u.String())
    if err != nil {
        fmt.Println("post failed, err:%v\n", err)
        return
    }
    defer resp.Body.Close()
    b, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("get resp failed,err:%v\n", err)
        return
    }
    fmt.Println(string(b))
}
```

对应的Server端HandlerFunc如下：

```go
func getHandler(w http.ResponseWriter, r *http.Request) {
    defer r.Body.Close()
    data := r.URL.Query()
    fmt.Println(data.Get("name"))
    fmt.Println(data.Get("age"))
    answer := `{"status": "ok"}`
    w.Write([]byte(answer))
}
```





## Post请求示例

上面演示了使用net/http包发送GET请求的示例，发送POST请求的示例代码如下：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "net/http"
    "strings"
)

// net/http post demo

func main() {
    url := "http://127.0.0.1:9090/post"
    // 表单数据
    //contentType := "application/x-www-form-urlencoded"
    //data := "name=枯藤&age=18"
    // json
    contentType := "application/json"
    data := `{"name":"枯藤","age":18}`
    resp, err := http.Post(url, contentType, strings.NewReader(data))
    if err != nil {
        fmt.Println("post failed, err:%v\n", err)
        return
    }
    defer resp.Body.Close()
    b, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("get resp failed,err:%v\n", err)
        return
    }
    fmt.Println(string(b))
}
```

对应的Server端HandlerFunc如下：

```go
func postHandler(w http.ResponseWriter, r *http.Request) {
    defer r.Body.Close()
    // 1. 请求类型是application/x-www-form-urlencoded时解析form数据
    r.ParseForm()
    fmt.Println(r.PostForm) // 打印form数据
    fmt.Println(r.PostForm.Get("name"), r.PostForm.Get("age"))
    // 2. 请求类型是application/json时从r.Body读取数据
    b, err := ioutil.ReadAll(r.Body)
    if err != nil {
        fmt.Println("read request.Body failed, err:%v\n", err)
        return
    }
    fmt.Println(string(b))
    answer := `{"status": "ok"}`
    w.Write([]byte(answer))
}
```



## 自定义Client

要管理HTTP客户端的头域、重定向策略和其他设置，创建一个Client：

```go
client := &http.Client{
    CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http://5lmh.com")
// ...
req, err := http.NewRequest("GET", "http://5lmh.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)
resp, err := client.Do(req)
// ...
```



## 自定义Transport

要管理代理、TLS配置、keep-alive、压缩和其他设置，创建一个Transport：

```go
tr := &http.Transport{
    TLSClientConfig:    &tls.Config{RootCAs: pool},
    DisableCompression: true,
}
client := &http.Client{Transport: tr}
resp, err := client.Get("https://5lmh.com")
```

Client和Transport类型都可以安全的被多个go程同时使用。出于效率考虑，应该一次建立、尽量重用。



## 服务端



### 默认的Server

ListenAndServe使用指定的监听地址和处理器启动一个HTTP服务端。处理器参数通常是nil，这表示采用包变量DefaultServeMux作为处理器。

Handle和HandleFunc函数可以向DefaultServeMux添加处理器。

```go
http.Handle("/foo", fooHandler)
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})
log.Fatal(http.ListenAndServe(":8080", nil))
```



### 默认的Server示例

使用Go语言中的net/http包来编写一个简单的接收HTTP请求的Server端示例，net/http包是对net包的进一步封装，专门用来处理HTTP协议的数据。具体的代码如下：

```go
// http server

func sayHello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello 枯藤！")
}

func main() {
    http.HandleFunc("/", sayHello)
    err := http.ListenAndServe(":9090", nil)
    if err != nil {
        fmt.Printf("http server failed, err:%v\n", err)
        return
    }
}
```

将上面的代码编译之后执行，打开你电脑上的浏览器在地址栏输入127.0.0.1:9090回车，此时就能够看到 `Hello 枯藤！`



### 自定义Server

要管理服务端的行为，可以创建一个自定义的Server：

```go
s := &http.Server{
    Addr:           ":8080",
    Handler:        myHandler,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
}
log.Fatal(s.ListenAndServe())
```